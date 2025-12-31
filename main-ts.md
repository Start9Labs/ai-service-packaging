# main.ts Patterns

## Basic Structure

```typescript
import { writeFile } from 'node:fs/promises'
import { sdk } from './sdk'
import { storeJson } from './fileModels/store.json'

export const main = sdk.setupMain(async ({ effects }) => {
  // 1. Read configuration
  const store = await storeJson.read((s) => s).const(effects)

  // 2. Get hostnames (for ALLOWED_HOSTS, CORS, etc.)
  // Use mapper function to extract only the data you need - service only restarts if mapped data changes
  const allowedHosts =
    (await sdk.serviceInterface
      .getOwn(effects, 'ui', (i) =>
        i?.addressInfo?.format('hostname-info').map((h) => h.hostname.value),
      )
      .const()) || []

  // 3. Create subcontainer
  const appSub = await sdk.SubContainer.of(
    effects,
    { imageId: 'my-service' },
    sdk.Mounts.of().mountVolume({
      volumeId: 'main',
      subpath: null,
      mountpoint: '/data',
      readonly: false,
    }),
    'my-service-sub',
  )

  // 4. Write config files to subcontainer rootfs (for ephemeral config)
  await writeFile(
    `${appSub.rootfs}/app/config.py`,
    generateConfig({ secretKey: store?.secretKey ?? '', allowedHosts }),
  )

  // 5. Define daemons and oneshots
  return sdk.Daemons.of(effects)
    .addOneshot('migrate', {
      subcontainer: appSub,
      exec: { command: ['python', 'manage.py', 'migrate', '--noinput'] },
      requires: [],
    })
    .addDaemon('main', {
      subcontainer: appSub,
      exec: { command: ['./start.sh'] },
      ready: {
        display: 'Web Interface',
        fn: () =>
          sdk.healthCheck.checkPortListening(effects, 8080, {
            successMessage: 'Service is ready',
            errorMessage: 'Service is not ready',
          }),
      },
      requires: ['migrate'],
    })
})
```

## Reactive vs One-time Reads

When reading configuration in `main.ts`, you choose how the system responds to changes:

| Method | Returns | Behavior on Change |
|--------|---------|-------------------|
| `.once()` | Parsed content only | Nothing - value is stale |
| `.const(effects)` | Parsed content | Re-runs the `setupMain` context, restarting daemons |

```typescript
// Reactive: re-runs setupMain when value changes (restarts daemons)
const store = await storeJson.read((s) => s).const(effects)

// One-time: read once, no re-run on change
const store = await storeJson.read((s) => s).once()
```

### Subset Reading

Use a mapper function to read only specific fields. This is more efficient and limits reactivity to only the fields you care about:

```typescript
// Read entire store - re-runs if ANY field changes
const store = await storeJson.read((s) => s).const(effects)

// Read only secretKey - re-runs only if secretKey changes
const secretKey = await storeJson.read((s) => s.secretKey).const(effects)
```

### Other Reading Methods

| Method | Purpose |
|--------|---------|
| `.onChange(effects, callback)` | Register callback for value changes |
| `.watch(effects)` | Create async iterator of new values |

## Getting Hostnames with Mapper

Use a mapper function to extract only the data you need. The service only restarts if the mapped result changes, not if other interface properties change:

```typescript
// With mapper - only restarts if hostnames change
const allowedHosts =
  (await sdk.serviceInterface
    .getOwn(effects, 'ui', (i) =>
      i?.addressInfo?.format('hostname-info').map((h) => h.hostname.value),
    )
    .const()) || []

// Without mapper - restarts on any interface change (not recommended)
const uiInterface = await sdk.serviceInterface.getOwn(effects, 'ui').const()
const allowedHosts = uiInterface?.addressInfo?.format('hostname-info').map((h) => h.hostname.value) ?? []
```

## Oneshots (Runtime)

Tasks that run on every startup before daemons. Use for idempotent operations like migrations:

```typescript
.addOneshot('migrate', {
  subcontainer: appSub,
  exec: { command: ['python', 'manage.py', 'migrate', '--noinput'] },
  requires: [],
})
.addOneshot('collectstatic', {
  subcontainer: appSub,
  exec: { command: ['python', 'manage.py', 'collectstatic', '--noinput'] },
  requires: ['migrate'],
})
```

**Important**: Do NOT put one-time setup tasks (like `createsuperuser`) in main.ts oneshots - they run on every startup and will fail on subsequent runs. Use `init/initializeService.ts` instead.

## One-time Setup (Install Only)

For tasks that should only run once during install (creating users, initial data), use `init/initializeService.ts`:

```typescript
// init/initializeService.ts
export const initializeService = sdk.setupOnInit(async (effects, kind) => {
  if (kind !== 'install') return

  const adminPassword = getRandomPassword()
  await storeJson.write(effects, { adminPassword })

  const appSub = await getAppSub(effects)

  // Write initial config
  await writeFile(`${appSub.rootfs}/app/config.py`, generateConfig({ ... }))

  // Run one-time setup tasks
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
    .runUntilSuccess(120_000)  // Run and wait for completion

  await sdk.action.createOwnTask(effects, getAdminCredentials, 'critical', {
    reason: 'Retrieve the admin password',
  })
})
```

## Environment Variables

```typescript
.addDaemon('main', {
  subcontainer: appSub,
  exec: {
    command: ['./start.sh'],
    env: {
      DATABASE_URL: 'sqlite:///data/db.sqlite3',
      SECRET_KEY: store?.secretKey ?? '',
    },
  },
  // ...
})
```

## Health Checks

```typescript
ready: {
  display: 'Web Interface',  // Shown in UI
  fn: () =>
    sdk.healthCheck.checkPortListening(effects, 8080, {
      successMessage: 'Service is ready',
      errorMessage: 'Service is not ready',
    }),
},
```

## Volume Mounts

```typescript
sdk.Mounts.of()
  // Mount entire volume (directory)
  .mountVolume({
    volumeId: 'main',
    subpath: null,
    mountpoint: '/data',
    readonly: false,
  })
  // Mount specific file from volume (requires type: 'file')
  .mountVolume({
    volumeId: 'main',
    subpath: 'config.py',
    mountpoint: '/app/config.py',
    readonly: true,
    type: 'file',  // Required when mounting a single file
  })
```

## Writing to Subcontainer Rootfs

For config files that are regenerated on every startup, write directly to the subcontainer's rootfs instead of using volume mounts. This is simpler and avoids file mount issues:

```typescript
// Create subcontainer first
const appSub = await sdk.SubContainer.of(
  effects,
  { imageId: 'my-service' },
  sdk.Mounts.of().mountVolume({
    volumeId: 'main',
    subpath: null,
    mountpoint: '/data',
    readonly: false,
  }),
  'my-service-sub',
)

// Write config directly to subcontainer rootfs
await writeFile(
  `${appSub.rootfs}/app/config.py`,
  generateConfig({ secretKey, allowedHosts }),
)
```

**When to use rootfs vs volume mounts:**
- **Rootfs**: Ephemeral config files regenerated on each startup (secrets, hostnames, etc.)
- **Volume mount (directory)**: Persistent data that survives restarts (databases, user files)
- **Volume mount (file)**: Persistent config that users might edit (requires `type: 'file'`)

## Config File Generation

```typescript
function generateConfig(config: { secretKey: string; allowedHosts: string[] }): string {
  const hostsList = config.allowedHosts.map((h) => `'${h}'`).join(', ')

  return `
SECRET_KEY = '${config.secretKey}'
ALLOWED_HOSTS = [${hostsList}]
DATABASE = '/data/db.sqlite3'
`
}
```
