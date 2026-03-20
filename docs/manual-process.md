# CloudBank Fund Sentinel — Manual Process Flow

> Captures Fernando's current manual workflow — the process Fund Sentinel automates.
> Diagrams use [Mermaid](https://mermaid.js.org/) and render in VS Code (install: *Markdown Preview Mermaid Support*) and GitHub.

---

## Daily Trigger

Every morning at **7:00 AM** a Fund Audit Zendesk ticket is auto-generated with three tables:

| Table | What it contains |
|---|---|
| Non-NAIRR Overspenders (CBF) | CBF funds with cash or credit balance < $0 |
| NAIRR Overspenders | NAIRR funds with cash or credit balance < $0 |
| NAIRR ≤10% Remaining | NAIRR funds with cash or credit ≤ 10% of award (not yet overspending) |

```mermaid
flowchart TD
    A([7:00 AM - Fund Audit Ticket]) --> B[Parse 3 tables from ticket]
    B --> C[CBF Overspenders]
    B --> D[NAIRR 10pct Remaining]
    B --> E[NAIRR Overspenders]
    C --> F[CBF Escalation Engine\nsee Diagram 1]
    D --> G[NAIRR 10pct Monitoring Loop\nsee Diagram 2]
    E --> H[NAIRR Overspend Response\nsee Diagram 3]
```

---

## Diagram 1 — CBF Escalation Engine

> Non-NAIRR CloudBank Funds. Reported only when overspending (cash or credit balance negative).

### Part A: New vs Existing Fund

```mermaid
flowchart TD
    A([CBF fund in audit ticket]) --> B{Fund in\nGoogle Sheet?}

    B -- No --> C[CB Portal lookup:\nPI name, billing accounts,\nusers per account]
    C --> D[Add row to sheet:\nfund_id as portal link\nPI name + email\nbilling accounts\nCB end date, award end date\ncloud_state = Active\naction = Wait\nbalance + today's date]
    D --> E[Search Zendesk for\nexisting tickets by fund_id]
    E --> F{Active tickets\nfound?}
    F -- Yes --> G[Write note in sheet cell:\nticket history IDs + LLM summary\nof recent activity]
    G --> H[Send Warning 1\nsee templates]
    F -- No --> H

    B -- Yes --> I[Update balance cell:\n$X.XX on MMM DD YYYY\nonly if changed since last record]
    I --> J[Compute daily_rate:\nbalance delta / days since last record]
    J --> K[Determine tier:\nTier 1 if daily_rate < $100\nTier 2 if $100-499\nTier 3 if >= $500]
    K --> L{Tier higher\nthan stored tier?}
    L -- Yes --> M[Send W1 of new tier immediately\nreset escalation clock]
    L -- No --> N[Check escalation timer\nsee Part B]
```

### Part B: Escalation Tiers & Timers

```mermaid
flowchart TD
    A([Check escalation state]) --> B{Current tier?}

    B -- Tier 1 < $100/day --> C{Last warning\nsent?}
    C -- None --> D[Send Warning 1]
    C -- W1 sent, < 7 days ago --> E[Wait]
    C -- W1 sent, >= 7 days ago --> F[Send Warning 2]
    C -- W2 sent, >= 7 days ago --> G[Send Warning 3\n+ SET TO CLOSING\n+ Slack Fernando]

    B -- Tier 2 $100-499/day --> H{Last warning\nsent?}
    H -- None --> I[Send Warning 1]
    H -- W1 sent, < 3 days ago --> E
    H -- W1 sent, >= 3 days ago --> J[Send Warning 2]
    H -- W2 sent, >= 2 days ago --> K[Send Warning 3\n+ SET TO CLOSING\n+ Slack Fernando]

    B -- Tier 3 >= $500/day --> L{Last warning\nsent?}
    L -- None --> M[Send Warning 1 URGENT:\nyou are heavily spending\nstop immediately\n24h or we stop instances]
    L -- W1 sent, >= 1 day ago --> N[SET TO CLOSING\n+ Slack Fernando\nFernando stops VMs manually\nthen: notify PI about\ndata retrieval user unlock]
```

### Part C: Reply Handling (CBF)

```mermaid
flowchart TD
    A([Reply received on CBF ticket]) --> B[LLM classifies reply]

    B -- Clean: stopping / migrating / done --> C[Ask: Have you retrieved\nall your data from the cloud?]
    C -- Yes --> D[Flag for soft-offboard\nNote in sheet\nSlack Fernando to click Closing]
    C -- No reply / unclear --> E[Wait and re-check\nwith next audit cycle]

    B -- Complex: questions, other funds,\nACCESS migration, unclear intent --> F[Slack Fernando:\nticket needs human attention\npause automation for this fund]

    B -- No reply --> G[Follow escalation timer\nsee Part B]
```

---

## Diagram 2 — NAIRR ≤10% Monitoring Loop

> NAIRR funds are credit-heavy, spend $1000s/day. Goal: avoid overspend entirely.
> Supplement option is offered once (in Warning 1). Never offered again.

### Part A: New vs Existing Fund

```mermaid
flowchart TD
    A([NAIRR fund in 10pct table]) --> B{Fund in\nGoogle Sheet?}

    B -- No --> C[Add row:\nfund_id link, PI email\nboth cash + credit balances\ncloud_state = Active]
    C --> D[Wait 1 day\nfor spend rate baseline]

    B -- Yes --> E{Special exception:\nNAIRR240242?}
    E -- Yes and cash just went negative --> F[Slack Fernando only:\ncash went negative but credits healthy\nno further action on this fund]
    E -- No --> G[Update balance cell\nonly if changed since last record\nFormat: $X.XX on MMM DD YYYY]
    G --> H[Compute daily_rate\nbalance delta / days since last]
    H --> I[Compute days_left:\ncurrent_balance / daily_rate]
    I --> J{Expiration date\n< 2 weeks?}
    J -- Yes --> K[Write heads-up in sheet:\nExpiration in N days\ncolleague may be handling]
    J -- No --> L{days_left < 5?}
    K --> L
    L -- No, days_left >= 5 --> M[Monitor silently\nre-evaluate tomorrow]
    L -- Yes --> N[Trigger Warning 1\nsee Part B]
```

### Part B: Warning & Escalation

```mermaid
flowchart TD
    A([days_left dropped below 5]) --> B[Send Warning 1:\nX% remaining on award\nbe mindful of spending\napply for supplement at submit-nairr.xras.org\nif using Azure OpenAI/Foundry: apply CloudBank Research\nif generic VMs/GPUs: apply Voltage Park or Expanse AIR\nif unsure: ask them what services they use\nstop instances or we will]

    B --> C{48h later:\ncheck spend delta}

    C -- Dropped to < $50/day --> D{Reply received?}
    D -- No reply --> E[Send: noticed your spending decreased\nthank you - supplement still available]
    D -- Reply received --> F[LLM classifies reply]
    F -- Clear intent, manageable --> G[Reply to ticket\nautomatically]
    F -- Questions / judgment needed --> H[Slack Fernando]

    C -- Still spending > $50/day --> I{In overspend\nalready?}
    I -- No --> J[Send 24h ultimatum:\nstop spending or we stop instances]
    I -- Yes --> K[Go to NAIRR\nOverspend Response\nDiagram 3]

    J --> L{24h later}
    L -- No reply --> M[SET TO CLOSING\nSlack Fernando\nFernando stops instances]
    L -- Reply: stopped instances --> N{Verify delta\nconfirms stop?}
    N -- Confirmed --> O[Send thank you message]
    N -- Not confirmed --> P[Wait 24h\nrecheck delta]
```

### Part C: Supplement & Balance Recovery

```mermaid
flowchart TD
    A([Daily balance check\nfor active NAIRR 10pct funds]) --> B{Balance jumped\nsignificantly?\nex: 2pct to 45pct}

    B -- Yes --> C[Supplement was approved]
    C --> D[Auto-reply on Zendesk ticket:\nI noticed your request was approved.\nHappy Clouding!]
    D --> E[Mark fund as resolved\nUpdate sheet status]

    B -- No --> F[Continue daily monitoring]
```

---

## Diagram 3 — NAIRR Overspend Response

> Rare — most NAIRR funds are caught at ≤10% before crossing into overspend.
> Supplement is NOT offered again here — it was already in Warning 1 of the ≤10% flow.

```mermaid
flowchart TD
    A([NAIRR fund in Overspenders table]) --> B[Send strong message:\nStop spending immediately\nor we stop your instances\n48h to respond]

    B --> C{48h later}

    C -- Reply: stopped instances --> D{Verify delta:\nspend actually dropped?}
    D -- Confirmed stopped --> E[Ask: Have you retrieved\nall your data from the cloud?]
    D -- Still spending --> F[Slack Fernando\nFernando stops instances manually]

    E -- Yes, data retrieved --> G[Flag for soft-offboard\nSlack Fernando to click Closing\nin CB Portal]
    E -- No, need to retrieve data --> H[Reply: We are setting fund to Closing\nto prevent further overspend.\nProvide a user ID - we will activate\nthat user for a data retrieval window.\nTell us when done.]
    H --> I[Fernando activates user\nUser retrieves data\nFernando finishes shutdown]

    C -- No reply after 48h --> F
```

---

## Special Cases

### NAIRR240242
- Has ~$0 cash remaining but healthy credits
- If cash goes negative: **Slack Fernando only** — do not trigger offboard or warnings
- This is a fund-specific exception, not a general rule

### Zendesk ticket history check (CBF)
Before sending Warning 1 on a newly detected CBF fund:
- Search Zendesk for any tickets referencing this fund_id
- If active/open tickets found: still send W1, but write a note in the sheet cell with ticket IDs and an LLM-generated one-line summary of recent activity
- Lets Fernando see context at a glance without digging into Zendesk

### Balance recording rule
- N8N only writes a new balance entry if the balance **changed** since the last recorded value
- Format in sheet cell: `$1,323.93 on Feb 4 2026`
- No change = no write (avoids noise in the sheet)

---

## Reply Classification Guide (for LLM node)

| Reply signals | Classification | N8N action |
|---|---|---|
| "stopped instances", "shutting down", "migrating to ACCESS", "all done" | `ready_to_offboard` | Ask about data retrieval |
| "supplement request in progress", "applied for supplement" | `supplement_in_progress` | Monitor balance for jump |
| Auto-reply, out-of-office, blank | `no_response` | Follow escalation timer |
| Questions about another fund, ACCESS details, billing, extensions, unclear intent | `human_needed` | Slack Fernando, pause automation |
| Dispute, frustration, anger, threats | `human_needed` | Slack Fernando, pause automation |

---

## CBF Migration Offer Rule (updated 2026-03-19, per Shava)

**CBF only — does not apply to NAIRR.**

Always offer the ACCESS migration option before off-boarding. Never skip it.
The migration recommendation is tailored by monthly spend (`daily_rate × 30`):

| monthly_spend | Recommendation in ticket |
|---|---|
| < $3,083/month (< $37K/year) | Encourage migration to ACCESS — good fit, can stay on CloudBank |
| ≥ $3,083/month (≥ $37K/year) | Include migration offer, but also write a note in the sheet: "High spender (~$Xk/month, ~$Xk/year) — may be a better fit for NAIRR, review with Shava" |

The $37K/year threshold (~$3,083/month) reflects what CloudBank can sustainably support. Above that, NAIRR is a better fit. The decision is still human-reviewed — N8N flags it, Fernando and Shava decide.

**monthly_spend source: Kion** — use `POST /v3/spend-report/funding-source` with the **last full calendar month** (not current month — partial data). Sum `spend` across all accounts for the fund. Fallback: `daily_rate × 30` if Kion unavailable.

If `monthly_spend ≥ $3,083`: Slack Fernando — "Fund X spending ~$Y/month (~$Z/year), exceeds $37K/year threshold, likely NAIRR candidate. W1 sent."

### ACCESS migration text (CBF W1 — approved by Shava)

> If you'd like to continue running on CloudBank, please migrate your account to ACCESS. You should have received an email on March 5 with instructions asking you to complete the migration by May 1. We recommend starting the migration as soon as possible since you are now overspent. Just as a reminder, instructions can be found here: https://www.cloudbank.org/training/access-cloudbank-research.

This replaces the previous text that said CloudBank "will transition in the next few months" — CloudBank is already in production.

---

## Warning Email Templates Reference

See `docs/templates.md` *(to be created in Sprint 3)* for full LLM prompt templates for each warning type.

| Template ID | Used for |
|---|---|
| `cbf_w1` | CBF Warning 1 (any tier) — always includes ACCESS migration offer |
| `cbf_w1_tier3` | CBF Warning 1, Tier 3 urgent — includes migration offer + heavy spend language |
| `cbf_w2` | CBF Warning 2 — re-emphasizes migration offer before escalating |
| `cbf_w3` | CBF Warning 3 + closing notice |
| `nairr_w1_10pct` | NAIRR ≤10% Warning 1 (includes supplement options) |
| `nairr_24h` | NAIRR 24h ultimatum |
| `nairr_overspend_strong` | NAIRR Overspend initial strong message |
| `nairr_thank_you` | NAIRR spending decrease acknowledgement |
| `supplement_approved` | Balance jumped — supplement approved |
