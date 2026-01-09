# manifest.ts

The manifest defines service identity, metadata, and build configuration.

**When**: Always - defines service identity and metadata.

## Basic Structure

```typescript
import { setupManifest } from "@start9labs/start-sdk";

export const manifest = setupManifest({
  id: "my-service",
  title: "My Service",
  license: "MIT",
  wrapperRepo: "https://github.com/Start9Labs/my-service-startos",
  upstreamRepo: "https://github.com/original/my-service",
  supportSite: "https://docs.example.com/",
  marketingSite: "https://example.com/",
  donationUrl: null,
  docsUrl: "https://docs.example.com/guides",
  description: {
    short: "Brief description (one line)",
    long: "Longer description explaining what the service does and its key features.",
  },
  volumes: ["main"],
  images: {
    /* see below */
  },
  alerts: {
    install: null,
    update: null,
    uninstall: null,
    restore: null,
    start: null,
    stop: null,
  },
  dependencies: {},
});
```

## Required Fields

| Field               | Description                                            |
| ------------------- | ------------------------------------------------------ |
| `id`                | Unique identifier (lowercase, hyphens allowed)         |
| `title`             | Display name shown in UI                               |
| `license`           | SPDX identifier (`MIT`, `Apache-2.0`, `GPL-3.0`, etc.) |
| `wrapperRepo`       | URL to the StartOS wrapper repository                  |
| `upstreamRepo`      | URL to the original project repository                 |
| `supportSite`       | URL for user support                                   |
| `marketingSite`     | URL for the project's main website                     |
| `donationUrl`       | Donation URL or `null`                                 |
| `docsUrl`           | URL to documentation                                   |
| `description.short` | One-line description                                   |
| `description.long`  | Extended description                                   |
| `volumes`           | Storage volumes (usually `['main']`)                   |
| `images`            | Docker image configuration                             |
| `alerts`            | User notifications for lifecycle events                |
| `dependencies`      | Service dependencies                                   |

## License

Check the upstream project's LICENSE file and use the correct SPDX identifier (e.g., `MIT`, `Apache-2.0`, `GPL-3.0`). Create a symlink from your project root to the upstream license:

```bash
ln -sf upstream-project/LICENSE LICENSE
```

## Icon

Symlink from upstream if available (svg, png, jpg, or webp, max 40 KiB):

```bash
ln -sf upstream-project/logo.svg icon.svg
```

## Images Configuration

### Pre-built Docker Tag

Use when an image exists on Docker Hub or another registry:

```typescript
images: {
  main: {
    source: {
      dockerTag: 'nginx:1.25',
    },
  },
},
```

### Local Docker Build

Use when building from a Dockerfile in the project

```typescript
// Dockerfile in project root
images: {
  main: {
    source: {
      dockerBuild: {},
    },
  },
},
```

**If upstream has a working Dockerfile**: Set `workdir` to the upstream directory. If the Dockerfile is named `Dockerfile`, you can omit the `dockerfile` field:

```typescript
images: {
  main: {
    source: {
      dockerBuild: {
        workdir: './upstream-project',
      },
    },
  },
},
```

For a non-standard Dockerfile name, specify `dockerfile` relative to project root:

```typescript
images: {
  main: {
    source: {
      dockerBuild: {
        workdir: './upstream-project',
        dockerfile: './upstream-project/sync-server.Dockerfile',
      },
    },
  },
},
```

**If you need a custom Dockerfile**: Create one in your project root:

```dockerfile
COPY upstream-project/ .
```

### GPU/Hardware Acceleration

For services requiring GPU access:

```typescript
images: {
  main: {
    source: {
      dockerTag: 'ollama/ollama:0.13.5',
    },
    nvidiaContainer: true,  // Enable NVIDIA GPU support
  },
},
hardwareAcceleration: true,  // Top-level flag
```

### Multiple Images

Services can define multiple images:

```typescript
images: {
  app: {
    source: { dockerTag: 'myapp:latest' },
  },
  db: {
    source: { dockerTag: 'postgres:15' },
  },
},
```

## Alerts

Display messages to users during lifecycle events:

```typescript
alerts: {
  install: 'After installation, run the "Get Admin Credentials" action to retrieve your password.',
  update: 'This update includes breaking changes. Backup your data first.',
  uninstall: 'All data will be permanently deleted.',
  restore: null,
  start: null,
  stop: null,
},
```

Set to `null` for no alert.

## Dependencies

Declare dependencies on other StartOS services:

```typescript
dependencies: {
  // Required dependency
  bitcoin: {
    description: 'Required for blockchain data',
    optional: false,
  },

  // Optional dependency with metadata
  'c-lightning': {
    description: 'Needed for Lightning payments',
    optional: true,
    metadata: {
      title: 'Core Lightning',
      icon: 'https://raw.githubusercontent.com/Start9Labs/cln-startos/refs/heads/master/icon.png',
    },
  },
},
```

## Volumes

Storage volumes for persistent data. Usually just `['main']`:

```typescript
volumes: ['main'],
```

For services needing separate storage areas:

```typescript
volumes: ['main', 'db', 'config'],
```

Reference in `main.ts` mounts as `'main'`, `'db'`, `'config'`.
