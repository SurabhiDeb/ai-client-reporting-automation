# Build Runbook — Project 4: Client Reporting (Compute + Two-Gate + Narrative)

Goal: get the workflow running end-to-end on mock QuickBooks data so you can demo both paths — a clean month that auto-assembles, and a bad month that both gates hold for a human — then swap in a live QuickBooks account. Target time to a working demo: **~30–40 minutes** (most of it is the OpenAI + Slack credentials).

Files: `report-n8n-workflow.json` (import), `report-agent-prompts.md` (the two prompts, already embedded in the workflow), `report-loom-demo-script.md` (what to record).

---

## 0. What you need

- An **n8n** instance (cloud or self-hosted). Self-hosted is the better story for a financial-services client — their data never leaves their environment.
- An **OpenAI** API key (or Claude — see step 5 to swap). The demo uses `gpt-4o-mini`; it's cheap and fine for narrative.
- A **Slack** workspace where you can post to a channel (create `#client-reporting`).
- No QuickBooks account is needed for the demo — the pull is mocked.

---

## 1. Import the workflow

1. n8n → **Workflows** → **Import from File** → select `report-n8n-workflow.json`.
2. You'll see 13 live nodes plus two sticky notes ("How the gates work", "Setup Notes"). Read the yellow sticky — it's the setup cheat-sheet.
3. Nothing is active yet and the trigger is Manual, so importing is safe.

---

## 2. Add the OpenAI credential

1. Open **LLM · Draft Report Narrative** → Credentials → **Create New** → paste your OpenAI API key → save.
2. Open **LLM · Summarize Hold for Reviewer** → select the same credential.
3. Both nodes are set to `gpt-4o-mini` with JSON output on. Leave as-is.

---

## 3. Add the Slack credential

1. Open **Slack · Report Ready (approve to send)** → Credentials → connect your Slack account (OAuth or bot token with `chat:write`).
2. Set the channel to `#client-reporting` (create it first).
3. Do the same on **Slack · Human Review Queue** (same credential, same channel).

> If you don't want to wire Slack for a first dry run, you can disable both Slack nodes and read the output straight from **Log Run** — but the Slack cards are the demo, so wire them before recording.

---

## 4. Run the CLEAN path (auto-assemble)

1. Open **Mock QuickBooks Pull**. Confirm the top line reads `const SCENARIO = 'clean';`.
2. Click **Execute Workflow**.
3. Watch it flow: Compute → Gate 1 passes → Gate 2 passes → decision `deliver` → narrative drafts → **Deliver or Hold?** takes the top branch → **Slack · Report Ready** posts a finished report card.
4. Check Slack `#client-reporting`: you should see a ✅ *Report ready — one tap to send to client* card for **Meridian Design Co. · June 2026**, with the headline and narrative. **Log Run** records `outcome: auto_assembled`.

What you just proved: a clean month assembles a client-ready report with no human doing the mechanical work.

---

## 5. Run the HELD path (the money shot)

1. Open **Mock QuickBooks Pull**, change the top line to `const SCENARIO = 'held';`, save.
2. Click **Execute Workflow**.
3. Watch it flow to the bottom branch: **Gate 1** trips (22% uncategorized + accounts not reconciled), **Gate 2** trips (gross margin down 6.8 pts, fuel expense +68.8%, cash runway 2.6 mo below the 3-mo floor, KPI out of range) → decision `hold` → **Summarize Hold** → **Slack · Human Review Queue**.
4. Check Slack: a ✋ *Report HELD — needs review before it can go out* card for **Northwind Logistics LLC**, with a one-liner, an ordered fix-list, the flags, and the draft narrative — and an explicit "nothing was sent to the client." **Log Run** records `outcome: held_for_review`, `gate_tripped: both`.

What you just proved — and this is the whole pitch — the system **refused to auto-send a client a report built on dirty books with a material cash problem**, and handed the accountant a ready-to-clear review packet instead. That refusal is the judgment a firm is paying for.

Set `SCENARIO` back to `'clean'` when you're done, or leave it on `'held'` to record the hold clip.

---

## 6. (Optional) Swap OpenAI → Claude

The gates and all math are model-agnostic. To use Claude: replace the two `@n8n/n8n-nodes-langchain.openAi` nodes with Anthropic Chat nodes (or an HTTP request to the Anthropic API), keep the exact system/user prompts from `report-agent-prompts.md`, and keep JSON output on. Nothing else changes — the workflow still owns the numbers and the decision.

---

## 7. Going live (QuickBooks + delivery)

Four swaps, none of which touch the gate or compute logic:

1. **Data source.** Delete **Mock QuickBooks Pull**; add a **QuickBooks Online** node (read-scoped) pulling current-period P&L + balance sheet, prior period, and budget, plus a categorization/reconciliation read. Map its output to the same shape the mock returns (see the mock node's comments for the exact fields). Xero works the same way.
2. **Trigger + client loop.** Replace the Manual Trigger with a **Schedule Trigger** (e.g. 1st of month) feeding a client list, so it runs per client automatically.
3. **Delivery.** On the deliver branch, add a render step that drops the assembled figures + narrative into the firm's **branded Google Doc / Sheet / PDF** template, then sends to the client (or posts to the approval card for one-tap send, per client policy).
4. **Logging.** Replace **Log Run**'s code with a **Google Sheets Append Row** (or DB insert) so auto-vs-held and hours-saved are reportable from month one.

Then tune the thresholds (they're all named consts at the top of **Gate 1** and **Gate 2**): uncategorized-share limit, runway floor, revenue-move band, budget-miss %, margin-drop points, expense-spike %, and per-client KPI ranges. These are the firm's policy — expose them, don't hard-code your own.

---

## 8. Pre-demo checklist

- [ ] CLEAN scenario posts a ✅ Report Ready card with a real narrative.
- [ ] HELD scenario posts a ✋ Report HELD card with fix-list + flags + draft, and says nothing was sent.
- [ ] Both runs write a row via **Log Run** (auto_assembled / held_for_review).
- [ ] Narrative contains no invented numbers — spot-check the figures against the mock.
- [ ] `SCENARIO` left where you want it for recording.

Once these pass, record the Loom (`report-loom-demo-script.md`) — the held card is the shot that sells this.
