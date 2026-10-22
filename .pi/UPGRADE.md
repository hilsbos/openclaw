# Upgrading the `custom` branch to latest upstream

## Overview

This repo is a fork of `openclaw/openclaw` at `https://github.com/hilsbos/openclaw`.
The `custom` branch stays a few commits ahead of `upstream/main` with local customizations.
All local changes are rebased on top of upstream — never merged.

Remotes:

- `origin` → `hilsbos/openclaw` (our fork — push here)
- `upstream` → `openclaw/openclaw` (read-only — push disabled)

## Pre-flight

Run everything as user `max` in `/home/max/Projects/openclaw`.

```bash
cd /home/max/Projects/openclaw
git status  # must be clean, on branch `custom`
```

### Check for newer pinned-binary releases

Dockerfile.local pins `gogcli` and `goplaces` via `GOGCLI_VERSION` /
`GOPLACES_VERSION`. Before rebuilding, check for newer upstream releases —
these binaries ship their own Go stdlib, so pinned versions carry Go stdlib
CVEs until bumped. We deliberately do NOT track `latest`: unpinned URLs
break silently when upstream renames assets (this is how goplaces was broken
for a while), and feature drift would land without review.

```bash
current_gog=$(grep '^ARG GOGCLI_VERSION=' Dockerfile.local | cut -d= -f2)
current_gp=$(grep '^ARG GOPLACES_VERSION=' Dockerfile.local | cut -d= -f2)
latest_gog=$(curl -sL https://api.github.com/repos/steipete/gogcli/releases/latest | grep tag_name | cut -d\" -f4)
latest_gp=$(curl -sL https://api.github.com/repos/steipete/goplaces/releases/latest | grep tag_name | cut -d\" -f4)
echo "gogcli:    pinned=v$current_gog  latest=$latest_gog"
echo "goplaces:  pinned=v$current_gp  latest=$latest_gp"
```

If either is stale, skim the release notes at
`https://github.com/steipete/<repo>/releases/tag/<tag>`, then bump the
corresponding `ARG` line in `Dockerfile.local`. Commit the bump as part of
this upgrade.

## Step 1: Fetch upstream and pick the target

**Prefer the latest release tag over `upstream/main`.** Tags are stable;
`main` may contain in-progress or beta work that isn't yet cut.

```bash
git fetch upstream --tags
git tag -l 'v*' --sort=-v:refname | grep -v beta | head -5   # candidate tags
TARGET=$(git tag -l 'v*' --sort=-v:refname | grep -v beta | head -1)
echo "Target: $TARGET"

git log --oneline custom..$TARGET | head -20             # review what is new
git rev-list --count custom..$TARGET                     # count new commits
git diff --name-only custom..$TARGET | wc -l             # changed file count
```

Only target `upstream/main` if you explicitly need an unreleased fix.
Substitute `$TARGET` for `upstream/main` throughout the rest of this doc.

## Step 2: Security review of upstream changes

Before rebasing, review the new upstream code for security issues.

### 2a. Dependency audit

```bash
pnpm audit 2>/dev/null || echo "pnpm not installed on host — skip or run inside container"
```

### 2b. Secrets scan (gitleaks)

Scan only the new commits for accidentally committed secrets:

```bash
gitleaks detect --source . --log-opts="custom..$TARGET" --no-banner --report-format json --report-path /tmp/gitleaks-report.json
```

Expect many false positives in `ui/src/i18n/.i18n/*.tm.jsonl` (translation memory)
and `*.test.ts` fixtures (obvious placeholder tokens). Filter those out:

```bash
python3 -c "
import json
leaks = json.load(open('/tmp/gitleaks-report.json'))
real = [l for l in leaks if '.i18n' not in l['File'] and '.test.' not in l['File']]
for l in real:
    print(f\"{l['RuleID']:20s}  {l['File']}:{l['StartLine']}  {l['Secret'][:60]}\")
"
```

### 2c. Static analysis (semgrep)

Semgrep's `--config r/security-audit` registry pull sometimes returns
"No config given" under concurrent `--baseline-commit` scans. The reliable
pattern is to scan a worktree of the target directly with `p/security-audit`:

```bash
git worktree add /tmp/openclaw-$TARGET $TARGET
cd /tmp/openclaw-$TARGET && \
  semgrep scan --config p/security-audit --json --quiet --metrics=off \
    --timeout 60 --max-target-bytes 2000000 . > /tmp/semgrep-report.json
cd -
git worktree remove /tmp/openclaw-$TARGET --force
```

Filter findings to the upstream-changed files only (ignore baseline findings
that exist in `custom` already):

```bash
git diff --name-only custom..$TARGET | grep -E '\.(ts|js|mjs|cjs|json|sh)$' > /tmp/changed-files.txt
python3 -c "
import json
d = json.load(open('/tmp/semgrep-report.json'))
changed = set(open('/tmp/changed-files.txt').read().splitlines())
for r in d.get('results', []):
    if r.get('path') in changed:
        print(f\"{r.get('extra', {}).get('severity', '?')}  {r['path']}:{r.get('start', {}).get('line')}  {r['check_id']}\")
"
```

### 2d. AI-assisted diff review

Ask the AI assistant to review the upstream diff for security concerns:

```bash
git diff custom..$TARGET -- '*.ts' '*.js' '*.sh' Dockerfile docker-compose.yml > /tmp/upstream-diff.txt
```

Then ask: "Review this diff for security issues: backdoors, obfuscated code, suspicious network calls, credential exposure, or supply chain risks."

Focus areas:

- Changes to Dockerfile, docker-compose.yml, any shell scripts
- New dependencies in package.json / pnpm-lock.yaml
- Changes to auth, gateway, or network-facing code
- New `eval()`, `exec()`, `child_process`, or dynamic imports
- Scan commit messages for a security track-record: `git log --oneline custom..$TARGET | grep -iE "security|auth|cred|secret"`

## Step 3: Rebase (choose a strategy based on divergence size)

First measure how far apart we are:

```bash
git rev-list --count custom..$TARGET                           # incoming commits
BASE=$(git merge-base custom $TARGET)
git diff --name-only ${BASE}..custom | wc -l                   # files WE touch
```

### Strategy A — literal rebase (small delta: < ~200 incoming commits)

```bash
git rebase $TARGET
# resolve conflicts via the cheatsheet below, then:
git add <resolved-file>
GIT_EDITOR=true git rebase --continue
# abort if things go sideways: git rebase --abort
git clean -fd
```

### Strategy B — category-replay (large delta: 200+ incoming commits)

Upstream moves fast (sometimes 500+ commits/day during refactor bursts).
A literal rebase across a thousand-commit delta is conflict-hell. Instead,
create a fresh branch at the target and replay our customizations in
categories. Our custom delta is usually ~20 files — most of which don't
conflict at all.

**Caveat (seen in the v2026.7.1 upgrade):** `git merge-base custom $TARGET`
can silently return a much older ancestor than the actual prior release tag
if upstream rewrote its own history between releases (upstream force-pushing
a rebased main is not under our control, and has now happened at least
twice). When that happens, the naive inventory loop below treats every file
upstream touched since that ancient point as "ours" — hundreds of false
positives. Sanity-check with
`git merge-base --is-ancestor <prior-release-tag> $TARGET` first; if it
fails (not an ancestor), use the prior release tag you upgraded FROM as
`BASE` directly instead of the computed merge-base — that's always the
correct base for "what did we actually customize," regardless of whether
upstream's history is linear.

```bash
BASE=$(git merge-base custom $TARGET)

# 1. Inventory what WE touched, classified by whether upstream changed it too
for f in $(git diff --name-only ${BASE}..custom); do
  if git cat-file -e $TARGET:"$f" 2>/dev/null; then
    us=$(git diff ${BASE}..custom -- "$f" | wc -l)
    up=$(git diff ${BASE}..$TARGET -- "$f" | wc -l)
    printf "MODIFIED  us=%4d  up=%5d  %s\n" "$us" "$up" "$f"
  else
    echo "ADDITIVE                    $f"
  fi
done
```

Then branch from the target and replay in three commits:

```bash
git checkout -b custom-NEW $TARGET

# Commit 1 — additive files (files upstream doesn't have). Zero conflict risk.
git checkout custom -- <additive files>
git commit -m "local: additive overlay files"

# Commit 2 — modified files where upstream had ZERO churn.
# `git checkout custom -- <file>` gives "upstream + our delta" exactly,
# because upstream didn't touch the file.
git checkout custom -- <zero-churn files>
git commit -m "local: replay on zero-churn files"

# Commit 3+ — files upstream ALSO changed. Use 3-way apply:
git diff ${BASE}..custom -- <conflict-risk file> > /tmp/patch.diff
git apply --3way --index /tmp/patch.diff
# inspect, resolve any <<<<<< markers, then:
git commit -m "local: replay on upstream-churned files"
```

