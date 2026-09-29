# PDD - No-PO Invoice Chaser

## Document History

| Date | Version | Author | Role | Comments |
|---|---|---|---|---|
| 2026-09-29 | 0.1 | uipath-analyst | Analyst | Initial PDD created from docs/jactiv-797-request-details.docx. |

| Date | Version | Author | Role | Comments |
|------|---------|--------|------|----------|
| 2026-09-29 | 0.1 | uipath-analyst | Analyst | Initial analysis from request-work/request-details.md (UiPath Cartographer, v1.0, 16 Sep 2026) |

## 1. Document Control

| Field | Value |
|-------|-------|
| Document | PDD - No-PO Invoice Chaser |
| Story key | JACTIV-797 |
| Epic key | [SME REVIEW] |
| Source file | request-work/request-details.md |
| Branch | analysis-jactiv-797 |
| Author | uipath-analyst |
| Status | Draft - pending SME approval |
| Version | 0.1 |

## 2. Introduction

| Field | Value |
|-------|-------|
| Process name | No-PO Invoice Chaser |
| Process full name | NoPoInvoiceChaser |
| Business objective | Replace daily manual Coupa review with a fully automated weekday run that identifies invoices with no linked PO and notifies the AP SME via Slack, enforcing the no-PO-no-pay policy consistently. |
| Owning department | Accounts Payable |

**Delivery Team**

| Role | Name / Contact |
|------|---------------|
| SME / Process Owner | Irina Capatina (irina.capatina@uipath.com, Slack ID WLX9BD8FN) |
| BA | uipath-analyst |
| Developer | [SME REVIEW] |

## 3. Process Overview

| Row | Value |
|-----|-------|
| Process full name | NoPoInvoiceChaser |
| Function and department | Accounts Payable – invoice compliance monitoring |
| Short description | Queries Coupa for draft/new invoices from the past seven days, excludes credit notes, counts those with no properly linked PO, and sends one Slack Block Kit message to the AP SME; sends nothing on a clean day. |
| Required roles | Automation (unattended); SME receives output only |
| Trigger and schedule | Weekday schedule at 10:00 Romania time (EET/EEST) |
| Volume (items per day / peak) | [SME REVIEW] – sample run cited 194 qualifying invoices; typical daily volume unknown |
| Average handling time | Manual: ~daily ad-hoc review; Automated target: <2 min per run [DEFAULT] |
| FTE effort | [SME REVIEW] |
| Estimated exception rate | Low – credit notes and description-only POs are the only named exceptions [SME REVIEW] |
| Input data | Coupa invoice list: invoice date, status, PO linkage, invoice type |
| Output data | Slack Block Kit DM to SME with qualifying count, policy note, action request, and filtered Coupa URL; or no message on a clean day |

## 4. To-Be Process (High Level)

The automation runs as a fully unattended weekday process. At 10:00 Romania time the scheduler fires the robot, which queries Coupa for all invoices dated within the past seven days with status draft or new, excludes credit notes, and counts those where no purchase order is properly linked (a PO number in the description field only does not qualify). If the count is greater than zero, one Slack Block Kit direct message is sent to the AP SME containing the count, a policy reminder, an action request, and a link to the filtered Coupa invoice list. If the count is zero, nothing is sent.

**Steps eliminated from the manual process:**
- Manual Coupa login and filter-panel interaction
- Manual field-copying into a message draft
- Manual grouping of invoices by requester
- Inconsistent daily timing dependent on AP staff availability

**Stays human:** The SME acts on the notification — raising or linking POs — and requester follow-up tracking remains outside scope.

## 5. Detailed Process Steps

