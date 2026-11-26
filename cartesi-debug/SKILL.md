---
name: cartesi-debug
version: 0.3.0
description: >-
  Diagnose and fix errors across the Cartesi Rollups stack including contracts
  v3 lifecycle issues (claim staging, foreclosure, emergency withdrawal,
  enabled/status). Use whenever the user hits an error or unexpected behaviour
  at any layer. Triggers on: "error", "failed", "not working", "stuck",
  "CLAIM_STAGED", "FAILED", "DIVERGED", "CORRUPTED", "foreclose", "withdrawal",
  "acceptClaim", "staging period", "PRT", "tournament", "debug",
  "troubleshoot".
---

## Skill Version

| Skill           | Version | Cartesi Rollups target | Compose setup       | Last updated |
| --------------- | ------- | ------------------------------------ | ------------------- | ------------ |
| `cartesi-debug` | `0.3.0` | contracts v3 — rollups-node `v2.0.0-alpha.14`, Cartesi CLI `2.0.0-alpha.42` | Mugen-Builders v2.0 | Nov 2026     |

> Lifecycle semantics and operator flows: `cartesi-contracts`. JSON-RPC field
> reference: `cartesi-jsonrpc`.

> **`cartesi-rollups-cli` execution context**: every `cartesi-rollups-cli`
> command in this file runs **inside the advancer container** of a
> compose-based self-hosted deployment. Prefix each command with
> `docker compose -f compose.local.yaml exec advancer …`. With `cartesi run`,
> `cartesi-rollups-cli` is not on the host — use `curl` against the inspect
> endpoint and the JSON-RPC API (see `cartesi-local-dev` and
> `cartesi-jsonrpc`), `cartesi logs` / `cartesi status`, or run the binary in
> the session's node container:
> `docker compose -p <app-name> exec rollups_node cartesi-rollups-cli …`.

# Cartesi Rollups — Debugging and Troubleshooting

## Goal

Quickly identify and resolve issues anywhere in the Cartesi stack. This skill
is organised by layer and symptom. Start with the symptom that matches the
user's problem, work through the diagnosis steps, and apply the fix.

---

## Layer 0 — CLI version and project version issues (check first)

These are the most common source of confusion. Always resolve version
mismatches before diagnosing anything else.

### Symptom: `cartesi deploy` does nothing / "deploy command is deprecated"

```
deploy command is deprecated, use 'cartesi-rollups-cli' directly instead
```

**Cause**: In Cartesi CLI v2.0-alpha `cartesi deploy` is a deprecated stub
(older alphas had no such command at all). Deployment in v2 is done via Docker
Compose and `cartesi-rollups-cli`.

**Fix**: Use the `cartesi-deploy` skill. The correct deployment flow is:

1. `cartesi build` to build the machine snapshot
2. Download `compose.local.yaml` from the Mugen-Builders repo and patch it
3. Create `.env` with your chain configuration
4. `docker compose -f compose.local.yaml --env-file .env up -d`
5. Register the app with `cartesi-rollups-cli deploy application` inside the
   advancer container

### Symptom: Unknown command or flag not recognised

Check which CLI version is installed:

```sh
cartesi --version
```

Then check available commands:

```sh
cartesi --help
```

If the user appears to have both versions installed (for example npm-global
plus a Homebrew-managed binary), list every `cartesi` binary on `PATH` and
inspect each:

```sh
which -a cartesi               # show every cartesi binary on PATH
npm ls -g @cartesi/cli         # version installed by npm (if any)
brew list --versions cartesi   # version installed by Homebrew (if any)
```

Use the CLI that matches the project version (check Dockerfile). If the
wrong one wins on `PATH`, either reorder `PATH`, alias `cartesi` to the
intended binary, or uninstall the unwanted version.

### Symptom: Built with wrong CLI version / image incompatible

A project built with v1.5 CLI cannot be run with a v2.0-alpha node and
vice versa. Identify the project version:

```sh
grep -E "MACHINE_EMULATOR_TOOLS_VERSION|MACHINE_GUEST_TOOLS_VERSION" Dockerfile
```

