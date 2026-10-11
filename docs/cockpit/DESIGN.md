# Herdr Cockpit — Unified Mac + Linux Session History

Date: 2026-10-10
Status: Revision 2 (2026-10-10, Opus architecture review). Awaiting owner approval; no V1 code exists.

> **Revision 2 summary.** §6 (no persisted cockpit index — native stores are the index; only a
> disposable offline cache of remote listings), §7 (concrete identity signals), §8 (fork-owned
> remote history route, fork remote bundle, single connection server), §9 (Codex2 = profile),
> §10 (resume through the existing workspace/tab create endpoints; Codex2 resume unknown),
> §12.4 (Stitch result) and §15 (phases) changed. Evidence and the full assumption audit:
> `docs/cockpit/OPUS_ARCHITECTURE_REVIEW.md`. Batches: `docs/cockpit/IMPLEMENTATION_PLAN.md`.

## 1. Purpose

Build a personal developer cockpit on top of the existing `herdr-web-ui` rather than replacing Herdr.

The cockpit must make Mac + Linux feel like one development environment while preserving Herdr as the runtime and the native stores of Claude/Codex/OpenCode/etc. as the source of truth for conversations.

The product should solve a daily operational problem even if it produces zero token savings:

- find historical sessions quickly;
- inspect prior conversations by project rather than by opaque session ID;
- use Mac and Linux from one interface;
- send prompts and continue live work from the same browser surface;
- start fresh sessions without losing project identity;
- reduce the need to jump between terminal tabs just to recover old work.

The cockpit is not a replacement for Git, Herdr, provider-native session stores, or the terminal.

---

## 2. Product principle

> Keep Herdr as the engine; make the browser the unified cockpit.

Secondary principles:

1. **One logical workspace, multiple physical machines.** Linux and Mac are execution locations, not separate products.
2. **Project first, session second, machine third.** The primary navigation hierarchy is project → sessions. Machine/runtime are metadata badges.
3. **Reference, do not duplicate.** List sessions from provider-native stores on request and read message history from them when the user opens a session. The cockpit stores no index of its own (§6).
4. **Live Herdr and historical provider state are different concerns.** Herdr owns currently running panes/agents. History comes from native stores, so it survives after panes disappear.
5. **Terminal remains first-class.** The cockpit should make routine work easier without preventing direct terminal use.
6. **No universal control plane in V1.** No scheduler, daemon, automatic provider failover, universal model router or LLM supervisor is justified by this project.

---

## 3. Existing upstream capabilities we will reuse

The implementation should begin from `devswha/herdr-web-ui`, not from a blank application.

Verified upstream capabilities include:

- Herdr workspaces/tabs/panes in the browser;
- chat and live terminal views;
- prompt input into running agents;
- agent status, approvals and questions;
- worktree/session creation flows;
- native transcript readers for supported agents;
- PWA/mobile support;
- remote PC support over SSH;
- machine-aware workspace/pane routing;
- provider subscription-usage UI on the server machine;
- localhost-first deployment and explicit remote-access controls.

Remote PCs are already modeled by upstream as:

```text
Browser
  → connection server
  → SSH local forward
  → remote loopback bridge
  → remote Herdr socket
```

Therefore **Mac remote support is not a custom feature**. Our first step after forking is to configure and verify the existing Remote PCs flow against the user's Mac.

---

## 4. Scope

### 4.1 V1 core

V1 consists of four tightly related capabilities:

1. **Remote Mac connection using upstream Remote PCs**
2. **Unified historical session list** across Linux and Mac (computed from native stores, §6)
3. **Project-centric historical session browser**
4. **Open historical conversation + decide between native resume and fresh continuation**

The existing upstream prompt/terminal/live-agent behavior remains intact.

### 4.2 Explicitly deferred

The following are not V1 requirements:

- automatic context rotation;
- scheduler or long-running orchestration daemon;
- automatic model routing;
- automatic provider failover;
- cross-provider semantic handoff;
- autonomous recovery from abrupt failure;
- global transcript search across full raw tool output;
- analytics dashboards;
- cost optimization engine;
- pstack orchestration;
- replication of Git/worktree logic already owned by Herdr;
- replacing provider-native session stores with our own transcript database.

These may become future phases only if measured pain justifies them.

---

## 5. User experience

### 5.1 Primary navigation

The left sidebar is project-centric.

Example:

```text
Projects

▼ AVA
  ● Current · Claude · Mac
  ○ 21:14 · Claude · Mac
  ○ 18:20 · Codex · Linux
  ○ Yesterday · Codex2 · Linux
  ○ Oct 08 · Claude · Linux

▶ CotaPay
▶ Transporte
▶ Echo
▶ Fabrica
```