| Step | Action | Application | Expected Result | Remarks |
|------|--------|-------------|-----------------|---------|
| 1.1 | Scheduler fires at 10:00 Romania time on a weekday | Scheduler | Run begins; timezone is EET/EEST [SME REVIEW for DST handling] | BR-04 trigger |
| 1.2 | Calculate date window: window_end = today, window_start = today − 7 days | Automation | window_start and window_end variables set | BR-04 |
| 1.3 | Authenticate to Coupa using stored credentials | Coupa | Authenticated session / API token acquired | Credential source [SME REVIEW] |
| 2.1 | Query Coupa invoice list with filters: invoice_date >= window_start, invoice_date <= window_end, status in (draft, new) | Coupa | Raw invoice record set returned | BR-04; API or web UI [SME REVIEW] |
| 2.2 | For each invoice record: check invoice type — if credit note, skip | Automation | Credit notes removed from working set | BR-03; exclusion reason logged [DEFAULT] |
| 2.3 | For each remaining invoice: check PO linkage field — if empty or invalid link, mark as qualifying; if PO number appears only in description field, also mark as qualifying | Automation | Qualifying invoice count incremented | BR-01, BR-02 |
| 2.4 | Record qualifying_count = number of marked invoices | Automation | qualifying_count variable set | BR-05 |
| 3.1 | If qualifying_count = 0, end run without sending message | Automation | Run completes, no Slack message sent | BR-07; contradicts exception row in source §7.1 — see OQ-01 |
| 3.2 | If qualifying_count > 0, build Coupa filtered URL using window_start and window_end and status=draft | Automation | coupa_url variable set | URL pattern from §4.3 of source |
| 3.3 | Compose Slack Block Kit payload: title `:receipt: {{invoice_count}} invoices need a purchase order`; body three icon-led lines; primary button "Open the list in Coupa" linking to coupa_url; footer with window dates and run date | Automation | Block Kit JSON payload ready | Placeholders: invoice_count, coupa_url, window_start, window_end, run_date; exact JSON in source §4.3 architectural notes |
| 4.1 | Send Block Kit payload as DM to Slack member ID WLX9BD8FN via Slack HTTP Request activity | Slack | Message delivered; 200 OK response received | BR-06; addressed by member ID, not email |
| 4.2 | Log run result (qualifying_count, run_date, send status) | Automation | Run log entry written | [DEFAULT] |
| 4.3 | End run | Automation | Run complete | BR-08: no retry or recovery |

## 6. Applications and Systems

| Application | Interface type | Access method | Login method | Credential handling | Comments |
|-------------|---------------|---------------|--------------|--------------------|----|
| Coupa | API [SME REVIEW] | REST API or web UI [SME REVIEW] | Service account / API key [SME REVIEW] | UiPath Orchestrator credential store [DEFAULT] | Read-only; base URL uipath-test.coupahost.com visible in source sample |
| Slack | API (HTTP) | Slack HTTP Request activity / Incoming Webhook or Bot Token [SME REVIEW] | Bot OAuth token or Webhook URL [SME REVIEW] | UiPath Orchestrator credential store [DEFAULT] | DM to member ID WLX9BD8FN; Block Kit payload |
| UiPath Orchestrator | Scheduler + credential store | Orchestrator API | Robot credential | Orchestrator-managed [DEFAULT] | Hosts schedule, credentials and run log |

## 7. Business Rules

| ID | Rule | Source | Applies at step |
|----|------|--------|----------------|
| BR-01 | Apply the no-PO-no-pay policy: an invoice without a properly linked PO must be flagged. | BR-001 | 2.3 |
| BR-02 | A PO number typed into the invoice description but not linked in the PO linkage field does not satisfy the PO requirement. | BR-002 | 2.3 |
| BR-03 | Exclude credit notes from the qualifying population. | BR-003 | 2.2 |
| BR-04 | Include only invoices with status draft or new and invoice date within the past seven days (window_start to window_end). | BR-004 | 1.2, 2.1 |
| BR-05 | Count the qualifying invoices; report that count only — individual invoices are not listed. | BR-005 | 2.4, 3.3 |
| BR-06 | Send the count to SME Irina Capatina (Slack member ID WLX9BD8FN) as a Block Kit DM with policy note, action request, and filtered Coupa link. | BR-006 | 4.1 |
| BR-07 | Send nothing when a successful query returns zero qualifying invoices. | BR-007 | 3.1 |
| BR-08 | No retry, fallback or recovery behaviour is required; a run that cannot complete is reported as a failed run. | BR-008 | 4.3 |
| BR-09 | A run that cannot complete produces no notification; no second message path exists. | BR-009 | 4.3 |
| BR-10 | The automation does not create or modify purchase orders, approve invoices, change Coupa records, or track requester completion. | BR-010 | All |

