---
name: cartesi-deploy
version: 0.3.0
description: >-
  Deploy a Cartesi Rollups application to a self-hosted rollups node for
  testnet or production-style deployment using Docker Compose. Covers contracts
  v3 deploy flags (claim-staging-period, withdrawal-config), quorum deploy,
  operator CLI (foreclose, prove-drive-root, withdraw), application lifecycle
  (enabled/status), and node ops. Triggers on: "deploy", "self-hosted",
  "testnet", "rollups node", "docker compose", "compose.local", "register
  application", "withdrawal config", "claim staging", "foreclose", "quorum
  deploy", "Sepolia", "Mugen-Builders".
---

## Skill Version

| Skill            | Version | Cartesi Rollups target | Contract suite              | Node / runtime pin                          | Compose setup       | Last updated |
| ---------------- | ------- | ---------------------- | --------------------------- | ------------------------------------------- | ------------------- | ------------ |
| `cartesi-deploy` | `0.3.0` | contracts v3           | `cartesi-rollups` 3.0.0-alpha.6 | `rollups-node` **v2.0.0-alpha.12** → `cartesi/rollups-runtime:0.12.0-alpha.41` | Mugen-Builders v2.0 | Jul 2026     |

> Confirm the `cartesi-rollups-runtime` image tag in `compose.local.yaml` matches
> both the **node binary** under test and the **contract suite** under test.
> Contract addresses and lifecycle semantics: **`cartesi-contracts`**. Do not
> reuse a v2-alpha database against a v3 node without an explicit migration plan.
> When bumping node versions (e.g. alpha.11 → alpha.12), wipe the DB volume.

### Version mapping (node alpha.12)

Cartesi uses different version schemes across artifacts. For **node alpha.12**:

| Artifact | Version |
| -------- | ------- |
| `rollups-node` release tag | `v2.0.0-alpha.12` |
| Docker runtime / database image | `0.12.0-alpha.40` or newer (use `0.12.0-alpha.41`) |
| `cartesi-rollups-cli` inside runtime | `2.0.0-alpha.12` |
| Cartesi CLI / SDK (host build) | `@cartesi/sdk@0.12.0-alpha.40` or newer |
| Contracts v3 suite | `cartesi-rollups` **3.0.0-alpha.6** (unchanged) |

Verify after starting the node:

```sh
docker compose -f compose.local.yaml exec advancer cartesi-rollups-node --version
# expect: cartesi-rollups-node version 2.0.0-alpha.12

docker compose -f compose.local.yaml exec advancer cartesi-rollups-cli --version
# expect: cartesi-rollups-cli version 2.0.0-alpha.12
```

> **Mugen-Builders caveat**: The upstream `compose.local.yaml` currently ships
> `0.12.0-alpha.39` images, which bundle **node alpha.11**. Always bump runtime
> and database images to `0.12.0-alpha.41` (or at least `0.12.0-alpha.40`) before
> starting the node when targeting alpha.12.

# Cartesi Rollups — Self-Hosted Node Deployment

## Goal

Deploy a Cartesi Rollups v2 application to a self-hosted rollups node using
the official Docker Compose setup. This is the authoritative deployment guide
for going beyond `cartesi run` — targeting testnets or production-style
environments.

Official references:

- Self-hosted deployment:
  https://docs.cartesi.io/cartesi-rollups/2.0/deployment/self-hosted/

- Compose setup source:
  https://github.com/Mugen-Builders/deployment-setup-v2.0

> **Scope note**: This is suited for development and testnet-style testing.
> Full production hardening (HA, monitoring, public snapshot verification)
> is out of scope.

---

## Step 0 — Detect CLI version and project version

Before running any commands, the agent must detect which CLI the user has
and which version of Cartesi their project targets. These two things
determine the correct build command and compatibility expectations.

### Detect CLI version

```sh
cartesi --version
```

| Output contains | CLI version | Notes                                          |
| --------------- | ----------- | ---------------------------------------------- |
| `1.5.x`         | v1.5        | Has `cartesi deploy` command (deprecated path) |
| `2.0.0-alpha`   | v2.0-alpha  | No `cartesi deploy`; use compose-based deploy  |

