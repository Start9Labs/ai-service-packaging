# StartOS Service Packaging Guide

This document explains how to package a service for StartOS based on patterns learned while packaging the Django Wedding Website.

## Project Structure

A StartOS service package has the following structure:

```
my-service-startos/
├── startos/                    # Core service configuration (TypeScript)
│   ├── manifest.ts             # Service metadata
│   ├── sdk.ts                  # SDK initialization (boilerplate)
│   ├── index.ts                # Main exports (boilerplate)
│   ├── main.ts                 # Daemon/runtime configuration
│   ├── interfaces.ts           # Network interface definitions
│   ├── backups.ts              # Backup configuration
│   ├── dependencies.ts         # External service dependencies
│   ├── utils.ts                # Shared utilities
│   ├── actions/                # User-triggered actions
│   │   └── index.ts
│   ├── init/                   # Initialization logic
│   │   ├── index.ts
│   │   └── initializeService.ts
│   ├── install/                # Version management
│   │   ├── versionGraph.ts
│   │   └── versions/
│   │       ├── index.ts
│   │       └── v1.0.0.ts
│   └── fileModels/             # Configuration file helpers
│       └── store.json.ts
├── Dockerfile                  # Docker image build
├── Makefile                    # Build automation
├── package.json                # Node.js dependencies
├── tsconfig.json               # TypeScript configuration
└── upstream-project/           # Git submodule of upstream project
```

## Getting Started

1. **Copy boilerplate from hello-world-startos**: Copy `package.json`, `tsconfig.json`, `Makefile`, and the `startos/` directory structure.

2. **Add upstream project as a git submodule** (if wrapping an existing project):
   ```bash
   git submodule add https://github.com/user/upstream-project.git upstream-project
   ```

3. **Update `package.json`** with your service name.

4. **Install dependencies**:
   ```bash
   npm install
   ```

## Key Files

### manifest.ts

Defines service metadata. See `hello-world-startos/startos/manifest.ts` for the basic structure.

Key fields to customize:
- `id`: Unique service identifier (lowercase, hyphens)
- `title`: Display name
- `description`: Short and long descriptions
- `volumes`: Storage volumes (usually just `['main']`)
- `images`: Docker images to use
- `alerts`: User notifications for install/update/uninstall

**Using a pre-built Docker image:**
```typescript
images: {
  'my-service': {
    source: { dockerTag: 'someuser/someimage:tag' },
  },
},
```

**Building a custom Docker image:**
```typescript
images: {
  'my-service': {
    source: {
      dockerBuild: {
        workdir: './',
        dockerfile: 'Dockerfile',
      },
    },
  },
},
```

### main.ts

Defines how the service runs. This is where you configure daemons, oneshots, and health checks.

**Basic structure:**
```typescript
import { writeFile } from 'node:fs/promises'
import { sdk } from './sdk'
import { storeJson } from './fileModels/store.json'

export const main = sdk.setupMain(async ({ effects }) => {
  // Read configuration from store
  const store = await storeJson.read((s) => s).const(effects)

  // Get hostnames for this service (for ALLOWED_HOSTS, CORS, etc.)
  const uiInterface = await sdk.serviceInterface.getOwn(effects, 'ui').const()
  if (!uiInterface) throw new Error('interfaces do not exist')
  const hostnames = uiInterface.addressInfo?.format('hostname-info')
  const allowedHosts = hostnames?.map((h) => h.hostname.value) ?? []

  // Write configuration files to the volume
  await writeFile(
    '/media/startos/volumes/main/config.py',
    generateConfig({ secretKey: store?.secretKey ?? '', allowedHosts }),
  )

  // Create subcontainer
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

  // Define daemons and oneshots
  return sdk.Daemons.of(effects)
    .addOneshot('setup', {
      subcontainer: appSub,
      exec: { command: ['./setup.sh'] },
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
      requires: ['setup'],
    })
})
```

**Key concepts:**

- **`.const(effects)`**: Makes the value reactive. If the underlying data changes, the service restarts with the new value. Use this for configuration that should trigger a restart when changed.

- **`.once()`**: Reads the value once without reactivity.

- **Oneshots**: One-time tasks that run before daemons (e.g., migrations, setup scripts).

- **Daemons**: Long-running processes.

- **`requires`**: Specifies dependencies between oneshots/daemons. A daemon/oneshot won't start until all its requirements have completed.

### interfaces.ts

Defines network interfaces. See `hello-world-startos/startos/interfaces.ts` for the basic structure.

**Exposing multiple interfaces (e.g., web UI and API):**
```typescript
export const setInterfaces = sdk.setupInterfaces(async ({ effects }) => {
  const uiMulti = sdk.MultiHost.of(effects, 'ui-multi')
  const uiMultiOrigin = await uiMulti.bindPort(8080, {
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

### Storing Service State (fileModels/store.json.ts)

Use `FileHelper.json` to persist service state (passwords, secrets, etc.):

```typescript
import { matches, FileHelper } from '@start9labs/start-sdk'

