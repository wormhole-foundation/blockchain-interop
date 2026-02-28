# NTT E2E Testing Guide

## Anvil-Based E2E Test (Fast, Local)

The primary E2E test is in `cli/e2e/e2e-anvil.test.ts`. It runs in ~72s.

### How it works

1. Starts two Anvil instances forking Sepolia (port 18545) and Base Sepolia (port 18546)
2. Uses `--auto-impersonate` so any address can send txs without a private key
3. Sets Wormhole core bridge message fee via `anvil_setStorageAt` (slot 7)
4. Impersonates the token admin to grant roles to Anvil's default account and NTT managers
5. Creates NTT project with `overrides.json` pointing at local Anvil RPCs
6. Deploys NTT to both forks, pulls, sets limits, pushes, verifies status

### Running

```bash
bun test cli/e2e/e2e-anvil.test.ts --timeout 600000
```

### Key Anvil RPC methods used

- `anvil_impersonateAccount` / `anvil_stopImpersonatingAccount` — send txs as any address
- `anvil_setStorageAt` — override contract storage (e.g., Wormhole message fee)
- `evm_setAutomine` — mine a block per tx (fast execution)

### overrides.json pattern

```json
{
    "chains": {
        "Sepolia": { "rpc": "http://127.0.0.1:18545" },
        "BaseSepolia": { "rpc": "http://127.0.0.1:18546" }
    }
}
```

### Custom Finality (CCL)

On Sepolia, deploy with `--unsafe-custom-finality "200:1"` (instant + 1 block).
Must pipe "yes\n" to stdin to confirm the CCL warning prompt.
CCL contract addresses are only configured for Sepolia and Linea on Testnet.
Base Sepolia does NOT have a CCL contract — deploy without custom finality there.

### Manual Relay (for future cross-chain transfer tests)

NTT's `WormholeTransceiver.receiveMessage(bytes encodedVAA)` is external and permissionless.
To manually relay a VAA on the destination chain:

```bash
cast send $DEST_TRANSCEIVER "receiveMessage(bytes)" $ENCODED_VAA \
    --private-key $KEY --rpc-url $DEST_RPC
```

On Anvil forks there's no real Wormhole Guardian network, so VAAs must be manually constructed or the guardian set must be overridden.

### Wormhole Core Bridge Addresses (Testnet)

| Chain        | Core Bridge                                  |
| ------------ | -------------------------------------------- |
| Sepolia      | `0x4a8bc80Ed5a4067f1CCf107057b8270E0cC11A78` |
| Base Sepolia | `0x79A1027a6A159502049F10906D333EC57E95F083` |

---

## Test Tokens (Testnet)

| Chain        | Address                                      |
| ------------ | -------------------------------------------- |
| Sepolia      | `0x771e6eD6057E6da9BA7f88f82833dF52B3Eb947A` |
| Base Sepolia | `0xb270c9F2cD63815e2c3a277Deeb0A35F514672Be` |

## E2E Test Workflow

### Phase 1: Smoke Tests (No On-Chain)

1. Verify CLI help output works
2. Verify `ntt init Testnet` creates valid deployment.json
3. Verify `ntt new` creates project directory
4. Verify all commands accept `--help` flag

### Phase 2: Deployment Test (On-Chain, Testnet)

1. Create fresh NTT project in temp directory
2. Initialize for Testnet
3. Add Sepolia chain with the token in burning mode
4. Add Base Sepolia chain with the token in burning mode
5. Push configuration
6. Set mint authority on both tokens
7. Run `ntt status` to verify

### Phase 3: Transfer Test

1. Mint test tokens if needed:
   ```bash
   cast send $TOKEN_ADDRESS "mint(address,uint256)" $USER_ADDRESS 1000000000000000000 \
       --private-key $ETH_PRIVATE_KEY --rpc-url $RPC_URL
   ```
2. Execute token-transfer from Sepolia to Base Sepolia
3. Verify transfer completes (may take several minutes for Guardian attestation)
4. Verify token balance on destination chain:
   ```bash
   cast call $TOKEN_ADDRESS "balanceOf(address)(uint256)" $USER_ADDRESS --rpc-url $RPC_URL
   ```

### Phase 4: Frontend SDK Test (Implement Protocol)

The final step of the End-to-End lifecycle is ensuring the newly deployed protocol interacts correctly with the frontend.
1. Write a small TypeScript script using the `@wormhole-foundation/sdk` to perform a `TokenTransfer`.
2. Monitor the Guardian RPC polling mechanisms in your frontend logs to ensure you are not hitting `429 Rate Limit` errors.
3. Verify that any VAA parsing or manual UI claim logic respects the layout serialization limits discussed in the Troubleshooting guide.

## Local CLI Invocation

When testing from this repo, use `bun` directly instead of the global `ntt` command:

```bash
# Instead of: ntt add-chain Sepolia --latest --mode burning --token 0x...
# Use:
bun run cli/src/index.ts add-chain Sepolia --latest --mode burning --token 0x...
```

## Setting Up Test Environment

```bash
# Required env vars
export ETH_PRIVATE_KEY=0x...
export SEPOLIA_SCAN_API_KEY=...
export BASESEPOLIA_SCAN_API_KEY=...

# Create test project
TESTDIR=$(mktemp -d)
cd $TESTDIR
bun run /path/to/cli/src/index.ts new test-deployment
cd test-deployment
bun run /path/to/cli/src/index.ts init Testnet
```

## Granting Mint Permission

### AccessControl tokens

Some tokens use OpenZeppelin AccessControl. The NTT Manager needs `MINTER_ROLE` to mint tokens in burning mode.

```bash
# MINTER_ROLE = keccak256("MINTER_ROLE") = 0x9f2df0fed2c77648de5860a4cc508cd0818c85b8b8a1ab4ceeef8d981c8956a6
MINTER_ROLE=$(cast call $TOKEN_ADDRESS "MINTER_ROLE()(bytes32)" --rpc-url $RPC_URL)

# Grant MINTER_ROLE to the NTT Manager (get manager address from deployment.json)
cast send $TOKEN_ADDRESS "grantRole(bytes32,address)" $MINTER_ROLE $NTT_MANAGER_ADDRESS \
    --private-key $ETH_PRIVATE_KEY --rpc-url $RPC_URL

# Verify
cast call $TOKEN_ADDRESS "hasRole(bytes32,address)(bool)" $MINTER_ROLE $NTT_MANAGER_ADDRESS --rpc-url $RPC_URL
```

burn(uint256) is from ERC20Burnable — permissionless for own tokens (no role needed).

### INttToken / setMinter tokens

```bash
cast send $TOKEN_ADDRESS "setMinter(address)" $NTT_MANAGER_ADDRESS \
    --private-key $ETH_PRIVATE_KEY --rpc-url $RPC_URL
```

## Verifying Token Interface

Before deployment, verify the token supports required functions:

```bash
# Check if token has mint function
cast call $TOKEN_ADDRESS "mint(address,uint256)" $ZERO_ADDR 0 --rpc-url $RPC_URL

# Check if token has burn function
cast call $TOKEN_ADDRESS "burn(uint256)" 0 --rpc-url $RPC_URL

# Check decimals
cast call $TOKEN_ADDRESS "decimals()(uint8)" --rpc-url $RPC_URL

# Check name/symbol
cast call $TOKEN_ADDRESS "name()(string)" --rpc-url $RPC_URL
cast call $TOKEN_ADDRESS "symbol()(string)" --rpc-url $RPC_URL
```
