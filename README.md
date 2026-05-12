# cartesi-skills

Reusable skill packs for AI coding agents building with Cartesi Rollups.

## What is in this repo

- `cartesi-workflow/` - starter guide and routing map; explains the phases of a full-stack Cartesi dApp and points to the right skill for each phase. Load this first when unsure where to start.
- `cartesi-scaffold/` - scaffold a new Cartesi Rollups v2 app from scratch; covers CLI version detection, `cartesi create`, and initial file layout.
- `cartesi-frontend/` - frontend-focused skill guidance for wallet, InputBox, JSON-RPC, inspect, and UI conventions.
- `cartesi-backend-core/` - language-agnostic backend core skill for `advance_state`, `inspect_state`, payload contracts, and finish-loop correctness.
- `cartesi-backend-py/` - Python-focused backend implementation skill.
- `cartesi-backend-js-ts/` - JavaScript/TypeScript-focused backend implementation skill.
- `cartesi-contracts/` - Solidity/Foundry skill for InputBox integrations, asset portals, voucher execution, and L1 wiring.
- `cartesi-local-dev/` - local and forked development workflow: build, run, advance/inspect, devnet tokens, and forked chain testing.
- `cartesi-deploy/` - deployment and operations skill for self-hosted environments; covers the full Docker Compose flow.
- `cartesi-jsonrpc/` - query a running node via the JSON-RPC 2.0 API (port 10011); full method reference with TypeScript types.
- `cartesi-debug/` - diagnose and fix errors across the full Cartesi stack, organised by layer and symptom.

Each skill is defined in a `SKILL.md` file and can include supporting docs (for example `DESIGN.md`).

## How to use in AI agents

Most agent systems use a "skill" file as instruction context during generation.

1. Pick the skill folder that matches the task.
2. Load and include that folder's `SKILL.md` in the agent context.
3. If present, also include supporting docs (for example `DESIGN.md`) as additional context.
4. Ask the agent to implement changes in the target app repo while following the loaded skill guidance.

## Suggested usage pattern

- **New to Cartesi / unsure where to start:** include `cartesi-workflow/SKILL.md` first — it routes to the right skill for each phase.
- **Starting a new project:** include `cartesi-scaffold/SKILL.md`.
- **Backend (language-agnostic core):** include `cartesi-backend-core/SKILL.md`.
- **Backend (Python):** include `cartesi-backend-py/SKILL.md` (plus backend core).
- **Backend (JS/TS):** include `cartesi-backend-js-ts/SKILL.md` (plus backend core).
- **Frontend tasks:** include `cartesi-frontend/SKILL.md`.
- **L1 contract tasks:** include `cartesi-contracts/SKILL.md`.
- **Local development:** include `cartesi-local-dev/SKILL.md`.
- **Deployment tasks:** include `cartesi-deploy/SKILL.md`.
- **Querying node outputs programmatically:** include `cartesi-jsonrpc/SKILL.md`.
- **Debugging errors or unexpected behaviour:** include `cartesi-debug/SKILL.md`.
- **Custom UI requests:** include `cartesi-frontend/DESIGN.md` to override default styling guidance.

## Notes

- Skills are guidance, not executable code.
- Keep skills updated as Cartesi packages, APIs, and best practices evolve.
