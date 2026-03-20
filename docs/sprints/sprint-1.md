# Sprint 1 — Connectivity + Ingestion

> **Duration:** Week 1 (~5 hours)
> **Goal:** Everything is connected and we can parse a real Fund Audit ticket end-to-end.
> **Rule:** Reads always use real APIs. Writes go to test Sheet + Zendesk test group only.

---

## Definition of Done (Sprint 1)

Parse a real Fund Audit ticket → extract all fund rows → enrich 3 funds from Kion → confirm data is correct.

---

## Pre-Sprint Checklist (do these before starting any N8N work)

- [ ] Create test Google Sheet (copy of real tracker, separate Sheet ID) → save ID as `test_sheet_id`
- [ ] Create Zendesk test group `Fund Sentinel TEST` with external email notifications disabled
- [ ] Set N8N environment variable `dry_run = true` by default
- [ ] Identify 3–5 closed/finished fund IDs safe for repeated testing → add to `.claude/TEST_FUNDS.md`
  - Good candidates: funds with `cloud_state = Finished` or `Off the Cloud` in the real tracker
  - Both CBF and NAIRR if possible

---

## Stories

### 1.1 — Set up N8N on Azure

**Task:** Deploy N8N docker container on Azure.

**Options (cheapest first):**
- Azure Container Instances (ACI) — pay per second, good for low-frequency cron
- Azure App Service (B1 tier) — ~$13/month, always on

**DoD:**
- [ ] N8N accessible at a stable URL
- [ ] Execution history retention set to 30 days
- [ ] Access restricted to CloudBank ops team

---

### 1.2 — Configure credentials in N8N vault

**Task:** Add all service credentials. Never store in workflow config or Sheet.

- [ ] Zendesk API token (Basic auth: `email/token:API_TOKEN`)
- [ ] Kion API token (Bearer)
- [ ] Google service account JSON
- [ ] Anthropic API key
- [ ] Slack incoming webhook

**DoD:** Each credential tested with a simple HTTP node — returns 200.

---

### 1.3 — Build: Parse Fund Audit Ticket (Workflow 1, Part 1)

**Task:** Read today's Fund Audit ticket from Zendesk and extract all fund rows from the HTML table.

**Steps:**
1. Zendesk search: find Fund Audit ticket by tag or subject pattern (confirm the exact tag/view in Week 1)
2. Code node: parse HTML table → extract 3 sections (CBF OS, NAIRR OS, NAIRR ≤10%)
3. For each row: extract `fund_id`, `funder`, `pi_email`, `cash_award`, `cash_balance`, `cash_balance_pct`, `credit_award`, `credit_balance`, `credit_balance_pct`

**DoD:**
- [ ] N8N reads a real Fund Audit ticket successfully
- [ ] Code node extracts all fund rows from all 3 tables
- [ ] Output is a clean JSON array with the fields above
- [ ] Tested on 2 different audit tickets (different days) — parsing is stable

---

### 1.4 — Validate: fund_id → Kion funding-source mapping

**Task:** Confirm that the numeric IDs in the Zendesk ticket (e.g. `1942014`, `NAIRR240173`) map directly to Kion `funding-source` IDs.

**Steps:**
1. Take 3 fund IDs from a real audit ticket
2. Call `GET /v3/funding-source/{id}` for each
3. Compare name / PI / dates to what you see in the CB Portal

**DoD:**
- [ ] 3 CBF fund IDs verified: Kion returns correct fund name and dates
- [ ] 1–2 NAIRR fund IDs verified (test with `NAIRR240173` style — try both with and without prefix)
- [ ] Document result in `.claude/TEST_FUNDS.md`: "fund_id X maps to Kion ID Y" (or "same")

> **If mapping fails:** This is the most critical assumption in the whole project. Stop and investigate before building anything else.

---

### 1.5 — Validate: Kion financials vs audit ticket balances

**Task:** Check if Kion `/financials` endpoint returns the same balances as the audit ticket.

**Steps:**
1. For 3 test funds: call `GET /v3/funding-source/{id}/financials`
2. Compare `amount_unspent` (Kion) vs `cash_balance` + `credit_balance` (ticket)

**DoD:**
- [ ] Balances match (within acceptable rounding) → **Kion financials confirmed usable**
- [ ] If they don't match: document the lag/discrepancy → decide if Kion or ticket is the right source

> Expected result: they match (both sourced from cloud billing). If they diverge, the audit ticket is more current.

---

### 1.6 — Validate: Zendesk search by fund_id

**Task:** Confirm that searching Zendesk for a fund_id returns related tickets.

**Steps:**
1. Pick a fund_id that has ticket history (e.g. `2312875`)
2. Test Zendesk search query: `subject:2312875 OR description:2312875`
3. Confirm it returns tickets (not just the Fund Audit ticket)

**DoD:**
- [ ] Search returns at least one historical ticket for a known fund
- [ ] Query syntax confirmed and documented

---

### 1.7 — Validate: Google Sheets read + write

**Task:** Confirm N8N can read from and write to the test Sheet.

**Steps:**
1. Read column A of the test Sheet
2. Append a test row
3. Update a specific cell in that row
4. Read it back

**DoD:**
- [ ] Append works: new row appears in Sheet
- [ ] Cell update works: specific column updated, others untouched
- [ ] Read works: can check if a fund_id exists in column A

---

## Sprint 1 Output

By end of week, you should be able to:
1. Run Workflow 1 manually
2. It reads a real Fund Audit ticket
3. Extracts all 3 fund tables
4. For 3 test fund IDs: calls Kion, gets enrichment data
5. Prints clean JSON with all fields needed for the Sheet

Nothing writes to the real Sheet yet. Everything is logged/printed in N8N execution output.
