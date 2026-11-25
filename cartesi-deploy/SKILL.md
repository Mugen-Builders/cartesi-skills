---
name: cartesi-deploy
version: 0.4.0
description: >-
  Deploy a Cartesi Rollups application to a self-hosted rollups node for
  testnet or production-style deployment using Docker Compose. Covers contracts
  v3 deploy flags (claim-staging-period, withdrawal-config), authority, quorum
  and PRT (Dave) deploys with sentries, operator CLI (foreclose,
  prove-drive-root, withdraw, refund), application lifecycle (enabled/status),
  and node ops for rollups-node v2.0.0-alpha.14. Triggers on: "deploy", "self-hosted",
  "testnet", "rollups node", "docker compose", "compose.local", "register
  application", "withdrawal config", "claim staging", "foreclose", "quorum
  deploy", "PRT", "Dave", "sentries", "Sepolia", "Mugen-Builders".
---

## Skill Version

| Skill            | Version | Cartesi Rollups target | Contract suite              | Node / runtime pin                          | Compose setup       | Last updated |
| ---------------- | ------- | ---------------------- | --------------------------- | ------------------------------------------- | ------------------- | ------------ |
| `cartesi-deploy` | `0.4.0` | contracts v3           | `cartesi-rollups` 3.0.0-alpha.10 + Dave (PRT) 3.0.0-alpha.6 | `rollups-node` **v2.0.0-alpha.14** → `cartesi/rollups-runtime:0.12.0-alpha.45` | Mugen-Builders v2.0 | Nov 2026     |

> Confirm the `cartesi/rollups-runtime` image tag in `compose.local.yaml` matches
> both the **node binary** under test and the **contract suite** under test.
> Contract addresses and lifecycle semantics: **`cartesi-contracts`**.

### Version mapping (node alpha.14)

Cartesi uses different version schemes across artifacts. For **node alpha.14**:

| Artifact | Version |
| -------- | ------- |
| `rollups-node` release tag | `v2.0.0-alpha.14` |
| Docker runtime / database image | `cartesi/rollups-runtime:0.12.0-alpha.45`, `cartesi/rollups-database:0.12.0-alpha.45` (PostgreSQL 18) |
| `cartesi-rollups-cli` inside runtime | `2.0.0-alpha.14` |
| Cartesi CLI (host build) | `@cartesi/cli@2.0.0-alpha.42` (defaults to SDK `0.12.0-alpha.45`) |
| Cartesi Machine | `0.22.0` |
| Rollups contracts | `cartesi-rollups` **3.0.0-alpha.10** |
| PRT (Dave) contracts | **3.0.0-alpha.6** |

Verify after starting the node:

```sh
docker compose -f compose.local.yaml exec advancer cartesi-rollups-node --version
# expect: 2.0.0-alpha.14

docker compose -f compose.local.yaml exec advancer cartesi-rollups-cli --version
# expect: 2.0.0-alpha.14
```

### Upgrading from alpha.12 / alpha.13 — not backward compatible

- **Rebuild every application.** Machines stored by cartesi-machine 0.21 or
  earlier cannot be loaded by 0.22, and every machine hash changes. Run
  `cartesi build` with CLI alpha.42 and **redeploy** the application contract
  (the template hash is part of it).
- **Fresh database.** The node's schema migration was edited in place; a
  database created by an earlier node cannot be reused and the version check
  will not catch it. Always `down -v` before the first alpha.14 start.
- **New contract addresses.** The suite moved from `cartesi-rollups`
  3.0.0-alpha.6 to **3.0.0-alpha.10**; InputBox, factories and portals all have
  new addresses. PRT contracts moved to Dave 3.0.0-alpha.6.
- **Settings that no longer exist.** `CARTESI_BLOCKCHAIN_WS_ENDPOINT` (gone
  since alpha.12) and
  `CARTESI_CLAIMER_MAX_ACCEPT_ATTEMPTS` no longer exist. `pending` is no longer
  a valid default block.

> **Mugen-Builders caveat**: The upstream `compose.local.yaml` currently ships
> `0.12.0-alpha.41` images (node alpha.12) and `cartesi-rollups` 3.0.0-alpha.6
> address defaults. Until it is bumped, apply the patch in Step 3 and the
> address overrides in Step 4. The file patched this way was run on
> 2026-11-25 against a local devnet: all services started on the alpha.45
> images, and Authority and PRT deploys, inputs, inspect, JSON-RPC and voucher
> execution worked. It has not been run against a public testnet. If a service
> does not start, compare with the reference `compose.individual-services.yaml`
> in the rollups-node repository.

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
| `2.0.0-alpha`   | v2.0-alpha  | `cartesi deploy` is a deprecated stub; use compose-based deploy |

