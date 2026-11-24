---
name: cartesi-scaffold
version: 0.2.0
description: >-
  Scaffold a new Cartesi Rollups v2 application from scratch using the Cartesi
  CLI. Use this whenever a user wants to create a new Cartesi dApp, set up the
  project structure, choose a language template, or understand the baseline
  file layout before writing any backend logic. Also handles detecting which
  CLI version (v1.5 or v2.0-alpha) the user has installed and identifying
  project version from the Dockerfile. Triggers on: "create a new Cartesi app",
  "scaffold", "cartesi create", "new dApp", "start a Cartesi project",
  "initialize a Cartesi application", "which version", "what CLI do I have".
---

## Skill Version

| Skill              | Version | Cartesi Rollups target               | Last updated |
| ------------------ | ------- | ------------------------------------ | ------------ |
| `cartesi-scaffold` | `0.2.0` | v2.0-alpha — Cartesi CLI `2.0.0-alpha.42` (also recognises CLI v1.5 projects) | Nov 2026     |

> If you are using a newer version of Cartesi Rollups, check whether an updated skill is available. Scaffold steps are generally stable across minor CLI bumps, but always verify with `cartesi --version`.

# Cartesi Rollups v2 — Scaffold a New Application

## Goal

Set up a clean, correctly structured Cartesi Rollups v2 project ready for
backend logic implementation. This skill covers only the scaffold phase:
project creation, template selection, and understanding the initial file
layout. Stop here — do not implement handlers. That is the job of the
`cartesi-backend-core` skill (plus `cartesi-backend-py` or
`cartesi-backend-js-ts` for the chosen language).

---

## Step 0 — Detect CLI version (ALWAYS do this first)

The Cartesi ecosystem currently has two CLI versions in active use. Commands
and capabilities differ between them. Always identify which version the user
has before running any commands.

```sh
cartesi --version
```

| Output          | Version        | Key differences                                                                                      |
| --------------- | -------------- | ---------------------------------------------------------------------------------------------------- |
| `1.5.x`         | **v1.5**       | Supports `cartesi deploy`; uses `MACHINE_EMULATOR_TOOLS_VERSION` in Dockerfile                       |
| `2.0.0-alpha.x` | **v2.0-alpha** | `cartesi deploy` is only a deprecated stub; uses `MACHINE_GUEST_TOOLS_VERSION` in Dockerfile; deployment via Docker Compose |

For rollups-node `v2.0.0-alpha.14`, use CLI **`2.0.0-alpha.42`** or newer: it
builds with SDK `0.12.0-alpha.45` (cartesi-machine 0.22). Images built by older
CLIs cannot be loaded by that node.

### Identify project version from an existing Dockerfile

If working with an existing project rather than creating a new one:

```sh
grep -E "MACHINE_EMULATOR_TOOLS_VERSION|MACHINE_GUEST_TOOLS_VERSION" Dockerfile
```

| Match                                                      | Project targets        |
| ---------------------------------------------------------- | ---------------------- |
| `MACHINE_EMULATOR_TOOLS_VERSION=0.14.1`                    | Cartesi **v1.5**       |
| `MACHINE_GUEST_TOOLS_VERSION=0.18.0` (current template; lower values mean an older scaffold) | Cartesi **v2.0-alpha** |

---

## Step 1 — Preconditions

Verify before scaffolding:

```sh
cartesi --version      # Cartesi CLI must be installed
docker --version       # Docker Desktop 4.x must be running
node --version         # Node.js LTS required for JS/TS templates
```

If `cartesi` is not found, direct the user to the official installation guide:
https://docs.cartesi.io/cartesi-rollups/2.0/development/installation/

Install Cartesi CLI:

```sh
npm install -g @cartesi/cli
# or on macOS with Homebrew:
brew install cartesi/tap/cartesi

# Verify system readiness
cartesi doctor
```

---

## Step 2 — Choose a template

Cartesi CLI supports multiple language templates. Ask the user which language
they prefer if not already specified:

