# Herdr Cockpit V1 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Extend `devswha/herdr-web-ui` into a single Mac + Linux cockpit that groups historical Claude/Codex/OpenCode sessions by logical project, opens native conversation history, and safely chooses native resume versus fresh continuation without duplicating transcripts.

**Architecture:** Fork upstream `herdr-web-ui` and preserve Herdr as the live runtime. Reuse upstream Remote PCs over SSH for the Mac, add a small machine-scoped historical-session index that stores only metadata/references, reuse existing provider-native transcript readers for message loading, and expose project-centric history through the existing React app. This plan deliberately stops before the Stitch visual redesign and before multi-machine quota telemetry; those are separate follow-on plans after functional V1 proves the data model and flows.

**Tech Stack:** Bun 1.4+, TypeScript 5.7, React 18, Vite 6, Herdr 0.9+, existing `herdr-web-ui` SSH bridge/RPC, existing Claude/Codex/OpenCode native transcript adapters, Bun test, Playwright-based UI regression scripts.

**Spec:** `docs/superpowers/specs/2026-10-10-herdr-cockpit-design.md`

## Global Constraints

- Keep Herdr as the engine/runtime; do not create a second runtime or scheduler.
- One logical workspace may span Mac and Linux; machine is secondary metadata, not the primary navigation hierarchy.
- Project identity is resolved from proven repository/workspace identity; matching folder names alone must never merge projects.
- The historical index stores metadata and opaque native references only; complete provider transcripts remain authoritative in their native stores.
- Historical discovery/read operations are read-only; malformed or unknown native formats fail closed.
- Reuse upstream Remote PCs over SSH for the Mac before adding custom remote transport.
- Reuse upstream Claude/Codex/OpenCode transcript readers rather than creating parallel parsers.
- AGY historical parsing is out of V1 unless a stable native source format is proven; live Herdr behavior remains available.
- Native resume must target the original machine/runtime/project explicitly; never silently switch provider/model/machine.
- Fresh continuation must not replay the complete old transcript.
- Localhost-first security and upstream auth/SSH boundaries remain intact; do not create a hosted relay.
- No automatic provider failover, context rotation, model router, orchestration daemon, pstack orchestration, analytics dashboard, or quota-driven actions in V1.
- Visual redesign through Stitch is a separate plan after functional V1; this plan may add minimal UI needed to expose the new functionality but must not restyle the whole product.

## Review Focus

1. **False project merge:** two unrelated repositories with the same directory basename must stay separate; Task 3 owns the regression test.
2. **Remote Mac unavailable:** last indexed metadata may render stale/offline, but transcript/resume actions must disable cleanly rather than falling back to Linux; Task 5 owns the regression test.
3. **Malformed or moved native transcript:** discovery skips/marks unavailable without crashing the index or returning arbitrary files; Task 4 owns the regression test.
4. **Resume into wrong checkout/project:** a stale historical session whose current target no longer matches its indexed project must be blocked; Task 7 owns the regression test.
5. **Duplicate live + historical representation:** the same native session discovered from Herdr live state and native storage must render once with live state attached; Task 6 owns the regression test.

---

## File Structure Locked For V1

New files in the fork:

- `shared/history.ts` — transport-safe normalized history/project/session types.
- `server/history/project-identity.ts` — deterministic logical-project identity and explicit alias/link handling.
- `server/history/discovery.ts` — provider-agnostic historical enumeration coordinator and deduplication.
- `server/history/claude.ts` — Claude historical-session enumeration using validated Claude store conventions.
- `server/history/codex.ts` — Codex/Codex2 historical-session enumeration using existing Codex home/store helpers.
- `server/history/opencode.ts` — OpenCode historical-session enumeration using existing OpenCode database helpers.
- `server/history/store.ts` — small disposable metadata index under the Web UI state directory; no transcript bodies.
- `server/history/read.ts` — historical conversation loader that delegates to existing provider-native readers.
- `server/history/resume.ts` — capability check and exact native resume descriptor; no execution side effects during capability calculation.
- `src/lib/history-api.ts` — typed client for history endpoints and machine-scoped requests.
- `src/lib/history-model.ts` — project/session grouping and live/historical merge helpers used by React.
- `src/components/ProjectHistorySidebar.tsx` — project-first navigation surface.
- `src/components/ProjectHistorySidebar.css` — minimal functional styling only.
- `src/components/HistoricalSessionView.tsx` — historical conversation shell using existing chat rendering primitives where possible.
- `src/components/HistoricalSessionView.css` — minimal functional styling only.

