# Herdr Cockpit V1 Implementation Plan (revision 2)

Revised 2026-10-10 after the Opus architecture review. Supersedes revision 1 (persisted index,
`POST /api/history/resume|fresh`, edits to `machine-api.ts`).

- **Spec:** `docs/cockpit/DESIGN.md` · **Review:** `docs/cockpit/OPUS_ARCHITECTURE_REVIEW.md` ·
  **Visual direction:** `docs/cockpit/STITCH_DIRECTION.md`
- **Status:** Architecture rev 2 APPROVED 2026-10-10, on condition that every offline cache
  answer carries explicit `freshness: "fresh" | "stale" | "unavailable"` and is never treated
  as current. Stitch O1–O8 approved (see `STITCH_DIRECTION.md`); O4 moves usage to a compact
  header readout with detail on demand (Batch 9). Batch 0 CLOSED 2026-10-11 (results in
  `CURRENT_STATE.md`, folded into B2–B5 and B10). Next: Batch 1. The upstream Mac bridge /
  `:7317` install must not be changed without new approval. Batches 0 and 10 contain external
  effects (Mac deployment) and need explicit owner approval at the time they run.

## Global rules for every worker

1. Read `AGENTS.md`, plus `server/AGENTS.md` / `src/AGENTS.md` / `shared/AGENTS.md` for the dirs
   you touch, `docs/cockpit/OPUS_ARCHITECTURE_REVIEW.md` §3–§8, and your batch card. Nothing else
   unless the card says so.
2. Touch only the files on your card. An "upstream file" patch must stay within the line budget on
   the card and carry a `// cockpit:` comment on its first line.
3. Never edit: `server/conversation.ts`, `server/codex.ts`, `server/claude-store.ts`,
   `server/opencode.ts`, `server/machine-api.ts`, `server/machines.ts`, `shared/protocol.ts`,
   `shared/herdr-api.generated.ts`, `Sidebar.tsx`, `AgentSidebar.tsx`, `Composer.tsx`.
   If you believe one must change, stop and report.
4. Provider stores are read-only: `new Database(path, { readonly: true, create: false })`; never
   write, never read `auth.json`/credentials; never return a filesystem path to the browser.
5. Errors: `{ error: { code, message } }` via `jsonResponse` (`server/http.ts`) with a local
   `fail(code, message, status)` like `server/machine-api.ts:6`, `badRequest`, or `errorResponse`.
6. Imports carry `.ts`/`.tsx`; relative paths; wire fields snake_case.
7. Tests: `bun:test`, no DOM. Unit file: `HERDR_TEST_MODE=unit bun test ./path/to/file.test.ts`
   (the `./` is required). Fixtures are built in `mkdtempSync` dirs (fake homes, fake SQLite,
   fake JSONL) — never the user's real `~/.claude`, `~/.codex*` or OpenCode DB.
   Every `createServer` in a test: temp `stateDir`, `port: 0`.
8. Before handing back: `bun run typecheck` and `bun run test:unit` must pass. If a contract
   test is part of the card, run the lane with `bun run check …` (its own herdr), or start the
   test herdr from zsh/bash (see `CURRENT_STATE.md` baseline note: a fish-started
   `herdr-web-ui-test` session fails `output.contract.test.ts` regardless of our code).
9. Hand back: changed files, test commands with real pass/fail counts, open questions. Never claim
   a test you did not run.
10. One worker at a time on any file. Batches marked ∥ may run in parallel with each other only.

## Batch overview

