# StartOS Service Packaging Guide

## Getting Started

1. [Environment Setup](./environment-setup.md) - Install required tools (Docker, Node.js, Start CLI, etc.)
2. [Quick Start](./quick-start.md) - Create your first package from the Hello World template
3. [Project Structure](./project-structure.md) - Understand the file layout and directory purposes

## Detailed Documentation

- [manifest.ts](./manifest-ts.md) - Service metadata, images, alerts, dependencies
- [Versioning](./versions.md) - ExVer format, version selection, migrations
- [main.ts Patterns](./main-ts.md) - Daemons, oneshots, health checks, volume mounts
- [Initialization Patterns](./init.md) - One-time setup, runUntilSuccess, bootstrapping via API
- [interfaces.ts Patterns](./interfaces-ts.md) - Network interfaces and port bindings
- [Actions](./actions.md) - User-triggered operations and SMTP configuration
- [File Models](./file-models.md) - Type-safe configuration files and store.json

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
├── assets/                 # Additional files (required, can be empty)
│   └── README.md
├── Dockerfile              # Optional - only if upstream doesn't have one
├── Makefile
├── package.json
├── tsconfig.json
├── icon.*                  # Symlink to upstream or custom (svg preferred, max 40 KiB)
├── LICENSE                 # Symlink to upstream license
├── .gitignore              # Ignore build artifacts, node_modules, etc.
└── upstream-project/       # Git submodule
```

## APIs and When to Use Them

### manifest.ts
**When**: Always - defines service identity and metadata.

| Field | Description |
|-------|-------------|
| `id`, `title`, `license`, `docsUrl` | Required metadata |
| `volumes` | Storage volumes (usually `['main']`) |
| `images` | Docker images (pre-built tag or local build) |
| `alerts` | User notifications for install/update/uninstall |
| `dependencies` | Service dependencies |

See [manifest.ts](./manifest-ts.md) for detailed configuration (images, alerts, dependencies, license/icon setup).

### main.ts - Runtime Configuration
**When**: Always - defines how the service runs.

| API | When to Use |
|-----|-------------|
| `storeJson.read((s) => s).const(effects)` | Read config reactively (restarts service on change) |
| `storeJson.read((s) => s).once()` | Read config once (no restart on change) |
| `sdk.serviceInterface.getOwn(effects, 'ui', mapper).const()` | Get service hostnames (with mapper to avoid unnecessary restarts) |
| `sdk.SubContainer.of(effects, {imageId}, mounts, name)` | Create container with volume mounts |
| `sdk.useEntrypoint()` | Use upstream image's ENTRYPOINT/CMD (prefer over custom command) |
| `sdk.useEntrypoint([args])` | Use upstream ENTRYPOINT with custom CMD arguments |
| `writeFile(\`${appSub.rootfs}/path\`, content)` | Write ephemeral config to subcontainer rootfs |
| `writeFile('/media/startos/volumes/main/...', content)` | Write persistent files to volume |
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
**When**: Need one-time setup on install (generate secrets, bootstrap via API, create tasks).

| API | When to Use |
|-----|-------------|
| `sdk.setupOnInit(async (effects, kind) => {...})` | Run code on init |
| `storeJson.write(effects, {...})` | Persist initial state |
| `sdk.action.createOwnTask(effects, action, priority, {reason})` | Prompt user to run action |
| `utils.getDefaultString({charset, len})` | Generate random strings |
| `.runUntilSuccess(timeout)` | Run daemons/oneshots and wait for completion |

See [Initialization Patterns](./init.md) for `runUntilSuccess` and bootstrapping via API.

### fileModels/store.json.ts
**When**: Need to persist service state (passwords, secrets, settings).

| API | When to Use |
|-----|-------------|
| `FileHelper.json({volumeId, subpath}, shape)` | JSON file with schema validation |
| `matches.object({...})` | Define shape with `matches` |

## Writing Files

- **Subcontainer rootfs** (`${appSub.rootfs}/path`): For ephemeral config regenerated on startup
- **Volume** (`/media/startos/volumes/main/`): For persistent data that survives restarts
- **Volume file mount**: Add `type: 'file'` when mounting a single file from a volume

See [main.ts patterns](./main-ts.md) for details on rootfs vs volume mounts.

## Dockerfile

For upstream projects, use git submodules:
```bash
git submodule add https://github.com/user/project.git upstream-project
```

See [manifest.ts](./manifest-ts.md) for Docker image configuration (`dockerTag` vs `dockerBuild`).

## Build Commands

```bash
npm run check    # TypeScript check
npm run build    # Build JS bundle
make             # Build .s9pk package
make install     # Install to local StartOS
```

## Checklist

- [ ] `.gitignore` copied from `hello-world-startos/.gitignore` (boilerplate)
- [ ] `manifest.ts` configured with correct license and `docsUrl`
- [ ] `LICENSE` symlink to upstream license file
- [ ] `icon.*` symlink to upstream icon or custom (svg preferred, max 40 KiB)
- [ ] `assets/` directory exists (can be empty with README.md)
- [ ] Docker image configured (upstream Dockerfile via `workdir` or custom Dockerfile)
- [ ] `main.ts` defines daemons/oneshots
- [ ] `interfaces.ts` exposes network
- [ ] `init/` generates secrets on install (if needed)
- [ ] `actions/` for user operations (if needed)
- [ ] `fileModels/store.json.ts` for state (if needed)
- [ ] `npm run check` passes
- [ ] `make` succeeds
