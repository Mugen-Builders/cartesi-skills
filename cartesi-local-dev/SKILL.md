---
name: cartesi-local-dev
version: 0.2.0
description: >-
  Run, test, and iterate on a Cartesi Rollups v2 application locally. Covers
  CLI version detection, building the Cartesi Machine image, running a local
  devnet, using cartesi run, sending advance and inspect requests via CLI,
  reading outputs, and running with a forked chain for realistic testing. Use
  this whenever the user wants to run their app locally, test their handlers,
  send inputs, read outputs, or debug application behaviour in a local
  environment. Triggers on: "run locally", "cartesi run", "test my app",
  "send input", "inspect", "read outputs", "local devnet", "Anvil", "fork",
  "sanity check", "cartesi build".
---

## Skill Version

| Skill               | Version | Cartesi Rollups target               | Contract suite         | Last updated |
| ------------------- | ------- | ------------------------------------ | ---------------------- | ------------ |
| `cartesi-local-dev` | `0.2.0` | Cartesi CLI `2.0.0-alpha.42` (SDK `0.12.0-alpha.45`, rollups-node `v2.0.0-alpha.14`, cartesi-machine 0.22) | `cartesi-rollups` 3.0.0-alpha.10, Dave 3.0.0-alpha.6 | Nov 2026     |

> **Version notes (CLI alpha.42)**
>
> - `cartesi build` / `cartesi run` use SDK `0.12.0-alpha.45`. Machines built
>   by an older CLI (cartesi-machine 0.21 or earlier) cannot be loaded: run
>   `cartesi build` again after upgrading.
> - `cartesi run` serves Anvil, JSON-RPC and inspect behind **one port**
>   (default `6751`). `localhost:8545` is **not** exposed.
> - `cartesi send` and `cartesi deposit` exist and sign with **devnet account
>   8** by default. Account 0 signs the node's claims — do not send
>   transactions from it.
> - `cartesi-rollups-cli` is not installed on the host. With `cartesi run` it
>   lives in the `rollups_node` container; the read/inspect commands in this
>   skill use `curl` and the JSON-RPC API instead.

# Cartesi Rollups v2 — Local Development and Testing

## Goal

Build the Cartesi Machine image, spin up a local devnet, send advance and
inspect requests, and read outputs — all locally. This skill covers the full
local development loop. It does not cover self-hosted node deployment (see
`cartesi-deploy`).

---

## Step 0 — Detect CLI and project version (ALWAYS do first)

```sh
cartesi --version
```

| Output          | CLI version | Build command   | Run command   |
| --------------- | ----------- | --------------- | ------------- |
| `1.5.x`         | v1.5        | `cartesi build` | `cartesi run` |
| `2.0.0-alpha.x` | v2.0-alpha  | `cartesi build` | `cartesi run` |

Detect project version from Dockerfile:

```sh
grep -E "MACHINE_EMULATOR_TOOLS_VERSION|MACHINE_GUEST_TOOLS_VERSION" Dockerfile
```

- `MACHINE_EMULATOR_TOOLS_VERSION` → v1.5 project
- `MACHINE_GUEST_TOOLS_VERSION` → v2.0-alpha project

Use the CLI version that matches the project version.

### Concrete Dockerfile differences between v1.5 and v2.0-alpha

Understanding these differences prevents build errors when working across projects.

**v1.5 Dockerfile pattern:**

```dockerfile
# Single-stage build
FROM --platform=linux/riscv64 cartesi/python:3.10-slim-jammy

ARG MACHINE_EMULATOR_TOOLS_VERSION=0.14.1
ADD https://github.com/cartesi/machine-emulator-tools/releases/download/v${MACHINE_EMULATOR_TOOLS_VERSION}/machine-emulator-tools-v${MACHINE_EMULATOR_TOOLS_VERSION}.tar.gz /
RUN tar -xzf /machine-emulator-tools-v${MACHINE_EMULATOR_TOOLS_VERSION}.tar.gz -C / && \
    rm /machine-emulator-tools-v${MACHINE_EMULATOR_TOOLS_VERSION}.tar.gz

ENV PATH="/opt/cartesi/bin:${PATH}"

WORKDIR /opt/cartesi/dapp
COPY requirements.txt .
RUN pip install -r requirements.txt --break-system-packages
COPY . .
ENTRYPOINT ["rollup-init"]
CMD ["python3", "dapp.py"]
```

