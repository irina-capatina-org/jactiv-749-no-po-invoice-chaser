# PDD - No-PO Invoice Chaser

## Document History

| Date | Version | Author | Role | Comments |
|------|---------|--------|------|----------|
| 2026-09-23 | 0.1 | uipath-analyst | Analyst | Initial PDD created from request-work/request-details.md (converted from docs/jactiv-749-request-details.docx). |
| 2026-09-23 | 0.2 | uipath-analyst | Analyst | Restructured sections to match required PDD schema; added Scope and Data Definitions sections. |
| 2026-09-23 | 0.3 | uipath-analyst | Analyst | Validated all 13 required section headings present and BR-nn rule IDs normalised. |

## 1. Document Control

| Field | Value |
|-------|-------|
| Document | PDD - No-PO Invoice Chaser |
| Story key | JACTIV-749 |
| Epic key | AP Automation |
| Source file | request-work/request-details.md |
| Branch | analysis-jactiv-749 |
| Author | uipath-analyst |
| Status | Draft - pending SME approval |
| Version | 0.2 |

## 2. Introduction

**Process name:** No-PO Invoice Chaser

**Process Full Name:** `NoPoInvoiceChaser`

| Field | Value |
|-------|-------|
| Process Full Name | NoPoInvoiceChaser |
| Business objective | Automatically detect and report invoices with no properly linked purchase order each weekday morning, enforcing the no-PO-no-pay policy and eliminating manual AP review effort |
| Owning department | Accounts Payable / Finance |

**Delivery Team**

| Role | Name / Contact |
|------|---------------|
| SME / Process Owner | Irina Capatina (irina.capatina@uipath.com, Slack ID WLX9BD8FN) |
| BA | uipath-analyst |
| Developer | [SME REVIEW] |

## 3. Process Overview

| Field | Value |
|-------|-------|
| Process full name | NoPoInvoiceChaser |
| Function and department | Accounts Payable – invoice compliance control |
| Short description | Weekday automated run that queries Coupa for invoices from the past seven days with status draft or new and no properly linked PO, then sends a single Slack summary to the AP SME; sends nothing on a clean day |
| Required roles | Automation service account (Coupa read, Slack write); no human roles in the normal path |
| Trigger and schedule | Time-based: weekdays at 10:00 Romania time (EET/EEST) via UiPath Orchestrator |
| Volume (items per day / peak) | ~194 qualifying invoices per run observed in live sample; daily average [SME REVIEW] |
| Average handling time | Manual: ~daily ad-hoc effort, timing inconsistent; Automated target: <2 minutes end-to-end |
| FTE effort | [SME REVIEW] |
| Estimated exception rate | Low – credit notes and description-only POs are the only known business exceptions |
| Input data | Coupa invoice records: invoice date, status, PO linkage at line level, invoice type |
| Output data | One Slack Block Kit direct message to SME with qualifying invoice count and filtered Coupa URL; or no message on a clean day |

## 4. Scope

**In Scope**

The automation covers the following activities:

- Weekday execution at 10:00 Romania time triggered by UiPath Orchestrator scheduler
- Querying Coupa via REST API to retrieve invoices from the past seven days
- Filtering invoices to status draft or new only
- Excluding credit notes from the qualifying population
- Detecting missing or invalid PO linkage at invoice line level (a PO number in the description field only does not qualify)
- Counting qualifying invoices and building a filtered Coupa URL for the same date window
- Sending one Slack Block Kit direct message to the SME (Irina Capatina, Slack member ID WLX9BD8FN) when one or more invoices qualify
- Suppressing the Slack message entirely on a clean day (zero qualifying invoices)

**Out of Scope**

The following are explicitly excluded from this automation:

- Purchase-order creation or modification
- Invoice approval or payment release
- Any Coupa record modification (read-only access only)
- Supplier communication
- Requester follow-up tracking or escalation
- Listing individual invoice details in the Slack message
- Retry, fallback and error-recovery behaviour beyond what the UiPath Orchestrator default provides
- Full invoice lifecycle automation

| **In Scope** | **Out of Scope** |
|---|---|
| Weekday execution at 10:00 Romania time | Purchase-order creation |
| Coupa invoice retrieval via REST API | Invoice approval or payment release |
| Past-seven-day date window filtering | Coupa record modification |
| Status draft or new filtering | Supplier communication |
| Credit-note exclusion | Requester follow-up tracking |
| Missing or invalid PO-link detection at line level | Listing individual invoices in the Slack message |
| Summary count of invoices without a linked PO | Retry, fallback and error-recovery behaviour |
| One Slack Block Kit message to the SME on qualifying days | Full invoice lifecycle automation |
| No message sent on a clean day | Audit-retention design |

