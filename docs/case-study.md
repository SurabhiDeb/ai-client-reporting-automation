# Automated Monthly Client Reporting — for Fractional CFO & Outsourced Accounting Firms

**A system that pulls each client's numbers from QuickBooks, computes the variances, writes the plain-English monthly narrative, and — before anything reaches the client — flags the two or three numbers a human actually needs to look at. Clean months assemble in minutes; risky months stop and wait for an accountant.**

> _Built for fractional-CFO firms, outsourced accounting/bookkeeping practices, and financial-services teams that owe every client the same monthly report — P&L, cash position, KPIs, and a "here's what happened" narrative — and lose a day a month per client assembling it by hand. It pulls the numbers, calculates the movements, drafts the write-up, and runs two hard checks before delivery: is the underlying data actually clean, and did anything move enough to need a human? If either check trips, the report is held for review instead of sent._

**Who it's for:** Fractional CFOs, outsourced accounting/bookkeeping firms, and financial-services teams producing recurring client-facing financial reports.
**What it replaces:** an analyst exporting the same QuickBooks reports, pasting them into a template, eyeballing what changed, and retyping the same variance commentary for every client, every month.
**The guardrail that makes it safe:** it never sends a client-facing financial report built on incomplete or uncategorized data, and never auto-delivers a report where a material number moved past threshold — those always route to a human first.

---

## The Problem

Every fractional-CFO and outsourced-accounting firm sells the same recurring deliverable: a monthly financial report per client. P&L versus last month and versus budget, cash position and runway, a few operating KPIs, and a paragraph or two of narrative that says *what changed and why it matters*. The client pays a retainer largely for that report and the judgment inside it.

The problem is that assembling it is almost entirely manual and almost entirely repetitive. An analyst opens QuickBooks, exports the P&L and balance sheet, drops the figures into a template or a deck, calculates the month-over-month and budget variances by hand or in a fragile spreadsheet, scans for anything weird, and writes the same style of commentary they wrote last month — for every client on the roster. A firm with fifteen clients can burn the better part of a week each month on report assembly that is 90% mechanical.

So firms reach for the obvious fix: automate the report. Connect QuickBooks to a dashboard, template the whole thing, and ship it on a schedule. And that creates a different, worse problem. A financial report is a trust document. If the automation quietly ships a client a report where half the month's transactions were still uncategorized, or where cash runway just dropped below three months and nobody flagged it, or where a one-time entry made margin look great — the firm has now put its name on a wrong or misleading number, automatically, at scale. In financial services a bad number isn't a typo; it's a liability and a lost client. Full automation is fast but reckless; the manual process is safe but slow and expensive.

The real need is a system that does the mechanical 90% instantly *and* knows which 10% a human must not skip — one that assembles the routine report without a person, but deliberately refuses to deliver when the data is dirty or a number moved enough to matter.

---

## What It Does

On a schedule (or on demand) for each client, the system:

- **Pulls the numbers** from QuickBooks Online — current-period P&L and balance sheet, the prior period, and the budget — plus a data-quality signal (how much is still uncategorized or unreconciled).
- **Normalizes and computes** the movements deterministically in code, not with an LLM: month-over-month and budget variances on every line, gross and net margin, cash balance and an approximate runway, and the largest expense swings. The math is done by the workflow so it's exact and auditable — the model never invents or recomputes a figure.
- **Runs a data-quality gate first.** Before it will build a client-facing report at all, it checks that the underlying data is trustworthy: uncategorized transactions under threshold, key accounts reconciled, prior-period and budget present. If the data is dirty, it stops — no report is drafted on numbers that aren't real yet — and routes to the firm with exactly what's missing.
- **Detects anomalies against thresholds** — revenue or margin moving beyond a set band, cash runway below a floor, an expense category spiking, a KPI outside its expected range. These are deterministic checks with named reasons, not vibes.
- **Drafts the narrative** with an LLM, grounded *only* in the computed figures: a clean, plain-English monthly write-up in the firm's voice that explains what moved and by how much — with hard instructions never to state a number it wasn't given and never to speculate on causes it can't see.
- **Runs the materiality gate.** If any anomaly crosses a material threshold, the report does **not** auto-deliver. It routes to an accountant with the flagged numbers highlighted and the draft narrative attached — "here are the two numbers worth your attention before this goes out." If nothing is material and the data is clean, the report assembles and moves to delivery (one-tap approval, or auto-send if the firm has opted in for that client).
- **Delivers and logs.** The assembled report goes out (or to the approval queue), and every run is logged: auto-assembled vs. held, which gate tripped, and the time saved — so the firm can see what the system is carrying and what it's correctly stopping.