| Match                            | Project version | Correct CLI                      |
| -------------------------------- | --------------- | -------------------------------- |
| `MACHINE_EMULATOR_TOOLS_VERSION` | v1.5            | `cartesi` (v1.5)                 |
| `MACHINE_GUEST_TOOLS_VERSION`    | v2.0-alpha      | `cartesi` (v2.0-alpha installed) |

Additional Dockerfile signals that confirm the version:

| Signal in Dockerfile               | Version            |
| ---------------------------------- | ------------------ |
| `cartesi/python:3.10-slim-jammy`   | v1.5 base image    |
| `cartesi/python:3.13.2-slim-noble` | v2.0-alpha base    |
| Single-stage build                 | v1.5 pattern       |
| Multi-stage build (`AS base`)      | v2.0-alpha pattern |
| `APT_UPDATE_SNAPSHOT=...`          | v2.0-alpha only    |
| `ENTRYPOINT ["rollup-init"]`       | Not a version marker — v1.5 and most v2 templates use it; v2 JS/TS templates use `ENTRYPOINT ["node"]` with `@cartesi/rollup` |
| Tools installed as `.deb` package  | v2.0-alpha         |
| Tools installed as `.tar.gz`       | v1.5               |

If the wrong CLI was used to build, clean the build and rebuild:

```sh
rm -rf .cartesi/image/
cartesi build
```

### Symptom: Machine built by an older CLI will not load

After upgrading to CLI alpha.42 / node alpha.14 the node or `cartesi run`
refuses an existing `.cartesi/image/`.

**Cause**: alpha.42 requires cartesi-machine 0.22. Machines stored by 0.21 or
earlier cannot be loaded, and all machine hashes change.

**Fix**: `cartesi build` again with the new CLI, then redeploy the application
(the template hash is part of the on-chain application). Start the node on a
fresh database.

### Symptom: `Unknown key '<key>' in [<table>]` from `cartesi build` / `run`

**Cause**: since CLI alpha.41, `cartesi.toml` keys the CLI does not read are
errors — a typo, a key of another drive builder, or a removed feature such as
`[withdrawal.config]`.

**Fix**: the message lists the accepted keys for that table; correct or remove
the key. Replace `[withdrawal.config]` by marking the accounts drive with
`accounts_drive = true` (see `cartesi-contracts`).

### Symptom: `cartesi send` rejects `--hex`, or the input comes from the wrong account

`cartesi send` exists in v2.0-alpha. Use `--encoding hex|string|abi|abi-packed`
(there is no `--hex` flag). Since alpha.42 `send` and `deposit` sign with
**devnet account 8** by default, because account 0 signs the node's claims;
pass `--from <address>` to choose another devnet account. A backend that
hard-codes the old default sender (`0xf39F…2266`) will see a different
`msg_sender`.

To send with `cast` instead, target the proxied Anvil endpoint:

```sh
cast send $(cartesi address-book InputBox) \
  "addInput(address,bytes)" $(cartesi address-book Application) <hex-payload> \
  --rpc-url http://127.0.0.1:6751/anvil \
  --private-key <devnet-key-other-than-account-0>
```

Or use the `cartesi-rollups-cli send` command inside the compose advancer:

```sh
docker compose -f compose.local.yaml exec advancer \
  cartesi-rollups-cli send <app-name> "your-payload"
```

---

## Layer 1 — Build failures (`cartesi build`)

### Symptom: `cartesi build` fails immediately

**Check Docker is running:**

```sh
docker info
```

If Docker is not running, start Docker Desktop or the daemon.

**Check Cartesi CLI is installed:**

```sh
cartesi --version
```

### Symptom: Build fails with missing package / dependency error

The Dockerfile defines the Cartesi Machine Linux environment. System
packages must be installed inside the Dockerfile, not on the host.

```dockerfile
# Wrong: installed on host, not inside machine
# apt install libsomething

# Correct: install inside the Dockerfile
RUN apt-get update && apt-get install -y libsomething
```

After fixing the Dockerfile, rebuild:

```sh
cartesi build
```

### Symptom: Language runtime version mismatch

Check the Dockerfile base image matches the language version you're using.
For Node.js:

```dockerfile
FROM node:20-alpine   # pin the version explicitly
```

---

## Layer 2 — Local run failures (`cartesi run`)

