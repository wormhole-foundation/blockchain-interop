# HyperEVM NTT Reference

## Big Block Mode
HyperEVM requires "big blocks" enabled for deploying large contracts (like NttManager with `--via-ir`). Use the NTT CLI:
```bash
ntt hype set-big-blocks            # Enable (reads network from deployment.json)
ntt hype set-big-blocks --disable  # Disable after deployment
```
Check status: `cast rpc eth_usingBigBlocks <your_address> --rpc-url https://rpc.hyperliquid.xyz/evm`

## Contract Verification on HyperEVM
HyperEVM uses hyperevmscan.io (powered by Etherscan, chain ID 999).
See `deployment-workflow.md` for the Etherscan v2 verification flow (applies to all EVMs).
Env var: `ETHEREUM_SCAN_API_KEY` (not chain-prefixed for HyperEVM).

## NttManagerWethUnwrap Upgrades (Proxy Pattern)
For upgrading the NttManager implementation behind a proxy:
1. Deploy new TransceiverStructs library (linked dependency)
2. Deploy NttManagerWethUnwrap with `--via-ir` and `--libraries` flag linking TransceiverStructs
3. Constructor args: `<token> <mode> <wormhole_chain_id> <rate_limit_duration> <skip_rate_limiting>`
   - HyperEVM mainnet: `0x5555555555555555555555555555555555555555 0 47 86400 false`
4. Upgrade proxy via guardian governance (`upgrade(address)` call)
5. Verify EIP-1967 implementation slot: `cast storage <proxy> 0x360894A13BA1A3210667C828492DB98DCA3E2076CC3735A920A3CA505D382BBC`

### forge create gotcha
`forge create` requires `--broadcast` to actually send the transaction. Without it, it does a dry run and prints the ABI. Put `--broadcast` early in the flags.

## HyperEVM NTT CLI Utilities
```bash
ntt hype set-big-blocks       # Enable/disable big blocks
ntt hype link                 # Link HyperCore spot token to ERC-20
ntt hype bridge-in <amount>   # Bridge ERC-20 to HyperCore
ntt hype bridge-out <amount>  # Bridge HyperCore to HyperEVM
ntt hype status               # Show HyperCore token status
```

## Key Addresses (Mainnet)
- RPC: `https://rpc.hyperliquid.xyz/evm`
- Chain ID: `999`
- Wormhole Chain ID: `47`
- Explorer: `https://hyperevmscan.io`
- Explorer API: `https://api.hyperevmscan.io/api`
- NTT Manager proxy: `0x452A2613f9200dB250Ec30d15643f4535934Ea77`
- Owner / Governance: `0x574B7864119C9223A9870Ea614dC91A8EE09E512`
- WETH (native token): `0x5555555555555555555555555555555555555555`
- WormholeTransceiver: `0x187558a6Cc56e1a0f0DF7a2760aB765297E6cef2`