A typical clean path: a client's month closes, the pull comes back fully categorized and reconciled, variances are all within band, cash runway is healthy. The system computes everything, drafts a tidy narrative — "Revenue up 4% MoM and 2% ahead of budget; operating expenses flat; cash runway steady at 8 months" — and drops a finished report into the approval queue for a five-second glance before it goes to the client.

A typical held path: another client's pull shows 22% of transactions still uncategorized and cash runway dropped to 2.4 months. The system does **not** assemble a client-facing report. The data-quality gate trips on the uncategorized backlog, and even if it hadn't, the materiality gate trips on the runway floor. It routes to the firm's accountant in Slack: "Held — 22% uncategorized (clean up before reporting); cash runway 2.4 mo, below your 3-mo floor. Draft narrative attached, not sent." The accountant fixes the categorization and personally frames the runway conversation — which is exactly the month you don't want a bot quietly shipping a report.

---

## Architecture & Decision Logic

The point of this build isn't "it can generate a report." Any dashboard can generate a report. The point is that it makes an explicit, defensible decision about **when a report is safe to deliver** — and deliberately refuses on the months where an automated send is a mistake.

**Trigger → orchestration.** A schedule (or manual run) fires an n8n workflow per client. n8n is the brain: it pulls, computes, runs both gates, drafts, routes, and logs. The LLM writes the narrative; it does not decide whether the report ships, and it does not do the math. The workflow does both.

**Pull + normalize.** The QuickBooks payload is pulled for the current period, prior period, and budget, plus a data-quality read. A normalization step maps it to a stable internal shape so everything downstream works the same regardless of the client's chart-of-accounts quirks.

**Deterministic computation (not the LLM).** Every variance, margin, and runway figure is calculated in code. This is deliberate: financial numbers must be exact and reproducible, so the model is never in the loop for arithmetic. It only ever *describes* numbers the workflow already computed.

**Gate 1 — the data-quality gate (proves data-quality awareness).** Before any client-facing narrative is drafted, the workflow checks the data is fit to report on: uncategorized share under threshold, key accounts reconciled, prior period and budget present. Dirty data → stop and route to the firm with the specific problem. This guards the single worst failure mode — confidently reporting on numbers that aren't finished — and it's the check that signals to a financial buyer that the builder understands their world.

**Gate 2 — the materiality gate (proves judgment).** After computation, deterministic anomaly checks run against firm-set thresholds. If nothing is material, the report is safe to auto-assemble and deliver. If anything crosses a material line — runway below floor, revenue/margin beyond band, an expense spike, a KPI out of range — the report is held for a human, with the flags highlighted and the draft attached. Two independent gates, each guarding a different risk: **Gate 1 asks "is the data real?", Gate 2 asks "did something move enough that a human must weigh in before the client sees it?"** A report ships automatically only if both say yes.

**Human-in-the-loop.** A held report is not a dead end that dumps raw exports on an accountant. The system still does the work — computes everything, drafts the narrative, and hands over a review packet with the flagged numbers called out and the reason for the hold — so the human's job is *review and decide*, not *rebuild the report*. That's what keeps held months fast to clear instead of shifting the whole burden back onto a person.

**Delivery + measurement.** Clean, immaterial reports go to a one-tap approval (or auto-send per client policy); held reports go to the review queue. Every run logs which path it took, which gate tripped, and estimated hours saved — so the firm can see, from month one, how much assembly the system absorbed and how often it correctly held.

