# Actions Patterns

## Action Without Input

```typescript
import { sdk } from '../sdk'
import { storeJson } from '../fileModels/store.json'

export const getAdminCredentials = sdk.Action.withoutInput(
  // ID
  'get-admin-credentials',

  // Metadata
  async ({ effects }) => ({
    name: 'Get Admin Credentials',
    description: 'Retrieve admin username and password',
    warning: null,
    allowedStatuses: 'any',  // 'any', 'only-running', 'only-stopped'
    group: null,
    visibility: 'enabled',   // 'enabled', 'disabled', 'hidden'
  }),

  // Handler
  async ({ effects }) => {
    const store = await storeJson.read((s) => s).once()

    return {
      version: '1' as const,
      title: 'Admin Credentials',
      message: 'Your admin credentials:',
      result: {
        type: 'group',
        value: [
          {
            type: 'single',
            name: 'Username',
            description: null,
            value: 'admin',
            masked: false,
            copyable: true,
            qr: false,
          },
          {
            type: 'single',
            name: 'Password',
            description: null,
            value: store?.adminPassword ?? 'UNKNOWN',
            masked: true,
            copyable: true,
            qr: false,
          },
        ],
      },
    }
  },
)
```

## Register Actions

In `actions/index.ts`:

```typescript
import { sdk } from '../sdk'
import { getAdminCredentials } from './getAdminCredentials'

export const actions = sdk.Actions.of().addAction(getAdminCredentials)
```

## Result Types

### Single Value
```typescript
result: {
  type: 'single',
  name: 'API Key',
  description: null,
  value: 'abc123',
  masked: true,
  copyable: true,
  qr: false,
}
```

### Group of Values
```typescript
result: {
  type: 'group',
  value: [
    { type: 'single', name: 'Username', value: 'admin', masked: false, copyable: true, qr: false },
    { type: 'single', name: 'Password', value: 'secret', masked: true, copyable: true, qr: false },
  ],
}
```

## Creating Tasks (Prompts)

In `init/initializeService.ts`, prompt user to run an action:

```typescript
await sdk.action.createOwnTask(effects, getAdminCredentials, 'critical', {
  reason: 'Retrieve the admin password',
})
```

Priority levels: `'critical'`, `'high'`, `'medium'`, `'low'`
