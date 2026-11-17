---
name: flaky-analyzer
description: >
  Analyze test execution history to identify flaky tests in this Cucumber/Playwright TaHub automation framework.
  Use this skill whenever the user wants to find flaky tests, analyze test stability, understand which tests
  are unreliable, or says anything like "analyze flaky tests", "find flaky tests", "which tests are flaky",
  "test stability report", "unreliable tests", "intermittent failures", "tests that sometimes fail",
  "test health check". Also triggers when the user asks about test reliability or wants to understand
  patterns of failure across multiple test runs. Can also perform static analysis on test code to detect
  anti-patterns that cause flakiness.
---

# Flaky Test Analyzer

Identify flaky tests by comparing results across multiple test runs and detect code anti-patterns that cause intermittent failures.

## Workflow Overview

1. **Collect historical data** — read timestamped reports from `reports/history/`
2. **Compare across runs** — map each scenario to pass/fail per run
3. **Identify flaky tests** — flag scenarios with mixed results
4. **Classify root cause** — determine the flakiness pattern
5. **Static analysis** (optional) — scan test code for anti-patterns
6. **Report findings** — present results with severity and suggested fixes

---

## Step 1: Collect Historical Data

Read all timestamped report files from the history directory:

```
reports/history/cucumber-report-{timestamp}.json
```

These files are created automatically after each test run by `htmlReportGenerator.cjs`.

If no history files exist, inform the user:
> "No historical reports found in `reports/history/`. Run your tests at least 2-3 times to generate history data for flaky test analysis."

**Minimum required**: At least 3 runs to identify meaningful patterns. With fewer runs, analysis is less reliable — note this in the report.

Parse each JSON file to extract:
- Run timestamp (from filename)
- Feature name and scenario name
- Pass/fail status for each scenario
- Error messages for failed scenarios

---

## Step 2: Compare Across Runs

Build a matrix of scenario results:

```
Scenario Name                    | Run 1 | Run 2 | Run 3 | Run 4 | Run 5
---------------------------------|-------|-------|-------|-------|------
Login to the application         |  PASS | PASS  | PASS  | PASS  | PASS
Verify File Master Info screen   |  PASS | FAIL  | PASS  | FAIL  | PASS
Archive Source File              |  FAIL | FAIL  | FAIL  | FAIL  | FAIL
User does not delete Alias       |  PASS | PASS  | FAIL  | PASS  | PASS
```

Categorize each scenario:
- **Stable Pass** — passes in all runs (100% pass rate)
- **Stable Fail** — fails in all runs (0% pass rate) — this is a bug, not flakiness
- **Flaky** — mixed results (passes in some runs, fails in others)

---

## Step 3: Identify Flaky Tests

A test is flaky if its pass rate is between 1% and 99% (inclusive). Rank by severity:

| Pass Rate | Severity | Color | Description |
|-----------|----------|-------|-------------|
| ≤ 50% | Critical | Red (#dc3545) | Highly unreliable — fails as often or more than it passes |
| 51-70% | High | Orange (#fd7e14) | Frequently flaky — fails in many runs |
| 71-85% | Medium | Yellow (#ffc107) | Occasionally flaky — fails sometimes |
| > 85% | Low | Green (#28a745) | Rarely flaky — almost always passes |

---

## Step 4: Classify Root Cause

For each flaky test, examine the error messages across failed runs to classify the pattern:

### Timing / Race Condition
**Indicators:**
- `TimeoutError`, `waiting for locator`, `Timeout Xms exceeded`
- Same locator fails intermittently
- Error at `waitForSelector` or `.textContent()` or `.click()`

**Typical fix:** Add explicit `waitForSelector` before interaction, increase timeout, add `waitForLoadState`

### Cascade Failure
**Indicators:**
- Scenario always fails when a specific earlier scenario also fails in the same run
- Error is `TA Iris page not found after 10 attempts` or `Target page, context or browser has been closed`
- Scenario passes in runs where the prior scenario also passes

**Typical fix:** Improve After hook recovery, make scenarios independent, add page re-initialization

### Data Dependency
**Indicators:**
- Scenario modifies data (add/delete alias, archive file) and later scenarios expect that data in a specific state
- Fails when run order changes or when run after a data-modifying scenario that failed midway
- Different errors across runs (sometimes assertion mismatch, sometimes element not found)

**Typical fix:** Reset test data before each scenario, use unique test data per scenario, add setup/teardown

### Environment / Infrastructure
**Indicators:**
- Fails at same time of day (deployments, maintenance windows)
- Network-related errors (`ERR_CONNECTION_REFUSED`, `net::ERR_TIMED_OUT`)
- Fails on specific environments but not others
- All tests in a run fail together

**Typical fix:** Not a test issue — investigate environment stability

---

## Step 5: Static Analysis (Optional)

When the user requests static analysis (or when no historical data is available), scan test code for anti-patterns that cause flakiness.

### Anti-patterns to detect:

**1. Missing wait before `.count()`**
```typescript
// BAD — count() returns immediately, may be 0 if grid hasn't rendered
for(;i<await actions_loc.count();i++){
```
Look for: `.count()` without a preceding `waitForSelector` on the same locator root.

**2. Hard-coded `.nth()` without guard**
```typescript
// BAD — if i exceeds available elements, times out
let fileName_loc = this.page.locator('...').nth(i)
return await fileName_loc.textContent()
```
Look for: `.nth(variable)` followed by `.textContent()` or `.click()` without a try/catch or bounds check.

**3. No wait between page transitions**
```typescript
// BAD — page may not have navigated yet
await button.click();
await this.page.locator('//next-page-element').click(); // may fail
```
Look for: `.click()` immediately followed by locator interaction without `waitForLoadState`, `waitForNavigation`, or `waitForSelector`.

**4. Shared mutable state between scenarios**
```typescript
// BAD — if scenario 1 sets fileName and fails, scenario 2 gets stale value
var fileName: string = "";
```
Look for: module-level `var` declarations used across multiple step definitions.

**5. `ensureIrisPage()` without sufficient context**
Look for: methods that call `ensureIrisPage()` but don't handle the case where the page was closed by a prior scenario's After hook.

### Files to scan:
- `src/pageobjects/RegulatoryTaxReporting/Iris/*.page.ts`
- `src/steps/RegulatoryTaxReporting/Iris/*.steps.ts`
- `src/steps/RegulatoryTaxReporting/common.steps.ts`

---

## Step 6: Report Findings

Present findings in this format:

```
## Flaky Test Analysis Report

**Reports analyzed**: N runs (date range: YYYY-MM-DD to YYYY-MM-DD)
**Total scenarios**: X
**Stable Pass**: X (Y%)
**Stable Fail**: X (Y%)
**Flaky**: X (Y%)

### Flaky Tests (ranked by severity)

| # | Scenario | Pass Rate | Severity | Root Cause | Suggested Fix |
|---|----------|-----------|----------|------------|---------------|
| 1 | Verify File Master Info | 60% | Critical | Timing | Add waitForSelector before grid iteration |
| 2 | User does not delete Alias | 80% | High | Cascade | Fix After hook page recovery |

### Detailed Analysis

#### 1. Verify File Master Info (60% pass rate)
- **Failed in runs**: 2026-08-01, 2026-08-03
- **Error pattern**: TimeoutError on `//ag-grid-angular[@id="fileInfoActiveGrid"]//div[@col-id="userDefinedFileName"]`
- **Root cause**: Grid not fully rendered before iteration starts
- **Fix**: Add `waitForSelector` for `.icon-entry` before calling `.count()` on actions column

### Static Analysis Findings (if run)

| # | File | Line | Anti-pattern | Risk |
|---|------|------|-------------|------|
| 1 | fileMasterInfo.page.ts | 417 | .count() without prior wait | High |
| 2 | fileMasterInfo.page.ts | 642 | .nth(i) without bounds check | High |
```

---

## Step 7: Generate HTML Report (MANDATORY — always run this)

**IMPORTANT**: After completing the analysis, you MUST ALWAYS execute the following command to generate the HTML report. This is not optional — run it every single time this skill is invoked, regardless of whether the user explicitly asked for it:

```bash
node ./flakyReportGenerator.cjs
```

This produces `reports/flaky-analysis-report.html` — a self-contained HTML file with:
- Summary stat cards (total scenarios, stable pass, flaky, stable fail)
- Color-coded severity badges (Critical/High/Medium/Low) using corrected thresholds (≤50% = Critical, not 40-60%)
- Run history bar chart with percentage above each bar and vertical date labels below
- **Collapsible "Flaky Tests" section** — starts expanded, click header to collapse
- **Collapsible "Stable Fail" section** — starts collapsed, shows scenarios that fail in every single run with "ALWAYS FAILS" badge
- Each scenario within sections is expandable (`<details>`) showing feature, tags, pass rate, root cause, suggested fix, and error history
- Pass/fail matrix with green/red cells per run

**Note**: The old `flakyAnalyzer.cjs` (CLI-only output) has been removed. `flakyReportGenerator.cjs` is the single source for flaky analysis.

The user can also run this independently via:
```bash
npm run flaky-report
```

After generating, always inform the user:
> "📊 HTML report generated at `reports/flaky-analysis-report.html` — open it in your browser for a visual summary."

---

## Rules

- Minimum 3 historical reports needed for meaningful analysis; note confidence level if fewer
- Never modify test files during analysis — this is read-only
- Always distinguish between flaky (intermittent) and consistently broken (bug)
- Present the cascade failure pattern clearly — a single root cause can cause 10+ "flaky" tests
- When cascade is detected, identify the true root failure (the first scenario that fails)
- Static analysis findings should be ranked by likelihood of causing flakiness
- If the user asks to fix flaky tests, hand off to the `/self-heal` skill for the actual fix
