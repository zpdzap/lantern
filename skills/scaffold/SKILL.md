---
name: scaffold
description: >
  Use when the user wants to set up lantern in a project for the first time.
  Triggers: first mention of lantern in a project, "set up lantern", "scaffold
  lantern", "I want to preview features in the browser", or no lantern directory
  detected. Also handles re-scaffolding: "re-scaffold lantern", "update the
  lantern setup", "refresh lantern config". Creates a lantern directory with
  Playwright config, utilities, and directory structure for fragments and
  journeys. Lantern is separate from the project's test suite — it drives a
  headed browser for visual verification.
---

# Scaffold Lantern

## Overview

Lantern is a system for driving headed browsers through web apps so developers can visually verify features without manual click-through. It is **not** a test framework and does not run in CI — it is a local, human-driven tool for visual verification.

This skill performs one-time project setup. It creates a `lantern/` directory containing:

- A Playwright config tuned for headed, single-browser operation
- Utility functions (`narrate` and `checkpoint`) used by journeys
- Empty `fragments/` and `journeys/` directories ready for content
- A local README explaining the setup

The lantern directory lives alongside the project's code but is completely separate from any existing test suite.

## When to Use

- First time lantern is mentioned in a project
- The user explicitly asks to scaffold or set up lantern
- No existing `lantern/` directory is detected in the project

## Process

### 1. Detect Project Structure

Before creating anything, understand the project layout:

- Check for monorepo patterns: look for a root `package.json` with `workspaces`, or directories like `apps/`, `packages/`.
- Check for existing e2e directories (`e2e/`, `tests/`, `test/`).
- Identify the frontend framework by inspecting `package.json` dependencies (React, Next.js, Vue, Svelte, etc.).
- Check for `tsconfig.json` to confirm TypeScript usage.
- Check for a `Makefile` at the project root.

### 2. Determine Lantern Directory Location

Place the `lantern/` directory using these heuristics:

- **Standard project with an `e2e/` directory:** place `lantern/` at the same level as `e2e/` (e.g., if `e2e/` is at the project root, `lantern/` goes at the project root).
- **Monorepo:** place `lantern/` inside the frontend app directory (e.g., `apps/web/lantern/`).
- **Simple project (no e2e, no monorepo):** place `lantern/` at the project root.
- **Ambiguous layout:** ask the user where they want it before proceeding.

### 3. Determine Server Management

Ask the user:

> How should lantern handle dev servers? Options:
> - **Managed:** Lantern starts and stops dev servers as part of test setup/teardown (you'll configure the start commands)
> - **Preexisting:** Lantern assumes dev servers are already running and just checks connectivity

**If managed:** Ask the user what commands start each server (backend, frontend) and what ports to expect. Create `lantern/setup.ts` that starts the servers before tests, waits for readiness, and tears them down after. Wire this into `playwright.config.ts` via the `globalSetup` option.

```typescript
// lantern/setup.ts — managed variant
import { spawn, ChildProcess } from 'child_process';
import { FullConfig } from '@playwright/test';

const servers: ChildProcess[] = [];

async function waitForReady(url: string, timeoutMs = 30_000): Promise<void> {
  const start = Date.now();
  while (Date.now() - start < timeoutMs) {
    try {
      const res = await fetch(url);
      if (res.ok) return;
    } catch {}
    await new Promise(r => setTimeout(r, 500));
  }
  throw new Error(`Server at ${url} not ready after ${timeoutMs}ms`);
}

async function globalSetup(config: FullConfig) {
  // Replace with actual start commands and ports from the user
  const backend = spawn('npm', ['run', 'dev:backend'], { stdio: 'pipe', shell: true });
  servers.push(backend);
  await waitForReady('http://localhost:8080/health');

  const frontend = spawn('npm', ['run', 'dev:frontend'], { stdio: 'pipe', shell: true });
  servers.push(frontend);
  await waitForReady(config.projects[0].use.baseURL || 'http://localhost:3000');
}

async function globalTeardown() {
  for (const proc of servers) {
    proc.kill('SIGTERM');
  }
}

export default globalSetup;
export { globalTeardown };
```

**If preexisting:** Create `lantern/setup.ts` that does a pre-flight connectivity check and fails with a clear message if the app isn't reachable.

```typescript
// lantern/setup.ts — preexisting variant
import { FullConfig } from '@playwright/test';

async function globalSetup(config: FullConfig) {
  const baseURL = config.projects[0].use.baseURL || 'http://localhost:3000';
  try {
    const res = await fetch(baseURL);
    if (!res.ok) {
      throw new Error(`Server responded with ${res.status}`);
    }
  } catch (err) {
    console.error(`\n  Lantern pre-flight check failed: ${baseURL} is not reachable.`);
    console.error(`  Start your dev servers before running lantern.\n`);
    process.exit(1);
  }
}

export default globalSetup;
```

In both cases, add `globalSetup: './setup.ts'` to the Playwright config (and `globalTeardown` for the managed variant).

### 4. Check for Playwright

Check if `@playwright/test` is already in the project's `devDependencies`.

If Playwright is **not** installed:

```sh
npm install -D @playwright/test
npx playwright install chromium
```

If Playwright **is** already installed, skip the npm install but still ensure chromium is available:

```sh
npx playwright install chromium
```

### 5. Create Playwright Config

Create `lantern/playwright.config.ts` with the contents from the Generated Code Reference section below.

### 6. Create Utilities

Create `lantern/utils.ts` with the contents from the Generated Code Reference section below.

### 7. Create Directory Structure

- Create `lantern/fragments/` with a `.gitkeep` file inside it.
- Create `lantern/journeys/` with a `.gitkeep` file inside it.

### 8. Create Local README

Create `lantern/README.md` with content along these lines:

```markdown
# Lantern

Local visual verification for this project. Lantern drives a headed browser
through your app so you can see features working without manual click-through.

## Running Journeys

```sh
# Run all journeys
npm run lantern

# Run a specific journey
npx playwright test --config lantern/playwright.config.ts lantern/journeys/my-journey.spec.ts
```

## Structure

- `journeys/` — end-to-end visual walkthroughs of features
- `fragments/` — reusable page interaction snippets
- `utils.ts` — narrate() and checkpoint() helpers

## Learn More

https://github.com/zpdzap/lantern
```

Adjust the run command if the project uses a Makefile instead of npm scripts.

### 9. Add a Run Script

If the project has a `package.json`, add a script entry:

```json
{
  "scripts": {
    "lantern": "npx playwright test --config lantern/playwright.config.ts"
  }
}
```

If the project uses a `Makefile`, add a target instead:

```makefile
.PHONY: lantern
lantern:
	npx playwright test --config lantern/playwright.config.ts
```

If both exist, add to both.

### 10. Commit the Scaffolded Files

Stage all the new lantern files and commit:

```
scaffold lantern for visual verification
```

Do not push — let the user decide when to push.

## Generated Code Reference

### lantern/playwright.config.ts

```typescript
import { defineConfig } from '@playwright/test';

export default defineConfig({
  testDir: './journeys',
  fullyParallel: false,
  workers: 1,
  retries: 0,
  reporter: 'list',
  use: {
    headless: process.env.LANTERN_HEADLESS === '1',
    baseURL: process.env.BASE_URL || 'http://localhost:3000',
    trace: 'off',
  },
  projects: [
    { name: 'chromium', use: { browserName: 'chromium' } },
  ],
});
```

### lantern/utils.ts

```typescript
import { Page } from '@playwright/test';

export function narrate(message: string): void {
  const divider = '-'.repeat(Math.min(message.length + 4, 60));
  console.log(`\n${divider}`);
  console.log(`  ${message}`);
  console.log(`${divider}\n`);
}

export async function checkpoint(
  page: Page,
  label: string,
  message?: string
): Promise<void> {
  const header = `[Checkpoint: ${label}]`;
  console.log(`\n${'='.repeat(60)}`);
  console.log(`  ${header}`);
  if (message) {
    console.log();
    message.split('\n').forEach(line => console.log(`  ${line}`));
  }
  console.log(`\n  Paused. Inspect the browser, then resume.`);
  console.log(`${'='.repeat(60)}\n`);
  await page.pause();
}
```

## Re-Scaffolding

The user can request re-scaffolding at any time. Triggers: "re-scaffold lantern", "update the lantern setup", "refresh lantern config".

When re-scaffolding, follow this process:

1. **Walk through the original setup steps again** — re-detect the project structure, confirm the lantern directory location, and re-ask about server management preferences.
2. **Compare current project structure with existing lantern setup** — check if the project layout has changed (new apps in a monorepo, moved directories, updated frameworks).
3. **Check existing journey/fragment files for consistency** — verify that existing fragments and journeys still match current project patterns (routes, selectors, component names).
4. **Determine what needs updating** — identify config changes (baseURL, server commands, ports), new fragments to extract based on new features, and setup.ts updates.
5. **Update only what's changed** — do not overwrite custom modifications to journeys, fragments, or utils without explicitly asking the user first. Show a summary of proposed changes and get confirmation before applying.

## Do NOT

- Touch existing e2e tests or test configuration
- Modify any existing Playwright config in the project
- Install browsers other than chromium
- Add CI configuration (lantern is local-only)
