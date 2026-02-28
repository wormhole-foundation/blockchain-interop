# NTT Environment & Local Development

This reference covers the required environment variables, configuration schemas, and commands for developing or testing the NTT CLI locally within the Wormhole repository.

## Environment Variables

Bun auto-loads `.env` from the Current Working Directory (cwd). You do not need to manually `export` these before running commands. Place your `.env` file next to your `deployment.json`.

```bash
# .env required values
ETH_PRIVATE_KEY=0x...                    # EVM deployer key
SEPOLIA_SCAN_API_KEY=...                 # Etherscan API key (chain-specific prefix)
BASESEPOLIA_SCAN_API_KEY=...             # Base Sepolia scan key
```

## deployment.json Structure

The `deployment.json` file is the source of truth for the local CLI.

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

## Local Testing (Wormhole Repo)

When developing the NTT CLI locally (inside the `wormhole` repository), you must use Bun directly instead of the global `ntt` binary to ensure you are executing the local codebase.

### Running CLI logic
```bash
bun run cli/src/index.ts <command> [args]
bun run cli/src/index.ts add-chain --help
```

### Running Tests
```bash
bun test cli/src/__tests__/                  # Run all tests (from project root!)
bun run --cwd cli typecheck                  # TypeScript check
```

**Valid Testnet Chain Names:** Wormhole supports a vast array of testnets (e.g., Sepolia, BaseSepolia, ArbitrumSepolia, OptimismSepolia, Bsc, Linea, Fuji, Solana devnet, etc.). You can view the live, complete list of supported chains by running `bun run cli/src/index.ts add-chain --help`.
