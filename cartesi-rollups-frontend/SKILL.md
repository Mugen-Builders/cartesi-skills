---
name: cartesi-rollups-frontend
description: >-
  Build Cartesi Rollups v2 frontend applications with clear module boundaries:
  wallet integration, app I/O, configuration, and optional asset bridging.
  Use this whenever the user asks for React/Next.js/Angular (or any web frontend)
  that connects wallets, sends inputs to InputBox, reads machine state via Cartesi
  JSON-RPC, or uses @cartesi/wagmi and @cartesi/viem packages.
---

# Cartesi Rollups v2 frontend app

## Goal

Implement a minimal but production-shaped Cartesi frontend with clean separation:

- **Wallet operations** (connect, disconnect, account, chain);
- **App interactions (I/O)** (send inputs, read outputs/reports through JSON-RPC);
- **Configuration** (addresses, RPC URLs, chain and environment setup);
- **Asset bridging** as a **separate component** only when needed.

Use vanilla CSS by default. Only switch UI styling approach if the user provides a specific design spec (for example, `design.md`).

## Authoritative references

- React frontend reference:
  [Build a React frontend for Cartesi Apps](https://docs.cartesi.io/cartesi-rollups/2.0/tutorials/react-frontend-application/)
- JSON-RPC methods and data model:
  [Cartesi Rollups v2 JSON-RPC Overview](https://docs.cartesi.io/cartesi-rollups/2.0/api-reference/jsonrpc/overview/)
- For non-React frameworks (Next.js, Angular, others), use each framework's official docs for app structure and rendering conventions:
  - [Next.js Docs](https://nextjs.org/docs)
  - [Angular Docs](https://angular.dev/overview)

## Baseline packages

Use current Cartesi alpha packages (or newer compatible alphas):

```sh
pnpm add @cartesi/wagmi@2.0.0-alpha.10 @cartesi/viem@2.0.0-alpha.10
```

If the user asks for "latest alpha", prefer latest compatible alpha versions and mention the selected versions explicitly.

## Required architecture

Keep these concerns in distinct modules/files:

1. **Configuration layer**
   - Environment parsing (`APP_ADDRESS`, Cartesi JSON-RPC URL, chain IDs).
   - Typed config object and validation on startup.
   - No wallet or business logic in config module.

2. **Wallet layer**
   - Wallet connectors (MetaMask/injected recommended by default).
   - Connection state and account/chain helpers.
   - No InputBox calls or JSON-RPC data fetching here.

3. **Cartesi I/O layer**
   - Send input payloads to InputBox contract.
   - Read notices, vouchers, delegated vouchers, reports, and input/epoch status via Cartesi JSON-RPC.
   - Data transformation utilities (hex <-> utf-8, formatting and pagination).

4. **UI layer**
   - Composes wallet + I/O modules into components/pages.
   - Uses vanilla CSS unless design requirements override.

5. **Optional asset bridging layer**
   - Separate component(s) for Ether/ERC20/ERC721 deposit/withdraw bridge interactions.
   - Do not mix bridge code into base input/output components.

## Implementation workflow

### 1) Scaffold for target framework

- **React**: follow Cartesi's React tutorial shape for providers/components.
- **Next.js/Angular/etc.**: apply same Cartesi integration model, but follow official framework conventions for routing, SSR, and state boundaries.

### 2) Set up providers and wallet

- Configure wagmi client and connectors.
- Prefer injected/MetaMask as the default connector unless user asks otherwise.
- Add `CartesiProvider` (or equivalent integration point) with JSON-RPC URL from config.

### 3) Send inputs through InputBox

- Use wallet client extended with Cartesi L1 actions from `@cartesi/viem`.
- Encode payloads (for example with `stringToHex`) consistently with backend expectations.
- Show tx hash and failure states in UI.

### 4) Read machine outputs via JSON-RPC

- Use Cartesi hooks/client from `@cartesi/wagmi` where practical.
- For low-level calls, map to documented JSON-RPC methods:
  - application listing/details
  - epoch listing/details
  - input listing/details and processed count
  - output listing/details
  - report listing/details
- Implement pagination and optional filters (`limit`, `offset`, `epoch_index`, `input_index`, etc.).
- Handle JSON-RPC error objects explicitly.

### 5) Add optional bridge component(s)

- Only when app requirements include asset bridging.
- Keep bridge transactions and state in dedicated files/components.
- Reuse shared wallet/config utilities, but keep bridge UI and logic isolated.

## Minimal output contract

When generating or refactoring frontend code, return:

- A short module map showing where wallet, config, I/O, and optional bridge live.
- Required environment variables.
- Installed package commands.
- A runnable baseline path (for example: connect wallet -> send input -> read outputs).

Keep examples concise; avoid over-engineering and unnecessary abstractions.

## Quality checks

Before finishing, verify:

- Wallet logic is not mixed with Cartesi JSON-RPC read code.
- Input submission goes through InputBox contract interactions.
- Output/report reads come from Cartesi JSON-RPC API paths or corresponding wrappers.
- Bridge logic is absent unless requested; if present, it is isolated.
- UI styling is vanilla CSS unless user provided design requirements.