## 5. To-Be Process (High Level)

The automation replaces the daily manual AP review entirely. A weekday schedule triggers the bot at 10:00 Romania time; it queries Coupa for the past seven days' invoices, applies status, date, PO-linkage and credit-note filters, counts qualifying records, and sends one Slack Block Kit direct message to the SME. On a clean day it sends nothing.

The process is a short linear sequence — read, filter, format, send — with no human decision points in the normal path and no AI classification. The automation operates within a strict notification boundary: it reads data and sends one message; it does not alter invoices or create POs.

**Manual steps eliminated:**

- AP member opening and filtering the Coupa invoice list daily
- Copying invoice fields and grouping by requester
- Composing and sending the Slack message

**What stays human:** The SME acts on the notification by ensuring a PO is raised and linked for each flagged invoice. Requester follow-up tracking is out of scope.

**Automation boundary:** Read and notify only. No Coupa writes, no PO creation, no invoice approval.

| **Step** | **Automation unit** | **Mode** | **Owner or system** | **Input** | **Output** | **Exception path** |
|---|---|---|---|---|---|---|
| 1 | Start weekday run | Fully automated | UiPath Orchestrator schedule | Schedule and Romania timezone | Run context initialised | None |
| 2 | Query and filter Coupa invoices | Fully automated | Coupa REST API | Date window and status filter | Qualifying invoice records | Invalid response → failed run |
| 3 | Compose the summary message | Fully automated | Automation | Qualifying invoice records | One Slack Block Kit payload | Zero count → no message sent |
| 4 | Deliver Slack notification | Fully automated | Slack API | Prepared Block Kit payload | Delivered direct message to SME | Delivery failure → failed run |

## 6. Detailed Process Steps

| Step | Action | Application | Expected Result | Remarks |
|------|--------|-------------|-----------------|---------|
| 1.1 | Orchestrator triggers the process on the weekday schedule | UiPath Orchestrator | Process starts; run context initialised with run date and Romania timezone | Schedule fires Mon–Fri at 10:00 EET/EEST (BR-04) |
| 1.2 | Calculate the invoice date window: window_end = today; window_start = today minus 7 days | Automation | window_start and window_end values available for Coupa query and Slack message footer | BR-04; dates formatted for Coupa query parameter and for display in message footer |
| 2.1 | Call Coupa REST API to retrieve invoices where invoice_date >= window_start AND invoice_date <= window_end AND status IN (draft, new) | Coupa API | List of invoice records returned | Interface: Coupa REST API (read); credentials from Orchestrator asset |
| 2.2 | Validate the Coupa response: HTTP 200 and parseable JSON body | Coupa API | Valid: proceed to 2.3; Invalid: log failure, mark run as failed, terminate | Decision point: an invalid response terminates the run; no Slack notification is sent (BR-09) |
| 2.3 | For each invoice record: check invoice type; if type equals credit note, exclude the record and log the exclusion reason | Automation | Credit notes removed from working set | BR-03; exclusion reason logged for audit trail |
| 2.4 | For each remaining invoice record: check PO linkage at the invoice LINE level; if no properly linked PO exists on any line, flag the invoice as qualifying | Automation | Qualifying invoices identified | BR-01, BR-02; a PO number present only in the invoice description field does not satisfy this check |
| 2.5 | Count the qualifying invoices; store result as invoice_count | Automation | invoice_count integer available for downstream steps | BR-05; individual invoice details are not collected for the message |
| 3.1 | Decision: is invoice_count equal to zero? | Automation | Yes → step 3.2 (clean day, no message sent); No → step 3.3 | BR-07; a clean result is distinguishable from a technical failure |
| 3.2 | Clean day path: terminate run successfully with no Slack message sent | Automation | Process ends; no notification dispatched; run logged as successful clean run | BR-07 |
| 3.3 | Build the filtered Coupa URL: base URL + invoice_date_gteq=window_start + invoice_date_lteq=window_end + status_eq=draft | Automation | coupa_url string assembled for inclusion in the Slack message button | PO-linkage filter cannot be expressed in the URL; the count in the message conveys that number |
| 3.4 | Compose the Slack Block Kit message payload: title with invoice_count; three body lines with warning, no_entry, and point_right icons; primary action button linking to coupa_url; footer with window_start, window_end, and run_date | Automation | Complete Block Kit JSON payload ready for delivery | Recipient: Slack member ID WLX9BD8FN (Irina Capatina); invoice_count appears in both title and first body line; exact Block Kit JSON is recorded in the SDD architectural considerations |
| 4.1 | POST the Block Kit payload to Slack as a direct message to member ID WLX9BD8FN | Slack API | HTTP 200 response and message delivered to SME inbox | BR-06; interface: Slack Web API chat.postMessage; credentials from Orchestrator asset |
| 4.2 | Validate the Slack API response: success means message delivered; failure means delivery could not be confirmed | Slack API | Delivered: proceed to 4.3 successfully; Failed: log failure, mark run as failed | Decision point; a failed delivery is recorded as a failed run |
| 4.3 | End run; log run outcome to UiPath Orchestrator | UiPath Orchestrator | Run outcome (success or failed) recorded | A failed run produces no business notification to the SME (BR-09) |

