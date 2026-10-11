# Prompt — Opus Architecture + UX Review Before Implementation

You are the principal architecture/product reviewer for **Herdr Cockpit**.

This is a planning/design pass. **Do not implement the feature batches yet.**
You may update planning/design documents and use Stitch MCP for the approved visual exploration, but do not start the V1 implementation tasks.

## Repository

Linux checkout:

`/home/lucas/Projects/herdr-cockpit`

Git:

- fork: `https://github.com/lubasilva/herdr-cockpit`
- upstream: `https://github.com/devswha/herdr-web-ui`
- planning branch: `planning/cockpit-v1`
- upstream source at the time of the fork: `devswha/herdr-web-ui`

Read first, completely:

1. `AGENTS.md`
2. `server/AGENTS.md`
3. `src/AGENTS.md`
4. `docs/cockpit/DESIGN.md`
5. `docs/cockpit/IMPLEMENTATION_PLAN.md`
6. upstream `README.md`
7. upstream `DESIGN.md`
8. `docs/remote-pcs.md`
9. relevant chat/session/usage docs and code you discover

Also inspect the actual repository code before agreeing with any proposed file/interface names.

## Product intent

The owner uses Herdr as his primary development cockpit across **Linux + Mac**.

He wants one browser surface where both computers feel like **one logical development environment**.

The core UX problem is not just live agents. It is historical continuity:

- open a logical project such as AVA;
- see its current session plus historical sessions;
- include sessions created on both Mac and Linux;
- include Claude, Codex/Codex2 and OpenCode where their native history is safely readable;
- click an old session and read the actual conversation;
- resume it natively where safe;
- otherwise start a fresh continuation in the same project;
- still send prompts to live agents from the browser;
- keep the terminal available as a first-class fallback.

The user does **not** want to identify old work by opaque UUIDs or manually browse `/sessions` in the terminal.

Navigation principle:

`Project → sessions → conversation`

Machine/runtime/model are secondary metadata badges.

## Important facts already verified — do not waste time rediscovering these from scratch

### Existing upstream Web UI

`devswha/herdr-web-ui` is already installed and working on the Linux machine.

It already provides substantial plumbing:

- browser chat + live terminal for Herdr panes;
- prompt sending;
- approvals/questions;
- workspaces/panes/worktrees;
- provider-native transcript readers;
- Remote PCs over SSH;
- PWA/mobile;
- subscription usage for accounts on the server machine.

We are therefore **forking/extending it**, not building a cockpit from zero.

### Linux + Mac Remote PCs

Remote Mac support has already been configured successfully using the upstream Remote PCs flow.

Current topology works:

`Browser → Linux herdr-web-ui connection server → SSH alias macbook → remote bridge → Mac Herdr`

The Mac reports connected in `/api/machines`.

Therefore:

- do not design a new Mac transport;
- do not replace SSH;
- do not invent another remote daemon unless existing architecture proves impossible for a specific history endpoint;
- extend the existing machine-scoped API/bridge if historical discovery needs additional allowlisted endpoints.

### Subscription usage / quota telemetry

The existing local `/api/usage` is already returning real provider windows for signed-in Claude, Codex and OpenCode accounts.

Do **not** rebuild provider usage clients from scratch.

The actual open question is narrower:

> how should this existing usage mechanism become machine-aware / multi-PC so the cockpit can show Linux + Mac account usage coherently?

This is a follow-on phase, not a blocker for historical-session V1.

### Configuration-first work already applied separately

The owner is also running a Phase 0 experiment for context/quota efficiency:

- Claude auto-compaction around 300k;
- concise global policies;
- Codex/Codex2 policy/hooks alignment;
- task-resume prototype;
- selective pstack evaluation.

Do not pull the old large control-plane concept back into this product.

The cockpit must remain useful even if token savings are zero.

## Architecture constraints already approved

Preserve these unless you find concrete code evidence that one is wrong:

1. **Herdr stays the runtime.**
2. **Fork upstream instead of rewriting it.**
3. **Mac + Linux are one logical workspace.**
4. **Project-first navigation.**
5. **Historical index stores metadata/references, not duplicate full transcripts.**
6. **Native provider stores remain authoritative for conversation contents.**
7. **Reuse upstream transcript readers.**
8. **No scheduler/control-plane daemon/model router/provider failover in V1.**
9. **AGY historical history may remain unsupported if no stable store can be proven.**
10. **Stitch redesign starts with one main screen, not the entire product.**

## Your job — Part 1: Adversarial architecture review