### Symptom: "application name is already in use"

A previous `cartesi run` session left a Docker Compose project running.
Tear it down and retry:

```sh
docker compose -p <app-name> down
cartesi run
```

Replace `<app-name>` with the name from `cartesi create`.

### Symptom: `ECONNREFUSED` on `localhost:8545`

`cartesi run` does **not** expose Anvil on port 8545. Everything is behind one
port (default `6751`, printed as `starting at http://127.0.0.1:<port>`):

| Path | Service |
| ---- | ------- |
| `/anvil` | L1 JSON-RPC (use as `--rpc-url`, wagmi/viem transport, forge) |
| `/rpc` | Cartesi JSON-RPC API |
| `/inspect/<app-name>` | Inspect |
| `/explorer` | Explorer — only when started with `--services explorer` |

If `6751` is taken the CLI picks the next free port up to `6761` — read the
printed address and update every command and frontend config.

### Symptom: `cartesi run` rejects `CARTESI_BLOCKCHAIN_HTTP_ENDPOINT` or `--default-block pending`

- A `CARTESI_BLOCKCHAIN_HTTP_ENDPOINT` exported in the shell must be an
  http(s) URL with a DNS name or IP address; the CLI now validates it before
  starting. Unset it for a plain local run.
- `--default-block` accepts only `latest`, `safe` or `finalized`.

### Symptom: Inspect URL returns 404, 503 or connection refused

- Confirm the URL format: `http://127.0.0.1:<port>/inspect/<app-name>`
- Confirm `<port>` matches what `cartesi run` printed
- `404`: the app name is unknown — it is the project (directory) name unless
  `--project-name` was passed
- `503`: the machine is not ready yet, the application was foreclosed, its
  execution terminated, or inspect capacity is exhausted
- Check the environment: `cartesi status`, `cartesi logs`

### Symptom: Inspect handler receives `{"payload":"0x..."}` instead of the route

The node passes the **raw request body** to the backend as the inspect payload.
A JSON-wrapped body is delivered literally, so route matching fails silently.

```sh
# Wrong
curl -X POST .../inspect/<app> -H "Content-Type: application/json" -d '{"payload":"0x7374617473"}'
# Right
curl -X POST .../inspect/<app> -H "Content-Type: application/octet-stream" --data-binary 'stats'
```

In a frontend, send the bytes as the `fetch` body, not `JSON.stringify(...)`.

### Symptom: JSON-RPC returns `-32602 Invalid parameters`, or the result looks empty

- `params` should be an **object** (`"params":{"application":"my-dapp"}`), not an
  array wrapping the object.
- Indexes are hex strings (`"0x0"`).
- Results are under `result.data` (lists also have `result.pagination`). Code
  reading `result.status` or `parseInt(result, 16)` gets `undefined` / `NaN`.

### Symptom: State seems unchanged after advance input

Possible causes:

1. **Input rejected**: the backend returned `"reject"`. Read reports to see
   the error:

   ```sh
   # With cartesi run — use JSON-RPC (default port 6751)
   curl -s -X POST "http://127.0.0.1:6751/rpc" \
     -H "Content-Type: application/json" \
     -d '{"jsonrpc":"2.0","id":1,"method":"cartesi_listReports","params":{"application":"<app-name>"}}'

   # With compose deployment — use cartesi-rollups-cli inside advancer
   docker compose -f compose.local.yaml exec advancer \
     cartesi-rollups-cli read reports <app-name>
   ```

2. **Wrong app address**: the input was sent to the wrong application.
3. **Backend restarted and lost ephemeral state**: state is in-memory.
   Resend from scratch or use snapshots.
4. **Input not yet processed**: wait and re-check, or lower polling interval.
5. **Application stopped**: check `cartesi_getApplication` — a `status` other
   than `OK` (see Layer 9) or `enabled=false` means inputs are not processed.
   The input's own `status` (`REJECTED`, `EXCEPTION`, `MACHINE_HALTED`,
   `OVERFLOW`, `UNEXPECTED_YIELD`, `INVALID_OUTPUTS_ROOT`) says how it ended.

---

## Layer 3 — Node startup failures

### Symptom: Node won't start — database connection error

```
error: failed to connect to postgres
```

