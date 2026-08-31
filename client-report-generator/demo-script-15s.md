# Demo Script — 15 Seconds (Failure-Alert Clip)

Goal: prove the failure path is real and specific, fast. No setup, no narration
over the canvas — just the alert landing and the log confirming it.

---

**0:00–0:02 — Cut straight to the inbox.**
An alert email is already visible, arriving in real time if possible (notification
badge / animation), subject "CRG Alert: Workflow Failed — Client Report Generator".

**0:02–0:10 — Open it. Hold on the body.**
Visible in frame: failing node name, the actual error message, the run ID, and the
"View execution in n8n" link. Let it sit long enough to read — don't cut away
early.
> (optional voiceover, 1 line) "Names the exact node, the real error, the run ID.
> Not a generic failure notice."

**0:10–0:15 — Cut to the run log.**
Show the CRG_RunLog Data Table, latest row: `status: failed`, matching run ID,
error text visible in the row.
> (optional voiceover) "And it's logged — nothing fails silently."

---

**Shot list:** alert email (subject → opened body) → run log row. Two shots. No
canvas, no trigger, no build-up — the point is that failure produces a specific,
logged, human-readable alert, shown in the time it takes to read one.
