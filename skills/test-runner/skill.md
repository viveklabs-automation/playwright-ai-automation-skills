---
name: test-runner
description: >
  Run Cucumber/Playwright test scenarios in this TaHub automation framework.
  Use this skill whenever the user wants to run tests, execute scenarios, trigger a feature file,
  or says anything like "run the tests", "execute @CTA-109319", "run FileMasterInfo",
  "run column filtering scenarios", "run @IRISRegression", "execute in TAHub mode",
  "run this feature", "run these scenarios", "trigger the test suite".
  Also triggers on: "run it", "execute it", "let's run", "try running", "kick off the tests",
  "run in standalone", "run in TAHub", "run all scenarios", "run the feature file".
  Do NOT use for creating or fixing tests — use /test-generator or /self-healer for those.
---

# Test Runner

Run existing Cucumber/Playwright test scenarios by feature name, scenario name, or tags.

## How It Works

The user provides one or more of:
- A **feature file name** (e.g., "FileMasterInfo", "ViewTransmissions")
- **Scenario names** or keywords (e.g., "column filtering", "pagination")
- **Tags** (e.g., "@CTA-109319", "@IRISRegression", "@MFDFileMasterInfo")

You build the correct `cucumber-js` command, always including Login and Logout, and execute it.

---

## Step 1: Determine What to Run

### From the user's input, resolve:

| Input Type | Example | How to use |
|-----------|---------|------------|
| Feature file name | "FileMasterInfo" | Find the feature file, use its feature-level tag with `--tags` |
| Scenario name/keywords | "column filtering" | Use feature-level tag + `--name "Login\|column filtering\|Logout"` |
| Feature-level tag (unique) | "@MFDFileMasterInfo" | Use directly with `--tags` + `--name "Login\|.*\|Logout"` to run all scenarios |
| Feature-level tag (shared) | "@MFDUserPermissions" | Use **file path** instead of `--tags` — see "Shared feature tags" below |
| Scenario tag(s) | "@CTA-109319" | Find which feature file contains it, use **file path** + `--name "Login\|<scenario name>\|Logout"` |
| "all" or suite tag | "@IRISRegression" | Use `--tags '@IRISRegression'` (Login/Logout are part of each feature) |

### Resolving feature file from name:
Look up in `features/RegulatoryTaxReporting/Iris/MFD/` for a matching `.feature` file. Read line 1 to get the feature-level tag.

### Resolving scenario name from tag:
If user provides a scenario tag like `@CTA-109319`, grep the feature files to find it, read the scenario name on the next line, and use that in `--name`.

### Shared feature tags — use file path instead of --tags:
Some feature files share the same feature-level tag (e.g., `@MFDUserPermissions` is on AUTO2, AUTO3, AUTO5, AUTO6). When a tag is shared across multiple feature files, using `--tags` will match ALL of them and run unwanted scenarios from other files.

**Solution**: Use the **feature file path** directly instead of `--tags`:
```bash
cucumber-js features/RegulatoryTaxReporting/Iris/MFD/UserPermissions/user_permissions_AUTO2.feature --name "Login|<scenario keywords>|Logout"
```

**How to detect**: After grepping for the scenario tag, check if the feature-level tag on that file also appears in other feature files. If it does, use the file path approach.

### Always include Login and Logout:
Every run MUST include Login (first scenario) and Logout (last scenario). Use `--name` with a regex:
```
--name "Login|<target scenario keywords>|Logout"
```

If running ALL scenarios in a feature (no specific scenario filter), just use `--tags '@featureTag'` without `--name` — since the feature file already has Login first and Logout last, all scenarios will run in order.

---

## Step 2: Determine Execution Mode

| Mode | When | BROWSER value |
|------|------|---------------|
| **Standalone** (default) | User doesn't mention TAHub, or says "chrome", "standalone", "browser" | `chrome` |
| **TAHub** | User explicitly says "TAHub", "TAHub mode", "CDP", "desktop app" | `TAHub` |

---

## Step 3: Apply Environment Variables

Use defaults unless the user provides overrides:

| Variable | Default | Override when user provides |
|----------|---------|---------------------------|
| `ENV` | `DEV` | Different environment name (UAT, INT, QA, SFIINT) |
| `BROWSER` | `chrome` | "TAHub" or "msedge" or "firefox" |
| `WORKSPACE` | `Regulatory - Tax Reporting` | Different workspace name |
| `APP` | `IRIS Reporting` | Micro-UI name — used for relaunching the micro-UI on failure to continue execution |
| `ExecutionKey` | `CTA-113520` | JIRA X-Ray execution ID — override if user provides a different one |
| `ENVIRONMENT` | *(not set)* | Tile environment (e.g., "MFD", "MFB") |

---

## Step 4: Build and Execute the Command

### Template — Standalone mode:
```bash
npx cross-env ENV={ENV} BROWSER=chrome WORKSPACE="{WORKSPACE}" APP="{APP}" ExecutionKey="{ExecutionKey}" cucumber-js --tags '{tags}' --name "{nameRegex}" 2>&1
```

### Template — TAHub mode:
```bash
npx cross-env BROWSER=TAHub WORKSPACE="{WORKSPACE}" APP="{APP}" ExecutionKey="{ExecutionKey}" cucumber-js --tags '{tags}' --name "{nameRegex}" 2>&1
```

