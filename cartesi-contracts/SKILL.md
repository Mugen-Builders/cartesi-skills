---
name: cartesi-contracts
version: 0.4.0
description: >-
  Wire L1 smart contracts into a Cartesi Rollups application via the InputBox
  contract. Covers contracts v3 lifecycle (claim staging, withdrawal config,
  guardian foreclosure, emergency accounts-drive withdrawal), InputBox ingress,
  refunds, PRT sentries, portals, voucher execution, and Foundry deployment. Use when the user needs
  on-chain inputs, voucher execution, withdrawal config at deploy time, or
  emergency withdrawal flows. Triggers on: "L1 contract", "InputBox", "addInput",
  "on-chain input", "oracle", "governance", "bridge", "voucher execution",
  "validate output", "Solidity contract", "Foundry", "on-chain ingress",
  "portal", "foreclose", "withdrawal config", "guardian", "emergency withdrawal",
  "accounts-drive", "claim staging", "CLAIM_STAGED".
---

## Skill Version

| Skill               | Version | Cartesi Rollups target | Contract suite              | Last updated |
| ------------------- | ------- | ---------------------- | --------------------------- | ------------ |
| `cartesi-contracts` | `0.4.0` | v2.0-alpha / contracts v3 | `cartesi-rollups` 3.0.0-alpha.10, Dave (PRT) 3.0.0-alpha.6 | Nov 2026     |

> **Contracts v3** is the current target suite (`rollups-contracts 3.0.0-alpha.10`, `dave 3.0.0-alpha.6`), used by rollups-node `v2.0.0-alpha.14` and Cartesi CLI `2.0.0-alpha.42`. Addresses differ from both v2.2.0 **and** 3.0.0-alpha.6 — resolve them from the checked-in manifest or the release deployment artifacts. For local devnet (`cartesi run`), use `cartesi address-book`. Mixing old contracts, old factory addresses, or an older node's database with this node binary is the highest-risk deployment mistake.

# Cartesi Rollups — L1 Contract Integration

## Goal

Wire Solidity smart contracts into a Cartesi dApp so that inputs can
originate from on-chain events, and so that application outputs (vouchers)
can be executed back on-chain. This skill is the **source of truth** for
contracts v3 lifecycle semantics: claim staging, `withdrawal_config`, guardian
foreclosure, and emergency accounts-drive withdrawal. It also covers InputBox
ingress, portals, access control, Foundry deployment, and on-chain voucher
execution. It does not cover frontend integration (see `cartesi-frontend`) or
compose node setup (see `cartesi-deploy`).

## Contracts v3 — lifecycle mental model

Contracts v3 is not only a binding bump. Three areas matter for L1 integration:

### Application health (`enabled` + `status`)

| Field | Meaning |
| ----- | ------- |
| `enabled` | Operator intent — whether the node should process the app |
| `status` | Node-side health: `OK`, `FAILED`, `DIVERGED`, `CORRUPTED`, or a machine outcome (`GUEST_EXCEPTION`, `MACHINE_HALTED`, `MCYCLE_OVERFLOW`, `UNEXPECTED_YIELD`, `INVALID_OUTPUTS_ROOT`) |
| `foreclose_block` | `0x0` until the on-chain foreclosure is observed |

**Key distinction:** foreclosure is a **normal emergency path**, not corruption,
and it is **not a status value**. A healthy foreclosed app is typically
`enabled=true`, `status=OK`, `foreclose_block > 0` (on-chain:
`isForeclosed() == true`). `DIVERGED` (the node's claim disagrees with the
chain) and `CORRUPTED` (local state missing or inconsistent) are integrity
faults. Every status except `OK` and `FAILED` is terminal for that application
on the node. The earlier `INOPERABLE` and `FORECLOSED` status values no
longer exist.

The old single `state` field (`ENABLED` / `DISABLED` / `INOPERABLE`) is replaced
by this split. JSON-RPC and CLI reads expose `enabled` and `status` — see
`cartesi-jsonrpc`.

### Claim finality

All consensus types stage claims before acceptance:

```
CLAIM_COMPUTED → CLAIM_SUBMITTED → CLAIM_STAGED → (wait claimStagingPeriod) → CLAIM_ACCEPTED
```

