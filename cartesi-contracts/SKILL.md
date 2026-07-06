---
name: cartesi-contracts
version: 0.3.0
description: >-
  Wire L1 smart contracts into a Cartesi Rollups application via the InputBox
  contract. Covers contracts v3 lifecycle (claim staging, withdrawal config,
  guardian foreclosure, emergency accounts-drive withdrawal), InputBox ingress,
  portals, voucher execution, and Foundry deployment. Use when the user needs
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
| `cartesi-contracts` | `0.3.0` | v2.0-alpha / contracts v3 | `cartesi-rollups` 3.0.0-alpha.6 | Jul 2026     |

> **Contracts v3** is the current target suite (`rollups-contracts 3.0.0-alpha.6`, `dave 3.0.0-alpha.3`). Addresses differ from v2.2.0 — always resolve from Cannon or `compose.local.yaml` image tags. For local devnet (`cartesi run`), use `cartesi address-book`. Mixing old contracts, old factory addresses, or a v2-alpha DB with a v3 node binary is the highest-risk deployment mistake.

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
| `status` | System health: `OK`, `FAILED`, `INOPERABLE`, `FORECLOSED` |

**Key distinction:** `FORECLOSED` is a **normal emergency path**, not corruption.
`INOPERABLE` means local mismatch or corruption. A healthy foreclosed app is
typically `enabled=true`, `status=FORECLOSED`, `foreclose_block > 0`.

The old single `state` field (`ENABLED` / `DISABLED` / `INOPERABLE`) is replaced
by this split. JSON-RPC and CLI reads expose `enabled` and `status` — see
`cartesi-jsonrpc`.

### Claim finality (Authority / Quorum)

Authority and Quorum consensus now stage claims before acceptance:

```
CLAIM_COMPUTED → CLAIM_SUBMITTED → CLAIM_STAGED → (wait claimStagingPeriod) → CLAIM_ACCEPTED
```

- **Authority:** submit and stage happen in the **same transaction**; `acceptClaim` is a separate tx after the staging period elapses.
- **Quorum:** the node watches events until majority voting stages one claim, then `acceptClaim` later.
- **PRT (Dave tournaments):** skips `CLAIM_STAGED` — moves `CLAIM_COMPUTED` → `CLAIM_ACCEPTED` directly.

Vouchers remain executable only after `CLAIM_ACCEPTED`. Do not assume
`CLAIM_SUBMITTED` is sufficient on v3 Authority/Quorum deployments.

Foreclosure before acceptance terminalizes pre-foreclosure claim work as
`CLAIM_FORECLOSED`.

### Withdrawal config and emergency exit

Each v3 application carries a **`withdrawal_config`** set at deploy time
(`--withdrawal-config` or `--withdrawal-config-file` via `cartesi-rollups-cli`).
It defines the **guardian** (can call `foreclose()`) and the accounts-drive
layout required for post-foreclosure recovery.

After foreclosure, normal claim submission stops. Recovery is permissionless
(except the guardian-only foreclose step):

1. **Guardian** calls `foreclose()` — app becomes `FORECLOSED`; `foreclose_block` recorded on L1.
2. **Anyone** calls `proveAccountsDriveMerkleRoot(root, proof)` — once per app.
3. **Anyone** calls `withdraw(account, accountProof)` — gas payer can differ from recipient.

Proof files come from `cartesi-rollups-machine-tool` (replay snapshot →
`prove accounts-drive`). Operator CLI wraps steps 1–3:

```sh
cartesi-rollups-cli foreclose <app>
cartesi-rollups-cli prove-drive-root <app> --proof-file drive-root-proof.json
cartesi-rollups-cli withdraw <app> --proof-file account-proof.json
cartesi-rollups-cli read withdrawals <app>
```

This is **not** the same as in-app voucher withdrawals emitted by the backend
handler — those follow the normal notice/voucher path and execute via
`cartesi-rollups-cli execute`. Emergency withdrawal is an L1 recovery path
after foreclosure; see `cartesi-backend-core` for the distinction.

