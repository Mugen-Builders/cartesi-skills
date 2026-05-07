---
name: cartesi-rollups-v2-app

description: >-
  Build and operate Cartesi Rollups v2 applications on any domain: scaffold with
  Cartesi CLI, implement advance and inspect handlers, run local or forked
  devnets, optionally wire L1 contracts to InputBox, add frontend harnesses,
  deploy a self-hosted rollups node for testnet-style testing, and troubleshoot
  common issues. Language-agnostic backend guidance with JS/Foundry/React called
  out as one stack option.
---

# Cartesi Rollups v2 application (generic)

## Goal

Use this skill when creating or evolving **any** Cartesi Rollups v2 dApp, including:

- Backend app logic (`advance_state` + `inspect_state`);
- Local execution via `cartesi build` / `cartesi run` (plain local or forked RPC);
- **Self-hosted rollups node** (development / testnet-style testing; not production-grade);
- Optional **on-chain components** that call `InputBox.addInput(application, payload)`;
- Frontend or CLI tools to submit inputs and query inspect;
- Sanity checks and troubleshooting across chains.

Adapt names, payload formats, and routes to **your** project. This file does not assume a particular business domain (games, oracles, DeFi, etc.).

Official reference (self-hosted): [Self-hosted deployment](https://docs.cartesi.io/cartesi-rollups/2.0/deployment/self-hosted/).

## Preconditions

- Docker running.
- Cartesi CLI installed (`cartesi --version`).
- If you use smart contracts: Foundry or your chosen toolchain (`forge --version` or equivalent).
- If you use a web UI: Node.js/npm (or the stack your repo uses).

## Core mental model

- **Advance** (`advance_state`): write path. Can mutate state and emit notices, vouchers, and reports.
- **Inspect** (`inspect_state`): read path. Should not mutate state; typically returns **reports** only.
- **InputBox** (`addInput(app, payload)`): canonical ingress from L1 or L2 into the rollups backend when you need chain-originated bytes.
- **Local devnet** may be a fresh chain or **forked** from a public RPC for realistic contract/state.

## Phase 1 — Scaffold the app

Create from a template (example: JavaScript):

```sh
cartesi create <app-name> --template javascript
```

Replace `<app-name>` with your dApp identifier; other templates exist—follow Cartesi CLI help and your team’s choice.

Baseline `/finish` loop should:

1. Post the current `finish.status` returned by handlers (do not discard it).
2. Dispatch by your chosen `request_type` or equivalent.
3. Set `finish.status` from the handler outcome (`accept` / `reject` as your framework expects).

Do **not** hardcode a success status in the `/finish` body after real handlers exist unless your design explicitly requires it.

## Phase 2 — Implement backend logic

Recommended structure (adapt to your language):

- **Payload helpers**: hex ↔ UTF-8 (or ABI decode) consistent with your L1 and frontend.
- **Emit helpers**: `/notice`, `/report`, optional `/voucher` as needed.
- **Validation**: reject malformed advances early; keep error reasons observable in logs or reports.
- **State**: reducers or explicit state machine for your domain; document invariants.
- **Inspect**: define **stable route strings** (e.g. `stats`, `items`, `history/<n>`) and **report shapes** (JSON or binary) so clients can rely on them.

Domain-specific rules (oracles, NFTs, order books, etc.) belong **in your code and README**, not in this skill—only the **pattern** is shared: validate → reduce → expose read models via inspect.

## Phase 3 — Local run and sanity

```sh
# Example; use yarn/pnpm/npm per project
yarn build
cartesi build
cartesi run -p <port>
```

Sanity checklist (adjust field names to your app):

- Send at least one **valid** advance input.
- Call a simple inspect route you defined.
- Confirm observable state or counters changed as expected.
- Decode at least one report payload and verify structure.

If “application name is already in use”, tear down the old compose project:

```sh
docker compose -p <app-name> down
```

## Phase 4 — Frontend or CLI harness (recommended)

Typical harness features:

- Configurable **L1 RPC URL**, **chain ID**, **application address**, **InputBox** address (if used).
- **Inspect base URL** separate from L1 JSON-RPC (rollups HTTP often differs from `eth_*` endpoints).
- Way to submit **advance** payloads (JSON, hex, or file) matching your backend.
- Buttons or scripts for **inspect** routes you support.
- Display of decoded reports (and errors).

**Chain ID must match** the signer and RPC (e.g. local Anvil often `31337`; public testnets have their own IDs). Mismatches commonly surface as `invalid chain id for signer`.

Use a dev proxy (e.g. Vite) to avoid CORS issues:

- Proxy **inspect** to the Cartesi / rollups HTTP port your setup prints.
- Proxy **L1 RPC** to your node (Anvil, fork, or remote HTTP).

## Phase 5 — Optional L1 → InputBox pattern

When inputs must originate from contracts (feeds, governance, bridges, etc.):

1. Deploy a contract that knows **InputBox**, **application**, and any external protocol addresses your app needs.
2. Contract builds a **byte payload** your backend already accepts (same encoding as direct `cartesi send`).
3. Contract calls `InputBox.addInput(application, payload)`.

Hardening that applies to many apps:

- **Access control**: `onlyOwner`, roles, or allowlisted callers on sensitive functions.
- **Events**: emit metadata for off-chain indexers and debugging.
- **Immutability**: application address is often fixed at deploy—plan migrations explicitly if you change apps.

If you expose a **privileged** function (e.g. “push latest data”), document **who may call it**; `msg.sender` must match your contract’s rules or the tx reverts (custom errors are easier to decode if included in the ABI you use client-side).

## Phase 6 — Forked chain workflow

```sh
cartesi run --fork-url <RPC_URL> [--fork-block-number <N>] -p <port>
```

Notes:

- Fork state is **static** until you move the fork or resend txs; repeated reads of external oracles may return identical values—expected.
- To test **changing** data, use new blocks, different calls, or **direct advance inputs** that bypass L1.

## Phase 7 — Self-hosted deployment (Rollups node)

Per [Cartesi Rollups 2.0 — Self-hosted deployment](https://docs.cartesi.io/cartesi-rollups/2.0/deployment/self-hosted/): this runs a **Cartesi Rollups node** with Docker Compose for **development and testnet-style testing**. The docs state it is **not** production-ready (no public snapshot verification, full hardening, or production ops).

### 7.1 Prerequisites

- Cartesi CLI
- Docker Desktop 4.x (or compatible engine)
- Installation: [Installation](https://docs.cartesi.io/cartesi-rollups/2.0/development/installation/)

### 7.2 Configure `.env` (node + chain)

Set variables the compose stack expects (names follow the official guide):

```shell
BLOCKCHAIN_ID=<blockchain-id>
AUTH_KIND="private_key"
CARTESI_AUTH_PRIVATE_KEY="<funded-private-key>"
BLOCKCHAIN_WS_ENDPOINT="<ws-endpoint>"
BLOCKCHAIN_HTTP_ENDPOINT="<http-endpoint>"
CARTESI_BLOCKCHAIN_DEFAULT_BLOCK="<latest or finalized>"
```

| Variable | Role |
| -------- | ---- |
| `BLOCKCHAIN_ID` | Target chain id (e.g. Sepolia `11155111`). |
| `BLOCKCHAIN_WS_ENDPOINT` | WebSocket RPC. |
| `BLOCKCHAIN_HTTP_ENDPOINT` | HTTP RPC. |
| `AUTH_KIND` | e.g. `private_key` for dev-style auth. |
| `CARTESI_AUTH_PRIVATE_KEY` | Funded key on that chain (handle secrets safely). |
| `CARTESI_BLOCKCHAIN_DEFAULT_BLOCK` | `latest` or `finalized` per your needs. |

### 7.3 Bring up the local Rollups node

Download the official compose file into the **project root** (per docs):

```shell
curl -L https://raw.githubusercontent.com/Mugen-Builders/deployment-setup-v2.0/main/compose.local.yaml -o compose.local.yaml
```

Build the application snapshot:

```shell
cartesi build
```

Start the node:

```shell
docker compose -f compose.local.yaml --env-file .env up -d
```

### 7.4 Register the application on the node

```shell
docker compose --project-name cartesi-rollups-node \
  exec advancer cartesi-rollups-cli deploy application <app-name> /var/lib/cartesi-rollups-node/snapshot \
  --epoch-length 10 \
  --salt <salt> \
  --register
```

- `<app-name>`: your dApp name (aligned with `cartesi create` / manifest).
- `<salt>`: **unique per deployment**; do not reuse. Example:

```shell
cast keccak256 "your-unique-string"
```

**Record the application contract address** for InputBox, frontends, and any relay contracts.

### 7.5 Manual fallback (authority + app)

If automated deploy fails, the docs describe:

1. Deploy an **authority** via `cast` against `AuthorityFactory` (addresses are **chain-specific**—see “Deployed Contracts” / `cartesi address-book`).
2. Re-run `deploy application` with `--consensus <Authority-contract>`.

Example shape (replace addresses, owner, RPC, key):

```shell
cast send <AuthorityFactory-Address> "newAuthority(address,uint256)" <Application-Owner-Address> \
  10 --private-key <PRIVATE-KEY> --rpc-url <RPC-URL> \
  --json | jq -r '.logs[-1].data' | sed 's/^0x000000000000000000000000/0x/'
```

Then:

```shell
docker compose --project-name cartesi-rollups-node \
  exec advancer cartesi-rollups-cli deploy application <app-name> /var/lib/cartesi-rollups-node/snapshot \
  --epoch-length 10 \
  --consensus <Authority-contract> \
  --json
```

### 7.6 Accessing the self-hosted node (default ports)

Typical HTTP entrypoints (confirm in your compose if ports differ):

- **Inspect:** `http://localhost:10012/inspect/<app-name>`
- **JSON-RPC:** `http://localhost:10011/rpc`

Point frontend proxies or `curl` here when using the self-hosted stack instead of `cartesi run`’s single-port proxy.

### 7.7 Contract addresses on chain

InputBox, portals, AuthorityFactory, etc. are **chain-specific**. Resolve from:

- Official **Deployed Contracts** for your chain, and/or
- `cartesi address-book` when using the Cartesi dev stack.

## Troubleshooting

- **`invalid chain id for signer`**  
  Align wallet / viem chain config with the RPC’s chain id.

- **Inspect OK but state unchanged**  
  No advance processed, wrong app address, or backend restarted and lost ephemeral state. Resend input and re-check inspect.

- **Fork “stuck” values**  
  Expected with a static fork height; advance directly or advance the fork.

- **Wrong HTTP port**  
  Read URLs from `cartesi run` output or self-hosted compose; update client defaults and proxies.

- **Self-hosted `deploy application` fails**  
  Try Phase 7.5; verify funded key, `BLOCKCHAIN_ID`, and factory addresses for that chain.

- **L1 contract revert with undecoded selector**  
  Ensure the client ABI includes your contract’s **custom errors** and **latest bytecode**; look up unknown selectors on [4byte](https://www.4byte.directory/) or Sourcify while debugging.

## Agent checklist

- Scaffold with `cartesi create` (or follow existing repo layout).
- Implement advance + inspect per project spec; document payload and routes in README.
- Ensure `/finish` (or equivalent) reflects real handler status.
- Run `cartesi build` / `cartesi run` and execute a minimal advance + inspect roundtrip.
- Add harness (frontend or scripts) with correct chain ID and URLs for L1 vs rollups.
- Optional: L1 contracts + deploy scripts + tests; wire InputBox and app address from deployment.
- Optional testnet-style: `.env`, `compose.local.yaml`, `docker compose up`, `cartesi-rollups-cli deploy application … --register`, persist app address and node URLs.

## Reusable command patterns

Advance (string payload—replace body with your JSON):

```sh
cartesi send --encoding string '<your-json-payload>'
```

Inspect (replace host, port, app name, and hex payload for your route):

```sh
curl -s -X POST "http://127.0.0.1:<port>/inspect/<app-name>" \
  -H "Content-Type: application/json" \
  -d '{"payload":"0x..."}'
```

Foundry script deployment (when your repo provides a script):

```sh
forge script script/<YourScript>.s.sol:<ContractName> \
  --rpc-url <RPC> \
  --broadcast
```