- **Authority:** submit and stage happen in the **same transaction**; `acceptClaim` is a separate tx after the staging period elapses.
- **Quorum:** the node watches events until majority voting stages one claim, then `acceptClaim` later.
- **PRT (Dave tournaments, 3.0.0-alpha.6):** no `CLAIM_SUBMITTED`; the node posts a commitment to the epoch's tournament and, once the tournament settles, the result is staged: `CLAIM_COMPUTED` → `CLAIM_STAGED` → `CLAIM_ACCEPTED`. The application can appoint **sentries** (and a **sentry manager** that can rotate them): a claim the sentries agree with is accepted without waiting out the claim staging period. With `claimStagingPeriod = 0` a staged claim is accepted at once and sentries have no effect.

Vouchers remain executable only after `CLAIM_ACCEPTED`. Do not assume
`CLAIM_SUBMITTED` or `CLAIM_STAGED` is sufficient.

Foreclosure before acceptance terminalizes pre-foreclosure claim work as
`CLAIM_FORECLOSED`.

### Withdrawal config and emergency exit

Each v3 application carries a **`withdrawal_config`** set at deploy time
(`--withdrawal-config` or `--withdrawal-config-file` via `cartesi-rollups-cli`).
It defines the **guardian** (can call `foreclose()`) and the accounts-drive
layout required for post-foreclosure recovery.

After foreclosure, normal claim submission stops. Recovery is permissionless
(except the guardian-only foreclose step):

1. **Guardian** calls `foreclose()` — `isForeclosed()` becomes true and the node records `foreclose_block`.
2. **Anyone** calls `proveAccountsDriveMerkleRoot(root, proof)` — once per app.
3. **Anyone** calls `withdraw(account, accountProof)` — gas payer can differ from recipient.

Deposits that were **not finalized** before the foreclosure (their epoch was
never accepted) are returned through a fourth path:

4. **Anyone** calls `issueRefund(inputIndex, input)` — once per deposit input.

Proof files come from `cartesi-rollups-machine-tool` (`replay` → `prove
accounts-drive`; flags in `cartesi-deploy` Step 8b). Operator CLI wraps the
steps:

```sh
cartesi-rollups-cli foreclose <app>
cartesi-rollups-cli prove-drive-root <app> --proof-file drive-root-proof.json
cartesi-rollups-cli withdraw <app> --proof-file account-proof.json
cartesi-rollups-cli refund <app> <input-index> --input-file input.hex
cartesi-rollups-cli read withdrawals <app>
```

`refund` on the node CLI requires `--input-file`: a file with the 0x-prefixed
`InputAdded.input` bytes of that deposit (read them from the InputBox event).

On a local `cartesi run` devnet the host CLI does the same without proof
files: `cartesi foreclose`, `cartesi withdraw --account <address>`,
`cartesi refund <input-index>`.

This is **not** the same as in-app voucher withdrawals emitted by the backend
handler — those follow the normal notice/voucher path and execute via
`cartesi-rollups-cli execute`. Emergency withdrawal is an L1 recovery path
after foreclosure; see `cartesi-backend-core` for the distinction.

Apps that may need emergency withdrawal must use an **accounts-drive layout**
compatible with the machine tool (for example the `erc20-withdrawal-dapp`
template). Confirm `withdrawal_config` is valid **before** any on-chain deploy
tx — partial or invalid config must be rejected by the CLI.

`withdrawal_config` fields: `guardian`, `log2_leaves_per_account`,
`log2_max_num_of_accounts`, `accounts_drive_start_index`,
`withdrawal_output_builder`.

With Cartesi CLI alpha.42 the layout is **derived from the built machine**, not
written by hand. In `cartesi.toml`, mark the raw drive that holds the accounts:

```toml
[drives.accounts]
builder = "empty"
format = "raw"
size = "4Mi"
accounts_drive = true          # this drive is the accounts drive
# accounts_drive_size = "1Mi"  # optional: use only the start of a larger drive

[withdrawal]                   # optional overrides
# guardian = "0x..."                   # defaults to devnet account 1
# withdrawal_output_builder = "0x..."  # defaults to the devnet builder; a fork MUST set it
# account_size = 32                    # bytes per account, power of two >= 32
```

`cartesi build` prints the accounts drive and the resulting configuration.
`[withdrawal.config]` is no longer supported. An accounts drive held in an
nvram deploys but cannot be withdrawn from yet — use a raw flash drive. Check
the drive keys against the template you are using; unknown keys in
`cartesi.toml` fail the build.

