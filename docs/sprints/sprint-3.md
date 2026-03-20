# Sprint 3 — Escalation + LLM Tickets

> **Duration:** Week 3 (~5 hours)
> **Goal:** System generates well-written warning tickets for all fund types. Dry-run verified.
> **Rule:** All Zendesk ticket creates go to test group (no real PI contact). dry_run = true throughout.

---

## Definition of Done (Sprint 3)

Two clean dry runs on real audit tickets. DryRunLog shows correct intended actions. 5 generated tickets reviewed manually and approved.

---

## Stories

### 3.1 — Build Workflow 2: Escalation State Reader

**Task:** Read Sheet rows that need action and determine what to do next.

**Logic per row:**
```
IF status == new_cbf_detected:
    → action = send W1

IF status == w1_sent:
    days_since_w1 = today - warning_1_date
    IF tier == 1 AND days_since_w1 >= 7  → action = send W2
    IF tier == 2 AND days_since_w1 >= 3  → action = send W2
    IF tier == 3 AND days_since_w1 >= 1  → action = send W3 + close

IF status == w2_sent:
    days_since_w2 = today - warning_2_date
    IF tier == 1 AND days_since_w2 >= 7  → action = send W3 + close
    IF tier == 2 AND days_since_w2 >= 2  → action = send W3 + close

IF tier escalated since last warning:
    → action = send W1 of new tier immediately (override timer)

IF status == new_nairr_detected AND days_left < 5:
    → action = send NAIRR W1 (10pct warning)

IF status == monitoring AND days_left < 5:
    → action = send NAIRR W1

IF status == w1_sent (NAIRR) AND days_since_w1 >= 2:
    → action = check_spend_delta (triggers reply monitor logic)
```

**DoD:**
- [ ] Correctly identifies which rows need action
- [ ] Does not re-send warnings already sent (idempotent)
- [ ] Tier escalation correctly overrides timer

---

### 3.2 — LLM Node: Write ticket bodies

**Task:** For each fund needing a warning, generate the ticket body using Claude.

**Inputs to LLM per ticket:**
- fund_id, PI name, funder, balance amounts, balance %, daily_rate
- days_left (NAIRR only), cloud_providers
- warning number, tier
- template_id (determines tone and content)

**Template IDs:**

| Template ID | When used | Key elements |
|---|---|---|
| `cbf_w1` | CBF first warning (T1/T2) | Overspending notice, ACCESS migration offer |
| `cbf_w1_tier3` | CBF first warning (T3) | Urgent: heavy spending, 24h or we stop instances |
| `cbf_w2` | CBF second warning | Confirm receipt, measures, offboarding risk |
| `cbf_w3` | CBF third warning + closing | Final warning, fund being set to Closing |
| `nairr_w1_10pct` | NAIRR ≤10% warning | Resource %, supplement instructions, ask about service types |
| `nairr_24h` | NAIRR spending continues after 48h | 24h ultimatum: stop or we do |
| `nairr_overspend_strong` | NAIRR crossed into overspend | Stop immediately, 48h to respond |
| `nairr_thank_you` | NAIRR spending decreased < $50/day | Acknowledge decrease, supplement still available |
| `supplement_approved` | Balance jumped significantly | Supplement approved, Happy Clouding |

**LLM prompt structure:**
```
You are CloudBank support staff writing a professional, concise follow-up email.
Fund ID: {fund_id}
PI: {pi_name}
Funder: {funder}
Current balance: {balance} ({balance_pct}%)
Daily spend rate: ${daily_rate}/day
{days_left_line if NAIRR}
Cloud providers: {cloud_providers}
Warning level: {warning_number}

Using the template below as a guide, write the email body.
Keep it professional but direct. Do not add new commitments or promises not in the template.
Template: {template_text}
```

**DoD:**
- [ ] LLM generates ticket body for each template type
- [ ] Output is well-written, not robotic, matches Fernando's tone
- [ ] Manually review 5 generated tickets — adjust prompt if needed
- [ ] LLM does NOT make up facts or add promises not in the template

---

### 3.3 — Zendesk ticket creation (test group)

**Task:** Create Zendesk tickets in the `Fund Sentinel TEST` group (no external email).

**For each warning to send:**
1. Create Zendesk ticket:
   - Subject: `[ATTENTION]: {warning_subject} {fund_id}`
   - Body: LLM output
   - Group: `Fund Sentinel TEST` (test group ID from env var)
   - Tags: `fund_sentinel`, `fund_{fund_id}`
   - Requester: PI email (real email — but TEST group suppresses delivery)
2. Update Sheet: `followup_ticket_id`, `warning_N_date`, `status`

**DoD:**
- [ ] Ticket appears in Zendesk under test group
- [ ] No email sent to real PI (verify with test group settings)
- [ ] Sheet row updated with ticket ID and date

---

### 3.4 — Closing trigger + Slack notification

**Task:** When a fund reaches the closing trigger (W3 sent, or Tier 3 + 24h), fire Slack alert.

**Logic:**
1. Update Sheet: `status = closing_flagged`, `offboard_ready = TRUE` if data confirmed
2. Send Slack message to ops channel:
   ```
   Fund {fund_id} has been set to CLOSING.
   PI: {pi_email}
   Daily rate: ${daily_rate}/day
   Zendesk ticket: {ticket_link}
   Action needed: Click "Closing" in CB Portal → {portal_link}
   ```

**DoD:**
- [ ] Slack message appears in ops channel on close trigger
- [ ] Message has correct fund details and portal link
- [ ] Sheet status updated to `closing_flagged`

---

### 3.5 — NAIRR240242 Slack alert

**Task:** When NAIRR240242's cash goes negative, send Slack to Fernando.

**Message:**
```
Fund NAIRR240242: cash balance went negative (${amount}).
Credits are healthy — no action taken.
FYI only.
```

**DoD:**
- [ ] Alert fires only for NAIRR240242
- [ ] Does not trigger offboard or warning flows

---

### 3.6 — dry_run switch

**Task:** Add dry_run gate to Workflow 2 (and verify Workflow 1 already respects it).

**Logic:**
```
IF dry_run == true:
    → Write intended action to DryRunLog tab:
      timestamp | fund_id | action | ticket_subject | ticket_body_preview
    → Skip all Zendesk create/update calls
    → Skip Slack messages
ELSE:
    → Proceed normally
```

**DoD:**
- [ ] dry_run = true → DryRunLog tab fills up, no Zendesk tickets created, no Slack sent
- [ ] dry_run = false → real actions fire (test with one safe test fund first)
- [ ] DryRunLog row has enough detail to review intended action

---

### 3.7 — Dry run x2

**Task:** Run both Workflow 1 + Workflow 2 with dry_run = true on two consecutive real audit tickets.

**Check after each run:**
- [ ] DryRunLog shows correct funds with correct actions
- [ ] No actions for funds that don't need them
- [ ] Ticket bodies look correct (spot check 5)
- [ ] No Zendesk tickets created
- [ ] No Slack messages sent

**Only flip `dry_run = false` after two clean runs.**

---

## Sprint 3 Output

By end of week:
1. Workflow 2 reads Sheet, determines escalation action per fund
2. Claude writes the ticket bodies
3. In test mode: tickets created in Zendesk test group, no real PIs contacted
4. Closing triggers send Slack to Fernando
5. Two clean dry runs completed and reviewed