**v2.0-alpha Dockerfile pattern** (python template, branch `prerelease/sdk-12`
of `cartesi/application-templates`, which `cartesi create` uses):

```dockerfile
ARG APT_UPDATE_SNAPSHOT=20261115T030400Z
ARG MACHINE_GUEST_TOOLS_VERSION=0.18.0
ARG MACHINE_GUEST_TOOLS_SHA256SUM=<sha256 from the template>

FROM --platform=linux/riscv64 cartesi/python:3.13.2-slim-noble AS base
# pins the APT snapshot for reproducible builds

FROM base
ARG MACHINE_GUEST_TOOLS_VERSION
ARG MACHINE_GUEST_TOOLS_SHA256SUM
ADD --checksum=sha256:${MACHINE_GUEST_TOOLS_SHA256SUM} \
    https://github.com/cartesi/machine-guest-tools/releases/download/v${MACHINE_GUEST_TOOLS_VERSION}/machine-guest-tools_riscv64.deb \
    /tmp/machine-guest-tools_riscv64.deb
RUN apt-get install -y --no-install-recommends --snapshot=${APT_UPDATE_SNAPSHOT} \
    busybox-static /tmp/machine-guest-tools_riscv64.deb

WORKDIR /opt/cartesi/dapp
COPY ./requirements.txt .
RUN pip install -r requirements.txt --no-cache
COPY ./dapp.py .
ENTRYPOINT ["rollup-init"]
CMD ["python3", "dapp.py"]
```

`rollup-init` starts the rollup HTTP server the backend polls. The JavaScript
and TypeScript templates are different: they have no `rollup-init` and use the
`@cartesi/rollup` package instead (see `cartesi-backend-js-ts`). The
`cpp-low-level` template also has no `rollup-init`; it links libcmt directly.

Do not hand-write these values: take the Dockerfile from the template and
keep its version, checksum and snapshot together.

Key differences at a glance:

| Aspect                | v1.5                              | v2.0-alpha                                |
| --------------------- | --------------------------------- | ----------------------------------------- |
| Dockerfile stages     | Single stage                      | Multi-stage (common pattern)              |
| Guest tools package   | `machine-emulator-tools` (tar.gz) | `machine-guest-tools` (.deb)              |
| Version variable      | `MACHINE_EMULATOR_TOOLS_VERSION`  | `MACHINE_GUEST_TOOLS_VERSION`             |
| Base Python image     | `cartesi/python:3.10-slim-jammy`  | `cartesi/python:3.13.2-slim-noble`        |
| APT snapshot pinning  | Not used                          | `APT_UPDATE_SNAPSHOT` for reproducibility |
| Checksum verification | Not used                          | sha256 checksum on downloaded package     |
| Entrypoint            | `rollup-init`                     | `rollup-init` (HTTP server) in most templates; `node` + `@cartesi/rollup` in JS/TS |

> **v2.0-alpha note**: Internal service versions (guest tools version, base image tags,
> APT snapshot dates) are still evolving as it is in alpha. Always check the latest
> Cartesi CLI release notes for the canonical Dockerfile template. A project
> scaffolded with guest tools older than `0.18.0` should be refreshed from the
> current template before building with CLI alpha.42.

### `cartesi.toml`

Build and run settings live in an optional `cartesi.toml` at the project root
(drives, `[machine]`, nvrams, `[withdrawal]`, `[prt]`). Since CLI alpha.41 an
**unknown key fails** `build`, `run` and `shell` with a message naming the key
and the keys that table accepts — fix the key rather than working around it.
`[withdrawal.config]` is no longer supported; mark the accounts drive with
`accounts_drive = true` instead (see `cartesi-contracts`).

---

## Preconditions

```sh
cartesi --version
docker --version    # Docker running
```

The application must already be scaffolded (`cartesi-scaffold`) and have
backend handlers implemented (`cartesi-backend-core` plus
`cartesi-backend-py` or `cartesi-backend-js-ts`).

---

## Phase 1 — Build the Cartesi Machine image

Every code change requires a rebuild before `cartesi run` picks it up:

```sh
cartesi build
```

This compiles the application into a RISC-V Cartesi Machine snapshot at
`.cartesi/image/`. A successful build prints:

```
Manual yield rx-accepted (1) (0x000020 data)
Cycles: <n>
<n>: 0x<machine hash>
Storing machine: please wait
```