For node alpha.14 deployments, the host CLI must be `2.0.0-alpha.42` or newer so
`cartesi build` uses SDK `0.12.0-alpha.45` (cartesi-machine 0.22), matching
`cartesi/rollups-runtime:0.12.0-alpha.45`. An image built by an older CLI will
not load. In alpha.42 `cartesi deploy` exists only as a deprecated stub that
points to `cartesi-rollups-cli`.

### Detect project version from Dockerfile

```sh
grep -E "MACHINE_EMULATOR_TOOLS_VERSION|MACHINE_GUEST_TOOLS_VERSION" Dockerfile
```

| Match found                                            | Project targets    |
| ------------------------------------------------------ | ------------------ |
| `MACHINE_EMULATOR_TOOLS_VERSION=0.14.1`                | Cartesi v1.5       |
| `MACHINE_GUEST_TOOLS_VERSION=0.18.0` (current template; older values need a template refresh) | Cartesi v2.0-alpha |

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
| `ENTRYPOINT ["rollup-init"]`       | Not a version marker (v1.5 and most v2 templates; v2 JS/TS use `ENTRYPOINT ["node"]`) |

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
cartesi hash          # prints the machine (template) hash of the built image
```

If the application supports emergency withdrawal, `cartesi build` also prints
the accounts drive and the resulting withdrawal configuration — keep that
output for the deploy step.

---

## Step 3 — Download the compose setup

Download the official Mugen-Builders deployment compose file into the
**project root** (alongside `.cartesi/`):

```sh
curl -L \
  https://raw.githubusercontent.com/Mugen-Builders/deployment-setup-v2.0/98847c19660b46cd9f1b1c8ff9f8cf7f88039f98/compose.local.yaml \
  -o compose.local.yaml
```

The URL is pinned to the commit the patch steps below were written and tested
against (node alpha.12 images, `cartesi-rollups` 3.0.0-alpha.6 defaults) — the
same commit the official self-hosted deployment docs use. Do not switch it to
`main`: once that repository is updated, the patch steps would no longer match
the file. If a newer commit already ships `0.12.0-alpha.45` images and the
3.0.0-alpha.10 addresses, use it as-is and skip the image and address patches.

This compose file defines 6 services that together form the Cartesi Rollups
Node. **Do not use the downloaded file as-is** — patch it for node alpha.14
before starting (see below).

| Service       | Typical role                            |
| ------------- | --------------------------------------- |
| `database`    | Postgres — shared state bus             |
| `evm-reader`  | Reads L1 events (inputs, foreclosure, withdrawals) |
| `advancer`    | Runs Cartesi Machine; inspect           |
| `validator`   | Computes epoch claims + proofs          |
| `claimer`     | Submit, stage, accept claims (Authority / Quorum) |
| `jsonrpc-api` | Query API                               |
| `prt` *(add for PRT apps)* | Dave tournaments: commitments, disputes, settlement |

The advancer mounts `.cartesi/image/` from the project root as the
application snapshot:

```yaml
volumes:
  - .cartesi/image:/var/lib/cartesi-rollups-node/snapshot/
```

### Patch compose for node alpha.14

After downloading, apply these changes to `compose.local.yaml`:

**1. Bump runtime and database images** (upstream ships alpha.41 = node alpha.12):

```yaml
# database service
image: cartesi/rollups-database:0.12.0-alpha.45