| # | Batch | Depends on | Tier | Codex | Sonnet | DeepSeek | Gate |
|---|---|---|---|---|---|---|---|
| 0 | Spike: resume flags, OpenCode project id, Codex2 env, deploy dry-run | approval | strong / owner | – | ok | – | owner + Opus |
| 1 | Shared contract + demo stubs | 0 | cheap | ✔ | ✔ | ✔ | Sonnet review |
| 2 | Repo identity module ∥3 | 1 | cheap | ✔ | ✔ | ✔ | test review |
| 3 | Codex + OpenCode listers ∥2 | 1 | cheap | ✔ | ✔ | ✔ | test review |
| 4 | Claude lister | 1 | mid | ok | ✔ | caution | Sonnet/Opus review |
| 5 | Local history service + HTTP handler + `index.ts` hook | 2,3,4 | mid | ok | ✔ | – | **Opus security review** |
| 6 | Remote forwarder, offline cache, project links | 5 | mid | – | ✔ | – | **Opus security review** |
| 7 | Client data layer (`src/lib/history.ts`) ∥8 | 1 (+5 for wiring) | cheap/mid | ✔ | ✔ | ✔ | Sonnet review |
| 8 | ChatView seam ∥7 | 1 | mid | – | ✔ | – | Opus conflict review |
| 9 | UI: ProjectSidebar, HistoricalSessionView, wiring, i18n, demo | 6,7,8 + Stitch approval | mid | – | ✔ | – | Opus + owner visual |
| 10 | Fork remote bundle + Mac deployment | 6 + owner approval | owner + mid | – | ✔ | – | owner |
| 11 | Real Linux + Mac acceptance | 9,10 | owner + mid | – | ✔ | – | owner |
| T1 | Telemetry follow-on (after V1) | 11 | mid | – | ✔ | – | Opus |

"Cheap" = Codex / DeepSeek / Haiku-class on a precise card. "Mid" = Sonnet-class.
"Strong" = Opus-class judgment.

---

## Batch 0 — Spike and environment gate (no product code)

**Objective.** Turn every UNKNOWN in the review into a recorded fact. Output is a short section
appended to `docs/cockpit/CURRENT_STATE.md`; no code.

Checks (read-only unless marked):

1. `claude --help`, `codex resume --help`, `opencode --help` on **both** PCs: exact resume syntax
   (`claude --resume <id>`, `codex resume <id>`, `opencode --session <id>` or `-s`), and whether
   each resumes in the cwd it is started in.
2. Codex2: confirm `~/.codex-pro-2` is the only extra home; confirm herdr `agent.start` cannot set
   `CODEX_HOME` (no env param in `shared/herdr-api.generated.ts`). Check whether a herdr agent
   profile/config can. Result decides `resumable` for non-default Codex homes.
3. OpenCode: for one git project, compare `project.id` with
   `git -C <worktree> rev-list --max-parents=0 HEAD` (metadata only).
4. Mac: Claude config dir, Codex homes, OpenCode DB location (`opencodeDatabasePath` rules), git
   availability, Bun/Node versions (bridge needs the bundle, not a dev checkout).
5. Deployment dry run (no install): `bun run build:remote darwin-arm64` on Linux succeeds; record
   bundle size and the manifest path. Decide with the owner: retire or disconnect the Mac from the
   upstream install on `:7317` (two connection servers must not both manage the Mac bridge).

**Files.** `docs/cockpit/CURRENT_STATE.md` only.
**Tests.** None (record command + output summary).
**Tier.** Opus or the owner with a Sonnet assistant; not suitable for cheap models (judgment + SSH).
**Gate.** Owner reads the facts; Opus updates review §8 / batch 5 if any flag differs.

## Batch 1 — Shared contract and demo stubs

**Objective.** Freeze the wire types so batches 2–9 can run against them.

**Files.**
- Create `shared/history.ts`: `HistoryProvider`, `HistoryRef`, `RepoIdentity`, `HistorySession`,
  `HistorySessionsResponse`, `ResumeCheck` exactly as in review §3, plus
  `historyRefParams(ref): URLSearchParams` and `parseHistoryRef(params): HistoryRef | null`
  (shape validation only: provider enum, `profile` `^[A-Za-z0-9._-]{1,64}$`, id per provider:
  Claude/Codex `^[0-9a-f-]{36}$`, OpenCode `^ses_[A-Za-z0-9]{8,64}$`).
- Create `shared/history.test.ts`.
- `site/demo/transport.ts`: fenced `// cockpit` block answering `GET /api/history/sessions`
  (empty list, `as_of` now), `GET /api/history/machines/*/sessions` (same), every other
  `/api/history/*` → `{ error: { code: "not_found" } }` 404.

**Prerequisite.** Batch 0 recorded (id shapes confirmed).
**Tests.** `HERDR_TEST_MODE=unit bun test ./shared/history.test.ts` (round-trip, rejects `..`, `/`,
oversize, unknown provider); `bun run typecheck` (covers `site/demo/transport.ts`).
**Tier.** Cheap. Codex ✔ Sonnet ✔ DeepSeek ✔.
**Gate.** Sonnet review of names against review §3.

## Batch 2 — Repo identity module (∥ Batch 3)

