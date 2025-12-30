# StartOS Service Packaging Guide

Reference `hello-world-startos/` for boilerplate files (`package.json`, `tsconfig.json`, `Makefile`, `startos/` structure).

## Project Structure

```
my-service-startos/
├── startos/
│   ├── manifest.ts         # Service metadata
│   ├── sdk.ts              # SDK init (boilerplate)
│   ├── index.ts            # Exports (boilerplate)
│   ├── main.ts             # Runtime: daemons, oneshots, health checks
│   ├── interfaces.ts       # Network interfaces
│   ├── backups.ts          # Backup config (boilerplate)
│   ├── dependencies.ts     # Service dependencies (boilerplate)
│   ├── utils.ts            # Shared utilities
│   ├── actions/            # User-triggered actions
│   ├── init/               # Initialization logic
│   ├── install/versions/   # Version management
│   └── fileModels/         # Persistent state (store.json.ts)
├── Dockerfile
├── Makefile
├── package.json
├── tsconfig.json
└── upstream-project/       # Git submodule (optional)
```

## APIs and When to Use Them

### manifest.ts
**When**: Always - defines service identity and metadata.
- `id`, `title`, `description`, `license`
- `volumes`: Storage volumes (usually `['main']`)
- `images`: Docker images (pre-built tag or local build)
- `alerts`: User notifications for install/update/uninstall

### main.ts - Runtime Configuration
**When**: Always - defines how the service runs.

| API | When to Use |
|-----|-------------|
| `storeJson.read((s) => s).const(effects)` | Read config reactively (restarts service on change) |
| `storeJson.read((s) => s).once()` | Read config once (no restart on change) |
| `sdk.serviceInterface.getOwn(effects, 'ui', mapper).const()` | Get service hostnames (with mapper to avoid unnecessary restarts) |
| `writeFile('/media/startos/volumes/main/...', content)` | Write config files to volume |
| `sdk.SubContainer.of(effects, {imageId}, mounts, name)` | Create container with volume mounts |
| `sdk.Daemons.of(effects).addOneshot(...)` | One-time setup tasks (migrations, etc.) |
| `sdk.Daemons.of(effects).addDaemon(...)` | Long-running processes |
| `sdk.healthCheck.checkPortListening(effects, port, msgs)` | Health check for daemon readiness |

See [main.ts patterns](./main-ts.md) for detailed examples.

### interfaces.ts
**When**: Service exposes network interfaces (web UI, API, etc.).

| API | When to Use |
|-----|-------------|
| `sdk.MultiHost.of(effects, 'ui-multi')` | Create network binding |
| `uiMulti.bindPort(port, {protocol: 'http'})` | Bind to a port |
| `sdk.createInterface(effects, {...})` | Define an interface (UI, API, etc.) |
| `origin.export([...interfaces])` | Export interfaces |

See [interfaces.ts patterns](./interfaces-ts.md) for multiple interfaces.

### actions/
**When**: Users need to trigger operations (get credentials, reset password, etc.).

| API | When to Use |
|-----|-------------|
| `sdk.Action.withoutInput(id, metadata, handler)` | Action with no user input |
| `sdk.Action.withInput(id, inputSpec, metadata, handler)` | Action requiring user input |
| `sdk.Actions.of().addAction(...)` | Register actions |

See [actions.md](./actions.md) for action patterns.

### init/initializeService.ts
**When**: Need one-time setup on install (generate secrets, create tasks).

| API | When to Use |
|-----|-------------|
| `sdk.setupOnInit(async (effects, kind) => {...})` | Run code on init |
| `storeJson.write(effects, {...})` | Persist initial state |
| `sdk.action.createOwnTask(effects, action, priority, {reason})` | Prompt user to run action |
| `utils.getDefaultString({charset, len})` | Generate random strings |

### fileModels/store.json.ts
**When**: Need to persist service state (passwords, secrets, settings).

| API | When to Use |
|-----|-------------|
| `FileHelper.json({volumeId, subpath}, shape)` | JSON file with schema validation |
| `matches.object({...})` | Define shape with `matches` |

## Volume Paths

- **Node.js (main.ts)**: `/media/startos/volumes/main/`
- **Container mount**: Use `sdk.Mounts.of().mountVolume({volumeId, subpath, mountpoint, readonly})`

## Dockerfile

No `ENTRYPOINT` or `CMD` needed - StartOS controls execution via `main.ts`.

For upstream projects, use git submodules:
```bash
git submodule add https://github.com/user/project.git upstream-project
```

Then in Dockerfile:
```dockerfile
COPY upstream-project/ .
```

## Build Commands

```bash
npm run check    # TypeScript check
npm run build    # Build JS bundle
make             # Build .s9pk package
make install     # Install to local StartOS
```

## Checklist

- [ ] `manifest.ts` configured
- [ ] `Dockerfile` builds image
- [ ] `main.ts` defines daemons/oneshots
- [ ] `interfaces.ts` exposes network
- [ ] `init/` generates secrets on install
- [ ] `actions/` for user operations
- [ ] `fileModels/store.json.ts` for state
- [ ] `npm run check` passes
- [ ] `make` succeeds