For node alpha.12 deployments, prefer host CLI `@cartesi/sdk@0.12.0-alpha.40` or
newer so `cartesi build` matches the runtime emulator and guest tools bundled in
`cartesi/rollups-runtime:0.12.0-alpha.41`.

### Detect project version from Dockerfile

```sh
grep -E "MACHINE_EMULATOR_TOOLS_VERSION|MACHINE_GUEST_TOOLS_VERSION" Dockerfile
```

| Match found                                            | Project targets    |
| ------------------------------------------------------ | ------------------ |
| `MACHINE_EMULATOR_TOOLS_VERSION=0.14.1`                | Cartesi v1.5       |
| `MACHINE_GUEST_TOOLS_VERSION=0.17.2` (or higher alpha) | Cartesi v2.0-alpha |

Both v1.5 and v2.0-alpha projects build with the same command:

```sh
cartesi build
```

Both versions produce the machine snapshot at `.cartesi/image/`.

### Quick Dockerfile fingerprints

| Signal in Dockerfile               | Version                          |
| ---------------------------------- | -------------------------------- |
| `MACHINE_EMULATOR_TOOLS_VERSION`   | v1.5                             |
| `MACHINE_GUEST_TOOLS_VERSION`      | v2.0-alpha                       |
| `cartesi/python:3.10-slim-jammy`   | v1.5 base                        |
| `cartesi/python:3.13.2-slim-noble` | v2.0-alpha base                  |
| `Multi-stage build `               | v2.0-alpha pattern               |
| `APT_UPDATE_SNAPSHOT=...`          | v2.0-alpha (reproducible builds) |
| `ENTRYPOINT ["rollup-init"]`       | v1.5                             |

> **Alpha note**: Guest tools version, base image tags, and APT snapshot dates
> continue to evolve. Always check the Cartesi CLI release notes for the latest
> canonical Dockerfile template before deploying a v2.0-alpha project.

---

## Step 1 — Prerequisites

```sh
docker --version          # Docker Desktop 4.x required
docker compose version    # Docker Compose plugin required
cartesi --version
```

The application must already be scaffolded (`cartesi-scaffold`) and have
backend handlers implemented (`cartesi-backend-core` plus
`cartesi-backend-py` or `cartesi-backend-js-ts`).

---

## Step 2 — Build the Cartesi Machine image

Build the application snapshot before starting the node:

```sh
cartesi build
```

This produces the machine snapshot at `.cartesi/image/`. The compose setup
mounts this directory directly into the advancer container — the node serves
whatever snapshot is in `.cartesi/image/` at startup.

Confirm the build succeeded:

```sh
ls .cartesi/image/
# Should contain: config.json, hash, 000000000000000.v8.f1048576.uarch
```

---

## Step 3 — Download the compose setup

Download the official Mugen-Builders deployment compose file into the
**project root** (alongside `.cartesi/`):

```sh
curl -L \
  https://raw.githubusercontent.com/Mugen-Builders/deployment-setup-v2.0/main/compose.local.yaml \
  -o compose.local.yaml
```

This compose file defines 6 services that together form the Cartesi Rollups
Node. **Do not use the downloaded file as-is** — patch it for node alpha.12
before starting (see below).

| Service       | Typical role                            |
| ------------- | --------------------------------------- |
| `database`    | Postgres — shared state bus             |
| `evm-reader`  | Reads L1 events (inputs, foreclosure, withdrawals) |
| `advancer`    | Runs Cartesi Machine; inspect           |
| `validator`   | Computes epoch claims + proofs          |
| `claimer`     | Submit, stage, accept claims            |
| `jsonrpc-api` | Query API                               |

The advancer mounts `.cartesi/image/` from the project root as the
application snapshot:

```yaml
volumes:
  - .cartesi/image:/var/lib/cartesi-rollups-node/snapshot/
```

### Patch compose for node alpha.12

After downloading, apply these changes to `compose.local.yaml`:

**1. Bump runtime and database images** (upstream ships alpha.39 = node alpha.11):

```yaml
# database service
image: cartesi/rollups-database:0.12.0-alpha.41

# all other services (evm-reader, advancer, validator, claimer, jsonrpc-api)
image: cartesi/rollups-runtime:0.12.0-alpha.41
```

**2. Remove WebSocket endpoint** — `CARTESI_BLOCKCHAIN_WS_ENDPOINT` is **not
supported** by rollups-node alpha.12. Delete the line from the shared `x-env`
block:

```yaml
# REMOVE this line from x-env:
# CARTESI_BLOCKCHAIN_WS_ENDPOINT: ${BLOCKCHAIN_WS_ENDPOINT}
```

**3. Set default block to `latest`** (required for testnet deployments that
track the chain tip):

```yaml
# In x-env:
CARTESI_BLOCKCHAIN_DEFAULT_BLOCK: "latest"

# evm-reader command:
command: cartesi-rollups-evm-reader --default-block latest
```

**4. Override v3 contract factory addresses** in `x-env` from
[`cartesi-contracts/cartesi-rollups-3.0.0-alpha.6.json`](../cartesi-contracts/cartesi-rollups-3.0.0-alpha.6.json).
The downloaded Mugen compose defaults are the **wrong** suite for v3 α.6 (e.g.
`InputBox` `0x1b51e…` vs `0x346B3d…`). Same addresses apply on mainnet and
testnet chains listed in that manifest — only `BLOCKCHAIN_ID` and RPC change.

---

## Step 4 — Create the `.env` file

The compose file reads configuration from a `.env` file in the project root.
Create it with the following variables:

```sh
# .env
AUTH_KIND=private_key
CARTESI_AUTH_PRIVATE_KEY=<your-funded-private-key-hex>
BLOCKCHAIN_ID=<chain-id>
BLOCKCHAIN_HTTP_ENDPOINT=<https-rpc-url>
CARTESI_BLOCKCHAIN_DEFAULT_BLOCK=latest
```

> **alpha.12 change**: Do **not** set `BLOCKCHAIN_WS_ENDPOINT` or
> `CARTESI_BLOCKCHAIN_WS_ENDPOINT`. WebSocket RPC was removed in rollups-node
> alpha.12; the EVM reader uses HTTP polling only.

For testnet Authority + withdrawal testing (e.g. `erc20-withdrawal-app`), use
`CARTESI_BLOCKCHAIN_DEFAULT_BLOCK=latest` so the node tracks the chain tip.
Use `finalized` only when you explicitly need reorg-safe block anchoring.

### Values by target network

| Network          | `BLOCKCHAIN_ID` | Example RPC (use your own key)                |
| ---------------- | --------------- | --------------------------------------------- |
| Ethereum Sepolia | `11155111`      | `https://sepolia.infura.io/v3/<KEY>`          |
| Optimism Sepolia | `11155420`      | `https://opt-sepolia.g.alchemy.com/v2/<KEY>`  |
| Arbitrum Sepolia | `421614`        | `https://arbitrum-sepolia.infura.io/v3/<KEY>` |
| Base Sepolia     | `84532`         | `https://sepolia.base.org`                    |
| Local Anvil      | `31337`         | `http://host.docker.internal:8545`            |

Mainnet chains using the same v3 α.6 manifest: `1`, `10`, `42161`, `8453`.

> **Security**: Never commit `.env` to version control. Add it to `.gitignore`.
> For the private key, use a funded wallet with only the minimum ETH needed
> for gas. For production, prefer `AUTH_KIND=aws` with AWS KMS.

### Contract addresses — contracts v3

Use **[`cartesi-contracts/cartesi-rollups-3.0.0-alpha.6.json`](../cartesi-contracts/cartesi-rollups-3.0.0-alpha.6.json)**
as the primary manifest. Infrastructure addresses are identical on Ethereum /
Optimism / Arbitrum / Base mainnet and Sepolia (`1`, `10`, `42161`, `8453`,
`11155111`, `11155420`, `421614`, `84532`). See **`cartesi-contracts`** for the
full table and Cannon as secondary verification.

Patch `compose.local.yaml` `x-env` or add to `.env` — **do not** trust Mugen
defaults:

```sh
# .env — values from cartesi-rollups-3.0.0-alpha.6.json (example)
CARTESI_CONTRACTS_INPUT_BOX_ADDRESS=0x346B3df038FE9f8380071eC6514D5a83aD143939
CARTESI_CONTRACTS_AUTHORITY_FACTORY_ADDRESS=0x3C1FE01c542a88A523FF6847eD1E26176c8C4ED0
CARTESI_CONTRACTS_QUORUM_FACTORY_ADDRESS=0x1f94009389F408B8D0ADfFcF8BBDCe5552BaCa5F
CARTESI_CONTRACTS_APPLICATION_FACTORY_ADDRESS=0xC549F89cF1ca43eDDECC64Ac2208F4b283B1c483
CARTESI_CONTRACTS_SELF_HOSTED_APPLICATION_FACTORY_ADDRESS=0x6145C5996a71a379E030aEb0440df79D60833418
```