Verify every touched file matches `custom` (files with upstream churn
should differ only by the upstream delta, which is intended):

```bash
for f in $(git diff --name-only ${BASE}..custom); do
  if [ -z "$(git diff custom..custom-NEW -- "$f")" ]; then
    echo "  OK match: $f"
  else
    echo "  DIFFERS (check upstream churn is intended): $f"
  fi
done
```

Then swap branches. Preserve the old `custom` with a version-suffixed name
so older safety branches aren't clobbered:

```bash
# Rename prior safety branch(es) if they exist, using the base version:
BASE_TAG=$(git describe $BASE)   # e.g., v2026.4.14
[ -e "$(git rev-parse --verify custom-old 2>/dev/null)" ] && \
  git branch -m custom-old custom-old-${BASE_TAG#v}
git branch -m custom custom-old-${BASE_TAG#v}-pre
git branch -m custom-NEW custom
git clean -fd
```

## Step 4: Push to fork

```bash
git push origin custom --force-with-lease
```

## Step 5: Rebuild and deploy Docker

The build is two-step: upstream Dockerfile produces the base image, then
Dockerfile.local layers our customizations.

```bash
cd /home/max/Projects/openclaw

# Step 1: Build upstream base with matrix extension (~15s cached, ~5min cold)
docker build -t openclaw-base:latest \
  --build-arg OPENCLAW_INSTALL_BROWSER=1 \
  --build-arg OPENCLAW_EXTENSIONS="matrix brave diagnostics-otel" \
  -f Dockerfile .

# Step 2: Build local overlay (adds our tools + fixes, ~5s cached)
docker build -t openclaw:latest -f Dockerfile.local .

docker compose down
docker compose up -d
docker compose logs -f --tail=50   # watch for startup errors
```

## Step 6: Post-deploy verification

```bash
# Container healthy?
docker ps | grep openclaw

# All expected plugins loaded? (compare names/count to previous version)
docker logs openclaw-openclaw-gateway-1 2>&1 | grep -E "listening \(.* plugins"  # v2026.6.5+: "http server listening (N plugins: ...)"

# No matrix "Cannot find package 'openclaw'" errors?
docker logs openclaw-openclaw-gateway-1 2>&1 | grep -iE "crypto bootstrap|channel exited|cannot find package"

# memory-core auto-registered its managed dreaming cron? (v4.10+)
docker exec openclaw-openclaw-gateway-1 openclaw cron list 2>&1 | grep -i "memory dreaming"

# Memory embedding provider resolves? (catches plugin allowlist regressions)
docker exec openclaw-openclaw-gateway-1 openclaw memory status 2>&1 | grep -E "Provider|Vector"

# Image vulnerability scan
trivy image openclaw:latest --severity HIGH,CRITICAL --quiet
```

### v4.10+ plugin allowlist gotcha

OpenClaw v4.10+ enforces `plugins.allow` strictly. Any extension your
customizations depend on MUST appear in both `plugins.allow` AND
`plugins.entries` in `~/.openclaw/openclaw.json`, or it will be silently
unloaded even if bundled in the image.

**Symptom:** `openclaw memory status` fails with
`Unknown memory embedding provider: ollama` (or similar), and the gateway
startup `ready` line shows fewer plugins than expected.

**Fix for ollama specifically:**

```json
{
  "plugins": {
    "allow": [ "...", "ollama" ],
    "entries": {
      "...": {},
      "ollama": { "enabled": true }
    }
  }
}
```

Then `docker compose restart openclaw-gateway`.

### v2026.4.15+ matrix self-reference gotcha

Matrix is bundled into `/app/dist/extensions/matrix/` with imports like
`from "openclaw/plugin-sdk/error-runtime"`. The matrix subtree's
`package.json` is named `@openclaw/matrix`, so Node's self-reference
resolution can't find a parent package named `openclaw` and falls back to
`node_modules/openclaw` lookup — which upstream doesn't stage.

**Symptom:** `openclaw: failed loading crypto bootstrap runtime: Cannot find
package 'openclaw' imported from /app/dist/extensions/matrix/errors-*.js`
at startup, followed by all matrix channels crashing in a restart loop:
`channel exited: Cannot find package 'openclaw' imported from
/app/dist/extensions/matrix/env-vars-*.js`.

**Fix:** Dockerfile.local adds `ln -sf /app /app/node_modules/openclaw`,
which makes `openclaw` resolvable via the standard `node_modules` lookup
path. Do NOT remove that line unless upstream publishes a fix.

---

## What our customization commits contain

The `custom` branch adds ~20 files on top of upstream. These fall into three buckets:

### Additive (files upstream doesn't have)

| File                          | Purpose                                                              |
| ----------------------------- | -------------------------------------------------------------------- |
| `.pi/UPGRADE.md`              | This document                                                        |
| `Dockerfile.local`            | Runtime overlay: UID/GID remap, socat, gh, gogcli, goplaces, matrix openclaw symlink, matrix plugin-dependency hoisting |
| `docker-compose.override.yml` | Dev overrides: 127.0.0.1 binding, env_file, ollama sidecar, extra env vars |

### Modified (our delta on upstream files)

| File                                                      | What changed                                                                          |
| --------------------------------------------------------- | ------------------------------------------------------------------------------------- |
| `.gitignore`                                              | Additional local ignores                                                              |
| `apps/ios/*` (4 files)                                    | iOS gateway connection + build settings for maX app                                   |
| `extensions/diagnostics-otel/src/service.ts`              | Extra telemetry fields                                                                |
| `extensions/matrix/src/cli.ts`                            | Custom matrix CLI behavior                                                            |
| `extensions/matrix/src/matrix/client/logging.ts`          | Matrix log formatting tweaks                                                          |
| `src/agents/embedded-agent-subscribe.handlers.tools.ts`   | Emit OTel `tool.call` diagnostic event on tool execution end (pre-v2026.6.5 path: `pi-embedded-subscribe.handlers.tools.ts`) |
| `src/infra/diagnostic-events.ts`                          | Custom diagnostic event shapes                                                        |
| `ui/index.html`                                           | Page title → `maX`                                                                    |
| `ui/src/styles/base.css`                                  | Electric blue accent theme (dark + light mode)                                        |
| `ui/src/ui/app-render.ts`                                 | Sidebar brand: no logo, eyebrow → `shushu`, title → `maX`                             |
| `ui/src/ui/components/dashboard-header.ts`                | Breadcrumb root text → `maX`                                                          |

### Upstream (not customized)

`Dockerfile` and `docker-compose.yml` are vendored from upstream unchanged
after the v2026.4.15 cleanup. All our runtime and compose customizations
live in the two files above (`Dockerfile.local`, `docker-compose.override.yml`),
which compose auto-merges on top of upstream.

## Conflict resolution cheatsheet

Only three files consistently have upstream churn AND our delta, so these
are the ones that need 3-way apply attention during rebase:

### `src/agents/embedded-agent-subscribe.handlers.tools.ts`

(Renamed from `pi-embedded-subscribe.handlers.tools.ts` in v2026.6.5's "internalize
OpenClaw agent runtime" refactor — if upstream renames it again, find the new home
with `git grep -l handleToolExecutionEnd <TARGET>` and port the patch by hand.)

- Keep our diagnostic `emitDiagnosticEvent({ type: "tool.call", ... })` block
  after the `emitToolResultOutput(...)` call.
- Take upstream's refactors (parameter additions, tool name normalization).

### `ui/src/ui/app-render.ts`

- Keep our `<span class="sidebar-brand__eyebrow">shushu</span>` + `<span
  class="sidebar-brand__title">maX</span>` block, and the
  "no logo" decision in the sidebar brand.
- Take upstream's new fields/refresh params on Overview dashboard render.

### Other UI files

- `base.css`: Keep our electric blue accent colors, take upstream's structural changes.
- `dashboard-header.ts`: Keep `maX` in breadcrumb.
- `index.html`: Keep `maX` title.

### Dockerfile / docker-compose.yml

Do NOT re-introduce custom delta here. Upstream is the source of truth.
All customizations live in `Dockerfile.local` and `docker-compose.override.yml`.
If a rebase accidentally puts custom content back into one of these, reset it:

```bash
git checkout $TARGET -- Dockerfile docker-compose.yml
```