Checks:

1. Is Postgres running?
   ```sh
   docker ps | grep postgres
   ```
2. Is `CARTESI_DATABASE_CONNECTION` set and correct?
   ```sh
   echo $CARTESI_DATABASE_CONNECTION
   # Expected: postgres://user:password@hostname:port/database
   ```
3. Have migrations been run?
   ```sh
   cartesi-rollups-cli db init
   ```
4. If using Docker networking, use `host.docker.internal` not `localhost`:
   ```sh
   export CARTESI_DATABASE_CONNECTION=postgres://postgres:password@host.docker.internal:5432/postgres?sslmode=disable
   ```

### Symptom: Node exits immediately — max startup time exceeded

```
error: service exceeded max startup time
```

Increase the startup timeout:

```sh
export CARTESI_MAX_STARTUP_TIME=60
```

Check logs for underlying cause before increasing:

```sh
export CARTESI_LOG_LEVEL=debug
cartesi-rollups-node
```

### Symptom: Node logs show blockchain connection errors

```
error: failed to connect to blockchain HTTP endpoint
```

Checks:

1. Is the RPC endpoint correct and reachable?
   ```sh
   curl -s http://localhost:8545 -X POST \
     -H "Content-Type: application/json" \
     -d '{"jsonrpc":"2.0","method":"eth_chainId","params":[],"id":1}'
   ```
2. Is `CARTESI_BLOCKCHAIN_ID` set to the correct chain ID?
3. `CARTESI_BLOCKCHAIN_WS_ENDPOINT` is **not** used any more — the node polls
   over HTTP only. Remove it from old configs.
4. Does the provider limit `eth_getLogs` ranges? Lower
   `CARTESI_BLOCKCHAIN_MAX_BLOCK_RANGE` (default `2000`).

### Symptom: Service refuses to start after a configuration change

**Cause**: alpha.14 saves some settings in the database the first time each
service starts — the default block policy (`CARTESI_BLOCKCHAIN_DEFAULT_BLOCK`)
and the claim-submission mode (`CARTESI_FEATURE_CLAIM_SUBMISSION_ENABLED`). A
later start with a different value stops.

**Fix**: restore the original value, or start over with a fresh database.
All services (EVM Reader, Claimer, PRT) must share the same default block.

---

## Layer 4 — Application deployment failures

### Symptom: `deploy application` fails with "insufficient funds"

The wallet derived from `CARTESI_AUTH_MNEMONIC` has no ETH on the target chain.
For Anvil devnet, use the default funded mnemonic:

```sh
export CARTESI_AUTH_MNEMONIC="test test test test test test test test test test test junk"
```

For testnets, fund the address from a faucet.

### Symptom: `deploy application` fails with unknown selector / revert

Check:

1. Contract addresses match the target chain. Verify with:
   ```sh
   cartesi address-book
   ```
2. The ABI used client-side includes custom errors from the contract.
   Look up unknown selectors at https://www.4byte.directory/

### Symptom: Deterministic deploy collision

If deploying the same app image with the same owner address again, the
contract address will collide. Use a different salt:

```sh
cartesi-rollups-cli deploy application myapp .cartesi/image/ \
  --salt $(cast keccak "your-unique-string-$(date +%s)")
```

### Symptom: `deploy application` succeeds but app is not processing inputs

The application must be **enabled** on the node. Check status:

```sh
cartesi-rollups-cli app status myapp
```

If disabled, enable it:

```sh
cartesi-rollups-cli app status myapp enabled
```

### Symptom: PRT deploy fails, or a PRT app never advances

1. `CARTESI_CONTRACTS_DAVE_APP_FACTORY_ADDRESS` must be set to the Dave
   3.0.0-alpha.6 factory **for that chain** (it differs per chain).
2. The `prt` service must be running (`cartesi-rollups-prt`) and its own
   signer (`CARTESI_PRT_AUTH_*`, separate from `CARTESI_AUTH_*`) must be
   funded.
3. Follow the tournament: `cartesi-rollups-cli read tournaments <app>`,
   `read commitments <app>`, `read matches <app>`, or the JSON-RPC PRT methods.
4. A long wait at `CLAIM_STAGED` is the claim staging period; sentries that
   agree with the claim shorten it.