## 7. Applications and Systems

| Application | Interface type | Access method | Login method | Credential handling | Comments |
|-------------|---------------|---------------|-------------|---------------------|----------|
| Coupa | API (REST) | Coupa REST API – read invoices and line-level PO linkage data | OAuth2 / API key [SME REVIEW] | Stored in UiPath Orchestrator asset | Base URL: https://uipath-test.coupahost.com; read-only access required; PO linkage is at line level, not invoice header |
| Slack | API (HTTP) | Slack HTTP Request activity – direct message POST via chat.postMessage | Bot token | Stored in UiPath Orchestrator asset | Recipient addressed by Slack member ID WLX9BD8FN; Block Kit JSON payload |
| UiPath Orchestrator | Orchestrator (schedule + assets) | Weekday time trigger; credential asset storage | Robot service account | Managed by Orchestrator | Trigger: Mon–Fri 10:00 EET/EEST; stores Coupa and Slack credentials as Orchestrator assets |

## 8. Business Rules

| ID | Rule | Source | Applies at step |
|----|------|--------|-----------------|
| BR-01 | Apply the no-PO-no-pay policy: invoices without a properly linked purchase order at line level are flagged as qualifying | BR-001 | 2.4 |
| BR-02 | A PO number typed into the invoice description but not properly linked at line level does not satisfy the PO requirement; the invoice is treated as missing a PO | BR-002 | 2.4 |
| BR-03 | Exclude credit notes from the qualifying population; record the exclusion reason for each excluded invoice | BR-003 | 2.3 |
| BR-04 | Include only invoices with status draft or new and an invoice date within the past seven days from the run date | BR-004 | 1.2, 2.1 |
| BR-05 | Count the qualifying invoices; the Slack message reports that count only; individual invoices are not listed in the message | BR-005 | 2.5, 3.3 |
| BR-06 | Send the count to the SME (Irina Capatina, Slack member ID WLX9BD8FN) by Slack direct message, with an explanation of the no-PO-no-pay policy impact, a request to link a PO, and a filtered Coupa URL covering the same date window | BR-006 | 3.4, 4.1 |
| BR-07 | Send nothing when a successful query returns zero qualifying invoices; a clean day is logged as a successful run | BR-007 | 3.1, 3.2 |
| BR-08 | No retry or fallback behaviour is required by the business; a run that cannot complete is recorded as a failed run and produces no notification | BR-008 | 2.2, 4.2 |
| BR-09 | A run that cannot complete produces no Slack notification; the failure is recorded as a failed run in Orchestrator | BR-009 | 4.3 |
| BR-10 | The automation does not create or modify purchase orders, approve invoices, change Coupa records, or track requester completion | BR-010 | All steps |

## 9. Business Exceptions

| ID | Name | Trigger step | Trigger condition | Action |
|----|------|-------------|------------------|--------|
| B1 | Credit note encountered | 2.3 | Invoice type field equals credit note | Exclude the record from the qualifying set; log exclusion reason; continue processing the remaining invoices |
| B2 | Description-only PO | 2.4 | PO number appears only in the invoice description field and not as a proper line-level linkage | Treat as missing PO per BR-02; include in qualifying set if other rules pass |
| B3 | No qualifying invoices (clean day) | 3.1 | invoice_count equals zero after all filters applied | Terminate run successfully with no Slack message sent; log as a clean successful run distinct from a failed run per BR-07 |

## 10. System Errors

| ID | Name | Trigger condition | Severity | Action |
|----|------|------------------|----------|--------|
| S1 | Coupa API unavailable or invalid response | Step 2.1–2.2: HTTP error or unparseable response body returned from Coupa | High | Log failure details to Orchestrator; mark run as failed; no Slack notification sent per BR-09 |
| S2 | Slack delivery failure | Step 4.1–4.2: Slack API returns an error or delivery cannot be confirmed | High | Log failure details to Orchestrator; mark run as failed; no retry attempted per BR-08 |
| S3 | Network timeout | Any step: network call exceeds the configured timeout threshold | Medium | Log exception details; mark run as failed |
| S4 | Credential expiry or invalid credential | Step 2.1 or 4.1: Orchestrator asset returns an invalid or expired credential | High | Log exception; alert Orchestrator administrator; mark run as failed |
| S5 | Unhandled exception | Any step: unexpected runtime error not covered by S1–S4 | High | Log full exception detail to Orchestrator; mark run as failed; no partial notification sent |

