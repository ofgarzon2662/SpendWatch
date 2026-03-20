# Context Handoff — Fund Sentinel

> Last updated: 2026-03-19
> Purpose: Full context for next Claude session. Read this + PLAN.md + CLAUDE.md before doing anything.

---

## What this project is

N8N automation replacing Fernando's manual fund overspend follow-up workflow at CloudBank.
Daily 7am Zendesk audit ticket → parse → enrich → Google Sheet → warning tickets → escalation → soft-offboard flag.

**Stack:** N8N (Azure) · Google Sheets · Zendesk API · Kion API · Claude API · Slack

---

## What was done in this session

### Codebase ingested
- `drupal/` — full CloudBank Drupal portal (fund lifecycle, Kion sync, Zendesk integration)
- `~/Downloads/swagger.json` — full Kion API spec (56k lines)
- `CB OverSpend Fernando's.xlsx` — Fernando's manual tracking spreadsheet

### Key facts confirmed from codebase
- **CB Portal has NO REST API** — soft-offboard (fund → Closing) is always a manual human click by Fernando
- Closing cascades automatically: billing accounts → closing, IAM users → deactivated
- Kion API: `GET /v3/funding-source/{id}/financials` gives totals only (no daily data)
- Kion API: `POST /v3/spend-report/funding-source` gives monthly spend (YYYYMM format) — used for $37K/year threshold check
- Fund IDs in Zendesk ticket likely map directly to Kion `funding-source` IDs — **must validate in Sprint 1**

### Docs created
All under `SpendWatch/SpendWatch/docs/`:
- `manual-process.md` — full Mermaid flow diagrams: CBF + NAIRR ≤10% + NAIRR Overspend
- `architecture.md` — N8N system context + 3 workflows + state machines + deployment
- `data-model.md` — all 38 Google Sheet columns, ownership rules, status values, tier logic
- `sprints/sprint-1.md` — Connectivity + Ingestion (7 stories)
- `sprints/sprint-2.md` — Enrichment + Sheet Sync (8 stories)
- `sprints/sprint-3.md` — Escalation + LLM Tickets (7 stories)
- `sprints/sprint-4.md` — Reply Monitor + Full Loop + Handoff (8 stories)

### .gitignore updated
Added `/.claude/TEST_FUNDS.md` (preemptively — will contain real fund IDs, must not be committed).

### Security review done
No BLOCK issues. No hardcoded credentials anywhere. Real fund IDs and internal hostnames in docs are acceptable for a private repo.

---

## The complete manual process (what N8N automates)

### CBF (Non-NAIRR CloudBank Funds)
- Reported in audit ticket only when overspending (balance < $0)
- **New fund:** check CB Portal for PI name + billing accounts → add row → check Zendesk history → send W1
- **Existing fund:** update balance + date in sheet (only if changed) → compute daily_rate → check tier

**Escalation tiers by daily_rate:**
| Tier | daily_rate | W1→W2 | W2→W3+Close |
|---|---|---|---|
| 1 | < $100/day | 7 days | 7 days |
| 2 | $100–$499/day | 3 days | 2 days |
| 3 | ≥ $500/day | — | 1 day → Close |

Tier escalation rule: if today's tier > stored tier → send W1 of new tier immediately, reset clock.

**W1 always includes ACCESS migration offer** (per Shava, 2026-03-19).
Migration spend check via Kion (last full month):
- < $3,083/month (< $37K/year) → standard ACCESS offer
- ≥ $3,083/month → same offer in ticket + sheet note + **Slack Fernando** with monthly/annual figures

**Closing trigger:** W3 sent (T1/T2) or 24h elapsed after W1 (T3) → SET TO CLOSING + Slack Fernando → Fernando clicks "Closing" in portal.

**Tier 3 extra step:** Fernando goes into cloud and STOPS (not deletes) high-spend VMs. Then notifies PI to identify a user for data retrieval window.

**Reply classification:**
- `ready_to_offboard` — "stopped/migrating/done" → ask about data retrieval
- `supplement_in_progress` — monitor balance for jump
- `no_response` — follow escalation timer
- `human_needed` — anything with questions/judgment needed → Slack Fernando, pause automation

**Soft-offboard:** PI confirms data retrieved → sheet: offboard_ready=TRUE → Slack Fernando → Fernando clicks Closing.

### NAIRR ≤10% Monitoring
- Reported when cash OR credit balance ≤ 10% of award (not yet overspending)
- Wait 1 day after first appearance to get rate baseline
- Compute `daily_rate` from balance delta + `days_left = balance / daily_rate`
- Balance recorded in sheet cell only if changed: `$X.XX on MMM DD YYYY`
- **days_left ≥ 5:** monitor silently
- **days_left < 5:** send Warning 1 (strong, includes supplement options)
- **Supplement offer:** ONCE in W1. NEVER offered again.

