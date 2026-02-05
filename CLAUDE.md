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
- [Cross-Service Dependencies](./cross-service-dependencies.md) - Dependency tasks, interface reading, volume mounting
- [Makefile](./makefile.md) - Build system with s9pk.mk

Reference `hello-world-startos/` for boilerplate files (`package.json`, `tsconfig.json`, `Makefile`, `s9pk.mk`, `startos/` structure).

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
├── Makefile                # Project-specific config (just includes s9pk.mk)
├── s9pk.mk                 # Shared build logic (boilerplate)
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

| Field                               | Description                                     |
| ----------------------------------- | ----------------------------------------------- |
| `id`, `title`, `license`, `docsUrl` | Required metadata                               |
| `volumes`                           | Storage volumes (usually `['main']`)            |
| `images`                            | Docker images (pre-built tag or local build)    |
| `alerts`                            | User notifications for install/update/uninstall |
| `dependencies`                      | Service dependencies                            |

See [manifest.ts](./manifest-ts.md) for detailed configuration (images, alerts, dependencies, license/icon setup).

### main.ts - Runtime Configuration

**When**: Always - defines how the service runs.

| API                                                          | When to Use                                                       |
| ------------------------------------------------------------ | ----------------------------------------------------------------- |
| `storeJson.read((s) => s).const(effects)`                    | Read config reactively (restarts service on change)               |
| `storeJson.read((s) => s).once()`                            | Read config once (no restart on change)                           |
| `sdk.serviceInterface.getOwn(effects, 'ui', mapper).const()` | Get service hostnames (with mapper to avoid unnecessary restarts) |
| `sdk.SubContainer.of(effects, {imageId}, mounts, name)`      | Create container with volume mounts                               |
| `sdk.useEntrypoint()`                                        | Use upstream image's ENTRYPOINT/CMD (prefer over custom command)  |
| `sdk.useEntrypoint([args])`                                  | Use upstream ENTRYPOINT with custom CMD arguments                 |
| `writeFile(\`${appSub.rootfs}/path\`, content)`              | Write ephemeral config to subcontainer rootfs                     |
| `sdk.volumes.main`                                           | Volume object for the 'main' volume (implements PathBase)         |
| `sdk.volumes.main.subpath('file.txt')`                       | Get absolute path to file within volume                           |
| `sdk.volumes.main.readFile('file.txt')`                      | Read file from volume (returns Buffer or string)                  |
| `sdk.volumes.main.writeFile('file.txt', data)`               | Write file to volume (creates parent dirs automatically)          |
| `writeFile(\`${appSub.rootfs}/path\`, content)`              | Write ephemeral config to subcontainer rootfs                     |
| `sdk.Daemons.of(effects).addOneshot(...)`                    | One-time setup tasks (migrations, etc.)                           |
| `sdk.Daemons.of(effects).addDaemon(...)`                     | Long-running processes                                            |
| `sdk.healthCheck.checkPortListening(effects, port, msgs)`    | Health check for daemon readiness                                 |

See [main.ts patterns](./main-ts.md) for detailed examples.

### interfaces.ts

**When**: Service exposes network interfaces (web UI, API, etc.).

| API                                          | When to Use                         |
| -------------------------------------------- | ----------------------------------- |
| `sdk.MultiHost.of(effects, 'ui-multi')`      | Create network binding              |
| `uiMulti.bindPort(port, {protocol: 'http'})` | Bind to a port                      |
| `sdk.createInterface(effects, {...})`        | Define an interface (UI, API, etc.) |
| `origin.export([...interfaces])`             | Export interfaces                   |

See [interfaces.ts patterns](./interfaces-ts.md) for multiple interfaces.

### actions/

**When**: Users need to trigger operations (get credentials, reset password, etc.).

| API                                                      | When to Use                 |
| -------------------------------------------------------- | --------------------------- |
| `sdk.Action.withoutInput(id, metadata, handler)`         | Action with no user input   |
| `sdk.Action.withInput(id, inputSpec, metadata, handler)` | Action requiring user input |
| `sdk.Actions.of().addAction(...)`                        | Register actions            |

See [actions.md](./actions.md) for action patterns.

### dependencies.ts - Cross-Service Dependencies

**When**: Service depends on another StartOS service.

