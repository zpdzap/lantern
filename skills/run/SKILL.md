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

This skill executes a lantern journey in a headed browser with terminal narration. The user watches the browser while the terminal provides context at each step. At every checkpoint, an in-page overlay appears showing the checkpoint label, message, and a Continue button. The user can drag the overlay out of the way, inspect the page, then click Continue to advance.

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

Execute the journey with Playwright using the local binary (avoids `npx` startup overhead):

```sh
./node_modules/.bin/playwright test --config lantern/playwright.config.ts --project=chromium lantern/journeys/[filename]
```

Run this command from the project root. The `--project=chromium` flag skips project matching and goes straight to launch.

**Note on path resolution:** The Playwright config sets `testDir: './journeys'` (relative to the config file). When passing a journey file on the CLI, use the full path from the project root (`lantern/journeys/[filename]`). Playwright resolves the CLI path independently of `testDir`, so both work together correctly. Do **not** pass just the filename without the directory prefix.

What happens during the run:

- The browser opens in headed mode (non-headless, as configured in `playwright.config.ts`)
- The terminal displays narration messages as the test progresses
- At each `checkpoint()` call, the terminal shows the checkpoint label and description, and a draggable overlay appears on the page with the label, message, and a Continue button
- The user inspects the browser (dragging the overlay out of the way if needed), then clicks Continue in the overlay to advance
- The journey proceeds through all checkpoints until completion

**Skipping to a specific checkpoint:** If the user only wants to see a particular checkpoint (e.g., "just show me checkpoint 3"), use Playwright's `--grep` flag with the checkpoint label:

```sh
./node_modules/.bin/playwright test --config lantern/playwright.config.ts --project=chromium --grep "Validation errors" lantern/journeys/[filename]
```

This only works if the journey is structured as multiple `test()` blocks. For single-test journeys (the default template), suggest the user that you can temporarily comment out earlier checkpoints for a faster iteration cycle, then restore them before committing.

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
   LANTERN_HEADLESS=1 ./node_modules/.bin/playwright test --config lantern/playwright.config.ts --project=chromium lantern/journeys/[filename]
   ```

3. **Analyze and fix:** Read the error output and diagnose the issue. Common patterns:

   - Element not found → selector may be wrong, or the page did not load as expected
   - Navigation timeout → server may be slow or the URL is incorrect
   - Assertion error → the page state did not match expectations

   Fix the journey file or fragment code as needed.

4. **Verify headlessly:** Re-run in headless mode (`LANTERN_HEADLESS=1`) to confirm the fix works. If it still fails, repeat the analyze-and-fix cycle.

5. **Maximum 3 debug attempts.** If the journey still fails after 3 fix-and-retry cycles, stop and inform the user:

   > "The journey is still failing after 3 debug attempts. Here's the latest error: [error]. This may be an issue with the app rather than the journey. Here's what I tried: [summary of fixes attempted]."

   Do not loop indefinitely. Let the user decide how to proceed.

6. **Re-launch headed:** Once the test passes headlessly, re-run in normal headed mode so the user sees the working journey in the browser.

The key principle: the user should never sit through debugging. The agent handles all debugging iterations in headless mode and only re-launches the headed browser once everything works.

If the fix requires changes outside the journey/fragment code (e.g. the app itself has a bug), inform the user what was found and suggest next steps instead of modifying application code.

## Troubleshooting

**"Browser didn't open"**
Check that `headless: false` is set in `lantern/playwright.config.ts` under `use`. If it is set to `true` or missing, update it.

**"Cannot find module '../fragments/...'"**
The journey references a fragment file that does not exist. Either create the fragment with `lantern:author` or remove the import from the journey file.

**"Overlay not appearing"**
The in-page overlay requires a headed browser. Verify that `headless: false` is set in the Playwright config (this is the default when `LANTERN_HEADLESS` is not set). If the page navigates away immediately after a checkpoint is expected, the overlay may be injected and then lost — check that there are no competing navigations.

**"Test timed out"**
The default config uses `timeout: 0` in headed mode (no timeout, since checkpoints wait for the user to click Continue in the overlay) and `timeout: 30_000` in headless mode. If you're hitting timeouts in headed mode, verify that `LANTERN_HEADLESS` is not set in your environment. For headless pre-flight timeouts, check that dev servers are responding — a slow or unresponsive server can cause navigation timeouts before the first checkpoint is reached.

**"Error: browserType.launch: Executable doesn't exist"**
Chromium is not installed. Run:

```sh
npx playwright install chromium
```