> **Local devnet**: These addresses do NOT apply to `cartesi run`. Use
> `cartesi address-book` for local Anvil.

### v3 ops environment variables

```sh
# Optional — default 5; caps repeated acceptClaim gas spend before app → FAILED
CARTESI_CLAIMER_MAX_ACCEPT_ATTEMPTS=5
```

Fund **guardian** and **gas-payer** accounts if the app may need emergency
withdrawal (`withdrawal_config` guardian must match the foreclose signer).

---

## Step 5 — Start the rollups node

Bring up all services:

```sh
docker compose -f compose.local.yaml --env-file .env up -d
```

Verify all containers are healthy:

```sh
docker compose -f compose.local.yaml ps
```

All services should show `running` or `healthy`. The `database` service has
a health check — other services wait for it before starting.

View logs:

```sh
# All services
docker compose -f compose.local.yaml logs -f

# Specific service
docker compose -f compose.local.yaml logs -f advancer
```

---

## Step 6 — Deploy and register the application

With the node running, deploy the application contracts to the blockchain
and register the application on the node using `cartesi-rollups-cli` inside
the advancer container.

### Contracts v3 — standard deploy

```sh
docker compose -f compose.local.yaml exec advancer \
  cartesi-rollups-cli deploy application <app-name> \
  /var/lib/cartesi-rollups-node/snapshot/ \
  --epoch-length 10 \
  --claim-staging-period <N> \
  --withdrawal-config-file <valid-config.json> \
  --salt <unique-32byte-hex-salt> \
  --register
```

- `--claim-staging-period <N>`: blocks that must elapse after staging before
  `acceptClaim` is valid on-chain.
- `--withdrawal-config-file`: JSON defining guardian and accounts-drive layout.
  Partial or invalid config **must** be rejected before any on-chain tx.
- Alternative: `--withdrawal-config` inline if supported by your CLI version.

Replace:

- `<app-name>`: your dApp identifier (e.g. `my-dapp`)
- `<unique-32byte-hex-salt>`: a unique value to prevent address collisions.
  Generate one with:
  ```sh
  cast keccak "$(echo $RANDOM-$(date +%s))"
  ```

> **Record the application contract address** printed after this command.
> You will need it for InputBox calls, frontends, and L1 contracts.

### Deploy quorum consensus (v3)

For multi-validator quorum tests:

```sh
docker compose -f compose.local.yaml exec advancer \
  cartesi-rollups-cli deploy quorum ...
```

See `cartesi-contracts` for quorum staging behaviour (`CLAIM_SUBMITTED` →
event-driven `CLAIM_STAGED` → `CLAIM_ACCEPTED`).

### Legacy deploy (no v3 flags)

If targeting a pre-v3 contract suite only:

```sh
docker compose -f compose.local.yaml exec advancer \
  cartesi-rollups-cli deploy application <app-name> \
  /var/lib/cartesi-rollups-node/snapshot/ \
  --epoch-length 10 \
  --salt <unique-32byte-hex-salt> \
  --register
```

### Deploy with an existing consensus contract

If you already deployed an authority consensus and want to reuse it:

```sh
docker compose -f compose.local.yaml exec advancer \
  cartesi-rollups-cli deploy application <app-name> \
  /var/lib/cartesi-rollups-node/snapshot/ \
  --consensus <authority-contract-address> \
  --register
```

### Manual authority fallback

If the automated `deploy application` fails (factory address mismatch, gas issues,
or custom consensus requirements), deploy an authority contract directly via `cast`
and pass it explicitly:

**Step 1** — Deploy an authority via the AuthorityFactory. Resolve the factory address
with `cartesi address-book` or the official Deployed Contracts page for your chain.

