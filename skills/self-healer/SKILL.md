---
name: self-healer
description: >
  Automatically diagnose and fix failing Cucumber/Playwright test cases in this TaHub automation framework.
  Use this skill whenever a test fails and the user wants it fixed automatically, or says anything like
  "fix this test", "self-healer", "self-heal", "the test is failing can you fix it", "debug and fix this failure",
  "retry and fix", "why is this test failing", "this locator is broken", "element not found fix it",
  "test was passing but now fails". Also triggers when the user runs a test, it fails, and they ask
  you to resolve the issue. Works by analyzing the error, inspecting the live application to find
  the correct state, updating page objects or step definitions, and re-running until green.
---

# Self-Heal

Automatically diagnose and fix failing Cucumber/Playwright tests by analyzing errors, inspecting the live application, and applying targeted fixes.

## Workflow Overview

1. **Capture the failure** — understand what failed and why
2. **Classify the error** — determine the failure category
3. **Inspect the live app** — use `playwright-cli` to see current application state
4. **Apply the fix** — update page object, step definition, or feature file
5. **Re-run the test** — verify the fix works
6. **Loop** — repeat steps 1-5 until the test passes or max retries reached

---

## Step 1: Capture the Failure

Determine the failure from one of these sources (in priority order):

### Source 1: Read from the test report (default)

After a test run, read the generated report to extract failures:

```bash
# Read the cucumber report JSON
cat reports/cucumber-report.json
```

Parse the report to extract:
- **Failed scenarios** — scenario name and tags
- **Which step failed** — the Gherkin step text with status `"failed"`
- **Error message** — the `error_message` field on the failed step
- **Embedded screenshot** — if present in the `embeddings` array

If multiple scenarios failed, either:
- Fix the specific scenario the user named
- If user said "fix all" or "fix the failures", process them one at a time starting with the first

### Source 2: User provides error output

If the user pastes error output directly in the conversation, use that instead of reading the report.

### Source 3: Run the test and capture output

If no report exists and no error output is provided, run the test. **Important**: Always include Login and Logout scenarios — they set up and tear down the browser session. Use the feature-level tag with `--name` to include them:

```bash
npx cross-env ENV={env} BROWSER=chrome WORKSPACE="Regulatory - Tax Reporting" cucumber-js --tags '@{featureTag}' --name "Login|{scenario name keywords}|Logout" 2>&1
```

Where `{featureTag}` is the feature-level tag (e.g., `@MFDFileMasterInfo`) and `{scenario name keywords}` matches the target scenario.

### What to extract (regardless of source):
- **Which step failed** — the Gherkin step text
- **Error type** — timeout, assertion, element not found, etc.
- **Error message** — the full Playwright error
- **File and line** — which page object or step definition threw
- **Screenshot path** — if available

---

## Step 2: Classify the Error

Determine which category the failure falls into:

| Category | Indicators | Typical Fix |
|----------|-----------|-------------|
| **Broken locator** | `TimeoutError`, `waiting for locator`, element not found | Re-inspect app, update XPath |
| **Timing issue** | Intermittent timeout, element exists but not interactable | Add/adjust `waitForTimeout` or `waitForSelector` |
| **Flow change** | Unexpected popup, different page state, navigation changed | Inspect current flow, update step sequence |
| **Assertion mismatch** | `expect` failed, text/value changed | Inspect current values, update expected text |
| **Page context issue** | Wrong page targeted in multi-window setup | Fix `ensureIrisPage()` or page selection logic |
| **Grid structure change** | AG Grid columns/rows not found | Inspect grid DOM, update grid locators |

### Not self-healable (stop and report to user):
- Backend/API returning errors (500, 404)
- Test data missing or invalid
- Environment down or unreachable
- Business logic fundamentally changed (test expectation itself is wrong)
- Authentication/permission issues

If the error falls into a non-healable category, report the diagnosis to the user and suggest what needs to change at the environment or requirements level.

---

## Step 3: Inspect the Live Application

Use the `playwright-cli` CLI tool (bash commands) to open the app and understand the current state. Do NOT use the Playwright MCP plugin tools directly — always use `playwright-cli` bash commands.

### Open the application:

```bash
# Open browser and navigate to the app URL (from env file)
playwright-cli open {app-url}

# Login if needed
playwright-cli snapshot
playwright-cli fill {username-field-ref} "AUTO2"
playwright-cli fill {password-field-ref} "automat"
playwright-cli click {sign-in-button-ref}

# Navigate to the screen where failure occurs
playwright-cli snapshot
playwright-cli click {element-ref}
```

### For broken locators:

```bash
# Search for the element by visible text
playwright-cli find "expected button text"

# Get the element's current attributes
playwright-cli eval "el => el.id" {ref}
playwright-cli eval "el => el.getAttribute('class')" {ref}
playwright-cli eval "el => el.outerHTML" {ref}

# Generate a Playwright locator for the element
playwright-cli generate-locator {ref} --raw

# Build the new XPath
playwright-cli eval "el => { /* walk up DOM to build unique XPath */ }" {ref}
```

### For flow changes:

```bash
# Take a snapshot to see the current page structure
playwright-cli snapshot

# Check if there's an unexpected dialog/popup
playwright-cli find "dialog"
playwright-cli find "modal"

# Check current page title (multi-window scenarios)
playwright-cli eval "document.title"
```

### For assertion mismatches:

```bash
# Navigate to the element and check its current value
playwright-cli find "the text that was expected"
playwright-cli eval "el => el.textContent" {ref}
playwright-cli eval "el => el.innerText" {ref}
```

### For grid issues:

```bash
# Inspect AG Grid structure
playwright-cli find "ag-grid"
playwright-cli eval "el => el.querySelectorAll('.ag-header-cell-text').length" {ref}
playwright-cli eval "el => [...el.querySelectorAll('.ag-header-cell-text')].map(e => e.textContent)" {ref}

# Check grid row count
playwright-cli eval "el => el.querySelectorAll('.ag-row').length" {ref}

# Check specific column values
playwright-cli eval "el => [...el.querySelectorAll('[col-id=\"fileName\"]')].map(e => e.textContent)" {ref}
```

Close the browser when done:
```bash
playwright-cli close
```

---

## Step 4: Apply the Fix

Based on the diagnosis and inspection, apply the minimal fix needed.

### For broken locators — update the page object:

1. Read the page object file that contains the broken locator
2. Replace the old XPath with the new one discovered in Step 3
3. Keep the same getter property name and method structure

```typescript
// Before (broken)
get submitButton(): Locator {
  return this.page.locator('//*[@id="old-submit-btn"]');
}

// After (fixed)
get submitButton(): Locator {
  return this.page.locator('//*[@id="btnSubmit"]');
}
```

### For timing issues — add appropriate waits:

```typescript
// Add waitForSelector before interaction
await this.page.waitForSelector('//xpath-here', { state: 'visible', timeout: 10000 });

// Or add a brief wait for dynamic content
await this.page.waitForTimeout(2000);
```

### For flow changes — update the step sequence or add handling:

If a new popup/dialog appeared that wasn't there before, add handling in the page object method:

```typescript
async clickSubmit() {
  await this.ensureIrisPage();
  await this.submitButton.click();
  // Handle new confirmation dialog that was added
  const dialog = this.page.locator('//*[@id="confirmDialog"]');
  if (await dialog.isVisible({ timeout: 3000 })) {
    await this.page.locator('//*[@id="confirmYes"]').click();
  }
}
```

### For assertion mismatches — update expected values:

If the application text legitimately changed (not a bug), update the feature file or step definition with the correct expected value.

### Fix rules:
- Change the minimum amount of code necessary
- Do NOT restructure or refactor unrelated code
- Do NOT change the framework infrastructure
- **Locator preference**: Prefer XPath and CSS selectors. When these are difficult to construct reliably (dynamic IDs, deeply nested elements, no unique attributes), use Playwright's built-in locators (`getByRole`, `getByText`, `getByLabel`, `getByTestId`) — pick whichever is most stable and least likely to be flaky
- Preserve the `ensureIrisPage()` pattern
- If unsure whether the app change is intentional, ask the user before updating assertions

---

## Step 5: Re-run the Test

Execute the test again, always including Login and Logout scenarios:

```bash
npx cross-env ENV={env} BROWSER=chrome WORKSPACE="Regulatory - Tax Reporting" cucumber-js --tags '@{featureTag}' --name "Login|{scenario name keywords}|Logout" 2>&1
```

If running from TAHub (only when user explicitly requests):
```bash
npx cross-env BROWSER=TAHub WORKSPACE="Regulatory - Tax Reporting" cucumber-js --tags '@{featureTag}' --name "Login|{scenario name keywords}|Logout" 2>&1
```

---

## Step 6: Loop or Report

### If the test passes:
- Report success to the user
- Summarize what was wrong and what was fixed

### If the test fails again:
- Check if it's the **same error** or a **different error**
- If same error — the fix didn't work. Re-inspect with different approach
- If different error — progress was made. Go back to Step 2 with the new error
- **Maximum 3 retry attempts** — if still failing after 3 fixes, stop and report:
  - What was tried
  - What the current error is
  - Recommendation for manual investigation

### Reporting format:

```
## Self-Heal Result

**Status**: Fixed / Partially Fixed / Could Not Fix
**Attempts**: N/3
**Original Error**: [error summary]
**Root Cause**: [what was actually wrong]
**Fix Applied**: [what was changed and where]
**Files Modified**: [list of files]
```

---

## Rules

- Maximum 3 fix-and-retry cycles — do not loop indefinitely
- Never change test expectations without confirming the app change is intentional
- Never modify framework infrastructure (hooks, world, config)
- **Locator strategy**: Prefer XPath and CSS. Fall back to Playwright built-in locators (`getByRole`, `getByText`, etc.) when XPath/CSS would be fragile — stability over purity
- Always inspect the live app before guessing at fixes — evidence-based fixes only
- If the failure is environmental (app down, wrong env, missing data), report immediately — don't try to "fix" code
- Keep fixes minimal — one targeted change per attempt, not broad rewrites
- Always preserve `ensureIrisPage()` pattern in page objects
- Close the playwright-cli browser session after each inspection
