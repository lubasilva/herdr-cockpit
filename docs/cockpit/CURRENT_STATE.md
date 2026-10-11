# Herdr Cockpit — Current State

Date: 2026-10-10
Branch: `planning/cockpit-v1`

## Repository

- Fork: `lubasilva/herdr-cockpit`
- Upstream: `devswha/herdr-web-ui`
- Linux checkout: `/home/lucas/Projects/herdr-cockpit`
- Herdr workspace: `Herdr Cockpit`
- `origin`: fork
- `upstream`: original project

## Existing installed Web UI

The upstream `devswha/herdr-web-ui` plugin is installed separately in the user Herdr and is currently serving on:

`http://127.0.0.1:7317`

The fork checkout is not replacing that installed plugin yet.

## Machines

The existing upstream Remote PCs flow has been used successfully.

Current Web UI machine roster includes:

- Linux local machine — connected
- MacBook Pro — connected through SSH alias `macbook`

Topology:

`Browser → Linux connection server → SSH → Mac remote bridge → Mac Herdr`

No custom transport was added.

## Existing subscription usage telemetry

`GET /api/usage` on the Linux Web UI is returning real usage windows for signed-in Claude, Codex and OpenCode accounts.

This proves provider telemetry already exists locally.

Open product gap: whether/how to aggregate or route the existing usage mechanism across connected remote PCs without double-counting the same account.

Do not reimplement provider usage clients until this is audited.

## Versions observed on Linux

- Herdr: `0.9.0`
- Bun: `1.4.3`
- Node: `v26.10.0`

The upstream repository currently documents Herdr `0.9.0+`, while parts of current remote-PC documentation/tests mention behavior verified against newer Herdr versions. Treat version compatibility as a baseline investigation item rather than silently upgrading the user runtime.

## Baseline verification

Re-verified 2026-10-10 (Opus review) on upstream `5792d71` + docs commits, no cockpit code:

- dependency install: PASS
- `bun run typecheck`: PASS
- `bun run test:unit`: **2459 pass, 8 skip, 0 fail** (185 files)
- `server/output.contract.test.ts` alone, 3 runs: **3/7 failing every run** (`observer high
  watermark`, `pause at high watermark`, `the retried attach streams the pane`)
- **Root cause: environment, not upstream code and not cockpit.** The reused isolated test herdr
  (`herdr-web-ui-test`) had been started from a fish shell, so its panes ran fish. The tests type
  POSIX syntax into panes (`echo attach-$((6*7))`), which fish rejects. `scripts/test-herdr.ts`
  only neutralises zsh startup files (`ZDOTDIR`). The `node not found` lines in the log are the
  test's deliberate `breakPath()` stub.
- After `herdr --session herdr-web-ui-test server stop` and a re-run with `SHELL=/usr/bin/zsh`:
  **7/7 pass**.
- Not run in this pass: full `bun test`, `bun run build`, `bun run test:ui`, `bun run check full`.

**Rule for every worker and acceptance gate:** this PC's login shell is fish. Run contract and
integration lanes through `bun run check …` (own herdr) or from zsh/bash with `SHELL=/usr/bin/zsh`,
after stopping any leftover `herdr-web-ui-test` session. A red `output.contract` under a
fish-started test herdr is not a regression.

## Planning artifacts

- `docs/cockpit/DESIGN.md` (revision 2)
- `docs/cockpit/IMPLEMENTATION_PLAN.md` (revision 2, batch cards)
- `docs/cockpit/OPUS_ARCHITECTURE_REVIEW.md`
- `docs/cockpit/STITCH_DIRECTION.md` + `docs/cockpit/stitch/` (3 screens, awaiting approval)
- `docs/cockpit/OPUS_REVIEW_PROMPT.md`

No V1 implementation task has started yet. See the checkpoint at the end.

## Batch 0 — spike results (2026-10-10, read-only)

Method: local CLI `--help`, `sqlite3 "file:…?mode=ro"` (schema, counts, ids only — no message
content read), `ssh -o BatchMode=yes macbook` read-only commands, local bundle build into a
scratch dir. Nothing was installed, restarted or written on the Mac; `:7317` untouched.

### Versions

