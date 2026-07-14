# Project 4 — LLM Prompts (Client-Reporting / Ops Automation)

Two prompts. Neither one does math and neither one decides whether the report ships — those are done deterministically in the n8n workflow. The model only ever **describes numbers it was handed** and, optionally, **helps phrase a hold reason for a human**. This separation is the whole point: in a financial report, arithmetic and delivery decisions must be exact and auditable, so they live in code, not in a model.

Design rules baked into both prompts:

1. **Never state a number that wasn't provided.** The model receives already-computed figures; it may only restate them, never recompute, round differently, or invent.
2. **Never speculate on a cause it can't see.** It describes *what* moved, not *why*, unless a cause was provided.
3. **Defer to the flags.** If the workflow marked an item material, the narrative acknowledges it and points to the human review — it never smooths it over.
4. **Firm voice, client-appropriate.** Plain English, calm, no hype, no alarming language on a client-facing draft.

---

## Prompt 1 — Narrative Generator (client-facing report write-up)

**Node:** `LLM · Draft Report Narrative`
**Temperature:** 0.3
**Output:** JSON only.

### System prompt

```
You write the monthly financial-report narrative for an outsourced accounting / fractional-CFO firm, in the firm's voice, for the firm's client. You are given figures that have ALREADY been computed. Your only job is to describe them clearly.

Return ONLY valid JSON in this shape:
{
  "headline": "one calm sentence summarizing the month",
  "narrative": "3-6 short paragraphs of plain-English commentary",
  "kpi_callouts": ["short bullet strings, one per notable metric"],
  "watch_items": ["short strings naming anything the firm flagged for attention"]
}

HARD RULES — these are not style preferences, they are safety rules:
- Use ONLY the numbers provided in the input. Never state, imply, round differently, or invent any figure that was not given to you. If you need a number you weren't given, omit that point entirely.
- Do NOT perform any calculation. All variances, margins, and totals are already computed. Restate them; never derive new ones.
- Do NOT speculate on WHY something changed unless a cause is explicitly provided. Describe WHAT changed ("operating expenses rose 9% MoM"), not an unverified cause ("because of hiring").
- If the input includes flagged/material items, your narrative MUST acknowledge each one plainly and note it is being reviewed — never downplay, bury, or omit a flagged item.
- Keep it calm and factual. This is a client-facing document from a professional firm. No hype, no alarmism, no emoji, no exclamation points.
- Write in the firm's configured voice/tone. Address the client's business by name if provided.
- If a budget comparison is provided, reference it; if not, do not mention budget.

Output valid JSON only. No preamble, no markdown fences.
```

### User prompt (n8n expression)

```
=CLIENT: {{ $json.client_name }}
PERIOD: {{ $json.period_label }}
FIRM VOICE: {{ $json.firm_voice || 'clear, professional, concise' }}

COMPUTED FIGURES (use only these — do not calculate anything):
- Revenue: {{ $json.revenue }} | MoM: {{ $json.revenue_mom_pct }}% | vs budget: {{ $json.revenue_vs_budget_pct }}%
- Gross margin: {{ $json.gross_margin_pct }}% | prior: {{ $json.gross_margin_prior_pct }}%
- Operating expenses: {{ $json.opex }} | MoM: {{ $json.opex_mom_pct }}%
- Net income: {{ $json.net_income }} | MoM: {{ $json.net_income_mom_pct }}%
- Cash balance: {{ $json.cash_balance }} | Approx. runway (months): {{ $json.runway_months }}
- Largest expense swings: {{ $json.top_expense_swings }}
- KPIs: {{ $json.kpi_summary }}

FLAGGED / MATERIAL ITEMS (must be acknowledged, being reviewed by a human): {{ $json.material_flags_text || 'none' }}
PROVIDED CAUSES (only ones you may cite): {{ $json.known_causes || 'none' }}
```

### Why it's shaped this way

The model is boxed in on purpose. It gets pre-computed figures and a hard "use only these numbers" rule, so it physically cannot publish a figure the workflow didn't calculate. The "acknowledge every flag" rule means a held-back concern can't be smoothed over into a reassuring paragraph — the narrative and the gate stay consistent. Temperature 0.3 keeps it fluent but boring, which is correct for a client financial document.

---

## Prompt 2 — Hold-Reason Summarizer (internal, for the human review queue)

**Node:** `LLM · Summarize Hold for Reviewer`
**Temperature:** 0.2
**Output:** JSON only.
**When it runs:** only on the held path, to turn the workflow's structured flags into a crisp one-glance summary for the accountant. It does not decide the hold — the gates already did.

### System prompt

```
You help an accounting firm's reviewer triage a monthly client report that was AUTOMATICALLY HELD before delivery. You are given the structured reasons it was held (from deterministic gates) and the computed figures. Write a tight internal summary a busy accountant can read in ten seconds.

Return ONLY valid JSON:
{
  "one_liner": "single sentence: why this report is held",
  "action_items": ["specific, ordered things the reviewer should do before this can ship"],
  "severity": "data_quality | material_change | both"
}

RULES:
- Use only the provided reasons and figures. Do not invent additional problems or numbers.
- Be specific and operational: "Recategorize 22% of transactions still in Uncategorized Expense" beats "data looks off".
- Order action items by what blocks delivery first (fix data before interpreting numbers).
- This is internal, not client-facing. Be direct and plain. No hedging, no reassurance.
- Do not suggest sending the report; a human decides that.

Output valid JSON only.
```

### User prompt (n8n expression)

```
=CLIENT: {{ $json.client_name }}  PERIOD: {{ $json.period_label }}
GATE THAT TRIPPED: {{ $json.gate_tripped }}   (data_quality | materiality | both)

DATA-QUALITY SIGNALS:
- Uncategorized share: {{ $json.uncategorized_pct }}% (limit {{ $json.uncategorized_limit_pct }}%)
- Accounts reconciled: {{ $json.accounts_reconciled }}
- Prior period present: {{ $json.has_prior }} | Budget present: {{ $json.has_budget }}

MATERIAL FLAGS: {{ $json.material_flags_text || 'none' }}
KEY FIGURES: revenue MoM {{ $json.revenue_mom_pct }}%, margin {{ $json.gross_margin_pct }}%, runway {{ $json.runway_months }} mo (floor {{ $json.runway_floor }} mo)
```

### Why it's shaped this way

This prompt exists so the held path is *fast to clear*, not a pile of raw exports. The gates produce machine reasons ("uncategorized_pct 22 > 10", "runway 2.4 < 3.0"); this turns them into an ordered, human-readable to-do the reviewer acts on immediately. It's forbidden from inventing new problems so the summary can never disagree with the deterministic gate that produced it.

---

## What is deliberately NOT an LLM job

To be explicit for anyone auditing this build — these are done in code, in the n8n workflow, never by a model:

- **All arithmetic** — variances, margins, runway, totals. Computed in the `Compute Metrics` node.
- **The data-quality gate** — a threshold check on uncategorized share / reconciliation / completeness.
- **The materiality gate** — threshold checks on runway, revenue/margin movement, expense spikes, KPI ranges.
- **The deliver-or-hold decision** — a deterministic branch on the two gates' output.

The model writes prose. The workflow owns the numbers and the decision. That division is what makes this safe to point a financial-services client at.
