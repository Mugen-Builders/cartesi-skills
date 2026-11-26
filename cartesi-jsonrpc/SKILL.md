---
name: cartesi-jsonrpc
version: 0.3.0
description: >-
  Query a running Cartesi Rollups node via its JSON-RPC 2.0 API (port 10011).
  Use this whenever you need to list or fetch applications, epochs, inputs,
  outputs (notices, vouchers, DELEGATECALL vouchers), reports, withdrawals,
  PRT dispute data (tournaments, commitments, matches, bonds), or node metadata
  programmatically. Covers all cartesi_ prefixed methods of rollups-node
  v2.0.0-alpha.14, the `{ data }` result envelope, batch requests, contracts v3
  application fields (enabled, status, withdrawal_config, foreclosure markers),
  epoch staging states, type definitions, pagination, and TypeScript
  interfaces. Triggers on: "JSON-RPC", "cartesi_listInputs",
  "cartesi_getOutput", "cartesi_listWithdrawals", "cartesi_listReports",
  "query node", "JSON-RPC API", "port 10011", "query outputs", "list epochs",
  "CLAIM_STAGED", "foreclose_block", "withdrawal", "enabled", "status",
  "cartesi_getNodeInfo", "cartesi_listTournaments", "batch request".
---

## Skill Version

| Skill             | Version | Cartesi Rollups target | Node runtime                              | Last updated |
| ----------------- | ------- | ---------------------- | ----------------------------------------- | ------------ |
| `cartesi-jsonrpc` | `0.3.0` | contracts v3           | `rollups-node` **v2.0.0-alpha.14** (`cartesi/rollups-runtime:0.12.0-alpha.45`) | Nov 2026     |

> Method names and response shapes target **rollups-node v2.0.0-alpha.14**. The
> authoritative schema is `internal/jsonrpc/jsonrpc-discover.json` (OpenRPC) in
> the rollups-node repository at that tag. The application model uses `enabled`
> + `status` (not the old single `state` field), and foreclosure is reported by
> `foreclose_block`, not by a status value. If the node runtime differs, call
> `cartesi_getNodeInfo` (or `cartesi_getNodeVersion`) and check the discover
> file of that tag. Lifecycle semantics: `cartesi-contracts`.

# Cartesi Rollups — JSON-RPC API

## Goal

Query a running Cartesi Rollups node using its JSON-RPC 2.0 interface. The
JSON-RPC API is served by the `jsonrpc-api` service on port **10011**. Use it
to fetch applications, epochs, inputs, outputs, reports, withdrawals, and PRT
dispute data — from scripts, frontends, or automated monitoring tools. The API
is read-only.

---

## Connection details

| Setup            | Endpoint                      |
| ---------------- | ----------------------------- |
| `cartesi run`    | `http://localhost:6751/rpc`   |
| Self-hosted node | `http://localhost:10011/rpc`  |

> **`cartesi run`** serves everything behind one port: `/rpc` (JSON-RPC),
> `/inspect` and `/anvil` (the L1 RPC). The port is `6751` unless
> it is taken (the CLI then picks the next free one up to `6761`) or `-p` is
> passed. Read the actual address from the `starting at` line it prints.

---

## Request / response format

All requests follow the JSON-RPC 2.0 specification:

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "cartesi_<methodName>",
  "params": { ... }
}
```

Send `params` as a JSON **object** with named fields. Wrapping that object in an
array (`"params":[{...}]`) is rejected with `-32602`. (Positional arrays in the
method's declared parameter order, such as `"params":["my-dapp"]`, are also
accepted, but named fields are clearer and are what this skill uses.) Methods without
parameters (`cartesi_getChainId`, `cartesi_getNodeVersion`,
`cartesi_getNodeInfo`) accept `{}` or no `params` at all.

All responses return either a `result` or an `error`. Every `result` is an
object with a `data` field; list methods add `pagination` next to it:

**1. List shape (`cartesi_list*` methods)** — `data` is an array:

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "data": [{ "...": "..." }],
    "pagination": {
      "total_count": 42,
      "limit": 10,
      "offset": 0
    }
  }
}
```

**2. Single-object shape (`cartesi_get*` methods that return a struct)** —
`data` is the resource:

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "data": {
      "name": "my-dapp",
      "enabled": true,
      "status": "OK",
      "...": "..."
    }
  }
}
```

**3. Scalar shape (`cartesi_getProcessedInputCount`,
`cartesi_getExecutedOutputCount`, `cartesi_getPendingExecutableOutputCount`,
`cartesi_getLastAcceptedEpochIndex`, `cartesi_getChainId`,
`cartesi_getNodeVersion`)** — `data` is the scalar (hex string, or a plain
string for the version):

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": { "data": "0x2a" }
}
```

