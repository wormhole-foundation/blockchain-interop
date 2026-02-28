# Wormhole NTT Troubleshooting & Gotchas

This document contains detailed troubleshooting steps, edge cases, and manual recovery workflows for when the standard NTT CLI commands fail.

## Common Gotchas

1. **Token Minting Permissions:** The token only needs `mint(address,uint256)` and `burn(uint256)` for burning mode. NTT never calls `setMinter` — permission is granted by the token owner via `grantRole` (AccessControl) or `setMinter` (Ownable), depending on the token's pattern.
2. **`ntt pull` is Mandatory:** `ntt pull` must be run to configure decimals in deployment.json. If EVM and SVM/Solana deployments are done in separate directories, the local `deployment.json` files will be unaware of sibling chains. Emphasize running `ntt pull --yes` before cross-chain transfers (`ntt token-transfer`).
3. **Scan Keys:** Scan API key env var format must be strictly `<CHAIN_NAME>_SCAN_API_KEY` (e.g., `SEPOLIA_SCAN_API_KEY`, `BASESEPOLIA_SCAN_API_KEY`).
4. **Rate Limit Decimals:** Rate limit decimals are 18 for EVM, 9 for SVM/Sui. If limits are incorrectly synced or set to 0, transactions will silently queue or revert. Always set > 0.
5. **Real Testnets:** Wormhole Testnet uses real testnets (Sepolia, Base Sepolia, etc.), not local forks.
6. **Set Mint Authority CLI:** The `set-mint-authority` CLI command is SVM-only. For EVM, grant permission manually via `cast`.
7. **Bun Environment:** Bun auto-loads `.env` from cwd — never manually export private keys. Keep them in `.env` next to `deployment.json`.
8. **Broken Git Worktrees:** If `ntt upgrade <Chain>` fails with `fatal: not a git repository`, manually remove the corrupted hidden worktree (`rm -rf .deployments/<Version>`) so the CLI can cleanly recreate it.
9. **Wallet Ownership Mismatch:** Operations like `ntt manual set-peer` will fail with authorization or gas errors if the loaded `ETH_PRIVATE_KEY` does not match the manager's current owner or lacks native gas tokens. Verify with `cast wallet address`.
10. **Locking Mode Allowance:** When `mode` is `locking`, the sender's wallet MUST call `approve()` on the Token contract to grant the `NttManager` allowance before a transfer can succeed.
11. **Locking Mode Liquidity:** If a transaction succeeds on the source chain but reverts on the destination chain during a `locking` transfer, verify the destination `NttManager` contract actually holds a sufficient balance of the token to release to the receiver.

## VAA Finality Delays

Seeing `wh_guardian_vaa_not_found` on Wormholescan is **expected behavior** and not an error. Wormhole Guardians wait for Safe Finality on the source chain before generating the signed VAA message (e.g., ~15-20 minutes on Sepolia/Ethereum). The CLI polls automatically and will succeed once finalized. Do not interrupt the polling process.

## Manual VAA Claim (Failed Transfers)

When a transfer fails on the destination chain (e.g., due to insufficient un-locked tokens or rate limits in the destination `NttManager`), the VAA is not consumed and the transfer can be retried manually once the issue is resolved.

The claim process requires passing the raw VAA bytes to the `WormholeTransceiver` contract on the **destination** chain.

### Approach A: CLI / Bash

1.  **Retrieve VAA ID:** Get the VAA identifier (`chainId/emitterAddress/sequenceNumber`) from the source transaction via Wormholescan.
2.  **Fetch Raw VAA Bytes:** The Transceiver smart contract requires the raw, signed VAA hex bytes, not the identifier string.

```bash
# Get your VAA_ID (format: chainId/emitterAddress/sequenceNumber) from Wormholescan
# Example format: "10002/000000000000000000000000.../5"
VAA_ID="<CHAIN_ID>/<EMITTER_ADDRESS>/<SEQUENCE_NUMBER>"

# Set the API endpoint based on the environment
WORMHOLE_API="https://api.testnet.wormholescan.io" # Use api.wormholescan.io for Mainnet

# Fetch from API, extract base64, decode, and convert to hex
RAW_VAA_HEXBYTES="0x$(curl -s "$WORMHOLE_API/api/v1/vaas/$VAA_ID" | jq -r '.data.vaa' | base64 -d | xxd -p | tr -d '\n')"
```

