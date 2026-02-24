---
name: author
description: >
  Use when the user wants to create a lantern journey file — a headed-browser
  walkthrough of a feature, change, or flow. Triggers: "write a journey",
  "tour the changes", "show me the feature", "lantern me through this",
  "walk me through the changes", "create a walkthrough". Analyzes the
  ticket, diff, or conversation context, composes reusable fragments with
  task-specific steps, and produces a journey file with tour or verify
  checkpoints depending on the nature of each change.
---

# Author a Lantern Journey

## Overview

This skill analyzes a ticket, diff, or conversation to understand what changed, then composes a journey file that walks a user through the relevant screens in a headed browser. The journey pauses at key checkpoints so the user can visually inspect the result.

Each journey is built from:

- **Fragments** — reusable setup and navigation routines (authentication, data seeding, common navigation paths)
- **Steps** — task-specific navigation and interactions unique to this journey
- **Checkpoints** — pause points where the user inspects the browser, in either **tour** mode (just look) or **verify** mode (evaluate something specific)

The output is a TypeScript file in `lantern/journeys/` that can be run with Playwright in headed mode.

## When to Use

- The user wants to preview a feature in the browser
- The user wants to tour the changes from a ticket or PR
- The user wants to verify specific UI behavior visually
- As a workflow step after implementing a feature — "now show me what we built"
- Mid-conversation when the user says "walk me through it" or similar

## Prerequisites

Lantern must already be scaffolded in the project. If there is no `lantern/` directory with `utils.ts` and `playwright.config.ts`, run the `lantern:scaffold` skill first.

## Process

### 1. Gather Context

Read the ticket, diff, or conversation to understand what changed. Identify:

- Which screens or flows are affected
- What is new vs. what was modified
- Any edge cases, error states, or conditional behavior called out in the ticket or acceptance criteria
- The base URL and any environment-specific considerations

If the context is ambiguous (e.g., a large diff touching many files), ask the user which parts they want to walk through.

### 2. Survey Existing Fragments

Check `lantern/fragments/` for reusable pieces. Common fragments include:

- Authentication / login setup
- Navigation helpers (e.g., "go to settings page")
- Data seeding (e.g., "create a test user with specific attributes")

List what is available and what is missing. If a fragment exists that covers part of the needed setup, plan to use it rather than duplicating the logic.

### 3. Plan the Journey

Outline the sequence of steps before writing any code:

1. **Setup** — What needs to happen before the user sees anything? Authentication, data seeding, feature flags, etc. This should use fragments where possible.
2. **Navigation** — What path gets the user to the relevant screen? Start from a known state (e.g., logged-in dashboard) and navigate to the feature.
3. **Checkpoints** — Where should the journey pause for inspection? Identify 3-6 points where the user needs to look at the browser.

If the journey would require more than 6 checkpoints, consider splitting it into multiple journey files.

### 4. Choose Checkpoint Mode

For each checkpoint, choose one of two modes:

**Tour** — Use for:
- New pages or screens ("here's the new settings page")
- Layout changes ("the sidebar is now collapsible")
- Navigation flows ("clicking this takes you here")
- General "does it render correctly" checks

Tour checkpoints use a short label only. The browser state speaks for itself.

```typescript
await checkpoint(page, 'New registration form');
```

**Verify** — Use for:
- Error states ("submit with empty fields and check the validation messages")
- Conditional logic ("free users should see the upgrade prompt, paid users should not")
- Copy changes ("the button should say 'Save changes' not 'Submit'")
- Edge cases ("what happens with a very long username")
- Anything with specific acceptance criteria

Verify checkpoints include a detailed message explaining what to look at, why, and what correct behavior looks like.

```typescript
await checkpoint(
  page,
  'Validation errors',
  'All three required fields should show red borders and inline error messages.\n'
  + 'The email field should say "Please enter a valid email address".\n'
  + 'The submit button should remain disabled.'
);
```