The user should not need to know a native session UUID to locate work.

Each session row may show:

- relative or absolute timestamp;
- runtime/model/profile when available;
- machine badge;
- live/historical status;
- branch/worktree badge when known;
- optional short session title.

### 5.2 Main conversation area

Opening a live session continues to use upstream chat/terminal behavior.

Opening a historical session shows:

- human messages;
- assistant messages;
- important structured events when already supported by the native adapter;
- compact metadata header;
- native session identity in a secondary/details view, not as the primary label.

Raw terminal/tool output should be folded by default rather than dominate the conversation.

### 5.3 Composer behavior

For a live resumable session:

```text
[ Type prompt… ] [Send]
```

For a historical session with a valid native resume mechanism:

```text
[Resume session]
```

The UI must state the destination machine/runtime before resuming.

For a historical session that cannot safely be resumed:

```text
[Start fresh continuation]
```

Fresh continuation should preserve project identity and may later use a compact recovery packet, but V1 must not copy the complete old transcript into the new session.

### 5.4 Machine model

The UI should make machines visible but secondary.

Examples:

- `Mac · Claude · Opus`
- `Linux · Codex2`

The user should be able to filter by machine when needed, but normal navigation stays project-first.

---

## 6. Historical session discovery (revised: no persisted index)

### 6.1 Native stores are the index

Revision 1 proposed a persisted Unified Historical Session Index. The code audit showed it is
not needed:

- Codex already keeps a SQLite thread index per home (`state_5.sqlite`, `threads`), with cwd,
  title, git origin/branch, archive and subagent flags.
- OpenCode keeps `session_v2` + `project` in `opencode.db`.
- Claude keeps one JSONL per session under `projects/<encoded-cwd>/`; a bounded head scan gives
  cwd, branch and a title, and `mtime` gives recency.

Each machine therefore answers "my recent sessions" **on request** from its own stores, with an
in-memory cache keyed by file `mtime/size` or `MAX(updated_at)`. There is no cockpit database,
no replace/dedupe semantics and nothing to migrate or repair.

### 6.2 Source of truth

Native stores are the only authority for sessions and transcripts. Herdr is the authority for
what is live. The browser merges the two per machine via `live_pane_id`.

### 6.3 Wire record

`HistorySession` in `shared/history.ts` (review §3): opaque `ref {provider, profile, native_id}`,
title (≤120 chars), cwd, repo identity, branch, model, times, `live_pane_id`, `resumable`.
No filesystem path is ever sent to the browser; the owning machine re-resolves the ref inside
its canonical store on every read.

### 6.4 What is persisted

Only on the connection server, both in `stateDir`, 0600, temp+rename, deletable at any time:

1. `history-cache/<machine-id>.json` — the last successful listing of each **remote** PC, used
   only to draw that PC's rows as stale while it is offline (mirrors upstream's `machines.json`
   roster snapshot). Metadata only, titles included (O3 approved). Anything served from it carries
   `freshness: "stale"` plus `as_of`; with no cache the answer is `freshness: "unavailable"`.
   A cached answer is never treated as current (owner condition on rev 2 approval).
2. `project-links.json` — the user's explicit "these are the same project" links.

No transcript text, tool output or image is ever copied.

---

## 7. Project identity and grouping (revised)

Identity is computed **on the machine that owns the session** and merged in the browser.

| Signal | Source | Use |
|---|---|---|
| Normalized remote `host/owner/repo` | Codex `git_origin_url`, else `git -C <cwd> config --get remote.origin.url` | equal on two PCs → **same project** |
| Root commit | `git rev-list --max-parents=0 HEAD`; OpenCode `project.id` if Batch 0 confirms | equal without a remote match → **suggested link** only |
| Explicit link | `project-links.json` | user-confirmed merge |
| `machine_id + toplevel` (or cwd) | fallback | machine-scoped project |

Rules: never merge by folder basename; a fork or template sharing a root commit is never merged
automatically; worktrees and branches group under their repo with a branch badge; a vanished
cwd keeps the identity recorded natively or falls back to machine scope; git probes have a 2 s
timeout and a 10-minute cache per cwd. Grouping is a pure client function
(`groupProjects`, `src/lib/history.ts`).

---

## 8. Machine architecture

### 8.1 Connection server

The preferred topology is one primary Web UI connection server, initially on Linux.

