# CloudBank Fund Sentinel — Data Model

> Google Sheets is the operational source of truth for MVP.
> This document defines every column, who owns it, and what values it can hold.

---

## Ownership Rules

- **N8N-owned columns:** Written by the system. Fernando should not edit these (or edits may be overwritten).
- **Human-owned columns:** Fernando and colleagues edit these. N8N never writes to them.
- **Shared columns:** N8N writes structured values; Fernando may add to adjacent cells or use them for filtering.

---

## Column Schema

| # | Column | Owner | Source | Description |
|---|---|---|---|---|
| A | `fund_id` | N8N | Zendesk ticket | Fund identifier. Hyperlinked to CB Portal page. CBF: numeric (e.g. `1942014`). NAIRR: e.g. `NAIRR240173` or `240280`. |
| B | `funder` | N8N | Zendesk ticket | `NSF` or `NAIRR` |
| C | `pi_email` | N8N | Zendesk ticket | PI email from audit ticket |
| D | `pi_name` | N8N | CB Portal (CBF) / Kion (NAIRR) | PI full name |
| E | `fund_name` | N8N | Kion | Funding source name from Kion (NAIRR only; CBF uses fund_id) |
| F | `fund_start` | N8N | Kion | Fund start date (ISO date) |
| G | `fund_end` | N8N | Kion | Fund end date (ISO date). Used for expiration warning. |
| H | `cloud_state` | Human | Manual | `Fully Active` / `Off the Cloud` / `Overspending` / `Were Provided Funds` / `Finished` |
| I | `offboarding_state` | Human | Manual | `Wait` / `In Progress` / `Finished` |
| J | `action` | Human | Manual | `Wait` / `Full OB` / `Leave Alone` / `Human Review` |
| K | `cloud_providers` | N8N | Zendesk ticket / Kion | Comma-separated: `AWS`, `GCP`, `Azure`, `IBM` |
| L | `billing_accounts` | N8N | CB Portal / Kion | Comma-separated account IDs |
| M | `cash_award` | N8N | Zendesk ticket | Total cash award ($) |
| N | `cash_spend` | N8N | Zendesk ticket | Total cash spent ($) |
| O | `cash_balance` | N8N | Zendesk ticket | Current cash balance ($, negative = overspend) |
| P | `cash_balance_pct` | N8N | Zendesk ticket | Cash balance as % of award |
| Q | `credit_award` | N8N | Zendesk ticket | Total credit award ($) |
| R | `credit_spend` | N8N | Zendesk ticket | Total credit spent ($) |
| S | `credit_balance` | N8N | Zendesk ticket | Current credit balance ($, negative = overspend) |
| T | `credit_balance_pct` | N8N | Zendesk ticket | Credit balance as % of award |
| U | `daily_rate` | N8N | Computed | Estimated daily spend ($). Derived from balance delta / days since last record. |
| V | `days_left` | N8N | Computed | `current_balance / daily_rate`. NAIRR only. |
| W | `tier` | N8N | Computed | CBF only. `1` / `2` / `3`. Based on daily_rate. |
| X | `status` | N8N | Computed | See Status Values below. |
| Y | `source_ticket_id` | N8N | Zendesk | Fund Audit ticket ID that first detected this fund |
| Z | `followup_ticket_id` | N8N | Zendesk | Zendesk ticket ID created by Fund Sentinel |
| AA | `ticket_history_note` | N8N | Zendesk search | Note written at first detection if existing tickets found: `Ticket history: #XXXX, #YYYY — [one-line summary]` |
| AB | `warning_1_date` | N8N | Computed | ISO date W1 was sent |
| AC | `warning_2_date` | N8N | Computed | ISO date W2 was sent |
| AD | `warning_3_date` | N8N | Computed | ISO date W3 was sent |
| AE | `last_reply_date` | N8N | Zendesk | ISO date of last PI reply on followup ticket |
| AF | `reply_classification` | N8N | Claude | `ready_to_offboard` / `supplement_in_progress` / `no_response` / `human_needed` |
| AG | `balance_history` | N8N | Zendesk ticket | Running log of balance snapshots. Format: `$X.XX on MMM DD YYYY`. Only updated when balance changes. |
| AH | `offboard_ready` | N8N | Computed | `TRUE` when fund confirmed data-retrieved and ready for portal Closing click |
| AI | `expiration_flag` | N8N | Kion | Written when fund_end < 2 weeks away: `Expiration in N days — colleague may be handling` |
| AJ | `last_enriched` | N8N | System | ISO datetime of last N8N write to this row |
| AK | `sentinel_version` | N8N | System | Workflow version that last touched this row |
| AL | `operator_notes` | Human | Manual | Free-text. Fernando and team notes. **N8N never writes here.** |

> Columns to the right of `operator_notes` are human-owned and N8N never writes to them.

---

## Status Values

| Status | Set by | Meaning |
|---|---|---|
| `new_cbf_detected` | Workflow 1 | CBF fund appeared in ticket for first time |
| `new_nairr_detected` | Workflow 1 | NAIRR fund appeared in ticket for first time |
| `monitoring` | Workflow 1 | NAIRR fund under daily watch, days_left >= 5 |
| `w1_sent` | Workflow 2 | Warning 1 sent |
| `w2_sent` | Workflow 2 | Warning 2 sent |
| `w3_sent` | Workflow 2 | Warning 3 sent |
| `closing_flagged` | Workflow 2 | Final warning sent + Slack to Fernando; awaiting portal click |
| `human_needed` | Workflow 3 | Complex reply received; automation paused |
| `supplement_pending` | Workflow 3 | PI mentioned supplement request in progress |
| `offboard_ready` | Workflow 3 | PI confirmed data retrieved; Fernando should click Closing |
| `resolved` | Human | Fernando marks resolved after portal action |

---

## Tier Logic (CBF only)

| Tier | daily_rate | W1 → W2 | W2 → W3 + Close |
|---|---|---|---|
| 1 | < $100/day | 7 days | 7 days |
| 2 | $100–$499/day | 3 days | 2 days |
| 3 | ≥ $500/day | — | 1 day → Close |

**Tier escalation rule:** If today's computed tier > stored tier → immediately send W1 of the new tier and reset the clock. No waiting.

---

## NAIRR days_left Thresholds

| Condition | Action |
|---|---|
| days_left >= 5 | Monitor silently |
| days_left < 5 | Send Warning 1 (strong, with supplement options) |
| Balance jumps significantly (>30% increase) | Supplement approved — auto reply "Happy Clouding" |
| Spending drops to < $50/day | Acknowledge decrease; offer supplement reminder |

---

## Special Fund Exceptions

| Fund ID | Rule |
|---|---|
| `NAIRR240242` | Cash at ~$0, credits healthy. If cash goes negative: Slack Fernando only. No offboard trigger. This is a fund-specific exception — not a general rule. |

---

## Sheet Tabs

| Tab | Purpose |
|---|---|
| `Tracker` | Main operational tracker (schema above) |
| `DryRunLog` | All intended writes/actions when `dry_run = true` |

---

## Google Sheets API Notes

- N8N uses Google Sheets node with service account credentials
- Fund lookup: search column A for `fund_id` (exact match)
- Balance history cell: append to existing cell value (not overwrite) when balance changes
- N8N only appends a balance snapshot if `today_balance != last_recorded_balance`
- Operator columns (`operator_notes` and beyond): N8N uses `getRange` that stops before column AL