```sh
# newAuthority(address owner, uint256 epochLength)
# Extracts the deployed authority address from the transaction logs
AUTH_ADDRESS=$(cast send <AUTHORITY_FACTORY_ADDRESS> \
  "newAuthority(address,uint256)" \
  <APPLICATION_OWNER_ADDRESS> \
  10 \
  --private-key <PRIVATE_KEY> \
  --rpc-url <RPC_URL> \
  --json | jq -r '.logs[-1].data' | sed 's/^0x000000000000000000000000/0x/')

echo "Authority deployed at: $AUTH_ADDRESS"
```

**Step 2** — Register the application using the deployed authority:

```sh
docker compose -f compose.local.yaml exec advancer \
  cartesi-rollups-cli deploy application <app-name> \
  /var/lib/cartesi-rollups-node/snapshot/ \
  --epoch-length 10 \
  --consensus $AUTH_ADDRESS \
  --register \
  --json
```

> Record both the authority address and the application contract address —
> you will need both for L1 integrations and future re-deployments.

### Register only (contracts already deployed)

If contracts are already on-chain and you only need to register:

```sh
docker compose -f compose.local.yaml exec advancer \
  cartesi-rollups-cli app register \
  --name <app-name> \
  --address <application-contract-address> \
  --template-path /var/lib/cartesi-rollups-node/snapshot/ \
  --consensus <consensus-address>
```

---

## Step 7 — Verify the deployment

### Check the app is registered and healthy

```sh
docker compose -f compose.local.yaml exec advancer \
  cartesi-rollups-cli app list

# v3 application fields
docker compose -f compose.local.yaml exec advancer \
  cartesi-rollups-cli contract <app-name>
```

Expect `enabled`, `status`, `claim_staging_period`, `withdrawal_config` —
not the old single `state` field. Healthy normal operation:
`enabled=true`, `status=OK`.

### Confirm claim staging path (Authority)

After sending an input and closing an epoch:

```sh
docker compose -f compose.local.yaml exec advancer \
  cartesi-rollups-cli read epochs <app-name>
```

Watch: `CLAIM_SUBMITTED` → `CLAIM_STAGED` → `CLAIM_ACCEPTED`.

### Send a test advance input

```sh
docker compose -f compose.local.yaml exec advancer \
  cartesi-rollups-cli send <app-name> "hello world"
```

Requires these env vars to be set (already in compose via `.env`):
`CARTESI_CONTRACTS_INPUT_BOX_ADDRESS`, `CARTESI_BLOCKCHAIN_HTTP_ENDPOINT`,
`CARTESI_AUTH_MNEMONIC` or `CARTESI_AUTH_PRIVATE_KEY`.

### Test an inspect request

```sh
curl -s -X POST "http://localhost:10012/inspect/<app-name>" \
  -H "Content-Type: application/json" \
  -d '{"payload":"0x<hex-encoded-route>"}'
```

### Read outputs via JSON-RPC

```sh
curl -s -X POST "http://localhost:10011/rpc" \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"cartesi_listOutputs","params":[{"application":"<app-name>"}],"id":1}'
```

---

## Step 8 — Application lifecycle management

All commands run inside the advancer container via `docker compose exec`:

```sh
# Prefix for all commands:
EXEC="docker compose -f compose.local.yaml exec advancer cartesi-rollups-cli"

# List all registered apps
$EXEC app list

# Get app status
$EXEC app status <app-name>

# Disable an app (required before removal)
$EXEC app status <app-name> disabled

# Re-enable an app
$EXEC app status <app-name> enabled

# Remove an app (disable first)
$EXEC app remove <app-name>
$EXEC app remove <app-name> --force   # skip confirmation

# Read processed inputs
$EXEC read inputs <app-name>
$EXEC read inputs <app-name> 42                          # by index
$EXEC read inputs <app-name> --epoch-index 0x0 --limit 20

# Read outputs
$EXEC read outputs <app-name>
$EXEC read outputs <app-name> 42

# Read reports
$EXEC read reports <app-name>

# Read epochs
$EXEC read epochs <app-name>
$EXEC read epochs <app-name> --status OPEN

# Validate an output on-chain
$EXEC validate <app-name> <output-index>

# Execute a voucher on-chain
$EXEC execute <app-name> <output-index>
$EXEC execute <app-name> <output-index> --yes
```

---

## Step 8b — Operator CLI (contracts v3 emergency flows)

All commands run inside the advancer container. Requires compose deployment —
not available with `cartesi run` alone.

