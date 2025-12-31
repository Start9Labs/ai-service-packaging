# Environment Setup

## Prerequisites

You must have a computer running StartOS to test packages. Follow the [flashing guide](https://docs.start9.com/flashing-guides/) to install StartOS on a physical device or VM.

## Required Tools

### Docker

Docker is essential for building and managing container images that will be used for the final `.s9pk` build. It handles pulling base images and building custom container images from Dockerfiles.

**Installation**: Follow the [official Docker installation guide](https://docs.docker.com/engine/install/).

### Make

Build automation tool used to execute build scripts defined in Makefiles and coordinate the packaging workflow (building and installing s9pk binaries to StartOS).

**Installation**:
- **Linux (Debian-based)**: `sudo apt install build-essential`
- **macOS**: `xcode-select --install`

### Node.js v22 (Latest LTS)

Node.js is required for compiling TypeScript code used in StartOS package configurations.

**Installation**: Use [nvm](https://github.com/nvm-sh/nvm) or download from [nodejs.org](https://nodejs.org/).

```bash
# Using nvm
nvm install 22
nvm use 22
```

### SquashFS

Tool for creating compressed filesystem images for packaging compiled service code.

**Installation**:
- **Linux (Debian-based)**: `sudo apt install squashfs-tools squashfs-tools-ng`
- **macOS**: `brew install squashfs` (requires Homebrew)

### Start CLI

The core development toolkit that provides package validation, s9pk file creation, and development workflow management.

**Installation**: Run the automated installer:

```bash
curl -fsSL https://start9labs.github.io/start-cli | sh
```

## Verification

After installation, verify all tools are available:

```bash
docker --version
make --version
node --version
mksquashfs -version
start-cli --version
```
