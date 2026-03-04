# Wormhole Guardian Governance Ceremonies

## Overview

When NTT contracts are owned by a Wormhole governance contract, all admin operations (upgrades, peer registration, etc.) must go through a guardian governance ceremony. Guardians sign a VAA (Verifiable Action Approval) that the governance contract then executes.

## Tools

- **guardiand**: Generates governance prototxt templates. Available via Docker:
  ```bash
  docker run --rm ghcr.io/wormhole-foundation/guardiand:latest template <subcommand> [flags]
  ```
- **cast**: Used to generate ABI-encoded call data (`cast calldata`)

## General Flow

All EVM governance operations follow the same pattern:

```bash
# 1. Generate the ABI-encoded call data for the target function
CALL_DATA=$(cast calldata "<function_signature>" <args...>)

# 2. Generate governance prototxt using guardiand
docker run --rm ghcr.io/wormhole-foundation/guardiand:latest template governance-evm-call \
  --chain-id <WORMHOLE_CHAIN_ID> \
  --governance-contract <GOVERNANCE_CONTRACT> \
  --target-address <TARGET_CONTRACT> \
  --call-data "$CALL_DATA" \
  > governance-<operation>.prototxt

# 3. Distribute prototxt to guardians for signing
# 4. Collect signatures and submit VAA on-chain
```

**Important:** `--chain-id` is the **Wormhole chain ID**, not the EVM chain ID.

## guardiand Flags

| Flag | Description |
|------|-------------|
| `--chain-id` | Wormhole chain ID of the target chain |
| `--governance-contract` | Address of the governance/owner contract |
| `--target-address` | Address of the contract being called |
| `--call-data` | ABI-encoded function call |
| `--idx` | Guardian set index (default: 4) |

The tool auto-generates `current_set_index`, `sequence`, and `nonce`.

## Common Wormhole Chain IDs

| Chain | Wormhole ID | EVM Chain ID |
|-------|------------|--------------|
| Ethereum | 2 | 1 |
| Base | 30 | 8453 |
| HyperEVM | 47 | 999 |
| Solana | 1 | N/A |
| Unichain | 44 | 130 |

## Common NTT Governance Operations

### 1. Upgrade NttManager Implementation

```bash
CALL_DATA=$(cast calldata "upgrade(address)" <NEW_IMPL_ADDR>)

docker run --rm ghcr.io/wormhole-foundation/guardiand:latest template governance-evm-call \
  --chain-id <WORMHOLE_CHAIN_ID> \
  --governance-contract <GOVERNANCE_CONTRACT> \
  --target-address <NTT_MANAGER_PROXY> \
  --call-data "$CALL_DATA" \
  > governance-upgrade-manager.prototxt
```

### 2. Register NttManager Peer (New Chain)

Registers a peer NttManager on another chain. Required when adding a new chain to an existing NTT deployment.

```bash
# setPeer(uint16 peerChainId, bytes32 peerContract, uint8 decimals, uint256 inboundLimit)
#
# peerChainId:  Wormhole chain ID of the peer chain
# peerContract: Peer NttManager address as bytes32 (left-padded with zeros)
# decimals:     Token decimals on the peer chain (e.g., 18 for EVM, 9 for Solana)
# inboundLimit: Inbound rate limit in raw token units (18 decimals for EVM)

# Convert an EVM address to bytes32:
PEER_BYTES32=$(cast --to-bytes32 <PEER_MANAGER_ADDRESS>)

CALL_DATA=$(cast calldata "setPeer(uint16,bytes32,uint8,uint256)" \
  <PEER_WORMHOLE_CHAIN_ID> \
  $PEER_BYTES32 \
  <PEER_TOKEN_DECIMALS> \
  <INBOUND_LIMIT>)

docker run --rm ghcr.io/wormhole-foundation/guardiand:latest template governance-evm-call \
  --chain-id <THIS_WORMHOLE_CHAIN_ID> \
  --governance-contract <GOVERNANCE_CONTRACT> \
  --target-address <NTT_MANAGER_PROXY> \
  --call-data "$CALL_DATA" \
  > governance-set-peer.prototxt
```

### 3. Register WormholeTransceiver Peer (New Chain)

Must be done alongside the NttManager peer registration.

```bash
# setWormholePeer(uint16 peerChainId, bytes32 peerContract)
PEER_BYTES32=$(cast --to-bytes32 <PEER_TRANSCEIVER_ADDRESS>)

CALL_DATA=$(cast calldata "setWormholePeer(uint16,bytes32)" \
  <PEER_WORMHOLE_CHAIN_ID> \
  $PEER_BYTES32)

docker run --rm ghcr.io/wormhole-foundation/guardiand:latest template governance-evm-call \
  --chain-id <THIS_WORMHOLE_CHAIN_ID> \
  --governance-contract <GOVERNANCE_CONTRACT> \
  --target-address <WORMHOLE_TRANSCEIVER> \
  --call-data "$CALL_DATA" \
  > governance-set-transceiver-peer.prototxt
```

### 4. Upgrade WormholeTransceiver Implementation

```bash
CALL_DATA=$(cast calldata "upgrade(address)" <NEW_TRANSCEIVER_IMPL>)

docker run --rm ghcr.io/wormhole-foundation/guardiand:latest template governance-evm-call \
  --chain-id <WORMHOLE_CHAIN_ID> \
  --governance-contract <GOVERNANCE_CONTRACT> \
  --target-address <WORMHOLE_TRANSCEIVER_PROXY> \
  --call-data "$CALL_DATA" \
  > governance-upgrade-transceiver.prototxt
```

### 5. Register New Transceiver with Manager

```bash
# setTransceiver(address transceiver)
CALL_DATA=$(cast calldata "setTransceiver(address)" <NEW_TRANSCEIVER_ADDRESS>)

docker run --rm ghcr.io/wormhole-foundation/guardiand:latest template governance-evm-call \
  --chain-id <WORMHOLE_CHAIN_ID> \
  --governance-contract <GOVERNANCE_CONTRACT> \
  --target-address <NTT_MANAGER_PROXY> \
  --call-data "$CALL_DATA" \
  > governance-set-transceiver.prototxt
```

## Adding a New Chain (Full Peer Registration Ceremony)

When adding a new chain to an existing governed NTT deployment, you need governance messages for **both directions** on the governed chain:

1. **On the governed chain** (e.g., HyperEVM — requires governance):
   - `setPeer` on NttManager → register the new chain's manager
   - `setWormholePeer` on WormholeTransceiver → register the new chain's transceiver

2. **On the new chain** (if owner-controlled — can use direct `cast send`):
   - `setPeer` on NttManager → register the governed chain's manager
   - `setWormholePeer` on WormholeTransceiver → register the governed chain's transceiver

Or use `ntt push` if the new chain's owner is an EOA (the CLI handles peer setup automatically for non-governed chains).

## Output Format

The generated prototxt looks like:
```
current_set_index: 4
messages: {
  sequence: <auto-generated>
  nonce: <auto-generated>
  evm_call: {
    chain_id: <wormhole_chain_id>
    governance_contract: "<governance_address>"
    target_contract: "<target_address>"
    abi_encoded_call: "<hex_calldata>"
  }
}
```

## Tips

- Always verify `CALL_DATA` before generating the governance message: `cast calldata --help`
- Use `cast --to-bytes32` to convert EVM addresses to bytes32 for peer registration
- For Solana peers, the bytes32 is already in the right format (base58 program ID → bytes32)
- Test governance calls on an anvil fork first by impersonating the governance contract
- Multiple operations can each be separate prototxt files — guardians sign them individually