# all other services (evm-reader, advancer, validator, claimer, jsonrpc-api)
image: cartesi/rollups-runtime:0.12.0-alpha.45
```

**2. No WebSocket endpoint** — `CARTESI_BLOCKCHAIN_WS_ENDPOINT` does not exist
in this node. Make sure no such line is present in `x-env`.

**3. One default block for all services.** `CARTESI_BLOCKCHAIN_DEFAULT_BLOCK`
must be `latest`, `safe` or `finalized` (`pending` is rejected; the node's own
default is `finalized`). EVM Reader, Claimer and PRT must all use the same
value. Each service **saves the policy on first start** and refuses to start
later with a different one, so pick it before the first start. (The safe way
to change it afterwards is a fresh database; an in-place change was not
tested.)

**4. Override contract addresses** from
[`cartesi-contracts/cartesi-rollups-3.0.0-alpha.10.json`](../cartesi-contracts/cartesi-rollups-3.0.0-alpha.10.json).
The compose defaults are the **3.0.0-alpha.6** suite (e.g. `InputBox`
`0x346B3d…`); alpha.14 needs **3.0.0-alpha.10** (`InputBox` `0xEbE9f4…`). Set
them in `.env` (Step 4) — the compose file reads each address from the
environment.

**5. PRT applications only — add a `prt` service** next to `claimer`:

```yaml
  prt:
    image: cartesi/rollups-runtime:0.12.0-alpha.45
    command: cartesi-rollups-prt
    depends_on:
      database:
        condition: service_healthy
    networks:
      - default
    ports:
      - "10006:10006"
    environment:
      <<: *env
```

and add to `x-env`:

```yaml
  CARTESI_CONTRACTS_DAVE_APP_FACTORY_ADDRESS: ${CARTESI_CONTRACTS_DAVE_APP_FACTORY_ADDRESS}
  CARTESI_PRT_AUTH_KIND: ${PRT_AUTH_KIND:-private_key}
  CARTESI_PRT_AUTH_PRIVATE_KEY: ${CARTESI_PRT_AUTH_PRIVATE_KEY}
```

The PRT service signs with its **own** key (`CARTESI_PRT_AUTH_*`), separate
from the claimer's `CARTESI_AUTH_*`. Fund both. Use one submitting process per
signer.

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

> Do **not** set `BLOCKCHAIN_WS_ENDPOINT` or `CARTESI_BLOCKCHAIN_WS_ENDPOINT`.
> WebSocket RPC was removed in rollups-node alpha.12; the EVM reader uses HTTP
> polling only. `CARTESI_BLOCKCHAIN_HTTP_ENDPOINT` must be an http(s) URL with a
> DNS name or IP address. If the provider needs an auth header, use
> `CARTESI_BLOCKCHAIN_HTTP_AUTHORIZATION` instead of putting secrets in the URL.

For testnet Authority + withdrawal testing (e.g. `erc20-withdrawal-app`), use
`CARTESI_BLOCKCHAIN_DEFAULT_BLOCK=latest` so the node tracks the chain tip.
Use `finalized` when you need reorg-safe block anchoring: with `latest` or
`safe`, observations can change after a reorg and the node does not roll back
what it stored.

### Values by target network

| Network          | `BLOCKCHAIN_ID` | Example RPC (use your own key)                |
| ---------------- | --------------- | --------------------------------------------- |
| Ethereum Sepolia | `11155111`      | `https://sepolia.infura.io/v3/<KEY>`          |
| Local Anvil      | `31337`         | `http://host.docker.internal:8545`            |

The checked-in manifest was verified for Ethereum Sepolia and the local devnet.
For any other chain (Ethereum mainnet, Optimism, Base, Arbitrum and their
testnets), first confirm the `cartesi-rollups` 3.0.0-alpha.10 and Dave
3.0.0-alpha.6 deployments for that chain in the `cartesi/rollups-contracts` and
`cartesi/dave` releases — do not assume addresses carry over.

> **Security**: Never commit `.env` to version control. Add it to `.gitignore`.
> For the private key, use a funded wallet with only the minimum ETH needed
> for gas. `AUTH_KIND` accepts `private_key`, `private_key_file`, `mnemonic`,
> `mnemonic_file` and `aws`. For production, prefer `AUTH_KIND=aws` with AWS KMS
> or a `*_file` kind with a mounted secret.

### Contract addresses — contracts v3

Use **[`cartesi-contracts/cartesi-rollups-3.0.0-alpha.10.json`](../cartesi-contracts/cartesi-rollups-3.0.0-alpha.10.json)**
as the primary manifest. See **`cartesi-contracts`** for the full table.

Add to `.env` — **do not** trust the compose defaults:

```sh
# .env — cartesi-rollups 3.0.0-alpha.10 (Ethereum Sepolia)
CARTESI_CONTRACTS_INPUT_BOX_ADDRESS=0xEbE9f4Dfc04ae10bBeE663859c3dc5A23f94eA3C
CARTESI_CONTRACTS_AUTHORITY_FACTORY_ADDRESS=0xB4d29c86e36385b5321a453C34D288AEB0ad11f9
CARTESI_CONTRACTS_QUORUM_FACTORY_ADDRESS=0x0754D5Eb680c71bf469B39e48C5b64AB0813fdb9
CARTESI_CONTRACTS_APPLICATION_FACTORY_ADDRESS=0x35Cd91f13141Bb6A6FC69E1eeDD241bbA1Ddd45F
CARTESI_CONTRACTS_SELF_HOSTED_APPLICATION_FACTORY_ADDRESS=0x9e6866A965dC5f99f95EF6B0d8399dad18eEf98b

# PRT applications only — Dave 3.0.0-alpha.6, differs per chain (Sepolia shown)
CARTESI_CONTRACTS_DAVE_APP_FACTORY_ADDRESS=0xBD2a661328595113669a6b4c235126eec8c06A15
CARTESI_PRT_AUTH_PRIVATE_KEY=<funded-key-for-the-prt-service>
```

> **Local devnet**: for `cartesi run`, use `cartesi address-book` instead. The
> devnet `DaveAppFactory` address differs from Sepolia and mainnet.

### Other node settings worth knowing

```sh
# Set to false to run a reader-only node that never submits claims
CARTESI_FEATURE_CLAIM_SUBMISSION_ENABLED=true
# Cap on the fee per gas for node transactions (empty = no cap)
CARTESI_BLOCKCHAIN_TX_MAX_FEE_PER_GAS=
# Blocks fetched per eth_getLogs call; lower it if the RPC provider limits ranges
CARTESI_BLOCKCHAIN_MAX_BLOCK_RANGE=2000
```

`CARTESI_CLAIMER_MAX_ACCEPT_ATTEMPTS` was removed — do not set it. The full
list for a version is in `internal/config/generate/Config.toml` of the
rollups-node repository at that tag.

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

### Contracts v3 — standard deploy (Authority)

```sh
docker compose -f compose.local.yaml exec advancer \
  cartesi-rollups-cli deploy application <app-name> \
  /var/lib/cartesi-rollups-node/snapshot/ \
  --epoch-length 10 \
  --claim-staging-period <N> \
  --withdrawal-config-file <valid-config.json> \
  --salt <unique-32byte-hex-salt>
```

Without `--consensus`, this deploys the application **and** a new Authority
through the SelfHostedApplicationFactory, then registers and enables the
application on the node (`--register` and `--enable` default to `true`).

- `--epoch-length`: blocks per epoch (default `10`).
- `--claim-staging-period <N>`: blocks that must elapse after staging before
  `acceptClaim` is valid on-chain (default `0`).
- `--withdrawal-config-file`: JSON defining the guardian and accounts-drive
  layout (or inline with `--withdrawal-config`). Partial or invalid config is
  rejected before any on-chain tx.
- `--application-owner`, `--authority-owner`: set owners other than the signer.
- `--json`: machine-readable output.

Withdrawal config JSON:

```json
{
  "guardian": "0x...",
  "log2_leaves_per_account": 0,
  "log2_max_num_of_accounts": 0,
  "accounts_drive_start_index": 0,
  "withdrawal_output_builder": "0x..."
}
```

Take the three layout numbers from the `cartesi build` output (they are derived
from the drive marked `accounts_drive = true` in `cartesi.toml`) rather than
computing them by hand. `withdrawal_output_builder` is a contract deployed for
your asset (for USD-like tokens, through `UsdWithdrawalOutputBuilderFactory`).
See `cartesi-contracts`.

Replace:

- `<app-name>`: your dApp identifier (e.g. `my-dapp`)
- `<unique-32byte-hex-salt>`: a unique value to prevent address collisions.
  Generate one with:
  ```sh
  cast keccak "$(echo $RANDOM-$(date +%s))"
  ```

> **Record the application contract address** printed after this command.
> You will need it for InputBox calls, frontends, and L1 contracts.

### Deploy on a PRT (Dave) consensus

Requires the `prt` service (Step 3) and `CARTESI_CONTRACTS_DAVE_APP_FACTORY_ADDRESS`.

```sh
docker compose -f compose.local.yaml exec advancer \
  cartesi-rollups-cli deploy application <app-name> \
  /var/lib/cartesi-rollups-node/snapshot/ \
  --prt \
  --claim-staging-period 300 \
  --sentry-manager <address> \
  --sentries <address-1>,<address-2> \
  --salt <unique-32byte-hex-salt>
```

