---
name: run
description: >
  Use when the user wants to execute a lantern journey in a headed browser.
  Triggers: "run the journey", "lantern run", "show me", "preview this",
  "walk me through it", "run it", or when a journey has been authored and the
  user is ready to see it in action. Launches Playwright in headed mode with
  terminal narration and pauses at each checkpoint so the user can inspect
  the browser before continuing.
---

# Run a Lantern Journey

## Overview

This skill executes a lantern journey in a headed browser with terminal narration. The user watches the browser while the terminal provides context at each step. At every checkpoint, the browser pauses via Playwright Inspector so the user can inspect the state, then click resume to continue.

## When to Use

- After a journey has been authored with `lantern:author` and the user wants to see it
- The user wants to re-run an existing journey
- The user says "run it", "show me", "preview this", or "walk me through it" and a journey file exists
- As the final step in a build-then-verify workflow: implement a feature, author a journey, then run it

## Prerequisites

- Lantern has been scaffolded in the project (`lantern/` directory exists with `playwright.config.ts` and `utils.ts`)
- A journey file exists in `lantern/journeys/`
- The project's dev servers are running and reachable at the configured `baseURL`

## Process

### 1. Select the Journey

If the user specifies a journey file, use it directly.

If not, check `lantern/journeys/` for available journey files. If there is only one, use it. If there are multiple, identify the most recently modified file and confirm with the user:

> "Found these journeys: [list]. The most recently modified is `[filename]`. Run that one?"

### 2. Pre-flight Checks

Before launching the browser, verify the environment is ready.

**Check lantern directory structure:**

- Verify `lantern/playwright.config.ts` exists
- Verify `lantern/utils.ts` exists

If either is missing, stop and suggest running `lantern:scaffold` first.

**Check dev servers:**

First, determine how servers are managed by checking for `lantern/setup.ts` (or `lantern/global-setup.ts`):

- **Managed servers** (setup file exists and contains `webServer` config or server start/stop logic): The Playwright config handles server lifecycle automatically. Note this to the user: *"Servers will be started automatically by the test harness."* No connectivity check needed.
- **Preexisting servers** (no setup file, or setup file does not manage servers): Attempt a fetch to the `baseURL` configured in `lantern/playwright.config.ts` (default: `http://localhost:3000`). Check the config file for a custom `baseURL` or `BASE_URL` environment variable. If the server is not reachable, warn the user:

> "Servers don't appear to be running on [url]. Start your dev servers and try again."

Do **not** attempt to start servers yourself when using preexisting server configuration. The user manages their own dev environment.

**Check journey imports:**

Read the journey file and verify that any fragment imports resolve to existing files in `lantern/fragments/`. If a fragment is missing, warn the user:

> "Journey imports `[fragment]` but that file doesn't exist. Run `lantern:author` to create it, or remove the import."

### 3. Run the Journey

Execute the journey with Playwright:

```sh
npx playwright test --config lantern/playwright.config.ts lantern/journeys/[filename]
```

Run this command from the project root.

What happens during the run:

- The browser opens in headed mode (non-headless, as configured in `playwright.config.ts`)
- The terminal displays narration messages as the test progresses
- At each `checkpoint()` call, the terminal shows the checkpoint label and description, then `page.pause()` activates the Playwright Inspector
- The user inspects the browser, then clicks the resume button in the Playwright Inspector to continue
- The journey proceeds through all checkpoints until completion

Let the test run to completion. Do not interrupt it.

### 4. After the Run

**If the test passed** (all checkpoints were resumed and the journey completed without errors):

Report success. Something like:

> "Journey completed successfully. All [N] checkpoints passed."

**If the test failed** (Playwright reported an error):

Do **not** ask the user to debug. Handle it automatically:

1. **Inform the user:** *"The journey hit an error. Debugging headlessly — one moment."*

2. **Reproduce headlessly:** Re-run the journey in headless mode to capture the full error output without making the user sit through it:

   ```sh
   LANTERN_HEADLESS=1 npx playwright test --config lantern/playwright.config.ts lantern/journeys/[filename]
   ```

3. **Analyze and fix:** Read the error output and diagnose the issue. Common patterns:

   - Element not found → selector may be wrong, or the page did not load as expected
   - Navigation timeout → server may be slow or the URL is incorrect
   - Assertion error → the page state did not match expectations

   Fix the journey file or fragment code as needed.

4. **Verify headlessly:** Re-run in headless mode (`LANTERN_HEADLESS=1`) to confirm the fix works. If it still fails, repeat the analyze-and-fix cycle.

5. **Re-launch headed:** Once the test passes headlessly, re-run in normal headed mode so the user sees the working journey in the browser.

The key principle: the user should never sit through debugging. The agent handles all debugging iterations in headless mode and only re-launches the headed browser once everything works.

If the fix requires changes outside the journey/fragment code (e.g. the app itself has a bug), inform the user what was found and suggest next steps instead of modifying application code.

## Troubleshooting

**"Browser didn't open"**
Check that `headless: false` is set in `lantern/playwright.config.ts` under `use`. If it is set to `true` or missing, update it.

**"Cannot find module '../fragments/...'"**
The journey references a fragment file that does not exist. Either create the fragment with `lantern:author` or remove the import from the journey file.

**"page.pause() not working"**
Playwright Inspector requires a headed browser. Verify that `headless: false` is set in the Playwright config. Also ensure `PWDEBUG` is not set to `0` in the environment, as that disables the inspector.

**"Test timed out"**
The default timeout may be too short for journeys with many checkpoints (each checkpoint waits for user interaction). Increase the timeout in `lantern/playwright.config.ts`:

```typescript
export default defineConfig({
  timeout: 0, // no timeout — user controls pacing via checkpoints
  // ...
});
```

Also check that dev servers are responding — a slow or unresponsive server can cause navigation timeouts before the first checkpoint is reached.

**"Error: browserType.launch: Executable doesn't exist"**
Chromium is not installed. Run:

```sh
npx playwright install chromium
```
