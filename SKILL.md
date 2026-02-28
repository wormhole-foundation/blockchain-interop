---
name: blockchain-interop
description: Wormhole ecosystem expertise — End-to-End lifecycle (Deploy, Transfer, Implement). Trigger this skill WHENEVER the user mentions Wormhole, cross-chain bridging, NTT (Native Token Transfers), CCTP, or Core Messaging. MUST trigger for deploying smart contracts (Hub-and-Spoke), configuring CLI/rate limits, or building bridge frontends/SDK integrations (TypeScript, Wormhole Connect React widgets, VAA layout serialization errors, missing deployment paths, 429 rate limits, and manual VAA claims).
---

# Wormhole Ecosystem & NTT Protocol

Comprehensive guide for the Wormhole cross-chain messaging ecosystem, with a specific focus on the Native Token Transfers (NTT) framework.

## When to Apply

Reference these guidelines when working within the Wormhole ecosystem:

- **End-to-End Orchestration:** Developing the full lifecycle of a Wormhole project (Smart Contract Strategy -> Frontend Integration -> Deployment Operations).
- **Architecture Planning:** Selecting the correct bridging primitive (NTT, Core Messaging, Connect, CCTP) based on token supremacy and liquidity needs.
- **Observability & Finality Design:** Designing robust, asynchronous frontend experiences that can gracefully handle multi-chain latencies (e.g., 15+ minute Guardian finality) and network rate limits.
- **Deep-Dive Engineering:** Navigating the dense, open-source Wormhole monorepo. When standard SDK docs fail, use this skill to understand the underlying mechanics of VAAs, rate limit bounds, and contract deployment gotchas.

## Product Ecosystem Overview

Wormhole operates via multiple interconnected products. When tasked with building or debugging in Wormhole, first identify which product applies to your objective and **immediately ingest its dynamic `llms.txt` documentation** before writing code:

### 1. Native Token Transfers (NTT)

**What it is:** A specialized framework for moving native tokens across chains seamlessly without liquidity pools or wrapping. Uses the `ntt` CLI for deployment.
**When to use:** Deploying cross-chain tokens, managing minting authorities, or configuring rate limits for a specific token project.
**Dynamic Docs:** `https://raw.githubusercontent.com/wormhole-foundation/wormhole-docs/main/llms-files/llms-ntt.txt`

### 2. Wormhole SDK (TypeScript)

**What it is:** The programmatic interface for writing Node.js backend scripts or frontend dApps.
**When to use:** Manually claiming VAAs on a frontend, programmatically executing transfers, parsing Wormhole messages, or writing custom bridge logic.
**Dynamic Docs:** `https://raw.githubusercontent.com/wormhole-foundation/wormhole-docs/main/llms-files/llms-typescript-sdk.txt`

### 3. Wormhole Connect (React / HTML)

**What it is:** A drop-in UI component (React or raw HTML widget) that gives users a pre-built frontend bridge experience.
**When to use:** Building user-facing React dApps that need a visual bridging widget quickly.
**Dynamic Docs:** `https://raw.githubusercontent.com/wormhole-foundation/wormhole-docs/main/llms-files/llms-connect.txt`

### 4. Cross-Chain Transfer Protocol (CCTP)

**What it is:** Circle's native USDC routing protocol, deeply integrated into Wormhole.
**When to use:** Explicitly and exclusively when transferring USDC across chains.
**Dynamic Docs:** `https://raw.githubusercontent.com/wormhole-foundation/wormhole-docs/main/llms-files/llms-cctp.txt`

### 5. Core Messaging & Relayers

**What it is:** The base layer of the Wormhole protocol that simply sends arbitrary byte messages (VAAs) between Smart Contracts.
**When to use:** Building low-level custom protocols, custom smart contract logic, or understanding how Guardians and Relayers function under the hood.
**Dynamic Docs:** `https://raw.githubusercontent.com/wormhole-foundation/wormhole-docs/main/llms-files/llms-relayers.txt`

