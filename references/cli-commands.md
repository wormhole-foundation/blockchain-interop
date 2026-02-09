# NTT CLI Command Reference

## Project Setup

| Command | Description | Flags |
|---------|-------------|-------|
| `ntt new <path>` | Create new NTT project directory | |
| `ntt init <network>` | Initialize deployment.json | `Testnet` or `Mainnet` |
| `ntt update` | Update NTT CLI | `--branch <name>`, `--path <local-path>` |

## Deployment

| Command | Description | Key Flags |
|---------|-------------|-----------|
| `ntt add-chain <chain>` | Deploy NTT to a chain | `--latest`, `--mode burning\|locking`, `--token <addr>`, `--skip-verify`, `--ver <version>`, `--local` |
| `ntt push` | Push local config changes on-chain | `--payer <keypair>` (SVM), `--yes` |
| `ntt pull` | Pull on-chain config to local | |
| `ntt status` | Verify deployment matches on-chain | |
| `ntt upgrade <chain>` | Upgrade contract on chain | `--ver <version>`, `--latest` |
| `ntt clone <network> <chain> <address>` | Init from existing contract | |

## Token Operations

| Command | Description | Key Flags |
|---------|-------------|-----------|
| `ntt token-transfer` | Transfer tokens cross-chain | `--network`, `--source-chain`, `--destination-chain`, `--amount`, `--destination-address`, `--deployment-path` |
| `ntt set-mint-authority` | Set mint authority to NTT Manager | `--chain`, `--token`, `--manager`, `--payer` |
| `ntt transfer-ownership <chain>` | Transfer NTT manager ownership | `--destination <addr>` |

## Configuration

| Command | Description |
|---------|-------------|
| `ntt config set-chain <chain> <key>` | Set config value |
| `ntt config unset-chain <chain> <key>` | Remove config value |
| `ntt config get-chain <chain> <key>` | Get config value |

## Solana Subcommands

| Command | Description |
|---------|-------------|
| `ntt solana key-base58 <keypair>` | Print private key in base58 |
| `ntt solana token-authority <programId>` | Print token authority address |
| `ntt solana ata <mint> <owner> <tokenProgram>` | Print ATA address |
| `ntt solana create-spl-multisig` | Create SPL multisig |
| `ntt solana build` | Build Solana program |

## Chain Names

### Testnet
EVM: Sepolia, BaseSepolia, ArbitrumSepolia, OptimismSepolia, Holesky
SVM: Solana (uses devnet)
Sui: Sui (uses testnet)

### Mainnet
EVM: Ethereum, Base, Arbitrum, Optimism, Polygon, Avalanche, Bsc, Fantom, Celo, Moonbeam
SVM: Solana
Sui: Sui

## Environment Variables

```bash
# EVM
ETH_PRIVATE_KEY=0x...                        # Deployer private key
SEPOLIA_SCAN_API_KEY=...                     # Etherscan verification
BASESEPOLIA_SCAN_API_KEY=...                 # Base Sepolia scan
ARBITRUMSEPOLIA_SCAN_API_KEY=...             # Arbitrum Sepolia scan

# Solana
# Uses keypair files, not env vars

# Sui
SUI_PRIVATE_KEY=...
```

## RPC Overrides (overrides.json)

```json
{
  "chains": {
    "Sepolia": { "rpc": "https://custom-rpc.example.com" },
    "Solana": { "rpc": "https://custom-solana-rpc.example.com" }
  }
}
```