```sh
EXEC="docker compose -f compose.local.yaml exec advancer cartesi-rollups-cli"

# Guardian foreclose (signer must match withdrawal_config guardian)
$EXEC foreclose <app-name>

# Prove accounts-drive merkle root (after machine-tool generates proof)
$EXEC prove-drive-root <app-name> --proof-file drive-root-proof.json

# Withdraw account (gas payer can differ from recipient)
$EXEC withdraw <app-name> --proof-file account-proof.json

# Read recorded withdrawal rows
$EXEC read withdrawals <app-name>
$EXEC read withdrawals <app-name> <account-index>
```

Proof files come from `cartesi-rollups-machine-tool` (replay snapshot →
`prove accounts-drive`). Full sequence: `cartesi-contracts`.

After foreclosure, do **not** disable the app automatically if you still need
post-foreclosure L1 observation (drive-prove and withdrawal events).

---

## Step 9 — Manage execution parameters

```sh
EXEC="docker compose -f compose.local.yaml exec advancer cartesi-rollups-cli"

# List all parameters
$EXEC app execution-parameters list <app-name>

# Get specific parameter
$EXEC app execution-parameters get <app-name> snapshot_policy

# Set specific parameter (disable app first)
$EXEC app status <app-name> disabled
$EXEC app execution-parameters set <app-name> snapshot_policy EVERY_INPUT
$EXEC app status <app-name> enabled

# Load full parameter set from JSON
$EXEC app execution-parameters load <app-name> <<< '{
  "snapshot_policy": "EVERY_INPUT",
  "advance_inc_cycles": "0x400000",
  "advance_max_cycles": "0x3fffffffffffffff",
  "inspect_inc_cycles": "0x400000",
  "inspect_max_cycles": "0x3fffffffffffffff",
  "advance_inc_deadline": "0x2540be400",
  "advance_max_deadline": "0x29e8d60800",
  "inspect_inc_deadline": "0x2540be400",
  "inspect_max_deadline": "0x29e8d60800",
  "load_deadline": "0x45d964b800",
  "store_deadline": "0x29e8d60800",
  "fast_deadline": "0x12a05f200",
  "max_concurrent_inspects": 10
}'
```

---

## Step 10 — Stopping and restarting the node

```sh
# Stop all services (preserves data volume)
docker compose -f compose.local.yaml down

# Stop and wipe all data (full reset — required when moving v2-alpha DB to v3,
# or when bumping node versions such as alpha.11 → alpha.12)
docker compose -f compose.local.yaml down -v

# Restart after code changes:
# 1. Rebuild the machine image
cartesi build

# 2. Restart the node (advancer will pick up new snapshot on startup)
docker compose -f compose.local.yaml down
docker compose -f compose.local.yaml --env-file .env up -d
```

---

## Node service ports reference

| Service       | Port    | Purpose                        |
| ------------- | ------- | ------------------------------ |
| `evm-reader`  | `10001` | Telemetry / health             |
| `advancer`    | `10002` | Telemetry / health             |
| `advancer`    | `10012` | Inspect HTTP endpoint          |
| `validator`   | `10003` | Telemetry / health             |
| `claimer`     | `10004` | Telemetry / health             |
| `jsonrpc-api` | `10005` | Telemetry / health             |
| `jsonrpc-api` | `10011` | JSON-RPC API (query interface) |

---

## Agent Output

After completing this skill, report back to the user with:

- CLI version and project version detected
- Runtime image tag and confirmed `cartesi-rollups-node --version` inside advancer
- Target network (chain ID, RPC endpoint, default block mode)
- Application contract address (from `cartesi-rollups-cli deploy application` output) — **this must be saved**
- Authority consensus contract address (if a new one was deployed or an existing one reused)
- Salt value used for deterministic deployment (record for reproducibility)
- All six Docker Compose service statuses from `docker compose ps`
- Confirm app appears in `cartesi-rollups-cli app list` with `enabled=true`, `status=OK`
- `claim_staging_period` and `withdrawal_config` recorded from deploy
- Epoch staging observed: `CLAIM_STAGED` before `CLAIM_ACCEPTED` (Authority)
- Test advance input result: accepted / rejected and any notices produced
- Inspect endpoint URL: `http://<host>:10012`
- JSON-RPC API endpoint URL: `http://<host>:10011`
- Any issues encountered and how they were resolved