### Binary versions in Dockerfile.local

`gh` runs on apt for auto-updates. `gogcli` and `goplaces` are pinned via
top-of-file ARGs — see pre-flight "Check for newer pinned-binary releases".

## Updating model versions in openclaw.json

When bumping a model to a newer version (e.g. claude-opus-4-7 → 4-8), you must
update it in **two separate places** in `~/.openclaw/openclaw.json`. Missing either
one will break the gateway.

### Step 1 — agents.defaults.models (alias/settings registry)

This is where you assign the `"opus"` alias and other per-model settings:

```json
"agents.defaults.models": {
  "anthropic/claude-opus-4-7": {},
  "anthropic/claude-opus-4-8": { "alias": "opus" }
}
```

Keep the old entry (without an alias) so existing sessions that stored the full
model ID can still resume.

### Step 2 — models.providers["anthropic"].models[] (provider registration)

Built-in Anthropic models like opus 4-7 are pre-registered by default.
Newer models are NOT. If a model ID doesn't appear in the provider's models list,
the gateway fails at startup with:

```
Unknown model: anthropic/claude-opus-4-8. Found agents.defaults.models[...],
but no matching models.providers["anthropic"].models[] entry.
Add { "id": "claude-opus-4-8" } to models.providers["anthropic"].models[].
```

Both `id` AND `name` are required — omitting `name` causes a second validation error:

```
models.providers.anthropic.models.0.name: Invalid input: expected string
```

Add the entry:

```json
"models": {
  "providers": {
    "anthropic": {
      "models": [
        { "id": "claude-opus-4-8", "name": "Claude Opus 4.8" }
      ]
    }
  }
}
```

### Safe update pattern (Python, run as max)

```bash
sudo -u max bash -c 'python3 << "PYEOF"
import json
path = "/home/max/.openclaw/openclaw.json"
with open(path) as f:
    d = json.load(f)

# Step 1: add alias to new model, strip alias from old
models_cfg = d["agents"]["defaults"]["models"]
models_cfg.setdefault("anthropic/claude-opus-4-8", {})["alias"] = "opus"
models_cfg.setdefault("anthropic/claude-opus-4-7", {}).pop("alias", None)

# Step 2: register in provider list if not already there
provider_models = d.setdefault("models", {}).setdefault("providers", {}).setdefault("anthropic", {}).setdefault("models", [])
if not any(m.get("id") == "claude-opus-4-8" for m in provider_models):
    provider_models.append({"id": "claude-opus-4-8", "name": "Claude Opus 4.8"})

with open(path, "w") as f:
    json.dump(d, f, indent=2)
    f.write("\n")
PYEOF
'
```

Then restart: `cd /home/max/Projects/openclaw && docker compose restart openclaw-gateway`

## Root-owned files in ~/.openclaw break config migrations (seen v2026.6.5)

Upstream config migrations write backups like
`openclaw.json.bak-pre-dreaming-separate` into `~/.openclaw`. If a file with
that exact name already exists owned by `root:root` (e.g. from a past manual
`sudo openclaw ...` run), every matrix channel crash-loops at startup with:

```
channel exited: Backup archive write failed: EACCES: permission denied,
open '/home/node/.openclaw/openclaw.json.bak-...'
```

The container is healthy and the gateway starts; only the channels die. Fix:

```bash
sudo find /home/max/.openclaw -maxdepth 1 -user root
# rename aside + chown to max, then: docker compose restart openclaw-gateway
```

Pre-flight tip: run that `find` before deploying to catch root-owned strays early.

## v2026.6.10 upgrade notes (2026-06-26)

### gogcli bumped 0.23.0 → 0.31.0

Releases v0.24–v0.31 are all Google Workspace feature additions (Calendar, Gmail,
Sheets, YouTube, Contacts, Photos, Auth). No security issues. Safe to upgrade.
`goplaces` stayed at v0.4.3 (already current).

### OpenClaw.entitlements conflict resolved by upstream

Our custom delta removed the hardcoded `aps-environment: development` key.
Upstream v2026.6.10 changed the same key to use
`$(OPENCLAW_APNS_ENTITLEMENT_ENVIRONMENT)` and added `com.apple.security.application-groups`.
The 3-way apply conflicted; resolution: take upstream version as-is (our intent to
remove the hardcoded value is already satisfied).

### Upstream Dockerfile now prunes openclaw symlinks in prod layer

v2026.6.10 added a `rm -rf /app/node_modules/openclaw ...` in the prod prune step.
Our `Dockerfile.local` re-adds `ln -sf /app /app/node_modules/openclaw` after this,
so the matrix self-reference fix still works — just layered correctly.

### Strategy B stats

- Incoming commits: 3073 (v2026.6.5 → v2026.6.10)
- Our custom delta: 15 files (3 additive, 2 zero-churn, 10 upstream-churned)
- 9 of 10 churned files applied cleanly with `--3way`; 1 conflict (entitlements, see above)

### agents.defaults.models schema change (v2026.6.5 → v2026.6.10)

v2026.6.10 changed `agents.defaults.models` from accepting a string array to
requiring a plain object (model IDs as keys, config object as values). The
gateway fails at startup with `agents.defaults.models: Invalid input` if the
old array form is present.

**Symptom:** Gateway crash-loops immediately after deploy — no plugin load line.

**Fix:** Convert the array to an object before or right after deploy:

```bash
python3 << "PYEOF"
import json
path = "/home/max/.openclaw/openclaw.json"
with open(path) as f:
    d = json.load(f)
models = d["agents"]["defaults"]["models"]
if isinstance(models, list):
    d["agents"]["defaults"]["models"] = {m: {} for m in models}
    with open(path, "w") as f:
        json.dump(d, f, indent=2)
        f.write("\n")
    print("Converted.")
PYEOF
docker compose restart openclaw-gateway
```

**Pre-flight tip:** Before deploying, check:
```bash
python3 -c "import json; d=json.load(open(/home/max/.openclaw/openclaw.json)); print(type(d[agents][defaults][models]))"
# Should print <class dict>; if <class list>, convert first.
```

## v2026.6.11 upgrade notes (2026-06-30)

### gogcli bumped 0.31.0 → 0.31.1

v0.31.1 adds Calendar feature additions (listing recently modified events, attendee
modifiers). No security issues. Safe to upgrade. `goplaces` stayed at v0.4.3 (already
current).

### embedded-agent-subscribe.handlers.tools.ts conflict

Our `emitDiagnosticEvent({ type: "tool.call", ... })` block conflicted with upstream's
changes. Resolution: kept our `emitDiagnosticEvent` block — the upstream side was empty
at the conflict location (upstream added new tool-call routing logic elsewhere in the
function). The file now matches our prior `custom` branch exactly.

### Prior rebase history fixed

The previous upgrade (v2026.6.10) used `git rebase` instead of Strategy B's
`git checkout -b custom-NEW $TARGET`, causing all upstream v2026.6.10 commits to be
rewritten with new hashes. This resulted in the merge-base between our branch and
v2026.6.11 being described as `v2026.4.19-beta.2-28665-g83785a6e79` rather than
v2026.6.10. This upgrade's Strategy B (`git checkout -b custom-NEW v2026.6.11`)
correctly creates the branch at the exact tag commit. Future upgrades will have
`git merge-base custom v2026.6.12` return the v2026.6.11 tag itself.

### Strategy B stats

- Incoming commits: 1067 (v2026.6.10 → v2026.6.11)
- Our custom delta: 14 files (3 additive, 5 zero-churn, 6 upstream-churned)
- 5 of 6 churned files applied cleanly with `--3way`; 1 conflict (handlers.tools.ts, see above)

### Trivy image scan findings (informational)

Image-level CVEs are all either Debian base OS issues (`will_not_fix`/`fix_deferred`)
or Go stdlib CVEs in pinned binaries (gogcli v0.31.1 embeds go1.26.2; needs 1.26.4 for
full fix — awaiting upstream binary release). The socat CRITICAL (CVE-2026-56123)
affects versions 1.8.0.0–1.8.1.1; we run 1.7.4.4 (not affected). No application-level
CVEs introduced by our changes.

## v2026.7.1 upgrade notes (2026-07-27)

### Upstream rewrote its own history again between v2026.6.11 and v2026.7.1

