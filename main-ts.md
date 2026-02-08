# main.ts Patterns

## Basic Structure

```typescript
import { writeFile } from 'node:fs/promises'
import { i18n } from './i18n'
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
        display: i18n('Web Interface'),
        fn: () =>
          sdk.healthCheck.checkPortListening(effects, 8080, {
            successMessage: i18n('Service is ready'),
            errorMessage: i18n('Service is not ready'),
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

**Important**: Do NOT put one-time setup tasks (like `createsuperuser`) in main.ts oneshots - they run on every startup and will fail on subsequent runs. Use `init/initializeService.ts` instead. See [Initialization Patterns](./init.md) for details.

## Exec Command

### Using Upstream Entrypoint

If the upstream Docker image has a compatible `ENTRYPOINT`/`CMD`, use `sdk.useEntrypoint()` instead of specifying a custom command. This is the simplest approach and ensures compatibility with the upstream image:

```typescript
.addDaemon('primary', {
  subcontainer: appSub,
  exec: {
    command: sdk.useEntrypoint(),
  },
  // ...
})
```

You can pass an array of arguments to override the image's `CMD` while keeping the `ENTRYPOINT`:

```typescript
.addDaemon('postgres', {
  subcontainer: postgresSub,
  exec: {
    command: sdk.useEntrypoint(['-c', 'random_page_cost=1.0']),
  },
  // ...
})
```

**When to use `sdk.useEntrypoint()`:**
- Upstream image has a working entrypoint that starts the service correctly
- You want to use the entrypoint but optionally override CMD arguments
- Examples: Ollama, Jellyfin, Vaultwarden, Postgres

### Custom Command

Use a custom command array when you need to bypass the entrypoint entirely:

```typescript
.addDaemon('primary', {
  subcontainer: appSub,
  exec: {
    command: ['/opt/app/bin/start.sh', '--port=' + uiPort],
  },
  // ...
})
```

## Environment Variables

```typescript
.addDaemon('main', {
  subcontainer: appSub,
  exec: {
    command: sdk.useEntrypoint(),  // or custom command
    env: {
      DATABASE_URL: 'sqlite:///data/db.sqlite3',
      SECRET_KEY: store?.secretKey ?? '',
    },
  },
  // ...
})
```

## Health Checks

All user-facing strings must be wrapped with `i18n()`:

```typescript
ready: {
  display: i18n('Web Interface'),  // Shown in UI
  fn: () =>
    sdk.healthCheck.checkPortListening(effects, 8080, {
      successMessage: i18n('Service is ready'),
      errorMessage: i18n('Service is not ready'),
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

## Executing Commands in SubContainers

Use `exec` or `execFail` to run commands in a subcontainer:

| Method | Behavior on Non-zero Exit |
|--------|---------------------------|
| `exec()` | Returns result with `exitCode`, `stdout`, `stderr` - does NOT throw |
| `execFail()` | Throws an error on non-zero exit code |

```typescript
// exec() - manual error handling (good for optional/warning cases)
const result = await appSub.exec(['update-ca-certificates'], { user: 'root' })
if (result.exitCode !== 0) {
  console.warn('Failed to update CA certificates:', result.stderr)
}

// execFail() - throws on error (good for required commands)
// Uses the default user from the Dockerfile (no need to specify { user: '...' })
await appSub.execFail(['git', 'clone', 'https://github.com/user/repo.git'])

// Override user when needed (e.g., run as root)
await appSub.exec(['update-ca-certificates'], { user: 'root' })
```

**User option**: The `user` option is optional. If omitted, commands run as the default user defined in the Dockerfile (`USER` directive). Only specify `{ user: 'root' }` when you need elevated privileges.

**Use `execFail()` when:**
- The command must succeed for the service to work correctly
- You're in `initializeService.ts` and want installation to fail if setup fails
- You want automatic error propagation

**Use `exec()` when:**
- The command failure is not critical (warnings, optional setup)
- You need to inspect the exit code or output regardless of success/failure
- You want custom error handling logic

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