**Objective.** Compute a session cwd's identity on the machine that owns it.

**Files.** `server/history/identity.ts`, `server/history/identity.test.ts`.

Interface:
```ts
export function normalizeRemote(url: string): string | null;
// git@github.com:Owner/Repo.git, https://user@github.com/owner/repo, ssh://git@host:22/o/r.git
//  → "github.com/owner/repo": scheme, user, port and .git dropped; host lowercased;
//  path lowercased only for github.com / gitlab.com / bitbucket.org (case-insensitive hosts)
export async function probeRepo(cwd: string, opts?: { timeoutMs?: number; now?: () => number }): Promise<RepoIdentity | null>;
// git -C cwd rev-parse --show-toplevel; config --get remote.origin.url (else first remote);
// rev-list --max-parents=0 HEAD (first line). 2 s per call, Bun.spawn with array args, no shell.
// Cached per cwd, TTL 10 min; missing cwd → null (cached 1 min).
```
Batch 0: `root_commit` always comes from this git probe. OpenCode's `project.id` is a machine-local
id from `<repo>/.git/opencode` and never equals the root commit; do not use it for identity.
**Tests.** `HERDR_TEST_MODE=unit bun test ./server/history/identity.test.ts` — table of URL forms;
temp git repos built with `git init` + one commit (+ a worktree, + a repo without remote); missing
dir; non-git dir; cache hit does not spawn twice.
**Tier.** Cheap. Codex ✔ Sonnet ✔ DeepSeek ✔.
**Gate.** Reviewer checks no shell interpolation and the timeout.

## Batch 3 — Codex and OpenCode listers (∥ Batch 2)

**Objective.** List recent top-level interactive sessions from native SQLite indexes.

**Files.** `server/history/codex.ts`, `server/history/opencode.ts`, `server/history/profiles.ts`
(enumerate stores: Codex homes by the `server/usage.ts:355` rule — import or replicate the small
`siblingDirs` logic, do not edit `usage.ts`; OpenCode via `opencodeDatabasePath`; Claude via
`defaultClaudeConfigDir` / `CLAUDE_CONFIG_DIR`), tests beside each. A `.claude-*` sibling counts as
a Claude store only if it has a `projects/` dir (Batch 0: `~/.claude-server-commander` has none).

Rules:
- Codex: `SELECT id, cwd, title, first_user_message, created_at, updated_at, git_branch,
  git_origin_url, model FROM threads WHERE archived = 0 AND source IN ('cli','vscode') AND
  (thread_source = 'user' OR thread_source IS NULL) AND updated_at >= ? ORDER BY updated_at DESC LIMIT ?`. Check the column set with
  `PRAGMA table_info` first; a missing column → that home is skipped and counted.
  `profile` = home dir basename (`.codex`, `.codex-pro-2`). `resumable` = `native` for the
  default home, `fresh-only` for others (Batch 0: `agent.start` takes no env, so `CODEX_HOME`
  cannot be set). Batch 0: subagent rows also have `agent_role IS NULL` (their `source` is JSON
  `{"subagent":…}`, `thread_source` `subagent`/`guardian_review`); `exec` is non-interactive.
  `created_at`/`updated_at` are unix seconds.
- OpenCode 2.x (has `session_v2`): `session_v2` where `parent_id IS NULL AND time_archived IS
  NULL`, joined to `project` for `worktree`/`vcs`. Never `repo.root_commit` = `project.id`
  (Batch 0: 0/15 match); identity comes from Batch 2.
- OpenCode 1.x (no `session_v2`, e.g. the Mac's 1.18): metadata only from `session` (same filter;
  columns `directory`, `title`, `time_updated`). Rows carry `conversation: "unsupported"` (owner
  decision 2026-10-11): no legacy `message`/`part` reader, no OpenCode upgrade on the Mac.
  Choose the table by `sqlite_master`, not by version string.
- Titles: native title, else first user message; one line, ≤120 chars.
- In-memory cache keyed by `(dbPath, MAX(updated_at), mtime)`.

**Tests.** `HERDR_TEST_MODE=unit bun test ./server/history/codex.test.ts ./server/history/opencode.test.ts ./server/history/profiles.test.ts`
with SQLite fixtures created in temp dirs (copy the column list, not data, from the real schema);
cover archived, subagent (JSON `source` with NULL `agent_role`), `exec`, NULL `thread_source`
`vscode`, OpenCode 1.x vs 2.x schema, missing table, locked/corrupt file → skipped.
**Tier.** Cheap. Codex ✔ Sonnet ✔ DeepSeek ✔.
**Gate.** Reviewer checks readonly open and that no path leaves the module except internally.

