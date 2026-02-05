# Makefile Build System

StartOS packages use a two-file Makefile system that separates reusable build logic from project-specific configuration.

## File Structure

```
my-service-startos/
├── Makefile     # Project-specific configuration (minimal)
└── s9pk.mk      # Shared build logic (copy from template)
```

## s9pk.mk

The `s9pk.mk` file contains all the common build logic shared across StartOS packages. Copy this file from `hello-world-startos/s9pk.mk` without modification.

### What It Provides

| Target | Description |
|--------|-------------|
| `make` or `make all` | Build for all architectures (default) |
| `make x86` | Build for x86_64 only |
| `make arm` | Build for aarch64 only |
| `make riscv` | Build for riscv64 only |
| `make universal` | Build a single package containing all architectures |
| `make install` | Install the most recent .s9pk to your StartOS server |
| `make clean` | Remove build artifacts |

### Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `ALL_ARCHES` | `x86 arm riscv` | Architectures to build by default |
| `ALL_TARGETS` | `all_arches` | Default build target |
| `VARIANT` | (unset) | Optional variant suffix for package name |

## Makefile

The project `Makefile` is minimal and just includes `s9pk.mk`:

```makefile
include s9pk.mk
```

### Adding Custom Targets

For services with variants (e.g., GPU support), extend the Makefile:

```makefile
ALL_TARGETS := generic rocm
ALL_ARCHES := x86 arm

include s9pk.mk

.PHONY: generic rocm

generic:
	$(MAKE) all_arches VARIANT=generic

rocm:
	ROCM=1 $(MAKE) all_arches VARIANT=rocm ALL_ARCHES=x86_64
```

This produces packages named `myservice_generic_x86_64.s9pk` and `myservice_rocm_x86_64.s9pk`.

### Overriding Defaults

Override variables before including s9pk.mk:

```makefile
# Build only for x86 and arm by default
ALL_ARCHES := x86 arm

include s9pk.mk
```

## Build Commands

```bash
# Build for all architectures
make

# Build for a specific architecture
make x86
make arm

# Install to StartOS server (requires ~/.startos/config.yaml)
make install

# Clean build artifacts
make clean
```

## Prerequisites

The build system checks for:
- `start-cli` - StartOS CLI tool
- `npm` - Node.js package manager
- `~/.startos/developer.key.pem` - Developer key (auto-initialized if missing)

## Installation

Configure your StartOS server in `~/.startos/config.yaml`:

```yaml
host: http://your-server.local
```

Then run `make install` to build and install the package.