## Core mental model

```
External protocol / user
        │
        ▼
┌──────────────────┐      addInput(app, payload)      ┌─────────────────────┐
│  Your L1 Contract│ ──────────────────────────────▶  │  InputBox Contract  │
└──────────────────┘                                   └──────────┬──────────┘
                                                                  │ InputAdded event
                                                                  ▼
                                                       ┌─────────────────────┐
                                                       │  Cartesi Rollups    │
                                                       │  Node (EVM Reader)  │
                                                       └──────────┬──────────┘
                                                                  │ forwards to
                                                                  ▼
                                                       ┌─────────────────────┐
                                                       │  Cartesi Machine    │
                                                       │  (advance handler)  │
                                                       └─────────────────────┘
```

The **InputBox** is the single canonical ingress point from L1 into Cartesi.
Any contract or wallet can call `InputBox.addInput(application, payload)` to
submit an advance input.

## Resolving contract addresses

> **Never hardcode contract addresses.** Cartesi contracts are redeployed with each protocol version and differ by chain. Always resolve addresses at task time using one of the methods below.

### For local devnet (`cartesi run`)

```sh
cartesi address-book
```

This prints every deployed address for the running Anvil network — InputBox, all portals, AuthorityFactory, ApplicationFactory, your app contract, and the auto-deployed test tokens. Use this as the single source of truth when working locally.

### For testnets and production — contracts v3 (3.0.0-alpha.10)

**Primary source:** [`cartesi-rollups-3.0.0-alpha.10.json`](cartesi-rollups-3.0.0-alpha.10.json)
in this skill directory, taken from the `cartesi/rollups-contracts`
`v3.0.0-alpha.10` deployment artifacts (`deployments/<chain-id>/<Contract>.txt`).
The deployment is deterministic (CREATE2), and the addresses below were
verified for:

| Chain ID | Network |
| -------- | ------- |
| `11155111` | Ethereum Sepolia |
| `31337` | Local devnet (`cartesi run`, rollups-node devnet) |

`InputBox` is also confirmed at the same address on Ethereum Mainnet (`1`).
For any other chain, read `deployments/<chain-id>/` in the release before
using an address — do not assume it carries over, and do not use Cannon
entries for 3.0.0-alpha.6.

Confirm the version by checking the `cartesi/rollups-runtime` image tag in
`compose.local.yaml`. The node, contract suite, and DB schema must match.

**v3 infrastructure addresses (3.0.0-alpha.10):**

| Contract | Address |
| -------- | ------- |
| `InputBox` | `0xEbE9f4Dfc04ae10bBeE663859c3dc5A23f94eA3C` |
| `AuthorityFactory` | `0xB4d29c86e36385b5321a453C34D288AEB0ad11f9` |
| `QuorumFactory` | `0x0754D5Eb680c71bf469B39e48C5b64AB0813fdb9` |
| `ApplicationFactory` | `0x35Cd91f13141Bb6A6FC69E1eeDD241bbA1Ddd45F` |
| `SelfHostedApplicationFactory` | `0x9e6866A965dC5f99f95EF6B0d8399dad18eEf98b` |
| `EtherPortal` | `0x035b11Be55656c6cfC822D1CaE568C1Af2e497b0` |
| `Erc20Portal` | `0x3332DE61a8BB9aC84893b2f552Fe81C9a6dC5419` |
| `Erc721Portal` | `0x397c352d18DFf47CC8a6143403142cf7afd5Ff7E` |
| `Erc1155SinglePortal` | `0x585F56351A66f131E176a345662215C772f80451` |
| `Erc1155BatchPortal` | `0xee33550a22e3Cf6Cc265524dC9bcfD99D2307EBe` |
| `SafeErc20Transfer` | `0xBB7c5367B74544542F5cE553Ed0DF4B3efAfEEdD` |
| `RefundOutputBuilder` | `0xff50b237b3e61cD810Bd1236644a887ec78762CF` |
| `UsdWithdrawalOutputBuilderFactory` | `0x60b8b3c0AecEb193755038fa9372FfE910E63e39` |

**PRT (Dave 3.0.0-alpha.6) — per chain:**

| Chain | `DaveAppFactory` |
| ----- | ---------------- |
| Ethereum Mainnet (`1`) | `0x37238515E8161486C53741abe615EF1452309128` |
| Ethereum Sepolia (`11155111`) | `0xBD2a661328595113669a6b4c235126eec8c06A15` |
| Local devnet (`31337`) | `0xF8f6dFA3A3e40C10926839FFd8a221C87C8b2ccD` |