> Always read `result.data`. Reading `result` directly (for example
> `parseInt(json.result, 16)` or `json.result.status`) is a bug on this node.

Error response:

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "error": {
    "code": -32000,
    "message": "Server error"
  }
}
```

**All hex values are prefixed with `0x`.** Index and block-number integers in
parameters and responses are hex strings (e.g. `"0x1"` for epoch index 1).
`limit`, `offset` and the `pagination` counters are plain JSON numbers.

### Limits and batch requests

- Request body: at most 1 MB. Response: 10 MB budget per HTTP request; an
  entry that does not fit gets error `-31003`.
- Batches: a non-empty JSON array of up to 100 requests. Entries run
  sequentially and answers come back in request order. A batch outside that
  size gets a single `-32040` error. Notifications are not suppressed: an entry
  without `id` is answered with `id: null`.
- In a batch, the effective `limit` values of all list calls are summed
  (omitted counts as 50); a total above 10000 rejects the whole batch with
  `-31004`. Keep heavy list calls out of large batches.

---

## Common query helper (curl)

```sh
cartesi_rpc() {
  curl -s -X POST "http://localhost:10011/rpc" \
    -H "Content-Type: application/json" \
    -d "$1" | jq .
}

# Example: list all inputs for an app
cartesi_rpc '{
  "jsonrpc":"2.0","id":1,
  "method":"cartesi_listInputs",
  "params":{"application":"my-dapp","limit":10,"offset":0}
}'
```

---

## Pagination

All `cartesi_list*` methods support pagination:

| Parameter | Type   | Default | Description                     |
| --------- | ------ | ------- | ------------------------------- |
| `limit`   | number | 50      | Maximum items per page (min: 1; values above 10000 are capped) |
| `offset`  | number | 0       | Starting index                  |
| `descending` | boolean | false | Sort by index, newest first     |

`cartesi_listEpochs`, `cartesi_listInputs`, `cartesi_listOutputs` and
`cartesi_listReports` also take `from` / `to`: inclusive bounds on the index
(hex), useful for resuming a scan without large offsets.

Response always includes a `pagination` object:

```json
{
  "total_count": 150,
  "limit": 10,
  "offset": 20
}
```

Iterate pages by incrementing `offset` by `limit` until `offset >= total_count`.

---

## Applications

### `cartesi_listApplications`

Returns all applications registered on this node.

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "cartesi_listApplications",
  "params": { "limit": 10, "offset": 0 }
}
```

### `cartesi_getApplication`

Fetch details for a single application by name or hex address.

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "cartesi_getApplication",
  "params": { "application": "my-dapp" }
}
```

Or by address:

```json
{ "application": "0xba3347e79665924033beeb7362629ca7992897d9" }
```

**Contracts v3 application fields** (via `cartesi_getApplication`):

| Field | Type | Notes |
| ----- | ---- | ----- |
| `enabled` | boolean | Operator intent |
| `status` | string | See `ApplicationStatus` below; `OK` when healthy |
| `consensus_type` | string | `AUTHORITY`, `QUORUM` or `PRT` |
| `claim_staging_period` | hex string | Blocks before `acceptClaim` is valid |
| `withdrawal_config` | object | Guardian and accounts-drive layout set at deploy |
| `foreclose_block` / `foreclose_transaction` | hex string | Block and transaction of the on-chain foreclosure; `foreclose_block` is `0x0` until one is observed |
| `accounts_drive_proved_block` / `accounts_drive_proved_transaction` | hex string | Set after drive root proved on L1 |
| `accounts_drive_merkle_root` | hash or null | The proved accounts-drive root; `null` until proved (the discover file says "all-zero", but the node returns `null`) |
| `reason` | string | Human-readable detail when `status` is not `OK` |

**Foreclosure is not a status.** A foreclosed application keeps its `status`
(normally `OK`) and is recognised by a non-zero `foreclose_block`. Foreclosure
is a normal emergency path — not corruption. `DIVERGED` and `CORRUPTED` are the
local integrity faults; every status other than `OK` and `FAILED` is terminal. See `cartesi-contracts` for lifecycle semantics.

---

## Epochs

### `cartesi_listEpochs`

List epochs for an application. Optionally filter by status.

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "cartesi_listEpochs",
  "params": {
    "application": "my-dapp",
    "status": "OPEN",
    "limit": 10,
    "offset": 0
  }
}
```

