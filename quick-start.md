# Quick Start

## Prerequisites

Complete the [environment setup](./environment-setup.md) before beginning.

## Creating Your Package Repository

### 1. Use the Hello World Template

1. Navigate to the [Hello World Template](https://github.com/Start9Labs/hello-world-startos) on GitHub
2. Click **"Use this template > Create new repository"** (requires GitHub login)
3. Name your repository as `[service-name]-startos` (e.g., `nextcloud-startos`)
4. Add description: "StartOS package for [Service Name]"
5. Ensure the repository is **public**
6. Click **"Create Repository"**

### 2. Clone and Initialize

```bash
git clone [your-repository-url]
cd [repository-name]
npm install
```

### 3. Add Upstream Project (Optional)

If wrapping an existing project, add it as a git submodule:

```bash
git submodule add https://github.com/user/project.git upstream-project
```

## Building Your Package

Run the build command:

```bash
make
```

This generates a `[service-id].s9pk` file in your project root.

## Installing to StartOS

### Option 1: Sideload via UI

1. Open your StartOS web interface
2. Navigate to **Marketplace > Sideload**
3. Upload the `.s9pk` file

### Option 2: Direct Install (Local Network)

If your StartOS device is accessible on your local network:

```bash
make install
```

This requires the `START_HOSTNAME` environment variable to be set, or you can modify the Makefile.

## Development Workflow

```bash
# TypeScript type checking
npm run check

# Build JS bundle only
npm run build

# Full package build
make

# Install to local StartOS
make install
```

## Next Steps

1. Review [project structure](./project-structure.md) to understand file layout
2. Update `startos/manifest/` with your service metadata and translations
3. Configure `main.ts` with daemons and health checks
4. Set up `interfaces.ts` for network exposure
5. Add i18n dictionaries in `startos/i18n/dictionaries/`
6. Add actions in `actions/` directory
7. Test on StartOS
