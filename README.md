# Lantern

**Let your AI light the way through your app.**

Manual click-through for visual verification is tedious. Creating test accounts, navigating multi-step flows, getting to the right screen in the right state — it all adds up. Lantern gives your AI coding agent the ability to drive a headed browser through your web app, narrating what it finds and pausing at checkpoints so you can see your features working without lifting a finger.

## How It Works

Lantern is a set of AI agent skills that follow a three-phase workflow: **scaffold**, **author**, **run**.

1. **Scaffold** — One-time setup. Creates a `lantern/` directory in your project with a Playwright config, utility functions, and the directory structure for fragments and journeys.
2. **Author** — Your agent analyzes a ticket, diff, or conversation to understand what changed, then composes a journey file — a scripted walkthrough with narration and checkpoints.
3. **Run** — Executes the journey in a headed browser. You watch the browser while your terminal narrates each step. At every checkpoint, the browser pauses so you can inspect, then you click resume to continue.

**Example scenario:** You're working on a ticket that adds validation to the registration form. You ask your agent to "lantern me through the changes." The agent reads the ticket, writes a journey that signs up with invalid data to trigger the error states, and runs it. A browser opens, navigates to the registration page, submits bad input, and pauses — your terminal tells you exactly what to look at. You verify the error messages look right, click resume, and the journey continues to the next checkpoint.

## Skills Reference

### scaffold

Sets up lantern in a project for the first time. Creates the `lantern/` directory with a Playwright config tuned for headed single-browser operation, utility functions (`narrate` and `checkpoint`), and empty `fragments/` and `journeys/` directories ready for content. Also adds a run script to your `package.json` or `Makefile`.

During setup, the scaffold asks whether lantern should **manage** your dev servers (start/stop them automatically during test setup/teardown) or treat them as **preexisting** (just check connectivity before running). Managed mode is useful when you want a single command to bring everything up; preexisting mode is better when you already have servers running in another terminal.

You can also **re-scaffold** at any time to update your lantern setup when the project changes — new services, different ports, restructured directories. Re-scaffolding walks through the original setup steps again but only updates what has changed, leaving your custom journeys and fragments untouched.

**Example prompts:**
- "set up lantern"
- "scaffold lantern"
- "I want to preview features in the browser"
- "set up visual verification for this project"
- "lantern init"
- "add lantern to this project"
- "initialize lantern"
- "re-scaffold lantern"
- "update the lantern setup"
- "refresh lantern config"

**Produces:**
- `lantern/playwright.config.ts`
- `lantern/setup.ts` (server management — managed or preexisting)
- `lantern/utils.ts`
- `lantern/fragments/` directory
- `lantern/journeys/` directory
- `lantern/README.md`
- A `lantern` script in `package.json` and/or `Makefile`

### author

Analyzes a ticket, diff, or conversation context and composes a journey file — a headed-browser walkthrough with reusable fragments and tour/verify checkpoints. Surveys existing fragments to avoid duplication, plans the checkpoint sequence, and writes the journey as a TypeScript file that Playwright can execute.

Before committing, the author skill type-checks the journey and runs it headlessly to verify it works — your time is never wasted on a broken walkthrough. If the headless run fails, the agent fixes the issue and re-runs (up to 3 attempts before escalating to you). The skill also reviews adjacent journey files for fragment extraction opportunities, keeping your lantern setup DRY as it grows.

**Example prompts:**
- "write a journey for ticket 12"
- "tour the changes"
- "show me the feature"
- "lantern me through this"
- "walk me through the signup changes"
- "create a walkthrough for the new error messages"
- "I want to see what this looks like"
- "write a walkthrough for the onboarding flow"

**Produces:**
- A journey file in `lantern/journeys/` (e.g., `ticket-12-registration-errors.ts`)
- New fragment files in `lantern/fragments/` if reusable setup routines are needed

### run

Executes a lantern journey in a headed browser with terminal narration. Performs pre-flight checks (dev servers running, fragment imports resolve, config exists), launches the browser, and pauses at each checkpoint via Playwright Inspector so you can inspect the state before continuing.

If a journey fails during the headed run, the agent automatically switches to headless mode to debug and fix the issue (up to 3 attempts) — you never have to sit through error screens. Once the fix is verified headlessly, the agent re-launches the headed browser so you see the working journey.

**Example prompts:**
- "run the journey"
- "lantern run"
- "show me"
- "preview this"
- "walk me through it"
- "run the walkthrough"
- "run it"

**Produces:**
- A headed browser session with terminal narration and checkpoint pauses
- Post-run summary with pass/fail status and suggested next steps

## Installation

Lantern is just skill files — no npm packages, no runtime dependencies beyond Playwright (which the scaffold skill installs for you). Add the skill files to whatever instruction/rules mechanism your AI agent supports.

### Claude Code

Symlink or copy each skill into your Claude skills directory:

```sh
ln -s /path/to/lantern/skills/scaffold ~/.claude/skills/lantern-scaffold
ln -s /path/to/lantern/skills/author ~/.claude/skills/lantern-author
ln -s /path/to/lantern/skills/run ~/.claude/skills/lantern-run
```

### Cursor

Copy the content of each `SKILL.md` file into your `.cursor/rules/` directory:

```sh
cp /path/to/lantern/skills/scaffold/SKILL.md .cursor/rules/lantern-scaffold.md
cp /path/to/lantern/skills/author/SKILL.md .cursor/rules/lantern-author.md
cp /path/to/lantern/skills/run/SKILL.md .cursor/rules/lantern-run.md
```

### Windsurf

Add the skill content to your `.windsurfrules` file or rules directory. Each `SKILL.md` can be concatenated into `.windsurfrules` or placed as individual files in your rules directory.

### Other Agents

