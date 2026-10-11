# Herdr Cockpit — Unified Mac + Linux Session History

Date: 2026-10-10
Status: Design approved in conversation; written spec awaiting explicit user review before implementation planning.

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
3. **Index, do not duplicate.** Store session metadata and references; read message history from provider-native stores when the user opens a session.
4. **Live Herdr and historical provider state are different concerns.** Herdr owns currently running panes/agents. The historical index must survive after panes disappear.
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
2. **Unified Historical Session Index** across Linux and Mac
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

## 6. Unified Historical Session Index

### 6.1 Why a separate index is required

Herdr knows about live panes and sessions, but the desired UX must survive after those panes no longer exist.

Provider-native histories also live in different locations and formats on different machines.

Therefore a small index is required to answer quickly:

- which project did this session belong to?
- which machine created it?
- which runtime/profile owns it?
- when did it start/update?
- where is its native source?
- is it currently live, resumable or historical-only?

### 6.2 Source of truth

The index is **not** the source of truth for conversation contents.

It stores references and normalized metadata.

When the user opens a session, the appropriate native adapter reads the native session store on the machine that owns it.

### 6.3 Minimum normalized record

Conceptual record:

```ts
interface IndexedSession {
  id: string;                  // cockpit-stable identity
  projectId: string;
  machineId: string;
  machineLabel: string;
  runtime: 'claude' | 'codex' | 'codex2' | 'opencode' | 'agy' | 'other';
  profile?: string;
  nativeSessionId?: string;
  nativeSourceRef: string;     // opaque adapter-owned reference, not necessarily a raw path
  cwd?: string;
  repoRoot?: string;
  branch?: string;
  worktree?: string;
  startedAt?: string;
  updatedAt: string;
  title?: string;
  liveState: 'live' | 'historical' | 'unknown';
  resumeCapability: 'native' | 'fresh-only' | 'unknown';
  confidence: 'verified' | 'inferred';
}
```

The final schema should follow upstream coding conventions rather than this exact shape if a better existing abstraction exists.

### 6.4 No raw transcript duplication

V1 must not store:

- complete Claude JSONL copies;
- complete Codex rollout copies;
- full OpenCode conversation copies;
- screenshots/base64;
- raw tool logs;
- private model thinking;
- credentials or auth files.

If caching is later necessary for performance, it must be bounded and disposable; provider-native storage remains authoritative.

---

## 7. Project identity and grouping

This is the most important data-model decision.

A session belongs to a logical project, not merely to a machine path.

Project identity resolution should use deterministic signals in this order where available:

1. Git repository identity / canonical repository root;
2. known Herdr workspace root;
3. configured project alias;
4. normalized local directory as a last resort.

Mac and Linux paths may differ while representing the same project.

Example:

```text
Linux: /home/lucas/Projects/projeto-ead
Mac:   /Users/ipremium/Projects/projeto-ead
```

Both must map to one logical `Projeto EAD` project if their repository/workspace identity proves equivalence.

The implementation must not merge two projects merely because their folder names match.

Where identity cannot be proved, show separate projects until the user explicitly links them.

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

### 8.2 Remote historical adapters

Historical session enumeration and transcript loading must execute on the machine that owns the provider store.

For Mac, that means the existing remote bridge/API should expose or be extended with narrowly scoped session-index/session-read endpoints rather than copying Mac stores to Linux.

Requirements:

- read-only for historical discovery;
- machine-scoped responses;
- no arbitrary filesystem browsing exposed to the browser;
- provider adapters validate canonical store locations;
- malformed/unknown native formats fail closed.

### 8.3 Disconnected machine behavior

If the Mac is offline:

- last indexed metadata may remain visible;
- UI must mark the machine/session as unavailable/stale;
- transcript loading that requires the Mac must fail clearly;
- the cockpit must not fabricate cached transcript content as current;
- live actions must be disabled.

---

## 9. Provider adapters

Reuse upstream transcript readers where possible.

