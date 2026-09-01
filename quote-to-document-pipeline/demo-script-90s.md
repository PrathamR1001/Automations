# Demo Script — 90 Seconds (Outcome Walkthrough)

Goal: lead with the result — a real quote landing in an inbox minutes after an
enquiry came in — then earn the "how" in whatever seconds remain. Don't open on
the n8n canvas.

---

**0:00–0:10 — Open on the enquiry.**
Show the raw lead — a form submission or a simple JSON payload — with a
free-text request: "want solar panels to cut our electric bill, no battery needed."
> "That's all it takes to start. No one on our side touched it yet."

**0:10–0:35 — Cut to the inbox. Open the quote email, then the PDF.**
Scroll the PDF: branded header, scope summary, property details, line items with
quantities and rates, total.
> "A minute or two later, a priced quote is already in their inbox. The line items
> aren't templated — they're built from what the prospect actually said, priced
> against our own rate table, not whatever the AI feels like charging."

**0:35–0:45 — Point at the "Clarifying Assumptions" section.**
> "Anything it had to assume — system size, roof condition — it says so, here,
> instead of quietly baking a guess into the price."

**0:45–0:55 — Cut to Airtable.**
Show the record: status `sent`, PDF link, quote total.
> "Every quote gets a record. This is what lets the pipeline know, later, whether
> it's still waiting on a signature."

**0:55–1:10 — Cut to the n8n canvas, briefly.**
Wide shot of the full pipeline, then zoom into Config.
> "One config node per client — pricing rules, brand name, which PDF engine to
> use. Same architecture as the reporting demo before this one."

**1:10–1:25 — Show a second inbox moment: the follow-up reminder.**
> "If nothing's changed by the time the follow-up window passes, it checks status
> and sends exactly one reminder — not sooner, not twice."

**1:25–1:30 — Close.**
> "E-signature isn't live here — that's a real account away — but everything up to
> that point runs end to end, unattended."

---

**Shot list:** enquiry payload → quote email → PDF (scrolled, pause on
assumptions) → Airtable record → n8n canvas (wide, then Config) → reminder email.