---

## Layer 5 — Advance input not processed

### Symptom: Input sent but never appears in node reads

Check if the EVM Reader is reading the correct InputBox address:

```sh
echo $CARTESI_CONTRACTS_INPUT_BOX_ADDRESS
# Should match the address on the target chain
```

Check the node logs for EVM Reader activity:

```sh
export CARTESI_LOG_LEVEL=debug
```

Verify the input was actually submitted on-chain:

```sh
cast logs \
  --address <InputBox-address> \
  --from-block 0 \
  --to-block latest \
  --rpc-url http://localhost:8545
```

### Symptom: Input appears in node but state is wrong

Read reports for that input to see what the backend returned:

```sh
cartesi-rollups-cli read reports <app-name>
```

If the report shows a backend error, the issue is in the advance handler.
Add more logging inside the handler and rebuild:

```sh
cartesi build && cartesi run
```

### Symptom: "invalid chain id for signer"

The wallet/signer chain ID does not match the RPC endpoint's chain ID.

Fix: align `CARTESI_BLOCKCHAIN_ID` with the RPC chain:

```sh
# Check actual chain ID from RPC
cast chain-id --rpc-url http://localhost:8545

# Set it
export CARTESI_BLOCKCHAIN_ID=31337   # adjust to actual value
```

---

## Layer 6 — Inspect issues

### Symptom: Inspect returns empty or no reports

1. Confirm the inspect endpoint URL is correct:
   ```sh
   export CARTESI_INSPECT_URL=http://localhost:10012
   ```
   (`CARTESI_INSPECT_ADDRESS` is the service's listen address, `:10012`.)
2. Confirm the route string matches what the backend handles:
   ```sh
   # Backend handles route "stats" — call with exactly "stats"
   cartesi-rollups-cli inspect myapp "stats"
   ```
3. Make sure the inspect handler emits at least one report — even on error.

### Symptom: Inspect returns stale data

Inspect forks the **current** machine state. If the Advancer is still
processing a large advance, inspect sees the state _before_ that advance.
Wait for advance processing to complete, then re-inspect.

### Symptom: Parallel inspects fail or timeout

Increase `max_concurrent_inspects` in execution parameters:

```sh
cartesi-rollups-cli app execution-parameters set myapp max_concurrent_inspects 20
```

---

## Layer 7 — Forked chain issues

### Symptom: Oracle/external data reads return identical values

Expected behaviour. A forked chain is static at the chosen block height.
To test changing data:

- Re-fork at a later block: `cartesi run --fork-url <RPC> --fork-block-number <N>`
- Or send advance inputs directly (bypassing L1) to simulate data changes.

### Symptom: Fork not loading the correct state

Confirm the RPC URL supports historical state at the chosen block:

```sh
cast block <block-number> --rpc-url <RPC_URL>
```

---

## Layer 8 — Voucher execution failures

### Symptom: Voucher cannot be executed — epoch not yet accepted

Vouchers require the epoch to be closed and the claim **accepted**. On
contracts v3, acceptance follows staging:

```sh
cartesi-rollups-cli read epochs <app-name>
# Authority/Quorum: CLAIM_SUBMITTED → CLAIM_STAGED → CLAIM_ACCEPTED
# PRT:              CLAIM_COMPUTED → (tournament) → CLAIM_STAGED → CLAIM_ACCEPTED
```

With `cartesi run`, the default epoch length is 720 blocks — pass
`--epoch-length 10` to see epochs accepted quickly.

If the epoch is `CLAIM_SUBMITTED` or `CLAIM_STAGED`, wait for the staging
period to elapse and for `acceptClaim` to succeed. Reduce epoch length or
`claim_staging_period` in development.

### Symptom: Epoch stuck at CLAIM_STAGED

1. Confirm enough blocks have passed (`claim_staging_period` from deploy).
2. Check claimer logs for repeated `acceptClaim` failures.
3. `CARTESI_CLAIMER_MAX_ACCEPT_ATTEMPTS` no longer exists in alpha.14 — remove
   it from old configs. If the application moved to `FAILED`, read `reason`
   (`app status` / `cartesi_getApplication`), repair the cause, then re-enable
   with `app status <app> enabled`.
4. Early `acceptClaim` (before staging period) reverts on-chain — expected.
5. Check the signer is funded and is not shared with another sending process.

---

## Layer 9 — Contracts v3 lifecycle

### Symptom: `enabled` / `status` missing in API or CLI reads

**Cause:** Old node binary, wrong contract suite, or JSON-RPC client still
expecting the deprecated `state` field.

**Fix:** Verify the `cartesi/rollups-runtime` image tag in `compose.local.yaml`
(`0.12.0-alpha.45` for node alpha.14, contracts `cartesi-rollups`
3.0.0-alpha.10). Update clients to read `enabled` + `status` from
`result.data`. See `cartesi-jsonrpc`.

### Symptom: Node behaves erratically after upgrade

**Cause:** Reused a database created by an earlier node. The single migration
(`000001`) is edited in place between alphas while the schema version stays
the same, so the node's version check does not catch it.

**Fix:** `docker compose -f compose.local.yaml down -v` and start with a
fresh database on every node version bump (required for alpha.14).

### Symptom: App `status` is not `OK` — what does it mean?

| `status` | Meaning | Action |
| -------- | ------- | ------ |
| `FAILED` | Recoverable node-side failure (for example OOM or a process crash) | Read `reason`, repair, verify snapshot/database consistency, then `app status <app> enabled` (clears `FAILED`) |
| `DIVERGED` | The node's computed claim disagrees with the chain | Terminal for this node instance — investigate determinism and the template hash; do not just re-enable |
| `CORRUPTED` | Local state missing or inconsistent | Terminal — rebuild the node state from a fresh database |
| `GUEST_EXCEPTION`, `MACHINE_HALTED`, `MCYCLE_OVERFLOW`, `UNEXPECTED_YIELD`, `INVALID_OUTPUTS_ROOT` | The application's own execution ended the machine | Fix the backend (see `reason` and the last input's `status`), rebuild and redeploy |