**Supplement recommendation in W1:**
- Generic VMs/GPUs → Voltage Park or Expanse AIR
- Azure OpenAI / Foundry / cloud-specific AI → CloudBank Research resource
- Unclear → ask them what services they use in the ticket

**48h check after W1:**
- Dropped to < $50/day: no reply → "noticed decrease, thank you"; reply → LLM classify
- Still spending > $50/day + not in overspend → 24h ultimatum
- 24h later, no reply → Close + Slack Fernando → Fernando stops instances

**Supplement approved detection:** balance jumps >30% → auto-reply "Happy Clouding!" → mark resolved.

**Expiration flag:** fund_end < 2 weeks → write note in sheet ("Expiration in N days — colleague may be handling"). Nothing else.

**Special exception — NAIRR240242 ONLY:**
If cash goes negative AND credits are healthy → Slack Fernando only. No warnings, no offboard.
This is a ONE-FUND exception. Not a general rule.

### NAIRR Overspend (rare)
- Strong message: "Stop immediately or we stop instances" → 48h
- Reply + stopped → verify delta → ask about data retrieval
  - Yes → soft-offboard → Slack Fernando
  - No → "Setting to Closing, provide user ID for data retrieval window"
- No reply → Slack Fernando → Fernando stops instances
- Supplement NOT offered (already in ≤10% ticket)

---

## Business rules confirmed with Shava (Fernando's boss)

1. **Always offer ACCESS migration to CBF overspenders before off-boarding** — Fernando's old behavior (skip unless PI mentions it) is now incorrect.
2. **$37K/year threshold** (~$3,083/month) — funds above this are likely better fits for NAIRR. N8N flags via Slack, human decides.
3. **Approved W1 ACCESS migration text:**
   > "If you'd like to continue running on CloudBank, please migrate your account to ACCESS. You should have received an email on March 5 with instructions asking you to complete the migration by May 1. We recommend starting the migration as soon as possible since you are now overspent. Just as a reminder, instructions can be found here: https://www.cloudbank.org/training/access-cloudbank-research."
4. **CBF only** — NAIRR flows unchanged by Shava's instructions.

---

## What N8N does NOT do (MVP scope)

- Click "Closing" in CB Portal — no REST API exists
- Stop/delete cloud VMs — requires cloud console access (Fernando does manually)
- AMIE state management — out of scope
- BigQuery integration — deferred to Phase 2
- Azure billing spreadsheet parsing — replaced by asking PI what services they use
- Beam (Nutanix) integration — daily_rate derived from balance delta instead

---

## What needs to happen next

### Immediate (before Sprint 1 starts)
- [ ] Create `TEST_FUNDS.md` (3–5 closed fund IDs safe for testing) — gitignored, never committed
- [ ] Deploy N8N on Azure
- [ ] Set up test Google Sheet + Zendesk test group
- [ ] Get API keys: Zendesk, Kion, Anthropic, Slack webhook, Google service account

### Sprint 1 focus
- Parse Fund Audit ticket HTML → extract 3 tables
- **Validate fund_id → Kion funding-source mapping** (MOST CRITICAL ASSUMPTION)
- Validate Kion financials vs audit ticket balances
- Validate Zendesk search by fund_id

See `docs/sprints/sprint-1.md` for full checklist.

---

## Key file locations

```
SpendWatch/SpendWatch/
  CLAUDE.md                    ← project context for Claude
  .gitignore                   ← includes TEST_FUNDS.md preemptively
  .claude/
    PLAN.md                    ← full architecture + original plan
    CONTEXT_HANDOFF.md         ← this file
    TEST_FUNDS.md              ← (create in Sprint 1, gitignored)
  docs/
    manual-process.md          ← Mermaid flow diagrams of Fernando's manual process
    architecture.md            ← N8N system architecture
    data-model.md              ← Google Sheet schema (38 columns)
    sprints/
      sprint-1.md through sprint-4.md

SpendWatch/drupal/             ← CB Portal Drupal codebase (ingested, has its own repo)
~/Downloads/swagger.json       ← Kion full API spec
```

---

## People

- **Fernando** — CloudBank Operations Engineer, building Fund Sentinel, learning N8N
- **Shava** — Fernando's boss, sets business rules (e.g. migration offer policy, $37K threshold)
- Colleague (unnamed) — handles "will you extend?" tickets when fund expiration < 2 weeks; sometimes handles off-boarding