Other PRT contracts and other chains: the `deployment-addresses` asset of the
`cartesi/dave` `v3.0.0-alpha.6` release.

> Artifact names are now spelled `Erc20Portal`, `Erc721Portal`,
> `Erc1155SinglePortal`, `Erc1155BatchPortal`, `SafeErc20Transfer` (previously
> `ERC20Portal`, …). `cartesi address-book` uses the same portal names but does
> **not** list `SafeErc20Transfer` or `QuorumFactory`, and names the test tokens
> `TestToken` (`TestFungibleToken`) and `TestNFT` (`TestNonFungibleToken`).

| Contract | Role |
| -------- | ---- |
| `InputBox` | Single ingress point — all inputs enter Cartesi through here |
| `EtherPortal` | Bridge native ETH into the Cartesi Machine |
| `Erc20Portal` | Bridge ERC-20 tokens into the Cartesi Machine |
| `Erc721Portal` | Bridge ERC-721 NFTs into the Cartesi Machine |
| `Erc1155SinglePortal` | Bridge a single ERC-1155 token |
| `Erc1155BatchPortal` | Bridge a batch of ERC-1155 tokens |
| `AuthorityFactory` | Deploy Authority (single-validator) consensus |
| `QuorumFactory` | Deploy Quorum (multi-validator) consensus |
| `ApplicationFactory` | Deploy Cartesi Application contracts |
| `SelfHostedApplicationFactory` | Deploy application + authority in one tx |
| `DaveAppFactory` | Deploy PRT (Dave tournament) applications |
| `SafeErc20Transfer` | Helper for ERC-20 transfers via delegated call vouchers |
| `RefundOutputBuilder` | Builds the refund output for deposits not finalized before foreclosure |
| `UsdWithdrawalOutputBuilderFactory` | Deploys a withdrawal output builder for a USD-like ERC-20 (used in `withdrawal_config`) |

### Legacy — cartesi-rollups v2.2.0

Only use v2.2.0 addresses when the running node explicitly targets v2.2.0.
Cannon: `https://usecannon.com/packages/cartesi-rollups/2.2.0/<chain-id>-main/deployment/contracts`

| Contract                       | Address (v2.2.0 example)                     |
| ------------------------------ | -------------------------------------------- |
| `InputBox`                     | `0x1b51e2992A2755Ba4D6F7094032DF91991a0Cfac` |
| `AuthorityFactory`             | `0x5E96408CFE423b01dADeD3bc867E6013135990cc` |
| `QuorumFactory`                | `0x1C91Ba8aa5648cdAC77E97eaC781447c646EF239` |
| `ApplicationFactory`           | `0x26E758238CB6eC5aB70ce0dd52aF2d7b82e1972E` |
| `SelfHostedApplicationFactory` | `0x010D3CbB4223F5bCc7b7B03cEE59f3aAea8eDb8A` |

Portals and `SafeERC20Transfer` for v2.2.0 are in the Cannon registry — do not
mix v2 portal addresses with a v3 node.

### Superseded — cartesi-rollups 3.0.0-alpha.6

Addresses such as `InputBox` `0x346B3df038FE9f8380071eC6514D5a83aD143939`
belong to 3.0.0-alpha.6 (rollups-node alpha.12). They are **not** valid for
node alpha.14.

## Step 1 — IInputBox interface

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

interface IInputBox {
    function addInput(
        address appContract,
        bytes calldata payload
    ) external returns (bytes32);
}
```

## Step 2 — Minimal L1 contract pattern

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "./IInputBox.sol";

contract MyCartesiFeeder is Ownable {
    IInputBox public immutable inputBox;
    address public immutable application;

    event InputSent(bytes payload, bytes32 inputHash);

    constructor(address _inputBox, address _application) Ownable(msg.sender) {
        inputBox = IInputBox(_inputBox);
        application = _application;
    }

    /// @notice Send a payload to the Cartesi application via InputBox.
    /// @dev Only privileged callers should invoke this for sensitive feeds.
    function sendInput(bytes calldata payload) external onlyOwner returns (bytes32) {
        bytes32 inputHash = inputBox.addInput(application, payload);
        emit InputSent(payload, inputHash);
        return inputHash;
    }
}
```