**`status` filter values**: `OPEN`, `CLOSED`, `INPUTS_PROCESSED`,
`CLAIM_COMPUTED`, `CLAIM_SUBMITTED`, `CLAIM_STAGED`, `CLAIM_ACCEPTED`,
`CLAIM_REJECTED`, `CLAIM_FORECLOSED`. Pass one value or an array of values
(OR semantics).

**Authority / Quorum path:** `CLAIM_SUBMITTED` → `CLAIM_STAGED` →
(wait staging period) → `CLAIM_ACCEPTED`. **PRT** applications also pass
through `CLAIM_STAGED` (the tournament result is staged, then accepted after
the claim staging period, or earlier when the sentries agree) — do not assume
PRT skips it. Foreclosure before acceptance moves the epoch to
`CLAIM_FORECLOSED`.

### `cartesi_getEpoch`

Fetch a specific epoch by index (hex encoded).

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "cartesi_getEpoch",
  "params": { "application": "my-dapp", "epoch_index": "0x0" }
}
```

### `cartesi_getEpochByVirtualIndex`

Fetch an epoch by its contiguous position (`virtual_index`, hex) instead of its
on-chain `index`, which can have gaps.

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "cartesi_getEpochByVirtualIndex",
  "params": { "application": "my-dapp", "virtual_index": "0x0" }
}
```

### `cartesi_getLastAcceptedEpochIndex`

Get the index of the most recently accepted epoch (for voucher readiness checks).

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "cartesi_getLastAcceptedEpochIndex",
  "params": { "application": "my-dapp" }
}
```

---

## Inputs

### `cartesi_listInputs`

List inputs sent to an application. Supports filtering by `epoch_index`,
`sender` and `transaction_hash`.

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "cartesi_listInputs",
  "params": {
    "application": "my-dapp",
    "epoch_index": "0x0",
    "sender": "0x71C7656EC7ab88b098defB751B7401B5f6d8976F",
    "limit": 10,
    "offset": 0
  }
}
```

### `cartesi_getInput`

Fetch a specific input by index (hex encoded). The response includes the decoded
`EvmAdvance` structure with sender, block number, timestamp, and payload.

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "cartesi_getInput",
  "params": { "application": "my-dapp", "input_index": "0x0" }
}
```

### `cartesi_getProcessedInputCount`

Returns the count of inputs the node has fully processed. Useful for polling
until a submitted input is confirmed.

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "cartesi_getProcessedInputCount",
  "params": { "application": "my-dapp" }
}
```

---

## Outputs

Outputs are notices, vouchers, and DELEGATECALL vouchers. All are indexed
globally per application (`output_index`).

### `cartesi_listOutputs`

List outputs with rich filtering options.

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "cartesi_listOutputs",
  "params": {
    "application": "my-dapp",
    "epoch_index": "0x0",
    "input_index": "0x5",
    "output_type": "0x70a08231",
    "voucher_address": "<destination-contract-address>",
    "limit": 10,
    "offset": 0
  }
}
```

Filter parameters:

| Parameter         | Description                                                  |
| ----------------- | ------------------------------------------------------------ |
| `epoch_index`     | Filter by epoch (hex)                                        |
| `input_index`     | Filter by the input that generated this output (hex)         |
| `output_type`     | First 4 bytes of raw data hex — filters by function selector. One selector or a non-empty array (OR semantics) |
| `voucher_address` | Filter vouchers destined for a specific contract             |
| `executed`        | `true`: only outputs with an execution transaction; `false`: only those without. Executions are observed out of index order, so do not use it as a resume cursor |

### `cartesi_getOutput`

Fetch a specific output by global index.

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "cartesi_getOutput",
  "params": { "application": "my-dapp", "output_index": "0x2" }
}
```

### `cartesi_getExecutedOutputCount` / `cartesi_getPendingExecutableOutputCount`

Counters (hex, in `result.data`) of outputs already executed on L1 and of
executable outputs still waiting. Both take `{ "application": "my-dapp" }`.

---

## Reports

Reports are emitted by both advance and inspect handlers. They carry no
on-chain proof — use them for debug data, error messages, and read results.

### `cartesi_listReports`

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "cartesi_listReports",
  "params": {
    "application": "my-dapp",
    "epoch_index": "0x0",
    "input_index": "0x3",
    "limit": 10,
    "offset": 0
  }
}
```

### `cartesi_getReport`

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "cartesi_getReport",
  "params": { "application": "my-dapp", "report_index": "0x0" }
}
```

---