| Template   | Flag                    | Best for                        |
| ---------- | ----------------------- | ------------------------------- |
| JavaScript | `--template javascript` | Quick prototypes, Node.js teams |
| TypeScript | `--template typescript` | Typed JS backends               |
| Python     | `--template python`     | Data-heavy or ML workloads      |
| Rust       | `--template rust`       | Performance-critical apps       |
| Go         | `--template go`         | Go teams                        |
| C++        | `--template cpp`        | Systems-level or legacy code    |
| C++ (low level) | `--template cpp-low-level` | Lower-level C++ variant (see the template README) |
| Java       | `--template java`       | JVM teams                       |
| Ruby       | `--template ruby`       | Ruby teams                      |
| Lua        | `--template lua`        | Lightweight scripting           |

`--template` is mandatory. Templates are downloaded from the
`prerelease/sdk-12` branch of `cartesi/application-templates`; `-b <branch>`
selects another branch.

## Step 3 — Create the project

```sh
cartesi create <app-name> --template <template>
```

Replace `<app-name>` with your dApp identifier (lowercase, hyphens OK).
Example:

```sh
cartesi create my-dapp --template javascript
```

This creates a `my-dapp/` directory with the full project scaffold.

## Step 4 — Understand the project structure

After `cartesi create`, the project layout (JavaScript example) looks like:

```
my-dapp/
├── .cartesi/           # Build artifacts (generated — do not edit)
│   └── image/          # Cartesi Machine image output after `cartesi build`
├── src/
│   └── index.js        # Main entry point — advance + inspect handlers live here
├── package.json        # Node dependencies (@cartesi/rollup), pnpm pinned via packageManager
├── pnpm-lock.yaml
├── pnpm-workspace.yaml
├── rolldown.config.js  # Bundler configuration
├── Dockerfile          # Defines the Cartesi Machine Linux environment
├── README.md
└── cartesi.toml        # Optional build/run configuration (not generated — create if needed)
```

The Python template, for comparison, is `Dockerfile`, `dapp.py`,
`requirements.txt`, `README.md`.

The exact files depend on the template — list the directory after
`cartesi create` instead of assuming this layout.

Key files to understand:

- **`src/index.js`** (or equivalent): All backend logic lives here — the
  request loop, advance handler, and inspect handler. JS/TS templates use the
  `@cartesi/rollup` package (`rollup.run({ advance, inspect })`); `cpp-low-level` links libcmt directly; the remaining templates poll the
  rollup HTTP server's `/finish` endpoint.
  See `cartesi-backend-core` (plus the matching language skill,
  `cartesi-backend-py` or `cartesi-backend-js-ts`) for implementation
  guidance.
- **`Dockerfile`**: Defines the RISC-V Linux environment the Cartesi Machine
  will run. Add system dependencies here (e.g. `apt install`, pip packages).
  **This file also indicates the project's Cartesi version:**
  - v1.5 projects contain `MACHINE_EMULATOR_TOOLS_VERSION`
  - v2.0-alpha projects contain `MACHINE_GUEST_TOOLS_VERSION`
- **`cartesi.toml`** (optional): machine and drive configuration (`[machine]`,
  `[drives.*]`, nvrams, `[withdrawal]`, `[prt]`). Unknown keys make
  `cartesi build` / `run` fail with a message naming the key, so only add keys
  the CLI documents.
- **`.cartesi/image/`**: Output of `cartesi build`. Contains the machine
  snapshot. The self-hosted deployment compose mounts this directory directly.

## Step 5 — Install dependencies (JS/TS only)

```sh
cd my-dapp
corepack enable pnpm
pnpm install
```

The templates pin pnpm through `packageManager` and ship a `pnpm-lock.yaml`
that the Dockerfile installs with `--frozen-lockfile`; use pnpm so the lockfile
stays in sync. `cartesi build` installs dependencies inside the image, so this
step is only needed for editor support and host-side runs.

For other language templates, follow the template's own README for
dependency setup.

