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

No V1 implementation task has started yet. Next: owner approves/rejects architecture and Stitch
direction; then Batch 0.
