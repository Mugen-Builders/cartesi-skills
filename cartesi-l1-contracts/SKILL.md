---
name: cartesi-l1-contracts
description: >-
  Implement and integrate EVM smart contracts that interact with Cartesi Rollups
  through InputBox and related on-chain components. Focuses on Solidity design,
  access control, payload encoding, deployment scripts, and contract testing.
---

# Cartesi Rollups L1 Contracts

## Objective

Use this skill when the task involves Solidity/Foundry contracts that submit inputs or integrate with Cartesi rollups contracts.

In scope:

- Contracts calling `InputBox.addInput(application, payload)`;
- Payload encoding and ABI compatibility;
- Access control and event design;
- Foundry deployment scripts and tests;
- Chain-specific address wiring and validation.

Out of scope:

- Backend application logic implementation;
- Frontend interaction UX;
- Self-hosted node operation details.

## Authoritative references

- Cartesi Rollups docs root:
  [https://docs.cartesi.io/cartesi-rollups/2.0/](https://docs.cartesi.io/cartesi-rollups/2.0/)
- Send inputs and assets:
  [https://docs.cartesi.io/cartesi-rollups/2.0/development/send-inputs-and-assets/](https://docs.cartesi.io/cartesi-rollups/2.0/development/send-inputs-and-assets/)

## Contract design requirements

- Keep `application` and `inputBox` addresses explicit and validated at construction.
- Encode payloads deterministically and document the schema expected by backend.
- Restrict privileged calls (`onlyOwner`/roles/allowlists as appropriate).
- Emit meaningful events for each input submission.
- Include custom errors for common failure causes.

## Address and chain policy

- Treat Cartesi contract addresses as chain-specific.
- Resolve addresses from trusted sources (official deployment references or `cartesi address-book`).
- Reject silent fallback to zero addresses.
- Validate chain id in deployment scripts.

## Foundry workflow baseline

- `forge build`
- `forge test`
- `forge script ... --broadcast` for deployment

For production-like flows:

- require explicit env vars for private keys and RPC URLs;
- include dry-run/simulation mode before broadcast where possible.

## Testing requirements

At minimum:

- constructor/address validation tests;
- access control tests;
- payload encode/decode consistency checks;
- event emission assertions;
- revert-path assertions with custom errors.

## Debugging guidance

- If tx reverts with unknown selector, ensure ABI includes custom errors and latest contract build.
- Verify signer account permissions and chain id.
- Confirm app address and InputBox are from same target chain.

## Completion contract

Return:

- contract interaction flow and trust boundaries;
- payload schema used for `addInput`;
- deployed/wired addresses by chain;
- test matrix and deployment command examples.