## 11. Data Definitions

| Field name | Source system | Data type | Format / valid values | Used at step | Notes |
|---|---|---|---|---|---|
| invoice_date | Coupa | Date | ISO 8601 date (YYYY-MM-DD) | 1.2, 2.1 | Used to calculate the seven-day window; compared to window_start and window_end |
| status | Coupa | String | draft, new (in scope); other values excluded | 2.1 | Filter applied during Coupa API query |
| invoice_type | Coupa | String | standard invoice, credit note; credit notes excluded | 2.3 | BR-03: credit notes removed from qualifying set |
| po_linkage (line level) | Coupa | Reference / boolean | Properly linked PO reference on the invoice line; description-only text does not count | 2.4 | BR-01, BR-02; PO linkage is evaluated at line level, not invoice header |
| invoice_count | Automation | Integer | Non-negative integer; 0 = clean day | 2.5, 3.1, 3.4 | Count of qualifying invoices after all filters; drives the clean-day decision and the Slack message content |
| window_start | Automation | Date | ISO 8601 date; run_date minus 7 days | 1.2, 3.3, 3.4 | Inclusive lower bound of the invoice date filter |
| window_end | Automation | Date | ISO 8601 date; equal to run_date | 1.2, 3.3, 3.4 | Inclusive upper bound of the invoice date filter |
| run_date | Automation | Date | ISO 8601 date; date on which the automation executes | 1.2, 4.3 | Used in the Slack message footer and Orchestrator run log |
| coupa_url | Automation | String (URL) | Coupa invoice list URL with query parameters for window_start, window_end, and status_eq=draft | 3.3, 3.4 | Embedded in the Slack message action button; PO-linkage filter cannot be expressed as a URL parameter |
| slack_member_id | Configuration | String | WLX9BD8FN | 3.4, 4.1 | Fixed identifier for the SME Irina Capatina; used as the Slack direct message recipient |

## 12. Assumptions, Dependencies and Open Questions

1. **OQ-01 - Retry policy.** [SME REVIEW] BR-008 states no retry is required; confirm this governs over any diagram artefacts that show retry steps for Coupa and Slack.
2. **OQ-02 - Coupa API authentication method.** [SME REVIEW] Confirm whether Coupa access uses OAuth2 client credentials or an API key; required before credential asset setup in Orchestrator.
3. **OQ-03 - Clean-day Slack message.** [SME REVIEW] BR-007 states nothing is sent on a clean day; confirm this applies and that no congratulatory message is required on zero-result days.
4. **OQ-04 - Coupa URL status parameter scope.** [SME REVIEW] Confirm whether the filtered Coupa URL should include both draft and new status values or only draft, so the link matches the qualifying population.
5. **OQ-05 - Slack Bot token scope.** [DEFAULT] Assumes the Slack bot has chat:write and im:write scopes to send direct messages by Slack member ID.
6. **OQ-06 - UiPath Orchestrator environment.** [DEFAULT] Assumes an existing UiPath Orchestrator tenant with a licensed unattended robot and the ability to create time-based weekday triggers.
7. **OQ-07 - Romania timezone handling.** [DEFAULT] EET (UTC+2) / EEST (UTC+3) applied via Orchestrator trigger timezone setting; assumes Orchestrator supports the Europe/Bucharest timezone zone identifier.
8. **OQ-08 - Coupa API pagination.** [DEFAULT] Assumes the Coupa API response for the seven-day window either returns all results in a single page or that pagination is handled by iterating all pages before filtering.

## 13. Success Criteria

1. A weekday test run executes at 10:00 Romania time and completes without manual intervention.
2. Only invoices with status draft or new and an invoice date within the past seven days are evaluated; invoices outside the window or with other statuses are not included.
3. Credit notes are excluded from the qualifying count and the exclusion is logged with a reason.
4. An invoice with a PO number present only in the description field is counted as missing a properly linked PO.
5. The Slack Block Kit message is delivered to Slack member ID WLX9BD8FN (Irina Capatina) and contains the qualifying invoice count and a working filtered Coupa URL covering the same date window used in the query.
6. A run where zero invoices qualify produces no Slack message and is logged as a successful clean run.
7. A run that fails for any technical reason is recorded as a failed run in Orchestrator and no business notification is sent to the SME.
8. No Coupa record is created, modified or deleted during any run.
9. The automation does not create purchase orders or approve invoices under any circumstances.
