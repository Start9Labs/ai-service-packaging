# AI Service Packaging

An AI-assisted toolkit for packaging self-hosted services for [StartOS](https://start9.com). This repo contains documentation, templates, and a `CLAUDE.md` that teaches Claude Code how to package any open-source project as a StartOS `.s9pk`.

## How It Works

When you run Claude Code in a directory that references this repo's `CLAUDE.md`, it learns the full StartOS packaging workflow — manifest configuration, daemon setup, networking, actions, dependencies, and more. You just tell it what service you want to package.

## Getting Started

1. Create a directory for your services and `cd` into it:

   ```sh
   mkdir services && cd services
   ```

2. Clone this repo:

   ```sh
   git clone https://github.com/Start9Labs/service-packaging.git ai-service-packaging
   ```

3. Create a `CLAUDE.md` in your services directory:

   ```sh
   echo 'read ai-service-packaging/CLAUDE.md' > CLAUDE.md
   ```

4. Run Claude Code and say hi:

   ```sh
   claude
   ```

   ```
   > hi
   ```

Claude will introduce itself, check for any pending tasks, and ask what you'd like to do — including packaging a new service. You can give it a repo URL, a project name, a product you want a self-hosted alternative to, or just describe what you need.
