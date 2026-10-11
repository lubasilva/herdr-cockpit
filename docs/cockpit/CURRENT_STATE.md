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

- dependency install: PASS
- TypeScript typecheck: PASS
- full `bun test`: started from pristine upstream fork and produced multiple timing/contract failures before being intentionally stopped to avoid spending more time on a known-red baseline
- `server/output.contract.test.ts` was re-run alone and the `observer high watermark` timeout reproduced, so it is **not proven to be only full-suite load**

These failures existed before any cockpit implementation code was added.

Likely investigation axes include Herdr/runtime version compatibility and local test-environment assumptions. Do not attribute them to cockpit changes and do not fix unrelated upstream behavior unless required by the feature.

## Planning artifacts

- `docs/cockpit/DESIGN.md`
- `docs/cockpit/IMPLEMENTATION_PLAN.md`
- `docs/cockpit/OPUS_REVIEW_PROMPT.md`

No V1 implementation task has started yet.