| API                                                                           | When to Use                                     |
| ----------------------------------------------------------------------------- | ----------------------------------------------- |
| `sdk.action.createTask(effects, packageId, action, severity, options)`        | Trigger an action on a dependency service        |
| `sdk.serviceInterface.get(effects, {id, packageId}, mapper).const()`          | Read a dependency's interface URL reactively     |
| `sdk.Mounts.of().mountDependency({dependencyId, volumeId, ...})`             | Mount a dependency's volume for file access      |

See [Cross-Service Dependencies](./cross-service-dependencies.md) for patterns and examples.

### init/initializeService.ts

**When**: Need one-time setup on install (generate secrets, bootstrap via API, create tasks).

| API                                                             | When to Use                                  |
| --------------------------------------------------------------- | -------------------------------------------- |
| `sdk.setupOnInit(async (effects, kind) => {...})`               | Run code on init                             |
| `storeJson.write(effects, {...})`                               | Persist initial state                        |
| `sdk.action.createOwnTask(effects, action, priority, {reason})` | Prompt user to run action                    |
| `utils.getDefaultString({charset, len})`                        | Generate random strings                      |
| `.runUntilSuccess(timeout)`                                     | Run daemons/oneshots and wait for completion |

See [Initialization Patterns](./init.md) for `runUntilSuccess` and bootstrapping via API.

### fileModels/store.json.ts

**When**: Need to persist service state (passwords, secrets, settings).

| API                                           | When to Use                      |
| --------------------------------------------- | -------------------------------- |
| `FileHelper.json({base: sdk.volumes.main, subpath}, shape)` | JSON file with schema validation |
| `matches.object({...})`                                     | Define shape with `matches`      |

## Writing Files

- **Subcontainer rootfs** (`${appSub.rootfs}/path`): For ephemeral config regenerated on startup
- **Volume via sdk.volumes** (`sdk.volumes.main.writeFile('file.txt', data)`): For persistent data that survives restarts
- **Volume via FileHelper** (`FileHelper.json({base: sdk.volumes.main, subpath}, shape)`): For type-safe config files on volumes
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

## Code Style Guidelines

### Formatting

Use Prettier for all TypeScript files. Configuration lives in `package.json`:

```json
{
  "prettier": {
    "trailingComma": "all",
    "tabWidth": 2,
    "semi": false,
    "singleQuote": true
  }
}
```

Run `npm run prettier` before committing.

Key rules:
- No semicolons
- Single quotes
- Trailing commas everywhere
- 2-space indentation

### TypeScript

- Enable `strict: true` in `tsconfig.json`
- Use `const` exports with arrow functions (e.g. `export const main = sdk.setupMain(async (...) => { ... })`)
- Prefer `const` over `let`; never use `var`
- Use the SDK's type system — don't cast with `as` or use `any` unless absolutely necessary

### Imports

- SDK/library imports first, then local imports
- Use relative paths for local imports (e.g. `'./sdk'`, `'../actions'`)

### Documentation & Comments

- Keep comments focused on "why" rather than "what"
- Don't add comments that just restate the code
- Mark boilerplate files with `/** Plumbing. DO NOT EDIT. */` so packagers know what to leave alone
- Update or remove comments when code changes

### Naming

- Files: `camelCase.ts` for code, `kebab-case.md` for docs
- Version files: `v_MAJOR_MINOR_PATCH_PREMAJOR_PREMINOR_PREPATCH.ts` (underscores, matching ExVer)
- Variables/functions: `camelCase`
- Types/interfaces: `PascalCase`
- Constants (ports, config keys): `camelCase` (e.g. `uiPort`, not `UI_PORT`)

### Commit Messages

