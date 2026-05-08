---
name: cartesi-local-dev
description: >-
  Run and troubleshoot Cartesi Rollups v2 applications in local and forked
  development environments. Covers fast inner-loop commands, chain/rpc
  alignment, inspect and input validation checks, and common local workflow
  failures.
---

# Cartesi Rollups Local Dev

## Objective

Use this skill for local development velocity and reliability, not for production deployment.

In scope:

- `cartesi build` and `cartesi run` workflows;
- forked-mode execution and testing;
- inspect/input smoke checks;
- local chain-id and RPC alignment;
- dev proxy/CORS and endpoint mismatch troubleshooting.

Out of scope:

- Self-hosted deploy orchestration (use `cartesi-deploy`);
- Business/domain logic design (use backend skills);
- Frontend styling/design concerns.

## Authoritative references

- Cartesi Rollups 2.0 docs:
  [https://docs.cartesi.io/cartesi-rollups/2.0/](https://docs.cartesi.io/cartesi-rollups/2.0/)
- Development guides:
  [https://docs.cartesi.io/cartesi-rollups/2.0/development/](https://docs.cartesi.io/cartesi-rollups/2.0/development/)
- Inputs and inspect:
  [https://docs.cartesi.io/cartesi-rollups/2.0/development/send-inputs-and-assets/](https://docs.cartesi.io/cartesi-rollups/2.0/development/send-inputs-and-assets/)

## Standard local loop

```sh
cartesi build
cartesi run -p <port>
```

Then verify:

1. One advance input accepted.
2. One inspect route returns expected report format.
3. Decoding path (hex/string/json) is correct end-to-end.

## Forked workflow

```sh
cartesi run --fork-url <RPC_URL> [--fork-block-number <N>] -p <port>
```

Use fork mode when reproducing behavior tied to deployed contracts/state.

Remember:

- Fork can appear static for external reads at fixed block heights.
- Repeatability is useful; dynamic updates may require new fork point/inputs.

## Local configuration checklist

- Wallet/signer chain id matches L1 RPC.
- Application address is from current run/deployment.
- Inspect URL and JSON-RPC URL are not conflated.
- Frontend/API proxies point to the correct local services.

## Fast troubleshooting map

- `invalid chain id for signer` -> chain mismatch between signer and RPC.
- Inspect succeeds, no state change -> no accepted advance, wrong app address, or stale runtime.
- Wrong output decoding -> sender/backend encoding mismatch.
- CORS errors in frontend dev -> missing local proxy.
- Port confusion -> read runtime output and pin explicit URLs in config.

## Completion contract

When done, return:

- commands used;
- local endpoints confirmed;
- smoke-test payloads used;
- observed outputs and decoding result;
- unresolved local risks.
