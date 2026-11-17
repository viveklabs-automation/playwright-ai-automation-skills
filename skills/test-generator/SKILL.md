---
name: test-generator
description: >
  Add a new end-to-end test case to this Cucumber/Playwright automation framework.
  Use this skill whenever the user wants to add a test, create a scenario, write a new feature file,
  implement step definitions, build page objects for this TaHub testing project, or says anything like
  "add a test for X", "automate this scenario", "create a new feature", "write steps for this flow",
  "I need a test case for [feature]", "create step definitions for this feature file",
  "implement the steps for [scenario name]". Also use when the user provides a Jira ticket and wants it
  automated, or asks to test a specific screen/flow in the application, or provides a feature file
  with steps already written and wants step definitions and page objects created. Triggers even if the
  user says "test" or "automate" in the context of this project.
---

# Test Generator

Add a new end-to-end test case to this Cucumber.js + Playwright automation framework following the existing project conventions exactly.

## Workflow Overview

1. **Gather requirements** — understand what to test
2. **Explore the application** — use the `playwright-cli` skill to inspect the live app
3. **Check for reusable code** — find existing step definitions, page objects, and utilities
4. **Implement** — create only the files that are needed (feature, page object methods, step definitions)
5. **Run the test** — execute and verify it passes

---

## Step 1: Gather Requirements

Collect these inputs from the user (ask only if none of the resolution methods below can provide them):

| Input | Required | How to resolve |
|-------|----------|----------------|
| Test scenario description | Yes | Plain language, Gherkin steps, or a feature file with steps already written |
| Application URL | Conditional | Resolve from `env/{WORKSPACE}/.env.{ENV}` if workspace + environment are given; OR user provides directly |
| Login credentials | Conditional | Use credentials already in the feature file's login step; OR read from `.env` file; OR user provides directly |
| Workspace | No | Defaults to `Regulatory - Tax Reporting` |
| Jira ticket ID | No | For tagging the scenario with `@CTA-XXXXX` |
| Environment | No | Defaults to `DEV` |

### Resolution priority for URL:
1. **User provides URL directly** → use it
2. **User provides workspace + environment** → read URL from `env/{WORKSPACE}/.env.{ENV}`
3. **Neither provided** → use defaults (`Regulatory - Tax Reporting` + `DEV`) and read from env file

