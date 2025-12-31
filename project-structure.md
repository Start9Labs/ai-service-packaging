# Project Structure

## Root Directory Layout

A StartOS package follows this organizational pattern:

```
my-service-startos/
├── assets/                 # Supplementary files (required, can be empty)
│   └── README.md
├── docs/                   # Optional service documentation
│   └── instructions.md
├── startos/                # Primary development directory
│   ├── actions/            # User-facing action scripts
│   ├── fileModels/         # Type-safe config file representations
│   ├── init/               # Container initialization logic
│   ├── install/            # Version management and migrations
│   │   └── versions/
│   ├── backups.ts          # Backup volumes and exclusions
│   ├── dependencies.ts     # Service dependencies
│   ├── index.ts            # Exports (boilerplate)
│   ├── interfaces.ts       # Network interface definitions
│   ├── main.ts             # Daemon runtime and health checks
│   ├── manifest.ts         # Static service metadata
│   ├── sdk.ts              # SDK initialization (boilerplate)
│   └── utils.ts            # Package-specific utilities
├── .gitignore
├── Dockerfile              # Optional - for custom images
├── icon.svg                # Service icon (max 40 KiB)
├── LICENSE                 # Package license (symlink to upstream)
├── Makefile
├── package.json
├── package-lock.json
├── README.md
├── tsconfig.json
└── upstream-project/       # Git submodule (optional)
```

## Core Files

### Boilerplate Files

These files typically require minimal modification:
- `.gitignore`
- `Makefile`
- `package.json` / `package-lock.json`
- `tsconfig.json`

### icon.svg

The service's visual identifier. Maximum size is 40 KiB. Accepts `.svg`, `.png`, `.jpg`, and `.webp` formats.

### LICENSE

The package's software license, typically matching the upstream service's license. Create a symlink:

```bash
ln -sf upstream-project/LICENSE LICENSE
```

### README.md

Documentation template that should be customized for your specific service.

## Key Directories

### assets/

Stores supplementary files and scripts needed by the service, such as configuration generators. **Required** - create with at least a README.md if empty.

### docs/

Optional directory for service documentation. Can include an `instructions.md` file that is displayed to users.

### startos/

The primary development directory containing SDK integration files and package logic.

## Startos Directory Details

### Core TypeScript Modules

| File | Purpose |
|------|---------|
| `manifest.ts` | Static service metadata (ID, name, description, requirements, images) |
| `main.ts` | Daemon runtime configuration and health checks |
| `interfaces.ts` | Network interface definitions and port bindings |
| `backups.ts` | Backup volumes and exclusion patterns |
| `dependencies.ts` | Service dependencies and version requirements |
| `sdk.ts` | SDK initialization (boilerplate) |
| `utils.ts` | Package-specific constants and helper functions |
| `index.ts` | Module exports (boilerplate) |

### Subdirectories

| Directory | Purpose |
|-----------|---------|
| `actions/` | Custom user-facing scripts displayed as buttons in the UI |
| `fileModels/` | Type-safe representations of config files (.json, .yaml, .toml, etc.) |
| `init/` | Container initialization logic (install, update, restart) |
| `install/` | Version management and migration logic |

## Initialization Triggers

Service containers initialize in these scenarios:
- Fresh installation
- Service updates or downgrades
- StartOS system boot
- Manual rebuild by user
