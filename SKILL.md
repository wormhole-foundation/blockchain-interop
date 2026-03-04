---
name: blockchain-interop
description: Wormhole interoperability expertise across the full product suite (Messaging, Token Transfers, NTT, CCTP Bridge, Connect, Settlement, Queries, MultiGov, SDK/CLI). Trigger this skill whenever users ask for Wormhole architecture, deployment, integration, operations, or debugging.
---

# Wormhole Interoperability Protocol & Product Suite

Comprehensive guide for building and operating cross-chain applications with Wormhole.

This skill is ecosystem-wide. NTT is one major workflow, not the only one.

## When to Apply

Use this skill when tasks involve any Wormhole product or core protocol behavior:

- Product and architecture selection for cross-chain apps
- Token transfer design (WTT, NTT, CCTP Bridge)
- Messaging protocol implementation and delivery guarantees
- Frontend bridge UX with Connect or custom SDK-based flows
- Intent and execution flows with Settlement
- Cross-chain state reads with Queries
- Governance operations with MultiGov
- Deployment, monitoring, and failure recovery across chains

## Product Suite Overview

Use the official docs root for navigation: `https://wormhole.com/docs/`.

| Product | High-level purpose | Typical use case |
| --- | --- | --- |
| Messaging | Pass arbitrary messages cross-chain | Custom app-level interoperability and protocol signaling |
| Token Transfers | Move assets across chains using token transfer primitives | General token bridge behavior and wrapped token flows |
| NTT (Native Token Transfers) | Native token transfer framework controlled by token issuers | Issuer-controlled token expansion across chains |
| CCTP Bridge | Native USDC transfer and messaging path | USDC-specific cross-chain payments and treasury flows |
| Connect | Drop-in transfer UI for apps | Fast integration of bridge UX in React/web apps |
| Settlement | Intent-based cross-chain execution framework | Intent routing and destination execution patterns |
| Queries | Read/verify on-chain state across networks | Cross-chain data access for apps and automation |
| MultiGov | Multi-chain governance coordination | Governance actions spanning multiple chains |
| TypeScript SDK | Programmatic interface to Wormhole products | Backend scripts, relayers, and custom dApp integrations |
| CLI Tooling | Operational commands for product workflows (for example NTT CLI) | Deployment, config sync, and status checks |

### Product docs links

- Messaging: `https://wormhole.com/docs/products/messaging/overview/`
- Token Transfers: `https://wormhole.com/docs/products/token-transfers/overview/`
- NTT: `https://wormhole.com/docs/products/token-transfers/native-token-transfers/overview/`
- CCTP Bridge: `https://wormhole.com/docs/products/token-transfers/cctp-bridge/overview/`
- Connect: `https://wormhole.com/docs/products/connect/overview/`
- Settlement: `https://wormhole.com/docs/products/settlement/overview/`
- Queries: `https://wormhole.com/docs/products/queries/overview/`
- MultiGov: `https://wormhole.com/docs/products/multigov/overview/`
- TypeScript SDK: `https://wormhole.com/docs/products/typescript-sdk/overview/`

## Product Selection Quick Guide

Map user intent to product first, then implement:

- "Send arbitrary payloads across chains" -> Messaging
- "Move token value across chains" -> Token Transfers / NTT / CCTP Bridge
- "USDC only" -> CCTP Bridge
- "Ship UI fast" -> Connect
- "Need custom transfer UX or backend control" -> TypeScript SDK
- "Need intent execution" -> Settlement
- "Need remote chain reads" -> Queries
- "Need governance over many chains" -> MultiGov

## Multi-Chain Architecture Domains

Evaluate every implementation through these domains:

| Domain | Focus | Key question |
| --- | --- | --- |
| Smart Contract Strategy | Asset and message model | Are authority, mint/burn, and verification boundaries explicit? |
| Frontend Integration | User journey and state transitions | Is asynchronous finality represented clearly in UX? |
| Deployment Operations | Multi-env consistency and configuration | Is environment/config parity maintained across chains? |
| Observability & Finality | Runtime confidence and recovery | Are attestations, retries, and manual recovery paths defined? |

## Universal Ecosystem Constraints

### 1. Strict version parity (critical)

Wormhole integrations often compose multiple packages (SDK core, product plugins, Connect routes, CLI tooling).
Version mismatches can fail at runtime with misleading errors.

Rules:

- Verify package compatibility before and after upgrades.
- Keep SDK core and product plugins aligned.
- Avoid mixing old and new examples without checking docs.

### 2. Asynchronous finality is normal

Guardian attestations and destination execution can take minutes depending on source chain finality.

Rules:

- Design UX and automation around eventual completion.
- Treat temporary "vaa not found" or waiting states as expected until finality.
- Implement retry/backoff for RPC and guardian polling, especially on 429s.

### 3. Step-by-step execution discipline

Cross-chain workflows are order-dependent. Skipping prerequisites causes cascading failures.

Rules:

- Follow runbooks in order for the selected product.
- Do not improvise workaround commands when a documented step fails.
- Reconcile local config with on-chain state before retrying.

### 4. Anti-hallucination directive

If a command, flag, API signature, or contract behavior is uncertain, look it up first.

Rules:

- Never fabricate CLI flags, function signatures, or config fields.
- Use local references first, then official docs pages, then dynamic `llms.txt` context.
- Keep assumptions explicit when behavior differs by chain family (EVM/SVM/Sui).

### 5. Secrets handling

- Never hardcode private keys, RPC URLs, or API keys in source.
- Use environment variables and secure secret managers.
- If a quick test needs inline secrets, label it unsafe and temporary.

## How to Use

### Standard execution order (all products)

1. Identify target product and chain set from user goal.
2. Read product overview docs and relevant local references.
3. Implement the minimum working path first.
4. Validate with end-to-end checks and failure-path checks.
5. Add observability and recovery steps before production rollout.

### NTT specialization (current deepest local coverage)

When the task is NTT deployment/operations, follow this exact order:

1. Read `references/deployment-workflow.md`.
2. Read `references/troubleshooting.md` before running commands.
3. Execute commands sequentially; do not skip setup steps.
4. Validate with `references/testing-guide.md`.

NTT-specific non-negotiables:

- `ntt new` is mandatory before `ntt init`.
- Run `ntt pull` to sync local state before transfer tests.
- Configure non-zero rate limits and correct decimals.
- For EVM burning mode, ensure manager mint permissions are granted.

## Dynamic LLMs Context

Use dynamic docs only when local references and standard docs are not enough, or when verifying exact signatures/version behavior.

- Messaging/Relayers: `https://raw.githubusercontent.com/wormhole-foundation/wormhole-docs/main/llms-files/llms-relayers.txt`
- Token Transfers: `https://raw.githubusercontent.com/wormhole-foundation/wormhole-docs/main/llms-files/llms-token-transfers.txt`
- NTT: `https://raw.githubusercontent.com/wormhole-foundation/wormhole-docs/main/llms-files/llms-ntt.txt`
- CCTP: `https://raw.githubusercontent.com/wormhole-foundation/wormhole-docs/main/llms-files/llms-cctp.txt`
- Connect: `https://raw.githubusercontent.com/wormhole-foundation/wormhole-docs/main/llms-files/llms-connect.txt`
- Settlement: `https://raw.githubusercontent.com/wormhole-foundation/wormhole-docs/main/llms-files/llms-settlement.txt`
- TypeScript SDK: `https://raw.githubusercontent.com/wormhole-foundation/wormhole-docs/main/llms-files/llms-typescript-sdk.txt`
- Master index: `https://raw.githubusercontent.com/wormhole-foundation/wormhole-docs/main/llms.txt`

For large `llms-*.txt` files, use `curl` from terminal:

```bash
curl -sL "https://raw.githubusercontent.com/wormhole-foundation/wormhole-docs/main/llms-files/llms-ntt.txt" > /tmp/llms-ntt.txt
grep -i "search term" /tmp/llms-ntt.txt
```

## Local Reference Files

Current local references are strongest for NTT and Connect implementation:

```text
references/deployment-workflow.md  # NTT deployment runbook
references/testing-guide.md        # E2E testing and balance checks
references/cli-commands.md         # NTT CLI command reference
references/troubleshooting.md      # Recovery and manual claim patterns
references/local-development.md    # Environment/config details
references/connect-integration.md  # Connect + NTT configuration
references/governance.md           # Guardian governance ceremonies (upgrades, peer registration)
references/hyperevm.md             # HyperEVM-specific deployment, verification, and big block mode
```

## Fallback Strategy for Unknowns

When documentation is insufficient:

1. Re-read local `references/` docs.
2. Check official Wormhole product docs pages.
3. Use dynamic `llms.txt` files for exact signatures/flags.
4. If available, query MCP doc sources (`wormhole-foundation/wormhole-docs`, `wormhole-foundation/wormhole-sdk-ts`).
5. Ask the user for missing project-specific constraints.

## Core Concepts Background

- Wormhole interoperability is asynchronous; attestations and destination execution are separate stages.
- Product choice is architecture: token movement, message movement, and UI strategy are different decisions.
- Cross-chain reliability depends on state parity, retries, and explicit recovery paths.
- For local CLI development inside Wormhole monorepo workflows, use Bun and keep env/config close to `deployment.json`.
