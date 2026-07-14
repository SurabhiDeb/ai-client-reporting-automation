# Loom Demo Script — Project 4: Automated Client Reporting

**Length:** 90–120 seconds. Two clips: the clean auto-assemble (fast), then the held month (the money shot — spend your time here).
**Goal for a watching client:** "This does my monthly report grunt work *and* it won't embarrass me by auto-sending a bad one." Lead with the refusal, not the automation.

**Before recording:** run through the pre-demo checklist in the runbook. Have n8n open on the workflow canvas and Slack `#client-reporting` visible in a second window.

---

## Clip 1 — The clean month (~30 sec)

**On screen:** n8n canvas, then Slack.

> "Every month, an accounting firm builds the same report for every client — pull QuickBooks, compute the variances, write the commentary. Here's that whole thing, automated. This is a clean month."

*(Click Execute. Let the nodes light up left to right.)*

> "It pulled the numbers, computed every variance in code — not with the AI — checked the data was clean, checked nothing moved past threshold, then wrote the narrative."

*(Switch to Slack, show the ✅ Report Ready card.)*

> "Finished report, ready for a five-second glance before it goes to the client. That's most months — done in under a minute instead of half a day."

---

## Clip 2 — The held month (~60–75 sec) — THE MONEY SHOT

**On screen:** n8n canvas → the Mock Pull node → Slack.

> "But here's what actually matters. Watch what happens on a *bad* month."

*(Open Mock QuickBooks Pull, change `SCENARIO` to `'held'`, save. Execute.)*

> "Same system. This client's books are a mess — 22% of transactions still uncategorized, accounts not reconciled — and the numbers are ugly: gross margin down almost 7 points, fuel costs up 69%, and cash runway just dropped to 2.6 months."

*(Point at the flow taking the bottom branch.)*

> "A dumb automation would have templated all of that and emailed it straight to the client. This one doesn't. It hit two gates. Gate one: is the data even real? No — too much uncategorized. Gate two: did anything move enough to need a human? Yes — margin, expenses, and runway all tripped."

*(Switch to Slack, show the ✋ Report HELD card. Scroll it slowly.)*

> "So instead of sending the client anything, it stops and hands the accountant this: one line on why it's held, an ordered fix-list — clean up the categorization first — the flagged numbers, and the draft it already wrote. Nothing reached the client. The human clears it in a couple of minutes instead of rebuilding it."

*(Pause on the "nothing was sent to the client" line.)*

> "That line is the whole point. The system does the mechanical work every month, but it will not put your name on a report built on dirty books or a cash problem you haven't seen yet. It knows which reports are safe to send and which two numbers need you first."

---

## Closing line (~10 sec)

> "Assembles the routine reports in minutes, flags the ones that need judgment, and never auto-sends a bad one. That's the difference between a dashboard and a system you'd actually trust with a client relationship. Happy to walk through wiring it to your QuickBooks."

---

## Recording notes

- **Slow down on both Slack cards.** The reviewer packet — fix-list, flags, "nothing was sent" — is the proof. Let it breathe.
- **Say "computed in code, not by the AI" once.** Financial buyers care that the arithmetic is deterministic and auditable; it separates you from "I plugged GPT into QuickBooks."
- **Don't over-explain the nodes.** The buyer cares about the decision and the refusal, not the canvas. Keep the technical bits to one sentence each.
- **The hook to open a proposal with:** *"Your team spends a day a month per client assembling the same report. This assembles it in minutes and flags the two numbers worth a human's attention — and it won't send a client anything built on uncategorized data or a number that just moved past your threshold."*
