# cartesi-skills

Reusable skill packs for AI coding agents building with Cartesi Rollups.

## What is in this repo

- `cartesi-rollups-backend/` - backend-focused skill guidance for Rollups app logic and runtime workflows.
- `cartesi-rollups-frontend/` - frontend-focused skill guidance for wallet, InputBox, JSON-RPC, inspect, and UI conventions.

Each skill is defined in a `SKILL.md` file and can include supporting docs (for example `DESIGN.md`).

## How to use in AI agents

Most agent systems use a "skill" file as instruction context during generation.

1. Pick the skill folder that matches the task.
2. Load and include that folder's `SKILL.md` in the agent context.
3. If present, also include supporting docs (for example `DESIGN.md`) as additional context.
4. Ask the agent to implement changes in the target app repo while following the loaded skill guidance.

## Suggested usage pattern

- **Backend tasks:** include `cartesi-rollups-backend/SKILL.md`.
- **Frontend tasks:** include `cartesi-rollups-frontend/SKILL.md`.
- **Custom UI requests:** include `cartesi-rollups-frontend/DESIGN.md` to override default styling guidance.

## Notes

- Skills are guidance, not executable code.
- Keep skills updated as Cartesi packages, APIs, and best practices evolve.