## Multi-Chain Architecture Domains

Evaluate every Wormhole integration through the lens of these four architectural domains to ensure a successful, end-to-end multi-chain app deployment.

| Domain                       | Focus                     | Key Concept              | Examples                                                                                                                                       |
| ---------------------------- | ------------------------- | ------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------- |
| **Smart Contract Strategy**  | Protocol Design           | Mint/Burn vs Lock/Unlock | Does the origin chain lock liquidity while sibling chains mint? Evaluating token supply models.                                                |
| **Frontend Integration**     | User Experience           | Wormhole Connect vs SDK  | Should we embed a drop-in React widget for speed, or build a custom polling hook via `@wormhole-foundation/sdk` for control?                   |
| **Deployment Operations**    | Multi-Environment Control | CLI & Configuration      | Managing environment variables, orchestrating `ntt push` across testnets, defining rate limits in `deployment.json`.                           |
| **Observability & Finality** | Network States            | Verifiable VAAs          | Designing applications to handle asynchronous block finality (15+ min delays) and providing elegant VAA retry mechanisms for failed transfers. |

## Multi-Chain Deployment Principles

When guiding a user through building a multi-chain application, apply these overarching principles:

### 1. Smart Contract Strategy (Protocol Layer)

- **Token Supremacy:** Wormhole does not create tokens; it moves them. The user's underlying ERC20/SPL tokens must be structurally ready for bridging (i.e. explicitly granting Mint/Burn permissions to the NTT Manager, or relying on locked liquidity pools).
- **Data Serialization:** Any custom cross-chain data (VAAs) must have mathematically identical byte-layouts on all connected chains, otherwise serialization fails silently across network boundaries.

### 2. Frontend Integration (Application Layer)

- **Asynchronous UX:** The biggest hurdle in multi-chain apps is User Experience during finality delays. Frontends must cleanly communicate to the user that a transaction has submitted, but might take 15 minutes to be verified by Guardians before it can be claimed on the destination.
- **Graceful Retries:** Always anticipate `429 Rate Limit` errors when polling Guardian nodes via the SDK. Applications must implement exponential backoff rather than crashing.

### 3. Deployment Operations (Infrastructure Layer)

- **Configuration Parity:** The `deployment.json` file is the master state. Sibling chains must constantly sync their awareness of each other via `ntt pull` before new transfers are tested.
- **Testnet Volatility:** Assume testnet deployments will drop transactions. Guide users to rely on explicit gas multipliers (`--gas-estimate-multiplier`) and manual nonce-overrides instead of typical local-development foundry flags.

## How to Use

### Dynamic LLMs Context

If your task does not clearly fall into one of the 5 products listed above, consult the master index `llms.txt` to find the correct domain:

- **General Token Bridge:** `https://raw.githubusercontent.com/wormhole-foundation/wormhole-docs/main/llms-files/llms-token-bridge.txt`
- **Full Ecosystem Master Index:** `https://raw.githubusercontent.com/wormhole-foundation/wormhole-docs/main/llms.txt`

### Local Reference Files

Read the following reference files for step-by-step instructions, troubleshooting details, and code examples:

```
references/deployment-workflow.md  # Step-by-step chain deployment
references/testing-guide.md        # E2E testing procedures and balance checks
references/cli-commands.md         # Full command and flag reference
references/troubleshooting.md      # Detailed gotchas and manual VAA claiming
references/local-development.md    # Environment variables, deployment.json schema, and bun testing apps
```

## Core Concepts Background

- **Deployment Models:** Hub-and-Spoke (hub locks, spoke burns) or Burn-and-Mint (all burn).
- **Architecture:** NttManager controls limits/verification; Transceivers route messages.
- **Token Compatibility:** NTT calls `mint(address, uint256)` on the host token. The token owner grants permission via `grantRole(MINTER_ROLE, address)` or `setMinter(address)`.
- **Bun Environment:** Use `bun run cli/src/index.ts` for local development instead of `ntt`. Auto-loads `.env` from cwd.