### Claude Code

Use the upstream/native Claude history reader and session identity logic. Historical discovery should enumerate only validated Claude session stores belonging to known project roots.

### Codex and Codex2

Treat `codex` and `codex2` as separate profiles when their storage/config homes differ, while still presenting both under the same project.

Reuse upstream Codex rollout discovery rather than inventing another parser.

### OpenCode

Reuse the supported OpenCode database/session integration where possible.

### AGY / Antigravity

Do not ship a heuristic parser without a stable, verified native source format. If no reliable adapter exists, V1 can represent AGY as `unknown`/unsupported historical history while preserving live Herdr behavior.

### Unsupported runtimes

Unknown agents remain usable through the existing Herdr live terminal. Historical chat support is adapter-specific, not universal.

---

## 10. Resume semantics

V1 must distinguish three cases.

### 10.1 Live Herdr agent

Prompt goes to the existing live agent using upstream Herdr behavior.

### 10.2 Native historical resume available

The cockpit may invoke the provider's supported native resume flow on the original machine/project.

Requirements:

- explicit destination machine/runtime;
- no hidden provider switch;
- no silent model switch;
- native session ID is validated before use;
- failure leaves the original history untouched.

### 10.3 Native resume unavailable or unsafe

Offer fresh continuation.

V1 fresh continuation may initially start an empty session in the same project. A future increment can integrate `task-resume` to inject a bounded recovery packet.

The complete historical transcript must never be automatically replayed into the fresh session.

---

## 11. Telemetry and quota

Telemetry is a later phase, not a blocker for the historical cockpit.

### 11.1 Existing upstream behavior

Upstream already contains subscription usage support for signed-in CLI accounts on the server machine.

We should reuse this mechanism.

### 11.2 Future unified usage

After the Mac remote integration and historical index are stable, add machine-aware usage aggregation if upstream does not already provide it.

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

- malformed native session → skip and record diagnostic, do not crash index;
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
8. Closing/restarting the browser does not lose historical index metadata.
9. Disconnecting Mac leaves its historical rows visible as unavailable/stale rather than deleting them.

---

## 15. Phased delivery

### Phase 1 — Upstream baseline and Mac connection

- fork upstream;
- preserve `upstream` remote;
- clone to Linux;
- add as normal Herdr project/workspace;
- configure Mac through existing Remote PCs;
- run upstream tests and establish clean baseline;
- document gaps against desired cockpit.

No visual redesign yet.

### Phase 2 — Unified session index

- session metadata contracts;
- project identity resolver;
- local Linux indexing;
- remote Mac indexing through bridge;
- supported provider adapters;
- persistent metadata cache/index;
- historical session sidebar.

### Phase 3 — Historical conversation UX

- open native historical transcript;
- project/session filters;
- live/historical distinction;
- native resume capability detection;
- fresh continuation fallback.

### Phase 4 — Stitch-led visual pass

- produce one approved main-screen direction;
- implement only that screen/system first;
- regression test existing live functionality;
- propagate visual system only after owner approval.

### Phase 5 — Usage/telemetry enhancement

- audit upstream subscription usage end-to-end;
- add remote-machine usage if actually missing;
- aggregate by account/machine without double counting;
- show exact/estimated/unknown provenance;
- no automatic routing/failover.

### Phase 6 — Optional recovery integration

Only if still useful after real usage:

- expose `task-resume` packet in fresh-continuation flow;
- never auto-inject full transcript;
- keep recovery deterministic and bounded.

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

**First custom feature:** unified historical session index grouped by logical project across Mac and Linux.

**First infrastructure step:** configure existing upstream Remote PCs against the Mac and prove one-browser/two-machine operation.

**First visual step:** one Stitch-designed primary workspace screen after the functional session-history vertical slice is planned.

**Telemetry:** reuse upstream subscription usage and extend to remote machines only if a verified gap remains.

**Recovery:** `task-resume` integration is optional and comes after the core cockpit proves useful.
