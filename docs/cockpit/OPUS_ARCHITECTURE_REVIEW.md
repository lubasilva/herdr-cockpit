# Herdr Cockpit — Opus Architecture Review

Date: 2026-10-10
Branch: `planning/cockpit-v1` (base `5792d71`, upstream `devswha/herdr-web-ui` main)
Scope: planning/design only. No V1 batch has been implemented.

**Verdict: PLAN NEEDS MATERIAL REVISION.** The product intent and the hard constraints hold.
The data architecture, the remote deployment story and several file/abstraction choices do not
match the upstream code. The revised plan is smaller than the original.

Evidence labels used below: **VERIFIED IN CODE** (file:line read in this pass), **VERIFIED ON
MACHINE** (read-only inspection of this Linux PC's stores and schema, no transcript content),
**CORRECT BUT NEEDS ADJUSTMENT**, **REDUNDANT**, **WRONG**, **UNKNOWN** (must be checked in
Batch 0 before code depends on it).

---

## 1. Verified upstream capabilities

| Capability | Where | Status |
|---|---|---|
| Path-based transcript page reader, independent of a pane: `transcriptPage(source, path, page, codexHome)` | `server/conversation.ts:1307` (exported) | VERIFIED IN CODE |
| Same for tool output / images by path: `transcriptToolOutput`, `transcriptImage` | `server/conversation.ts:1414,1458` (exported) | VERIFIED IN CODE |
| OpenCode reader by `(dbPath, sessionId, page)`: `opencodeConversation`, `opencodeToolOutput`, `opencodeImage` | `server/opencode.ts:458,564,575` | VERIFIED IN CODE |
| Claude session-id → transcript lookup inside the store (fast path by encoded cwd, then all projects) | `claudeTranscriptFile` `server/claude-store.ts:278` | VERIFIED IN CODE |
| Codex rollout path validation (canonical path inside `<home>/sessions`, rejects subagent rollouts) | `codexRolloutPath` `server/codex.ts:521` | VERIFIED IN CODE |
| Codex keeps a **native SQLite thread index** (`<CODEX_HOME>/state_5.sqlite`, table `threads`) with `id, rollout_path, cwd, title, name, first_user_message, created_at, updated_at, git_branch, git_origin_url, git_sha, model, reasoning_effort, source, archived, agent_role, thread_source` | upstream already queries it (`server/codex.ts:1257,1350`); schema read on this PC | VERIFIED IN CODE + ON MACHINE (355 threads in `~/.codex`, 119 in `~/.codex-pro-2`) |
| OpenCode keeps a native index: `session_v2(id, project_id, parent_id, directory, title, model, agent, time_created, time_updated, time_archived …)` and `project(id, worktree, vcs, name)`; git projects have a 40-hex `project.id` | opened read-only by upstream | VERIFIED ON MACHINE (193 sessions, 16 git projects) |
| Second Codex account discovery rule: `CODEX_HOME`, `~/.config/codex`, `~/.codex`, every `~/.codex-*` sibling | `server/usage.ts:355` | VERIFIED IN CODE (`~/.codex-pro-2` is the user's `codex2`, per `~/.config/fish/functions/codex2.fish`) |
| Live pane → native session resolution (Claude via process/session, Codex via threads + process, OpenCode via herdr `agent_session`) | `resolveTranscript`, `claudePaneSession`, `codexTranscriptPath`, `opencodeSessionId` | VERIFIED IN CODE |
| Start an agent with **arbitrary args** in a new workspace/tab: `POST /api/workspace/create` / `/api/tab/create` `{ cwd, agent: { kind, args } }` | `server/index.ts:161-164,250` (`agentLaunch`) | VERIFIED IN CODE — already allowlisted for remote PCs |
| Remote PC proxy, explicit allowlist, bridge bearer only, `machine_offline` 503 when disconnected | `server/machine-api.ts:7,95-97` | VERIFIED IN CODE |
| Bridge-token authorization is a **separate path list** on the remote server (`bridgePath`) | `server/index.ts:1429` | VERIFIED IN CODE — any new remote-callable endpoint must be added here as well |
| Last roster snapshot per PC persisted; disconnected PC keeps it with controls disabled | `<stateDir>/machines.json`, `docs/remote-pcs.md` | VERIFIED IN CODE/DOCS |
| Subscription usage, keyed per provider **account** (`ProviderUsage.key` = provider + account id), with `checked_at` and `problem` | `shared/protocol.ts:266-288`, `server/usage.ts` | VERIFIED IN CODE |
| `/api/usage` is **not** reachable for remote PCs (absent from `MACHINE_PROXY_PATH` and `bridgePath`) | `server/machine-api.ts:7`, `server/index.ts:1429` | VERIFIED IN CODE |

## 2. Assumptions we got wrong

| # | Original assumption | Classification | Evidence and correction |
|---|---|---|---|
| A1 | A persisted Unified Historical Session Index (`server/history/store.ts`, replace/dedupe per machine) is required | **WRONG for local, ADJUST for remote** | Codex and OpenCode already keep native indexes; Claude is one file per session. Each machine can answer "list sessions" from native stores on request (mtime-keyed in-memory cache). The only thing that needs persisting is the **last answer from a remote PC**, so its rows stay visible while it is offline. That is a disposable cache of a remote response, not an index with its own dedupe semantics. |
| A2 | New history routes just need `server/machine-api.ts` allowlisting | **WRONG / incomplete** | (a) The Mac runs whatever server code is in its **remote bundle**, which by default is downloaded from **upstream's** `remote-v25` release (`server/remote-bundle.ts`, `docs/remote-pcs.md`). Fork routes do not exist on the Mac until the fork ships its own bundle. (b) Bridge currency is decided only by `bundle_version === "25"` (`server/machines.ts:435-460,545`), so an upstream v25 bridge already on the Mac is considered current and is reused. (c) The bridge token is only accepted on `bridgePath` (`server/index.ts:1429`). (d) The upstream install at `127.0.0.1:7317` is also connected to the same Mac; two connection servers with different bundles will fight over the bridge (auto bridge-update is on by default). |
| A3 | `POST /api/history/resume` and `/api/history/fresh` server endpoints | **REDUNDANT** | `workspace/create` / `tab/create` with `agent: { kind, args }` already exist, are remote-allowlisted, and start the agent in a new pane. The cockpit needs only a read-only **resume check** that returns validated args; the client then calls the existing endpoint. |
| A4 | `server/history/read.ts` needs refactors in `conversation.ts` | **REDUNDANT** | The path-based readers are already exported. The history layer resolves an opaque `{provider, profile, native_id}` to a validated path with existing helpers and calls `transcriptPage` / `opencodeConversation`. No change to `conversation.ts`. |
| A5 | Register routes inside `server/index.ts` | **ADJUST** | Upstream convention: handler modules return a `Response` or `null` to fall through. One handler module `server/history/http.ts`; `index.ts` gets one import, one hook line and the `bridgePath` addition. |
| A6 | Project identity via "Git repository identity + Herdr workspace root" | **CORRECT BUT NEEDS ADJUSTMENT** | Herdr workspaces carry no repository identity. Signals that actually exist: Codex rows carry `git_origin_url`; OpenCode git projects carry a 40-hex id (believed to be the repo's root commit; UNKNOWN until checked in Batch 0); any session's `cwd` can be probed with `git` **on its own machine** (remote URL, top level, root commit). Herdr workspace root adds nothing beyond `cwd`. |
| A7 | `AgentSidebar.tsx` is where project navigation goes | **ADJUST** | The sidebar is `MachineSidebar` (PC → workspace via `Sidebar`) with `AgentSidebar` (the Agents list) inside it. The lowest-conflict seam is a `Projects / PCs` mode switch in `MachineSidebar`, rendering a new `ProjectSidebar` and leaving `Sidebar`/`AgentSidebar` untouched. |
| A8 | `ChatView` can be reused as is for history | **ADJUST** | `ChatView` is pane-bound: it polls `pane/conversation` and builds `pane/conversation/{tool-output,image}?pane_id=` URLs (`ChatView.tsx:254,264,397`). Minimal seam: a small context that supplies those asset URLs plus exporting the memoised `Turn` renderer, so `HistoricalSessionView` renders turns identically. |
| A9 | Codex vs Codex2 = separate runtimes | **ADJUST** | Codex2 is `codex` with `CODEX_HOME=~/.codex-pro-2`. Model it as runtime `codex` + `profile` (the home's directory name), discovered by the same rule `usage.ts` uses. Badge text "Codex2" is a display alias. |
| A10 | Codex2 native resume works like Codex | **UNKNOWN** | herdr `agent.start` takes `kind, args, name, pane_id` and **no environment** (`shared/herdr-api.generated.ts:57`). A `codex resume <id>` started this way uses the server's default home and will not find a `.codex-pro-2` thread. V1: Codex2 is `fresh-only` plus "copy resume command", unless Batch 0 finds a supported way (e.g. a herdr agent profile). |
| A11 | Plan steps `bun test server/history/store.test.ts` | **WRONG** | Repo rules: unit tests run with `HERDR_TEST_MODE=unit` and need the `./` prefix. |
| A12 | Plan omits i18n, demo transport and route order | **WRONG (omission)** | Every new `t("…")` needs ko/ja/zh entries (`i18n.test.ts` fails otherwise); new endpoints need an answer in `site/demo/transport.ts`; route order in `createServer().fetch` is fixed. |
| A13 | Plan spec path `docs/superpowers/specs/…` | **WRONG** | The spec is `docs/cockpit/DESIGN.md`. |
| A14 | `output.contract.test.ts` failures may be flaky upstream timing | **WRONG** | Environment issue, see §11. |
| A15 | AGY historical history can stay unsupported | VERIFIED as acceptable | No stable store was investigated or needed; it stays live-only. |

## 3. Final recommended V1 architecture

```
Browser (React)
  ├─ ProjectSidebar  ── groups sessions of every PC into logical projects (pure client function)
  ├─ HistoricalSessionView ── reuses ChatView's Turn renderer, read-only
  └─ existing PaneTerminal / ChatView / Composer for live panes (unchanged)

Connection server (Linux, the fork)                       Mac bridge (fork bundle)
  server/history/http.ts                                    server/history/http.ts (same code)
   GET /api/history/sessions            ── local stores     GET /api/history/sessions
   GET /api/history/conversation[...]   ── local stores     GET /api/history/conversation[...]
   GET /api/history/resume-check        ── local stores     GET /api/history/resume-check
   GET /api/history/machines/:id/*  ──► bridge (endpoint + bearer from MachineManager)
       sessions answer cached per PC in <stateDir>/history-cache/<id>.json (metadata only)
       offline → cached rows with freshness:"stale" + as_of; conversation → 503 machine_offline
  existing /api/workspace/create, /api/tab/create (+ /api/machines/:id/…) start resume/fresh
```

Principles:

1. **No cockpit index.** Every machine lists its own native stores on request. Results are cached
   in memory per store, keyed by file `mtime/size` (Claude) or `MAX(updated_at)` (SQLite).
2. **The only persistence** is the connection server's last answer per remote PC (stale rows for
   screen 3), plus the user's explicit project links. Both are 0600 temp+rename files in `stateDir`.
3. **References, not paths.** The browser only ever holds `{machine_id, provider, profile, native_id}`.
   The owning machine re-resolves the path inside its canonical store on every read.
4. **Remote history goes through a fork-owned route** (`/api/history/machines/:id/*`) that uses
   `MachineManager.endpoint(id)` exactly as `machine-api.ts` does (bearer only, no cookies, no
   arbitrary URLs). `MACHINE_PROXY_PATH` is not edited.
5. **Resume = existing agent start.** The check endpoint validates and returns `{kind, args, cwd}`;
   the client calls `workspace/create` or `tab/create` on the same PC.
6. **Grouping is a pure client function** over every PC's answer, testable with `bun:test`.

### Wire contract (new file `shared/history.ts`)

```ts
type HistoryProvider = "claude" | "codex" | "opencode";
interface HistoryRef { provider: HistoryProvider; profile: string; native_id: string }   // profile: ".claude", ".codex", ".codex-pro-2", "opencode"
interface RepoIdentity {
  remote: string | null;        // normalized "host/owner/repo", lowercase host, no scheme/user/.git
  root_commit: string | null;   // first root commit sha when known
  toplevel: string | null;      // absolute path on that machine
}
interface HistorySession {
  ref: HistoryRef;
  title: string;                // native title, else first prompt; trimmed to 120 chars, one line
  cwd: string | null;
  repo: RepoIdentity | null;
  branch: string | null;
  model: string | null;
  started_at: string | null;    // ISO
  updated_at: string;           // ISO
  live_pane_id: string | null;  // set when a live pane on this PC is reading this session
  resumable: "native" | "fresh-only";
}
interface HistorySessionsResponse { machine_id: string; sessions: HistorySession[]; truncated: boolean; as_of: string;
  freshness: "fresh" | "stale" | "unavailable" } // required; cached data is never "fresh"
interface ResumeCheck { ok: true; kind: string; args: string[]; cwd: string } | { ok: false; reason: "cwd_missing" | "session_missing" | "live_elsewhere" | "unsupported_profile" | "unsupported_provider"; live_pane_id?: string }
```

Endpoints (each machine): `GET /api/history/sessions?days=90&limit=500`,
`GET /api/history/conversation?provider&profile&id[&before&since&from]` → existing
`ConversationResponse` shape, `GET /api/history/conversation/tool-output?…&ref`,
`GET /api/history/conversation/image?…&ref`, `GET /api/history/resume-check?…`.

## 4. Precise extension seams/files

New (fork-only, no conflict risk):

| File | Purpose |
|---|---|
| `shared/history.ts` | wire types above |
| `server/history/identity.ts` | `normalizeRemote(url)`, `probeRepo(cwd)` (git on this machine, 2 s timeout, cached per cwd) |
| `server/history/codex.ts` | list threads from every Codex home (`state_5.sqlite`, read-only, `archived = 0`, `agent_role IS NULL`, interactive `source` only) |
| `server/history/opencode.ts` | list `session_v2` rows with `parent_id IS NULL`, join `project` |
| `server/history/claude.ts` | list `<configDir>/projects/*/<uuid>.jsonl`; read a bounded head (≤64 KB) for `cwd`, `gitBranch`, first human prompt/summary; `updated_at` from mtime |
| `server/history/list.ts` | merge providers, attach `live_pane_id` for live agent panes using existing resolvers, sort, cap |
| `server/history/read.ts` | `HistoryRef` → validated path → `transcriptPage` / `opencodeConversation` / tool output / image |
| `server/history/resume.ts` | resume check (§8) |
| `server/history/remote.ts` | `/api/history/machines/:id/*` forwarder + offline cache |
| `server/history/links.ts` | explicit project links `<stateDir>/project-links.json` |
| `server/history/http.ts` | one handler: returns `Response` or `null` |
| `src/lib/history.ts` (+ `.test.ts`) | fetchers + `groupProjects(answers, links)` + live/historical merge |
| `src/components/ProjectSidebar.tsx/.css` | project → sessions tree |
| `src/components/HistoricalSessionView.tsx/.css` | read-only transcript + action bar |

Upstream files touched (small, documented patches):

| File | Patch |
|---|---|
| `server/index.ts` | import; `/api/history/` in `bridgePath`; one `handleHistoryRequest(...)` hook after the machines handler and before the `/api/*` 404 |
| `src/components/ChatView.tsx` | export `Turn`; read tool-output/image URL builders from a `ConversationAssets` context (default = current pane behaviour) |
| `src/components/MachineSidebar.tsx` | `Projects / PCs` segmented switch; render `ProjectSidebar` in Projects mode |
| `src/App.tsx` | one selection state `historySelection`; when set, render `HistoricalSessionView` where the pane canvas renders |
| `src/lib/i18n.{ko,ja,zh}.ts` | appended block of new strings |
| `src/lib/settings.ts` | `sidebarMode: "projects" | "pcs"` default + sanitizer |
| `site/demo/transport.ts` | answers for the new endpoints (empty lists) |
| `shared/machines.ts` | fork bundle version (Batch 9 only) |

Not touched: `server/conversation.ts`, `server/codex.ts`, `server/claude-store.ts`,
`server/opencode.ts`, `server/machine-api.ts`, `server/machines.ts`, `Sidebar.tsx`,
`AgentSidebar.tsx`, `Composer.tsx`, `PaneTerminal.tsx`, `shared/protocol.ts`.

## 5. Upstream Divergence Risk

| Patch | Upstream churn | Conflict risk | Mitigation |
|---|---|---|---|
| `server/index.ts` hook + `bridgePath` | very high (2987 lines, every release) | **Medium**: 3 lines in a long boolean expression and the route chain | keep it to 3 lines with a `// cockpit:` marker; re-check route order at every sync |
| `src/App.tsx` selection | high | **Medium-high** | all state lives in a `useHistorySelection()` hook in `src/lib/history.ts`; App gets ~6 lines |
| `src/components/ChatView.tsx` seam | high | **Medium** | ~15 lines; generally useful, candidate to upstream as a PR |
| `MachineSidebar.tsx` switch | medium | **Low-medium** | switch is one element + one conditional |
| i18n appends | every release | **Low** (end-of-file conflicts) | one fenced `// cockpit` block per file |
| `site/demo/transport.ts` | medium | Low | fenced block |
| `shared/machines.ts` bundle version | each `remote-vN` | **Certain but trivial** | rule in DESIGN §8: fork version = upstream N + suffix only in the manifest metadata, re-applied at each sync |
| Fork remote bundle build/publish | each upstream bundle bump | **Operational**, not textual | Batch 9 script; one connection server per Mac |
| Stitch visual direction | `src/styles.css`/`DESIGN.md` churn | **Low if** the direction stays on existing tokens (it does: §STITCH_DIRECTION) | new components only; no global token rewrite |

Upstream tests are untouched; fork tests live beside fork modules.

## 6. Historical-data safety and privacy model

1. Read-only everywhere: SQLite opened `{ readonly: true, create: false }`, files opened for read; no writes to any provider store.
2. The browser never supplies a path. `native_id` is validated by shape (Claude UUID, Codex thread id `[0-9a-f-]{36}`, OpenCode `ses_[A-Za-z0-9]+`), then resolved inside the canonical store with existing validators (`codexRolloutPath`, Claude store lookup, OpenCode session existence); anything else → 404.
3. Profiles are enumerated by the server (`~/.claude` or `CLAUDE_CONFIG_DIR`; Codex homes by the `usage.ts` rule; OpenCode by `opencodeDatabasePath`). `profile` in a request must equal one of them.
4. Listing returns metadata only; transcript bodies are paged through the existing reader with its existing 16 MB window.
5. Subagent and non-interactive sessions are excluded (Codex `source`/`agent_role`/`thread_source`, OpenCode `parent_id`, Claude non-UUID files and `subagents/`).
6. The offline cache stores the listing only (titles ≤120 chars). It is deletable, 0600, and drawn as stale with its `as_of`. **O3 approved 2026-10-10:** titles may be cached, shown with stale/last-seen. Every cached answer carries explicit `freshness` (`stale`, or `unavailable` when there is no cache) and is never treated as current.
7. Auth: same gates as every `/api/*` route; remote calls carry only the bridge bearer. Credentials and `auth.json` files are never read by history code.
8. Malformed rows/files are skipped and counted (`skipped` in the answer), never fatal.

## 7. Mac/Linux project identity strategy

Computed on the owning machine, merged in the browser:

1. **Remote URL** (normalized `host/owner/repo`): from Codex `git_origin_url`, else `git -C <cwd> config --get remote.origin.url` (fallback: first remote). Same normalized remote on two PCs → **one project**.
2. **Root commit**: from `git rev-list --max-parents=0 HEAD` (or OpenCode `project.id`, once Batch 0 confirms it is the root commit). Same root commit **without** a remote match → shown as a **suggested link**, never merged automatically (templates and forks share roots).
3. **Explicit link**: the user links two project groups; stored in `<stateDir>/project-links.json` on the connection server (`{ links: [{ keys: [...], label }] }`).
4. **Fallback**: `machine_id + toplevel || cwd` — a machine-scoped project. Basename never merges.

Worktrees resolve to the same remote, so they group under the project with their branch as the
badge. A `cwd` that no longer exists keeps the identity recorded natively (Codex) or falls back
to rule 4. Probes are cached per `cwd` with a 10-minute TTL and a 2 s timeout per git call.

## 8. Resume strategy

| Selected row | Primary action | Mechanics |
|---|---|---|
| Live pane (`live_pane_id` set, PC online) | open the live pane (chat/terminal), normal Send | unchanged upstream path |
| Historical, PC online, `resume-check` ok | **Resume on <PC>** | `POST /api/[machines/:id/]workspace/create { cwd, label, agent: { kind, args } }` with `claude --resume <id>`, `codex resume <id>`, `opencode --session <id>` (exact flags confirmed in Batch 0) |
| Historical, check fails (`cwd_missing`, `session_missing`, `unsupported_profile`) | **Start fresh continuation** | same endpoint, `agent: { kind }` without args, `cwd` = the project's checkout on that PC; if the cwd is missing, the user picks a checkout (existing directory picker). No transcript, no summary injected |
| Historical, `live_elsewhere` | **Go to live pane** | never start a second agent on the same native session |
| Historical, PC offline | both disabled; explicit "Start in <project> on <other PC>…" opens the New workspace dialog prefilled | never automatic |

Rules: no hidden machine/provider/model switch; resume always on the session's own PC and cwd;
resume check is read-only and re-run just before the start call; failure leaves the history
untouched and shows the herdr error.

## 9. Deliberately deferred

- Multi-PC usage/quota (follow-on below), task-resume packets, cross-session search, AGY history,
  mobile layout of the new surfaces, persisting anything beyond the offline cache and links,
  upstreaming the ChatView seam (after V1 proves it).

### Telemetry follow-on (plan only)

1. Add `/api/usage` to `bridgePath` and let `server/history/remote.ts`'s forwarder (or a sibling
   `usage-remote.ts`) fetch `/api/usage` from each connected PC.
2. Client merges reports: rows with the same `ProviderUsage.key` (provider + account id) from two
   PCs are **one account**; show the newest `checked_at`, list the PCs where it is signed in.
   Rows keyed by location (no account id) stay per PC.
3. Provenance: provider-reported windows = `exact` with `checked_at`; `problem` or no data =
   `unknown`; nothing in the current code produces `estimated`, and V-next adds none.
4. Mac keychain reads over SSH may answer `locked` — shown as `unknown`, never retried in a loop.
5. Visibility only; no routing, failover or kills.

## 10. Implementation order for worker agents

See `IMPLEMENTATION_PLAN.md` for the full batch cards. Order:

0. **Spike & environment gate** (no product code) — strong model or owner-assisted
1. Shared contract + demo transport stubs — cheap
2. Repo identity module — cheap
3. Codex + OpenCode listers — cheap
4. Claude lister — mid
5. History list/read/resume-check + HTTP handler + index.ts hook — mid, security review
6. Remote forwarder + offline cache + project links — mid, security review
7. Client data layer (`groupProjects`, fetchers, selection hook) — cheap/mid
8. ChatView seam — mid, conflict review
9. UI: ProjectSidebar, HistoricalSessionView, App/MachineSidebar wiring, i18n — mid
10. Fork remote bundle + Mac deployment — owner-approved external effect
11. Real Linux + Mac acceptance — owner + mid model

## 11. Baseline verification

| Check | Result |
|---|---|
| `git status` / remotes | clean; `origin` = `lubasilva/herdr-cockpit`, `upstream` = `devswha/herdr-web-ui`; HEAD on upstream `5792d71` + 4 docs commits |
| `bun run typecheck` | PASS |
| `bun run test:unit` | **2459 pass, 8 skip, 0 fail** (185 files) |
| `server/output.contract.test.ts` alone, 3 runs | **3/7 fail on every run** (`observer high watermark`, `pause at high watermark`, `the retried attach streams the pane`) — deterministic, not load |
| Root cause | the reused isolated test herdr (`herdr-web-ui-test`, pid 37598) had been started from a **fish** environment, so its panes ran fish. The tests type POSIX syntax into the pane (`echo attach-$((6*7))`; fish rejects `$((…))`). `scripts/test-herdr.ts` neutralises only zsh (`ZDOTDIR`). The `node not found` lines in the log come from the test's own deliberate `breakPath()` stub. |
| Confirmation | stopped only the isolated test session and re-ran with `SHELL=/usr/bin/zsh`: **7/7 pass** |
| Acceptance gate | run contract/integration lanes via `bun run check …` (own herdr) or with `SHELL=/usr/bin/zsh`; stop a leftover test session first. Not an upstream bug; not a cockpit regression. Optional upstream issue: harness should pin the pane shell. |

Full `bun test`, `bun run build`, `bun run test:ui` were not run in this pass.