### With all variables (when ENVIRONMENT is also needed):
```bash
npx cross-env ENV={ENV} BROWSER={BROWSER} WORKSPACE="{WORKSPACE}" APP="{APP}" ExecutionKey="{ExecutionKey}" ENVIRONMENT="{ENVIRONMENT}" cucumber-js --tags '{tags}' --name "{nameRegex}" 2>&1
```

Only include `ENVIRONMENT` if the user provides it — all other variables always have defaults and should always be included.

### Running a full feature (all scenarios):
When running ALL scenarios in a feature, omit `--name` entirely:
```bash
npx cross-env ENV=DEV BROWSER=chrome WORKSPACE="Regulatory - Tax Reporting" APP="IRIS Reporting" ExecutionKey="CTA-113520" cucumber-js --tags '@MFDFileMasterInfo' 2>&1
```

### Running specific scenarios:
```bash
npx cross-env ENV=DEV BROWSER=chrome WORKSPACE="Regulatory - Tax Reporting" APP="IRIS Reporting" ExecutionKey="CTA-113520" cucumber-js --tags '@MFDFileMasterInfo' --name "Login|column filtering|Logout" 2>&1
```

---

## Step 5: Generate HTML Report

After the test execution completes (whether passed or failed), **always** run the HTML report generator using the same `ENV` and `BROWSER` values from the test run:

```bash
npx cross-env ENV='{ENV}' BROWSER='{BROWSER}' node ./htmlReportGenerator.cjs 2>&1
```

Use the exact same `ENV` and `BROWSER` that were used in the cucumber-js command:
- If the test ran with `ENV=DEV BROWSER=chrome` → report uses `ENV='DEV' BROWSER='chrome'`
- If the test ran with `ENV=UAT BROWSER=msedge` → report uses `ENV='UAT' BROWSER='msedge'`
- If the test ran with `BROWSER=TAHub` (no ENV) → report uses `ENV='DEV' BROWSER='TAHub'` (default ENV to DEV)

---

## Step 6: Report Results

After execution, report:
- ✅ Total scenarios passed / failed
- ⏱️ Duration
- ❌ If any failed — show the failure message and which step failed
- 📊 HTML report generated (note pass percentage if shown in output)

---

## Examples

**User says**: "run FileMasterInfo"
```bash
npx cross-env ENV=DEV BROWSER=chrome WORKSPACE="Regulatory - Tax Reporting" APP="IRIS Reporting" ExecutionKey="CTA-113520" cucumber-js --tags '@MFDFileMasterInfo' 2>&1
```

**User says**: "run the column filtering scenarios"
```bash
npx cross-env ENV=DEV BROWSER=chrome WORKSPACE="Regulatory - Tax Reporting" APP="IRIS Reporting" ExecutionKey="CTA-113520" cucumber-js --tags '@MFDFileMasterInfo' --name "Login|column filtering|Logout" 2>&1
```

**User says**: "run @CTA-109319 and @CTA-109320"
→ Grep to find these are "column filtering" scenarios in FileMasterInfo.feature
```bash
npx cross-env ENV=DEV BROWSER=chrome WORKSPACE="Regulatory - Tax Reporting" APP="IRIS Reporting" ExecutionKey="CTA-113520" cucumber-js --tags '@MFDFileMasterInfo' --name "Login|column filtering|Logout" 2>&1
```

**User says**: "run FileMasterInfo in TAHub mode"
```bash
npx cross-env BROWSER=TAHub WORKSPACE="Regulatory - Tax Reporting" APP="IRIS Reporting" ExecutionKey="CTA-113520" cucumber-js --tags '@MFDFileMasterInfo' 2>&1
```

**User says**: "run @IRISRegression in UAT"
```bash
npx cross-env ENV=UAT BROWSER=chrome WORKSPACE="Regulatory - Tax Reporting" APP="IRIS Reporting" ExecutionKey="CTA-113520" cucumber-js --tags '@IRISRegression' 2>&1
```

**User says**: "run pagination and sorting scenarios from FileMasterInfo"
```bash
npx cross-env ENV=DEV BROWSER=chrome WORKSPACE="Regulatory - Tax Reporting" APP="IRIS Reporting" ExecutionKey="CTA-113520" cucumber-js --tags '@MFDFileMasterInfo' --name "Login|pagination|sort|Logout" 2>&1
```

**User says**: "run @CTA-108502 and @CTA-116706"
→ Grep to find these are in user_permissions_AUTO2.feature (shared tag @MFDUserPermissions)
→ Use file path approach since @MFDUserPermissions is shared across AUTO2/3/5/6
```bash
npx cross-env ENV=DEV BROWSER=chrome WORKSPACE="Regulatory - Tax Reporting" APP="IRIS Reporting" ExecutionKey="CTA-113520" cucumber-js features/RegulatoryTaxReporting/Iris/MFD/UserPermissions/user_permissions_AUTO2.feature --name "Login|Accepted with access level .D.|Generate Corrected File|Logout" 2>&1
```

**User says**: "run user_permissions_AUTO5 feature"
→ Feature has shared tag @MFDUserPermissions — use file path to avoid running AUTO2/3/6
```bash
npx cross-env ENV=DEV BROWSER=chrome WORKSPACE="Regulatory - Tax Reporting" APP="IRIS Reporting" ExecutionKey="CTA-113520" cucumber-js features/RegulatoryTaxReporting/Iris/MFD/UserPermissions/user_permissions_AUTO5.feature 2>&1
```