The `INOPERABLE` and `FORECLOSED` values of earlier alphas no longer exist.

### Symptom: App was foreclosed — is this corruption?

**No.** Foreclosure is the normal emergency path after guardian `foreclose()`.
It is not a `status` value: a healthy foreclosed app is typically
`enabled=true`, `status=OK`, with a non-zero `foreclose_block`. Inspect
requests return `503` for a foreclosed app.

### Symptom: Foreclose succeeds but no withdrawal rows

**Cause:** Accounts-drive merkle root not proved yet.

**Fix:** Run the full sequence — machine-tool replay/prove →
`prove-drive-root` → `withdraw`. Withdrawal rows appear only after drive root
is proved on L1. See `cartesi-deploy` Step 8b and `cartesi-contracts`.

### Symptom: `cartesi-rollups-cli foreclose` / `withdraw` not found

**Cause:** `cartesi-rollups-cli` is not installed on the host.

**Fix:** With `cartesi run`, use the host CLI wrappers — `cartesi foreclose`,
`cartesi withdraw --account <address>`, `cartesi refund <input-index>`. With a
compose deployment, run `cartesi-rollups-cli` inside the advancer container
(`cartesi-deploy` Step 8b).

### Symptom: Deposit made just before foreclosure is missing

**Cause:** its epoch was never accepted, so it is not in the proved accounts
drive.

**Fix:** refund it — `cartesi-rollups-cli refund <app> <input-index> --input-file <file>`
(the file holds the 0x-prefixed `InputAdded.input` bytes; required there) (or
`cartesi refund <input-index>` locally).

### Symptom: Bad withdrawal config at deploy

Partial or invalid `withdrawal_config` should be rejected before any on-chain
tx. If deploy succeeded with wrong guardian or drive layout, emergency recovery
will fail — redeploy with corrected config.

---

## Layer 10 — Voucher validation (on-chain)

1. Verify the output proof was generated (epoch must be CLAIM_ACCEPTED).
2. Check the destination contract is deployed and ABI matches.
3. Validate before executing:
   ```sh
   cartesi-rollups-cli validate <app-name> <output-index>
   ```

---

## General debug toolkit

### Enable verbose logging

```sh
export CARTESI_LOG_LEVEL=debug
cartesi-rollups-node
```

