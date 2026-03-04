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

## Handling On-Chain Deployment Failures

Deploying to testnets and mainnets can be flaky. If `ntt push` or `ntt add-chain` hangs or crashes midway:

- **RPC Timeouts:** Public RPCs (like Ankr or public Infura) aggressively rate-limit deployment scripts. If the CLI hangs, switch to a dedicated API key in your `.env` (e.g., Alchemy, QuickNode) and use `overrides.json` to force the CLI to use it.
- **Gas Estimation Failures:** Testnets experience sudden gas spikes. Do NOT use standard `--gas-price` flags. The Wormhole NTT CLI relies on `ethers.js` dynamic fee estimation. If transactions stall, append `--gas-estimate-multiplier <NUMBER>` (e.g., `1.5`) to force the CLI to overpay for gas.
- **Nonce Conflicts & Stuck TXs:** The CLI uses `NonceManager.increment()` which can desync if a transaction drops. You _must_ manually cancel the stuck TX in MetaMask or wait for network drops before retrying `ntt push`.
- **Partial Deployments:** If a deployment fails halfway through, **do not immediately restart**. Run `ntt status` to see what actually landed on-chain. You might need to manually intervene or cleanly delete the pending state from `.deployments/` before retrying.

## Contract Verification (Etherscan v2)

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

Common chain IDs: Sepolia=11155111, BaseSepolia=84532, HyperEVM=999, Ethereum=1, Base=8453.

For contracts with constructor args, add:
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
- **Verification fails**: Use `--skip-verify` flag on `ntt add-chain`, verify later manually with `forge verify-contract`
- **Rate limit stuck**: Ensure limits > 0 before any transfers
- **Decimals wrong**: Run `ntt pull` to sync decimals from on-chain

## Step 10: Implement (Frontend/SDK)

Once deployment and CLI transfers are verified, transition to the **Implement** phase of the E2E lifecycle:

1. Extract your deployed `token` and `manager` addresses from the generated `deployment.json`.
2. Review the **Product Ecosystem Overview** in `SKILL.md` to select your interface (Connect UI widget vs TypeScript SDK).
3. Ingest the corresponding dynamic `llms.txt` file (e.g., `llms-connect.txt`) and pass your `deployment.json` addresses into the frontend configuration to complete the integration.
