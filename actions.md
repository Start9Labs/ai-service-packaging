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

## SMTP Configuration Action

The SDK provides a built-in SMTP input specification for managing email credentials. This supports three modes: disabled, system SMTP (from StartOS settings), or custom SMTP.

### 1. Add SMTP to store.json.ts

```typescript
import { matches, FileHelper } from '@start9labs/start-sdk'
import { sdk } from '../sdk'

const { object, string } = matches

const shape = object({
  adminPassword: string.optional().onMismatch(undefined),
  secretKey: string.optional().onMismatch(undefined),
  smtp: sdk.inputSpecConstants.smtpInputSpec.validator.onMismatch({
    selection: 'disabled',
    value: {},
  }),
})

export const storeJson = FileHelper.json(
  { volumeId: 'main', subpath: 'store.json' },
  shape,
)
```

### 2. Create manageSmtp.ts action

```typescript
import { storeJson } from '../fileModels/store.json'
import { sdk } from '../sdk'

const { InputSpec } = sdk

export const inputSpec = InputSpec.of({
  smtp: sdk.inputSpecConstants.smtpInputSpec,
})

export const manageSmtp = sdk.Action.withInput(
  'manage-smtp',

  async ({ effects }) => ({
    name: 'Configure SMTP',
    description: 'Add SMTP credentials for sending emails',
    warning: null,
    allowedStatuses: 'any',
    group: null,
    visibility: 'enabled',
  }),

  inputSpec,

  // Pre-fill form with current values
  async ({ effects }) => ({
    smtp: (await storeJson.read((s) => s.smtp).const(effects)) || undefined,
  }),

  // Save to store
  async ({ effects, input }) => storeJson.merge(effects, { smtp: input.smtp }),
)
```

### 3. Register the action

```typescript
import { sdk } from '../sdk'
import { getAdminCredentials } from './getAdminCredentials'
import { manageSmtp } from './manageSmtp'

export const actions = sdk.Actions.of()
  .addAction(getAdminCredentials)
  .addAction(manageSmtp)
```

### 4. Use SMTP credentials in main.ts

```typescript
import { T } from '@start9labs/start-sdk'

export const main = sdk.setupMain(async ({ effects }) => {
  const store = await storeJson.read((s) => s).const(effects)

  // Resolve SMTP credentials based on selection
  const smtp = store?.smtp
  let smtpCredentials: T.SmtpValue | null = null

  if (smtp?.selection === 'system') {
    // Use system-wide SMTP from StartOS settings
    smtpCredentials = await sdk.getSystemSmtp(effects).const()
    if (smtpCredentials && smtp.value.customFrom) {
      smtpCredentials.from = smtp.value.customFrom
    }
  } else if (smtp?.selection === 'custom') {
    // Use custom SMTP credentials
    smtpCredentials = smtp.value
  }
  // If smtp.selection === 'disabled', smtpCredentials remains null

  // Pass to config generation
  const config = generateConfig({
    smtp: smtpCredentials,
    // ... other config
  })

  // ...
})
```

### 5. Initialize with SMTP disabled

In `init/initializeService.ts`:

```typescript
await storeJson.write(effects, {
  adminPassword,
  secretKey,
  smtp: { selection: 'disabled', value: {} },
})
```

### T.SmtpValue Type

The resolved SMTP credentials have this structure:

```typescript
interface SmtpValue {
  server: string
  port: number
  login: string
  password?: string | null
  from: string
}
```
