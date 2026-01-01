# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Build and Development Commands

Always use Node.js/npm instead of Bun.

### Skill (Core Server/Client)

```bash
cd skills/dev-browser

npm install                    # Install dependencies
npm run start-server           # Start the dev-browser server (launch mode)
npm run start-extension        # Start relay server (extension mode)
npm run dev                    # Run dev mode with watch
npm test                       # Run tests (vitest)
npx tsc --noEmit               # TypeScript check
```

### Chrome Extension

```bash
cd extension

npm install                    # Install dependencies
npm run dev                    # Dev mode with hot reload
npm run build                  # Build for production
npm run zip                    # Create distributable zip
npm test                       # Run tests (vitest)
```

### Root (Formatting)

```bash
npm run format                 # Format all files with Prettier
npm run format:check           # Check formatting
```

## Before Completing Code Changes

**Always run these checks:**

1. **TypeScript check**: `npx tsc --noEmit` - Ensure no type errors
2. **Tests**: `npm test` - Ensure all tests pass

Common TypeScript issues:

- Use `import type { ... }` for type-only imports (required by `verbatimModuleSyntax`)
- Browser globals in `page.evaluate()` callbacks: use `globalThis as any` pattern since DOM lib is not included

## Project Architecture

### Overview

A browser automation tool for developers and AI agents that maintains browser state across script executions. Unlike typical Playwright scripts that start fresh, dev-browser keeps pages alive and reusable.

### Two Operating Modes

1. **Launch Mode** (default): Launches a persistent Chromium instance
2. **Extension Mode**: Controls the user's existing Chrome browser via a Chrome extension

### Directory Structure

```
skills/dev-browser/     # Core automation skill
  src/
    index.ts            # Launch mode server (Express + Playwright)
    relay.ts            # Extension mode relay server (Hono + WebSocket)
    client.ts           # Client library for connecting to either mode
    types.ts            # Shared TypeScript types
    snapshot/           # DOM snapshot utilities for LLM-friendly page inspection

extension/              # Chrome extension (WXT framework)
  entrypoints/
    background.ts       # Service worker - main extension logic
    popup/              # Extension popup UI
  services/
    ConnectionManager.ts  # WebSocket connection to relay
    CDPRouter.ts         # Routes CDP commands to Chrome debugger API
    TabManager.ts        # Tracks attached tabs/sessions
    StateManager.ts      # Persists extension state
```

### Path Aliases

The skill uses `@/` as a path alias to `./src/`:

```typescript
import { connect } from "@/client.js";
import { serve } from "@/index.js";
```

### How It Works

**Launch Mode** (`serve()` in `src/index.ts`):
- Launches Chromium with `launchPersistentContext` (preserves cookies, localStorage)
- HTTP API on port 9222, CDP WebSocket on port 9223
- Pages registered by name persist until closed

**Extension Mode** (`serveRelay()` in `src/relay.ts`):
- Relay server bridges Playwright clients and Chrome extension
- Extension connects via WebSocket, receives CDP commands
- Uses Chrome Debugger API to control user's existing tabs

**Client** (`connect()` in `src/client.ts`):
- Works with both modes transparently
- Uses CDP `targetId` for reliable page lookup
- Returns standard Playwright `Page` objects

### Key API Endpoints

- `GET /` - Server info (wsEndpoint, mode, extensionConnected)
- `GET /pages` - List named pages
- `POST /pages` - Get or create page by name (`{ name: string }`)
- `DELETE /pages/:name` - Close a page

### Usage Pattern

```typescript
import { connect } from "@/client.js";

const client = await connect("http://localhost:9222");
const page = await client.page("my-page");
await page.goto("https://example.com");
// Page persists for future scripts
await client.disconnect();
```

## Node.js Guidelines

- Use `npx tsx` for running TypeScript files
- Use `node:fs` for file system operations
