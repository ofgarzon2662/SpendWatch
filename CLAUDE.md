# CloudBank Fund Sentinel — Claude Context

This project automates the CloudBank fund overspend follow-up workflow using N8N.

**Full plan**: `.claude/PLAN.md`

---

## What this project does

Starts from daily Fund Audit Zendesk tickets (auto-generated at 07:00) and automates:
1. Parsing fund IDs and PI emails from the ticket
2. Enriching each fund from Kion API
3. Syncing to the Google Sheets operational tracker
4. Creating follow-up Zendesk tickets (LLM-written)
5. Multi-stage escalation with delay-based state management
6. Classifying inbound replies (LLM)
7. Flagging funds ready for soft-offboard to human operators

## Tool stack

- **N8N** — workflow automation (engineer is learning it on the go)
- **Google Sheets** — state store / operational tracker
- **Zendesk API** — read Fund Audit tickets, create follow-up tickets
- **Kion API** — fund enrichment (`https://kion.cloudbank.org/api/v3/`)
- **Claude/LLM** — ticket parsing, reply classification, message writing
- **Azure** — deployment target (N8N docker container)

## Key technical facts

### Zendesk Fund Audit ticket
- Structured HTML table — parseable with regex, no LLM needed for parsing
- Already contains: `fund_id`, `pi_email`, balances, overspend %, funder type (NAIRR vs non-NAIRR)
- PI email is in the ticket — contact resolution is not a blocker

### Kion API
- "Fund" in the domain = `funding-source` in Kion
- Full swagger spec at `~/Downloads/swagger.json`
- API token already configured
- Key endpoints:
  - `GET /v3/funding-source/{id}` — name, dates, ou_id
  - `GET /v3/funding-source/{id}/financials` — balances
  - `GET /v3/funding-source/{id}/permission-mapping` — contacts

### CloudBank portal (Drupal)
- **No REST API** — confirmed by codebase analysis
- Soft-offboard = human clicks fund → "Closing" in the portal
- That single action cascades: billing accounts → closing, IAM users → deactivated
- Key module: `drupal/web/modules/custom/cloudbank_acct/`

### Critical unvalidated assumption
Fund IDs in Zendesk tickets (e.g., `1942014`, `NAIRR240173`) must map directly to Kion `funding-source` IDs. Validate this first in Week 1 before building anything else.

## Codebase structure

```
.claude/
  PLAN.md             ← full project plan, architecture, test strategy, weekly checklist
  TEST_FUNDS.md       ← (create in Week 1) closed fund IDs safe for repeated testing

drupal/
  web/modules/custom/
    cloudbank_acct/   ← core fund workflow logic, KionUtils.php, Zendesk integration
    cloudbank_rest/   ← REST layer (no endpoints currently)
    cloudbank_common/ ← shared HTTP utilities
```

## Test strategy (summary)

- Reads: always use real APIs (Zendesk, Kion) — safe
- Writes: always isolated during dev (test Google Sheet copy, Zendesk test group with no external notifications)
- Test fixtures: 3-5 already-closed fund IDs (real data, no active PIs)
- Never create a Zendesk ticket to a real PI during testing
- Dry-run switch in every N8N workflow before going to production

## Soft-offboard scope (MVP)

System flags funds ready for offboard. Human does the portal click.
System does NOT automate cloud console actions (AWS/GCP/Azure resource deletion).

## Preferences

- Practical over elegant
- 4-week timeline, ~5h/week
- Keep suggestions scoped — do not over-engineer
- N8N is the tool; avoid suggesting a custom backend unless there is a strong reason