Existing files expected to change:

- `shared/protocol.ts` — export history API response/request envelopes if upstream convention keeps protocol types centralized.
- `server/index.ts` — register local history endpoints.
- `server/machine-api.ts` — allowlist machine-scoped history endpoints through the existing remote bridge.
- `server/conversation.ts` — expose/reuse the smallest safe provider-native read primitives needed by `server/history/read.ts`; do not duplicate transcript parsing.
- `server/claude-store.ts` — only if historical enumeration needs an exported validated store locator; no parser rewrite.
- `server/codex.ts` — only if historical enumeration needs an exported validated rollout enumerator/home helper.
- `server/opencode.ts` — only if historical enumeration needs an exported read-only session enumerator.
- `src/lib/api.ts` — only where common machine-aware request plumbing should be reused rather than duplicated.
- `src/App.tsx` — add selected project/historical session state and route the central surface between live Herdr panes and historical sessions.
- `src/components/AgentSidebar.tsx` — integrate or replace only the project/session portion necessary for project-first navigation; preserve upstream live-agent behavior.
- `src/components/ChatView.tsx` — reuse rendering surface where possible; do not fork message rendering unless historical read semantics require a small adapter.
- `scripts/ui-regression.ts` — add project/history interaction coverage.
- `server/api.contract.test.ts` — add local and remote history endpoint contracts.

Do not create a new transcript database or a parallel set of Claude/Codex/OpenCode parsers.

---

### Task 1: Fork Baseline + Remote Mac Acceptance Gate

**Files:**
- No product code required unless an upstream configuration bug is proven.
- Modify only after the baseline test: fork metadata/README note if needed.
- Evidence: `docs/cockpit-v1-baseline.md` in the fork.

**Interfaces:**
- Consumes: upstream `devswha/herdr-web-ui` v0.4.6+ Remote PCs flow.
- Produces: a verified fork checkout with `origin=<user fork>`, `upstream=devswha/herdr-web-ui`, clean baseline tests, and a Mac registered through existing Remote PCs or a documented blocker.

- [ ] **Step 1: Fork and clone without modifying upstream code**

Create `lubasilva/herdr-cockpit` from `devswha/herdr-web-ui`, clone to `~/Projects/herdr-cockpit`, set `upstream` to `https://github.com/devswha/herdr-web-ui.git`, and verify `git status --short` is empty.

- [ ] **Step 2: Run the upstream baseline**

Run:

```bash
bun install
bun test
bun run typecheck
bun run build
```

Expected: all commands PASS before cockpit changes.

- [ ] **Step 3: Verify existing Remote PCs with the Mac**

Use the current Web UI Settings → Remote PCs → Add PC flow with the existing SSH identity/alias. Do not modify SSH keys, firewall, or expose the Web UI publicly unless the existing flow explicitly requires an approved local setup step.

Expected: Mac appears as a distinct machine, its Herdr workspaces/panes are readable, and switching machine does not stop local Linux panes.

- [ ] **Step 4: Record the acceptance evidence**

Create `docs/cockpit-v1-baseline.md` with upstream commit, Bun/Herdr versions, baseline test results, Mac connection result, and any blocker. Do not store host secrets, tokens, private keys, or transcript content.

- [ ] **Step 5: Commit**

```bash
git add docs/cockpit-v1-baseline.md
git commit -m "docs: record cockpit v1 baseline"
```

---

### Task 2: Shared Historical Session Contract + Metadata Store