`cartesi hash` then prints the template hash of the stored image.

Common build failures:

- **Missing system packages**: Add them to the `Dockerfile` via `apt install`.
- **Dependency errors**: Install packages inside the Dockerfile, not just on
  the host.
- **Docker not running**: Start Docker Desktop or the Docker daemon.

---

## Phase 2 — Run locally with `cartesi run`

`cartesi run` starts a local Anvil devnet with the Cartesi contracts
pre-deployed, a database and the Rollups Node, all in Docker, then deploys and
registers the application from `.cartesi/image/`.

```sh
cartesi run
```

It prints `<project> starting at http://127.0.0.1:<port>` and then stays
attached with a small menu: **View logs**, **Build and redeploy**, **Quit**.
"Build and redeploy" rebuilds the machine and redeploys the application without
restarting the environment. Run all other commands from a second terminal in
the same project directory.

Everything is served behind **one port** (default `6751`; the next free port up
to `6761` is used if it is taken; `-p <port>` sets it explicitly):

| Service      | URL                                          |
| ------------ | -------------------------------------------- |
| Anvil L1 RPC | `http://127.0.0.1:6751/anvil`                |
| JSON-RPC API | `http://127.0.0.1:6751/rpc`                  |
| Inspect      | `http://127.0.0.1:6751/inspect/<app-name>`   |
| Explorer *(only with `--services explorer`)* | `http://127.0.0.1:6751/explorer` |

`localhost:8545` is **not** exposed. Any `cast`, `forge`, wallet or frontend
RPC URL must be `http://127.0.0.1:<port>/anvil`.

```sh
export RPC_URL=http://127.0.0.1:6751/anvil   # adjust the port if cartesi run printed another
export NODE_URL=http://127.0.0.1:6751
```

`<app-name>` is the project name: the directory name, unless
`--project-name` is passed.

Useful flags (`cartesi run --help` lists all):

| Flag | Default | Purpose |
| ---- | ------- | ------- |
| `-p, --port <n>` | `6751` | Port to listen on |
| `--block-time <s>` | `2` | Anvil block interval |
| `--epoch-length <blocks>` | `720` | Lower it (for example `10`) to see epochs close and claims accepted quickly |
| `--claim-staging-period <blocks>` | `0` | Blocks between a claim being staged and accepted |
| `--default-block <tag>` | `latest` | `latest`, `safe` or `finalized` (`pending` is no longer accepted) |
| `--prt` | off | Deploy on a PRT (Dave) consensus using the `[prt]` section of `cartesi.toml` |
| `--services <list>` | none | Optional extra services, comma separated: `bundler`, `explorer`, `paymaster`, `passkey` |
| `--fork-url`, `--fork-block-number` | — | Fork a live chain (Phase 7) |
| `--dry-run` | — | Print the Docker Compose configuration only |

Companion commands (second terminal, same directory):

```sh
cartesi status            # is the environment up, with app address, status and enabled flag
                          # (`--json` fails in alpha.42: "Do not know how to serialize a BigInt")
cartesi logs -f           # node logs (and PRT nodes); `cartesi logs <service>` for one
cartesi address-book      # every deployed address; --json for scripts
```

**Teardown**: choose **Quit** in the menu (or Ctrl-C). If a previous session
was left behind:

```sh
docker compose -p <app-name> down -v
```

Then re-run `cartesi run`. The node database is recreated on every start, so
nothing persists between sessions.

---

## Phase 3 — Send advance inputs (state-changing)

Advance inputs are submitted **on-chain** to the InputBox contract. The node
monitors the contract and forwards new inputs to your backend.

### Using the Cartesi CLI (recommended)

```sh
# Interactive: prompts for the payload
cartesi send

# Plain string (encoded as UTF-8)
cartesi send 'hello'
cartesi send --encoding string '{"action":"buy","item":"sword","qty":1}'

# Hex payload (a value starting with 0x is taken as hex)
cartesi send 0x68656c6c6f
cartesi send --encoding hex 0x68656c6c6f

# ABI-encoded payload
cartesi send --encoding abi --abi-params 'address,uint256' '0x70997970C51812dc3A010C7d01b50e0d17dc79C8,42'
```

`--encoding` accepts `hex`, `string`, `abi` and `abi-packed`. There is no
`--hex` flag. Check `cartesi send --help` for the exact value syntax of the ABI
encodings.