## Batch 4 — Claude lister

**Objective.** List Claude sessions from `<configDir>/projects/*/<uuid>.jsonl` without a
full-file parse.

**Files.** `server/history/claude.ts`, `server/history/claude.test.ts`.

Rules: stores without `projects/` are skipped (Batch 0); only `^[0-9a-f-]{36}\.jsonl$` at depth 1
(skip `subagents/`, `memory/`, `*.meta.json`); `updated_at`
= mtime; read at most the first 64 KB, line by line, for `cwd`, `gitBranch`, a `summary` record
or the first human `user` text (skip tool results, meta, command wrappers); malformed lines are
skipped; files younger than the window only (`days`). Cache per file by `(mtime, size)`.
Do not reimplement the full transcript parser; this is a head scan only.

**Tests.** `HERDR_TEST_MODE=unit bun test ./server/history/claude.test.ts` with synthetic JSONL
fixtures (summary first, prompt first, tool-result first, truncated last line, 2 MB file with the
fields inside the first 64 KB, non-UUID file, subagent dir).
**Tier.** Mid (format judgment). Codex ok, Sonnet ✔, DeepSeek caution.
**Gate.** Sonnet or Opus review of the head-scan heuristics.

## Batch 5 — Local history service, HTTP handler, `index.ts` hook

**Objective.** Serve list / read / tool-output / image / resume-check for this machine.

**Files.**
- `server/history/list.ts`: merge providers; attach `repo` via `probeRepo` (Codex rows use
  `git_origin_url` first); attach `live_pane_id` by resolving only **live agent panes** with the
  existing resolvers (`opencodeSessionId`, `claudePaneSession`, `codexTranscriptPath`); sort by
  `updated_at`; cap with `truncated`.
- `server/history/read.ts`: `HistoryRef` → canonical path (Claude: `claudeTranscriptFile`;
  Codex: `threads.rollout_path` → `codexRolloutPath`; OpenCode: session exists in DB; OpenCode 1.x
  → 501 `unsupported` with that reason, never an empty transcript) →
  `transcriptPage` / `opencodeConversation`, `transcriptToolOutput` / `opencodeToolOutput`,
  `transcriptImage` / `opencodeImage`. Keep their ETag/`version` behaviour.
- `server/history/resume.ts`: `resumeCheck(ref)` → session exists, cwd exists on this machine,
  no live pane holds it (`live_elsewhere` + `live_pane_id`), profile supported; returns
  `{ kind, args, cwd }` with Batch 0's exact flags (`claude --resume <id>`, `codex resume <id>`
  + `-C <cwd>`, `opencode <cwd> -s <id>`). Read-only. Batch 0: OpenCode 2.x `-s` **creates** a
  session when the id is unknown, so the id must be confirmed in the local DB in the same check;
  any doubt → `session_missing`, never a resume.
- `server/history/http.ts`: `handleHistoryRequest(req, url, deps): Promise<Response | null>`
  for `GET /api/history/{sessions,conversation,conversation/tool-output,conversation/image,resume-check}`;
  non-GET → 405; unknown `/api/history/*` → `null` so the existing 404 answers.
- `server/index.ts` (≤ 4 lines, `// cockpit:`): import; add `url.pathname.startsWith("/api/history/")`
  to the `bridgePath` expression (`server/index.ts:1429`); call the handler **after** the machines
  handler and `/ws`, **before** the `/api/*` handlers' 404, respecting the existing auth gate.
- `server/api.contract.test.ts` is upstream: put new endpoint contract tests in
  `server/history/api.contract.test.ts` instead.

**Tests.**
- Unit: `HERDR_TEST_MODE=unit bun test ./server/history/list.test.ts ./server/history/read.test.ts ./server/history/resume.test.ts`
  (fake homes; traversal attempts `../`, absolute paths, other-profile ids → 404; resolver
  injection for live panes).
- Contract: `server/history/api.contract.test.ts` via the integration lane — real `createServer`
  with temp `stateDir`, `port: 0`, fake provider homes through env/options; verifies auth gate,
  error envelope, `bridgePath` acceptance of a bridge bearer.
