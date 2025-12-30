# interfaces.ts Patterns

## Single Interface

See `hello-world-startos/startos/interfaces.ts` for the basic pattern.

## Multiple Interfaces

Expose multiple paths (e.g., web UI and admin panel):

```typescript
import { sdk } from './sdk'

const uiPort = 8080

export const setInterfaces = sdk.setupInterfaces(async ({ effects }) => {
  const uiMulti = sdk.MultiHost.of(effects, 'ui-multi')
  const uiMultiOrigin = await uiMulti.bindPort(uiPort, {
    protocol: 'http',
  })

  const ui = sdk.createInterface(effects, {
    name: 'Web UI',
    id: 'ui',
    description: 'The web interface',
    type: 'ui',
    masked: false,
    schemeOverride: null,
    username: null,
    path: '',
    query: {},
  })

  const admin = sdk.createInterface(effects, {
    name: 'Admin Panel',
    id: 'admin',
    description: 'Admin interface',
    type: 'ui',
    masked: false,
    schemeOverride: null,
    username: null,
    path: '/admin/',
    query: {},
  })

  const uiReceipt = await uiMultiOrigin.export([ui, admin])
  return [uiReceipt]
})
```

## Interface Options

```typescript
sdk.createInterface(effects, {
  name: 'Display Name',           // Shown in UI
  id: 'unique-id',                // Used in sdk.serviceInterface.getOwn()
  description: 'Description',     // Shown in UI
  type: 'ui',                     // 'ui' or 'api'
  masked: false,                  // Hide from discovery?
  schemeOverride: null,           // Force 'https' or 'http'?
  username: null,                 // Auth username (if any)
  path: '/some/path/',            // URL path
  query: {},                      // URL query params
})
```