**Files:**
- Create: `shared/history.ts`
- Create: `server/history/store.ts`
- Test: `server/history/store.test.ts`
- Modify: `shared/protocol.ts` only if required by existing shared-protocol conventions.

**Interfaces:**
- Consumes: machine IDs/labels already used by upstream Remote PCs.
- Produces:
  - `HistoryRuntime = "claude" | "codex" | "codex2" | "opencode" | "agy" | "other"`
  - `HistoryConfidence = "verified" | "inferred"`
  - `HistoryLiveState = "live" | "historical" | "unknown"`
  - `HistoryResumeCapability = "native" | "fresh-only" | "unknown"`
  - `IndexedSession`
  - `LogicalProject`
  - `HistoryIndexStore` with `listSessions()`, `replaceMachineSessions(machineId, sessions)`, `getSession(id)`, `listProjects()`.

- [ ] **Step 1: Write failing store tests**

Add tests asserting:

```ts
it("stores metadata without transcript bodies")
it("replaces one machine without deleting another machine")
it("deduplicates stable machine + runtime + nativeSessionId")
it("survives a missing/corrupt index file by returning an empty index")
```

The persisted record must contain no `turns`, `messages`, `toolOutput`, `thinking`, `credentials`, or transcript-body field.

- [ ] **Step 2: Run the tests and verify failure**

Run:

```bash
bun test server/history/store.test.ts
```

Expected: FAIL because the history types/store do not exist.

- [ ] **Step 3: Define the normalized types in `shared/history.ts`**

Required `IndexedSession` fields:

```ts
id: string
projectId: string
machineId: string
machineLabel: string
runtime: HistoryRuntime
profile?: string
nativeSessionId?: string
nativeSourceRef: string
cwd?: string
repoRoot?: string
branch?: string
worktree?: string
startedAt?: string
updatedAt: string
title?: string
liveState: HistoryLiveState
resumeCapability: HistoryResumeCapability
confidence: HistoryConfidence
```

`nativeSourceRef` is opaque to the browser and must never become an arbitrary-path read parameter.

- [ ] **Step 4: Implement `HistoryIndexStore`**

Use the existing Web UI state directory convention. Persist only normalized metadata. Writes must be atomic (temporary file + rename or the upstream equivalent).

- [ ] **Step 5: Run tests**

Run:

```bash
bun test server/history/store.test.ts
bun run typecheck
```

Expected: PASS.

- [ ] **Step 6: Commit**

```bash
git add shared/history.ts server/history/store.ts server/history/store.test.ts shared/protocol.ts
git commit -m "feat: add historical session metadata index"
```

---

### Task 3: Deterministic Logical Project Identity

**Files:**
- Create: `server/history/project-identity.ts`
- Test: `server/history/project-identity.test.ts`

**Interfaces:**
- Consumes: `{ machineId, cwd?, repoRoot?, gitRemote?, herdrWorkspaceRoot?, configuredAlias? }`.
- Produces:
  - `resolveProjectIdentity(input: ProjectIdentityInput) -> ResolvedProjectIdentity`
  - stable `projectId`
  - human label
  - `confidence: "verified" | "inferred"`
  - optional `reason` for diagnostics.

- [ ] **Step 1: Write failing identity tests**

Cover:

```ts
it("maps Mac and Linux checkouts of the same canonical git repository to one project")
it("keeps same-basename directories separate when repository identity differs")
it("uses a configured project alias only when explicitly linked")
it("falls back to normalized machine-scoped directory identity instead of merging by basename")
```

- [ ] **Step 2: Verify tests fail**

Run:

```bash
bun test server/history/project-identity.test.ts
```

Expected: FAIL because `resolveProjectIdentity` is missing.

- [ ] **Step 3: Implement `resolveProjectIdentity`**

Identity precedence:

1. canonical Git repository identity when verified;
2. known Herdr workspace/repository root mapping;
3. explicit configured alias/link;
4. machine-scoped normalized directory identity.

Never merge solely on directory basename.

