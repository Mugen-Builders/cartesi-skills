# Changelog

All notable changes to this project are documented in this file.

## [0.1.0] — 2026-05-12

First release of reusable **Cartesi Rollups v2** skill packs for AI coding agents.

### Compatibility & baseline deps

- Written for **Cartesi Rollups v2**, with docs aligned to **Cartesi CLI `2.0.0-alpha.34`**, **rollups-node `2.0.0-alpha.11`**, and **rollups contracts v2.2.0** (resolve live addresses per chain as each skill describes).
- **`cartesi-deploy`** follows the Mugen-Builders self-hosted Compose setup: [Mugen-Builders/deployment-setup-v2.0](https://github.com/Mugen-Builders/deployment-setup-v2.0) (`compose.local.yaml`).

### Skills in this release

- **`cartesi-workflow`**, **`cartesi-scaffold`**, **`cartesi-backend-core`**, **`cartesi-backend-py`**, **`cartesi-backend-js-ts`**, **`cartesi-contracts`**, **`cartesi-frontend`** (optional **`cartesi-frontend/DESIGN.md`** for UI tokens), **`cartesi-local-dev`**, **`cartesi-deploy`**, **`cartesi-jsonrpc`**, **`cartesi-debug`**.

### Notes

- Skills are guidance only; verify CLI output, ports, and image tags on your machine.
- The same skill packs are published live at [https://skills.mugen.builders/](https://skills.mugen.builders/).