### Check all running containers

```sh
docker ps
docker logs <container-id>

# With cartesi run
cartesi status
cartesi logs -f                 # node + PRT nodes; `cartesi logs <service>` for one
```

Per-service log levels: `CARTESI_LOG_LEVEL_ADVANCER`, `_CLAIMER`,
`_EVM_READER`, `_JSONRPC_API`, `_PRT`, `_VALIDATOR`.

### Inspect database state directly

```sh
psql "postgres://postgres:password@localhost:5432/rollupsdb"
# List tables (application, epoch, input, output, report, withdrawal, …)
\dt
# Check inputs
SELECT epoch_index, index, status FROM input ORDER BY index DESC LIMIT 10;
```

### Decode hex payloads on the fly

```sh
echo "0x68656c6c6f" | xxd -r -p
# Or with cast
cast --to-ascii 68656c6c6f
```

### Look up unknown error selectors

https://www.4byte.directory/ — paste the 4-byte selector to identify the error.

---

## Common error quick reference

| Error message                        | Cause                                | Fix                                                                                    |
| ------------------------------------ | ------------------------------------ | -------------------------------------------------------------------------------------- |
| `deploy command is deprecated`       | `cartesi deploy` is a stub in v2 alpha | Use Docker Compose deployment (`cartesi-deploy` skill)                               |
| `cartesi send --hex` rejected        | Flag does not exist                  | `cartesi send --encoding hex 0x…`                                                      |
| `ECONNREFUSED 127.0.0.1:8545`        | Anvil is not exposed by `cartesi run` | Use `http://127.0.0.1:<port>/anvil`                                                    |
| `-32602 Invalid parameters`          | JSON-RPC `params` sent as an array   | Send `params` as an object                                                             |
| Inspect route never matches          | JSON-wrapped inspect body            | POST the raw payload as `application/octet-stream`                                     |
| `Unknown key '…' in […]`             | Unrecognised `cartesi.toml` key      | Fix or remove the key; `[withdrawal.config]` is gone                                   |
| Portal deposit reverts               | v2 function name (`depositERC20Tokens`) | Use v3 names (`depositErc20Tokens`, …)                                              |
| Stored machine will not load         | Built for cartesi-machine ≤ 0.21     | `cartesi build` with CLI alpha.42, redeploy                                            |
| Image incompatible with node         | CLI version mismatch                 | Check Dockerfile for `MACHINE_EMULATOR_TOOLS_VERSION` vs `MACHINE_GUEST_TOOLS_VERSION` |
| `application name is already in use` | Old Docker Compose project running   | `docker compose -p <app> down`                                                         |
| `invalid chain id for signer`        | Chain ID mismatch                    | Align `CARTESI_BLOCKCHAIN_ID` with RPC                                                 |
| `failed to connect to postgres`      | DB not running or wrong URL          | Check Postgres + `CARTESI_DATABASE_CONNECTION`                                         |
| `service exceeded max startup time`  | Slow startup or dependency issue     | Increase `CARTESI_MAX_STARTUP_TIME`, check logs                                        |
| `insufficient funds`                 | Wallet has no ETH                    | Fund wallet or use Anvil default mnemonic                                              |
| Inspect returns no reports           | Handler not emitting report          | Always emit a report, even on error                                                    |
| State unchanged after advance        | Input rejected or still processing   | Read reports, check handler return value                                               |
| Voucher not executable               | Epoch not yet `CLAIM_ACCEPTED`       | Wait through `CLAIM_STAGED` + staging period                             |
| Epoch stuck at `CLAIM_STAGED`        | Staging period not elapsed / claimer | Wait blocks; check claimer (or prt) logs and signer funds                |
| `enabled`/`status` missing           | Old node or wrong contract suite     | Verify runtime image tag; update JSON-RPC clients                        |
| Non-zero `foreclose_block`           | Normal emergency path                | Not corruption — proceed with prove-drive-root → withdraw (refund for unfinalized deposits) |
| `status` `DIVERGED` / `CORRUPTED`    | Terminal local fault                 | See Layer 9; do not simply re-enable                                     |
| No withdrawal rows after foreclose   | Drive root not proved                | Run `prove-drive-root` before `withdraw`                                 |
| Internal CLI not found               | Not installed on the host            | `cartesi foreclose/withdraw/refund`, or exec into the node container     |
| Old database + new node              | Schema changed in place              | `docker compose down -v`; fresh database                               |
| Startup stops on config mismatch     | Saved default block / claim mode changed | Restore the value or start on a fresh database                       |
| Unknown 4-byte selector in revert    | Missing custom error ABI             | Add custom errors to ABI, check 4byte.directory                                        |