- `--sentries`: addresses appointed as PRT sentries. A claim the sentries agree
  with is accepted without waiting out the claim staging period; without
  agreement the staging period applies.
- `--sentry-manager`: account allowed to rotate the sentry addresses.
- With `--claim-staging-period 0` a staged claim is accepted at once, so
  sentries have no effect.

PRT epochs go `CLAIM_COMPUTED` → (tournament) → `CLAIM_STAGED` →
`CLAIM_ACCEPTED`. The epoch stays at `CLAIM_COMPUTED` while the tournament
runs, which takes much longer than an Authority claim — the node posts its
commitment right away, but settlement waits out the tournament's time windows.
A PRT application reports `epoch_length` `0x0`: epoch boundaries come from the
PRT consensus, so `--epoch-length` does not apply. Tournament progress is visible through
`cartesi-rollups-cli read tournaments|commitments|matches <app-name>` and the
JSON-RPC PRT methods (`cartesi-jsonrpc`).

### Deploy authority / quorum consensus separately

```sh
# A standalone Authority (owner defaults to the signer)
docker compose -f compose.local.yaml exec advancer \
  cartesi-rollups-cli deploy authority --epoch-length 10 --claim-staging-period <N>

# A Quorum of validators
docker compose -f compose.local.yaml exec advancer \
  cartesi-rollups-cli deploy quorum \
  --validator <validator-1> --validator <validator-2> --validator <validator-3> \
  --epoch-length 10 --claim-staging-period <N>
```

Pass the resulting address to `deploy application --consensus <address>`.
See `cartesi-contracts` for quorum staging behaviour (`CLAIM_SUBMITTED` →
event-driven `CLAIM_STAGED` → `CLAIM_ACCEPTED`).

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

If the automated `deploy application` fails (factory address mismatch, gas
issues, or custom consensus requirements), deploy the consensus first with
`cartesi-rollups-cli deploy authority` (above) and pass it explicitly:

```sh
docker compose -f compose.local.yaml exec advancer \
  cartesi-rollups-cli deploy application <app-name> \
  /var/lib/cartesi-rollups-node/snapshot/ \
  --consensus <authority-address> \
  --json
```

If you must call the factory directly, note the v3 signature takes the claim
staging period: `newAuthority(address authorityOwner, uint256 epochLength,
uint256 claimStagingPeriod)` (a second overload adds `bytes32 salt`). The old
two-argument call no longer exists.

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

# Node view of one application
docker compose -f compose.local.yaml exec advancer \
  cartesi-rollups-cli app status <app-name>

# On-chain view, read straight from the contracts (experimental command;
# takes the application ADDRESS and a subcommand)
docker compose -f compose.local.yaml exec advancer \
  cartesi-rollups-cli contract summary <application-address>
```

`contract` subcommands: `app`, `consensus`, `inputbox`, `epoch`, `output`,
`summary`, and for PRT `tournament`, `commitment`, `match`.

Expect `enabled`, `status`, `consensus_type`, `claim_staging_period`,
`withdrawal_config` in JSON-RPC and CLI reads — not the old single `state`
field. Healthy normal operation: `enabled=true`, `status=OK`,
`foreclose_block=0x0`. Other `status` values (`FAILED`, `DIVERGED`,
`CORRUPTED`, `GUEST_EXCEPTION`, `MACHINE_HALTED`, `MCYCLE_OVERFLOW`,
`UNEXPECTED_YIELD`, `INVALID_OUTPUTS_ROOT`) are explained in `cartesi-jsonrpc`.

### Confirm claim staging path

After sending an input and closing an epoch:

```sh
docker compose -f compose.local.yaml exec advancer \
  cartesi-rollups-cli read epochs <app-name>
```

Watch: `CLAIM_SUBMITTED` → `CLAIM_STAGED` → `CLAIM_ACCEPTED`.

### Send a test advance input

```sh
docker compose -f compose.local.yaml exec advancer \
  cartesi-rollups-cli send <app-name> "hello world" --yes