Audit `docs/cockpit/DESIGN.md` and `docs/cockpit/IMPLEMENTATION_PLAN.md` against the real upstream code.

Do not preserve our architecture out of politeness.

For every major assumption, classify it:

- VERIFIED IN CODE
- CORRECT BUT NEEDS ADJUSTMENT
- REDUNDANT — UPSTREAM ALREADY DOES IT
- WRONG
- UNKNOWN

Pay particular attention to:

- whether we truly need a persisted historical metadata index at all;
- whether upstream already has reusable indexes/readers that can enumerate historical sessions rather than only resolve a live pane;
- correct extension point for remote-machine historical discovery;
- project identity across different Mac/Linux paths;
- Codex versus Codex2 profile/home handling;
- deduping a live Herdr pane with the same native historical session;
- stale/disconnected remote-machine behavior;
- resume semantics and whether native providers already expose better primitives;
- preserving upstream updateability and minimizing fork divergence.

If the current implementation plan names the wrong files or abstractions, rewrite it before implementation begins.

## Part 2: Minimize fork divergence

We want to keep pulling upstream changes.

Design every custom feature with this question:

> Can this be a small new module/component/API extension instead of a rewrite of an upstream core file?

Specifically inspect the large upstream files such as `src/App.tsx`, `ChatView`, `Composer`, server conversation machinery and machine routing.

Recommend seams that let us avoid permanent high-conflict edits.

Where possible:

- add modules instead of replacing them;
- extend existing types instead of parallel protocols;
- reuse native adapters;
- keep custom UI composition isolated;
- preserve upstream tests;
- document any fork-only patch that is likely to conflict during upstream sync.

Produce an explicit **Upstream Divergence Risk** section.

## Part 3: Historical-session architecture

Design the smallest reliable implementation for:

### Historical discovery

Across local Linux and connected Mac, enumerate supported native sessions and normalize only metadata required for discovery/navigation.

At minimum evaluate:

- Claude Code native session stores;
- Codex default home;
- Codex2/custom CODEX_HOME;
- OpenCode 2 native DB;
- unsupported/unknown runtimes.

No heuristic parser just to achieve feature parity.

### Logical project identity

A project can exist at different filesystem paths:

- Linux `/home/lucas/Projects/projeto-ead`
- Mac `/Users/ipremium/Projects/projeto-ead`

Same basename is **not sufficient evidence**.

Determine the most reliable identity signals available in this codebase and environment, preferably Git remote/repository identity + workspace/repo root, with explicit user linking only for ambiguous cases.

### Historical conversation read

Reuse upstream readers/parsers.

Do not persist full transcript copies.

If some existing readers are coupled to a live pane, determine the smallest refactor that makes them reusable by an explicit historical session reference without weakening their current safety checks.

### Resume

Define exact semantics for:

- live agent → normal send;
- historical provider session that can resume → native resume on original machine/project;
- unsafe/unavailable resume → fresh continuation.

Do not replay a complete old transcript into a fresh context.

## Part 4: Stitch MCP — visual direction

The owner explicitly wants Stitch MCP used for the visual direction.

Do **not** redesign every screen.

Create a strong design brief and then use Stitch MCP for only the primary cockpit experience first.

The desired character:

- local developer cockpit;
- desktop-first;
- high information density without clutter;
- dark mode should feel calm and professional, not gamer/neon;
- no generic admin dashboard;
- no ocean of cards;
- conversation and terminal are first-class;
- keyboard-friendly;
- hierarchy comes from typography, spacing, borders/surfaces and state;
- should feel at home next to tools used for hours every day.

Useful design qualities:

- Linear: disciplined density/navigation;
- ChatGPT/Claude: conversation legibility;
- VS Code: projects/workspaces mental model;
- modern terminal tooling: operational clarity.

Do not copy any product literally.

### Stitch screen 1 — required

Primary desktop workspace:

- project sidebar;
- collapsible sessions below each project;
- both Mac and Linux sessions in the same logical project;
- active/live session state;
- historical session state;
- main conversation surface;
- folded tool activity;
- composer;
- terminal/chat toggle;
- machine/runtime/model/branch metadata without overwhelming the UI;
- small machine connectivity indication;
- existing upstream functionality must remain discoverable.

### Stitch screen 2 — required

Same screen with an old historical session selected:

- obviously read-only until Resume/Fresh Continuation is chosen;
- source machine/runtime visible;
- `Resume` when native resume is valid;
- `Start fresh continuation` otherwise;
- do not confuse the old conversation with the currently live agent.

