# Wormhole Connect Integration Reference

This file covers setting up Wormhole Connect (the drop-in React bridge widget) with NTT routes for custom token transfers.

## Package Installation

```bash
npm install @wormhole-foundation/wormhole-connect
```

## NTT Route Configuration

The `nttRoutes` function is exported from a **subpath**, not the main package entry:

```typescript
// ✅ Correct — use the /ntt subpath
import { nttRoutes } from '@wormhole-foundation/wormhole-connect/ntt';

// ❌ Wrong — this does not export nttRoutes
import { nttRoutes } from '@wormhole-foundation/wormhole-connect';
```

This is the most common Connect + NTT integration error. The `/ntt` subpath exists because NTT route logic is tree-shakeable — it's only loaded when explicitly imported.

## Minimal NTT Config Example

A complete working config requires three pieces:

1. `nttManualRoute(config)` + `nttExecutorRoute({ ntt: config })` in the `routes` array
2. Matching entries in `tokensConfig` for each chain's token (key format: `{SYMBOL}{chainLowercase}`)
3. A top-level `tokens` array containing the **NTT group names** (NOT the per-chain tokensConfig keys)

> [!CAUTION]
> The `tokens` top-level array contains NTT group names like `["MYTOKEN"]`, NOT per-chain keys like `["MYTOKENsepolia", "MYTOKENbasesepolia"]`. Getting this wrong causes the token selector to show "No tokens found".

```typescript
import { type config } from '@wormhole-foundation/wormhole-connect';
import {
    nttExecutorRoute,
    nttManualRoute,
    type NttRoute,
    type NttExecutorRoute,
} from '@wormhole-foundation/wormhole-connect/ntt';

const nttConfig: NttRoute.Config = {
  tokens: {
    MYTOKEN: [   // <-- This key is the NTT group name
          {
          { chain: 'Sepolia', manager: '0x...Manager', token: '0x...Token',
            transceiver: [{ address: '0x...Transceiver', type: 'wormhole' }] },
          { chain: 'BaseSepolia', manager: '0x...Manager', token: '0x...Token',
            transceiver: [{ address: '0x...Transceiver', type: 'wormhole' }] },
        ],
      },
  },
};

// Use both route types: executor (auto relay) and manual (user claims VAA)
const nttRoutes = [
    nttExecutorRoute({ ntt: nttConfig } satisfies NttExecutorRoute.Config),
    nttManualRoute(nttConfig),
] as unknown as config.WormholeConnectConfig['routes'];

const wormholeConfig: config.WormholeConnectConfig = {
  network: 'Testnet',
  chains: ['Sepolia', 'BaseSepolia'],
  tokens: ['MYTOKEN'],   // <-- NTT group names, NOT per-chain keys!

  ui: {
    title: 'My Token Bridge',
    defaultInputs: {
      source: { chain: 'Sepolia' },
      destination: { chain: 'BaseSepolia' },
    },
  },

  routes: nttRoutes,

  tokensConfig: {
    MYTOKENsepolia: {         // Key format: {SYMBOL}{chain.toLowerCase()}
      symbol: 'MYTOKEN',
      tokenId: { chain: 'Sepolia', address: '0x...Token...' },
      icon: 'https://example.com/token-icon.png',
      decimals: 18,
    },
    MYTOKENbasesepolia: {     // lowercase chain suffix!
      symbol: 'MYTOKEN',
      tokenId: { chain: 'BaseSepolia', address: '0x...Token...' },
      icon: 'https://example.com/token-icon.png',
      decimals: 18,
    },
  },
};
```

## Next.js SSR Prevention

Wormhole Connect uses browser-only APIs (`window`, `localStorage`). In Next.js (App Router), use dynamic import with `ssr: false` to prevent hydration crashes:

```typescript
'use client';

import dynamic from 'next/dynamic';

const WormholeConnect = dynamic(
    () =>
        import('@wormhole-foundation/wormhole-connect').then((m) => m.default),
    { ssr: false },
);
```

## Next.js Server→Client Boundary (Critical Gotcha)

`nttRoutes()` returns non-serializable objects (route classes with methods). In Next.js App Router, if you define the config in a Server Component (e.g., `page.tsx`) and pass it as a prop to a Client Component, you will get:

> **"Functions cannot be passed directly to Client Components unless you explicitly expose it by marking it with 'use server'."**

This error is misleading — the fix is NOT `"use server"`. The fix is to keep the **entire config including `nttRoutes()`** inside the `"use client"` component that renders `<WormholeConnect />`. Do not try to separate config from rendering across the Server/Client boundary.

```typescript
// ✅ Correct — config lives inside the client component
"use client";

export default function WormholeBridge() {
  const { nttRoutes } = require("@wormhole-foundation/wormhole-connect/ntt");
  const config = {
    routes: [...nttRoutes({ tokens: { ... } })],
    // ...rest of config
  };
  return <WormholeConnect config={config} />;
}
```

```typescript
// ❌ Wrong — passing config from Server Component to Client Component
// page.tsx (Server Component)
import { wormholeConfig } from "./config"; // contains nttRoutes()
export default function Page() {
  return <WormholeBridge config={wormholeConfig} />; // 💥 serialization error
}
```

## Where to Find Contract Addresses

All addresses for `manager`, `token`, and `transceiver` come directly from your `deployment.json` file (generated by the NTT CLI during deployment). The mapping is:

| Connect Config Field     | deployment.json Path                              |
| ------------------------ | ------------------------------------------------- |
| `manager`                | `chains[ChainName].manager`                       |
| `token`                  | `chains[ChainName].token`                         |
| `transceiver[0].address` | `chains[ChainName].transceivers.wormhole.address` |