### Resolution priority for credentials:
1. **Credentials already in the feature file** (in the login step's DataTable) → use as-is, do not ask
2. **Credentials in `.env` file** → read from there
3. **User provides explicitly** → use what's given
4. **None available** → ask the user

### When user provides a feature file with steps already written:
If the user points to a feature file (or a specific scenario by name) that already has Gherkin steps written, skip requirement gathering for the scenario description. Instead:
- Read the feature file
- Extract the steps from the specified scenario (or all scenarios if not specified)
- Proceed directly to Step 3 (Check for Reusable Code) to determine which steps need new definitions

---

## Step 2: Explore the Application

Use the `playwright-cli` CLI tool (bash commands) to understand the application's UI, DOM structure, and behavior. Do NOT use the Playwright MCP plugin tools directly — always use `playwright-cli` bash commands. The goal is to identify the correct selectors and interaction patterns.

```bash
# Open the application
playwright-cli open <app-url>

# Take a snapshot to see page structure and element refs
playwright-cli snapshot

# Search for specific text on the page
playwright-cli find "element text"

# Navigate — login, click through to the target screen
playwright-cli fill {username-ref} "AUTO2"
playwright-cli fill {password-ref} "automat"
playwright-cli click {sign-in-ref}

# Capture selectors — use eval to get IDs, classes, attributes
playwright-cli eval "el => el.id" e5
playwright-cli eval "el => el.getAttribute('data-testid')" e5
playwright-cli eval "el => el.className" e5
playwright-cli eval "el => el.outerHTML" e5

# Generate locators for elements you'll interact with
playwright-cli generate-locator e5 --raw
```

Focus on:
- Element IDs, XPaths, and attributes used by existing page objects (this project uses XPath heavily)
- The flow of user interactions needed for the test scenario
- Any loading states, popups, or multi-page transitions
- AG Grid structures (this app uses AG Grid extensively)

Close the browser session when done exploring:
```bash
playwright-cli close
```

---

## Step 3: Check for Reusable Code

Before writing anything, search the codebase for existing implementations you can reuse.

### Existing step definitions to check:
- `src/steps/Global/global.steps.ts` — login, logout, X-Ray test case ID
- `src/steps/RegulatoryTaxReporting/common.steps.ts` — fund sponsor selection, landing page, feature card selection, continue button
- `src/steps/RegulatoryTaxReporting/Iris/filemasterInfo.steps.ts` — File Master Info screen interactions
- `src/steps/RegulatoryTaxReporting/Iris/userPermissions.steps.ts` — feature card/menu status, search by status, action icons

### Existing page objects:
- `src/pageobjects/Global/global.page.ts` — login, system selection, browser management
- `src/pageobjects/RegulatoryTaxReporting/Iris/fileMasterInfo.page.ts` — File Master Info screen

### Key patterns to reuse:
- Login scenario outline is always the first scenario in a feature file
- Logout scenario is always the last scenario
- `ensureIrisPage()` pattern for multi-page handling
- `ICustomWorld` typing on all step functions
- Global variables for cross-step state (`fileName`, `aliasName`, etc.)

If an existing step definition matches what you need, reuse it. Do NOT create duplicates.

---

## Step 4: Implement

### Directory structure convention:

```
features/{Workspace}/{SubArea}/{Feature}.feature
src/pageobjects/{Workspace}/{SubArea}/{feature}.page.ts
src/steps/{Workspace}/{SubArea}/{feature}.steps.ts
```

For this project specifically:
```
features/RegulatoryTaxReporting/Iris/MFD/{Feature}.feature
src/pageobjects/RegulatoryTaxReporting/Iris/{feature}.page.ts
src/steps/RegulatoryTaxReporting/Iris/{feature}.steps.ts
```

### Feature file conventions:

```gherkin
@IRISRegression @MFD{FeatureName}
Feature: This feature contains scenarios related to the {Feature Name} screen

  Scenario Outline: Login to the application
    Given User logs into the application with
      | Username | AUTO2   |
      | Password | automat |
    And User selects System and Fund Sponsors as "<Management>"
    And User selects "Continue" button
    Then User is in landing page
    And User selects System and Fund Sponsors as "<Management>"
    And User clicks on Continue button when it appeared
    And User selects "IRIS Reporting" feature card

    Examples:
      | Management              |
      | MFD/CD - AMERICAN FUNDS |

  @CTA-{ticketId}
  Scenario Outline: {Test scenario description}
    When {first step}
    ...

    Examples:
      | Management     |
      | AMERICAN FUNDS |

  Scenario: Logout from the application
    Given User logs out from the application
```

Key feature file rules:
- Always use `Scenario Outline` with `Examples` table (even for single data set)
- Tag with `@IRISRegression` and feature-specific tag
- Tag individual scenarios with Jira IDs: `@CTA-XXXXXX`
- Login is always first, logout is always last
- Use the exact login step wording shown above

### Page object conventions:

```typescript
import { expect, type Locator, type Page } from '@playwright/test';

export class {Feature}Page {
  page: Page;
  constructor(page: Page) {
    this.page = page;
  }

  private async ensureIrisPage(): Promise<void> {
    for (let attempt = 0; attempt < 10; attempt++) {
      const listofPages = this.page.context().pages();
      for (let i = 0; i < listofPages.length; i++) {
        try {
          const title = await listofPages[i].title();
          if (title.includes('TA IRIS Reporting')) {
            this.page = listofPages[i];
            return;
          }
        } catch (e) {
        }
      }
      await new Promise((r) => setTimeout(r, 1000));
    }
    throw new Error('TA Iris page not found after 10 attempts');
  }

  // Locators as getter properties
  get elementName(): Locator {
    return this.page.locator('//xpath-here');
  }

  // Methods call ensureIrisPage() first
  async actionMethod(param: string) {
    await this.ensureIrisPage();
    // interaction logic
  }
}
```

Key page object rules:
- Always include `ensureIrisPage()` private method (copy from existing page objects)
- **Locator preference** (in order): XPath first, CSS second. If neither produces a reliable locator, use Playwright's built-in locator strategies (`getByRole`, `getByText`, `getByLabel`, etc.) — but only when XPath/CSS would be fragile or overly complex. The goal is stability over purity.
- Locators are getter properties
- Methods are async and call `ensureIrisPage()` before interacting
- Use `this.page.waitForTimeout()` or `new Promise((r) => setTimeout(r, ms))` for waits
- Use `@playwright/test` expect for assertions within page objects
- Grid interactions use `ag-grid-angular[@id="gridId"]` pattern

### Step definition conventions:

```typescript
import { ICustomWorld } from '../../../support/custom-world';
import { Given, When, Then } from '@cucumber/cucumber';
import { expect } from '@playwright/test';
import { {Feature}Page } from '../../../pageobjects/{Workspace}/{SubArea}/{feature}.page';

Then('{step text with {string} params}', async function (this: ICustomWorld, param: string) {
  const featurePage = new {Feature}Page(this.page!);
  await featurePage.methodName(param);
});
```

Key step definition rules:
- Type `this` as `ICustomWorld`
- Instantiate page object with `this.page!`
- Use `{string}` for Gherkin string parameters
- Use module-level `var` for shared state between steps (e.g., `var fileName: string = ""`)
- Import from relative paths using `../` notation

---

## Step 5: Run the Test

**Important**: The test scenarios depend on Login (first) and Logout (last) scenarios in the feature file. Since these don't have individual tags, use the **feature-level tag** combined with `--name` to include Login, the target scenario(s), and Logout.

**Default: Standalone mode.** Execute the test in a browser:

```bash
npx cross-env ENV={env} BROWSER=chrome WORKSPACE="Regulatory - Tax Reporting" cucumber-js --tags '@{featureTag}' --name "Login|{scenario name keywords}|Logout"
```

**TAHub mode (only when explicitly requested by the user):**

```bash
npx cross-env BROWSER=TAHub WORKSPACE="Regulatory - Tax Reporting" cucumber-js --tags '@{featureTag}' --name "Login|{scenario name keywords}|Logout"
```

Where:
- `{featureTag}` is the feature-level tag (e.g., `@MFDFileMasterInfo`, `@MFDViewTransmissions`) — NOT the individual scenario tag
- `{scenario name keywords}` is a regex matching the target scenario name (e.g., `column filtering`, `pagination`)
- `Login` and `Logout` must always be included in the `--name` regex so the browser session is set up and torn down

Example:
```bash
npx cross-env ENV=DEV BROWSER=chrome WORKSPACE="Regulatory - Tax Reporting" cucumber-js --tags '@MFDFileMasterInfo' --name "Login|column filtering|Logout"
```

TAHub mode requires:
- TaHub desktop app available on the machine
- Chrome 136+ with registry modification (`chromeRegistry/ChromeForCDPRegEdit.reg`)
- Connects via CDP to `http://localhost:9221`

Where `{tag}` is the Jira ticket tag (e.g., `@CTA-108265`) or a custom debug tag.

If the test fails:
1. Read the error output carefully
2. Use `playwright-cli` to re-inspect the failing element/interaction
3. Fix the page object or step definition
4. Re-run until green

---

## Rules — Do NOT Violate

- Never modify existing files unless absolutely necessary to add new functionality
- Never create a step definition that duplicates an existing one — grep first
- **Locator strategy**: Prefer XPath and CSS selectors. When these are difficult to construct reliably (dynamic IDs, deeply nested elements, no unique attributes), use Playwright's built-in locators (`getByRole`, `getByText`, `getByLabel`, `getByTestId`) — pick whichever is most stable and least likely to be flaky
- Never add unnecessary abstractions, utilities, or helper files
- Never change the framework infrastructure (hooks, world, config)
- Keep implementations minimal — only what the test case requires
- Follow the exact Gherkin wording patterns from existing features
- Use `Scenario Outline` with `Examples` even for single data sets