- `bun run typecheck`, `bun run test:unit`.
**Tier.** Mid. Sonnet ✔, Codex ok with review, DeepSeek –.
**Gate.** **Opus security review** (path resolution, auth, route order) before merge.

## Batch 6 — Remote forwarder, offline cache, project links

**Objective.** Make every PC's history reachable from the connection server and keep the last
listing visible while a PC is offline.

**Files.**
- `server/history/remote.ts`: `GET /api/history/machines/:id/<sub>` where `<sub>` ∈ the five
  local routes (explicit allowlist). `machines.endpoint(id)` → `fetch(endpoint.url + "/api/history/" + sub + search, { headers: { authorization: "Bearer " + token, if-none-match } })`,
  timeout 20 s (list) / 75 s (read), no cookies, no other headers forwarded. Same behaviour as
  `server/machine-api.ts:95-130`, but a separate module.
  Offline/timeout: `sessions` → cached answer with `freshness: "stale"` + `as_of` (no cache → empty list,
  `freshness: "unavailable"`); every other sub → 503 `machine_offline`.
- `server/history/cache.ts`: `<stateDir>/history-cache/<machine-id>.json`, dir 0700, file 0600,
  temp + rename; written after each successful remote `sessions` answer; metadata only; removed
  when the PC is removed (listen to the manager or prune unknown ids on read).
- `server/history/links.ts` + `GET/POST /api/history/links`: explicit project links in
  `<stateDir>/project-links.json`; POST requires same-origin + `x-herdr-machine: 1`; validated,
  bounded (≤200 links, ≤8 keys each).
- `server/history/http.ts`: dispatch the above (no new `index.ts` lines; the local machine id is
  `local` → served locally).
- `site/demo/transport.ts`: links GET → `{ links: [] }`.

**Tests.** `HERDR_TEST_MODE=unit bun test ./server/history/remote.test.ts ./server/history/cache.test.ts ./server/history/links.test.ts`
with a stub machine manager and a stub bridge `Bun.serve({ port: 0 })`: forwards bearer only;
rejects unknown subpaths and `//`; offline with/without cache; `freshness` is never `fresh` for cached data; file modes; atomic
rename; CSRF header required on POST.
**Tier.** Mid. Sonnet ✔.
**Gate.** **Opus security review**.

## Batch 7 — Client data layer (∥ Batch 8)

**Objective.** All non-visual client logic, testable without DOM.

**Files.** `src/lib/history.ts`, `src/lib/history.test.ts`.
- `fetchMachineHistory(machineId)`: local → `/api/history/sessions`, remote →
  `/api/history/machines/:id/sessions`.
- `groupProjects(answers: HistorySessionsResponse[], links, liveAgents): ProjectGroup[]` —
  merge rule from review §7: same normalized remote → one group; same root commit only →
  `suggested_link`; explicit link → merged; else machine-scoped by toplevel/cwd. Never by
  basename. Live rows from the existing roster replace their historical twin via `live_pane_id`.
  Sorted by most recent activity; stale answers flagged per row.
- `useHistorySelection()` hook (selected `{ machine_id, ref }` or null), storage key via
  `paneStorageId`-style helper so it is per-PC.
- `resumeOrFresh(machine, session)`: calls `resume-check`, then the existing
  `workspace/create` / `tab/create` through `useMachineApi()`; never retries automatically.

**Tests.** `HERDR_TEST_MODE=unit bun test ./src/lib/history.test.ts` — merge table (same remote on
Mac+Linux, fork with same root commit, same basename different repos, worktree, stale PC, live
twin dedupe, explicit link).
**Tier.** Cheap/mid. Codex ✔ Sonnet ✔ DeepSeek ✔ (pure functions).
**Gate.** Sonnet review of the merge table against review §7.

## Batch 8 — ChatView seam (∥ Batch 7)

**Objective.** Render historical turns with the exact live renderer.

**Files.** `src/components/ChatView.tsx` (≤ 20 changed lines): export `Turn`; add
`ConversationAssetsContext` with `toolOutputUrl(ref)` / `imageUrl(ref)`; default value reproduces
today's `pane/conversation/...?pane_id=` URLs (`ChatView.tsx:254,264,397`) so live behaviour is
byte-identical.
**Tests.** `bun run typecheck`; `bun run build && bun scripts/ui-regression.ts` (live chat unchanged).
**Tier.** Mid. Sonnet ✔.
**Gate.** Opus conflict review (diff size, default path unchanged).

