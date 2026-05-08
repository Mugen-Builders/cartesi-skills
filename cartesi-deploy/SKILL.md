---
name: cartesi-deploy
description: >-
  Deploy and operate Cartesi Rollups v2 applications across local, forked, and
  self-hosted node environments. Covers build, node bring-up, app registration,
  address capture, and deployment troubleshooting. Use when the task is about
  shipping/running infrastructure and application deployment, not business logic.
---

# Cartesi Deploy

## Objective

Use this skill when the user asks to deploy, register, or operate a Cartesi Rollups v2 application.

Scope includes:

- Build artifacts and snapshots;
- Local and forked run commands;
- Self-hosted rollups node setup;
- Application registration and address persistence;
- Deployment and chain-configuration troubleshooting.

Out of scope:

- Implementing app business logic (`advance_state`/`inspect_state`);
- Frontend UI development;
- Deep Solidity contract authoring beyond deployment wiring.

## Authoritative references

- Cartesi Rollups 2.0 docs root:
  [https://docs.cartesi.io/cartesi-rollups/2.0/](https://docs.cartesi.io/cartesi-rollups/2.0/)
- Self-hosted deployment:
  [https://docs.cartesi.io/cartesi-rollups/2.0/deployment/self-hosted/](https://docs.cartesi.io/cartesi-rollups/2.0/deployment/self-hosted/)
- Development installation:
  [https://docs.cartesi.io/cartesi-rollups/2.0/development/installation/](https://docs.cartesi.io/cartesi-rollups/2.0/development/installation/)

When docs conflict with older local notes, prefer official docs and call out the mismatch.

## Preconditions checklist

- Docker is running.
- Cartesi CLI is installed (`cartesi --version`).
- Project can produce a valid snapshot (`cartesi build` succeeds).
- Target chain endpoints and funded deploy key are available when using self-hosted mode.

## Deployment modes

### 1) Local run (fast dev loop)

Use for feature iteration and functional checks:

```sh
cartesi build
cartesi run -p <port>
```

Record runtime URLs printed by the command (inspect and RPC paths vary by setup).

### 2) Forked run (realistic chain state)

Use when app behavior depends on existing chain contracts/state:

```sh
cartesi run --fork-url <RPC_URL> [--fork-block-number <N>] -p <port>
```

Notes:

- Fork reads are static for a given fork point unless you advance/change it.
- Chain-id alignment still matters for signer and client tooling.

### 3) Self-hosted rollups node (testnet-style)

Use when the user needs a full node-style environment:

1. Create `.env` with chain/auth values expected by the compose stack.
2. Retrieve `compose.local.yaml` from the official deployment setup.
3. Build snapshot with `cartesi build`.
4. Start services with Docker Compose.
5. Register the app via `cartesi-rollups-cli deploy application ... --register`.
6. Persist deployed application address in project config/docs.

#### Self-hosted command recipe (exact steps)

```sh
# 1) Configure chain/auth values
cat > .env <<'EOF'
BLOCKCHAIN_ID=<blockchain-id>
AUTH_KIND="private_key"
CARTESI_AUTH_PRIVATE_KEY="<funded-private-key>"
BLOCKCHAIN_WS_ENDPOINT="<ws-endpoint>"
BLOCKCHAIN_HTTP_ENDPOINT="<http-endpoint>"
CARTESI_BLOCKCHAIN_DEFAULT_BLOCK="<latest-or-finalized>"
EOF

# 2) Get the official compose file
curl -L https://raw.githubusercontent.com/Mugen-Builders/deployment-setup-v2.0/main/compose.local.yaml -o compose.local.yaml

# 3) Build snapshot
cartesi build

# 4) Bring up self-hosted rollups node
docker compose -f compose.local.yaml --env-file .env up -d

# 5) Register app on the node (replace <app-name> and <salt>)
docker compose --project-name cartesi-rollups-node \
  exec advancer cartesi-rollups-cli deploy application <app-name> /var/lib/cartesi-rollups-node/snapshot \
  --epoch-length 10 \
  --salt <salt> \
  --register
```

Salt helper:

```sh
cast keccak256 "your-unique-string"
```

Quick verification:

```sh
# Inspect endpoint (replace <app-name>)
curl -s -X POST "http://localhost:10012/inspect/<app-name>" \
  -H "Content-Type: application/json" \
  -d '{"payload":"0x"}'

# JSON-RPC endpoint
curl -s "http://localhost:10011/rpc" \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"get_status","params":[]}'
```

Teardown:

```sh
docker compose -f compose.local.yaml --env-file .env down
```

## Environment and secrets policy

- Never hardcode private keys in code or committed files.
- Use `.env` and `.env.example` separation.
- For generated docs, mask key-like values.
- Explicitly document required variables and expected formats.

## Address management requirements

Always capture and store:

- Application address;
- InputBox and portal addresses (if relevant for task);
- Chain id;
- Inspect and JSON-RPC URLs for the selected mode.

For chain-specific contracts, use official deployed-contract references and/or `cartesi address-book`.

## Operational runbook

For each deployment task, produce:

1. Exact commands executed (or to execute).
2. Expected success signals (logs/endpoints/status).
3. Verification commands (`curl` inspect, input send, and output check).
4. Rollback/teardown command (`docker compose ... down`, stop commands).

## Common failure signatures

- `invalid chain id for signer`: signer chain does not match RPC chain.
- Deploy/register failure: missing funds, wrong chain id, wrong endpoint, or stale/mismatched snapshot.
- Inspect works but no state changes: no advance accepted, wrong app address, or no processed input.
- Port confusion: client points to wrong inspect/RPC endpoint.

## Required handoff format

When completing a deploy task, return:

- Deployment mode used (`local`, `forked`, `self-hosted`);
- App address and relevant contract addresses;
- Endpoints (inspect and JSON-RPC);
- Verified checks performed;
- Remaining risks or follow-up actions.

## Skill boundaries

If the user asks for domain/business logic implementation, route to:

- `cartesi-backend-core` plus language-specific backend skill.

If the user asks for Solidity integration contracts, route to:

- `cartesi-l1-contracts`.

If the user asks for frontend interaction flow, route to:

- `cartesi-frontend`.
