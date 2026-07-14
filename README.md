# Automated Monthly Client Reporting — Fractional CFO & Outsourced Accounting

**Pulls each client's numbers from QuickBooks, computes the variances, writes the plain-English monthly narrative, and flags the two or three numbers a human actually needs to look at before anything reaches the client. Clean months assemble in minutes; risky months stop and wait for an accountant.**

Built for fractional-CFO firms and outsourced accounting/bookkeeping practices that owe every client the same monthly report and lose a day per client assembling it by hand.

`n8n` `QuickBooks Online / Xero` `Claude or OpenAI` `Slack` `Google Docs/Sheets/PDF`

---

## The problem

Assembling a monthly client report is almost entirely manual and repetitive: export the P&L and balance sheet, calculate variances by hand, scan for anomalies, write the same style of commentary — for every client, every month. The obvious fix (full automation) creates a worse problem: a financial report is a trust document, and an automation that ships a report built on uncategorized data or a missed anomaly puts the firm's name on a wrong number, automatically, at scale.

## What it does

- Pulls current period, prior period, and budget from QuickBooks, plus a data-quality signal (uncategorized/unreconciled share)
- **Computes every variance, margin, and runway figure deterministically in code** — the LLM never does the math
- **Gate 1 — data quality:** won't draft a client-facing report at all if the underlying data isn't clean (uncategorized share over threshold, accounts unreconciled, budget/prior period missing)
- **Gate 2 — materiality:** if any KPI crosses a set threshold (runway floor, margin/revenue band, expense spike), the report holds for a human instead of auto-delivering
- Drafts the narrative with an LLM, grounded *only* in the already-computed figures — never invents a number or a cause
- Delivers clean reports via one-tap approval (or auto-send per client policy); routes held reports to an accountant with the flagged numbers and draft attached

## Architecture

```
Scheduled (or manual) run, per client
        │
        ▼
   Pull QuickBooks: current period, prior period, budget, data-quality signal
        │
        ▼
   Gate 1 — data quality: uncategorized under threshold? key accounts reconciled?
        │                   prior period + budget present?
        │
        ├─ FAILS ──► STOP. Route to firm: "here's what's missing." No report drafted.
        │
        └─ PASSES
              │
              ▼
        Compute variances, margins, runway — in code, deterministic
              │
              ▼
        Gate 2 — materiality: did anything cross threshold (runway floor,
                  margin/revenue band, expense spike, KPI range)?
              │
              ├─ YES ──► LLM drafts narrative ──► held for accountant, flags highlighted
              │
              └─ NO  ──► LLM drafts narrative ──► auto-assemble ──► one-tap approval / auto-send
```

**Two independent gates, each guarding a different risk:** Gate 1 asks "is the data real?" Gate 2 asks "did something move enough that a human must weigh in before the client sees it?" A report ships automatically only if both say yes.

## Mock vs. live

| | Demo (this repo) | Client deployment |
|---|---|---|
| Data source | Mock JSON payload (realistic P&L + balance sheet + budget) | Client's real QuickBooks Online (or Xero) |
| Review/approval | Test Slack channel | Firm's real Slack (or email) |
| Report render | Structured output | Firm's branded Google Doc/Sheet/PDF template |
| Thresholds (materiality floors, uncategorized limit, runway floor, KPI ranges) | Sample config | Config the firm tunes per client |

## Projected outcome

*Modeled from typical firm reporting effort — this build is pre-deployment, figures will be replaced with real data as it goes live.*

~6 hours per client per month reclaimed · anomalies surfaced before the client notices · zero client-facing reports on incomplete or materially-anomalous data without sign-off, by design · consistent on-time reporting at scale · every run logged (auto-assembled vs. held, gate tripped, hours saved).

## What's in this repo

```
workflow/n8n-workflow.json   importable n8n workflow (13 nodes, both gates, validated)
docs/case-study.md           full problem → architecture → outcome writeup
docs/agent-prompts.md        narrative-drafting prompt (grounded-only, no invented numbers)
docs/build-runbook.md        step-by-step setup/deploy instructions
docs/demo-script.md          walkthrough script used for the recorded demo (clean path + held path)
```

## Adapting this pattern

The pull → compute deterministically → gate on data-quality → gate on materiality → draft → hold-or-deliver pattern transfers to wealth/RIA client reporting (drawdown/allocation-drift gates), marketing-agency reporting (spend-pacing/performance-anomaly gates), and internal ops/KPI reporting for leadership.

---

Built by Surabhi Deb as part of an AI automation portfolio. [See the full project index]([https://github.com/YOUR-USERNAME](https://surabhideb.github.io/earlyecho-website/)) for related builds (voice lead qualification, RAG support agents, speed-to-lead systems).
<img width="1786" height="702" alt="Client Reporting" src="https://github.com/user-attachments/assets/e1e38da9-ff5b-4f60-acf0-10f7c24e3116" />


## License

MIT — see [LICENSE](LICENSE).