Key hardening rules:

- **Access control**: Use `onlyOwner`, roles, or an allowlist on any function
  that sends sensitive data. Document who may call it.
- **Events**: Always emit an event with the payload metadata for off-chain
  indexers and debugging.
- **Immutability**: The `application` address is typically fixed at deploy.
  If the app address changes (e.g. re-deployment), you must redeploy or
  upgrade the L1 contract — plan migrations explicitly.

## Step 3 — Payload encoding

The payload your L1 contract sends must match what the Cartesi backend
decodes. Use consistent encoding across both layers.

### ABI encoding (Solidity → backend)

Solidity side:

```solidity
bytes memory payload = abi.encode(
    msg.sender,      // address
    amount,          // uint256
    tokenAddress     // address
);
inputBox.addInput(application, payload);
```

Backend side (JavaScript with viem):

```js
import { decodeAbiParameters, parseAbiParameters } from "viem";

const [sender, amount, token] = decodeAbiParameters(
  parseAbiParameters("address sender, uint256 amount, address token"),
  payload,
);
```

### UTF-8 / JSON encoding (simpler, less gas-efficient)

Solidity side:

```solidity
bytes memory payload = bytes('{"action":"oracle_update","value":42}');
inputBox.addInput(application, payload);
```

Backend side:

```js
const input = JSON.parse(hexToString(data.payload));
```

Choose ABI encoding for structured data from contracts; JSON encoding for
human-readable or prototype payloads.

## Step 4 — On-chain oracle / feed pattern

For contracts that push external data into Cartesi on a schedule or trigger:

```solidity
contract PriceFeedFeeder is Ownable {
    IInputBox public immutable inputBox;
    address public immutable application;
    AggregatorV3Interface public immutable priceFeed;

    constructor(
        address _inputBox,
        address _application,
        address _priceFeed
    ) Ownable(msg.sender) {
        inputBox = IInputBox(_inputBox);
        application = _application;
        priceFeed = AggregatorV3Interface(_priceFeed);
    }

    /// @notice Push latest price into the Cartesi app. Only owner may call.
    function pushLatestPrice() external onlyOwner {
        (, int256 price, , uint256 updatedAt, ) = priceFeed.latestRoundData();
        bytes memory payload = abi.encode(price, updatedAt);
        inputBox.addInput(application, payload);
    }
}
```

## Step 5 — Asset portals (ERC-20, Ether, ERC-721, ERC-1155)

Cartesi provides official portal contracts for bridging assets into the
Cartesi Machine. Use these instead of writing custom deposit logic.

| Asset    | Portal Contract       | Function                                                         |
| -------- | --------------------- | ---------------------------------------------------------------- |
| Ether    | `EtherPortal`         | `depositEther(address appContract, bytes execLayerData)` (payable) |
| ERC-20   | `Erc20Portal`         | Approve first, then `depositErc20Tokens(token, appContract, value, execLayerData)` |
| ERC-721  | `Erc721Portal`        | `setApprovalForAll` first, then `depositErc721Token(token, appContract, tokenId, baseLayerData, execLayerData)` |
| ERC-1155 | `Erc1155SinglePortal` | `setApprovalForAll` first, then `depositSingleErc1155Token(token, appContract, tokenId, value, baseLayerData, execLayerData)` |
| ERC-1155 batch | `Erc1155BatchPortal` | `setApprovalForAll` first, then `depositBatchErc1155Token(token, appContract, tokenIds, values, baseLayerData, execLayerData)` |

> **Function names changed in contracts v3**: `depositErc20Tokens`,
> `depositErc721Token`, `depositSingleErc1155Token`, `depositBatchErc1155Token`
> (lower-case `rc`). The v2 names (`depositERC20Tokens`, …) have different
> selectors and revert against v3 portals. Update `cast` calls, ABIs and
> frontend code. Deposit **payload layouts** (below) are unchanged.

### Local devnet portal addresses

When using `cartesi run`, all portal addresses are auto-deployed on Anvil. Resolve them with:

```sh
cartesi address-book
```

This lists `Erc20Portal`, `Erc721Portal`, `Erc1155SinglePortal`, `Erc1155BatchPortal`, `EtherPortal`, and all other deployed contracts. `cartesi address-book Erc20Portal` prints one address (the name is matched case-insensitively); `--json` prints them all. Do not use hardcoded addresses — they can change between CLI versions.

