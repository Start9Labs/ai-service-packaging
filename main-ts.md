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

  // 3. Write config files to volume
  await writeFile(
    '/media/startos/volumes/main/config.py',
    generateConfig({ secretKey: store?.secretKey ?? '', allowedHosts }),
  )

  // 4. Create subcontainer
  const appSub = await sdk.SubContainer.of(
    effects,
    { imageId: 'my-service' },
    sdk.Mounts.of()
      .mountVolume({
        volumeId: 'main',
        subpath: null,
        mountpoint: '/data',
        readonly: false,
      })
      .mountVolume({
        volumeId: 'main',
        subpath: 'config.py',
        mountpoint: '/app/config.py',
        readonly: true,
      }),
    'my-service-sub',
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

```typescript
// Reactive: service restarts when value changes
const store = await storeJson.read((s) => s).const(effects)

// One-time: read once, no restart on change
const store = await storeJson.read((s) => s).once()
```

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

## Oneshots

One-time tasks that run before daemons:

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
.addOneshot('create-superuser', {
  subcontainer: appSub,
  exec: {
    command: ['python', 'manage.py', 'createsuperuser', '--noinput'],
    env: {
      DJANGO_SUPERUSER_USERNAME: 'admin',
      DJANGO_SUPERUSER_PASSWORD: store?.adminPassword ?? '',
    },
  },
  requires: ['migrate'],
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
  // Mount entire volume
  .mountVolume({
    volumeId: 'main',
    subpath: null,
    mountpoint: '/data',
    readonly: false,
  })
  // Mount specific file
  .mountVolume({
    volumeId: 'main',
    subpath: 'config.py',
    mountpoint: '/app/config.py',
    readonly: true,
  })
```

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
