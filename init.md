# Initialization Patterns

## Overview

Use `init/initializeService.ts` for one-time setup tasks that should only run during install (not on every startup). This is the place for:
- Generating passwords/secrets
- Creating admin users
- Initial database setup
- Bootstrapping services that require API calls

## Basic Structure

```typescript
// init/initializeService.ts
import { utils } from '@start9labs/start-sdk'
import { sdk } from '../sdk'
import { storeJson } from '../fileModels/store.json'
import { getAdminPassword } from '../actions/getAdminPassword'

export const initializeService = sdk.setupOnInit(async (effects, kind) => {
  if (kind !== 'install') return

  // Generate and store password
  const adminPassword = utils.getDefaultString({
    charset: 'a-z,A-Z,0-9',
    len: 22,
  })
  await storeJson.write(effects, { adminPassword })

  // Create task prompting user to retrieve password
  await sdk.action.createOwnTask(effects, getAdminPassword, 'critical', {
    reason: 'Retrieve the admin password',
  })
})
```

## Registering initializeService

Add to `init/index.ts`:

```typescript
import { sdk } from '../sdk'
import { setDependencies } from '../dependencies'
import { setInterfaces } from '../interfaces'
import { versionGraph } from '../install/versionGraph'
import { actions } from '../actions'
import { restoreInit } from '../backups'
import { initializeService } from './initializeService'

export const init = sdk.setupInit(
  restoreInit,
  versionGraph,
  setInterfaces,
  setDependencies,
  actions,
  initializeService,  // Add this
)

export const uninit = sdk.setupUninit(versionGraph)
```

## runUntilSuccess Pattern

Use `runUntilSuccess(timeout)` to run daemons and oneshots during init, waiting for completion before continuing. This is essential for setup tasks that need a running server.

### Oneshots Only

For simple sequential tasks (like database migrations):

```typescript
await sdk.Daemons.of(effects)
  .addOneshot('migrate', {
    subcontainer: appSub,
    exec: { command: ['python', 'manage.py', 'migrate', '--noinput'] },
    requires: [],
  })
  .addOneshot('create-superuser', {
    subcontainer: appSub,
    exec: {
      command: ['python', 'manage.py', 'createsuperuser', '--noinput'],
      env: {
        DJANGO_SUPERUSER_USERNAME: 'admin',
        DJANGO_SUPERUSER_PASSWORD: adminPassword,
      },
    },
    requires: ['migrate'],
  })
  .runUntilSuccess(120_000)  // 2 minute timeout
```

### Daemon + Dependent Oneshot

For services that require calling an API after the server starts (e.g., bootstrapping via HTTP):

```typescript
await sdk.Daemons.of(effects)
  .addDaemon('server', {
    subcontainer: appSub,
    exec: { command: ['node', 'server.js'] },
    ready: {
      display: null,
      fn: () =>
        sdk.healthCheck.checkPortListening(effects, 8080, {
          successMessage: 'Server ready',
          errorMessage: 'Server not ready',
        }),
    },
    requires: [],
  })
  .addOneshot('bootstrap', {
    subcontainer: appSub,
    exec: {
      command: [
        'node',
        '-e',
        `fetch('http://127.0.0.1:8080/api/bootstrap', {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify({ password: '${adminPassword}' })
        }).then(r => {
          if (!r.ok) throw new Error('Bootstrap failed');
          process.exit(0);
        }).catch(e => {
          console.error(e);
          process.exit(1);
        })`,
      ],
    },
    requires: ['server'],  // Waits for daemon to be healthy
  })
  .runUntilSuccess(120_000)
```

**How it works:**
1. The daemon starts and runs its health check
2. Once healthy, the dependent oneshot executes
3. When the oneshot completes successfully, `runUntilSuccess` returns
4. All processes are cleaned up automatically

### Making HTTP Calls Without curl

Many slim Docker images don't have curl. Use the runtime's built-in HTTP capabilities:

**Node.js (v18+):**
```typescript
command: [
  'node',
  '-e',
  `fetch('http://127.0.0.1:${port}/api/endpoint', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ key: 'value' })
  }).then(r => r.ok ? process.exit(0) : process.exit(1))
    .catch(() => process.exit(1))`,
]
```

**Python:**
```typescript
command: [
  'python',
  '-c',
  `import urllib.request, json
req = urllib.request.Request(
  'http://127.0.0.1:${port}/api/endpoint',
  data=json.dumps({'key': 'value'}).encode(),
  headers={'Content-Type': 'application/json'},
  method='POST'
)
urllib.request.urlopen(req)`,
]
```

## Common Patterns

### Generate Random Password

```typescript
import { utils } from '@start9labs/start-sdk'

const password = utils.getDefaultString({
  charset: 'a-z,A-Z,0-9',
  len: 22,
})
```

### Create User Task

Prompt user to run an action after install:

```typescript
await sdk.action.createOwnTask(effects, getAdminPassword, 'critical', {
  reason: 'Retrieve the admin password',
})
```

Priority levels: `'critical'`, `'high'`, `'medium'`, `'low'`

### Check Install vs Restore

```typescript
export const initializeService = sdk.setupOnInit(async (effects, kind) => {
  if (kind !== 'install') return  // Skip on restore

  // ... install-only setup
})
```