The sender defaults to **devnet account 8**
(`0x23618e81E3f5cdF7f54C3d65f7FBc0aBf5B21E8f`). Use `--from <address>` for
another devnet account. Only accounts 0–9 of the devnet mnemonic are unlocked
on Anvil and can sign this way. Avoid account 0: the node signs its claims with it, and sharing a
signer causes nonce conflicts. With `--prt`, accounts 6 and 20–29 are used by
the PRT nodes.

### Using cast directly against the InputBox (alternative)

```sh
INPUT_BOX=$(cartesi address-book InputBox)
APP_ADDRESS=$(cartesi address-book Application)

cast send $INPUT_BOX \
  "addInput(address,bytes)" $APP_ADDRESS <hex-payload> \
  --rpc-url $RPC_URL \
  --private-key $DEV_KEY
```

`DEV_KEY` is the private key of devnet account 8 (standard Anvil test key —
never use it outside a local devnet):

```sh
export DEV_KEY=0xdbda1821b80551c9d65939329250298aa3472ba22feea921c0cf5d620ea67b97
export DEV_ADDR=0x23618e81E3f5cdF7f54C3d65f7FBc0aBf5B21E8f
```

---

## Phase 3.5 — Local devnet test tokens

The devnet started by `cartesi run` already contains test token contracts for
exercising deposit flows.

### Resolving test token and portal addresses

> **Never hardcode token or portal addresses.** They change between contract
> releases. Always resolve them at task time using `cartesi address-book`.

```sh
cartesi address-book            # table of every address
cartesi address-book --json     # same, as JSON
cartesi address-book InputBox   # one address only (name is case-insensitive)
```

Names printed by CLI alpha.42:

| Name | Contract |
| ---- | -------- |
| `Application` | Your deployed application |
| `InputBox` | Input ingress |
| `EtherPortal`, `Erc20Portal`, `Erc721Portal`, `Erc1155SinglePortal`, `Erc1155BatchPortal` | Asset portals |
| `TestToken` | ERC-20 test token (`TestFungibleToken`, 18 decimals) |
| `TestUsdc` | ERC-20 test token with 6 decimals |
| `TestNFT` | ERC-721 test token |
| `TestMultiToken` | ERC-1155 test token |
| `ApplicationFactory`, `AuthorityFactory`, `SelfHostedApplicationFactory`, `DaveAppFactory` | Factories |
| `RefundOutputBuilder`, `UsdWithdrawalOutputBuilderFactory`, `TestUsdWithdrawalOutputBuilder` | Output builders (refund / emergency withdrawal) |

> Portal names are now `Erc20Portal`, `Erc721Portal`, … (not `ERC20Portal`).
> `cartesi address-book <name>` matches case-insensitively, but scripts that
> parse the table or the JSON by exact key must use the new spelling.

Capture what you need into shell variables:

```sh
export TEST_TOKEN=$(cartesi address-book TestToken)
export TEST_NFT=$(cartesi address-book TestNFT)
export TEST_MULTI=$(cartesi address-book TestMultiToken)
export ERC20_PORTAL=$(cartesi address-book Erc20Portal)
export ERC721_PORTAL=$(cartesi address-book Erc721Portal)
export ERC1155_PORTAL=$(cartesi address-book Erc1155SinglePortal)
export APP_ADDRESS=$(cartesi address-book Application)

# Verify — all variables should be non-empty 40-char hex addresses
echo "TEST_TOKEN=$TEST_TOKEN  TEST_NFT=$TEST_NFT  TEST_MULTI=$TEST_MULTI"
echo "ERC20_PORTAL=$ERC20_PORTAL  ERC721_PORTAL=$ERC721_PORTAL  ERC1155_PORTAL=$ERC1155_PORTAL"
echo "APP_ADDRESS=$APP_ADDRESS"
```

The test tokens have **open mint functions** — anyone can mint to any address.
Nothing is pre-minted to a fixed owner, so mint what the test needs first.

### Depositing tokens via the CLI (recommended)

```sh
cartesi deposit                 # interactive: pick the asset type
cartesi deposit ether 1         # 1 ETH
cartesi deposit erc20 100 --token $(cartesi address-book TestToken)   # without --token it prompts
cartesi deposit erc721 1 --token $(cartesi address-book TestNFT)      # token ID
cartesi deposit erc1155-single  # token ID and amount
cartesi deposit erc1155-batch   # comma-separated token IDs and amounts
```