## Routing Guide

| What the user wants to do next                     | Go to skill            |
| -------------------------------------------------- | ---------------------- |
| Wire an L1 contract or oracle to the deployed app  | `cartesi-contracts` |
| Query outputs and epochs via JSON-RPC API          | `cartesi-jsonrpc`      |
| Debug node startup, advance processing, or inspect | `cartesi-debug`        |
| Run app locally first before deploying to testnet  | `cartesi-local-dev`    |
| Execute a voucher after epoch is accepted          | `cartesi-contracts` |
| Emergency foreclosure / withdrawal operator flow | `cartesi-contracts` |

## Resources

> **Conflict rule**: If any resource below contradicts guidance in this skill,
> report the contradiction to the user and follow the skill's instructions.

- [Cartesi Self-Hosted Deployment Guide](https://docs.cartesi.io/cartesi-rollups/2.0/deployment/self-hosted/) — official deployment walkthrough
- [Mugen-Builders compose setup](https://github.com/Mugen-Builders/deployment-setup-v2.0) — `compose.local.yaml` source
- [`cartesi-contracts/cartesi-rollups-3.0.0-alpha.6.json`](../cartesi-contracts/cartesi-rollups-3.0.0-alpha.6.json) — checked-in infrastructure addresses (8 chains, identical suite)
- [Cannon registry — cartesi-rollups 3.0.0-alpha.6](https://usecannon.com/packages/cartesi-rollups/3.0.0-alpha.6/84532-main/deployment/contracts) — secondary verification by chain
- [Cartesi Rollups node v2.0.0-alpha.12 release](https://github.com/cartesi/rollups-node/releases/tag/v2.0.0-alpha.12) — node binary release
- [Cartesi Rollups node GitHub](https://github.com/cartesi/rollups-node) — runtime image tags and release notes
- [Cartesi CLI SDK 0.12.0-alpha.41 release](https://github.com/cartesi/cli/releases/tag/%40cartesi%2Fsdk%400.12.0-alpha.41) — host CLI that bundles node alpha.12

## Agent checklist

- [ ] Detected CLI version with `cartesi --version` (prefer `@cartesi/sdk@0.12.0-alpha.40+` for alpha.12)
- [ ] Detected project version from Dockerfile (`MACHINE_EMULATOR_TOOLS_VERSION` vs `MACHINE_GUEST_TOOLS_VERSION`)
- [ ] Ran `cartesi build` — snapshot at `.cartesi/image/`
- [ ] Downloaded `compose.local.yaml` to project root
- [ ] Patched compose for alpha.12: bumped images to `0.12.0-alpha.41`, removed `CARTESI_BLOCKCHAIN_WS_ENDPOINT`, set `latest` default block
- [ ] Verified `cartesi-rollups-node --version` reports `2.0.0-alpha.12` inside advancer
- [ ] Started node: `docker compose -f compose.local.yaml --env-file .env up -d`
- [ ] Verified all containers healthy: `docker compose -f compose.local.yaml ps`
- [ ] Ran `cartesi-rollups-cli deploy application` inside advancer — recorded app contract address
- [ ] Sent test advance input and read output to confirm end-to-end
- [ ] Verified runtime image tag in `compose.local.yaml` matches contracts v3 suite
- [ ] Overrode compose factory addresses from `cartesi-rollups-3.0.0-alpha.6.json` — not Mugen defaults or v2.2.0
- [ ] `.env` has no `BLOCKCHAIN_WS_ENDPOINT` (alpha.12 uses HTTP only)
- [ ] On v3 deploy: passed `--claim-staging-period` and valid `--withdrawal-config-file`
- [ ] Confirmed `cartesi-rollups-cli contract <app>` shows `enabled`, `status`, v3 fields
- [ ] Observed epoch `CLAIM_STAGED` → `CLAIM_ACCEPTED` on test input (Authority)
- [ ] Fresh DB used for v3 alpha (no reused v2-alpha volume without migration; wipe on node version bump)

## What comes next

| Next task                     | Skill to use           |
| ----------------------------- | ---------------------- |
| Wire L1 contracts to InputBox | `cartesi-contracts` |
| Debug node or app issues      | `cartesi-debug`        |