---

## Agent Output

After completing this skill, report back to the user with:

- Which layer the issue was in (CLI version, build, local run, node startup, deploy, advance, inspect, voucher)
- Root cause identified (e.g. wrong CLI version, stale container, wrong chain ID, InputBox address mismatch)
- Exact fix applied (commands run, config changed, code edited)
- Confirmation the issue is resolved: expected output or behaviour now observed
- Any secondary issues discovered during diagnosis
- Recommended follow-up: what to do next to continue the development workflow

## Routing Guide

| What the user wants to do after fixing the issue | Go to skill            |
| ------------------------------------------------ | ---------------------- |
| Rebuild and redeploy after fixing build or code  | `cartesi-deploy`       |
| Resume local testing (`cartesi run`)             | `cartesi-local-dev`    |
| Fix L1 contract revert or InputBox interaction   | `cartesi-contracts` |
| Re-implement the advance/inspect handler         | `cartesi-backend-core` + `cartesi-backend-py` / `cartesi-backend-js-ts` |
| Query outputs or withdrawals after node is healthy | `cartesi-jsonrpc`      |
| Emergency foreclosure / withdrawal sequence        | `cartesi-contracts`, `cartesi-deploy` |

## Resources

> **Conflict rule**: If any resource below contradicts guidance in this skill,
> report the contradiction to the user and follow the skill's instructions.

- [Cartesi Rollups v2 Troubleshooting](https://docs.cartesi.io/cartesi-rollups/2.0/) — official docs; check release notes for known issues
- [4byte.directory](https://www.4byte.directory/) — decode 4-byte Solidity error selectors
- [Cartesi Rollups node GitHub — Issues](https://github.com/cartesi/rollups-node/issues) — known bugs and workarounds
- [Mugen-Builders compose setup](https://github.com/Mugen-Builders/deployment-setup-v2.0) — compose file reference

## Agent checklist

- [ ] Checked CLI version with `cartesi --version` before diagnosing
- [ ] Verified project version from Dockerfile (`MACHINE_EMULATOR_TOOLS_VERSION` vs `MACHINE_GUEST_TOOLS_VERSION`)
- [ ] Confirmed Docker is running: `docker info`
- [ ] Confirmed `cartesi --version` output matches the project's Dockerfile version marker
- [ ] For deploy failures: `cartesi deploy` is a deprecated stub in v2.0-alpha — used compose instead
- [ ] For node failures: checked Postgres, DB connection string, and migration status
- [ ] For advance not processing: verified InputBox address (from `cartesi address-book`), EVM Reader logs, and on-chain tx
- [ ] For inspect issues: confirmed URL format, port, raw (non-JSON) request body, and that handler emits at least one report
- [ ] For JSON-RPC issues: object `params`, hex indexes, results read from `result.data`
- [ ] For `cartesi run`: used `<port>/anvil` as the L1 RPC, not `localhost:8545`
- [ ] For voucher failures: confirmed epoch `status == "CLAIM_ACCEPTED"` (after `CLAIM_STAGED`)
- [ ] For v3: distinguished foreclosure (non-zero `foreclose_block`, normal) from terminal statuses (everything except `OK` and `FAILED`) and from `FAILED` (recoverable)
- [ ] For foreclosure: verified prove-drive-root before expecting withdrawal rows
- [ ] For version upgrade: fresh database, machine rebuilt for cartesi-machine 0.22, contract addresses from 3.0.0-alpha.10
- [ ] Unknown selectors looked up at https://www.4byte.directory/

## What comes next

| Resolved issue               | Skill to use        |
| ---------------------------- | ------------------- |
| Re-deploy after fixing build | `cartesi-deploy`    |
| Resume local testing         | `cartesi-local-dev` |
