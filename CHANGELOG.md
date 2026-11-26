# Changelog

All notable changes to this project are documented in this file.

## [Unreleased] — rollups-node alpha.14 / CLI alpha.42

Skills re-based on **Cartesi CLI `2.0.0-alpha.42`** (SDK `0.12.0-alpha.45`),
**rollups-node `v2.0.0-alpha.14`**, **rollups contracts 3.0.0-alpha.10**,
**Dave (PRT) 3.0.0-alpha.6** and **cartesi-machine 0.22**. This stack is not
backward compatible: applications must be rebuilt and redeployed, and the node
needs a fresh database.

### Corrected (previous guidance fails on this stack)

- **Contract addresses**: `cartesi-contracts/cartesi-rollups-3.0.0-alpha.6.json`
  replaced by `cartesi-rollups-3.0.0-alpha.10.json` (new InputBox, factories,
  portals; `DaveAppFactory` per chain). Verified for Ethereum Sepolia and the
  local devnet only.
- **Portal functions** renamed in contracts v3: `depositErc20Tokens`,
  `depositErc721Token`, `depositSingleErc1155Token`, `depositBatchErc1155Token`;
  artifact names `Erc20Portal`, … . Devnet test tokens have open `mint`
  functions (no pre-minted owner).
- **JSON-RPC**: every result is `{ data }` (lists add `pagination`); `params`
  must be an object. Examples and TypeScript snippets fixed.
- **Application status**: `OK | FAILED | DIVERGED | CORRUPTED | GUEST_EXCEPTION
  | MACHINE_HALTED | MCYCLE_OVERFLOW | UNEXPECTED_YIELD | INVALID_OUTPUTS_ROOT`.
  `INOPERABLE` and `FORECLOSED` removed; foreclosure is read from
  `foreclose_block`.
- **Input status**: `OVERFLOW`, `UNEXPECTED_YIELD`, `INVALID_OUTPUTS_ROOT`
  replace the `*_LIMIT_EXCEEDED` values.
- **PRT** applications do pass through `CLAIM_STAGED`.
- **Inspect** takes the raw request body (`application/octet-stream`), not a
  JSON wrapper.
- **`cartesi run`** serves `/anvil`, `/rpc` and `/inspect` on one port
  (default `6751`; `/explorer` with `--services explorer`); `localhost:8545` is
  not exposed.
- **`cartesi-backend-js-ts`** rewritten for the `@cartesi/rollup` package the
  JS/TS templates use (no rollup HTTP server, no `rollup-init`, pnpm). The old
  `/finish` + `openapi-fetch` guidance did not run on a freshly scaffolded
  project.
- `ENTRYPOINT ["rollup-init"]` is no longer described as a v1.5-only marker.
- **`cartesi send` / `cartesi deposit`** exist in v2.0-alpha and default to
  devnet account 8; `send` takes `--encoding` (no `--hex`) (account 0 is the node's claim
  signer).
- **Node settings that no longer exist** (the first was already gone in
  alpha.12 but still appeared in the skills): `CARTESI_BLOCKCHAIN_WS_ENDPOINT`,
  `CARTESI_CLAIMER_MAX_ACCEPT_ATTEMPTS`; `pending` default block.
  `app remove` takes `--allow-abandonment [--yes]` (the `--force` flag the
  skills showed does not exist); `contract` reads are subcommands that take an
  application address; `newAuthority` takes a claim staging
  period.

### Added

- PRT (Dave) coverage: `deploy application --prt`, sentries, the `prt` service
  and its separate signer, `cartesi run --prt`, PRT JSON-RPC methods.
- New JSON-RPC surface: `cartesi_getNodeInfo`, `getEpochByVirtualIndex`,
  executed/pending output counts, batch requests, limits and error codes,
  `descending` / `from` / `to` / `executed` filters.
- Emergency flows: `refund` for unfinalized deposits, machine-tool flags,
  `cartesi foreclose` / `withdraw` / `refund` for local runs, accounts drive in
  `cartesi.toml`.
- New CLI commands and behaviour: `cartesi logs`, `status`, `address-book
  <name>` / `--json`, strict `cartesi.toml` keys, extra templates.
- `cartesi-debug`: new symptoms for the changes above.

### Skill versions

`cartesi-workflow` 0.2.0, `cartesi-scaffold` 0.2.0, `cartesi-local-dev` 0.2.0,
`cartesi-frontend` 0.3.0, `cartesi-jsonrpc` 0.3.0, `cartesi-debug` 0.3.0,
`cartesi-contracts` 0.4.0, `cartesi-deploy` 0.4.0, `cartesi-backend-js-ts`
0.3.0. `cartesi-backend-core` gained a note on the two transports;
`cartesi-backend-py` is unchanged.

### Not verified

- Public testnet or mainnet deployment: all live runs used a local devnet.
- A PRT epoch through to `CLAIM_ACCEPTED`: the PRT deploy and the node's
  commitment were observed, the tournament settlement was not waited for.
- The emergency flows (foreclose, prove-drive-root, withdraw, refund) and the
  machine-tool proof generation were checked against source and `--help`, not
  executed.
- Contract addresses for chains other than Ethereum Sepolia and the devnet.

### Tested (2026-11-25, local devnet, CLI alpha.42)

- `cartesi create`, `build`, `run`, `status`, `logs`, `address-book`, `send`
  (all encodings), `deposit` (ether, erc20, erc721); `cast` mint and deposits
  for ERC-20, ERC-721 and ERC-1155 with the v3 function names (the v2 name
  reverts).
- JSON-RPC result envelope, parameter forms, enums, error codes, batch
  requests; inspect with raw and JSON-wrapped bodies.
- The patched Mugen compose on alpha.45 images: Authority deploy with claim
  staging (`CLAIM_STAGED` → `CLAIM_ACCEPTED` observed), voucher validate and
  execute, `deploy authority`, `deploy quorum`, `deploy application --prt` with
  sentries, app disable and removal.
- A backend written from `cartesi-backend-js-ts`, on the host mock driver and
  inside the machine.

## [0.1.0] — 2026-05-12

First release of reusable **Cartesi Rollups v2** skill packs for AI coding agents.

### Compatibility & baseline deps

- Written for **Cartesi Rollups v2**, with docs aligned to **Cartesi CLI `2.0.0-alpha.34`**, **rollups-node `2.0.0-alpha.11`**, and **rollups contracts v2.2.0** (resolve live addresses per chain as each skill describes).
- **`cartesi-deploy`** follows the Mugen-Builders self-hosted Compose setup: [Mugen-Builders/deployment-setup-v2.0](https://github.com/Mugen-Builders/deployment-setup-v2.0) (`compose.local.yaml`).

### Skills in this release

- **`cartesi-workflow`**, **`cartesi-scaffold`**, **`cartesi-backend-core`**, **`cartesi-backend-py`**, **`cartesi-backend-js-ts`**, **`cartesi-contracts`**, **`cartesi-frontend`** (optional **`cartesi-frontend/DESIGN.md`** for UI tokens), **`cartesi-local-dev`**, **`cartesi-deploy`**, **`cartesi-jsonrpc`**, **`cartesi-debug`**.

### Notes

- Skills are guidance only; verify CLI output, ports, and image tags on your machine.
- The same skill packs are published live at [https://skills.mugen.builders/](https://skills.mugen.builders/).
