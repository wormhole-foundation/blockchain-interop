# NTT Multi-Chain Deployment Workflow (EVM, SVM, Sui)

This workflow applies universally across all Wormhole-supported ecosystems. The `ntt` CLI abstracts most of the complexity, but you must branch your approach slightly depending on the network (EVM vs Solana vs Sui).

## Prerequisites

- Tokens already deployed on source and destination chains
- EVM: Token must implement `mint` / `burn` / `grantRole` or `setMinter`
- Solana (SVM): Have your program keypair and payer keypair JSON files ready
- Sui: Have your Treasury Cap object ID ready for burning mode
- Private keys funded with native gas on all target chains

## Step 1: Create & Initialize Project

> **CAUTION: `ntt new` is MANDATORY. Do NOT substitute with `mkdir`.** The `ntt new` command clones a git repository that the CLI requires internally for version resolution (the `--latest` flag runs `git tag`), project structure validation, and contract source management. If you skip `ntt new` and create the directory manually, ALL subsequent commands will fail with `fatal: not a git repository` or `Run this command from the root of an NTT project`.

```bash
ntt new my-ntt-project
cd my-ntt-project
ntt init Testnet
```

Creates `deployment.json` with `{ "network": "Testnet", "chains": {} }` inside a properly initialized NTT project directory.

## Step 2: Set Environment Variables

```bash
export ETH_PRIVATE_KEY=0x...
export SEPOLIA_SCAN_API_KEY=...
export BASESEPOLIA_SCAN_API_KEY=...
```

## Step 3: Add Chains (EVM, SVM, Sui)

The `add-chain` command syntax differs slightly based on the network's security model.

**EVM (Ethereum, Base, Arbitrum, etc.)**

```bash
ntt add-chain Sepolia --latest --mode burning --token 0xYourTokenAddress
```

**SVM (Solana)**
_Note: Solana requires explicit keypair paths instead of environment variables._

```bash
ntt add-chain Solana --latest --mode burning --token SolTokenAddress1... --payer ./payer.json
```

**Sui**
_Note: Sui burning mode requires explicitly passing the treasury cap._

```bash
ntt add-chain Sui --latest --mode burning --token 0xSuiToken... --sui-treasury-cap 0xTreasuryCapObj...
```

## Step 5: Configure Rate Limits

Edit `deployment.json` to set limits:

```json
"limits": {
    "outbound": "1000.000000000000000000",
    "inbound": {
        "BaseSepolia": "500.000000000000000000"
    }
}
```

Use 18 decimal places for EVM chains.

## Step 6: Push Configuration

```bash
ntt push
```

This:

- Sets peer relationships between chains
- Configures rate limits
- Sets transceiver peers
- Registers transceivers with managers

## Step 6: Configure Manager Liquidity / Permissions

The NTT Manager must have access to tokens before it can process inbound transfers. The setup depends entirely on your `--mode`:

### Option A: Burning Mode (Burn-and-Mint)

The Manager must be granted permission to mint the Host Token. This process is highly chain-specific:

**EVM (Manual Contract Call):**
The CLI cannot do this for EVM. You must use Foundry or Hardhat.

```bash
# Get manager address from deployment.json
cast send $TOKEN_ADDRESS "setMinter(address)" $NTT_MANAGER_ADDRESS \
    --private-key $ETH_PRIVATE_KEY --rpc-url $RPC_URL
```

**SVM (Solana CLI Native):**
Wormhole provides a built-in CLI command for Solana SPL tokens.

```bash
ntt set-mint-authority --chain Solana --token SolToken1... --manager SolManager1... --payer ./payer.json
```

**Sui:**
Burn authority is handled automatically during `add-chain` if you correctly pass the `--sui-treasury-cap`.

### Option B: Locking Mode (Hub-and-Spoke)

If a chain is deployed in `locking` mode, the Manager does not mint tokens. Instead, it unlocks existing tokens from a liquidity pool. Before a locking chain can receive a transfer, you must seed the Manager contract with actual token liquidity.

```bash
# Transfer tokens to the NttManager address to seed initial inbound liquidity
cast send $TOKEN_ADDRESS "transfer(address,uint256)" $NTT_MANAGER_ADDRESS 1000000000000000000000 \
    --private-key $ETH_PRIVATE_KEY --rpc-url $RPC_URL
```

