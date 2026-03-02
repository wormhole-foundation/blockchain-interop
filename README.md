# Blockchain Interop Skill (Wormhole Ecosystem)

[![Status](https://img.shields.io/badge/status-production%20ready-22c55e)](#)
[![Scope](https://img.shields.io/badge/scope-Wormhole%20Products-0ea5e9)](#)
[![Coverage](https://img.shields.io/badge/lifecycle-Architecture%20->%20Deploy%20->%20Operate-f59e0b)](#)

A practical, execution-first skill for building with the **Wormhole interoperability protocol** across multiple products, chains, and integration surfaces.

This repository is not NTT-only. It covers Wormhole product selection at a high level and includes local runbooks for the strongest implemented paths in this repo (currently centered on NTT deployment and Connect integration).

---

## Table of Contents

- [What This Repo Is](#what-this-repo-is)
- [Wormhole Product Coverage](#wormhole-product-coverage)
- [Quick Start](#quick-start)
- [Local References](#local-references)
- [Project Structure](#project-structure)
- [Contributing](#contributing)

---

## What This Repo Is

This repo provides a reusable skill framework for Wormhole-based cross-chain work:

- Product selection guidance (which Wormhole product to use for which job)
- Cross-chain architecture and implementation constraints
- Deployment and debugging workflows for real environments
- Local references for NTT lifecycle execution and Connect setup

---

## Wormhole Product Coverage

High-level product map (aligned with Wormhole Docs):

- **Messaging**: arbitrary cross-chain message passing between contracts/apps.
- **Token Transfers (WTT)**: wrapped token transfer model for moving assets across chains.
- **Native Token Transfers (NTT)**: issuer-controlled native token transfer framework.
- **CCTP Bridge**: native USDC transfer and messaging flow powered by CCTP.
- **Connect**: drop-in bridge UI for apps (React / web).
- **Settlement**: intent-based cross-chain execution and routing.
- **Queries**: fetch and verify on-chain state across chains.
- **MultiGov**: multi-chain governance coordination.
- **TypeScript SDK + CLI Tooling**: programmatic integrations and operational workflows.

Primary docs: `https://wormhole.com/docs/`

---

## Quick Start

### 1. Start with Product Selection

Choose the product first, based on goal:

- Move arbitrary data: Messaging
- Move tokens: Token Transfers / NTT / CCTP Bridge
- Embed UI quickly: Connect
- Execute intent flows: Settlement
- Read remote chain state: Queries
- Coordinate governance cross-chain: MultiGov

### 2. Read Core Skill Guidance

- `SKILL.md` for ecosystem-wide rules, product mapping, and execution discipline

### 3. Use Local Runbooks for Implemented Flows

For current deep implementation support in this repo:

1. `references/deployment-workflow.md`
2. `references/troubleshooting.md`
3. `references/testing-guide.md`
4. `references/connect-integration.md`

---

## Local References

- `references/deployment-workflow.md` - NTT chain deployment workflow
- `references/testing-guide.md` - E2E and transfer validation
- `references/cli-commands.md` - NTT CLI commands and flags
- `references/troubleshooting.md` - failure modes and recovery steps
- `references/local-development.md` - env vars and local execution details
- `references/connect-integration.md` - Connect integration for NTT routes

Notes:

- Local references are currently deepest for NTT and Connect scenarios.
- Other Wormhole products are covered in `SKILL.md` with links to official docs.

---

## Project Structure

```text
blockchain-interop/
|-- README.md
|-- SKILL.md
`-- references/
    |-- deployment-workflow.md
    |-- testing-guide.md
    |-- cli-commands.md
    |-- troubleshooting.md
    |-- local-development.md
    `-- connect-integration.md
```

---

## Contributing

Contributions are welcome when they improve correctness and operational clarity.

High-value additions:

- New runbooks for non-NTT products (Messaging, Settlement, Queries, MultiGov)
- Product-specific troubleshooting playbooks
- Version compatibility updates for SDK, Connect, CLI, and protocol packages