const { object, string } = matches
const shape = object({
  adminPassword: string.optional().onMismatch(undefined),
  secretKey: string.optional().onMismatch(undefined),
})

export const storeJson = FileHelper.json(
  {
    volumeId: 'main',
    subpath: 'store.json',
  },
  shape,
)
```

### Initialization (init/initializeService.ts)

Run one-time setup on install:

```typescript
import { sdk } from '../sdk'
import { storeJson } from '../fileModels/store.json'
import { utils } from '@start9labs/start-sdk'

function getRandomPassword(length: number = 24): string {
  return utils.getDefaultString({
    charset: 'a-z,A-Z,0-9',
    len: length,
  })
}

export const initializeService = sdk.setupOnInit(async (effects, kind) => {
  if (kind !== 'install') return

  // Generate secrets on fresh install
  const adminPassword = getRandomPassword()
  const secretKey = getRandomPassword(50)

  await storeJson.write(effects, { adminPassword, secretKey })

  // Create a task prompting user to get credentials
  await sdk.action.createOwnTask(effects, getAdminCredentials, 'critical', {
    reason: 'Retrieve the admin password',
  })
})
```

### Actions (actions/getAdminCredentials.ts)

User-triggered actions:

```typescript
import { sdk } from '../sdk'
import { storeJson } from '../fileModels/store.json'

export const getAdminCredentials = sdk.Action.withoutInput(
  'get-admin-credentials',

  async ({ effects }) => ({
    name: 'Get Admin Credentials',
    description: 'Retrieve admin username and password',
    warning: null,
    allowedStatuses: 'any',
    group: null,
    visibility: 'enabled',
  }),

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

Register actions in `actions/index.ts`:
```typescript
import { sdk } from '../sdk'
import { getAdminCredentials } from './getAdminCredentials'

export const actions = sdk.Actions.of().addAction(getAdminCredentials)
```

## Dockerfile

When wrapping an upstream project via submodule:

```dockerfile
FROM python:3.12-slim

ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1

WORKDIR /app

# Copy from submodule
COPY upstream-project/ .

# Install dependencies
RUN pip install --no-cache-dir -r requirements.txt

# Create data directory
RUN mkdir -p /data

EXPOSE 8080
```

Note: No `ENTRYPOINT` or `CMD` needed - StartOS controls execution via `main.ts`.

## Volume Paths

- **Host path**: `/media/startos/volumes/main/` - Use this in `main.ts` with Node's `fs` APIs
- **Container mount**: Configure in `sdk.Mounts.of().mountVolume(...)` - Maps volume paths into the container

## Writing Configuration Files

Use Node's `fs/promises` to write files directly to the volume, then mount them into the container:

```typescript
import { writeFile } from 'node:fs/promises'

// In main.ts
await writeFile(
  '/media/startos/volumes/main/app-config.json',
  JSON.stringify(config),
)

// Mount it in the subcontainer
sdk.Mounts.of()
  .mountVolume({
    volumeId: 'main',
    subpath: 'app-config.json',
    mountpoint: '/app/config.json',
    readonly: true,
  })
```

## Build and Test

```bash
# Check TypeScript
npm run check

# Build JavaScript bundle
npm run build

# Build .s9pk package (requires start-cli)
make

# Build for specific architecture
make x86_64
make aarch64

# Install to local StartOS server
make install
```

## Common Patterns

### Getting Hostnames for ALLOWED_HOSTS/CORS

```typescript
const uiInterface = await sdk.serviceInterface.getOwn(effects, 'ui').const()
if (!uiInterface) throw new Error('interfaces do not exist')
const hostnames = uiInterface.addressInfo?.format('hostname-info')
const allowedHosts = hostnames?.map((h) => h.hostname.value) ?? []
```

### Generating Config Files with Template Strings

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

### Environment Variables in Daemons

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

### Running Setup Commands (Oneshots)

```typescript
.addOneshot('migrate', {
  subcontainer: appSub,
  exec: {
    command: ['python', 'manage.py', 'migrate', '--noinput'],
  },
  requires: [],
})
.addOneshot('create-admin', {
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
.addDaemon('gunicorn', {
  // ...
  requires: ['migrate', 'create-admin'],
})
```

## Checklist

- [ ] `manifest.ts` - Service metadata configured
- [ ] `Dockerfile` - Builds the container image
- [ ] `main.ts` - Daemons and oneshots defined
- [ ] `interfaces.ts` - Network interfaces exposed
- [ ] `backups.ts` - Volume backups configured
- [ ] `init/initializeService.ts` - Secrets generated on install
- [ ] `actions/` - User actions (e.g., get credentials)
- [ ] `fileModels/store.json.ts` - Persistent state storage
- [ ] Git submodule for upstream project (if applicable)
- [ ] TypeScript compiles (`npm run check`)
- [ ] Package builds (`make`)
