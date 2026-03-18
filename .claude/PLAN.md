# CloudBank Fund Sentinel — Project Plan

> Last updated: 2026-03-17
> Owner: CloudBank Operations Engineering
> Tool: N8N workflow automation
> Timeline: 4 weeks, ~5h/week (~20h total)

---

## Project Goal

Automate the CloudBank fund overspend follow-up workflow. Starting from Fund Audit Zendesk tickets (auto-generated daily by existing cron), the system should:

1. Parse fund IDs and PI emails from the ticket
2. Enrich each fund from Kion API
3. Sync to the Google Sheets operational tracker
4. Search Zendesk for existing tickets (dedup gate)
5. Create follow-up Zendesk tickets (LLM-written)
6. Drive multi-stage escalation with delay-based state management
7. Classify inbound replies (LLM) and advance state
8. Flag funds ready for soft-offboard to human operators

---

## Architecture

### System shape

```
Daily Cron (N8N)
  │
  ▼
Workflow 1: INGEST
  Read Fund Audit Zendesk ticket
  → parse HTML table (Code node)
  → extract: fund_id, pi_email, balances, overspend status
  → for each fund:
      → search Zendesk for existing tickets (dedup gate)
      → check Google Sheet: new or existing?
      → if new: Kion enrichment → append Sheet row
      → if existing: update balances in Sheet

Workflow 2: ESCALATE
  Scheduled (every 15min or daily)
  → read Sheet rows where status needs action
  → LLM: write follow-up ticket body
  → create Zendesk ticket (or add comment)
  → update Sheet: status, ticket_id, date_sent

Workflow 3: REPLY MONITOR
  Scheduled (daily or webhook)
  → poll Zendesk for replies on open follow-up tickets
  → LLM: classify reply (acknowledged / disputing / no response / human needed)
  → update Sheet: reply_status, next_action
  → if human needed: flag in Sheet + internal Zendesk note

Workflow 4: SOFT-OFFBOARD FLAGGING
  → LLM decision: is this fund ready for soft-offboard?
  → if yes: post internal Zendesk note to operator
  → human does the portal click (fund → Closing)
```

### State store

**Google Sheet** — remains the operational source of truth for MVP.

#### Sheet columns (system-owned)

| Column | Description |
|---|---|
| fund_id | Canonical identifier (from Zendesk ticket) |
| funder | NSF / NAIRR |
| pi_email | From Zendesk ticket |
| fund_name | From Kion |
| fund_start | From Kion |
| fund_end | From Kion |
| cash_award | From ticket |
| cash_spend | From ticket |
| cash_balance | From ticket |
| cash_balance_pct | From ticket |
| credit_award | From ticket |
| credit_spend | From ticket |
| credit_balance | From ticket |
| credit_balance_pct | From ticket |
| kion_ou_id | From Kion enrichment |
| cloud_accounts | From Kion (comma-separated) |
| cloud_providers | AWS / GCP / Azure |
| status | new / enriched / warning_1_sent / warning_2_sent / warning_3_sent / human_needed / resolved |
| source_ticket_id | Zendesk Fund Audit ticket ID |
| followup_ticket_id | Zendesk ticket created by this system |
| warning_1_date | ISO date |
| warning_2_date | ISO date |
| warning_3_date | ISO date |
| last_reply_date | ISO date |
| reply_classification | LLM output |
| offboard_ready | TRUE / FALSE |
| operator_notes | Human-editable — system never overwrites |
| last_enriched | ISO datetime |
| sentinel_version | Workflow version that last touched this row |

#### Operator-owned columns (system never writes)

- `operator_notes`
- Any columns to the right of `sentinel_version`

---

## Integrations

### Zendesk

- Auth: API token (already available)
- Read: Fund Audit tickets (search by view or tag)
- Search: existing tickets by fund_id (dedup gate)
- Write: create follow-up tickets, add internal notes
- Ticket format: HTML table, structured, parseable with regex

### Kion API

- Base URL: `https://kion.cloudbank.org/api/v3/`
- Auth: API token (already available)
- Key endpoints for MVP:
  - `GET /v3/funding-source/{id}` — name, dates, ou_id
  - `GET /v3/funding-source/{id}/financials` — balances (cross-check against ticket)
  - `GET /v3/funding-source/{id}/permission-mapping` — contacts
  - `GET /v3/user/{id}` — email, name
- Fund IDs in Zendesk ticket map directly to Kion `funding-source` IDs (validate in Week 1)

### Google Sheets