## Withdrawals (contracts v3)

Withdrawal rows are recorded by the EVM Reader after on-chain
`withdraw(account, accountProof)` calls in the post-foreclosure lifecycle.
They appear only after the accounts-drive merkle root has been proved on L1.

> **Not voucher withdrawals:** these RPC methods track **emergency L1
> withdrawals** after foreclosure. Backend-emitted voucher withdrawals are
> outputs — use `cartesi_listOutputs` and execute via `cartesi-rollups-cli
> execute`. See `cartesi-contracts`.

### `cartesi_listWithdrawals`

List withdrawal events for an application. Optionally filter by
`account_index` (hex).

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "cartesi_listWithdrawals",
  "params": {
    "application": "my-dapp",
    "limit": 10,
    "offset": 0
  }
}
```

### `cartesi_getWithdrawal`

Fetch a specific withdrawal by account index (hex encoded).

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "cartesi_getWithdrawal",
  "params": {
    "application": "my-dapp",
    "account_index": "0x0"
  }
}
```

Verify operator CLI `read withdrawals` and JSON-RPC return consistent rows.

---

## PRT dispute data (Dave tournaments)

For applications deployed on a PRT consensus (`consensus_type == "PRT"`), the
node records what it observes of the dispute tournaments. All methods are
read-only and application-scoped.

| Method | Required params (besides `application`) | Optional filters |
| ------ | --------------------------------------- | ---------------- |
| `cartesi_listTournaments` | — | `epoch_index`, `level`, `parent_tournament_address`, `parent_match_id_hash` |
| `cartesi_getTournament` | `address` | |
| `cartesi_listCommitments` | — | `epoch_index`, `tournament_address` |
| `cartesi_getCommitment` | `epoch_index`, `tournament_address`, `commitment` | |
| `cartesi_listMatches` | — | `epoch_index`, `tournament_address` |
| `cartesi_getMatch` | `epoch_index`, `tournament_address`, `id_hash` | |
| `cartesi_listMatchAdvances` | `epoch_index`, `tournament_address`, `id_hash` | |
| `cartesi_getMatchAdvance` | `epoch_index`, `tournament_address`, `id_hash`, `tx_hash`, `log_index` | |
| `cartesi_listBondEvents` | — | `epoch_index`, `tournament_address` |
| `cartesi_getBondEvent` | `tx_hash`, `log_index` | |
| `cartesi_listStandingChanges` | `epoch_index`, `tournament_address` | |
| `cartesi_getStandingChange` | `epoch_index`, `tournament_address`, `tx_hash`, `log_index` | |