- [ ] **Step 4: Run tests and typecheck**

```bash
bun test server/history/project-identity.test.ts
bun run typecheck
```

Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add server/history/project-identity.ts server/history/project-identity.test.ts
git commit -m "feat: resolve logical projects across machines"
```

---

### Task 4: Provider Historical Discovery Without New Transcript Parsers

**Files:**
- Create: `server/history/discovery.ts`
- Create: `server/history/claude.ts`
- Create: `server/history/codex.ts`
- Create: `server/history/opencode.ts`
- Test: `server/history/discovery.test.ts`
- Test: provider-specific history tests as needed.
- Modify: `server/claude-store.ts`, `server/codex.ts`, `server/opencode.ts` only to export safe read-only discovery primitives.

**Interfaces:**
- Consumes: machine-local home/config paths and the project identity resolver from Task 3.
- Produces:
  - `discoverHistoricalSessions(context: HistoryDiscoveryContext): Promise<IndexedSession[]>`
  - provider functions `discoverClaudeSessions`, `discoverCodexSessions`, `discoverOpenCodeSessions` returning normalized metadata only.

- [ ] **Step 1: Write failing discovery tests**

Cover:

```ts
it("discovers a Claude session from the validated Claude projects store")
it("discovers default Codex and codex2 profile sessions as distinct runtime/profile metadata")
it("discovers OpenCode sessions through the existing OpenCode store helper")
it("skips malformed or out-of-store native sources")
it("does not parse transcript bodies during enumeration")
it("marks AGY unsupported/unknown instead of heuristic parsing")
```

For Review Focus #3, include a fixture with a malformed/moved transcript and assert discovery does not crash or accept an arbitrary path.

- [ ] **Step 2: Run the tests and verify failure**

```bash
bun test server/history/discovery.test.ts server/history/*.test.ts
```

Expected: FAIL because discovery adapters do not exist.

- [ ] **Step 3: Export the smallest safe upstream helpers**

Where necessary, expose validated enumeration helpers from the existing Claude/Codex/OpenCode modules. Do not copy parser logic from `server/conversation.ts`, `server/codex.ts`, or `server/opencode.ts` into the history directory.

- [ ] **Step 4: Implement provider enumeration**

Enumeration reads only metadata required for identity, timestamps, title/first prompt if already cheaply available, and native session references. It must not deserialize complete tool-heavy histories.

- [ ] **Step 5: Implement provider-agnostic discovery + dedupe**

Deduplication key must include machine + provider/profile + stable native session identity. A same session found from multiple evidence sources resolves to one `IndexedSession` with the stronger confidence.

- [ ] **Step 6: Run tests, typecheck, and targeted existing transcript tests**

```bash
bun test server/history/discovery.test.ts server/claude-session.contract.test.ts server/chat-history.test.ts
bun run typecheck
```

Expected: PASS with no regression in live chat transcript behavior.

- [ ] **Step 7: Commit**

```bash
git add server/history server/claude-store.ts server/codex.ts server/opencode.ts
git commit -m "feat: discover native historical agent sessions"
```

---

### Task 5: Local + Remote History API

**Files:**
- Modify: `server/index.ts`
- Modify: `server/machine-api.ts`
- Modify: `shared/protocol.ts`
- Test: `server/api.contract.test.ts`
- Test: `server/machine-api.test.ts` or existing closest machine-relay contract test.

**Interfaces:**
- Consumes: `HistoryIndexStore` and `discoverHistoricalSessions`.
- Produces machine-scoped endpoints:
  - `GET /api/history/projects`
  - `GET /api/history/sessions?project_id=<id>`
  - `GET /api/history/session?id=<indexed-session-id>`
  - remote equivalents through `/api/machines/:id/...` using the existing allowlisted relay.

- [ ] **Step 1: Write failing local API contract tests**

Assert authentication/method behavior matches existing server conventions and that responses never expose arbitrary native filesystem paths or transcript bodies in list endpoints.

- [ ] **Step 2: Write failing remote/offline tests**

For Review Focus #2:

```ts
it("returns machine unavailable for remote transcript/history read when Mac bridge is offline")
it("does not fall back to local history when the requested machine is offline")
it("keeps last metadata distinguishable as stale if the UI has a cached roster")
```

- [ ] **Step 3: Run tests to verify failure**

```bash
bun test server/api.contract.test.ts server/*machine*.test.ts
```

Expected: FAIL on missing history routes.

- [ ] **Step 4: Add local history routes**

Register endpoints in `server/index.ts` following existing auth/error envelopes. `GET /api/history/session` returns metadata/capabilities only at this task; transcript content is Task 6.

- [ ] **Step 5: Add history routes to the remote-machine allowlist**

Route through the current SSH bridge exactly like other machine-scoped APIs. Browser input must never contain a raw remote filesystem path.

- [ ] **Step 6: Run contract/integration tests**

```bash
bun test server/api.contract.test.ts server/*machine*.test.ts
bun run typecheck
```

Expected: PASS.

- [ ] **Step 7: Commit**

```bash
git add server/index.ts server/machine-api.ts shared/protocol.ts server/api.contract.test.ts server/*machine*.test.ts
git commit -m "feat: expose machine-scoped historical sessions"
```

---

### Task 6: Historical Conversation Read + Live/Historical Merge

**Files:**
- Create: `server/history/read.ts`
- Modify: `server/conversation.ts` only to expose reusable safe read entry points if necessary.
- Modify: `server/index.ts`
- Create: `src/lib/history-api.ts`
- Create: `src/lib/history-model.ts`
- Test: `server/history/read.test.ts`
- Test: `src/lib/history-model.test.ts`

**Interfaces:**
- Consumes: `IndexedSession`, upstream `ConversationResponse`/turn types, and existing machine-aware request transport.
- Produces:
  - `readHistoricalConversation(session: IndexedSession, page): Promise<ConversationResponse>`
  - `fetchHistoricalConversation(machineId, sessionId, page)`
  - `mergeLiveAndHistoricalSessions(live, historical): HistoryProjectView[]`.

- [ ] **Step 1: Write failing conversation-read tests**

Assert Claude/Codex/OpenCode historical reads delegate to the same parser/reader path used by live chat, reject an invalid indexed source, and preserve existing pagination/window limits.

- [ ] **Step 2: Write failing merge test**

For Review Focus #5, construct one native session present in both live Herdr state and historical discovery. Assert it appears once, with `liveState="live"` and live pane identity attached.

- [ ] **Step 3: Run tests to verify failure**

```bash
bun test server/history/read.test.ts src/lib/history-model.test.ts
```

Expected: FAIL on missing read/merge functions.

- [ ] **Step 4: Implement historical read using provider-native adapters**

The history layer must pass a validated opaque reference/session identity into provider code; it must not accept arbitrary paths from the browser. Preserve upstream transcript page limits and tool-output folding behavior.

- [ ] **Step 5: Add `GET /api/history/conversation`**

Input: indexed session ID + cursor/page fields. Machine-scoped remote routing follows Task 5. Response type is the existing `ConversationResponse` wherever possible.

- [ ] **Step 6: Implement client fetch and live/history merge**

Reuse the existing machine-aware fetch pattern in `src/lib/api.ts`. Do not create a parallel transport layer if the current helper can be extended cleanly.

- [ ] **Step 7: Run targeted regression tests**

```bash
bun test server/history/read.test.ts src/lib/history-model.test.ts server/chat-history.test.ts
bun run typecheck
```

Expected: PASS.

- [ ] **Step 8: Commit**

```bash
git add server/history/read.ts server/conversation.ts server/index.ts src/lib/history-api.ts src/lib/history-model.ts server/history/read.test.ts src/lib/history-model.test.ts
git commit -m "feat: read and merge historical conversations"
```

---

### Task 7: Resume Capability + Fresh Continuation Safety

**Files:**
- Create: `server/history/resume.ts`
- Modify: `server/index.ts`
- Modify: `shared/history.ts`
- Modify: `src/lib/history-api.ts`
- Test: `server/history/resume.test.ts`

**Interfaces:**
- Consumes: `IndexedSession`, machine/runtime identity, current checkout/project validation.
- Produces:
  - `resolveResumeDescriptor(session): Promise<ResumeDescriptor>` where descriptor contains `capability`, `machineId`, `runtime`, `cwd`, and an exact provider-native resume argv/command description only when validated.
  - `POST /api/history/resume` to request upstream Herdr creation/resume flow after server-side validation.
  - `POST /api/history/fresh` to start a new session in the same logical project without replaying history.

- [ ] **Step 1: Write failing resume tests**

Cover:

```ts
it("returns native capability only when provider session and project target still validate")
it("blocks resume when indexed repo/project no longer matches target checkout")
it("never silently changes machine or runtime")
it("returns fresh-only for unsupported historical runtimes")
it("fresh continuation does not include the historical transcript")
```

The second assertion is Review Focus #4.

- [ ] **Step 2: Run tests and verify failure**

```bash
bun test server/history/resume.test.ts
```

Expected: FAIL because resume capability does not exist.

- [ ] **Step 3: Implement capability resolution**

Use provider-native resume semantics already supported by Herdr/runtime. Build argv as structured data where upstream supports it; avoid shell interpolation of session IDs or paths.

- [ ] **Step 4: Implement resume/fresh endpoints through existing Herdr session creation primitives**

Resume failure must leave the historical session untouched. Fresh starts in the logical project checkout selected by the user/current mapping; it receives no full transcript injection.

- [ ] **Step 5: Run tests + existing worktree/new-session contracts**

```bash
bun test server/history/resume.test.ts server/api.contract.test.ts
bun run typecheck
```

Expected: PASS.

- [ ] **Step 6: Commit**

```bash
git add server/history/resume.ts server/index.ts shared/history.ts src/lib/history-api.ts server/history/resume.test.ts
git commit -m "feat: add safe historical resume and fresh continuation"
```

---

### Task 8: Project-First Historical UI

**Files:**
- Create: `src/components/ProjectHistorySidebar.tsx`
- Create: `src/components/ProjectHistorySidebar.css`
- Create: `src/components/HistoricalSessionView.tsx`
- Create: `src/components/HistoricalSessionView.css`
- Modify: `src/App.tsx`
- Modify: `src/components/AgentSidebar.tsx` only where needed to integrate project-first navigation.
- Modify: `src/components/ChatView.tsx` only to reuse existing message rendering primitives cleanly.
- Test: `scripts/ui-regression.ts`

**Interfaces:**
- Consumes: history project/session APIs, `mergeLiveAndHistoricalSessions`, existing `ConversationResponse`, resume/fresh endpoints.
- Produces: project → sessions sidebar; historical read-only conversation view; resume/fresh actions; machine/runtime badges.

- [ ] **Step 1: Add failing browser regression for project grouping**

Fixture should expose one logical project with Linux + Mac sessions and another unrelated same-basename directory. Assert the first is grouped, the second remains separate, and machine badges render.

- [ ] **Step 2: Add failing browser regression for historical session open**

Click a historical session and assert human/assistant turns appear using the existing conversation semantics, raw tool work remains folded according to upstream behavior, and live terminal controls are not falsely shown for a historical-only session.

- [ ] **Step 3: Add failing browser regression for resume/fresh states**

Assert:

- native-resumable session shows destination machine/runtime before resume;
- fresh-only session shows `Start fresh continuation`;
- offline Mac disables resume and explains unavailable state.

- [ ] **Step 4: Run UI regression and verify failures**

```bash
bun run build
bun scripts/ui-regression.ts
```

Expected: new assertions FAIL before UI implementation.

- [ ] **Step 5: Implement `ProjectHistorySidebar`**

Minimal functional UI only. Preserve upstream workspace/agent functionality; do not perform the Stitch redesign in this task.

- [ ] **Step 6: Implement `HistoricalSessionView`**

Reuse upstream chat rendering/message components. Historical mode is read-only except explicit resume/fresh actions.

- [ ] **Step 7: Wire selection state in `App.tsx`**

Selection key includes machine + indexed session/project identity. Switching from historical back to a live pane restores existing upstream behavior.

- [ ] **Step 8: Run UI regression, unit tests, typecheck**

```bash
bun run test:ui
bun test
bun run typecheck
```

Expected: PASS.

- [ ] **Step 9: Commit**

```bash
git add src/components/ProjectHistorySidebar.* src/components/HistoricalSessionView.* src/App.tsx src/components/AgentSidebar.tsx src/components/ChatView.tsx scripts/ui-regression.ts
git commit -m "feat: browse historical sessions by project"
```

---

### Task 9: Real Linux + Mac End-to-End Acceptance

**Files:**
- Create: `docs/cockpit-v1-acceptance.md`
- Modify product code only for defects proven by acceptance; each defect gets its own test first.

**Interfaces:**
- Consumes: complete V1 from Tasks 1–8.
- Produces: evidence that the cockpit works with real local Linux and remote Mac session stores without transcript replication.

- [ ] **Step 1: Seed non-destructive real acceptance cases**

Use existing historical sessions where possible. If a new test session is required, send a harmless prompt such as `Responda apenas COCKPIT_V1_OK e não altere arquivos.` on both machines.

- [ ] **Step 2: Verify required flows**

Record PASS/PARTIAL/FAIL for:

1. Linux Claude history;
2. Mac Claude history;
3. Linux Codex/default + codex2 histories;
4. Mac Codex history;
5. OpenCode history where a valid session exists;
6. one logical project grouping equivalent Mac/Linux repositories;
7. unrelated same-basename repos remaining separate;
8. opening a historical conversation after its Herdr pane no longer exists;
9. native resume on a supported session;
10. fresh continuation on unsupported/unavailable resume;
11. Mac disconnect/reconnect behavior;
12. no full transcript bodies written into the cockpit metadata index.

- [ ] **Step 3: Run full verification**

```bash
bun test
bun run typecheck
bun run build
bun run test:ui
```

Expected: PASS.

- [ ] **Step 4: Write acceptance evidence**

`docs/cockpit-v1-acceptance.md` must include tested upstream/fork commit, machines/runtimes tested, PASS/PARTIAL/FAIL table, known unsupported AGY behavior, and any follow-on gaps. Do not include private transcript text or credentials.

- [ ] **Step 5: Commit**

```bash
git add docs/cockpit-v1-acceptance.md
git commit -m "docs: record cockpit v1 acceptance"
```

---

## Follow-On Plans — Explicitly Not Part of This Plan

After Task 9 acceptance, create separate specs/plans only if still desired:

1. **Stitch Visual Pass** — run Stitch MCP against one approved desktop workspace state first; then implement the approved visual direction without changing V1 data semantics.
2. **Unified Usage/Telemetry** — extend existing upstream subscription usage to machine/account-aware aggregation only after proving what remote usage data is missing.
3. **`task-resume` Integration** — add bounded recovery packet to `Start fresh continuation` only if native history + fresh project continuation still leaves meaningful recovery friction.
4. **Historical Search** — search prompts/final results across indexed sessions; do not index raw tool output by default.

Do not fold these into V1 merely because the data model makes them possible.

---

## Whole-Branch Verification Before Review

Run:

```bash
bun test
bun run typecheck
bun run build
bun run test:ui
```

Then verify `git status --short` contains only intentional changes and compare the fork against the upstream base to ensure no unrelated refactor entered the branch.

The final reviewer should explicitly check:

- no transcript bodies are persisted in the history index;
- no raw path can be supplied by the browser to read arbitrary files;
- remote history requests stay machine-scoped;
- same-basename projects cannot merge without stronger identity evidence;
- resume cannot switch machine/runtime/project silently;
- existing live Herdr chat/terminal/worktree flows still pass regression tests.
