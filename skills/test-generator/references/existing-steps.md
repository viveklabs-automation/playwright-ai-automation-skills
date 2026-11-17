# Existing Step Definitions Reference

This file catalogs all existing step definitions available for reuse. Always check here before creating new steps.

## Global Steps (`src/steps/Global/global.steps.ts`)

| Step | Parameters | Description |
|------|-----------|-------------|
| `Given User logs into the application with` | DataTable (Username, Password) | Login and launch browser |
| `Given User logs out from the application` | None | Logout and cleanup |
| `Given User has X-Ray Test Case Id` | DataTable | Set Xray test case ID |

## Common Steps (`src/steps/RegulatoryTaxReporting/common.steps.ts`)

| Step | Parameters | Description |
|------|-----------|-------------|
| `Then User selects System and Fund Sponsors as {string}` | fund sponsor name | Select system/fund sponsor dropdown |
| `Then User is in landing page` | None | Verify landing page visible |
| `Then User clicks on Continue button when it appeared` | None | Click Continue if visible |
| `Given User selects {string} feature card` | feature card name | Click on a feature card |
| `Given User moves to the {string} feature card` | feature card name | Scroll to feature card |
| `When User selects {string} button` | button name | Click a button by name |

## File Master Info Steps (`src/steps/RegulatoryTaxReporting/Iris/filemasterInfo.steps.ts`)

| Step | Parameters | Description |
|------|-----------|-------------|
| `Then User validates the IRIS {string} and {string}` | header, description | Verify header and description text |
| `Then User sees the File Master Information screen is displayed` | None | Verify FMI screen loaded |
| `Then User closes the IRIS Reporting screen` | None | Close IRIS page |
| `Then User sees the Active Files, Archive Files tabs` | None | Verify both tabs visible |
| `Then User sees the Active Files Tab is selected and viewed upon launching the screen` | None | Verify Active tab is default |
| `Then User sees the Tax Year` | None | Verify tax year visible |
| `Then User sees the {string} tab with columns {string}` | grid name, columns CSV | Verify grid column headers |
| `Then User sees the Reset filter, Export icons in {string} tab` | tab name | Verify grid icons |
| `Then User sees the {string} button exists` | button name | Verify button visible |
| `Then User clicks on the columns of the {string} to see if sorting happens in ascending order` | grid name | Verify ascending sort |
| `Then User clicks on the columns of the {string} to see if sorting happens in descending order` | grid name | Verify descending sort |
| `When User clicks on {string} Tab` | tab name | Click Active/Archived Files tab |
| `Then User validates the pagination feature is working in forward direction in {string}` | tab name | Verify forward pagination |
| `Then User validates the pagination feature is working in backward direction in {string}` | tab name | Verify backward pagination |
| `Then User validates the filter is working in {string}` | tab name | Verify column filtering |
| `When User looks for the file that has action icon entry present in {string}` | tab name | Find file with actions |
| `When User selects the {string} menu option from the Actions column in {string} tab` | menu option, tab | Click action menu option |
| `Then User sees a warning message {string} relating to make the file Archived` | message text | Verify archive warning |
| `Then User clicks on the {string} button relating to make the file Archived` | button name | Click Yes/No on archive warning |
| `Then User clicks on the {string} button relating to make the file Active` | button name | Click Yes/No on unarchive warning |
| `Then User looks for the file that is in {string} but it is not present` | tab name | Verify file NOT in tab |
| `Then User looks for the file that is in {string} but it is present` | tab name | Verify file IS in tab |
| `Then User resets the filters in {string} tab` | tab name | Reset grid filters |
| `Then User moves to {string} menu and clicks on it` | menu name | Click side nav menu |
| `Then User enters an alias in {string} textbox in {string} tab` | textbox name, tab | Enter 15-char alias |
| `Then User enters a long alias name in {string} textbox in {string} tab` | textbox name, tab | Enter 45-char alias |
| `Then User clicks on the {string} button relating to the file alias` | button name | Click Save/Cancel/Delete on alias |
| `Then User sees the success message {string}` | message text | Verify success toast |
| `Then User verifies the updated Alias in its column in {string} tab` | tab name | Verify alias in grid |
| `Then User verifies the updated Alias in its column having eliipeses in {string} tab` | tab name | Verify truncated alias |
| `Then User verifies the Alias is not added in its column in {string} tab` | tab name | Verify no alias added |
| `When User looks for the file that has action icon entry and alias are present in {string}` | tab name | Find file with actions AND alias |
| `When User sees the Alias Information dialog` | None | Verify manage alias dialog |
| `When User sees {string} button on the Alias Information` | button name | Verify button on alias dialog |
| `When User sees New Alias textbox on the Alias Information` | None | Verify New Alias input |
| `Then User verifies the Alias is not updated in its column in {string} tab` | tab name | Verify alias unchanged |
| `Then User sees confirmation dialog relating to Updating Alias is present` | None | Verify update confirmation |
| `Then User verifies the Alias is updated in its column in {string} tab` | tab name | Verify alias was updated |
| `Then User sees confirmation dialog relating to Deleting Alias is present` | None | Verify delete confirmation |
| `Then User verifies the Alias is removed in its column in {string} tab` | tab name | Verify alias removed |
| `When User sees the Add Alias dialog` | None | Verify add alias dialog |
| `Then User verifies the Alias is not removed in its column in {string} tab` | tab name | Verify alias NOT removed |
| `Then User changes System and Fund Sponsors to {string}` | fund sponsor | Change management on IRIS screen |

## User Permissions Steps (`src/steps/RegulatoryTaxReporting/Iris/userPermissions.steps.ts`)

| Step | Parameters | Description |
|------|-----------|-------------|
| `Then User sees the {string} feature card is {string}` | card name, enabled/disabled | Verify feature card status |
| `Then User sees the {string} menu is {string}` | menu name, enabled/disabled | Verify side nav menu status |
| `Then User searches for {string} status in {string} tab` | status, grid name | Filter by status via Search Criteria |
| `Then User sees no menu renderer in action column in {string} tab` | grid name | Verify no action icons |
| `When User clicks on the action icon entry in {string}` | tab name | Click action icon |
| `Then User sees menu options such as {string} in {string} tab` | menus CSV, tab | Verify action menu options |