```text
Browser
  │
  ├── Local Linux Herdr + native stores
  │
  └── SSH remote bridge
        └── Mac Herdr + native stores
```

The browser should not need direct filesystem or SSH access.

### 8.2 Remote history route

The same `server/history/*` code runs on every PC (the remote bridge runs the same server code).
The connection server reaches a remote PC through a fork-owned route,
`GET /api/history/machines/:id/<sub>`, which takes `MachineManager.endpoint(id)` and the bridge
bearer exactly like `server/machine-api.ts` does, with an explicit allowlist of `<sub>`.
`MACHINE_PROXY_PATH` is not edited. On the remote side `/api/history/` is added to the
`bridgePath` gate (`server/index.ts:1429`) so the bridge token is accepted.

### 8.3 Disconnected machine behavior

- Listing: cached rows, `freshness: "stale"`, `as_of` (no cache → empty, `freshness: "unavailable"`); drawn dim with a dashed glyph and "last seen".
- Conversation, tool output, image, resume check: 503 `machine_offline`; inline "Mac is offline"
  state; no Linux fallback copy (there is none).
- Live status that was "needs you" becomes "unknown" — never shown as blocked while unreachable.
- Actions disabled with reasons; an explicit "Start in <project> on <other PC>…" opens the New
  workspace dialog prefilled. Nothing happens automatically.

### 8.4 Remote bundle deployment (new, required)

The Mac bridge runs a **remote bundle**, which by default is upstream's release bundle. Fork
routes do not exist there until the fork ships its own bundle:

- build with `bun run build:remote darwin-arm64` (works from Linux) and point the connection
  server at it with a local `remote-bundles/manifest-<os>-<cpu>.json` or `HERDR_WEB_BUNDLE_MANIFEST`;
- bridge currency is an exact `bundle_version` string match and `newerBundle` only accepts
  `^\d+$`; the fork's version rule (keep upstream's number + local manifest, or a reserved
  number) is decided in Batch 0 and re-applied at every upstream sync;
- exactly **one connection server** manages the Mac. The upstream install on `127.0.0.1:7317`
  must be disconnected from the Mac or stopped, or the two will keep replacing each other's bridge;
- deploying is an external effect: owner-approved, logged `outcome_unknown` until verified.

---

## 9. Provider adapters

Reuse upstream transcript readers where possible.

### Claude Code

Use the upstream/native Claude history reader and session identity logic. Historical discovery should enumerate only validated Claude session stores belonging to known project roots.

### Codex and Codex2

`codex2` is `codex` with `CODEX_HOME=~/.codex-pro-2`. Model it as provider `codex` with
`profile` = the home's directory name, discovered by the same rule `server/usage.ts` uses
(`CODEX_HOME`, `~/.config/codex`, `~/.codex`, every `~/.codex-*`). "Codex2" is a display alias.

Reuse upstream Codex rollout discovery rather than inventing another parser.

### OpenCode

Reuse the supported OpenCode database/session integration where possible.

### AGY / Antigravity

Do not ship a heuristic parser without a stable, verified native source format. If no reliable adapter exists, V1 can represent AGY as `unknown`/unsupported historical history while preserving live Herdr behavior.

### Unsupported runtimes

Unknown agents remain usable through the existing Herdr live terminal. Historical chat support is adapter-specific, not universal.

---

## 10. Resume semantics (revised)

No new mutating endpoint. Resume and fresh continuation use the existing
`POST /api/workspace/create` / `/api/tab/create` (`agent: { kind, args }`), already allowlisted
for remote PCs. The only new endpoint is a read-only `GET /api/history/resume-check`.

### 10.1 Live Herdr agent

Row has `live_pane_id` → open the pane; normal Send. A historical row whose session is live
elsewhere offers **Go to live pane**, never a second agent.

### 10.2 Native resume

On the session's own PC and cwd: `claude --resume <id>`, `codex resume <id>`,
`opencode --session <id>` (exact flags confirmed in Batch 0). The check is re-run immediately
before the create call. No provider, model or machine is ever switched silently.

### 10.3 Fresh continuation

New agent of the same kind in the project's checkout on the chosen PC, with no transcript or
summary injected. Used when native resume is unavailable (`cwd_missing`, `session_missing`,
`unsupported_profile`) or when the user prefers it.

### 10.4 Codex2 (UNKNOWN)

herdr `agent.start` has no environment parameter, so `codex resume <id>` would run against the
default home. Until Batch 0 finds a supported way, non-default Codex homes are `fresh-only`, with
a "copy resume command" affordance.

---

## 11. Telemetry and quota