## 8. Business Exceptions

| ID | Name | Trigger step | Trigger condition | Action |
|----|------|-------------|------------------|--------|
| B1 | Credit note | 2.2 | Invoice type = credit note | Exclude from working set; log exclusion reason |
| B2 | Description-only PO | 2.3 | PO number appears only in description field, not in PO linkage field | Treat as missing PO; include in qualifying count if other rules pass |
| B3 | No qualifying invoices | 3.1 | qualifying_count = 0 after all filters | Send nothing (BR-07); see OQ-01 for conflict with source §7.1 |

## 9. System Errors

| ID | Name | Trigger condition | Severity | Retry policy | Action |
|----|------|------------------|----------|-------------|--------|
| S1 | Coupa authentication failure | Cannot acquire session or API token | High | No retry (BR-08) | Log failure; report run as failed |
| S2 | Coupa query error | API/UI returns error or unexpected response | High | No retry (BR-08) | Log error; report run as failed |
| S3 | Slack delivery failure | HTTP Request to Slack returns non-200 or throws | High | No retry (BR-08) | Log error; report run as failed |
| S4 | Application unresponsive | Coupa or Slack unreachable at connection time | High | No retry (BR-08) [DEFAULT] | Log; report run as failed |
| S5 | Credential expiry | Orchestrator credential retrieval fails | High | No retry [DEFAULT] | Log; alert Orchestrator admin [DEFAULT] |
| S6 | Unhandled exception | Unexpected error in any step | High | No retry (BR-08) [DEFAULT] | Log full stack trace; report run as failed |

## 10. Assumptions, Dependencies and Open Questions

1. **OQ-01 - Clean-day behaviour conflict.** BR-07 (source BR-007) says send nothing on a clean day; §7.1 exception table says send a congratulations Slack message — one of these must be authoritative. **[SME REVIEW]**
2. **OQ-02 - Coupa access method.** Source lists Coupa access type as "read" but does not specify API vs web UI; the build method depends on whether a REST API key is available. **[SME REVIEW]**
3. **OQ-03 - Slack integration mechanism.** Source specifies Slack HTTP Request activity but does not confirm Bot Token vs Incoming Webhook; token scope and channel permissions must be confirmed. **[SME REVIEW]**
4. **OQ-04 - Coupa URL status filter.** The sample URL filters to status=draft only; BR-04 requires draft and new — confirm whether the Coupa URL supports both status values simultaneously. **[SME REVIEW]**
5. **OQ-05 - DST and timezone.** 10:00 Romania time shifts between EET (UTC+2) and EEST (UTC+3); confirm Orchestrator schedule handles DST or is set in local Romania time. **[SME REVIEW]**
6. **OQ-06 - Block Kit JSON payload location.** Source §4.3 states the exact Block Kit JSON is in "architectural considerations, section 4" — this content is not present in the source document supplied; it is a missing input. **[SME REVIEW]**
7. **Credential storage default.** UiPath Orchestrator credential store is assumed for all credentials. **[DEFAULT]**
8. **Run logging default.** Run outcome (count, date, status) is logged to Orchestrator by default; no separate audit store is designed here. **[DEFAULT]**
9. **OQ-07 - Requester field.** Source §3.1 notes AP groups by requester in the manual process; confirm whether the requester field is needed by the automation or only relevant to the eliminated manual grouping step. **[SME REVIEW]**

## 11. Success Criteria

1. A weekday test run executes at 10:00 Romania time and completes without error.
2. Invoices with status draft or new and invoice date within the past seven days are evaluated; all others are excluded.
3. Credit notes are excluded from the qualifying count.
4. An invoice with a PO number only in the description field is counted as having no linked PO.
5. Exactly one Slack DM is sent to Slack member ID WLX9BD8FN containing the qualifying count and a working filtered Coupa link when count > 0.
6. No Slack message is sent when the qualifying count is zero (pending resolution of OQ-01).
7. No Coupa record is created, modified, or deleted by any run.
8. A run that fails at any step is recorded as a failed run in Orchestrator with no Slack notification sent.