With `cartesi run` the L1 RPC is `http://127.0.0.1:<port>/anvil` (default port
`6751`), not `localhost:8545`, and transactions should be sent from a devnet
account other than account 0 (the node's claim signer). See `cartesi-local-dev`
for `RPC_URL`, the default sender and the test-token mint functions.

### Depositing ERC-20 via portal (cast commands)

```sh
# 1. Approve the portal to spend tokens
cast send <ERC20_TOKEN_ADDRESS> \
  "approve(address,uint256)" \
  <ERC20_PORTAL_ADDRESS> <AMOUNT_WEI> \
  --rpc-url <RPC_URL> --private-key <PRIVATE_KEY>

# 2. Deposit tokens into the Cartesi application
cast send <ERC20_PORTAL_ADDRESS> \
  "depositErc20Tokens(address,address,uint256,bytes)" \
  <ERC20_TOKEN_ADDRESS> <APP_ADDRESS> <AMOUNT_WEI> 0x \
  --rpc-url <RPC_URL> --private-key <PRIVATE_KEY>
```

### Depositing ERC-721 via portal (cast commands)

```sh
# 1. Grant approval to portal
cast send <NFT_ADDRESS> \
  "setApprovalForAll(address,bool)" \
  <ERC721_PORTAL_ADDRESS> true \
  --rpc-url <RPC_URL> --private-key <PRIVATE_KEY>

# 2. Deposit the NFT (token ID, baseLayerData, execLayerData)
cast send <ERC721_PORTAL_ADDRESS> \
  "depositErc721Token(address,address,uint256,bytes,bytes)" \
  <NFT_ADDRESS> <APP_ADDRESS> <TOKEN_ID> 0x 0x \
  --rpc-url <RPC_URL> --private-key <PRIVATE_KEY>
```

### Depositing ERC-1155 via portal (cast commands)

```sh
# 1. Grant approval
cast send <ERC1155_ADDRESS> \
  "setApprovalForAll(address,bool)" \
  <ERC1155_SINGLE_PORTAL_ADDRESS> true \
  --rpc-url <RPC_URL> --private-key <PRIVATE_KEY>

# 2. Deposit single token (tokenId, amount, baseLayerData, execLayerData)
cast send <ERC1155_SINGLE_PORTAL_ADDRESS> \
  "depositSingleErc1155Token(address,address,uint256,uint256,bytes,bytes)" \
  <ERC1155_ADDRESS> <APP_ADDRESS> <TOKEN_ID> <AMOUNT> 0x 0x \
  --rpc-url <RPC_URL> --private-key <PRIVATE_KEY>
```

### Handling portal deposits in your backend

Portal interactions arrive as advance inputs where the sender
(`metadata.msg_sender` in the HTTP API, `request.msgSender` in `@cartesi/rollup`)
equals the portal contract address. Your backend **must** check the sender to determine
how to decode the payload.

**ERC-20 deposit payload format (bytes layout):**

```
bytes  0–19:  token address (20 bytes)
bytes 20–39:  depositor address (20 bytes)
bytes 40–71:  amount (uint256, 32 bytes, big-endian)
```

JavaScript decode example. It is written for the **rollup HTTP server** model
(`data.metadata.msg_sender`, hex-string `data.payload`), which Python, Go, Rust
and older JS projects use. Projects scaffolded from the current JS/TS templates
use `@cartesi/rollup` instead: there the sender is `request.msgSender` and
`request.payload` is a `Buffer` — see `cartesi-backend-js-ts` for that form.

```js
import { ethers } from "ethers";
import { getAddress } from "viem";

// Resolve portal address from environment — never hardcode.
// Set CARTESI_PORTAL_ERC20 from `cartesi address-book` output before starting the app.
const ERC20_PORTAL = (process.env.CARTESI_PORTAL_ERC20 || "").toLowerCase();

function parseERC20Deposit(payload) {
  const token = getAddress(ethers.dataSlice(payload, 0, 20));
  const depositor = getAddress(ethers.dataSlice(payload, 20, 40));
  const amount = BigInt(ethers.dataSlice(payload, 40, 72));
  return { token, depositor, amount };
}

async function handle_advance(data) {
  const sender = data.metadata.msg_sender.toLowerCase();

  if (ERC20_PORTAL && sender === ERC20_PORTAL) {
    const { token, depositor, amount } = parseERC20Deposit(data.payload);
    // credit balance, emit notice, etc.
    return "accept";
  }
  // ... handle other input types
}
```

**ERC-721 deposit payload format:**

```
bytes  0–19:  token address (20 bytes)
bytes 20–39:  depositor address (20 bytes)
bytes 40–71:  token ID (uint256, 32 bytes)
```

**ERC-1155 single deposit payload format:**

```
bytes  0–19:  token address (20 bytes)
bytes 20–39:  depositor address (20 bytes)
bytes 40–71:  token ID (uint256, 32 bytes)
bytes 72–103: amount (uint256, 32 bytes)
```

> **Rule**: Always isolate portal decode logic from your core application logic.
> Keep portal detection in a dedicated dispatcher. Always keep portal code in a
> **separate component** from your core input logic.

## Step 6 — Voucher execution on-chain

After an epoch closes and its claim reaches **`CLAIM_ACCEPTED`**, vouchers can
be executed on-chain. On contracts v3 a claim is normally staged
(**`CLAIM_STAGED`**) and accepted only after the on-chain staging period — or,
on PRT, as soon as all sentries agree — poll epoch status via JSON-RPC or
`cartesi-rollups-cli read epochs <app-name>`.

```sh
# Confirm epoch is accepted (not merely submitted or staged)
cartesi-rollups-cli read epochs <app-name>

# Validate output proof (check it is on-chain verifiable)
cartesi-rollups-cli validate <app-name> <output-index>

# Execute a voucher on-chain (requires funded wallet)
cartesi-rollups-cli execute <app-name> <output-index>
cartesi-rollups-cli execute <app-name> <output-index> --yes
```

Required env vars for execution:

```sh
export CARTESI_DATABASE_CONNECTION=...
export CARTESI_BLOCKCHAIN_HTTP_ENDPOINT=...
export CARTESI_AUTH_MNEMONIC="..."
```

Execute with a key other than the one the running node uses for claims (one
submitting process per signer). Executed and pending counts are available
through `cartesi_getExecutedOutputCount` /
`cartesi_getPendingExecutableOutputCount`, and
`cartesi-rollups-cli contract output <application-address> <output-index>`
reads the execution status straight from the chain.

Vouchers encode a full contract call (`destination` + ABI-encoded payload).
Only Vouchers and Delegated Call Vouchers are executable. Notices are
attestation-only.

> **Not emergency withdrawal:** backend-emitted vouchers executed with
> `execute` are the normal app egress path. Post-foreclosure
> `withdraw(account, accountProof)` is a separate L1 recovery flow — see
> the contracts v3 lifecycle section above and `cartesi-deploy` for operator
> CLI steps.

## Step 7 — Foundry deployment

### Deploy script

```solidity
// script/Deploy.s.sol
pragma solidity ^0.8.20;
import "forge-std/Script.sol";
import "../src/MyCartesiFeeder.sol";

contract DeployScript is Script {
    function run() external {
        vm.startBroadcast();

        // Resolve InputBox address from `cartesi address-book` (local)
        // or from cartesi-rollups-3.0.0-alpha.10.json / the rollups-contracts release artifacts
        address inputBox   = vm.envAddress("INPUT_BOX_ADDRESS");
        address application = vm.envAddress("APP_ADDRESS");

        new MyCartesiFeeder(inputBox, application);

        vm.stopBroadcast();
    }
}
```

### Deploy command

```sh
# Local devnet started by `cartesi run` (L1 RPC is behind /anvil; devnet account 8 key)
forge script script/Deploy.s.sol:DeployScript \
  --rpc-url http://127.0.0.1:6751/anvil \
  --broadcast \
  --private-key 0xdbda1821b80551c9d65939329250298aa3472ba22feea921c0cf5d620ea67b97

# Testnet (Sepolia)
forge script script/Deploy.s.sol:DeployScript \
  --rpc-url $SEPOLIA_RPC_URL \
  --broadcast \
  --private-key $PRIVATE_KEY \
  --verify
```

### Test contract interactions

```sh
# Call sendInput on deployed contract
cast send <feeder-address> "sendInput(bytes)" <hex-payload> \
  --rpc-url http://127.0.0.1:6751/anvil \
  --private-key <key>
```

## Agent Output

After completing this skill, report back to the user with:

- Solidity contract(s) written: filename, contract name, constructor arguments
- Deployed contract address (from Foundry broadcast output)
- InputBox address used (sourced from `cartesi address-book` or the 3.0.0-alpha.10 manifest)
- Access control scheme applied (onlyOwner / roles / allowlist) and who holds the role
- Payload encoding chosen (ABI or UTF-8/JSON) and the corresponding backend decode snippet
- Events emitted and their indexed fields
- Any portal contracts wired (ERC-20/721/1155) and the portal addresses resolved
- Migration plan if the app address is expected to change
- Test interaction result: `cast send` output confirming `InputSent` event

## Routing Guide

| What the user wants to do next                         | Go to skill         |
| ------------------------------------------------------ | ------------------- |
| Deploy the Cartesi app to testnet / self-hosted node   | `cartesi-deploy`    |
| Test L1 contract interactions locally with cartesi run | `cartesi-local-dev` |
| Debug revert errors or unknown selectors               | `cartesi-debug`     |
| Implement backend handler for the new input type       | `cartesi-backend-core` + `cartesi-backend-py` / `cartesi-backend-js-ts` |
| Query emitted vouchers after epoch close               | `cartesi-jsonrpc`   |

## Resources

> **Conflict rule**: If any resource below contradicts guidance in this skill,
> report the contradiction to the user and follow the skill's instructions.

- [Cartesi InputBox contract](https://docs.cartesi.io/cartesi-rollups/2.0/rollups-apis/json-rpc/input-box/) — `addInput` signature and event ABI
- [https://docs.cartesi.io/cartesi-rollups/2.0/development/](https://docs.cartesi.io/cartesi-rollups/2.0/development/) - Development guides
- [https://docs.cartesi.io/cartesi-rollups/2.0/development/send-inputs-and-assets/](https://docs.cartesi.io/cartesi-rollups/2.0/development/send-inputs-and-assets/) - Inputs and inspect
- [Cartesi Portal contracts](https://docs.cartesi.io/cartesi-rollups/2.0/development/asset-handling/) — ERC-20/721/1155 deposit interfaces and payload layouts
- [cartesi/rollups-contracts v3.0.0-alpha.10](https://github.com/cartesi/rollups-contracts/releases/tag/v3.0.0-alpha.10) — contract sources, ABIs and per-chain deployment artifacts
- [cartesi/dave v3.0.0-alpha.6](https://github.com/cartesi/dave/releases/tag/v3.0.0-alpha.6) — PRT contracts and deployment addresses
- [Foundry Book](https://book.getfoundry.sh/) — Forge/Cast/Anvil documentation
- [4byte.directory](https://www.4byte.directory/) — decode unknown 4-byte error selectors from reverts

## Agent checklist

- [ ] IInputBox interface included in contract with correct address
- [ ] Access control on privileged functions (onlyOwner / roles / allowlist)
- [ ] Events emitted with payload metadata for off-chain indexing
- [ ] Application address is immutable — migration plan documented if needed
- [ ] Payload encoding matches backend decoding (ABI or UTF-8 — pick one)
- [ ] Foundry deploy script written and tested on local Anvil first
- [ ] Contract custom errors included in ABI used client-side
- [ ] Portal logic isolated from core input logic (if bridging assets)
- [ ] Portal deposit handler checks `msg_sender` against portal address
- [ ] Portal payload decoded using correct byte offsets for asset type
- [ ] `cartesi address-book` used to resolve portal addresses for local devnet
- [ ] Portal calls use the v3 function names (`depositErc20Tokens`, `depositErc721Token`, `depositSingleErc1155Token`)
- [ ] Testnet/mainnet addresses taken from the 3.0.0-alpha.10 manifest or release artifacts — not 3.0.0-alpha.6, not v2.2.0
- [ ] On v3 deploys: `withdrawal_config` validated before on-chain tx; guardian address matches funded key
- [ ] Epoch reached `CLAIM_ACCEPTED` (after `CLAIM_STAGED`) before voucher execution
- [ ] Voucher execution tested with `cartesi-rollups-cli execute` after epoch acceptance
- [ ] Emergency withdrawal (`foreclose` → prove drive root → `withdraw`, plus `refund` for unfinalized deposits) distinguished from voucher `execute`

## What comes next

| Next task                        | Skill to use     |
| -------------------------------- | ---------------- |
| Deploy to self-hosted node       | `cartesi-deploy` |
| Debug contract or backend errors | `cartesi-debug`  |
