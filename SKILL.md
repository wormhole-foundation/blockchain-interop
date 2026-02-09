---
name: wormhole
description: Wormhole cross-chain protocol knowledge — NTT (Native Token Transfers) deployment, CLI commands, architecture, and testing workflows. Use when working with Wormhole NTT CLI, deploying cross-chain token transfers, or debugging NTT configurations.
---

# Wormhole Protocol — NTT Focus

## Core Concepts

**Wormhole** is a cross-chain messaging protocol. **NTT (Native Token Transfers)** is its framework for transferring native tokens across blockchains without wrapping.

### Deployment Models

- **Hub-and-Spoke (locking):** Hub chain locks tokens, spoke chains mint. Hub uses `locking` mode, spokes use `burning` mode. Only needs standard ERC-20 on hub.
- **Burn-and-Mint:** All chains use `burning` mode. Requires `burn(uint256)` and `mint(address,uint256)` on token contract. Mint authority must be set to NTT Manager on ALL chains.
- **Lock-and-Lock:** NOT supported by NTT.

### Architecture

- **NttManager:** Controls token locking/burning, rate limiting, message verification
- **Transceivers:** Route messages cross-chain (Wormhole Guardian or custom verification)
- **INttToken interface:** Defines `mint(address,uint256)`, `burn(uint256)`, `setMinter(address)` — but NTT never calls `setMinter` itself. It only calls `mint` and relies on `burn` being available.
- Same transceiver type required for all routes (can't mix per-chain)

### Token Compatibility (Burning Mode)

NTT works with two common ERC-20 patterns for minting permission:

1. **AccessControl (OZ):** Token uses `grantRole(MINTER_ROLE, address)`. Grant `MINTER_ROLE` to the NTT Manager. `MINTER_ROLE = keccak256("MINTER_ROLE") = 0x9f2df0fed...`
2. **Ownable + INttToken:** Token has `setMinter(address)` or similar owner-only method to designate one minter. Call it to set NTT Manager as the minter.

In both cases, NTT Manager only ever calls `mint(address, uint256)` on the token. The `burn(uint256)` is typically from ERC20Burnable (permissionless for own tokens). The `setMinter` in the INttToken interface is for the token owner to configure — NTT contracts never invoke it.

### Rate Limiting

- Outbound: single limit for all destinations
- Inbound: configurable per source chain
- Uses 18 decimals for EVM, 9 for SVM/Sui
- Zero limits cause stuck transactions — always set > 0
- Queuing: `shouldQueue=true` queues transfers exceeding limits

## NTT CLI Quick Reference

See `references/cli-commands.md` for full command reference.
See `references/deployment-workflow.md` for step-by-step deployment.
See `references/testing-guide.md` for E2E testing procedures.

### Key Commands

```bash
ntt new <path>              # Create project
ntt init Testnet|Mainnet    # Initialize deployment.json
ntt add-chain <Chain> --latest --mode burning|locking --token <addr> [--skip-verify]
ntt push                    # Push config on-chain
ntt pull                    # Sync local from on-chain
ntt status                  # Verify deployment
ntt token-transfer          # Transfer tokens between chains
```

### Environment Variables

Bun auto-loads `.env` from cwd — just place `.env` in the project directory (next to `deployment.json`). No manual `export` needed.

```bash
# .env
ETH_PRIVATE_KEY=0x...                    # EVM deployer key
SEPOLIA_SCAN_API_KEY=...                 # Etherscan API key (chain-specific prefix)
BASESEPOLIA_SCAN_API_KEY=...             # Base Sepolia scan key
```

### deployment.json Structure

```json
{
    "network": "Testnet",
    "chains": {
        "Sepolia": {
            "version": "3.0.0",
            "mode": "burning",
            "paused": false,
            "owner": "0x...",
            "manager": "0x...",
            "token": "0x...",
            "transceivers": {
                "threshold": 1,
                "wormhole": { "address": "0x..." }
            },
            "limits": {
                "outbound": "1000.000000000000000000",
                "inbound": { "BaseSepolia": "500.000000000000000000" }
            }
        }
    }
}
```

## Local Development (This Repo)

### Running NTT CLI locally

```bash
bun run cli/src/index.ts <command> [args]    # NOT `ntt` — use bun directly
bun run cli/src/index.ts --help
bun run cli/src/index.ts add-chain --help
```

### Testing

```bash
bun test cli/src/__tests__/                  # Run all tests (from project root!)
bun run --cwd cli typecheck                  # TypeScript check
```

### Chain Names (Testnet)

Sepolia, BaseSepolia, ArbitrumSepolia, OptimismSepolia, Solana (devnet)

### Setting Mint Authority (EVM burning mode)

**INttToken pattern** (has `setMinter`):

```bash
cast send $TOKEN_ADDRESS "setMinter(address)" $NTT_MANAGER_ADDRESS \
    --private-key $ETH_PRIVATE_KEY --rpc-url $RPC_URL
```

**AccessControl pattern** (OZ `grantRole`):

```bash
MINTER_ROLE=0x9f2df0fed2c77648de5860a4cc508cd0818c85b8b8a1ab4ceeef8d981c8956a6
cast send $TOKEN_ADDRESS "grantRole(bytes32,address)" $MINTER_ROLE $NTT_MANAGER_ADDRESS \
    --private-key $ETH_PRIVATE_KEY --rpc-url $RPC_URL
```

NTT Manager only calls `mint(address,uint256)` — as long as it has permission, both patterns work.

## Common Gotchas

1. Token only needs `mint(address,uint256)` and `burn(uint256)` for burning mode. NTT never calls `setMinter` — permission is granted by the token owner via `grantRole` (AccessControl) or `setMinter` (Ownable), depending on the token's pattern.
2. `ntt pull` must be run to configure decimals in deployment.json
3. Scan API key env var format: `<CHAIN>_SCAN_API_KEY` (e.g., `SEPOLIA_SCAN_API_KEY`)
4. Rate limit decimals: 18 for EVM, 9 for SVM/Sui
5. Wormhole Testnet uses real testnets (Sepolia, Base Sepolia, etc.)
6. `set-mint-authority` CLI command is SVM-only. For EVM, grant permission manually via cast.
7. Bun auto-loads `.env` from cwd — never manually export private keys. Keep them in `.env` next to `deployment.json`.
8. Deployment workflow: `add-chain` (deploy) -> `pull` (sync decimals/state) -> edit limits in deployment.json -> `push` (set peers + limits) -> `status` (verify).