- Auth: Google service account (set up in N8N credentials)
- One sheet: operational tracker (test copy during dev, real copy in production)
- Read: check if fund_id exists in column A
- Append: new fund rows
- Update: specific columns only (never operator columns)

### LLM (Claude via Anthropic API)

- Used for:
  1. Parsing ticket HTML if regex fails
  2. Writing follow-up ticket body (given fund data → polished message)
  3. Classifying inbound replies
  4. Soft-offboard decision support
- Not used for: state transitions, date logic, idempotency checks

### CloudBank Portal (Drupal)

- No REST API available
- Soft-offboard (fund → Closing) is a **manual human action** in the portal for MVP
- The cascade from "Closing" is automatic in the portal: billing accounts → closing, IAM users → deactivated
- Phase 2 option: thin Drupal endpoint (~30 lines PHP) to automate this

### BigQuery

- Deferred to Phase 2 — Kion financials likely sufficient for MVP
- Tables available if needed: `cloudbank-project-admin.AllCloudUsage.DailySpend`

---

## Soft-Offboard Scope (MVP)

The system **flags** funds ready for soft-offboard. A human operator performs the portal actions:

1. Mark fund as "Closing" in CloudBank portal (cascades automatically)
2. Add negative transfer (balance → $5, or full amount if $0 spend)

The system does NOT automate:
- Cloud console actions (AWS/GCP/Azure resource deletion)
- Beam ARN removal
- Any action requiring cloud console access

---

## Phased Delivery

### Week 1 — Connectivity + Ingestion

- [ ] Set up N8N (cloud instance on Azure)
- [ ] Configure credentials: Zendesk, Kion, Google Sheets
- [ ] Create test Google Sheet (copy of real tracker)
- [ ] Create Zendesk test group (no external email notifications)
- [ ] Build Workflow 1: parse Fund Audit ticket → extract fund rows
- [ ] Validate: fund_id in ticket maps to Kion funding-source ID (spot-check 3 funds)
- [ ] Validate: Kion permission-mapping returns PI email (or confirm ticket email is sufficient)
- [ ] Validate: Zendesk search query finds tickets by fund_id

Deliverable: can parse a real Fund Audit ticket and enrich 3+ funds from Kion.

### Week 2 — Enrichment + Sheet Sync

- [ ] Complete enrichment node chain (Kion reads → Sheet row)
- [ ] Implement dedup gate (Zendesk search before ticket creation)
- [ ] Implement Sheet check: new vs. existing fund
- [ ] Append new fund rows to test Sheet
- [ ] Update balances on existing rows
- [ ] Run idempotency test: same ticket twice → no duplicates

Deliverable: full ingest + enrich + sheet sync working end-to-end.

### Week 3 — Escalation + LLM Tickets

- [ ] Build Workflow 2: escalation state reader
- [ ] LLM node: write follow-up ticket body (template + LLM polish)
- [ ] Create Zendesk ticket in test group (no real PI contact)
- [ ] Review 5 generated tickets manually — adjust prompt if needed
- [ ] Add dry-run switch: all writes go to DryRunLog Sheet tab when enabled
- [ ] Two clean dry runs on real Fund Audit tickets

Deliverable: system creates well-written follow-up tickets, dry-run verified.

### Week 4 — Reply Monitoring + Escalation Loop + Handoff

- [ ] Build Workflow 3: poll for replies, LLM classify
- [ ] Test classifier with 5-6 real reply patterns (anonymized)
- [ ] Implement warning_2 and warning_3 logic (date threshold checks)
- [ ] Implement human-needed flag + internal Zendesk note
- [ ] Implement soft-offboard flagging
- [ ] Flip dry_run = false for one real fund (closed/safe fund first)
- [ ] Confirm no unintended side effects
- [ ] Operator sign-off

Deliverable: full end-to-end workflow in supervised production.

---

## Test Strategy

### Principle

```
Reads   → always real (Zendesk, Kion) — safe
Writes  → always isolated during testing (test Sheet, Zendesk test group)
```

### Layer 1: Node-level smoke tests (Week 1)

Test each integration in isolation before connecting anything.

- Zendesk: read a real Fund Audit ticket, confirm body returned
- Zendesk: parse HTML table in Code node, confirm fund rows extracted
- Zendesk: search by fund_id, confirm query syntax works
- Kion: `GET /funding-source/{id}` for 3 real fund IDs — confirm fields present
- Kion: `GET /funding-source/{id}/financials` — compare balances to ticket values
- Kion: `GET /funding-source/{id}/permission-mapping` — confirm contact fields
- Google Sheets: append a test row, update it, read it back

### Layer 2: Workflow integration tests (Week 2)

Use real input, controlled output targets.

