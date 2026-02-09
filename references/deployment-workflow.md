# NTT EVM Deployment Workflow (Step by Step)

## Prerequisites
- Tokens already deployed on source and destination chains
- Token must implement `INttToken` interface for burning mode: `mint(address,uint256)`, `burn(uint256)`, `setMinter(address)`
- Private key with ETH for gas on both chains
- Etherscan API keys for contract verification

## Step 1: Create & Initialize Project
```bash
ntt new my-ntt-project
cd my-ntt-project
ntt init Testnet
```
Creates `deployment.json` with `{ "network": "Testnet", "chains": {} }`.

## Step 2: Set Environment Variables
```bash
export ETH_PRIVATE_KEY=0x...
export SEPOLIA_SCAN_API_KEY=...
export BASESEPOLIA_SCAN_API_KEY=...
```

## Step 3: Add First Chain
```bash
ntt add-chain Sepolia --latest --mode burning --token 0xYourTokenAddress
```
This deploys:
- NttManager proxy + implementation
- WormholeTransceiver proxy + implementation
- Configures manager with token, mode, and transceiver

## Step 4: Add Second Chain
```bash
ntt add-chain BaseSepolia --latest --mode burning --token 0xYourOtherTokenAddress
```
Same deployment on second chain.

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

## Step 7: Set Mint Authority
For burning mode, the NTT Manager needs mint permission:
```bash
# Get manager address from deployment.json
cast send $TOKEN_ADDRESS "setMinter(address)" $NTT_MANAGER_ADDRESS \
    --private-key $ETH_PRIVATE_KEY --rpc-url $RPC_URL
```
Do this on BOTH chains for burn-and-mint.

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

## Troubleshooting
- **"No protocols registered for Evm"**: Import `@wormhole-foundation/sdk-evm-ntt`
- **Verification fails**: Use `--skip-verify` flag, verify later manually
- **Rate limit stuck**: Ensure limits > 0 before any transfers
- **Decimals wrong**: Run `ntt pull` to sync decimals from on-chain