### Stitch screen 3 — required

Same primary screen with Mac disconnected:

- historical metadata remains visible;
- Mac sessions clearly unavailable/stale;
- no fake fallback to Linux;
- live actions disabled appropriately;
- state should be understandable without a modal blocking the app.

Do not propagate the visual system to the whole codebase yet.

Save/export enough Stitch evidence/links/artifacts so the owner can review these three states before visual implementation.

## Part 5: Telemetry recommendation — plan only

Inspect the existing usage implementation and Remote PC machine proxy.

Do not implement multi-PC quota telemetry in this pass.

Produce the smallest follow-on design for eventually showing provider/account windows across machines.

Important constraints:

- reuse existing provider usage code;
- account identity must not be confused with profile identity;
- same account logged in on both machines should not necessarily appear as two independent quotas;
- every displayed number needs provenance: exact / estimated / unknown;
- telemetry is visibility only — no automatic task migration/failover in this product phase.

## Part 6: Baseline and testing

Before changing implementation files:

1. inspect the current branch/status;
2. verify `origin` and `upstream`;
3. run targeted relevant tests/typecheck rather than assuming the fork is clean;
4. inspect any baseline failures before attributing them to our branch.

The initial full upstream `bun test` on this Linux machine showed timing-related failures in `server/output.contract.test.ts` while many hundreds of other tests passed. Treat these as **baseline evidence to reproduce/isolate**, not as regressions from cockpit code. Re-run that file alone and determine whether this is load/flakiness/environment before planning acceptance gates.

Do not “fix upstream” unless the cockpit requires it.

## Part 7: Revise the plan

After the code audit and Stitch exploration:

Update, as necessary:

- `docs/cockpit/DESIGN.md`
- `docs/cockpit/IMPLEMENTATION_PLAN.md`

Create:

- `docs/cockpit/OPUS_ARCHITECTURE_REVIEW.md`
- `docs/cockpit/STITCH_DIRECTION.md`

`OPUS_ARCHITECTURE_REVIEW.md` must contain:

1. verified upstream capabilities;
2. assumptions we got wrong;
3. final recommended V1 architecture;
4. precise extension seams/files;
5. upstream-divergence risks;
6. historical-data safety/privacy model;
7. Mac/Linux project identity strategy;
8. resume strategy;
9. what is deliberately deferred;
10. exact implementation order for cheaper worker agents.

`STITCH_DIRECTION.md` must contain:

1. exact prompt(s) sent to Stitch;
2. artifact/reference IDs or links produced;
3. rationale for the chosen direction;
4. unresolved visual decisions that require owner approval;
5. explicit instruction not to implement the visual redesign until owner approves the selected direction.

## Part 8: Worker decomposition

The owner wants Opus for difficult planning/judgment and cheaper agents for execution.

Turn the final implementation plan into worker-sized batches.

For each batch specify:

- objective;
- files/interfaces;
- prerequisite batch;
- tests;
- what model capability it actually needs;
- whether it is suitable for Codex/Sonnet/DeepSeek-style execution;
- reviewer gate.

Avoid parallel workers touching the same core files at the same time.

Prefer bounded fresh-agent tasks over one giant autonomous session.

## Hard constraints

Do NOT:

- implement the V1 batches yet;
- rewrite Herdr;
- create a new remote transport;
- create a second transcript authority;
- copy entire provider histories into a new DB;
- build scheduler/failover/router/control-plane features;
- install a large orchestration framework;
- restyle the entire product before the 3 Stitch states are approved;
- “fix” unrelated upstream code;
- silently merge ambiguous projects;
- expose credentials/provider auth files to the browser.

## Final response format

Return:

### A. Architecture verdict
Choose one:
- PLAN VALID AS WRITTEN
- PLAN VALID WITH SMALL CHANGES
- PLAN NEEDS MATERIAL REVISION

### B. What upstream already solved
Short and concrete.

### C. What we actually need to build
Ranked V1 list.

### D. Changes made to DESIGN / IMPLEMENTATION_PLAN
List exact files and substantive changes.

### E. Stitch result
- screen 1
- screen 2
- screen 3
- recommended direction
- owner decisions still needed

### F. Worker batches
Ordered implementation batches with recommended execution-model tier.

### G. Baseline verification
Typecheck/tests and the status of the existing `output.contract.test.ts` failures.

### H. STOP
Do not start implementation. End by asking the owner to approve/reject the architecture + Stitch direction.