These are Markdown files with structured instructions. Add them to whatever instruction, rules, or system prompt mechanism your agent supports. The three files you need are:

- `skills/scaffold/SKILL.md`
- `skills/author/SKILL.md`
- `skills/run/SKILL.md`

## Example Workflow

Here's a complete end-to-end example:

**The situation:** You're working on ticket 12, which adds validation error messages to the registration form. You've finished the implementation and want to see it in action.

**Step 1 — Scaffold (first time only)**

> "Set up lantern in this project"

Your agent detects the project structure, asks whether to manage servers or assume they're preexisting, installs Playwright if needed, and creates the `lantern/` directory with config, utilities, and directory structure. This only happens once per project (though you can re-scaffold later if things change).

**Step 2 — Author**

> "Lantern me through the ticket 12 changes"

Your agent reads the ticket, examines the diff, and finds an existing `sign-up` fragment in `lantern/fragments/`. It writes a journey file that:

1. Uses the sign-up fragment to navigate to the registration page
2. Submits the form with empty fields (tour checkpoint: "Empty form submission")
3. Fills in an invalid email and short password (verify checkpoint: "Validation errors" — with a detailed message about which error messages should appear and where)
4. Corrects the input and submits successfully (tour checkpoint: "Successful registration")

**Step 3 — Run**

> "Run it"

Your agent checks that dev servers are running, then launches the browser. You see:

```
--------------------------------------------
  Setting up: navigating to registration...
--------------------------------------------

============================================================
  [Checkpoint: Empty form submission]

  Paused. Inspect the browser, then resume.
============================================================
```

The browser is paused on the registration page with validation errors showing. You look, confirm it's correct, and click resume in Playwright Inspector. The journey continues through each checkpoint until it's done.

## Writing Custom Fragments

Fragments are reusable pieces of navigation and setup that journeys compose together. They live in `lantern/fragments/` and handle the boring parts — authentication, data seeding, navigating to a specific page — so journey files can focus on the interesting checkpoints.

**When to extract a fragment:**

- The same setup appears (or will appear) in multiple journeys
- A multi-step flow clutters the journey file
- Setup involves state creation that other journeys will need

**API-first pattern:** Fragments should prefer API calls and `page.request` over clicking through UI for setup steps. The goal is to get to the interesting part fast. Don't click through a five-step signup wizard when you can POST to `/api/users` directly.

**Auth state propagation:** Use `page.request` for API setup calls — it shares the browser's cookie jar, so cookies set by login endpoints automatically apply to subsequent `page.goto()` navigation. For token-based auth (no cookies), store the token in the browser via `page.evaluate()` or `page.setExtraHTTPHeaders()`.

**Fragment template:**

```typescript
import { Page } from '@playwright/test';
import { narrate } from '../utils';

export async function fragmentName(
  page: Page,
  options?: { /* optional config */ }
): Promise<{ /* return type */ }> {
  // API-first setup — page.request shares the browser cookie jar,
  // so session cookies from login calls automatically apply to page navigation.
  // e.g., await page.request.post('/api/login', { data: { email, password } });
  //
  // For token-based auth (no cookies):
  // const { token } = await res.json();
  // await page.evaluate(t => localStorage.setItem('auth_token', t), token);

  narrate('What just happened — e.g., Created test user and logged in');
  return { /* state for the journey — e.g., email, userId */ };
}
```

## Checkpoint Types

Journeys use two types of checkpoints, chosen based on the nature of each pause point:

### Tour

Use for new pages, layout changes, navigation flows — anything where the browser state speaks for itself. Tour checkpoints have a short label and no detailed message.

**Examples:**
- A newly designed settings page
- A layout change to the sidebar
- The landing page after a navigation flow

```typescript
await checkpoint(page, 'New settings page');
await checkpoint(page, 'Dashboard after login');
```

### Verify

Use for error states, conditional UI, copy changes, edge cases — anything with specific acceptance criteria. Verify checkpoints include a detailed message explaining what to look at, why, and what correct behavior looks like.

**Examples:**
- Validation error messages after submitting bad input
- Different UI for free vs. paid users
- A button label that should read "Save changes" not "Submit"

```typescript
await checkpoint(
  page,
  'Validation errors',
  'All three required fields should show red borders and inline error messages.\n'
  + 'The email field should say "Please enter a valid email address".\n'
  + 'The submit button should remain disabled.'
);
```

**Deciding which to use:** If you can describe "what correct looks like" in a specific sentence, it's a verify checkpoint. If the user just needs to eyeball a screen, it's tour.

## Configuration

The scaffold creates two files you can customize:

### lantern/playwright.config.ts

- **`baseURL`** — Defaults to `http://localhost:3000`. Change to match your dev server. Can also be set via the `BASE_URL` environment variable.
- **`browserName`** — Defaults to `chromium`. Can be changed to `firefox` or `webkit` under `projects`.
- **`timeout`** — Defaults to `0` (no timeout) in headed mode so checkpoints can wait for user interaction, and `30_000` in headless mode so pre-flight tests don't hang.
- **`headless`** — Controlled by the `LANTERN_HEADLESS` env var. Defaults to `false` (headed) for normal runs. Set `LANTERN_HEADLESS=1` for pre-flight and debug runs.
- **`workers`** — Set to `1`. Journeys run sequentially since you're watching them.

### lantern/utils.ts

- **`narrate()`** — Prints a framed message to the terminal. Customize the divider format or add color codes if you want.
- **`checkpoint()`** — Prints a checkpoint banner and calls `page.pause()` in headed mode. In headless mode (`LANTERN_HEADLESS=1`), it logs the checkpoint but skips `page.pause()` so pre-flight tests run to completion without hanging. You can adjust the banner format, add sounds, or integrate with other notification mechanisms.

## License

MIT