Apps that may need emergency withdrawal must use an **accounts-drive layout**
compatible with the machine tool (for example the `erc20-withdrawal-dapp`
template). Confirm `withdrawal_config` is valid **before** any on-chain deploy
tx — partial or invalid config must be rejected by the CLI.

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

### For testnets and production — contracts v3 (3.0.0-alpha.6)

**Primary source:** [`cartesi-rollups-3.0.0-alpha.6.json`](cartesi-rollups-3.0.0-alpha.6.json)
in this skill directory. Infrastructure addresses are **identical** on these
chains (deterministic Cannon deployment):

| Chain ID | Network |
| -------- | ------- |
| `1` | Ethereum Mainnet |
| `10` | Optimism Mainnet |
| `42161` | Arbitrum One |
| `8453` | Base Mainnet |
| `11155111` | Ethereum Sepolia |
| `11155420` | Optimism Sepolia |
| `421614` | Arbitrum Sepolia |
| `84532` | Base Sepolia |

Set `BLOCKCHAIN_ID` and RPC for your target chain; override compose factory
addresses from the manifest `contracts` object (Mugen-Builders defaults are
**not** this suite).

**Secondary verification:** Cannon registry (client-side rendered):

```
https://usecannon.com/packages/cartesi-rollups/3.0.0-alpha.6/<chain-id>-main/deployment/contracts
```

Confirm the version by checking the `cartesi-rollups-runtime` image tag in
`compose.local.yaml`. The node, contract suite, and DB schema must match.

**v3 infrastructure addresses** (all listed chains):

| Contract | Address |
| -------- | ------- |
| `InputBox` | `0x346B3df038FE9f8380071eC6514D5a83aD143939` |
| `AuthorityFactory` | `0x3C1FE01c542a88A523FF6847eD1E26176c8C4ED0` |
| `ApplicationFactory` | `0xC549F89cF1ca43eDDECC64Ac2208F4b283B1c483` |
| `SelfHostedApplicationFactory` | `0x6145C5996a71a379E030aEb0440df79D60833418` |
| `QuorumFactory` | `0x1f94009389F408B8D0ADfFcF8BBDCe5552BaCa5F` |
| `EtherPortal` | `0x8b53327575ac999bdfa8003f4b5134DFF9027516` |
| `ERC20Portal` | `0x22E57511C30CcE6CDaa742E13CE3b774fDC663b1` |
| `ERC721Portal` | `0xcA3a0a47915C12F020CF70B938aCC8e744414cb8` |
| `ERC1155SinglePortal` | `0x13663E193673756a02e84b724B8a3422A9a7aab4` |
| `ERC1155BatchPortal` | `0x3649c5E2De91C69a7Bb80D864f0039da5E511096` |
| `SafeERC20Transfer` | `0x15E45E779ED795E5ac4643f6C428B161ccDE7A61` |
| `UsdWithdrawalOutputBuilderFactory` | `0xdB4EC04a2792A04cF7421f99A70F624681dd8e50` |

| Contract | Role |
| -------- | ---- |
| `InputBox` | Single ingress point — all inputs enter Cartesi through here |
| `EtherPortal` | Bridge native ETH into the Cartesi Machine |
| `ERC20Portal` | Bridge ERC-20 tokens into the Cartesi Machine |
| `ERC721Portal` | Bridge ERC-721 NFTs into the Cartesi Machine |
| `ERC1155SinglePortal` | Bridge a single ERC-1155 token |
| `ERC1155BatchPortal` | Bridge a batch of ERC-1155 tokens |
| `AuthorityFactory` | Deploy Authority (single-validator) consensus |
| `QuorumFactory` | Deploy Quorum (multi-validator) consensus |
| `ApplicationFactory` | Deploy Cartesi Application contracts |
| `SelfHostedApplicationFactory` | Deploy application + authority in one tx |
| `DaveAppFactory` | Deploy PRT (Dave tournament) applications — resolve from Cannon if needed |
| `SafeERC20Transfer` | Helper for ERC-20 transfers via delegated call vouchers |

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

## Step 1 — IInputBox interface

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