The CLI handles the portal approval itself, but the sender must already hold
the tokens — mint first (below). Pass `--token` in scripts; without it the
command stops at an interactive prompt. Deposits sign with devnet account 8
unless `--from` is given. Use
`cartesi deposit <asset> --help` for the options of each asset type.

### Depositing ERC20 via cast (manual)

> **Prerequisite**: set `RPC_URL`, `DEV_KEY`, `DEV_ADDR` and the address
> variables from the sections above.
>
> **Portal function names changed in contracts v3**: `depositErc20Tokens`,
> `depositErc721Token`, `depositSingleErc1155Token`, `depositBatchErc1155Token`
> (lower-case `rc`). The old `depositERC20Tokens(...)` spelling is a different
> selector and reverts.

Step 0 — Mint test tokens to the sender:

```sh
cast send $TEST_TOKEN "mint(address,uint256)" $DEV_ADDR 300000000000000000000 \
  --rpc-url $RPC_URL --private-key $DEV_KEY
```

Step 1 — Approve the portal to spend tokens:

```sh
cast send $TEST_TOKEN "approve(address,uint256)" $ERC20_PORTAL 300000000000000000000 \
  --rpc-url $RPC_URL --private-key $DEV_KEY
```

Step 2 — Deposit to your application:

```sh
cast send $ERC20_PORTAL \
  "depositErc20Tokens(address,address,uint256,bytes)" \
  $TEST_TOKEN $APP_ADDRESS 202 0x \
  --rpc-url $RPC_URL --private-key $DEV_KEY
```

### Minting and depositing ERC721 via cast

```sh
# 1. Mint token ID 1 to the sender
cast send $TEST_NFT "mint(address,uint256)" $DEV_ADDR 1 \
  --rpc-url $RPC_URL --private-key $DEV_KEY

# 2. Approve the portal
cast send $TEST_NFT "setApprovalForAll(address,bool)" $ERC721_PORTAL true \
  --rpc-url $RPC_URL --private-key $DEV_KEY

# 3. Deposit token ID 1 (tokenId, baseLayerData, execLayerData)
cast send $ERC721_PORTAL \
  "depositErc721Token(address,address,uint256,bytes,bytes)" \
  $TEST_NFT $APP_ADDRESS 1 0x 0x \
  --rpc-url $RPC_URL --private-key $DEV_KEY
```

### Minting and depositing ERC1155 via cast

```sh
# 1. Mint 1 unit of token ID 1 to the sender
cast send $TEST_MULTI "mint(address,uint256,uint256)" $DEV_ADDR 1 1 \
  --rpc-url $RPC_URL --private-key $DEV_KEY

# 2. Approve the portal
cast send $TEST_MULTI "setApprovalForAll(address,bool)" $ERC1155_PORTAL true \
  --rpc-url $RPC_URL --private-key $DEV_KEY

# 3. Deposit 1 unit of token ID 1 (tokenId, value, baseLayerData, execLayerData)
cast send $ERC1155_PORTAL \
  "depositSingleErc1155Token(address,address,uint256,uint256,bytes,bytes)" \
  $TEST_MULTI $APP_ADDRESS 1 1 0x 0x \
  --rpc-url $RPC_URL --private-key $DEV_KEY
```

### Testing with a second user

Mint directly to another devnet address and send from it:

```sh
# devnet account 9
cast send $TEST_TOKEN "mint(address,uint256)" \
  0xa0Ee7A142d267C1f36714E4a8F75612F20a79720 202 \
  --rpc-url $RPC_URL --private-key $DEV_KEY

cartesi send --from 0xa0Ee7A142d267C1f36714E4a8F75612F20a79720 'hello from another user'
```

> **Note**: All state resets when you stop `cartesi run`. Token balances, minted NFTs,
> and any deposits do not persist across restarts. For persistent state across sessions,
> deploy using the compose-based setup (`cartesi-deploy` skill).

---

## Phase 4 — Send inspect requests (read-only)

Inspect requests are sent directly to the Rollups Node via HTTP — no
blockchain transaction required.

The node takes the **raw request body** as the inspect payload
(`Content-Type: application/octet-stream`). Do **not** wrap it in JSON: a body
of `{"payload":"0x..."}` reaches the backend as that literal JSON text.

### With `cartesi run` — use curl