`git merge-base --is-ancestor v2026.6.11 v2026.7.1` failed — none of
v2026.6.11's commits are reachable from v2026.7.1, even though our own
Strategy B branch was created correctly last cycle (`custom-NEW` was
branched exactly at the v2026.6.11 tag commit). This is upstream rewriting
their own main branch history between release cuts, not something under our
control. `git merge-base custom v2026.7.1` fell back to a `v2026.4.19-beta.2`
ancestor, which would have made the naive file-inventory loop flag ~230
files as "ours" instead of the real ~13. See the new caveat added to the
Strategy B section above — use the known prior release tag (`v2026.6.11`)
directly as `BASE`, not the computed merge-base.

### gogcli bumped 0.31.1 → 0.34.1, goplaces 0.4.3 → 0.4.4

v0.32–0.34.1 are all Workspace feature additions (Calendar, Docs, Sheets,
Auth keychain trust). No security issues. goplaces 0.4.4 likewise
feature-only.

### UI refactor: app-render.ts split into components; port pattern for next time

Upstream restructured the Control UI between v2026.6.11 and v2026.7.1:
`ui/src/ui/app-render.ts` (a single template-function render) no longer
exists — the sidebar brand markup moved into a `renderBrand()` method on
the new `ui/src/components/app-sidebar.ts` LitElement class.
`ui/src/ui/components/dashboard-header.ts` moved to
`ui/src/components/dashboard-header.ts` and was also converted to a
LitElement class, gaining brand-crumb dedup logic
(`agentLabel.toLowerCase() === "openclaw" ? "" : agentLabel`, to avoid
"OpenClaw › OpenClaw › …").

Since these paths no longer exist at the old location, `git apply --3way`
can't touch them (the inventory script correctly reports them ADDITIVE
against the target, meaning "doesn't exist here" — not "safe to blind-copy").
They need to be ported by hand into the new file/method:
- `app-sidebar.ts` `renderBrand()`: replaced the `<img class="sidebar-brand__logo">`
  + single title with our no-logo, `shushu` eyebrow + `maX` title block.
- `dashboard-header.ts`: swapped `OpenClaw` → `maX` in both breadcrumb
  branches, and updated the new dedup check to compare against `"max"`
  instead of `"openclaw"`.
- `layout.css`: upstream **removed** `.sidebar-brand__copy`/`.sidebar-brand__eyebrow`
  entirely when it simplified to a single-line title — these have to be
  re-added (sized to match upstream's new tighter `.sidebar-brand__title`,
  14px/650 weight, not the old 15px/700 values) or the eyebrow+title render
  unstyled.

If upstream refactors this again, `git grep -n "sidebar-brand" <TARGET> -- 'ui/*'`
finds the new home.

### GatewayConnectionController.swift conflict: unrelated upstream refactor in the same spot

Upstream independently changed `finish()` from taking a plain fingerprint
string to a `GatewayTLSFingerprintProbeResult` enum, and split out a new
`didCompleteWithError` delegate method — unrelated to our change, but
touching the exact same lines. Resolution: kept upstream's new
enum-based `finish()` calls and the new delegate method, but kept our
actual fix (`completionHandler(.useCredential, credential)` instead of
`.cancelAuthenticationChallenge`, so the TLS handshake completes) inside
that new structure.

### Memory Core legacy migration: doctor --fix has no path for meta/chunks conflicts

Two agents (jarvis, sensei) hit
`Skipped Memory Core legacy memory index import ... legacy rows could not
be imported` on gateway startup, which is treated as fatal (`refusing to
report the gateway ready`) — same failure class as the `main` conflict
noted in the v2026.6.5 section, but `openclaw doctor --fix` does **not**
have an automated resolution path for it (ran it, warning was unchanged
afterward). Investigated with sqlite3 directly before doing anything:

```bash
sqlite3 /home/max/.openclaw/memory/<agent>.sqlite "select count(*) from meta;"  # etc.
# ATTACH the canonical agents/<agent>/agent/openclaw-agent.sqlite and diff
# meta.providerKey, and check every overlapping chunk id's hash/model match.
```

For both agents the canonical per-agent store was already a strict
superset of the legacy sidecar (every legacy file path and chunk id present
in canonical, with matching hash/model on all overlaps) — canonical had
clearly been active independently for a while. jarvis's conflict was a
genuine `meta.providerKey` mismatch (safety check refusing to merge
possibly-different embedding configs); sensei's was a same-content
primary-key collision on blind insert, not real divergence. Once confirmed
safe, the fix was a manual archive-rename (matching what the automatic
migration does on success):

```bash
mv /home/max/.openclaw/memory/<agent>.sqlite /home/max/.openclaw/memory/<agent>.sqlite.migrated
chown max:max /home/max/.openclaw/memory/<agent>.sqlite.migrated
docker compose restart openclaw-gateway
```

Do NOT do this without first confirming the legacy→canonical superset
relationship per-agent — the check above takes a couple minutes and turns
this from a guess into a verified-safe operation.

### Trivy image scan findings (informational)

141 Debian OS package findings (`affected`/`fix_deferred`/`will_not_fix`,
no upstream patch — same acceptance pattern as prior cycles) and the usual
Go stdlib CVE in the pinned `gogcli`/`goplaces` binaries (needs go1.26.5,
awaiting upstream binary release).

The Node.js app-dependency section flagged `tar` CRITICAL (CVE-2026-59873)
at two different installed versions — but neither is in `/app`: one is
bundled inside `npm`'s own vendored tooling
(`/usr/local/lib/node_modules/npm/node_modules/tar`, 7.5.13) and the other
inside corepack's pnpm (`/usr/local/share/corepack/.../pnpm/11.2.2/dist/node_modules/tar`,
7.5.15). `/app/node_modules/tar` itself correctly resolved to the patched
7.5.19 (upstream's own package.json bump). `@vitest/browser` CRITICAL is
present in `/app/node_modules` as a transitive dev/test dependency (not in
`dependencies` or `devDependencies` directly — pulled in via
`@copilotkit/aimock`), but it's a browser-mode test tool never imported by
the running gateway; not part of the live attack surface.

## v2026.7.1-2 upgrade notes (2026-08-22)

### Upstream release trains diverged: extended-stable (6.x) vs feature mainline (7.x/8.x)

`git tag -l 'v*' --sort=-v:refname | grep -v beta | head -1` still mechanically
picks the right tag, but this cycle showed why the sort alone isn't enough
evidence: `v2026.6.34` and `v2026.7.1-2` are on **completely divergent**
branches (`git merge-base --is-ancestor` fails both directions; common
ancestor is `v2026.4.19-beta.2`+29704 commits back), even though 6.34 has a
*later* publish date. Checking `v2026.6.34`'s own `CHANGELOG.md` explained
it: upstream now maintains a separate "extended-stable" maintenance branch
("this maintenance release carries targeted security and reliability
repairs without adding new release-line features") alongside the feature
mainline (`7.1` -> `7.1-1` -> `7.1-2` -> `7.2-beta.x` -> `8.1-beta.x`, which
`v2026.8.1-beta.2` is also NOT an ancestor-related to, i.e. a third
divergent line). `custom` has always tracked the feature mainline, so
`v2026.7.1-2` was correct — but if a future cycle finds the top non-beta tag
by version-sort is chronologically *older* than another non-beta tag,
check `CHANGELOG.md` at both tags before picking a target; don't assume
version-sort alone disambiguates.

### gogcli pinned-binary asset layout changed silently on a *pinned* version

Bumping `GOGCLI_VERSION` 0.34.1 -> 0.37.0 broke the build:
`tar: gog: Not found in archive`. The `steipete/gogcli` repo was
transferred to `openclaw/gogcli` as of v0.35.0 (old URLs still 301-redirect
and work fine); as part of that move, the release CI started wrapping the
binary in a `./` directory entry (`./gog`) instead of a flat `gog` member,
which broke our exact-name `tar xz -C /usr/local/bin gog` extraction. This
is exactly the "unpinned URLs break silently when upstream renames assets"
failure mode the top of this doc warns about, except it hit a *pinned*
version bump because the archive **layout**, not just the URL, changed.

**Fix (now baked into `Dockerfile.local`):** extract into a scratch dir and
`find -name gog -exec install ...` rather than naming the tar member
directly, so future layout changes from either project don't silently
break the pin again. Verified both `gog --version` and `goplaces --version`
run correctly in the built image.

### Strategy A stats

- Incoming commits: 9 (v2026.7.1 -> v2026.7.1-2)
- Our custom delta: 15 files, zero rebase conflicts (literal `git rebase`,
  not category-replay — small enough delta this cycle)

### Trivy image scan findings (informational)

134 Debian OS package findings (`affected`/`fix_deferred`/`will_not_fix`,
same acceptance pattern as prior cycles; 18 have a Debian-side fix
available but not yet pulled into upstream's base image layer — inherent
to how upstream's unmodified `Dockerfile` builds, not something to patch
in this fork). `gogcli`'s embedded Go stdlib is now **fully clean** (0
findings — the version bump this cycle happened to also clear its stdlib
CVE). `goplaces` still needs go1.26.6 (one patch behind, awaiting an
upstream binary release, same as last cycle).

Node-ecosystem findings again split cleanly between npm/pnpm's own
vendored tooling (`tar` 7.5.13/7.5.15, `undici` 6.25.0 — not `/app`) and
`/app`'s own pinned versions: `/app/node_modules/tar` (7.5.19, one HIGH
short of 7.5.21) and `/app/node_modules/undici` (8.5.0, one HIGH short of
8.9.0) are both upstream's own dependency pins, unchanged by this cycle's
9-commit delta. `@vitest/browser` CRITICAL is the same transitive
dev/test-only dependency noted last cycle (not part of the live attack
surface).
## v2026.9.3 upgrade notes (2026-09-08)