```

`--yes` skips the confirmation prompt, which cannot be answered through a
non-interactive `docker compose exec`.

Requires these env vars to be set (already in compose via `.env`):
`CARTESI_CONTRACTS_INPUT_BOX_ADDRESS`, `CARTESI_BLOCKCHAIN_HTTP_ENDPOINT`,
`CARTESI_AUTH_MNEMONIC` or `CARTESI_AUTH_PRIVATE_KEY`.

> **Signer conflict**: inside the node containers this command signs with the
> node's own claimer key. Do not send from an active node's signer while it is
> submitting claims — use it only for a quick smoke test on an idle app, or
> send from a separate wallet (`cast send` to the InputBox, or
> `-e CARTESI_AUTH_PRIVATE_KEY=<other-key>` on `docker compose exec`).

### Test an inspect request

```sh
# The request body is the raw payload — do not wrap it in JSON
curl -s -X POST "http://localhost:10012/inspect/<app-name>" \
  -H "Content-Type: application/octet-stream" \
  --data-binary '<route>'
```

### Read outputs via JSON-RPC

```sh
curl -s -X POST "http://localhost:10011/rpc" \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"cartesi_listOutputs","params":{"application":"<app-name>"},"id":1}'
```

Send `params` as an object (not an object wrapped in an array); results are under `result.data`.

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

# Re-enable after repairing a FAILED app, without a prompt
$EXEC app status <app-name> enabled --yes

# Remove an app and its stored data (disable first). --allow-abandonment
# acknowledges that protection, alerts and outstanding work are abandoned;
# --yes only skips the prompt. There is no --force flag.
$EXEC app remove <app-name> --allow-abandonment
$EXEC app remove <app-name> --allow-abandonment --yes

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

# PRT dispute data
$EXEC read tournaments <app-name>
$EXEC read commitments <app-name>
$EXEC read matches <app-name>

# Validate an output on-chain
$EXEC validate <app-name> <output-index>

# Execute a voucher on-chain
$EXEC execute <app-name> <output-index>
$EXEC execute <app-name> <output-index> --yes
```

---

## Step 8b — Operator CLI (contracts v3 emergency flows)

All commands run inside the advancer container. (For a local `cartesi run`
session the host CLI wraps the same flows: `cartesi foreclose`,
`cartesi withdraw`, `cartesi refund`.)

```sh
EXEC="docker compose -f compose.local.yaml exec advancer cartesi-rollups-cli"

# Guardian foreclose (signer must match withdrawal_config guardian)
$EXEC foreclose <app-name>

# Prove accounts-drive merkle root (after machine-tool generates proof)
$EXEC prove-drive-root <app-name> --proof-file drive-root-proof.json

# Withdraw account (gas payer can differ from recipient)
$EXEC withdraw <app-name> --proof-file account-proof.json

# Refund a deposit that was not finalized before foreclosure.
# --input-file is required: 0x-prefixed InputAdded.input bytes of that deposit
$EXEC refund <app-name> <input-index> --input-file input.hex

# Read recorded withdrawal rows
$EXEC read withdrawals <app-name>
$EXEC read withdrawals <app-name> <account-index>
```

`foreclose`, `prove-drive-root`, `withdraw` and `refund` prompt for
confirmation; add `--yes` in scripts.

Proof files come from `cartesi-rollups-machine-tool`, in two steps:

```sh
TOOL="docker compose -f compose.local.yaml exec advancer cartesi-rollups-machine-tool"

# 1. Replay accepted inputs into a copy of the template
$TOOL replay \
  --template /var/lib/cartesi-rollups-node/snapshot/ \
  --application <app-name> \
  --store /tmp/replayed

# 2. Produce both proof files for one account
$TOOL prove accounts-drive \
  --snapshot /tmp/replayed \
  --account <account-address> \
  --accounts-drive-start-index <N> \
  --log2-max-num-of-accounts <N> \
  --log2-leaves-per-account <N> \
  --out-drive-root-proof drive-root-proof.json \
  --out-withdraw-proof account-proof.json
```

Use the layout values from the application's `withdrawal_config`. `replay`
reads the database from `CARTESI_DATABASE_CONNECTION` (or
`--database-connection`) and accepts `--to-epoch` / `--to-input-index`. Full
sequence: `cartesi-contracts`.

After foreclosure the application keeps its `status`; it is identified by a
non-zero `foreclose_block`. Do **not** disable the app automatically if you
still need post-foreclosure L1 observation (drive-prove and withdrawal events).

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

# Stop and wipe all data (full reset — required when bumping node versions,
# e.g. alpha.12/alpha.13 → alpha.14, or when changing the default block policy)
docker compose -f compose.local.yaml down -v