All list methods take `limit`, `offset`, `descending`. Field-level schemas
(`Tournament`, `Commitment`, `Match`, `BondEvent`, `StandingChanged`, …) are in
the discover file; do not guess field names — read them from there.

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "cartesi_listTournaments",
  "params": { "application": "my-dapp", "epoch_index": "0x0" }
}
```

---

## Node information

### `cartesi_getNodeInfo`

Returns `chain_id`, `version` and `default_block` in one call, for example
`{"chain_id":"0x7a69","version":"2.0.0-alpha.14","default_block":"LATEST"}`
(the block policy is upper-case here).

```json
{ "jsonrpc": "2.0", "id": 1, "method": "cartesi_getNodeInfo", "params": {} }
```

### `cartesi_getChainId`

Returns the chain ID the node is operating on (hex string).

```json
{ "jsonrpc": "2.0", "id": 1, "method": "cartesi_getChainId", "params": {} }
```

### `cartesi_getNodeVersion`

Returns the node's semantic version string.

```json
{ "jsonrpc": "2.0", "id": 1, "method": "cartesi_getNodeVersion", "params": {} }
```

---

## Type reference

### Basic types

All hex values use these patterns:

| Type               | Pattern                 | Example                                        |
| ------------------ | ----------------------- | ---------------------------------------------- |
| `EthereumAddress`  | `^0x[a-fA-F0-9]{40}$`   | `"0x71C7656EC7ab88b098defB751B7401B5f6d8976F"` |
| `Hash`             | `^0x[a-fA-F0-9]{64}$`   | `"0x1234...abcd"`                              |
| `ByteArray`        | `^0x[a-fA-F0-9]*$`      | `"0x68656c6c6f"`                               |
| `UnsignedInteger`  | `^0x[a-fA-F0-9]{1,16}$` | `"0x1"`                                        |
| `UnsignedInteger256` | `^0x[a-fA-F0-9]{1,64}$` | `EvmAdvance.prev_randao`, `Voucher.value`      |
| `Uint256`          | canonical lowercase hex, no leading zeros | PRT dispute fields            |
| `FunctionSelector` | `^0x[a-fA-F0-9]{8}$`    | `"0xa9059cbb"`                                 |
| `ApplicationName`  | `^[a-z0-9_-]+$`         | `"my-dapp"`                                    |
| `NameOrAddress`    | Either of the above two | `"my-dapp"` or `"0x71C7...976F"`               |
| `Timestamp`        | ISO 8601                | `"2024-01-01T00:00:00Z"`                       |

### Enums

**`ApplicationStatus`**: `OK` | `FAILED` | `DIVERGED` | `CORRUPTED`
| `GUEST_EXCEPTION` | `MACHINE_HALTED` | `MCYCLE_OVERFLOW` | `UNEXPECTED_YIELD`
| `INVALID_OUTPUTS_ROOT`

| Status | Meaning |
| ------ | ------- |
| `OK` | Healthy; processed when `enabled` and not foreclosed |
| `FAILED` | Recoverable node-side failure (for example OOM or a process crash); re-enable after repairing the cause |
| `DIVERGED` | The computed claim disagrees with the chain |
| `CORRUPTED` | Local state missing or inconsistent |
| `GUEST_EXCEPTION`, `MACHINE_HALTED`, `MCYCLE_OVERFLOW`, `UNEXPECTED_YIELD`, `INVALID_OUTPUTS_ROOT` | The application's own execution ended the machine; see `reason` |

Only `OK` and `FAILED` are non-terminal: every other status is final for that
application on this node.

Use with `enabled` (boolean). The old single `ApplicationState` field
(`ENABLED` / `DISABLED` / `INOPERABLE`) and the earlier `INOPERABLE` /
`FORECLOSED` status values no longer exist. Foreclosure is read from
`foreclose_block`.

**`Consensus`**: `AUTHORITY` | `QUORUM` | `PRT`

**`EpochStatus`**: `OPEN` → `CLOSED` → `INPUTS_PROCESSED` → `CLAIM_COMPUTED`
→ `CLAIM_SUBMITTED` → `CLAIM_STAGED` → `CLAIM_ACCEPTED`
| `CLAIM_REJECTED` | `CLAIM_FORECLOSED`

All three consensus types use `CLAIM_STAGED`.

**`InputCompletionStatus`**: `NONE` | `ACCEPTED` | `REJECTED` | `EXCEPTION`
| `MACHINE_HALTED` | `OVERFLOW` | `UNEXPECTED_YIELD` | `INVALID_OUTPUTS_ROOT`

**`SnapshotPolicy`**: `NONE` | `EVERY_INPUT` | `EVERY_EPOCH`

### TypeScript interfaces

```typescript
// Every method returns one of these two envelopes in `result`.
interface Single<T> {
  data: T;
}
interface Page<T> {
  data: T[];
  pagination: Pagination;
}

interface WithdrawalConfig {
  guardian: string; // EthereumAddress
  log2_leaves_per_account: string; // UnsignedInteger
  log2_max_num_of_accounts: string; // UnsignedInteger
  accounts_drive_start_index: string; // UnsignedInteger
  withdrawal_output_builder: string; // EthereumAddress
}

type ApplicationStatus =
  | "OK"
  | "FAILED"
  | "DIVERGED"
  | "CORRUPTED"
  | "GUEST_EXCEPTION"
  | "MACHINE_HALTED"
  | "MCYCLE_OVERFLOW"
  | "UNEXPECTED_YIELD"
  | "INVALID_OUTPUTS_ROOT";

interface Application {
  name: string; // ApplicationName
  iapplication_address: string; // EthereumAddress
  iconsensus_address: string; // EthereumAddress
  iinputbox_address: string; // EthereumAddress
  template_hash: string; // Hash
  epoch_length: string; // UnsignedInteger
  claim_staging_period: string; // UnsignedInteger — contracts v3
  withdrawal_config: WithdrawalConfig; // never null; all-zero fields when not configured
  consensus_type: "AUTHORITY" | "QUORUM" | "PRT";
  enabled: boolean;
  status: ApplicationStatus;
  reason: string | null;
  iinputbox_block: string; // UnsignedInteger
  last_epoch_check_block: string; // UnsignedInteger
  last_input_check_block: string; // UnsignedInteger
  last_output_check_block: string; // UnsignedInteger
  last_tournament_check_block: string; // UnsignedInteger
  last_foreclose_check_block: string; // UnsignedInteger
  last_accounts_drive_proved_check_block: string; // UnsignedInteger
  last_withdrawal_check_block: string; // UnsignedInteger
  processed_inputs: string; // UnsignedInteger
  foreclose_block: string; // UnsignedInteger — 0x0 until foreclosed
  foreclose_transaction: string | null; // Hash
  accounts_drive_proved_block: string; // UnsignedInteger — 0x0 until proved
  accounts_drive_proved_transaction: string | null; // Hash
  accounts_drive_merkle_root: string | null; // Hash — null until proved
  created_at: string; // ISO 8601
  updated_at: string;
  execution_parameters: ExecutionParameters;
}