interface IInputBox {
    function addInput(
        address application,
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
| Ether    | `EtherPortal`         | `depositEther(address app, bytes execLayerData)`                 |
| ERC-20   | `ERC20Portal`         | Approve first, then `depositERC20Tokens(...)`                    |
| ERC-721  | `ERC721Portal`        | `setApprovalForAll` first, then `depositERC721Token(...)`        |
| ERC-1155 | `ERC1155SinglePortal` | `setApprovalForAll` first, then `depositSingleERC1155Token(...)` |

### Local devnet portal addresses

When using `cartesi run`, all portal addresses are auto-deployed on Anvil. Resolve them with:

```sh
cartesi address-book
```

This lists `ERC20Portal`, `ERC721Portal`, `ERC1155SinglePortal`, `EtherPortal`, and all other deployed contracts. Do not use hardcoded addresses — they can change between CLI versions.

### Depositing ERC-20 via portal (cast commands)

```sh
# 1. Approve the portal to spend tokens
cast send <ERC20_TOKEN_ADDRESS> \
  "approve(address,uint256)" \
  <ERC20_PORTAL_ADDRESS> <AMOUNT_WEI> \
  --rpc-url <RPC_URL> --private-key <PRIVATE_KEY>

# 2. Deposit tokens into the Cartesi application
cast send <ERC20_PORTAL_ADDRESS> \
  "depositERC20Tokens(address,address,uint256,bytes)" \
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
  "depositERC721Token(address,address,uint256,bytes,bytes)" \
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
  "depositSingleERC1155Token(address,address,uint256,uint256,bytes,bytes)" \
  <ERC1155_ADDRESS> <APP_ADDRESS> <TOKEN_ID> <AMOUNT> 0x 0x \
  --rpc-url <RPC_URL> --private-key <PRIVATE_KEY>
```

### Handling portal deposits in your backend

Portal interactions arrive as advance inputs where `metadata.msg_sender` equals
the portal contract address. Your backend **must** check the sender to determine
how to decode the payload.

**ERC-20 deposit payload format (bytes layout):**

```
bytes  0–19:  token address (20 bytes)
bytes 20–39:  depositor address (20 bytes)
bytes 40–71:  amount (uint256, 32 bytes, big-endian)
```

JavaScript decode example:

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
be executed on-chain. On contracts v3 Authority/Quorum deployments, the
epoch must pass through **`CLAIM_STAGED`** and the on-chain staging period
before acceptance — poll epoch status via JSON-RPC or
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
        // or from https://usecannon.com/packages/cartesi-rollups/3.0.0-alpha.6/<chain-id>-main/deployment/contracts
        address inputBox   = vm.envAddress("INPUT_BOX_ADDRESS");
        address application = vm.envAddress("APP_ADDRESS");

        new MyCartesiFeeder(inputBox, application);

        vm.stopBroadcast();
    }
}
```

### Deploy command

```sh
# Local Anvil
forge script script/Deploy.s.sol:DeployScript \
  --rpc-url http://localhost:8545 \
  --broadcast \
  --private-key 0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80

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
  --rpc-url http://localhost:8545 \
  --private-key <key>
```

## Agent Output

After completing this skill, report back to the user with:

- Solidity contract(s) written: filename, contract name, constructor arguments
- Deployed contract address (from Foundry broadcast output)
- InputBox address used (sourced from `cartesi address-book` or Cannon registry)
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
- [Cannon registry — cartesi-rollups 3.0.0-alpha.6](https://usecannon.com/packages/cartesi-rollups/3.0.0-alpha.6/84532-main/deployment/contracts) — canonical v3 contract addresses (client-side rendered; open in browser)
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
- [ ] On v3 deploys: `withdrawal_config` validated before on-chain tx; guardian address matches funded key
- [ ] Epoch reached `CLAIM_ACCEPTED` (via `CLAIM_STAGED` on Authority/Quorum) before voucher execution
- [ ] Voucher execution tested with `cartesi-rollups-cli execute` after epoch acceptance
- [ ] Emergency withdrawal (`foreclose` → prove drive root → `withdraw`) distinguished from voucher `execute`

## What comes next

| Next task                        | Skill to use     |
| -------------------------------- | ---------------- |
| Deploy to self-hosted node       | `cartesi-deploy` |
| Debug contract or backend errors | `cartesi-debug`  |