# Restart after code changes:
# 1. Rebuild the machine image
cartesi build

# 2. Restart the node (advancer will pick up new snapshot on startup)
docker compose -f compose.local.yaml down
docker compose -f compose.local.yaml --env-file .env up -d
```

A rebuilt image has a different template hash. An application already deployed
on-chain is bound to its original hash, so changed code means a **new
`deploy application`** (new salt, new application address), not just a restart.
The node checks the machine hash at load (`CARTESI_FEATURE_MACHINE_HASH_CHECK_ENABLED`).

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
| `prt`         | `10006` | Telemetry / health (PRT apps)  |
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
- All Docker Compose service statuses from `docker compose ps` (six, plus `prt` for PRT apps)
- Consensus type (Authority / Quorum / PRT) and, for PRT, the sentries and sentry manager
- Confirm app appears in `cartesi-rollups-cli app list` with `enabled=true`, `status=OK`
- `claim_staging_period` and `withdrawal_config` recorded from deploy
- Epoch reached `CLAIM_ACCEPTED` (normally via `CLAIM_STAGED`; a node catching up in reader mode can record `CLAIM_ACCEPTED` directly)
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
- [`cartesi-contracts/cartesi-rollups-3.0.0-alpha.10.json`](../cartesi-contracts/cartesi-rollups-3.0.0-alpha.10.json) — checked-in infrastructure addresses
- [cartesi/rollups-contracts v3.0.0-alpha.10](https://github.com/cartesi/rollups-contracts/releases/tag/v3.0.0-alpha.10) — deployment artifacts per chain
- [cartesi/dave v3.0.0-alpha.6](https://github.com/cartesi/dave/releases/tag/v3.0.0-alpha.6) — PRT contracts and deployment addresses
- [Cartesi Rollups node v2.0.0-alpha.14 release](https://github.com/cartesi/rollups-node/releases/tag/v2.0.0-alpha.14) — node binary release; `compose.yaml` and `compose.individual-services.yaml` at that tag are the reference compose files
- [Cartesi CLI 2.0.0-alpha.42 release](https://github.com/cartesi/cli/releases/tag/%40cartesi%2Fcli%402.0.0-alpha.42) — host CLI that defaults to SDK 0.12.0-alpha.45 (node alpha.14)

## Agent checklist

- [ ] Detected CLI version with `cartesi --version` (`2.0.0-alpha.42` or newer for node alpha.14)
- [ ] Detected project version from Dockerfile (`MACHINE_EMULATOR_TOOLS_VERSION` vs `MACHINE_GUEST_TOOLS_VERSION`)
- [ ] Ran `cartesi build` with that CLI — snapshot at `.cartesi/image/` (machines built for 0.21 are not reusable)
- [ ] Downloaded `compose.local.yaml` to project root
- [ ] Patched compose for alpha.14: images `0.12.0-alpha.45`, no `CARTESI_BLOCKCHAIN_WS_ENDPOINT`, one default block for all services, `prt` service if PRT
- [ ] Set contract addresses from `cartesi-rollups-3.0.0-alpha.10.json` in `.env` — not compose defaults, not alpha.6, not v2.2.0
- [ ] Started on a fresh database (`down -v` before the first alpha.14 start)
- [ ] Verified `cartesi-rollups-node --version` reports `2.0.0-alpha.14` inside advancer
- [ ] Started node: `docker compose -f compose.local.yaml --env-file .env up -d`
- [ ] Verified all containers healthy: `docker compose -f compose.local.yaml ps`
- [ ] Ran `cartesi-rollups-cli deploy application` inside advancer — recorded app contract address
- [ ] On v3 deploy: passed `--claim-staging-period` and valid `--withdrawal-config-file`
- [ ] For PRT: passed `--prt`, sentries and sentry manager; funded the `CARTESI_PRT_AUTH_*` key
- [ ] Sent test advance input (not from the node's active signer) and read output to confirm end-to-end
- [ ] Confirmed `app status <app>` shows `enabled`, `status=OK`
- [ ] Observed the test epoch reach `CLAIM_ACCEPTED` (normally after `CLAIM_STAGED`)

## What comes next

| Next task                     | Skill to use           |
| ----------------------------- | ---------------------- |
| Wire L1 contracts to InputBox | `cartesi-contracts` |
| Debug node or app issues      | `cartesi-debug`        |