**The demo-to-live swap point.** In the portfolio build, the QuickBooks pull is a mock JSON payload (a realistic P&L + balance sheet + budget for a sample client), computation and both gates run locally, the review/approval posts to a test Slack channel, and the "delivered report" is the assembled structured output (ready to render into the firm's Google Doc / Sheet / PDF template). For a real client, four things swap in with no change to the core logic: (1) their QuickBooks Online (or Xero) account on the pull node, (2) their branded report template on the render/delivery step, (3) their Slack (or email) as the review and approval surface, and (4) their thresholds — materiality floors, the uncategorized-share limit, the runway floor, expected KPI ranges — which are all config, not hard-coded. The gates, the computation, and the narrative logic don't change. Clean seams, so a demo becomes a deployment in days.

---

## Projected Outcomes

_These figures are illustrative targets based on typical firm reporting effort, not results from a live client deployment — this build is being finished now. They're framed the way I'd model ROI for a real firm, and I'll replace them with actual data as the system goes live._

- **~6 hours per client per month reclaimed** — the mechanical export-compute-template-write cycle collapses from most of a day to a few-minute review, per client, per month.
- **Anomalies surfaced before the client notices** — a cash-runway floor or margin swing is flagged to the firm's accountant on assembly, not discovered on the client call.
- **Zero client-facing reports on incomplete or materially-anomalous data without a human sign-off** — by design, because the two gates hold those back automatically. The data-quality discipline and the judgment call are built into the workflow, not left to whoever's rushing at month-end.
- **Consistent, on-time reporting at scale** — every client gets the same standard of report on schedule, and the firm can add clients without adding proportional assembly hours.
- **Every run logged** — auto-assembled vs. held, which gate tripped, and hours saved are reportable from day one, so the firm can see exactly what the system is carrying.

The headline claim I'd put on a proposal: **"Your team spends a day a month per client assembling the same report. This assembles it in minutes and flags the two numbers worth a human's attention — and it won't send a client anything built on uncategorized data or a number that just moved past your threshold."**

---

## Adapting to Your Business

The architecture is written for financial reporting, but the pattern — pull, compute deterministically, gate on data-quality, gate on materiality, draft, and hold-or-deliver — transfers cleanly to any recurring report where a wrong or premature number is costly:

- **Wealth / RIA client reporting:** same shape on portfolio/performance data; the materiality gate flags drawdowns, allocation drift, or anything requiring an advisor's framing before it reaches a client.
- **Marketing-agency client reporting:** swap QuickBooks for GA4/Ads/CRM; the gates become spend-pacing and performance-anomaly checks, so a client never gets an auto-report showing a blown budget without the account manager weighing in first.
- **Ops / KPI reporting for leadership:** internal weekly or monthly operating reports where routine weeks auto-assemble and any metric out of band routes to a human before it lands in the leadership channel.

In each case the swap is the data source and the specific thresholds — the pull → compute → two-gate → draft → hold-or-deliver backbone stays the same.

---

## How It's Built

Stack: n8n (orchestration + deterministic computation + both gates) · QuickBooks Online / Xero (financial data) · Claude or OpenAI (narrative only) · Slack (review + one-tap approval) · Google Docs / Sheets / PDF (branded report render)

- **n8n** orchestrates everything and owns every decision: it pulls the data, computes all variances and metrics in code, runs the data-quality gate and the materiality gate, calls the LLM to draft the narrative, and routes to auto-deliver or hold. Self-hostable, so financial data stays in the firm's environment and there's no per-workflow platform lock-in. The gate logic lives here, in the open, auditable — not buried inside a black-box reporting tool.
- **QuickBooks Online** (or Xero) is the source of truth: the workflow pulls the P&L, balance sheet, prior period, budget, and the categorization/reconciliation signal that feeds the data-quality gate, using a read-scoped connection.
- **The LLM** (Claude or OpenAI) does one narrow job: turn the already-computed figures into a clear, firm-voiced narrative — with hard instructions never to state a number it wasn't given, never to invent a cause, and to defer to the flagged items. It is explicitly not trusted to compute anything or to decide whether the report ships.
- **Slack** is the human-in-the-loop surface: held reports arrive as a review packet — the flagged numbers, the reason for the hold, and the draft narrative — ready to fix, approve, or send; clean reports arrive as a one-tap "send to client."
- **The render step** drops the assembled figures and narrative into the firm's branded Google Doc / Sheet / PDF template, so the client-facing artifact looks like the firm, not like a tool.

The whole system is built to be importable and configurable: the n8n workflow exports as a JSON file a firm can drop into their own instance, and the materiality thresholds, the uncategorized-share limit, the runway floor, the expected KPI ranges, and the firm's narrative voice are parameters — not hard-coded — so it adapts to a specific firm and each of its clients without touching the core logic.