## Step 6 — Confirm scaffold is valid

Do a quick build to verify the scaffold compiles without errors:

```sh
cartesi build
```

This compiles the application into a Cartesi Machine image at `.cartesi/image/`.
A successful build confirms the scaffold is ready for handler implementation.

Expected output ends with something like:

```
Manual yield rx-accepted (1) (0x000020 data)
Cycles: <n>
<n>: 0x<machine hash>
Storing machine: please wait
```

`cartesi hash` then prints the template hash of the stored image.

If the build fails, check:

- Docker is running
- The Dockerfile installs all required system packages
- Language runtime version matches the Dockerfile base image

> **v2.0-alpha note**: `cartesi deploy` is not a working deployment path in
> the v2.0-alpha CLI (in alpha.42 it is a deprecated stub that points to
> `cartesi-rollups-cli`). Deployment is done via Docker Compose — see the
> `cartesi-deploy` skill.

## Agent Output

After completing this skill, report back to the user with:

- CLI version detected (`cartesi --version` output)
- Project version identified (v1.5 or v2.0-alpha, with the Dockerfile marker found)
- App name and language template chosen
- Confirmation that `cartesi build` succeeded and snapshot exists at `.cartesi/image/`
- The full project directory tree (one level deep)
- Any build warnings or Dockerfile issues encountered
- Clear note that `cartesi deploy` is not the deployment path in v2.0-alpha — next step is `cartesi-deploy` skill for deployment

## Routing Guide

| What the user wants to do next           | Go to skill                                                            |
| ---------------------------------------- | ---------------------------------------------------------------------- |
| Write advance and inspect handlers       | `cartesi-backend-core` + `cartesi-backend-py` / `cartesi-backend-js-ts` |
| Run the app locally and send test inputs | `cartesi-local-dev`                                                    |
| Wire an L1 contract to InputBox          | `cartesi-contracts`                                                    |
| Deploy to a testnet or self-hosted node  | `cartesi-deploy`                                                       |
| Debug a build failure                    | `cartesi-debug`                                                        |

## Resources

> **Conflict rule**: If any resource below contradicts guidance in this skill,
> report the contradiction to the user and follow the skill's instructions.

- [Cartesi Rollups v2 Installation](https://docs.cartesi.io/cartesi-rollups/2.0/development/installation/) — CLI setup and system requirements
- [Cartesi Rollups v2 Overview](https://docs.cartesi.io/cartesi-rollups/2.0/) — architecture and mental model
- [Cartesi CLI create command](https://docs.cartesi.io/cartesi-rollups/2.0/development/cli-commands/) — available templates and flags
- [Cartesi Rollups GitHub](https://github.com/cartesi/rollups-node) — source for the rollups node

## What comes next

| Next task                            | Skill to use                                                            |
| ------------------------------------ | ----------------------------------------------------------------------- |
| Implement advance/inspect handlers   | `cartesi-backend-core` + `cartesi-backend-py` / `cartesi-backend-js-ts` |
| Run locally and test                 | `cartesi-local-dev`                                                     |
| Wire L1 contracts to InputBox        | `cartesi-contracts`                                                     |
| Deploy to a self-hosted rollups node | `cartesi-deploy`                                                        |

## Agent checklist

- [ ] Ran `cartesi --version` and identified CLI version (v1.5 or v2.0-alpha)
- [ ] If existing project: checked Dockerfile for `MACHINE_EMULATOR_TOOLS_VERSION` vs `MACHINE_GUEST_TOOLS_VERSION`
- [ ] Confirmed Docker is running (`docker info`)
- [ ] Confirmed language template with user (or inferred from context)
- [ ] Ran `cartesi create <app-name> --template <template>`
- [ ] Verified project directory structure and Dockerfile version marker
- [ ] Ran `cartesi build` and confirmed snapshot at `.cartesi/image/`
- [ ] Informed user that `cartesi deploy` is only a deprecated stub in v2.0-alpha — use `cartesi-deploy` skill for deployment