```sh
# The body is the payload, byte for byte
curl -s -X POST "$NODE_URL/inspect/<app-name>" \
  -H "Content-Type: application/octet-stream" \
  --data-binary 'stats'

# Binary payload from hex
printf '7374617473' | xxd -r -p | \
  curl -s -X POST "$NODE_URL/inspect/<app-name>" \
    -H "Content-Type: application/octet-stream" --data-binary @-
```

Response:

```json
{
  "status": "Accepted",
  "reports": [{ "payload": "0x..." }],
  "processed_input_count": 3
}
```

`status` is one of `Accepted`, `Rejected`, `Exception`, `MachineHalted`,
`Failed`. `Exception` adds `exception_data`; `Exception` and `Failed` add an
`error` string. HTTP `404` means the application is unknown; `503` means the
machine is not ready, the application was foreclosed, its execution has
terminated, or inspect capacity is exhausted. Inspect runs on a temporary fork of the machine — it never changes
application state.

### With compose deployment — use `cartesi-rollups-cli` inside the container

If you deployed via the compose file (see `cartesi-deploy` skill), the CLI is
available inside the advancer container:

```sh
docker compose -f compose.local.yaml exec advancer \
  cartesi-rollups-cli inspect <app-name> "stats"

docker compose -f compose.local.yaml exec advancer \
  cartesi-rollups-cli inspect <app-name> 0x7374617473 --hex
```

The same binary is in the `rollups_node` container of a `cartesi run` session:

```sh
docker compose -p <app-name> exec rollups_node cartesi-rollups-cli inspect <app-name> "stats"
```

Decode the report payload from the response:

```sh
# Response contains hex payload — decode it:
echo "0x<hex-payload>" | xxd -r -p
# Or with cast:
cast --to-ascii <hex-without-0x>
```

---

## Phase 5 — Read outputs from the node

After sending advance inputs and waiting for processing, read the results.

> With `cartesi run`, use the JSON-RPC API exposed by the local node (see the
> `cartesi-jsonrpc` skill for the full method reference). `params` must be a
> JSON **object**, and every result is wrapped in `{ "data": ... }`.

### With `cartesi run` — use the JSON-RPC API

The endpoint is `$NODE_URL/rpc` (`http://127.0.0.1:6751/rpc` by default; port
`10011` applies to the compose setup only).

```sh
# Read all outputs for your app
curl -s -X POST "$NODE_URL/rpc" \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"cartesi_listOutputs","params":{"application":"<app-name>"},"id":1}'

# Read all reports for your app
curl -s -X POST "$NODE_URL/rpc" \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"cartesi_listReports","params":{"application":"<app-name>"},"id":1}'

# Read all inputs processed
curl -s -X POST "$NODE_URL/rpc" \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"cartesi_listInputs","params":{"application":"<app-name>"},"id":1}'

# Read epoch status
curl -s -X POST "$NODE_URL/rpc" \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"cartesi_listEpochs","params":{"application":"<app-name>"},"id":1}'
```

See the `cartesi-jsonrpc` skill for the complete method reference and TypeScript client patterns.

### With compose deployment — use `cartesi-rollups-cli` inside the container

If you deployed via the compose file (see `cartesi-deploy` skill), the CLI is
available inside the advancer container. All commands must be prefixed with
`docker compose -f compose.local.yaml exec advancer`:

```sh
EXEC="docker compose -f compose.local.yaml exec advancer cartesi-rollups-cli"

# Read all outputs (notices, vouchers, delegated call vouchers)
$EXEC read outputs <app-name>
$EXEC read outputs <app-name> 42        # specific output by index

# Read reports
$EXEC read reports <app-name>
$EXEC read reports <app-name> --limit 10 --offset 0

# Read inputs processed by the node
$EXEC read inputs <app-name>
$EXEC read inputs <app-name> 5          # specific input by index
$EXEC read inputs <app-name> --epoch-index 0x0 --limit 20

# Read epoch status
$EXEC read epochs <app-name>
$EXEC read epochs <app-name> --status OPEN
```

---

## Phase 6 — Sanity checklist after startup

After `cartesi run` and `cartesi build`, run this minimal roundtrip to confirm
everything works end-to-end.

> Use `curl` and the JSON-RPC API for read operations with `cartesi run`. The
> `cartesi-rollups-cli` commands shown further below apply to the compose-based
> self-hosted deployment.

### With `cartesi run`

1. **Send one valid advance input**:

   ```sh
   cartesi send --encoding string '{"ping":true}'
   ```