**Test fund fixtures**: pick 3-5 funds already in `closing` or `closed` state.
- Real data, no active PIs, safe to use repeatedly
- Write these fund IDs to `.claude/TEST_FUNDS.md`

**Run**: real ticket → parse → enrich test fund IDs only → test Sheet → test Zendesk group

**Check**:
- Sheet has correct columns populated
- Zendesk ticket body is correct
- Running twice produces no duplicates (idempotency)
- Funds already in Sheet are updated, not duplicated

### Layer 3: Escalation timing tests (Week 3)

Testing wait states without waiting.

- Set `warning_1_date` to 8 days ago in test Sheet row manually
- Trigger escalation workflow
- Confirm: threshold detected → warning_2 created → Sheet updated

Repeat for warning_3 and human-flag threshold.

**LLM reply classification test**: prepare 5-6 reply scenarios:
- "Thanks, shutting everything down this week" → should classify: acknowledged
- "We need an extension until June" → should classify: needs human review
- Auto-reply / out-of-office → should classify: no response
- Dispute / anger → should classify: human needed
- Blank reply → should classify: no response

### Layer 4: Dry-run pre-production (Week 4)

Add a `dry_run` switch to every workflow:

```
IF dry_run == true
  → log intended action to DryRunLog Sheet tab
  → skip all Zendesk create/update calls
  → skip all real Sheet writes
ELSE
  → proceed normally
```

Run against two consecutive real Fund Audit tickets with `dry_run = true`.
Review DryRunLog after each. Only flip `dry_run = false` after two clean runs.

### What is never tested with live data

| Action | Reason |
|---|---|
| Zendesk ticket to real PI | They receive an email immediately |
| Write to real tracking Sheet during dev | Operators read it daily |
| Soft-offboard on an active fund | Cascades: deactivates real users |
| Any Kion write (enforcement, budget) | Affects real cloud accounts |

### Test artifacts to create before Week 1

- [ ] `TEST_FUNDS.md` — list of 3-5 closed fund IDs safe for repeated testing
- [ ] Test Google Sheet — copy of real tracker, separate Sheet ID
- [ ] Zendesk test group — `Fund Sentinel TEST`, no external notifications
- [ ] N8N `dry_run` environment variable — `true` by default, flip to `false` for production

---

## Security

- Zendesk API token: stored in N8N credentials vault (never in Sheet or workflow config)
- Kion API token: same
- Google service account JSON: same
- Anthropic API key: same
- All N8N workflows: access restricted to CloudBank operations team
- No personal credentials used — service accounts only
- PII (PI emails, names): present in Sheet and Zendesk tickets — treat as internal sensitive data
- Audit trail: N8N execution history provides full log of every workflow run and node output

---

## Open Questions

1. **Identifier mapping** — do fund IDs in the Zendesk ticket (`1942014`, `NAIRR240173`) map directly to Kion `funding-source` IDs? Validate in Week 1 before building anything else.
2. **Contact resolution** — is PI email from the ticket sufficient, or do we need Kion permission-mapping for co-PI / managers?
3. **Zendesk ticket format stability** — is the HTML table format consistent across all Fund Audit tickets? Pull 3 historical tickets to confirm.
4. **Escalation delays** — what are the agreed intervals between warning_1, warning_2, warning_3? (e.g., 3 days, 7 days, 14 days)
5. **Reply detection** — does Zendesk reliably thread replies onto the follow-up ticket, or do PIs sometimes open new tickets?
6. **Soft-offboard trigger** — what exact conditions (balance, %, days since warning) should trigger the offboard flag?
7. **Kion financials vs. ticket data** — do Kion balances match ticket balances, or is there a lag? If they match, BigQuery is not needed for MVP.

---

## Key Files

```
.claude/
  PLAN.md               ← this file
  TEST_FUNDS.md         ← closed fund IDs safe for testing (create in Week 1)

drupal/
  web/modules/custom/
    cloudbank_acct/     ← fund workflow logic, KionUtils.php, Zendesk ticket creation
    cloudbank_rest/     ← REST layer (no endpoints currently)
    cloudbank_common/   ← shared HTTP utilities

swagger.json            ← Kion API full spec (~/Downloads/)
```

---

## N8N Deployment (Azure)

- Host: Azure Container Instance or Azure App Service (cheapest options)
- N8N docker image: `n8nio/n8n`
- Credentials stored in N8N's encrypted credential store
- Environment variables: `dry_run`, `test_sheet_id`, `prod_sheet_id`, `zendesk_test_group_id`
- Execution history retention: 30 days minimum for audit purposes
