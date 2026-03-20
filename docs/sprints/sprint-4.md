# Sprint 4 — Reply Monitor + Full Loop + Handoff

> **Duration:** Week 4 (~5 hours)
> **Goal:** Full end-to-end system in supervised production. Fernando signs off.
> **Rule:** First live run on one real closed/safe fund. Only flip dry_run = false after confirming no side effects.

---

## Definition of Done (Sprint 4)

Full workflow runs on real data. One real PI ticket sent. Fernando confirms behavior is correct. Operator sign-off.

---

## Stories

### 4.1 — Build Workflow 3: Reply Monitor

**Task:** Poll Zendesk for replies on open followup tickets → classify → update Sheet → alert.

**Logic:**
```
1. Read Sheet rows where followup_ticket_id is set AND status not in [resolved, human_needed]
2. For each: GET /v2/tickets/{id}/comments
3. Filter for comments newer than last_reply_date
4. If new comment found:
   a. Send to Claude for classification
   b. Update Sheet: reply_classification, last_reply_date, next_action
   c. Route based on classification:
      - ready_to_offboard → post Zendesk comment asking about data retrieval
      - supplement_in_progress → update status, monitor balance
      - human_needed → Slack Fernando with ticket link + one-line summary
      - no_response → no action (escalation timer handles it in W2)
```

**DoD:**
- [ ] Classifies replies correctly for all 5 test scenarios (see test plan below)
- [ ] human_needed triggers Slack with correct info
- [ ] ready_to_offboard posts data-retrieval question as Zendesk comment
- [ ] Sheet updated correctly in all cases

---

### 4.2 — Reply classifier test scenarios

**Task:** Prepare 6 test replies and verify LLM classification.

| Reply | Expected classification |
|---|---|
| "Thanks, shutting everything down this week" | `ready_to_offboard` |
| "Yes we stopped instances and are migrating to ACCESS" | `ready_to_offboard` |
| "We submitted a supplement request, awaiting approval" | `supplement_in_progress` |
| Auto-reply / out-of-office | `no_response` |
| "We need an extension, can you help us understand the process?" | `human_needed` |
| Gabriele-style: multiple questions, another fund, ACCESS details | `human_needed` |

**DoD:**
- [ ] All 6 scenarios classified correctly
- [ ] Adjust LLM prompt if any fail (document prompt version)

---

### 4.3 — Supplement approval detection

**Task:** When a NAIRR fund's balance jumps significantly (supplement approved), auto-reply.

**Logic (in Workflow 1):**
- Compare today's balance to yesterday's balance
- If balance increased by > 30% of award (or went from <5% to >20%): supplement detected
- Post Zendesk comment: "I noticed your supplement request was approved. Happy Clouding!"
- Update Sheet: `status = resolved`, `reply_classification = supplement_approved`

**DoD:**
- [ ] Triggers correctly on a test row with manually set low-then-high balance
- [ ] Does not false-trigger on normal day-to-day fluctuations
- [ ] Auto-reply posted to correct Zendesk ticket

---

### 4.4 — NAIRR Overspend Response flow

**Task:** Handle funds that appear in NAIRR Overspenders table (rare but must work).

**Logic:**
```
IF fund in NAIRR Overspenders AND status != already handling overspend:
    → Send strong message: "Stop immediately or we stop instances. 48h to respond."
    → Update Sheet: status = nairr_overspend_w1_sent, warning_1_date = today

48h later (Workflow 3 checks reply):
    IF reply confirms stopped → verify delta:
        IF confirmed → ask about data retrieval
        IF not confirmed → wait 24h, recheck
    IF reply about data → handle data retrieval user unlock flow
    IF no reply → Slack Fernando, Fernando stops instances

Note: supplement NOT offered in overspend flow (already offered in ≤10% ticket)
```

**DoD:**
- [ ] Strong message generated and sent correctly
- [ ] 48h check works (can test by setting warning_1_date to 2 days ago manually)
- [ ] Delta verification logic works
- [ ] Data retrieval flow: correct message sent, Sheet updated

---

### 4.5 — offboard_ready flag + operator workflow

**Task:** When a fund is confirmed ready for soft-offboard, make it obvious to Fernando.

**Logic:**
1. PI confirms data retrieved → update Sheet: `offboard_ready = TRUE`, `status = offboard_ready`
2. Send Slack:
   ```
   Fund {fund_id} is ready for soft-offboard.
   PI confirmed data retrieved.
   Action: Go to CB Portal and set fund to "Closing"
   Portal link: {portal_link}
   Zendesk ticket: {ticket_link}
   ```
3. Fernando clicks "Closing" in portal (manual)
4. Fernando updates Sheet: `status = resolved` (human action)

**DoD:**
- [ ] Slack fires with correct info and portal link
- [ ] Sheet shows `offboard_ready = TRUE` clearly
- [ ] N8N stops touching this fund after `offboard_ready` is set

---

### 4.6 — First live run (single safe fund)

**Task:** Flip `dry_run = false` for one real fund and confirm no unintended side effects.

**Candidate fund:** A fund that is already in `closed` or `finished` state — real data, no active PI.

**Pre-flight checklist:**
- [ ] dry_run = true runs show correct behavior for this fund
- [ ] Test group still configured in Zendesk (even in prod mode, use test group for first run)
- [ ] Fernando reviews DryRunLog and approves
- [ ] Backup of real Sheet taken before first live run

**Post-run check:**
- [ ] Only intended rows touched in Sheet
- [ ] No tickets sent to unexpected PIs
- [ ] No unexpected Slack messages
- [ ] Execution history in N8N shows clean run

---

### 4.7 — Transition to production

**Task:** Move from test group to real Zendesk group for one real overspending fund.

**Pre-flight:**
- [ ] Two clean test-group runs completed
- [ ] Fernando has reviewed 5+ generated ticket bodies
- [ ] All env vars set: `dry_run = false`, `zendesk_group_id = prod_group_id`

**DoD:**
- [ ] First real PI ticket sent correctly
- [ ] Fernando confirms ticket is acceptable
- [ ] No unintended side effects
- [ ] Fernando signs off on the system

---

### 4.8 — Operator handoff documentation

**Task:** Write a one-page runbook so Fernando (or a colleague) can operate the system.

**Contents of `docs/runbook.md`:**
- [ ] How to check what the system did today (N8N execution history)
- [ ] How to pause automation for a specific fund (`status = human_needed`)
- [ ] How to flip dry_run on/off
- [ ] How to handle a fund after Fernando clicks "Closing" in portal (`status = resolved`)
- [ ] What to do if a workflow fails (N8N error notification)
- [ ] Which columns Fernando owns vs N8N owns

---

## Sprint 4 Output

By end of week:
1. Full loop working: ingest → escalate → reply monitor → offboard flag
2. NAIRR overspend handled
3. First real PI ticket sent and confirmed correct
4. Fernando signed off
5. Runbook written
6. System running in supervised production