Use [Conventional Commits](https://www.conventionalcommits.org/):

```
<type>(<scope>): <description>
```

**Types:** `feat`, `fix`, `docs`, `style`, `refactor`, `test`, `chore`

**Scope** is the service name (e.g. `hello-world`, `nextcloud`, `bitcoin`).

**Examples:**
```
feat(nextcloud): add smtp configuration action
fix(bitcoin): resolve health check timeout
chore(hello-world): bump SDK to 0.4.0-beta.48
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

## Supplementary Files

These files live at the **session root** (the directory where Claude is launched, i.e. the parent directory that contains this repo), not inside this repo:

- `TODO.md` - Pending tasks for AI agents (check this first, remove items when completed)
- `USER.md` - Current user identifier (gitignored, varies per developer)

### Session Startup

On startup:

1. **Check for `USER.md` at the session root** - If it doesn't exist, prompt the user for their name/identifier and create it there. This file is gitignored since it varies per developer.

2. **Check `TODO.md` at the session root for relevant tasks** - Show TODOs that either:
   - Have no `@username` tag (relevant to everyone)
   - Are tagged with the current user's identifier

   Skip TODOs tagged with a different user.

3. **Ask "What would you like to do today?"** - Offer options:
   - Each relevant TODO item
   - **"Package a new service"** (see below)
   - "Something else"

### Package a New Service

The user can request a new package in 4 ways. Each entry point resolves to a specific open-source project before moving on to the shared workflow below.

#### Entry Point 1: Repository URL

The user provides a URL to an open-source repo (e.g. `https://github.com/user/project`).

- Use the URL directly as the upstream repo.
- Research the repo (README, Dockerfile, docker-compose, docs) to gather the required info below.

#### Entry Point 2: Project name (open source)

The user gives the name of a known open-source self-hosted project (e.g. "Nextcloud", "Gitea").

- Search for the project on GitHub/GitLab/etc. to find the canonical repo.
- If ambiguous (multiple projects with similar names), prompt the user to choose.
- Once the repo is identified, proceed as Entry Point 1.

#### Entry Point 3: Product name (closed source)

The user names a closed-source product they want a self-hosted alternative to (e.g. "Google Docs", "Slack").

- Search for open-source, self-hostable alternatives to that product.
- If multiple viable options exist, present them to the user with brief descriptions and let them choose.
- Once the user picks a project, proceed as Entry Point 1.

#### Entry Point 4: Description of need

The user describes what they need in general terms (e.g. "something that manages my bookmarks", "a private photo gallery").

- Search for open-source, self-hostable projects that fit the description.
- Present the best candidates to the user with brief descriptions and let them choose.
- Once the user picks a project, proceed as Entry Point 1.

#### Gathering Required Info

Once an upstream repo is identified, gather the following information. Research the repo first (README, Dockerfile, docker-compose files, docs, environment variables) — extract as much as you can automatically and only ask the user about what's still unclear.

**Required info:**
1. **Service name** - What is the service called?
2. **Upstream repo** - URL to the source code (e.g. GitHub)
3. **What it does** - Brief description for the manifest
4. **License** - Check the upstream repo's LICENSE file for the SPDX identifier
5. **Docker image** - Does the upstream project publish a Docker image (e.g. on Docker Hub / GHCR), or does the repo contain a Dockerfile, or do we need to write one?
6. **Ports** - What port(s) does the service listen on? Which are web UIs vs APIs?
7. **Configuration** - Does the service need config files? Environment variables? Command-line flags?
8. **Persistent data** - Where does the service store its data? (database path, data directory, etc.)
9. **Authentication** - Does the service have its own auth (admin password, API keys)? How is it initially configured?
10. **Dependencies** - Does it depend on other StartOS services (e.g. PostgreSQL, Bitcoin)?

**Then follow this workflow:**
1. Clone the hello-world-startos template into a new `[service-name]-startos/` directory (sibling to `service-packaging/`)
2. Add the upstream project as a git submodule
3. Work through each file in the project structure using the documentation in this guide:
   - `manifest.ts` — fill in metadata, images, license, dependencies ([manifest-ts.md](./manifest-ts.md))
   - `main.ts` — set up daemons, health checks, volume mounts, config file generation ([main-ts.md](./main-ts.md))
   - `interfaces.ts` — expose ports and interfaces ([interfaces-ts.md](./interfaces-ts.md))
   - `init/` — generate secrets, bootstrap state on install ([init.md](./init.md))
   - `actions/` — add user actions like "Get Credentials" ([actions.md](./actions.md))
   - `fileModels/store.json.ts` — define persistent state shape ([file-models.md](./file-models.md))
   - `Dockerfile` — write one if upstream doesn't provide a suitable image ([manifest-ts.md](./manifest-ts.md))
   - Symlink `LICENSE` and `icon.*` from upstream
4. Run `npm run check` and fix any type errors
5. Run `make` to build the `.s9pk`
6. Walk through the [Checklist](#checklist) to verify nothing was missed
