# Project Structure

## Root Directory Layout

A StartOS package follows this organizational pattern:

```
my-service-startos/
├── assets/                 # Supplementary files (required, can be empty)
│   └── README.md
├── startos/                # Primary development directory
│   ├── actions/            # User-facing action scripts
│   ├── fileModels/         # Type-safe config file representations
│   ├── i18n/               # Internationalization
│   │   ├── index.ts        # setupI18n() call (boilerplate)
│   │   └── dictionaries/
│   │       ├── default.ts  # English strings keyed by index
│   │       └── translations.ts  # Translations for other locales
│   ├── init/               # Container initialization logic
│   ├── install/            # Version management and migrations
│   │   └── versions/
│   ├── manifest/           # Static service metadata
│   │   ├── index.ts        # setupManifest() call
│   │   └── i18n.ts         # Translated description, alerts
│   ├── backups.ts          # Backup volumes and exclusions
│   ├── dependencies.ts     # Service dependencies
│   ├── index.ts            # Exports (boilerplate)
│   ├── interfaces.ts       # Network interface definitions
│   ├── main.ts             # Daemon runtime and health checks
│   ├── sdk.ts              # SDK initialization (boilerplate)
│   └── utils.ts            # Package-specific utilities
├── .gitignore
├── Dockerfile              # Optional - for custom images
├── icon.svg                # Service icon (max 40 KiB)
├── LICENSE                 # Package license (symlink to upstream)
├── Makefile                # Project config (includes s9pk.mk)
├── s9pk.mk                 # Shared build logic (boilerplate)
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
- `Makefile` - Just includes `s9pk.mk` (see [Makefile](./makefile.md))
- `s9pk.mk` - Shared build logic, copy from template without modification
- `package.json` / `package-lock.json`
- `tsconfig.json`

### icon.svg

The service's visual identifier. Maximum size is 40 KiB. Accepts `.svg`, `.png`, `.jpg`, and `.webp` formats.

### LICENSE

The package's software license, ALWAYS matching the upstream service's license. Create a symlink:

```bash
ln -sf upstream-project/LICENSE LICENSE
```

### README.md

Documentation template that should be customized for your specific service.

## Key Directories

### assets/

Stores supplementary files and scripts needed by the service, such as configuration generators. **Required** - create with at least a README.md if empty.

### startos/

The primary development directory containing SDK integration files and package logic.

## Startos Directory Details

### Core TypeScript Modules

| File              | Purpose                                                               |
| ----------------- | --------------------------------------------------------------------- |
| `main.ts`         | Daemon runtime configuration and health checks                        |
| `interfaces.ts`   | Network interface definitions and port bindings                       |
| `backups.ts`      | Backup volumes and exclusion patterns                                 |
| `dependencies.ts` | Service dependencies and version requirements                         |
| `sdk.ts`          | SDK initialization (boilerplate)                                      |
| `utils.ts`        | Package-specific constants and helper functions                       |
| `index.ts`        | Module exports (boilerplate)                                          |

### Subdirectories

| Directory     | Purpose                                                               |
| ------------- | --------------------------------------------------------------------- |
| `actions/`    | Custom user-facing scripts displayed as buttons in the UI             |
| `fileModels/` | Type-safe representations of config files (.json, .yaml, .toml, etc.) |
| `i18n/`       | Internationalization: default dictionary and translated strings       |
| `init/`       | Container initialization logic (install, update, restart)             |
| `install/`    | Version management and migration logic                                |
| `manifest/`   | Service metadata (ID, name, description, images) with i18n            |

## Initialization Triggers

Service containers initialize in these scenarios:

- Fresh installation
- Service updates or downgrades
- StartOS system boot
- Manual rebuild by user