## Step 8: Verify Deployment

```bash
ntt status
```

Checks that on-chain state matches local deployment.json.

## Step 9: Test Transfer

```bash
ntt token-transfer \
    --network Testnet \
    --source-chain Sepolia \
    --destination-chain BaseSepolia \
    --amount 0.1 \
    --destination-address 0xRecipient \
    --deployment-path ./deployment.json
```

## Hub-and-Spoke Variant

If using hub-and-spoke instead of burn-and-mint:

- Hub chain: `--mode locking` (no mint authority needed, standard ERC-20 ok)
- Spoke chains: `--mode burning` (needs INttToken + mint authority)

## Gas Token (ETH/Native) — WethUnwrap Variant

To bridge the native gas token (e.g. ETH), use WETH as the locked token on the hub chain and deploy with `--manager-variant wethUnwrap`. When tokens arrive back at the hub, the manager automatically unwraps WETH → ETH and sends native ETH to the recipient.

**When to use:** Hub chain only (locking mode). Token must be the WETH contract address on that chain.

**Step 1: Add the hub chain with wethUnwrap variant**
```bash
# Ethereum Mainnet WETH: 0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2
ntt add-chain Ethereum --latest --mode locking \
  --token <WETH_ADDRESS> \
  --manager-variant wethUnwrap
```

This stores `"managerVariant": "wethUnwrap"` in `deployment.json`. On `ntt upgrade`, the variant is read automatically — no need to re-pass the flag.

**Step 2: Add spoke chains normally (burning mode)**
```bash
ntt add-chain BaseSepolia --latest --mode burning --token <WRAPPED_ETH_TOKEN>
```
Spoke chains use their own wrapped/synthetic ETH token. Grant mint authority to the spoke NttManager as usual.

**How it works internally:**
- Standard NttManager: releases WETH tokens to recipient on unlock
- WethUnwrap: calls `weth.withdraw(amount)`, then sends native ETH via `payable(recipient).call{value: amount}`
- The contract has `receive() external payable` to accept ETH from WETH's `withdraw()` callback
- `NttManagerWethUnwrap` constructor casts `token` to `IWETH` — so the token **must** be the WETH contract

**Requirements:**
- EVM only (no Solana/Sui equivalent)
- Requires deploy script version 2+ (all current NTT versions)
- Locking mode only — burning mode never calls `_unlockTokens`
- Token must implement the WETH interface (`withdraw(uint256)`)

**Common WETH addresses:**
- Ethereum Mainnet: `0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2`
- Base: `0x4200000000000000000000000000000000000006`
- Arbitrum One: `0x82aF49447D8a07e3bd95BD0d56f35241523fBab1`
- Sepolia (testnet): `0xfFf9976782d46CC05630D1f6eBAb18b2324d6B14`

## Handling On-Chain Deployment Failures

Deploying to testnets and mainnets can be flaky. If `ntt push` or `ntt add-chain` hangs or crashes midway:

- **RPC Timeouts:** Public RPCs (like Ankr or public Infura) aggressively rate-limit deployment scripts. If the CLI hangs, switch to a dedicated API key in your `.env` (e.g., Alchemy, QuickNode) and use `overrides.json` to force the CLI to use it.
- **Gas Estimation Failures:** Testnets experience sudden gas spikes. Do NOT use standard `--gas-price` flags. The Wormhole NTT CLI relies on `ethers.js` dynamic fee estimation. If transactions stall, append `--gas-estimate-multiplier <NUMBER>` (e.g., `1.5`) to force the CLI to overpay for gas.
- **Nonce Conflicts & Stuck TXs:** The CLI uses `NonceManager.increment()` which can desync if a transaction drops. You _must_ manually cancel the stuck TX in MetaMask or wait for network drops before retrying `ntt push`.
- **Partial Deployments:** If a deployment fails halfway through, **do not immediately restart**. Run `ntt status` to see what actually landed on-chain. You might need to manually intervene or cleanly delete the pending state from `.deployments/` before retrying.

## Contract Verification

### NTT CLI Per-Chain Verifier Configuration

The CLI supports per-chain verifier settings via `ntt config set-chain`. This controls how `ntt add-chain` verifies contracts during deployment. Config keys:

- `verifier` — verifier type: `etherscan` (default), `sourcify`, or `blockscout`
- `verifier_url` — custom verifier API URL (required for `sourcify` and `blockscout`)
- `scan_api_key` — Etherscan API key (also read from `<CHAIN>_SCAN_API_KEY` env var)

```bash
ntt config set-chain <Chain> verifier <type>
ntt config set-chain <Chain> verifier_url <url>
```

Well-known chains (Ethereum, Base, Arbitrum, Optimism, BSC) work with the default `etherscan` verifier — just set the `<CHAIN>_SCAN_API_KEY` env var.

### Chains Requiring Custom Verifier Config

**Monad** (chain ID 143) — use Sourcify via BlockVision:
```bash
ntt config set-chain Monad verifier sourcify
ntt config set-chain Monad verifier_url https://sourcify-api-monad.blockvision.org/
```

**MegaETH** (chain ID 4326) — use Etherscan v2 with explicit URL:
```bash
ntt config set-chain MegaETH verifier etherscan
ntt config set-chain MegaETH verifier_url "https://api.etherscan.io/v2/api?chainid=4326"
```

**HyperEVM** (chain ID 999) — use Sourcify:
```bash
ntt config set-chain HyperEVM verifier sourcify
ntt config set-chain HyperEVM verifier_url https://sourcify.dev/server/
```

Set these **before** running `ntt add-chain` so verification happens automatically during deployment.

### Manual Verification (Etherscan v2)

The old Etherscan v1 API endpoints are deprecated. All EVM contract verification must use the **Etherscan v2 API**.

**Do NOT use `--verify` inline with `forge create`** — it uses the v1 endpoint and will fail with "deprecated V1 endpoint" error. Instead, deploy first, then verify separately:

```bash
# Deploy (no --verify flag)
forge create --broadcast --rpc-url $RPC_URL --private-key $ETH_PRIVATE_KEY \
  <contract_path>:<ContractName>

# Verify separately using Etherscan v2
forge verify-contract <DEPLOYED_ADDRESS> <contract_path>:<ContractName> \
  --verifier etherscan \
  --verifier-url "https://api.etherscan.io/v2/api?chainid=<CHAIN_ID>" \
  --etherscan-api-key $SCAN_API_KEY \
  --watch
```

### Manual Verification (Sourcify — e.g., Monad)

```bash
forge verify-contract <DEPLOYED_ADDRESS> <contract_path>:<ContractName> \
  --chain <EVM_CHAIN_ID> \
  --verifier sourcify \
  --verifier-url https://sourcify-api-monad.blockvision.org/
```

No API key needed for Sourcify. No `--constructor-args` needed — Sourcify resolves them automatically.

Common chain IDs: Sepolia=11155111, BaseSepolia=84532, HyperEVM=999, Ethereum=1, Base=8453, Monad=143, MegaETH=4326.

For contracts with constructor args (Etherscan only), add:
```bash
  --constructor-args $(cast abi-encode "constructor(type1,type2,...)" arg1 arg2 ...)
```

For contracts with linked libraries, add:
```bash
  --libraries src/libraries/Lib.sol:Lib:<LIB_ADDR>
```

## Troubleshooting

- **"No protocols registered for Evm"**: Import `@wormhole-foundation/sdk-evm-ntt`
- **"deprecated V1 endpoint"**: Use separate `forge verify-contract` with `--verifier etherscan --verifier-url "https://api.etherscan.io/v2/api?chainid=<ID>"`
- **"No known Etherscan API URL for chain X"**: Chain not in forge's built-in registry. Use `--verifier-url` explicitly, or configure via `ntt config set-chain`
- **Verification fails**: Use `--skip-verify` flag on `ntt add-chain`, verify later manually with `forge verify-contract`
- **Rate limit stuck**: Ensure limits > 0 before any transfers
- **Decimals wrong**: Run `ntt pull` to sync decimals from on-chain

## Step 10: Implement (Frontend/SDK)

Once deployment and CLI transfers are verified, transition to the **Implement** phase of the E2E lifecycle:

1. Extract your deployed `token` and `manager` addresses from the generated `deployment.json`.
2. Review the **Product Ecosystem Overview** in `SKILL.md` to select your interface (Connect UI widget vs TypeScript SDK).
3. Ingest the corresponding dynamic `llms.txt` file (e.g., `llms-connect.txt`) and pass your `deployment.json` addresses into the frontend configuration to complete the integration.