Telemetry is a later phase, not a blocker for the historical cockpit.

### 11.1 Existing upstream behavior

Upstream already contains subscription usage support for signed-in CLI accounts on the server machine.

We should reuse this mechanism.

### 11.2 Future unified usage

After the Mac remote integration and history listing are stable, add machine-aware usage aggregation if upstream does not already provide it.

Desired normalized view:

```text
Claude Max · account A · Mac
5h      72% used · resets 02:14
week    41% used · resets Monday
source  exact

Codex · account B · Linux
5h      18% used
source  exact
```

Every metric must carry provenance:

- `exact`
- `estimated`
- `unknown`

Never present an estimate as a provider-authoritative percentage.

### 11.3 Not an orchestration engine

Telemetry V1/V2 is visibility only.

It must not automatically kill, switch provider, move tasks or trigger failover.

Those behaviors remain out of scope until separately measured and designed.

---

## 12. Visual direction / Stitch brief

The existing Web UI proves the plumbing but is not yet the desired long-form work surface.

The visual redesign should start with **one main screen only** before propagating changes.

### 12.1 Product character

Target:

- local developer cockpit;
- desktop-first but responsive;
- high information density without visual noise;
- serious productivity tool, not corporate SaaS dashboard;
- conversation and terminal are first-class;
- dark mode should feel native and calm rather than neon/gamer;
- hierarchy should come from spacing, typography, surfaces and state, not decorative cards.

Inspirational qualities, not copies:

- Linear: density and navigation discipline;
- ChatGPT/Claude: conversation legibility;
- VS Code: workspace/project mental model;
- modern terminal tooling: state clarity and keyboard efficiency.

### 12.2 Main screen composition

```text
┌────────────────────────────────────────────────────────────────┐
│ Herdr Cockpit                     Linux ●        Mac ●          │
├──────────────────┬─────────────────────────────────────────────┤
│ Projects         │ Projeto EAD                                 │
│                  │ Claude · Opus · Mac · 21:14                  │
│ ▼ Projeto EAD    │─────────────────────────────────────────────│
│   ● Current      │                                             │
│   ○ 21:14 Mac    │ conversation                                │
│   ○ 18:20 Linux  │                                             │
│   ○ Yesterday    │ tool work folded                            │
│                  │                                             │
│ ▶ CotaPay        │                                             │
│ ▶ Transporte     │─────────────────────────────────────────────│
│ ▶ Echo           │ [ Type prompt…                         ] ➤   │
├──────────────────┴─────────────────────────────────────────────┤
│ Claude 5h 72% · Codex 18% · main · clean                      │
└────────────────────────────────────────────────────────────────┘
```

### 12.3 Required Stitch output

Before implementation of the redesign, generate and review:

1. one desktop primary workspace screen;
2. same screen with historical session selected;
3. same screen with Mac disconnected;
4. compact responsive/mobile interpretation only after desktop direction is approved.

Do not ask Stitch to redesign every page at once.

### 12.4 Result (2026-10-10)

Screens 1–3 were generated in Stitch project `3911610958165352778`; PNG/HTML in
`docs/cockpit/stitch/`. Direction "lamp-lit graphite console" uses the existing upstream tokens
(no global restyle). Prompts, rationale, artifacts to ignore and owner decisions O1–O8:
`docs/cockpit/STITCH_DIRECTION.md`. **Not approved yet; do not implement the redesign.**

The design must account for existing upstream features rather than hiding them:

- chat/terminal toggle;
- approvals/questions;
- workspaces/panes;
- Remote PCs;
- status;
- settings;
- composer.

---

## 13. Error handling and safety

### Session discovery errors

- malformed native session → skip and record diagnostic, do not fail the listing;
- missing native source → mark unavailable;
- ambiguous project identity → do not auto-merge;
- duplicate detection → prefer stable native identity + machine ID;
- unsupported format/version → explicit unsupported state.

### Remote machine errors

- SSH unavailable → stale/offline state;
- host-key mismatch → fail closed using upstream security behavior;
- bridge incompatibility → surface upgrade requirement;
- timeout → do not silently fall back to another machine.

### Resume errors

- provider rejects resume → keep historical view and show failure;
- session already running → follow provider/upstream supported semantics;
- wrong cwd/project validation → block action rather than resuming into a different project.

### Privacy

- localhost-only by default;
- reuse upstream auth/security model;
- no new hosted relay;
- no central cloud transcript copy;
- credentials stay on the machine where provider CLI is configured.

---

## 14. Testing strategy

### Unit/adapter tests

Each provider adapter needs fixtures for:

- valid session;
- malformed session;
- missing metadata;
- multiple projects;
- duplicate ID across machines;
- branch/worktree metadata;
- unsupported format.

### Project identity tests

Must verify:

- same Git repo at different Mac/Linux paths groups correctly;
- same folder name but different repos does not group;
- non-Git workspace fallback behavior;
- explicit user alias/link overrides only when deliberately configured.

### Remote tests

Reuse/extend upstream SSH remote test harness for:

- Mac online/offline;
- historical enumeration through bridge;
- transcript read through bridge;
- machine-scoped identity;
- stale metadata behavior;
- no path traversal/arbitrary store access.

### UI tests

At minimum:

- project expansion/collapse;
- mixed Mac/Linux sessions under one project;
- live versus historical state;
- open historical conversation;
- resume enabled/disabled state;
- disconnected remote machine;
- switching between chat and terminal for live panes;
- composer targets correct machine/session.

### Acceptance scenario

A successful first vertical slice is:

1. Linux Web UI is running.
2. Mac is connected using upstream Remote PCs.
3. `Projeto EAD` exists on both machines.
4. The sidebar shows one `Projeto EAD` project.
5. Historical Claude/Codex sessions from both machines appear beneath it.
6. User opens an old Mac Claude session and reads its native conversation.
7. User switches to an active Linux Codex session and sends a harmless prompt.
8. Closing/restarting the browser or server loses nothing: history is re-read from native stores; offline PCs reappear from the stale cache.
9. Disconnecting Mac leaves its historical rows visible as unavailable/stale rather than deleting them.

---

## 15. Phased delivery (revised)

Batch details: `docs/cockpit/IMPLEMENTATION_PLAN.md`.

1. **Gate** — Batch 0 spike: resume flags, Codex2 env, OpenCode project id, Mac stores, bundle
   dry run, single connection server decision.
2. **Server history** — Batches 1–6: contract, identity, listers, local routes, remote route,
   offline cache, links.
3. **Client** — Batches 7–9: data layer, ChatView seam, UI for the three approved states.
4. **Deploy + accept** — Batches 10–11 with the owner.
5. **Follow-on** — usage across PCs (T1), then anything in §4.2.

---

## 16. Fork strategy

Preferred repository strategy after written-spec approval:

- create a user-owned fork of `devswha/herdr-web-ui`;
- keep `origin` pointing to the fork;
- keep `upstream` pointing to `devswha/herdr-web-ui`;
- avoid renaming core package identifiers until necessary;
- maintain small, separable commits so upstream changes can be merged regularly;
- contribute generally useful fixes/features upstream when they are not personal-workflow-specific.

Custom product-specific features such as unified project history can remain fork-only initially until their general value and upstream fit are understood.

---

## 17. Out-of-scope architecture decisions

This spec intentionally does **not** choose or implement:

- a universal LLM state machine;
- a central semantic task database;
- automatic continuation after provider quota exhaustion;
- provider capability matrix;
- autonomous workflow scheduler;
- durable supervisor agent;
- token-budget kill switches;
- pstack as the primary operating framework.

Those belong to the separate configuration-first experiment and should not contaminate this cockpit build.

---

## 18. Success criteria

The project succeeds if, during normal work, the owner can:

1. open one browser UI and see both Linux and Mac;
2. navigate by logical project rather than machine/session ID;
3. find an old session in seconds;
4. read the native conversation without hunting files manually;
5. send prompts to active sessions without returning to a separate terminal for routine work;
6. intentionally resume or start fresh without ambiguity about machine/runtime/project;
7. keep using the Herdr TUI and native CLIs in parallel;
8. receive upstream Herdr Web UI improvements without maintaining a full independent runtime;
9. avoid creating a second authoritative transcript store;
10. preserve local-first privacy and explicit remote-machine boundaries.

Token savings are not a success criterion for this product.

---

## 19. Design decision summary

**Chosen architecture:** fork and extend `herdr-web-ui`.

**Do not build:** new Herdr runtime, new terminal multiplexer, scheduler/control plane, universal router or copied transcript database.

**First custom feature:** unified historical session list grouped by logical project across Mac and Linux.

**First infrastructure step:** configure existing upstream Remote PCs against the Mac and prove one-browser/two-machine operation.

**First visual step:** one Stitch-designed primary workspace screen after the functional session-history vertical slice is planned.

**Telemetry:** reuse upstream subscription usage and extend to remote machines only if a verified gap remains.

**Recovery:** `task-resume` integration is optional and comes after the core cockpit proves useful.