| | Linux | Mac (`macbook`, Darwin arm64, macOS 27.0.1, zsh) |
|---|---|---|
| herdr | 0.9.0 | 0.9.0 (installed upstream bundle `remote-v25` ships herdr 0.9.3) |
| Claude Code | 2.1.296 | 2.1.296 |
| Codex CLI | 0.162.1 | 0.161.0 |
| OpenCode | **2.0.23** | **1.18.35** |
| Bun / Node | 1.4.3 / v26.10.0 | not on PATH / v24.7.0 (bundle carries bun 1.4.2, node v22.23.2) |
| git, sqlite3 | yes | yes (Apple Git 2.54.0) |

### 1. Resume syntax

- Claude: `claude --resume <uuid>` (`-r`); `--fork-session` exists. Not exercised (would create
  state); cwd scoping assumed per store layout `projects/<cwd-slug>/<uuid>.jsonl` → start in the
  session's cwd.
- Codex (both PCs): `codex resume [SESSION_ID] [PROMPT]`; `-C/--cd <DIR>` sets the working root,
  so resume can pass the cwd explicitly. `-p` layers `$CODEX_HOME/<name>.config.toml`; it does
  **not** select a home.
- OpenCode Linux 2.x: `opencode [directory] -s <id>` — help says "continue, **or create if it does
  not exist**": a wrong/foreign id silently creates a new session → resume-check must verify the id
  exists in the local DB first. Mac 1.x: `opencode [project] -s <id>` (no create clause).

### 2. Codex homes

- Both PCs: `~/.codex` and `~/.codex-pro-2` only (Mac also has `~/.codexbar`, not a Codex home;
  the `.codex-` prefix rule excludes it). No `~/.config/codex`, no `CODEX_HOME` set.
- Linux `codex2` is a fish function that sets `CODEX_HOME=~/.codex-pro-2`.
- herdr 0.9.0 `agent.start` = `{name, kind, pane_id, args?, timeout_ms?}` — no env; `--kind codex`
  runs the canonical executable. No herdr config hook found. **Decision: `.codex-pro-2` sessions
  are `resumable: "fresh-only"` in V1** (as planned).
- Schema (`state_5.sqlite`, both PCs): `threads` has every planned column, `created_at`/`updated_at`
  are unix **seconds** (`*_ms` columns also exist); ids are 36-char UUID(v7).
- **Plan correction:** subagent threads have `agent_role IS NULL` too. Their `source` is a JSON
  string `{"subagent":…}` and `thread_source` is `subagent`/`guardian_review`. Interactive filter
  must be `source IN ('cli','vscode') AND (thread_source = 'user' OR thread_source IS NULL)`
  (Mac has 13 old `vscode` rows with NULL `thread_source`). `exec` = non-interactive, excluded.

### 3. OpenCode project identity

- `project.id` **never** equals `rev-list --max-parents=0 HEAD` (0/15 repos on Linux). It comes
  from the machine-local `<repo>/.git/opencode` file. **Decision: never use `project.id` as
  `root_commit`; Batch 2 `probeRepo` computes it from git.**

### 4. Store schemas per PC (blocking divergence)

- Linux OpenCode 2.x: `session_v2` (+ legacy `session`), messages in `session_message`
  (what upstream `server/opencode.ts` reads). Session ids `ses_` + 26 chars (30 total).
- Mac OpenCode 1.x: **no `session_v2`**; sessions in `session` (906 children / 284 top-level),
  messages in `message`/`part`; `session_message` exists but is empty. The upstream conversation
  reader cannot read Mac OpenCode history. **V1 options (owner):** (a) list Mac OpenCode from
  `session` (columns `parent_id`, `time_archived`, `directory`, `title` present) and mark
  conversation `unsupported`; (b) upgrade OpenCode on the Mac (not authorized); (c) write a 1.x
  reader (scope growth). Recommendation: (a).
- Claude: default `~/.claude` on both; `CLAUDE_CONFIG_DIR` unset. `~/.claude-server-commander`
  matches the `.claude-` sibling rule but has no `projects/` → lister must require `projects/`.
  Linux: 43 UUID transcripts at depth 2, 0 non-UUID; `memory/` dirs present at depth 2 (skip).
  Mac: 5 project dirs.

### 5. Deployment dry run

- `bun run build`: PASS. `HERDR_BUNDLE_OUTPUT=<scratch> bun run build:remote darwin-arm64`: PASS —
  `herdr-web-ui-darwin-arm64.tgz` 104,346,474 B, sha256 `f0c9dbae…186a0ccc`, manifest
  `manifest-darwin-arm64.json` `{version:"25"}`. Downloads are pinned + checksummed; PTY not
  executed (cross-platform).