3.  **Get Transceiver Address:** Find the destination chain's `WormholeTransceiver` address in `deployment.json` (under `[destinationChain].transceivers.wormhole.address`).
4.  **Execute Claim:** Call `receiveMessage` on the destination Transceiver:

```bash
# Using cast (Foundry)
cast send $DESTINATION_TRANSCEIVER "receiveMessage(bytes)" $RAW_VAA_HEXBYTES \
    --private-key $ETH_PRIVATE_KEY --rpc-url $DESTINATION_RPC_URL
```

*Note: The Transceiver will verify the VAA with the Core Bridge and forward it to the NttManager to complete the token mint/unlock.*

### Approach B: Frontend SDK (TypeScript)

If you are building a dApp or need to programmatically claim the VAA using the `@wormhole-foundation/sdk`, use the `completeTransfer` method which abstracts the VAA fetching and contract calls.

```typescript
import { wormhole, signSendWait } from "@wormhole-foundation/sdk";
import evmLoader from "@wormhole-foundation/sdk/evm";
import "@wormhole-foundation/sdk-evm-ntt"; // Register the NTT plugin

async function manuallyClaimVAA() {
  // Initialize Wormhole for Testnet, loading EVM protocol support
  const wh = await wormhole("Testnet", [evmLoader]);

  // 1. Setup destination signer
  const dstChain = wh.getChain("BaseSepolia"); // Replace with your target chain
  
  // NOTE: You must provide your own destination chain Signer implementation
  // const destSigner = await getSigner(dstChain); 

  // 2. Fetch the transfer details using the exact TxHash from the source chain
  const sourceTxHash = "0xYourSourceTransactionHash...";
  
  // Fetch the VAA directly from the network
  // Native Token Transfers use the "Ntt:WormholeTransfer" payload type
  console.log("Fetching VAA from Guardian Network...");
  const vaa = await wh.getVaa(
    sourceTxHash,
    "Ntt:WormholeTransfer",
    15 * 60 * 1000 // 15 minute timeout for finality
  );

  // 3. Initialize the NTT Protocol on the destination chain
  const dstNtt = await dstChain.getProtocol("Ntt", {
    ntt: {
      manager: "0xYourNttManagerAddress...", // From deployment.json
      token: "0xYourTokenAddress...",        // From deployment.json
    },
  });

  // 4. Complete the transfer on the destination chain
  console.log("Submitting VAA to destination contract...");
  const txreceipt = await signSendWait(
    dstChain,
    dstNtt.redeem([vaa!], destSigner.address.address),
    destSigner.signer
  );
  console.log("Claim Successful:", txreceipt);
}

manuallyClaimVAA().catch(console.error);
```

## SDK Implementation Gotchas (TypeScript)

When building a full end-to-end integration using the Wormhole `@wormhole-foundation/sdk`, developers frequently encounter these implementation hurdles:

1. **Layout Sizing Mismatches:** When defining custom layouts for cross-chain message payloads, the `size` attribute must precisely match the byte length of your data. If you define `{ binary: 'bytes', size: 32 }` but pass 31 bytes, serialization will fail silently or corrupt data.
2. **Missing Try-Catch on Deserialization:** Always wrap `deserializeLayout` in a `try/catch`. If a maliciously formatted or truncated message arrives on the destination chain, it will crash the relayer/frontend.
3. **VAA Polling Rate Limits:** When using `wh.getVaa(msgId)` or other modern SDK polling methods, you can hit rate limits (`fetch error 429`) against the Guardian RPCs if you poll too aggressively or if the network is congested. You must wrap your fetching logic in an exponential backoff retry loop to handle `429 Too Many Requests` elegantly, rather than letting the process crash.