## Batch 9 — UI (blocked on Stitch approval)

**Objective.** Implement the three approved states with existing tokens.

**Files.**
- New: `src/components/ProjectSidebar.tsx/.css`, `src/components/HistoricalSessionView.tsx/.css`.
- `src/components/MachineSidebar.tsx` (≤ 15 lines): `Projects | PCs` switch (owner decision O1).
- `src/lib/settings.ts`: `sidebarMode` default + sanitizer.
- `src/App.tsx` (≤ 10 lines): render `HistoricalSessionView` in place of the pane canvas when a
  history selection is active.
- `src/lib/i18n.{ko,ja,zh}.ts`: one fenced block each for every new `t("…")`.
- Colocated CSS: tokens only, no color literals, no `!important`; endless animations use
  `steps()` or `var(--ease-pulse)`. Icons: lucide only. Wording: "New workspace".
- If a new token is unavoidable: add it to `src/styles.css` **and** `DESIGN.md` (root) together.

**Tests.** `HERDR_TEST_MODE=unit bun test ./src/lib/i18n.test.ts ./src/motion.test.ts`;
`bun run test:ui`; new Playwright script `scripts/history-regression.ts` against the demo/test
session (selection, read-only view, resume button disabled offline) — served from `dist/`, build
first; screenshots only from `herdr-web-ui-demo` or a test session.
**Tier.** Mid. Sonnet ✔.
**Gate.** Opus code review + owner visual sign-off against `docs/cockpit/stitch/*.png`.

## Batch 10 — Fork remote bundle and Mac deployment (external effect)

**Objective.** Run the fork's server code on the Mac bridge.

Steps (owner present):
1. Fork namespace (owner decision 2026-10-11; prepared in code before this batch, inert until
   deployed): the cockpit never reuses or overwrites upstream's `remote-v25`, manifest or bridge
   registry. Own bundle base dir (e.g. `~/.local/share/herdr-cockpit/remote-v<N>`, today
   `server/remote-host.ts:23`), own registry dir (not `~/.config/herdr-web-ui/bridges/`), own
   numeric version line (`newerBundle` accepts only `^\d+$`), own manifest (local file or fork
   release; not `devswha` URLs, `server/remote-bundle.ts:20`), and the `bundle-older` probe must
   not match upstream dirs. Contract-test that an upstream `remote-v25` present on the host is
   neither reported ready nor touched.
2. `bun run build:remote darwin-arm64`; place the manifest at
   `remote-bundles/manifest-darwin-arm64.json` (or `HERDR_WEB_BUNDLE_MANIFEST`).
3. With separate namespaces both bridges can coexist on the Mac; still, only one connection
   server should drive a given Mac pane. Any change to the upstream `:7317` setup needs new owner
   approval.
4. Update the bridge through the existing UI flow. Record the result in the agent handoff as
   `outcome_unknown` until step 5 passes.
5. Verify: `/api/history/machines/<mac>/sessions` answers from the Mac.

**Tests.** The verification call; `bun run check fast` on the build that produced the bundle.
**Tier.** Owner + Sonnet. Not for unattended agents.
**Gate.** Owner.

## Batch 11 — Real acceptance

Checklist on the real PCs (no screenshots of the live session leave the machine):
- Projeto EAD (or any repo present on both) shows one project with Mac and Linux sessions.
- Two different repos with the same folder name stay separate.
- Opening a historical Claude, Codex and OpenCode session reads natively, pages back, opens a
  tool output and an image.
- Resume on the original PC starts the agent in a new workspace; a live session offers
  "Go to live pane" instead.
- Codex2 behaves as Batch 0 decided.
- Mac asleep: rows stale, conversation unavailable, actions disabled, Linux fully usable; Mac
  wakes → rows refresh without reload.
- `bun run check full` green on the merged branch (from zsh/bash).

**Tier.** Owner + Sonnet. **Gate.** Owner sign-off.

## T1 — Telemetry follow-on (after V1, plan only)

See review §9. Files would be `server/history/remote.ts` (forward `/api/usage`), `server/index.ts`
`bridgePath` (+1 condition), `src/lib/usage-merge.ts` + test (merge by `ProviderUsage.key`).
Visibility only.
