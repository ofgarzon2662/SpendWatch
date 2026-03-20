# Sprint 2 — Enrichment + Sheet Sync

> **Duration:** Week 2 (~5 hours)
> **Goal:** Full ingest pipeline working end-to-end. New funds added, existing funds updated, no duplicates.
> **Rule:** All writes go to test Sheet. No Zendesk ticket creation yet (that's Sprint 3).

---

## Definition of Done (Sprint 2)

Run Workflow 1 twice on the same audit ticket → Sheet has correct rows → no duplicates → balance cells updated.

---

## Stories

### 2.1 — Fund dedup gate (Sheet lookup)

**Task:** Before writing anything, check if fund_id already exists in the Sheet (column A).

**Logic:**
- Read column A of test Sheet → build lookup map
- If fund_id found → "existing fund" path
- If not found → "new fund" path

**DoD:**
- [ ] New fund → row appended with all initial fields
- [ ] Existing fund → only balance + date fields updated (other fields untouched)
- [ ] Running twice: second run produces no new rows

---

### 2.2 — New CBF fund: Sheet row creation

**Task:** When a CBF fund appears for the first time, add a full row to the test Sheet.

**Fields to populate (from sources in sprint 1 validations):**

| Field | Source |
|---|---|
| fund_id | Zendesk ticket (as CB Portal hyperlink) |
| funder | Zendesk ticket |
| pi_email | Zendesk ticket |
| cash_award / cash_balance / cash_balance_pct | Zendesk ticket |
| credit_award / credit_balance / credit_balance_pct | Zendesk ticket |
| cloud_state | Default: `Fully Active` |
| action | Default: `Wait` |
| balance_history | `$X.XX on MMM DD YYYY` (today's balance) |
| status | `new_cbf_detected` |
| source_ticket_id | Zendesk audit ticket ID |
| last_enriched | Now (ISO datetime) |
| sentinel_version | Current workflow version |

> Note: PI name, billing accounts, fund dates not auto-fetched for CBF in MVP.
> The CB Portal has no REST API — these fields are left blank for Fernando to fill manually,
> or Fernando can add them via the existing CB Portal lookup.

**DoD:**
- [ ] New CBF row appears in test Sheet with all fields above
- [ ] fund_id cell is hyperlinked to CB Portal URL
- [ ] Running twice: same row, no duplicate

---

### 2.3 — New NAIRR fund: Sheet row creation + Kion enrichment

**Task:** When a NAIRR fund appears for the first time, enrich from Kion and add row.

**Kion calls:**
1. `GET /v3/funding-source/{id}` → fund_name, fund_start, fund_end, ou_id
2. `GET /v3/funding-source/{id}/permission-mapping` → PI name (if not in ticket)

**Additional fields vs CBF:**

| Field | Source |
|---|---|
| fund_name | Kion |
| fund_start | Kion |
| fund_end | Kion |
| kion_ou_id | Kion |
| days_left | Computed: credit_balance / daily_rate (day 1: unknown, set to null) |
| status | `new_nairr_detected` |

**DoD:**
- [ ] NAIRR row in test Sheet with Kion enrichment fields populated
- [ ] fund_end date present (used for expiration warning in Sprint 3)
- [ ] Kion call handles both numeric IDs (`240280`) and prefixed (`NAIRR240173`)

---

### 2.4 — Existing fund: balance update + daily_rate computation

**Task:** For a fund already in the Sheet, update balances and compute rate/tier.

**Logic:**
1. Read last recorded balance from `balance_history` cell (parse `$X.XX on MMM DD YYYY`)
2. Compare to today's balance from audit ticket
3. If changed: append `$X.XX on MMM DD YYYY` to balance_history cell
4. If no change: skip (do not write)
5. Compute `daily_rate = |balance_delta| / days_since_last_record`
6. CBF: determine tier (1/2/3), check if tier escalated
7. NAIRR: compute `days_left = current_balance / daily_rate`

**DoD:**
- [ ] balance_history cell shows running log (newest append, not overwrite)
- [ ] No write if balance unchanged
- [ ] daily_rate and tier columns updated for CBF
- [ ] days_left updated for NAIRR

---

### 2.5 — NAIRR240242 exception handler

**Task:** Special handling for fund NAIRR240242 only.

**Logic:**
- If this fund_id's cash_balance goes negative AND credit_balance > 0:
  → Write note to sheet: `Cash went negative ($X.XX) on [date] — credits healthy`
  → Do NOT set status to closing or trigger any warning
  → (Slack alert added in Sprint 3)

**DoD:**
- [ ] Exception triggers only for NAIRR240242, not for any other fund
- [ ] Normal NAIRR funds with cash going negative are treated normally

---

### 2.6 — Expiration date check (NAIRR)

**Task:** If a NAIRR fund's `fund_end` date is within 2 weeks of today, write a heads-up note.

**Logic:**
- Check `fund_end` field for all NAIRR rows in Sheet
- If `fund_end - today <= 14 days` AND `expiration_flag` cell is empty:
  → Write to `expiration_flag` cell: `Expiration in N days — colleague may be handling`

**DoD:**
- [ ] Flag written correctly for a test fund with near expiration date
- [ ] Not written again if already flagged (idempotent)

---

### 2.7 — Zendesk ticket history check (CBF new funds)

**Task:** Before creating Warning 1 for a new CBF fund, search Zendesk for existing tickets.

**Logic:**
1. Search Zendesk: `subject:{fund_id} OR description:{fund_id}` (exclude Fund Audit tickets)
2. Filter for non-closed/non-solved tickets
3. If found:
   → Send ticket bodies to Claude: generate one-line summary of recent activity
   → Write to `ticket_history_note` cell: `Ticket history: #XXXX, #YYYY — [summary]`
4. If not found: skip

**DoD:**
- [ ] Finds existing tickets for a test fund with known history
- [ ] LLM summary is one line, informative, not hallucinated
- [ ] No crash if no tickets found

---

### 2.8 — Idempotency test

**Task:** Run the full Workflow 1 twice on the same audit ticket. Verify no side effects.

**Check:**
- [ ] No duplicate rows in Sheet
- [ ] Balance cells show same value (not doubled)
- [ ] Status columns unchanged on second run
- [ ] No extra Zendesk calls triggered

---

## Sprint 2 Output

By end of week:
1. Workflow 1 runs end-to-end on a real audit ticket against the **test Sheet**
2. New funds get rows, existing funds get balance updates
3. NAIRR funds have Kion enrichment + days_left computed
4. Running twice = no duplicates
5. Sheet looks like Fernando's manual tracker but filled in automatically