interface ExecutionParameters {
  snapshot_policy: "NONE" | "EVERY_INPUT" | "EVERY_EPOCH";
  advance_inc_cycles: string; // UnsignedInteger (nanoseconds)
  advance_max_cycles: string;
  inspect_inc_cycles: string;
  inspect_max_cycles: string;
  advance_inc_deadline: string;
  advance_max_deadline: string;
  inspect_inc_deadline: string;
  inspect_max_deadline: string;
  load_deadline: string;
  store_deadline: string;
  fast_deadline: string;
  max_concurrent_inspects: number;
  created_at: string;
  updated_at: string;
}

interface Epoch {
  index: string; // UnsignedInteger
  first_block: string; // UnsignedInteger
  last_block: string; // UnsignedInteger
  input_index_lower_bound: string; // UnsignedInteger
  input_index_upper_bound: string; // UnsignedInteger
  machine_hash: string | null; // Hash
  commitment: string | null; // Hash
  claim_transaction_hash: string | null; // Hash
  tournament_address: string | null; // EthereumAddress — PRT only
  staged_at_block: string | null; // UnsignedInteger
  status: EpochStatus;
  // also: tx_buffer_*, iflags_y_*, htif_tohost_* and commitment_proof
  // (machine proof leaves used by v3 claims)
  virtual_index: string; // UnsignedInteger
  created_at: string;
  updated_at: string;
}

interface Input {
  epoch_index: string; // UnsignedInteger
  index: string; // UnsignedInteger
  block_number: string; // UnsignedInteger
  raw_data: string; // ByteArray
  decoded_data: EvmAdvance | null;
  status: InputCompletionStatus;
  exception_data: string | null; // ByteArray — set when status is EXCEPTION
  machine_hash: string | null; // Hash
  transaction_hash: string; // Hash — L1 transaction that added the input
  log_index: string; // UnsignedInteger
  created_at: string;
  updated_at: string;
}

interface EvmAdvance {
  chain_id: string; // UnsignedInteger
  application_contract: string; // EthereumAddress
  sender: string; // EthereumAddress
  block_number: string; // UnsignedInteger
  block_timestamp: string; // UnsignedInteger (unix)
  prev_randao: string; // ByteArray
  index: string; // UnsignedInteger
  payload: string; // ByteArray — your app data
}

interface Output {
  epoch_index: string; // UnsignedInteger
  input_index: string; // UnsignedInteger
  index: string; // UnsignedInteger (global)
  raw_data: string; // ByteArray
  decoded_data: Notice | Voucher | DelegateCallVoucher | null;
  hash: string | null; // Hash (available after epoch close)
  output_hashes_siblings: string[] | null;
  execution_transaction_hash: string | null;
  created_at: string;
  updated_at: string;
}

interface Notice {
  type: string; // FunctionSelector
  payload: string; // ByteArray — your notice data
}

interface Voucher {
  type: string; // FunctionSelector
  destination: string; // EthereumAddress — contract to call
  value: string; // ETH value in wei (hex)
  payload: string; // ByteArray — ABI-encoded call
}

interface DelegateCallVoucher {
  type: string; // FunctionSelector
  destination: string; // EthereumAddress
  payload: string; // ByteArray — ABI-encoded call
}

interface Report {
  epoch_index: string; // UnsignedInteger
  input_index: string; // UnsignedInteger
  index: string; // UnsignedInteger
  raw_data: string; // ByteArray — your report data
  created_at: string;
  updated_at: string;
}

interface Withdrawal {
  account_index: string; // UnsignedInteger
  account: string; // ByteArray — account bytes as stored in the accounts drive
  output: string; // ByteArray — output built by the WithdrawalOutputBuilder
  block_number: string; // UnsignedInteger
  transaction_hash: string; // Hash
  log_index: string; // UnsignedInteger
  created_at: string;
  updated_at: string;
}

