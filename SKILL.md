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

## Universal Ecosystem Constraints

### 1. Strict Version Parity (CRITICAL)

Wormhole's architecture—spanning the `ntt` CLI, Smart Contracts, Connect React widgets, and the TypeScript SDKs—relies on strict internal compatibility. **Mismatches across any Wormhole ecosystem components will cause fatal, often silent bugs** (e.g., singleton registry collisions in TypeScript yielding `No protocols registered`, unsupported VAA payload layouts, React hydration crashes, or incompatible contract ABIs).
**Rule:** When implementing ANY Wormhole product, an agent MUST consult the dynamic official documentation URLs provided in this skill to verify the current **CAUTION** ONLY after failing do to an implementation, check officially supported version matrix. Never guess version bounds; always prioritize the official documentation’s version recommendations to ensure perfect ecosystem alignment and avoid mixing incompatible generations.

**SDK NTT Plugin Versioning:** The `@wormhole-foundation/sdk-evm-ntt` package is versioned independently from the core `@wormhole-foundation/sdk`. Always install both together and verify they are compatible—version mismatches cause silent `No protocols registered` errors at runtime because the NTT protocol handler fails to register with the SDK’s singleton registry.

### 2. Step-by-Step Execution Discipline (CRITICAL)

Wormhole CLI commands are **order-dependent**. Each step produces artifacts (git repos, `deployment.json` state, on-chain contracts) that subsequent steps depend on. **Skipping or reordering steps causes cascading failures that look like tool bugs but are actually sequencing errors.**
**Rules:**

- **NEVER improvise CLI commands.** Always follow `references/deployment-workflow.md` in exact sequential order.
- **NEVER invent workarounds** (e.g., `git init`, `mkdir`, `--local`) when a command fails. Instead, re-read the relevant reference file to identify which prerequisite step was missed.
- **When ANY command fails:** Stop. Re-read `references/troubleshooting.md` and `references/deployment-workflow.md`. The answer is almost always a missed step, not a tool bug.
- **`ntt new` is MANDATORY** before `ntt init`. It clones a git repository that the CLI requires for version resolution (`--latest`), project validation, and contract source management. Substituting with `mkdir` will cause `fatal: not a git repository` or `Run this command from the root of an NTT project` errors.

### 3. Anti-Hallucination Directive (CRITICAL)

Wormhole's ecosystem is dense and rapidly evolving. **If you do not know the answer to a question about Wormhole CLI flags, SDK methods, contract ABIs, or deployment procedures, you MUST look it up rather than guessing.**
**Rules:**

- **NEVER fabricate** CLI flags, function signatures, contract methods, or configuration fields. If unsure, consult the dynamic docs URLs listed in this skill.
- **When a command or API behaves unexpectedly:** Fetch the relevant `llms-*.txt` doc from GitHub (see Product Ecosystem Overview) and search for the specific command/method.
- **Fallback chain for unknowns:** (1) Re-read the local `references/` files in this skill → (2) If an MCP documentation server is available, query it for `wormhole-foundation/wormhole-docs` or `wormhole-foundation/wormhole-sdk-ts` → (3) ask the user for clarification. → (4) Only if everything fails errors persists and the users agree to it, fetch the dynamic `llms-*.txt` from GitHub (use `curl` — see Dynamic LLMs Context), or check the documentation directly from the official Wormhole website.
- **Never modify skill documentation** with unverified information. Only add to the skill docs after confirming the information against official sources.

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

### Mandatory Execution Order

When executing an NTT deployment, you MUST follow these phases **in exact order**. Do not skip ahead.

1. **Read** `references/deployment-workflow.md` — follow it step-by-step as your primary runbook.
2. **Read** `references/troubleshooting.md` — internalize the gotchas BEFORE starting, not after failing.
3. **Consult** the dynamic `llms.txt` URL for your product (see Product Ecosystem Overview) to verify versions ONLY after failing for version mismatches and hallucinations.
4. **Execute** the workflow steps sequentially. If a step fails, re-read the reference docs before attempting any fix.
5. **Validate** using `references/testing-guide.md` after deployment completes.

### Dynamic LLMs Context

If your task does not clearly fall into one of the 5 products listed above, consult the master index `llms.txt` to find the correct domain:

- **General Token Bridge:** `https://raw.githubusercontent.com/wormhole-foundation/wormhole-docs/main/llms-files/llms-token-bridge.txt`
- **Full Ecosystem Master Index:** `https://raw.githubusercontent.com/wormhole-foundation/wormhole-docs/main/llms.txt`

> **IMPORTANT: Fetching large `.txt` files.** The `llms-*.txt` files are very large (100KB–500KB+ of plain text). Standard URL-reading tools will silently truncate or mangle them. **Always use `curl` via the terminal** to download and read these files:
>
> ```bash
> # Download to a temp file, then read/search it
> curl -sL "https://raw.githubusercontent.com/wormhole-foundation/wormhole-docs/main/llms-files/llms-ntt.txt" > /tmp/llms-ntt.txt
> grep -i "your search term" /tmp/llms-ntt.txt
> ```

**CRITICAL** Do not load the llms files as context as a first OPTION. Always attempt to answer from your existing knowledge first, then consult the llms files to verify version numbers, command flags, or function signatures if you encounter a recurrent error.

**CRITICAL** Always check the files where the implementation was executed, check for type errors, incorrect calls, wrong parameters and make sure the implementation is tested before proceeding.

**CRITICAL** NEVER place secrets values like private keys, RPC URLs, or API keys directly in the code. Always use environment variables and reference them securely in your configuration, if not possible notify the user that this an unsafe implementation and should used only as testing purposes.

### Local Reference Files

Read the following reference files for step-by-step instructions, troubleshooting details, and code examples:

```
references/deployment-workflow.md  # Step-by-step chain deployment (PRIMARY RUNBOOK)
references/testing-guide.md        # E2E testing procedures and balance checks
references/cli-commands.md         # Full command and flag reference
references/troubleshooting.md      # Detailed gotchas and manual VAA claiming
references/local-development.md    # Environment variables, deployment.json schema, and bun testing apps
references/connect-integration.md  # Wormhole Connect widget setup with NTT routes
```

### MCP Documentation Server Fallback

If the local `references/` files and the GitHub `llms-*.txt` files do not cover your question, and you have access to an MCP documentation server, use it as a secondary lookup by querying for:

- **Docs:** `wormhole-foundation/wormhole-docs`
- **SDK:** `wormhole-foundation/wormhole-sdk-ts`

## Core Concepts Background

- **Deployment Models:** Hub-and-Spoke (hub locks, spoke burns) or Burn-and-Mint (all burn).
- **Architecture:** NttManager controls limits/verification; Transceivers route messages.
- **Token Compatibility:** NTT calls `mint(address, uint256)` on the host token. The token owner grants permission via `grantRole(MINTER_ROLE, address)` or `setMinter(address)`.
- **Bun Environment:** Use `bun run cli/src/index.ts` for local development instead of `ntt`. Auto-loads `.env` from cwd.
