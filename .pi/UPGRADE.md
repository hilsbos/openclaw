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
| `Dockerfile.local`            | Runtime overlay: UID/GID remap, socat, gh, gogcli, goplaces, matrix openclaw symlink |
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