interface Pagination {
  total_count: number;
  limit: number;
  offset: number;
}
```

---

## Common patterns

### Poll until an input is processed

```typescript
async function waitForInputProcessed(
  rpcUrl: string,
  app: string,
  targetCount: number,
  pollIntervalMs = 2000,
): Promise<void> {
  while (true) {
    const res = await fetch(rpcUrl, {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({
        jsonrpc: "2.0",
        id: 1,
        method: "cartesi_getProcessedInputCount",
        params: { application: app },
      }),
    });
    const json = await res.json();
    const count = parseInt(json.result.data, 16);
    if (count >= targetCount) return;
    await new Promise((r) => setTimeout(r, pollIntervalMs));
  }
}
```

### Read and decode all reports for an input

```typescript
async function getReportsForInput(
  rpcUrl: string,
  app: string,
  inputIndex: number,
): Promise<string[]> {
  const res = await fetch(rpcUrl, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({
      jsonrpc: "2.0",
      id: 1,
      method: "cartesi_listReports",
      params: {
        application: app,
        input_index: `0x${inputIndex.toString(16)}`,
        limit: 100,
        offset: 0,
      },
    }),
  });
  const { result } = await res.json();
  return result.data.map((r: Report) =>
    Buffer.from(r.raw_data.replace(/^0x/, ""), "hex").toString("utf8"),
  );
}
```

### Check if epoch is ready for voucher execution

```typescript
async function isEpochAccepted(
  rpcUrl: string,
  app: string,
  epochIndex: number,
): Promise<boolean> {
  const res = await fetch(rpcUrl, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({
      jsonrpc: "2.0",
      id: 1,
      method: "cartesi_getEpoch",
      params: {
        application: app,
        epoch_index: `0x${epochIndex.toString(16)}`,
      },
    }),
  });
  const { result } = await res.json();
  return result.data.status === "CLAIM_ACCEPTED";
}
```

### Poll until epoch reaches CLAIM_STAGED

```typescript
async function isEpochStaged(
  rpcUrl: string,
  app: string,
  epochIndex: number,
): Promise<boolean> {
  const res = await fetch(rpcUrl, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({
      jsonrpc: "2.0",
      id: 1,
      method: "cartesi_getEpoch",
      params: {
        application: app,
        epoch_index: `0x${epochIndex.toString(16)}`,
      },
    }),
  });
  const { result } = await res.json();
  const staged = result.data.status === "CLAIM_STAGED";
  const accepted = result.data.status === "CLAIM_ACCEPTED";
  return staged || accepted;
}
```

### Check application is healthy for processing

```typescript
async function isAppOperational(
  rpcUrl: string,
  app: string,
): Promise<boolean> {
  const res = await fetch(rpcUrl, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({
      jsonrpc: "2.0",
      id: 1,
      method: "cartesi_getApplication",
      params: { application: app },
    }),
  });
  const { result } = await res.json();
  const app = result.data;
  // A foreclosed app keeps status OK but no longer processes inputs.
  const foreclosed = BigInt(app.foreclose_block ?? "0x0") !== 0n;
  return app.enabled === true && app.status === "OK" && !foreclosed;
}
```

### Paginate through all outputs

```typescript
async function getAllOutputs(rpcUrl: string, app: string): Promise<Output[]> {
  const results: Output[] = [];
  let offset = 0;
  const limit = 50;
  while (true) {
    const res = await fetch(rpcUrl, {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({
        jsonrpc: "2.0",
        id: 1,
        method: "cartesi_listOutputs",
        params: { application: app, limit, offset },
      }),
    });
    const { result } = await res.json();
    results.push(...result.data);
    if (offset + limit >= result.pagination.total_count) break;
    offset += limit;
  }
  return results;
}
```

---

## Error handling

Always check both HTTP status and JSON-RPC error field:

```typescript
async function rpcCall(rpcUrl: string, method: string, params: object) {
  const res = await fetch(rpcUrl, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ jsonrpc: "2.0", id: 1, method, params }),
  });

  if (!res.ok) {
    throw new Error(`HTTP error: ${res.status}`);
  }

  const json = await res.json();
  if (json.error) {
    throw new Error(`RPC error ${json.error.code}: ${json.error.message}`);
  }

  return json.result; // { data } or { data, pagination }
}
```

Common error codes:

| Code     | Meaning                                    |
| -------- | ------------------------------------------ |
| `-32700` | Parse error — invalid JSON sent            |
| `-32600` | Invalid request — missing required field   |
| `-32601` | Method not found                           |
| `-32602` | Invalid params — check hex encoding, types, and that `params` is an object |
| `-32603` | Internal error — node-side failure; alarm or back off, do not poll |
| `-32040` | Batch size outside 1–100 entries           |
| `-32070` | Timeout — the request did not finish in the server time limit |
| `-31001` | Resource not found — the application is known but the epoch/input/output is not there (yet); safe to poll for the next index |
| `-31002` | Application not found — a configuration error; retrying will not help |
| `-31003` | Response size limit exceeded — retry alone or with a smaller `limit` |
| `-31004` | Batch list-work budget exceeded (sum of `limit` above 10000) |

---

## Agent Output

After completing this skill, report back to the user with:

- JSON-RPC endpoint URL confirmed reachable (port 10011)
- Chain ID and application name/address confirmed via `cartesi_getChainId` and `cartesi_getApplication`
- Application health confirmed: `enabled == true` and `status == "OK"`, and whether `foreclose_block` is non-zero
- Total processed input count (`cartesi_getProcessedInputCount`)
- Summary of outputs found: number of notices, vouchers, delegated call vouchers, and reports
- Emergency withdrawal rows (`cartesi_listWithdrawals`) if app is foreclosed
- Any vouchers ready for on-chain execution (epoch `CLAIM_ACCEPTED`) — include output index
- Code snippet(s) generated for the user's specific query need (poll, paginate, etc.)
- Any errors encountered (RPC error codes, empty results) and their resolution

## Routing Guide

| What the user wants to do next                       | Go to skill            |
| ---------------------------------------------------- | ---------------------- |
| Execute a voucher on-chain after epoch is accepted   | `cartesi-contracts` |
| Deploy the node to testnet to start producing epochs | `cartesi-deploy`       |
| Debug why outputs are missing or inputs are rejected | `cartesi-debug`        |
| Foreclosure / emergency withdrawal operator flows    | `cartesi-contracts`, `cartesi-deploy` |
| Implement backend logic that emits the outputs       | `cartesi-backend-core` + `cartesi-backend-py` / `cartesi-backend-js-ts` |
| Test and send inputs locally before querying         | `cartesi-local-dev`    |

## Resources

> **Conflict rule**: If any resource below contradicts guidance in this skill,
> report the contradiction to the user and follow the skill's instructions.

- [Cartesi JSON-RPC API reference](https://docs.cartesi.io/cartesi-rollups/2.0/rollups-apis/json-rpc/overview/) — method signatures and parameter types
- [Cartesi Rollups node GitHub](https://github.com/cartesi/rollups-node) — source for JSON-RPC server implementation
- [OpenRPC discover file at v2.0.0-alpha.14](https://github.com/cartesi/rollups-node/blob/v2.0.0-alpha.14/internal/jsonrpc/jsonrpc-discover.json) — authoritative method, parameter and schema list
- [JSON-RPC 2.0 specification](https://www.jsonrpc.org/specification) — base protocol (error codes, request/response format)
- [Cartesi Epoch lifecycle](https://docs.cartesi.io/cartesi-rollups/2.0/) — OPEN → CLAIM_ACCEPTED state transitions

## Agent checklist

- [ ] Confirmed JSON-RPC endpoint is reachable: `cartesi_getNodeInfo` on `http://localhost:10011/rpc` (or `http://localhost:6751/rpc` with `cartesi run`)
- [ ] Read every result from `result.data` (and `result.pagination` for lists)
- [ ] Sent `params` as an object with named fields — never an object wrapped in an array
- [ ] Verified node is on correct chain: `cartesi_getChainId`
- [ ] Application is operational: `cartesi_getApplication` → `enabled == true` and `status == "OK"`
- [ ] Used `cartesi_getProcessedInputCount` to confirm inputs are processed before querying outputs
- [ ] Used pagination for list queries — never assumed all results fit in one page
- [ ] Hex-encoded all index parameters (e.g. `"0x0"` not `0`)
- [ ] Decoded `raw_data` from hex before interpreting report content
- [ ] Checked `status` on Input objects — only ACCEPTED inputs have valid outputs
- [ ] For vouchers: confirmed epoch `status == "CLAIM_ACCEPTED"` (after `CLAIM_STAGED`)
- [ ] For emergency withdrawals: used `cartesi_listWithdrawals` / `cartesi_getWithdrawal`, not output queries

## What comes next

| Next task                  | Skill to use           |
| -------------------------- | ---------------------- |
| Execute vouchers on-chain  | `cartesi-contracts` |
| Deploy to self-hosted node | `cartesi-deploy`       |
| Debug node or query issues | `cartesi-debug`        |