### v2026.9.3 matrix plugin-dependency hoisting gotcha

**Symptom:** All matrix accounts crash-loop right after deploy:

```
[matrix] [default] channel exited: Cannot find package 'matrix-js-sdk' imported from /app/dist/send-<hash>.mjs
```

(the exact `send-<hash>.mjs` filename is a content hash and changes on every
build; the same failure also shows up from other shared chunks —
`sdk-<hash>.mjs`, `crypto-runtime-<hash>.mjs`, `crypto-node.runtime-<hash>.mjs`,
`monitor-<hash>.mjs`, and the `music-metadata` format-parser chunks —
depending on which one loads first).

**Root cause:** v2026.9.3 code-splits several matrix-adjacent (and
music-metadata) code paths into shared bundle chunks that live directly
under top-level `/app/dist/`, not under `/app/dist/extensions/matrix/`.
Node's module resolution from one of those chunks walks straight to
`/app/node_modules/`, but the new `runtime-assets`-stage
plugin-dependency-linking step in `Dockerfile` only links a plugin's
dependencies under that *plugin's own* `dist/extensions/<id>/node_modules/`
— it never hoists them to top-level `/app/node_modules/`. So a package that
used to be reachable only from inside the plugin's own directory tree is no
longer reachable from a shared chunk that lives outside it. Confirmed via
`docker exec <gateway> bash -c 'ls /app/node_modules/matrix-js-sdk'` (absent)
vs. `ls /app/dist/extensions/matrix/node_modules/matrix-js-sdk` (present,
correctly symlinked into `node_modules/.pnpm/matrix-js-sdk@42.2.0_.../`).
Affects four packages total, all confirmed missing at top level but present
and correctly linked under `dist/extensions/matrix/node_modules/`:
`matrix-js-sdk`, `music-metadata`, `@matrix-org/matrix-sdk-crypto-nodejs`,
`@matrix-org/matrix-sdk-crypto-wasm` (`fake-indexeddb` and `markdown-it` are
also missing at top level but not referenced from any top-level chunk, so
they don't need this fix).

This is **not** the existing "v2026.4.15+ matrix self-reference gotcha"
above (that one is about the package literally named `openclaw`, fixed via
`ln -sf /app /app/node_modules/openclaw`, and is unrelated/unaffected here)
— it's a new gap in upstream's prod-dependency-pruning restructuring this
cycle (the `production-deps`/`runtime-build-output`/`runtime-assets` stage
split described in the Dockerfile changes section above).

**Fix:** `Dockerfile.local` now symlinks the four affected packages into
top-level `/app/node_modules/` through the already-correct plugin-local
links (not the pnpm virtual-store path directly, so this survives a future
rebuild even if the store's patch-hash suffix changes), right after the
`openclaw` self-reference symlink, with a `node -e "require.resolve(...)"`
build-time check so a future re-break fails the build instead of shipping
silently:

```dockerfile
RUN mkdir -p /app/node_modules/@matrix-org && \
    for pkg in matrix-js-sdk music-metadata @matrix-org/matrix-sdk-crypto-nodejs @matrix-org/matrix-sdk-crypto-wasm; do \
      ln -sf "/app/dist/extensions/matrix/node_modules/$pkg" "/app/node_modules/$pkg"; \
    done && \
    node -e "require.resolve('matrix-js-sdk'); require.resolve('music-metadata'); require.resolve('@matrix-org/matrix-sdk-crypto-nodejs'); require.resolve('@matrix-org/matrix-sdk-crypto-wasm')"
```

Verified post-fix: all 4 matrix accounts reach `running, connected,
health:healthy` (`openclaw channels status`), crypto bootstrap succeeds
(decrypting backlog events, including "Decrypted event on retry" for
messages that arrived before sync completed), zero `channel exited` /
`Cannot find package` log lines, and each account's "starting provider" log
line appears exactly once (no restart loop). The handful of
`DecryptionError: ... key backup is not working` lines seen alongside this
are pre-existing/expected (messages sent before a given device logged in,
unrelated to this fix — normal Matrix E2EE behavior, not a regression).

If a future cycle adds a new bundled extension with plugin-scoped
dependencies, check whether upstream's `dist/` code-splitting references
that extension's deps from a top-level chunk the same way
(`grep -rl "<package-name>" /app/dist/*.mjs`, excluding
`dist/extensions/<id>/`) before assuming the plugin-local link alone is
sufficient.

### Config schema drift required `openclaw doctor --fix` mid-upgrade

This cycle's ~2-month/24,853-commit delta (far bigger than prior weekly-ish
cycles) meant `openclaw.json` had accumulated enough deprecated/renamed keys
that the gateway refused to start at all (`Invalid config`: ~15 unrecognized
keys spanning `meta.lastTouchedAt`, `env.*` flattened keys, compaction
settings, `tools.media.audio.models`, `messages.tts`, `commands.ownerDisplay`,
several `gateway.*` keys, `plugins.bundledDiscovery`) plus a required state
DB schema migration (`audit-events-v2`). Both are explicitly what
`openclaw doctor --fix` is for (the gateway's own crash message names it).
Ran it via `docker exec openclaw-openclaw-cli-1 openclaw doctor --fix` —
first attempt raced the still-restarting gateway container for the state
lock and bailed early; stopped the gateway container first, reran cleanly,
then restarted the gateway. Confirmed the fix by checking `openclaw.json`
directly (`meta.lastTouchedAt` gone, `env` keys now under `env.vars`) rather
than trusting doctor's own "complete" banner alone.

**Pre-flight tip for next cycle:** if the incoming delta is on the order of
months/tens-of-thousands of commits rather than days/hundreds, expect
`openclaw doctor --fix` to be a required step, not just a fallback for the
already-documented gotchas below — factor it into the runbook up front
rather than discovering it at deploy time.

### Strategy B stats

- Incoming commits: 24,853 (v2026.7.1-2 → v2026.9.3, per `git rev-list --count`)
- `git merge-base custom v2026.9.3` again fell back to an ancient ancestor
  (upstream rewrote its own main history again between these releases,
  same phenomenon as v2026.6.11→v2026.7.1 and v2026.6.10→v2026.6.11) — used
  the known prior release tag `v2026.7.1-2` as `BASE` directly, per the
  existing caveat above.
- Confirmed `v2026.9.3` is on the feature mainline we track (not the
  "extended-stable" maintenance branch) by reading `CHANGELOG.md` content at
  v2026.8.1/8.2/9.1/9.2/9.3: each references the prior release's features
  and none says "maintenance release... without new release-line features."
- Our custom delta: 15 files (3 truly additive, 0 zero-upstream-churn,
  10 upstream-churned + 2 files whose sole customization is now obsolete —
  see below). Every touched file had upstream churn this cycle (no
  zero-churn files, unlike prior smaller-delta cycles).
- Conflicts and resolutions:
  - `GatewayConnectionController.swift`: upstream's TLS-fingerprint-probe
    refactor (enum-based `finish()`, new delegate method) already contains
    our exact fix (`completionHandler(.useCredential, credential)` instead
    of `.cancelAuthenticationChallenge`) natively — no port needed, just
    took upstream's version as-is.
  - `extensions/diagnostics-otel/src/service.ts`: upstream split this
    monolithic file into `service-metrics.ts`, `service-events.ts`,
    `service-recorders-tools.ts`, etc. Ported our `toolCallCounter`/
    `toolCallDurationHistogram` OTel instruments into `service-metrics.ts`
    and the `recordToolCall` handler + `tool.call` switch case into
    `service-recorders-tools.ts`/`service-events.ts` respectively, at the
    same logical anchor points (after `talkAudioBytesHistogram`; after
    `onToolStreamBoundary`/before `run.progress`→`diagnostic.heartbeat`).
    Note: upstream has also built out a much richer native
    `tool.execution.started/completed/error/blocked` instrumentation family
    (`toolExecutionDurationHistogram`, `openclaw.tool.execution` spans,
    etc.) that may now partially overlap with our simpler `tool.call`
    event — kept both per established precedent (v2026.6.11 notes: "kept
    our emitDiagnosticEvent block") rather than unilaterally judging ours
    redundant; worth a design review outside this upgrade.
  - `src/agents/embedded-agent-subscribe.handlers.tools.ts`: upstream moved
    `handleToolExecutionEnd` into a new sibling
    `embedded-agent-subscribe.handlers.tools.completion.ts` (the original
    file is now just a re-export). Ported our `tool.call` diagnostic-event
    emission to the new file at the identical anchor (after the
    tool-stream-boundary callback, before the `after_tool_call` hook).
  - `ui/src/components/app-sidebar.ts` + `ui/src/components/dashboard-header.ts`
    + the eyebrow/title rules in `ui/src/styles/layout.css`: **intentionally
    not ported, by design decision, not oversight.** Upstream's Control UI
    sidebar redesign replaced the old static logo/eyebrow/title brand block
    (`renderBrand()` in `app-sidebar.ts`) with a dynamic
    `<openclaw-sidebar-agent-card>` (now in the new `app-sidebar-render.ts`)
    that shows the active agent's *configured* display name/avatar — there
    is no static logo to hide and no static title slot to override anymore;
    the "maX" identity now surfaces automatically from agent config. Upstream
    also removed the mobile-viewport `dashboard-header.ts` breadcrumb
    entirely (`app-topbar.ts`'s narrow header is now a fixed OpenClaw/logo
    brand with no per-route breadcrumb), so `dashboard-header.ts`'s sole
    consumer is gone — deleted rather than resurrected. Verified via
    `git grep -n "sidebar-brand__title\|sidebar-brand__eyebrow"` at
    v2026.9.3: zero matches anywhere in `ui/src`, confirming this isn't a
    porting gap but a genuine upstream structural removal. If a future
    cycle wants "maX" branding restored to the sidebar, it should be done
    by setting the agent's display name to "maX" in config (if not already)
    rather than re-adding dead CSS/markup — check the memory/config first.
  - `ui/src/styles/base.css`: preserved the electric-blue accent theme
    through upstream's larger palette restructuring (new `--rail-header-*`,
    `--lobster-icon-*`, `--link`/`--link-hover` derivation tokens, and a
    WCAG-audit comment block describing the *red* palette's contrast
    ratios). Kept all of upstream's new structural tokens, substituted only
    the color values our patch always touched
    (`--border`/`--border-strong`/`--border-hover`/`--input`/`--ring`/
    `--accent`/`--accent-hover`/`--accent-muted`/`--accent-subtle`/
    `--accent-glow`/`--primary`), and dropped the WCAG comment block since
    its specific contrast numbers describe colors we no longer ship (no
    attempt made to recompute accurate ratios for blue — flag for a real
    audit if that matters). New: upstream introduced `--primary-hover`
    (previously `--primary` had no separate hover token); set it to
    `#2563eb` (a darker blue already used as this palette's light-theme
    accent) to preserve the existing "primary tracks accent" intent now
    that hover states are split — a judgment call, not something upstream
    specified, worth a design look.
- Verification loop (`git diff custom..custom-NEW`) flagged every touched
  file as "DIFFERS" from old `custom`, as expected for a cycle where every
  file also had upstream churn — reviewed each individually above rather
  than treating that loop's output as pass/fail.

### Dockerfile/docker-compose.yml changes (reviewed, no security concerns)

Upstream restructured the multi-stage build significantly this cycle:
plugin selection moved from inline shell/grep to a real Node script
(`scripts/lib/docker-plugin-selection.mjs`, with input validation on
`OPENCLAW_EXTENSIONS` ids); dependency install split into a dedicated
`production-deps` stage inherited by both `build` and the final
`runtime-assets` stage (avoiding a second full `pnpm prune` pass); the
runtime image now runs `apt-get dist-upgrade` for current Debian point-release
security fixes, adds `openssh-client` (sandbox backend spawns `ssh` directly)
and `libgomp1` (llama-server OpenMP dependency); npm's own bundled
dependency tree gets patched via `npm install --global npm@latest` +
`--ignore-scripts` update; the Docker GPG-key fetch now has explicit
`--connect-timeout`/`--max-time`; the healthcheck moved from an inline
`node -e fetch(...)` to a checked-in `dist/docker-healthcheck.js` (reviewed:
just probes the local gateway port via the existing lock-file/config
resolution, nothing new). All reviewed directly, no backdoors/obfuscation/
credential exposure/supply-chain concerns. Base image digests
(node:24-bookworm, node:24-bookworm-slim, bun 1.3.13→1.4.0) bumped to
current upstream digest pins.

### gogcli 0.37.0 → 0.39.1, goplaces 0.4.4 → 0.4.9

Release-noted as feature/reliability-only across both (Workspace feature
additions, pagination-loop hardening, HTTP timeout/retry bounding — see
commit for the per-release breakdown). goplaces 0.4.5–0.4.8 were confirmed
unpublished duplicates of 0.4.9 (release-automation bring-up tags per its own
changelog). Both extraction commands in `Dockerfile.local` already use the
robust `find -name <bin> -exec install` pattern from the v2026.7.1-2 fix, so
no further hardening needed for this bump.

### Security review (Step 2) — clean

- **gitleaks** on `custom..v2026.9.3` (24,558 commits scanned): 278,923 raw
  hits, 51 after the documented `.i18n`/`.test.` filter. All 51 manually
  reviewed: GitHub App OAuth **client IDs** (`Iv23li...`, meant to be
  public, not secrets) recorded in CI workflow files, doc examples with
  placeholder discord/API-key values, `test-support`/`test-helpers`/
  `test-utils`/QA-scenario fixture files with explicitly-fake tokens
  (`sk-1234567890abcdef`, `AKIAFAKE...`, `xoxb-intentionally...`), and a
  handful of string constants misidentified as secrets by regex
  (`notifications.web.v1`, `idempotency-1`, `ed25519PrivateKeyPemFromRaw` —
  a function name, not a key). No real leaked credential found. The
  false-positive surface is much wider than the doc's `.i18n`/`.test.`
  classes for a delta this size — OAuth client IDs and QA-scenario YAML in
  particular are worth adding to the standard filter for future cycles.
- **semgrep** `p/security-audit` against a `v2026.9.3` worktree, filtered to
  the 32,776 changed files matching the doc's extension list: 0 findings.
- **Dependency diff** (root `package.json`, BASE vs TARGET): 11 new direct
  deps (`@trycua/cua-driver`, `acorn`, `entities`, `execa`, `iconv-lite`,
  `koffi`, `ms`, `p-limit`, `p-map`, `pretty-ms`, `semver`), 32 new
  devDependencies (playwright, vite, postcss/stylelint tooling, `baileys`,
  `nostr-tools`, etc.) — all well-known, non-typosquat packages appropriate
  to features visible elsewhere in the changelog (computer-use, WhatsApp/
  Nostr channels, UI tooling). `glob`, `proper-lockfile`, `@grammyjs/types`
  removed.
  - `node --input-type=module -e 'await import("grammy")'` build-time
    sanity check confirms `grammy` (the actual runtime dep, not
    `@grammyjs/types`) still resolves post-restructuring.
- Commit-message security-track-record grep
  (`security|auth|cred|secret`, case-insensitive): 1,080 matches across the
  delta — all routine fix/refactor/hardening work (credential redaction,
  unauthorized-scrape rejection, OAuth flow fixes), nothing resembling a
  backdoor or an intentional auth bypass.

### Trivy image scan (informational)

Much lighter than prior cycles: Debian OS packages carry the usual
`affected`/`fix_deferred`/`will_not_fix` baseline (including the
long-standing `will_not_fix` zlib CRITICAL, unchanged from prior cycles —
not introduced by us), reduced further this cycle by the new
`apt-get dist-upgrade` step. `gogcli`/`goplaces` binaries: **zero**
HIGH/CRITICAL findings (embedded Go stdlib fully clean on both, continuing
last cycle's gogcli improvement and now goplaces catching up too). Node
app-dependency layer: one HIGH, `adm-zip@0.5.17` (fixed in 0.6.0) — a
transitive dependency, not declared directly in our `package.json` or any
bundled extension's; upstream-owned, not introduced by our delta.
`usr/bin/gh` (apt-installed): 2 HIGH, both the routine Go-stdlib-in-binary
class seen every cycle.

## v2026.10.5 upgrade notes (2026-10-19)

### Strategy B stats

- Incoming commits: 17,472 (v2026.9.3 → v2026.10.5). Upstream rewrote its
  main history again (`git merge-base --is-ancestor v2026.9.3 v2026.10.5`
  fails), so `BASE=v2026.9.3` was used directly, per the standing caveat.
- Release-line check: `v2026.9.33` sorts *below* `v2026.10.5` but was
  published between 10.3 and 10.4; its `CHANGELOG.md` lists only 9.x
  entries, 10.5's lists 10.5 after 9.9 and omits 9.33 → 9.33 is the
  extended-stable line, 10.5 is the feature mainline we track.
- Custom delta: 14 files (3 additive, 1 zero-churn, 10 upstream-churned).
  3-way apply: 2 clean, 6 conflicts, 1 intentionally dropped (see below).
- Pre-flight found an uncommitted Ollama sidecar tuning in
  `docker-compose.override.yml` (from 2026-09-12); committed it first so
  the tree was clean.

### Conflicts and resolutions

- `extensions/diagnostics-otel/src/service-{metrics,recorders-tools,events}.ts`:
  upstream restructured all three again. `createCounter`/`createHistogram`
  are now positional `(name, description, unit?, buckets?)` with an
  automatic `openclaw.` prefix (so our metric names are `"tool.calls"` /
  `"tool.duration_ms"` in source, same wire names as before); recorders are
  now methods on a returned object literal that read `runtime.*` instead
  of destructured consts; the event dispatcher is a `DiagnosticHandlers`
  map (optional keys, so adding `"tool.call"` to the payload union does not
  break exhaustiveness) using `recorders.recordX.bind(recorders)`. The
  3-way result was unusable; reset to upstream and re-ported by hand at
  the same anchors. Note the whole extension now bundles into a shared
  `dist/runtime-api-<hash>.mjs` chunk, so verify the port with
  `grep -c recordToolCall /app/dist/runtime-api-*.mjs`, not under
  `dist/extensions/diagnostics-otel/`.
- `src/infra/diagnostic-events.ts`: upstream replaced several `*Event`
  types with `(DiagnosticBaseEvent & *Fields)` intersections; kept
  upstream's union and re-added `DiagnosticToolCallEvent`.
- `embedded-agent-subscribe.handlers.tools.completion.ts`: import block
  conflict only; took upstream's imports + our `emitDiagnosticEvent`.
- `ui/src/styles/base.css`: took our blue values in all three hunks; also
  overrode the new claw-dark-family `:root[data-theme="dark"]
  --primary-hover: #c22e2e` → `#2563eb`. `--focus` is still the upstream
  red (`rgba(255,92,92,.2)` dark / `rgba(189,69,49,.15)` light) — it was
  never part of our patch; flagged, not changed.
- `apps/ios/Sources/Gateway/GatewayConnectionController.swift`: **dropped.**
  The delta we had been carrying since v2026.9.3 was a verbatim duplicate
  of `GatewayTLSFingerprintProbe`, which upstream ships in its own
  `GatewayTLSFingerprintProbe.swift` (two `private` classes with the same
  name in different files compile, which is why nobody noticed). Upstream's
  file taken as-is; the custom-file inventory is now 13 files.

### Security review (Step 2) — clean

- gitleaks on `custom..v2026.10.5` (17,420 commits): 94,824 raw hits, 115
  after the `.i18n`/`.test.` filter, 104 after also dropping
  `test-support|test-helpers|fixtures?/|/docs/|\.md$|/examples?/`. All 104
  reviewed: `src/logging/redact-*-fixture.json` placeholder tokens (the
  secret-redaction test corpus — worth adding `redact-.*-fixture` to the
  filter), 48-char sha hashes in `ci-test-native-bun-qualification.json`
  and `extensions/onnx/scripts/export-gliclass-instruct.py`, iOS unit-test
  strings, one public GitHub App client ID. No real credential.
- semgrep `p/security-audit`: 9 results in the whole tree, 5 in changed
  files, all benign (loopback fetch in an e2e fixture; a nonce+origin
  validated `postMessage` probe under a strict CSP; the sandboxed HTML
  preview that JSON-escapes `<`).
- Dockerfile/compose/scripts: digests re-pinned, no new apt packages,
  no new remote fetches. New deps (`@babel/*`, `chrome-devtools-mcp`,
  `import-meta-resolve`, `npm`, `esbuild`/`htmlparser2` moved to prod) are
  canonical packages tied to visible features. Auth surface only
  tightened (per-client auth-policy snapshots, operator-access gating,
  redacted-secret config rejection).
- **Behavioral change, mitigated:** the image `ENTRYPOINT` is now
  `tini -s -- node /app/docker-entrypoint.mjs`, which runs
  `openclaw doctor --fix --non-interactive` on every image swap before
  exec'ing the gateway. One of Doctor's repairs can npm/ClawHub-install a
  plugin that is configured but absent from the image. Our override now
  sets `OPENCLAW_UPDATE_DEFER_CONFIGURED_PLUGIN_INSTALL_REPAIR=1` and
  `OPENCLAW_NO_AUTO_UPDATE=1` (gateway/node-host self-update is off by
  default but belt-and-braces). Do NOT override the entrypoint: the
  boot-time Doctor is what performed this cycle's schema migrations
  (agent DB 19→25, shared state 16→20, Matrix account SQLite v13→v20,
  plaintext model credential → agent SQLite, implicit primary model →
  explicit `agents.defaults.model.primary`).

### gogcli 0.39.1 → 0.45.0, goplaces 0.4.9 → 0.4.13

Both carry security fixes this time: gogcli 0.41 retains patched gRPC
(GHSA-2v4p-qf9q-27wj), 0.44 built with patched Go 1.27.2; goplaces
0.4.10/0.4.11 refuse cross-origin API redirects and redact API keys from
errors, 0.4.12 Go stdlib fixes. Rest is feature work. `goplaces` now
publishes an `ASSET-INVENTORY.json` alongside the archives; the
`find -name` extraction is unaffected.

### Deploy gotcha 1: stale Matrix `crypto-idb-snapshot.json.lock` files

**Symptom:** all four Matrix accounts `error:Matrix startup aborted,
running, disconnected, health:error` right after deploy, Telegram fine,
gateway `healthy`. The abort reason is only in the gateway log *file*
(`/tmp/openclaw/openclaw-<date>.log` inside the container, not stdout):
`[IdbPersistence] Failed to restore IndexedDB snapshot from SQLite: Error:
file lock timeout for .../matrix/accounts/<acct>/<server>/<id>/crypto-idb-snapshot.json`.

**Cause:** four zero-byte `crypto-idb-snapshot.json.lock` files dated
2026-07-27 (the v2026.7.1 cycle) had been sitting next to the
`.migrated` snapshots. Older releases ignored them; 10.5's `file-lock`
implementation waits on them and times out.

**Fix:** with no process holding them, move them aside and restart:

```bash
sudo find /home/max/.openclaw/matrix/accounts -maxdepth 4 \
  -name crypto-idb-snapshot.json.lock -size 0 \
  -exec sh -c 'mv "$1" "$1.stale-$(date +%Y%m%d)"' _ {} \;
cd /home/max/Projects/openclaw && sudo -u max docker compose restart openclaw-gateway
```

All four accounts reached `running, connected, health:healthy` on the next
start. The `.lock.stale-20261019` copies are safe to delete.

Pre-flight tip: `find ~/.openclaw -name '*.lock' -size 0 -mtime +30`
before deploying.

### Deploy gotcha 2: `proxy_attribution_required` (HTTP 403) on every
gateway-authenticated route

**Symptom:** `https://max.shushu.be/` (and `/readyz`, hooks, Control UI)
return `403 {"error":{"type":"proxy_attribution_required"}}` while
channels are healthy. Gateway log: `observed unattributable proxy-shaped
traffic from 172.19.0.6`.

**Cause:** 10.5 fails closed when a request arrives from a
`gateway.trustedProxies` address and the `X-Forwarded-For` chain, after
stripping every trusted hop, leaves no client (`src/gateway/net.ts
resolveClientIp`; `X-Real-IP` is only honored with
`gateway.allowRealIpFallback=true`, which the security audit flags). Our
config trusted all of `172.17–19.0.0/16`, i.e. every container, and the
ingress is `shushu-edge-router` (TLS SNI passthrough, 172.19.0.3) →
`max-nginx` (172.19.0.6) → gateway. max-nginx's gateway `location /` only
set `X-Real-IP`; after adding `X-Forwarded-For` the chain was
`172.19.0.3` — itself trusted — so still unattributable.

**Fix (two files outside this repo + config):**

1. `~/Projects/max-nginx/nginx/nginx.conf`, gateway `location /`: added
   `proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;` and
   `X-Forwarded-Proto $scheme;` (the grafana/matrix locations already had
   them). Re-rendered the template in place and reloaded:
   `docker exec max-nginx sh -c '/docker-entrypoint.d/20-envsubst-on-templates.sh; nginx -t && nginx -s reload'`.
2. `~/Projects/max-nginx/docker-compose.yml`: pinned
   `networks.openclaw_default.ipv4_address: 172.19.0.6` (its existing
   address) so the next point survives container recreation.
3. `~/.openclaw/openclaw.json`: `gateway.trustedProxies` →
   `["172.19.0.6/32"]` (backup at
   `openclaw.json.bak-pre-trustedproxies-20261019`), then
   `docker compose restart openclaw-gateway`.

Now the edge router (172.19.0.3) is the attributed client for all external
traffic (the gateway's per-IP auth rate limit therefore sees one IP for
everything behind Tailscale — fine for a single-operator box), fritzbridge
(172.19.0.9) and host-local `127.0.0.1:18789` callers are ordinary
direct clients, and `max.shushu.be` serves the Control UI again. Real
client IPs would need PROXY protocol from the edge router, which would
also have to be enabled on `fritz-nginx` — not done.

**Keep in sync:** the `/32` in `openclaw.json` and the `ipv4_address` in
max-nginx's compose must match.

### Memory reindex

`Vector search: paused` on all four agents after deploy (reason this time:
`index chunking implementation changed (code: chunking_version)`).
`openclaw memory status --index --agent <a>` for main/jarvis/sensei/nemo
took 6 minutes total (vs. ~90 last time) thanks to the stored embedding
cache plus `OLLAMA_NUM_PARALLEL=2`; all four now `Dirty: no`, vector
search ready.

### Other post-deploy observations

- `openclaw-openclaw-cli-1` is `unhealthy` with "cannot verify Gateway
  ownership from this process (different PID namespace)". Same known
  limitation as before (see "doctor --fix needs gateway stopped"); adding
  `pid: "service:openclaw-gateway"` to the cli service in the override
  would likely fix it — not done this cycle.
- Doctor's remaining notices, all informational: 2 deferred historical
  transcript imports (`openclaw doctor --session-sqlite dry-run
  --session-sqlite-all-agents`), the bundled `github` plugin is not in
  `plugins.allow` (set `plugins.entries.github.enabled=false` to silence),
  and `openclaw update cleanup --dry-run` to retire the pre-migration
  `*.pre-startup-migration-*.bak` SQLite copies once the upgrade is
  trusted. Pre-upgrade tarball of the whole state dir:
  `~/.openclaw/backups/pre-upgrade-2026.10.5.tar.gz` (372 MB).
- 42 `Error decrypting event ... The sender's device has not sent us the
  keys` lines on Matrix start are all for June-2026 backlog events —
  pre-existing, same as prior cycles.
- Trivy: Debian baseline unchanged (127 HIGH / 5 CRITICAL, all
  `affected`/`fix_deferred`/`will_not_fix`); the 5 Node HIGHs are inside
  npm's own vendored tree (`/app/node_modules/.pnpm/npm@12.2.0/...` — npm
  is now a runtime dependency — and corepack's pnpm), not `/app`'s own
  pins (`/app` undici is 8.11.2, fixed); `usr/bin/gh` has the usual
  Go-stdlib class. gogcli/goplaces binaries: zero findings.
- Pushing this repo: `origin` is an HTTPS URL, so the SSH-key push pattern
  needs a URL rewrite:
  `sudo bash -c 'cd /home/max/Projects/openclaw && GIT_SSH_COMMAND="ssh -i /home/patrick/.ssh/id_ed25519 -o StrictHostKeyChecking=accept-new" git -c url."git@github.com:".insteadOf="https://github.com/" push origin custom --force-with-lease'`
  then `sudo find .git -user root -exec chown max:max {} +`.

## v2026.10.6 upgrade notes (2026-10-22)

### Strategy A stats

- Incoming commits: 294 (v2026.10.5 → v2026.10.6). For once upstream did
  NOT rewrite history: `git merge-base --is-ancestor v2026.10.5 v2026.10.6`
  passes, so a literal `git rebase v2026.10.6` of our 7 commits applied
  with zero conflicts. Custom delta unchanged at 13 files.
- Why this release matters: upstream now requires Node 24.22.1+ (the
  October 2026 Node.js security releases reject TLS chains signed with
  compromised certificate keys). The Dockerfile re-pins
  `node:24.22.1-bookworm{,-slim}` by digest; the 10.5 image ran 24.21.0.
  Verified `node --version` = v24.22.1 in the built image.

### gogcli 0.45.0 → 0.46.0, goplaces 0.4.13 → 0.4.14

Both are Go 1.27.3 rebuilds for GO-2026-6809 (crypto/x509 certificate
validation). gogcli also adds `gog auth rekey`, a `GOG_GOOGLE_API_BASE_URL`
relay option (refuses cross-origin redirects, sends no credentials) and
Gmail forward formatting. goplaces is security-only. `find -name` extraction
unaffected.

### Security review (Step 2) — clean

- gitleaks on `custom..v2026.10.6`: 181 raw hits, 0 after the standard
  filter (`.i18n|.test.|test-support|test-helpers|fixtures?/|/docs/|\.md$|
  /examples?/|redact-.*-fixture`).
- semgrep `p/security-audit`: the same 9 baseline results as last cycle,
  0 in changed files.
- `Dockerfile`: only the Node digest re-pin. `docker-entrypoint.mjs`: same
  boot-time `doctor --fix --non-interactive` flow, now passes an
  `OPENCLAW_IMAGE_ACTIVATION_CAPTURE_MARKER` env so Doctor skips its full
  pre-repair capture on plain container stop/start (still captures on a new
  image). Signal listeners are now removed after Doctor exits. No new remote
  fetches; `api.gmi-serving.com` is a new provider model definition.
- Root `package.json`: Node engine floor 24.22.1/26.12.1, pnpm 12.15.0,
  new `release:*` scripts, one new plugin-sdk export. No new runtime deps.
- Commit-message grep: 8 hits, all routine (OAuth refresh fence, MCP auth
  error text, scheduled follow-up authority).

### Deploy

- Boot-time Doctor ran cleanly; pre-repair capture retained at
  `~/.openclaw.update-captures/doctor-<id>/`. No schema migrations this
  time. Same informational notices as 10.5 (github plugin not in
  `plugins.allow`, 2 deferred historical transcript imports) plus a new
  "newest backup is more than 14 days old" nudge — the pre-upgrade tarball
  `~/.openclaw/backups/pre-upgrade-2026.10.6.tar.gz` (544 MB) covers it,
  but `openclaw backup enable --repository <dir>` would silence it.
- New `[state/agent-admission] Skipped openclaw-agent.sqlite: database is
  not a configured target` line at startup refers to
  `agents/openclaw/agent/openclaw-agent.sqlite` (688 KB, an agent id that
  is not configured — likely a stray from a past CLI run without
  `--agent`). Harmless; left in place.
- 11 plugins loaded; all 4 Matrix + 4 Telegram accounts `running,
  connected, health:healthy` on first start — no lock-file or
  proxy-attribution issues this cycle (both fixes from 10.5 held).
- `grep -c recordToolCall /app/dist/runtime-api-*.mjs` → 2 in one chunk
  (OTel port intact).
- Memory: `Dirty: yes` on all four agents after deploy; `memory status
  --index` for main/jarvis/sensei/nemo took 75 s total.
- Trivy: Debian baseline unchanged. Node findings: 4 inside npm's own
  vendored tree (`.pnpm/npm@12.2.0/...`: brace-expansion,
  http-cache-semantics, undici 6.x) and `markdown-it@15.0.2` (CVE-2026-111758,
  fixed in 15.0.3) which is upstream's pin — already at 15.0.2 in 10.5, not
  introduced this cycle.
- `openclaw-openclaw-cli-1` still `unhealthy` (PID-namespace ownership
  check), unchanged.
