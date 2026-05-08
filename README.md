# cartesi-skills

Reusable skill packs for AI coding agents building with Cartesi Rollups.

## What is in this repo

- `cartesi-frontend/` - frontend-focused skill guidance for wallet, InputBox, JSON-RPC, inspect, and UI conventions.
- `cartesi-backend-core/` - language-agnostic backend core skill for `advance_state`, `inspect_state`, payload contracts, and finish-loop correctness.
- `cartesi-python-backend/` - Python-focused backend implementation skill.
- `cartesi-js-backend/` - JavaScript/TypeScript-focused backend implementation skill.
- `cartesi-l1-contracts/` - Solidity/Foundry skill for InputBox integrations and L1 wiring.
- `cartesi-local-dev/` - local and forked development workflow and troubleshooting skill.
- `cartesi-deploy/` - deployment and operations skill for local/forked/self-hosted environments.

Each skill is defined in a `SKILL.md` file and can include supporting docs (for example `DESIGN.md`).

## How to use in AI agents

Most agent systems use a "skill" file as instruction context during generation.

1. Pick the skill folder that matches the task.
2. Load and include that folder's `SKILL.md` in the agent context.
3. If present, also include supporting docs (for example `DESIGN.md`) as additional context.
4. Ask the agent to implement changes in the target app repo while following the loaded skill guidance.

## Suggested usage pattern

- **Backend (language-agnostic core):** include `cartesi-backend-core/SKILL.md`.
- **Backend (Python):** include `cartesi-python-backend/SKILL.md` (plus backend core).
- **Backend (JS/TS):** include `cartesi-js-backend/SKILL.md` (plus backend core).
- **Frontend tasks:** include `cartesi-frontend/SKILL.md`.
- **L1 contract tasks:** include `cartesi-l1-contracts/SKILL.md`.
- **Local dev/fork debugging:** include `cartesi-local-dev/SKILL.md`.
- **Deployment tasks:** include `cartesi-deploy/SKILL.md`.
- **Custom UI requests:** include `cartesi-frontend/DESIGN.md` to override default styling guidance.

## Notes

- Skills are guidance, not executable code.
- Keep skills updated as Cartesi packages, APIs, and best practices evolve.