**When unsure:** If it is not obvious whether a checkpoint should be tour or verify, ask the user. A good heuristic: if you can describe "what correct looks like" in a sentence, it is a verify checkpoint. If the user just needs to eyeball a screen, it is tour.

### 5. Check If New Fragments Are Needed

If the journey includes a reusable flow that does not already exist as a fragment, write it. A flow is worth extracting into a fragment if:

- It will likely be used in future journeys (authentication, common navigation)
- It involves multiple steps that would clutter the journey file
- It sets up state that other journeys will also need

Fragment guidelines:

- Accept `page` as the first argument
- Use API-first patterns where possible — prefer direct API calls or `page.request` over clicking through UI for setup steps. The point is to get to the interesting part fast.
- **Auth state propagation:** When a fragment creates a user or logs in via API, the browser session needs to be authenticated too. Use `page.request` (which shares the browser's cookie jar) for API calls so cookies set by login endpoints automatically apply to subsequent `page.goto()` navigation. If the app uses token-based auth instead of cookies, store the token and set it via `page.evaluate()` or add it as a header via `page.setExtraHTTPHeaders()`.
- Call `narrate()` to explain what setup just happened (so the user is not confused by a blank screen suddenly becoming populated)
- Return any state the journey needs: created user email, auth tokens, resource IDs, etc.

### 6. Write the Journey File

Create the file in `lantern/journeys/` with a descriptive name. Naming conventions:

- Ticket-based: `ticket-12-registration-errors.ts`
- Feature-based: `onboarding-flow.ts`
- Exploratory: `dashboard-overview.ts`

The journey file should:

- Import from `@playwright/test`, `../utils`, and any relevant fragments
- Use a single `test()` block with a descriptive name
- Call fragments for setup, narrating transitions
- Navigate to the relevant screens
- Use `checkpoint()` at each inspection point
- Use `narrate()` for transitions between checkpoints to give the user context

Keep the file focused. One journey = one coherent walkthrough. If the ticket touches unrelated areas, write separate journeys.

### 7. Review for Fragment Extraction Opportunities

Before committing, review the journey you just wrote for extraction opportunities:

1. **Review the new journey** — Are there any multi-step sequences (navigation, setup, form-filling) that could be extracted into reusable fragments? If a sequence involves 3+ steps and feels like it would apply to other journeys, extract it.

2. **Check adjacent journeys** — Read other journey files in `lantern/journeys/`. Do any of them share setup or navigation patterns with the new journey? If two or more journeys duplicate the same sequence, extract it into a fragment and update all of them to use it.

3. **Check existing e2e tests** — If the project has existing e2e tests (e.g., in a `tests/` or `e2e/` directory), scan them for test helpers or setup utilities that could be adapted into lantern fragments. Test helpers for authentication, data seeding, or API setup are common candidates.

4. **Extract if warranted** — If extraction opportunities are found, create the new fragment(s) in `lantern/fragments/` following the fragment file template, and update the journey file(s) to import and use them. Do not extract for the sake of it — only extract if the pattern genuinely appears in or will appear in multiple journeys.

### 8. Headless Pre-flight Test

Run the journey headlessly to verify it works before the user ever sees it. The user's time should never be wasted on a broken journey.

**Prerequisites:** The project's `lantern/playwright.config.ts` must support the `LANTERN_HEADLESS` env var (set up during scaffold). When `LANTERN_HEADLESS=1` is set, the config overrides `headless` to `true` and sets a 30-second timeout (vs. no timeout in headed mode).

**Process:**

1. **Type-check first** — catch import errors and typos without launching a browser:
   ```bash
   npx tsc --noEmit --esModuleInterop --moduleResolution node --module esnext lantern/journeys/[filename]
   ```
   If this fails, fix the TypeScript errors and re-check before proceeding. This is much faster than a full Playwright run and catches the most common first-attempt mistakes (bad imports, missing fragments, typos).

2. **Run the journey headlessly** — use the local Playwright binary to avoid `npx` overhead:
   ```bash
   LANTERN_HEADLESS=1 ./node_modules/.bin/playwright test --config lantern/playwright.config.ts --project=chromium lantern/journeys/[filename]
   ```

3. If the test **passes**, proceed to commit.

4. If the test **fails**:
   - Read the error output carefully
   - Fix the journey or fragment code (broken selectors, missing waits, incorrect URLs, etc.)
   - Re-run headlessly
   - **Maximum 3 attempts.** If the journey still fails after 3 fix-and-retry cycles, stop and inform the user:
     > "The journey is failing after 3 attempts. Here's the latest error: [error]. This may be an issue with the app itself rather than the journey code. Here's what I've tried: [summary]."
   - Do not loop indefinitely. The user can decide whether to investigate the app, adjust the journey scope, or skip the pre-flight.

Do not skip this step. Do not commit a journey that fails headlessly. A journey that crashes in headed mode wastes the user's time and undermines trust in lantern.

### 9. Commit

Stage the new journey file and any new or updated fragments. Commit with a message like:

```
add lantern journey for [feature/ticket description]
```

Do not push — let the user decide when to push.

## Journey File Template

```typescript
import { test } from '@playwright/test';
import { narrate, checkpoint } from '../utils';
// import fragments as needed

test('[Context]: [Description]', async ({ page }) => {
  // Setup via fragments
  narrate('Setting up: [what setup does]...');
  // const { ... } = await someFragment(page);

  // Navigate to feature
  narrate('[Transitional context]...');
  // await page.goto('/...');

  // Tour checkpoint — new screen, just look at it
  await checkpoint(page, '[Short label]');

  // Verify checkpoint — specific thing to evaluate
  await checkpoint(
    page,
    '[Label]',
    '[What to look at, why, and what correct looks like]'
  );
});
```

## Fragment File Template

```typescript
import { Page } from '@playwright/test';
import { narrate } from '../utils';

export async function fragmentName(
  page: Page,
  options?: { /* optional config */ }
): Promise<{ /* return type */ }> {
  // API-first setup (prefer over clicking through UI)
  // Use page.request so cookies propagate to the browser session automatically:
  // const res = await page.request.post('/api/login', { data: { email, password } });
  //
  // For token-based auth (no cookies), store the token in the browser:
  // const { token } = await res.json();
  // await page.evaluate(t => localStorage.setItem('auth_token', t), token);

  narrate('[What just happened]');
  return { /* state for the journey */ };
}
```

## Common Mistakes

- **Using `page.waitForTimeout()`** — Playwright has built-in auto-waiting. Use `toBeVisible()`, `waitForResponse()`, `waitForURL()`, or other auto-waiting methods. Hard timeouts are flaky and slow.
- **Too many checkpoints** — 3-6 per journey is the sweet spot. More than that becomes tedious for the user. If you need more, split into multiple journeys.
- **Mixing setup into checkpoints** — Setup steps (logging in, seeding data, navigating to a page) should be narrated, not paused on. The user does not need to inspect the login screen unless that is what changed.
- **Hardcoding URLs** — Use the `baseURL` from Playwright config. Write `page.goto('/')` not `page.goto('http://localhost:3000/')`.
- **Brittle selectors** — Prefer `data-testid` attributes or accessible roles (`getByRole`, `getByLabel`) over CSS class selectors or deeply nested DOM paths.
- **Fragments that are too specific** — A fragment should be reusable across multiple journeys. If it only applies to one journey, inline it instead of extracting.
- **Forgetting to narrate transitions** — Without narration, the user sees the browser jump between pages with no explanation. A short `narrate()` call before navigation keeps the walkthrough coherent.
- **Skipping the planning step** — Writing the journey without planning leads to a disorganized walkthrough. Outline the checkpoints first, then write the code.
