# ACME Client Security Hash: Dispatcher

A UiPath process that reads the Work Items list in ACME System 1 and adds one Orchestrator queue item for every open work item of type WI5. It is the Dispatcher half of a Dispatcher/Performer solution based on the UiPath Academy exercise "Calculate Client Security Hash". A separate REFramework Performer processes the queue.

## What it does

1. Reads the ACME URL and login from Orchestrator assets.
2. Opens Chrome and logs in to ACME.
3. Opens Work Items and extracts the whole table, following the Next link through every page.
4. Checks that no page was skipped or read twice.
5. Keeps the rows whose Type is `WI5` and Status is `Open`.
6. Adds one queue item per row, using the work item ID (WIID) as the unique reference.
7. Logs a summary and closes the browser, also when a step fails.

## Project structure

| Workflow | Purpose |
|---|---|
| `Main.xaml` | Orchestration, error handling, summary log |
| `Framework\GetOrchestratorAssets.xaml` | Get Asset for the ACME URL |
| `ACME\ACME_Login.xaml` | Open the browser, read the credential, log in, confirm the dashboard |
| `ACME\ACME_NavigateToWorkItems.xaml` | Go to the dashboard if needed, open Work Items |
| `ACME\ACME_ExtractWorkItems.xaml` | Extract all pages and verify completeness |
| `ACME\ACME_Close.xaml` | Log out and close the browser |
| `Dispatcher\FilterWorkItems.xaml` | Validate columns, filter by type and status |
| `Dispatcher\AddWorkItemsToQueue.xaml` | Add queue items, skip duplicates |

Workflows under `ACME\` only drive the user interface. Workflows under `Dispatcher\` only hold business rules. A change to the ACME screens therefore touches one folder, and a change to the dispatch rules touches the other.

## Requirements

- UiPath Studio 2026 (Windows project, VB expressions), Chrome with the UiPath extension
- Packages: `UiPath.System.Activities` 26.8.2, `UiPath.UIAutomation.Activities` 26.10.5
- An ACME System 1 account at https://acme-test.uipath.com/

Orchestrator objects, all in the same folder:

| Type | Name | Notes |
|---|---|---|
| Queue | `ACME_ClientSecurityHash` | Unique Reference = Yes, Auto Retry = Yes, max 2 retries |
| Credential asset | `ACME_Credential` | ACME email and password |
| Text asset | `ACME_URL` | `https://acme-test.uipath.com/` |

## Arguments

All arguments of `Main.xaml` have defaults, so the process runs without input. They can be overridden when starting the job.

| Argument | Default | Meaning |
|---|---|---|
| `in_QueueName` | `ACME_ClientSecurityHash` | Target queue |
| `in_UrlAssetName` | `ACME_URL` | Asset holding the ACME URL |
| `in_CredentialAssetName` | `ACME_Credential` | Asset holding the login |
| `in_WorkItemType` | `WI5` | Work item type to dispatch |
| `in_WorkItemStatus` | `Open` | Work item status to dispatch |
| `in_PageSize` | `10` | Rows per page in the Work Items list |

## Queue item contract

The Dispatcher defines the data the Performer receives.

| Field | Value |
|---|---|
| Reference | WIID |
| `WIID` | Work item ID; the Performer opens the work item with it |
| `Type` | Work item type, for example `WI5` |
| `Description` | Work item description |
| `Status` | Status at the time of dispatch |
| `Date` | Work item date as shown in ACME |

All values are text.

## Design decisions

**Safe to re-run.** The queue enforces unique references, and the Dispatcher treats a "Duplicate Reference" error as a skip, logged as a warning. Running it twice adds nothing the second time.

**Complete extraction is verified.** The workflow reads the number of pages from the pager, extracts exactly that many pages, then checks that the row count fits and that every WIID is present exactly once. If a check fails, navigation and extraction are retried up to three times; both steps only read data, so repeating them is harmless.

**Login tolerates an open session.** If ACME is already logged in, the login form is skipped and the run continues from the dashboard.

**The browser always closes.** Closing runs in a Finally block, so it happens on success and on failure. A failure to close is logged and does not hide the original error.

**One bad row does not stop the batch.** Each queue item is added inside its own Try Catch. Failures other than duplicates are counted and logged as errors, and the job is marked Faulted at the end so they are not missed.

**No secrets in the project.** The credential is read inside the Login workflow, in the same scope where it is typed. The password stays a SecureString, is entered with Secure Text, and is never passed between workflows, converted to plain text or logged.

## Logging

Every log line carries three extra fields: `ProcessRole`, `QueueName` and `WorkItemType`. The run ends with one summary line, for example:

```
Dispatcher summary | Extracted=123 | Filtered=17 | Added=17 | DuplicatesSkipped=0 | Failed=0 | Pages=13
```

| Level | Used for |
|---|---|
| Info | Start and end of each workflow, counts, summary |
| Warn | Duplicate skipped, extraction retried, browser could not be closed |
| Error | A queue item failed, or the run failed |
| Trace | Each item added |

## Operating notes

- **After resetting ACME test data, delete the old queue items before re-running.** Unique references also cover completed items, so work item IDs that reappear after a reset would be skipped as duplicates.
- The process is attended: it needs a desktop session with Chrome.
- If ACME shows fewer or more than 10 rows per page, set `in_PageSize` to match.