- Mac today: `~/.local/share/herdr-web-ui/remote-v25/` (+ `remote-v25-81f0020c2f771355`),
  `bundle.json` `{version:"25", herdr:"0.9.3"}`; `remote-entry.ts` running from that bundle; bridge
  registry in `~/.config/herdr-web-ui/bridges/`.
- **Blocking finding:** the fork reuses `REMOTE_BUNDLE_VERSION = "25"` and the same paths
  (`server/remote-host.ts:23` `BUNDLE_DIR`, the `bridges/` registry) and the manifest URL points at
  `devswha/herdr-web-ui` releases (`server/remote-bundle.ts:20`). As-is the fork would see
  `bundle-ready`, keep running upstream code on the Mac (no `/api/history`), or overwrite the
  upstream install if forced. Batch 10 must give the fork its own namespace (bundle dir, registry
  dir, version, local manifest) **or** the owner retires the upstream `:7317` Mac connection first.
  No action taken.

### Baseline before Batch 1 (2026-10-10, HEAD c2586b1, code identical to upstream `5792d71`)

| Run | Result |
|---|---|
| `bun run check full`, system herdr **0.9.0**, `SHELL=zsh` | fast lane PASS (actionlint, generated types, typecheck, build, unit **2459 pass / 8 skip / 0 fail**); integration **385/5 fail**; browser FAIL (`ui-regression.ts`: "Delete anyway" dialog never shown) |
| `bun run check integration browser`, CI's pinned herdr **0.9.3** on a scratch `PATH` (system herdr untouched), `SHELL=zsh` | browser **PASS** (all scripts); integration **388/2 fail** (both in `reattach.contract.test.ts`) |
| `check run … reattach.contract.test.ts`, herdr 0.9.3, `SHELL=zsh`, ×2 | 2/2 fail both times (deterministic) |
| same, `SHELL=/bin/bash` | **4/4 PASS** |

Diagnosis (environment, not code):
- herdr 0.9.0 vs 0.9.3: 0.9.0 answers `invalid_request` instead of `pane_not_found`, has no
  `pane clear` route answer, and reports worktree removal as `worktree_remove_failed` instead of
  `dirty_worktree_requires_force` (which is also the browser "Delete anyway" failure). CI pins
  0.9.3; the README floor says 0.9.0+.
- `reattach.contract.test.ts` spawns its own herdr for the live handoff without the empty
  `ZDOTDIR` that `scripts/test-herdr.ts` sets, so a zsh pane reads `~/.zshrc`.

**Gate rule (extends the fish rule above):** acceptance runs use herdr 0.9.3 on `PATH` (scratch
copy, checksum `18a8dc65…4dba7` as in `ci.yml`) and `SHELL=/bin/bash`. The red results above under
herdr 0.9.0 or zsh are not regressions. Mac and Linux user herdr stay at 0.9.0 (no upgrade done).

## Checkpoint (2026-10-11)

HEAD: see `git log -1` on `planning/cockpit-v1` (this checkpoint's commit; parent `d002a57`).

Approved decisions:
- Architecture rev 2, on condition that every offline/cached answer carries explicit
  `freshness: fresh | stale | unavailable` and is never treated as current.
- Stitch O1–O8 (see `STITCH_DIRECTION.md`).
- Mac OpenCode 1.x: list metadata from `session`; transcript/conversation `unsupported`. No legacy
  reader, no OpenCode upgrade on the Mac.
- Remote bundle: fork gets its own bundle dir, registry, version line and manifest; never reuses
  or overwrites upstream `remote-v25`. No change on the Mac before Batch 10 and new approval.
- Batch 0 closed; findings folded into plan cards B2–B5 and B10.

Baseline gate: herdr **0.9.3** first on `PATH` (scratch copy, sha256 as in `ci.yml`) and
`SHELL=/bin/bash`; `bun run check full` (or `fast` + `integration browser`). Red results under
herdr 0.9.0 or a fish/zsh pane shell are environment, not regressions.

Next batch: **Batch 1** (shared contract + demo stubs), not started.

Real blockers: none for Batch 1. Before Batch 10: the fork bundle namespace has to be written
(code, inert until deployed) and the owner approves the Mac deployment. Open, non-blocking:
local herdr is 0.9.0 vs CI's 0.9.3; `reattach.contract` needs bash in this environment.