2. **Confirm the input was processed** via JSON-RPC:

   ```sh
   curl -s -X POST "$NODE_URL/rpc" \
     -H "Content-Type: application/json" \
     -d '{"jsonrpc":"2.0","method":"cartesi_listInputs","params":{"application":"<app-name>"},"id":1}'
   ```

3. **Call a simple inspect route** and decode the response:

   ```sh
   curl -s -X POST "$NODE_URL/inspect/<app-name>" \
     -H "Content-Type: application/octet-stream" --data-binary 'stats'
   # decode a report payload:
   cast --to-utf8 <report-payload-hex>
   ```

4. **Read reports** and verify the payload decodes correctly:

   ```sh
   curl -s -X POST "$NODE_URL/rpc" \
     -H "Content-Type: application/json" \
     -d '{"jsonrpc":"2.0","method":"cartesi_listReports","params":{"application":"<app-name>"},"id":1}'
   ```

5. **Read outputs** and confirm notices appear for accepted inputs.

6. **Check state changed**: call the inspect route again and verify it reflects the advance input.

### With compose deployment

```sh
EXEC="docker compose -f compose.local.yaml exec advancer cartesi-rollups-cli"

# 1. Send a test input
$EXEC send <app-name> '{"ping":true}'

# 2. Confirm input is indexed
$EXEC read inputs <app-name>

# 3. Inspect a route
$EXEC inspect <app-name> "stats"

# 4. Read reports
$EXEC read reports <app-name>

# 5. Read outputs (notices/vouchers)
$EXEC read outputs <app-name>
```

---

## Phase 7 — Forked chain workflow

Fork a public chain to test with realistic contract state. The CLI validates
the fork target before starting: the chain must be one it supports (the
Cartesi contracts must be deployed there), and with `--prt` the PRT contracts
too. An unsupported chain fails with an explicit error.

```sh
# Fork at the chain's latest block
cartesi run --fork-url <RPC_URL>

# Fork at a specific historical block (deterministic)
cartesi run --fork-url <RPC_URL> --fork-block-number <N>
```

Important caveats for forked chains:

- Fork state is **static** at the chosen block height. Repeated oracle reads
  return identical values — this is expected.
- To test changing external data, use new advance inputs that bypass L1
  or re-fork at a later block.
- Chain ID must match your wallet / signer config — mismatches surface as
  `invalid chain id for signer`.
- The L1 RPC is still `http://127.0.0.1:<port>/anvil`, not the fork URL.
- Test tokens (`TestToken`, `TestNFT`, …) are devnet-only and are not in the
  address book of a fork. A fork that uses emergency withdrawal must set
  `withdrawal_output_builder` in `[withdrawal]` of `cartesi.toml`.

---

## Environment variables for compose deployment CLI commands

> **This section only applies to compose-based self-hosted deployments** (see `cartesi-deploy`
> skill). `cartesi-rollups-cli` does not run on the host machine, and a `cartesi run`
> session already sets these inside its `rollups_node` container. They are only needed
> when running `cartesi-rollups-cli` commands inside the advancer container via
> `docker compose exec`.

When the compose file's `.env` does not cover a needed variable, set it inside the container:

```sh
export CARTESI_DATABASE_CONNECTION=postgres://postgres:password@database:5432/rollupsdb?sslmode=disable
export CARTESI_BLOCKCHAIN_HTTP_ENDPOINT=http://<l1-rpc-host>:8545
export CARTESI_BLOCKCHAIN_ID=31337
export CARTESI_AUTH_MNEMONIC="test test test test test test test test test test test junk"
# A different account than the node's claimer (index 0):
export CARTESI_AUTH_MNEMONIC_ACCOUNT_INDEX=8
export CARTESI_CONTRACTS_INPUT_BOX_ADDRESS=<InputBox address for the chain>
export CARTESI_INSPECT_URL=http://localhost:10012
export CARTESI_JSONRPC_API_URL=http://localhost:10011/rpc
```

Notes for rollups-node alpha.14:

- `CARTESI_BLOCKCHAIN_WS_ENDPOINT` no longer exists — the node uses HTTP only.
- `CARTESI_INSPECT_ADDRESS` is the **listen** address of the inspect service
  (`:10012`); client commands read `CARTESI_INSPECT_URL`.
- Use one submitting process per signer: never send from the key the running
  node uses for claims.
- Database and host names depend on the compose file; the values above are
  examples.

For Anvil's devnet, chain ID `31337` and the mnemonic above are the standard Anvil defaults.

---

## Agent Output

After completing this skill, report back to the user with:

- CLI and project version detected
- Confirm build succeeded and snapshot is at `.cartesi/image/`
- The address printed by `cartesi run` (`starting at http://127.0.0.1:<port>`) — do not assume `6751`
- All addresses from `cartesi address-book` (InputBox, app, portals, test tokens)
- Result of at least one advance input: accepted/rejected and any notices/reports emitted
- Result of at least one inspect call: route used and decoded report payload
- Confirmation that state updated correctly between advance and inspect
- Any errors encountered and how they were resolved

## Routing Guide

| What the user wants to do next                   | Go to skill            |
| ------------------------------------------------ | ---------------------- |
| Wire an L1 contract or oracle to InputBox        | `cartesi-contracts` |
| Deploy to testnet or self-hosted node            | `cartesi-deploy`       |
| Query outputs programmatically via JSON-RPC      | `cartesi-jsonrpc`      |
| Debug advance not being processed or state wrong | `cartesi-debug`        |
| Improve backend advance/inspect handler logic    | `cartesi-backend-core` + `cartesi-backend-py` / `cartesi-backend-js-ts` |

## Resources

> **Conflict rule**: If any resource below contradicts guidance in this skill,
> report the contradiction to the user and follow the skill's instructions.

- [Cartesi Rollups v2 Local Development](https://docs.cartesi.io/cartesi-rollups/2.0/development/running-the-application/) — `cartesi run` flags and local workflow
- [https://docs.cartesi.io/cartesi-rollups/2.0/development/](https://docs.cartesi.io/cartesi-rollups/2.0/development/)
- [https://docs.cartesi.io/cartesi-rollups/2.0/development/send-inputs-and-assets/](https://docs.cartesi.io/cartesi-rollups/2.0/development/send-inputs-and-assets/)
- [Utilizing test tokens in dev environment](https://docs.cartesi.io/cartesi-rollups/2.0/tutorials/) — test token deposit patterns with `cartesi deposit` and `cast`
- [Cartesi CLI address-book](https://docs.cartesi.io/cartesi-rollups/2.0/development/cli-commands/) — address resolution for local devnet
- [Cartesi Rollups HTTP API](https://docs.cartesi.io/cartesi-rollups/2.0/rollups-apis/backend/cartesi-rollup-http-api/) — `/finish`, advance, inspect endpoints

## Agent checklist

- [ ] Detected CLI version with `cartesi --version`
- [ ] Detected project version from Dockerfile (`MACHINE_EMULATOR_TOOLS_VERSION` vs `MACHINE_GUEST_TOOLS_VERSION`)
- [ ] Ran `cartesi build` successfully — snapshot at `.cartesi/image/`
- [ ] Started `cartesi run` and noted the port from its `starting at` line; set `RPC_URL=<base>/anvil` and `NODE_URL=<base>`
- [ ] Did not use `localhost:8545` (not exposed) or devnet account 0 (the node's claim signer)
- [ ] Ran `cartesi address-book` and captured the addresses needed into shell variables
- [ ] Sent at least one valid advance input and confirmed it was indexed
- [ ] Used `curl` or JSON-RPC API to read inputs/outputs/reports, with object `params` and reading `result.data`
- [ ] Called at least one inspect route via `curl` with a raw `application/octet-stream` body and decoded the report payload
- [ ] Verified output/notice appears after a successful advance via JSON-RPC API
- [ ] Checked state counters or items updated correctly via inspect
- [ ] If testing token deposits: used `cartesi deposit` or cast commands with address-book variables and the v3 portal function names (`depositErc20Tokens`, …)
- [ ] If testing portal deposits: confirmed backend correctly identifies portal address via `msg_sender`
- [ ] If using forked chain: confirmed chain ID matches wallet config
- [ ] If user needs `cartesi-rollups-cli` commands: used `docker compose -p <app-name> exec rollups_node cartesi-rollups-cli …` or the compose-based deployment (`cartesi-deploy` skill)

## What comes next

| Next task                     | Skill to use           |
| ----------------------------- | ---------------------- |
| Wire L1 contracts to InputBox | `cartesi-contracts` |
| Deploy to self-hosted node    | `cartesi-deploy`       |
| Debug unexpected behaviour    | `cartesi-debug`        |
