# Case Study: Quote to Signed Document Pipeline

**A reference implementation** — built end-to-end in n8n, tested against a sample
solar-installation enquiry and a battery of deliberate failure scenarios. Not a
deployed client project; no client name, quote, or result below is invented or
implied to be real.

## Problem

Service businesses that quote jobs from inbound leads — the reference scenario
here is a home-solar installer — typically handle each enquiry by hand: read the
free-text request, work out a scope and price, type it into a document, export a
PDF, email it, and remember to follow up. Every step is manual and repetitive, the
lead cools while it sits in a queue, and there's no structured record of whether a
quote was ever actually signed.

## Built

A 5-workflow n8n pipeline:

- **Webhook intake** with input validation and idempotency (a repeat enquiry ID is
  detected and skipped before any external service runs).
- **AI-drafted quote** (Google Gemini) that breaks a prospect's free-text
  requirements into structured, categorized line items — grounded strictly in what
  was said or standard-implied by the job type, with anything assumed listed
  separately rather than folded silently into the quote.
- **Deterministic pricing** — Config-driven base rates per category, a size-tier
  multiplier, and a discount threshold, computed in code rather than left to the AI.
- **Branded PDF rendering** via the same swappable-engine architecture as the
  Client Report Generator demo (self-hosted Gotenberg by default, a hosted API
  alternative fully built for n8n Cloud).
- **Delivery and record-keeping** — Google Drive, Gmail, and an Airtable record per
  quote.
- **A stubbed e-signature path** — no live paid e-sign account, but the callback
  receiver that would mark a quote signed is real, tested, and wired up, with the
  exact point to plug in a live provider clearly marked.
- **A genuine follow-up mechanism** — not a cron job bolted on after the fact, but
  a Wait step in the same execution that re-checks status and reminds only if
  nothing has changed.
- **Idempotency, retries, input/output validation, centralized error alerting, and
  append-only run logging** — the majority of the build effort, same as the first
  demo in this series, because an unattended pipeline has to survive bad input, a
  down dependency, and a malformed AI response without producing a duplicate or
  broken quote.

## Result

Verified through deliberate testing, not claimed from production use:

- **Idempotency confirmed** — a repeat enquiry ID skips cleanly with no duplicate
  email, PDF, or Airtable record, logged as `skipped`.
- **Full happy path verified live**, including the follow-up mechanism: a quote
  sent, then — after the configured delay — re-checked, found still unsigned, and
  reminded automatically, with the reminder and the original send both correctly
  reflected in the run log.
- **Three of four deliberate failure scenarios verified live**: a malformed
  webhook payload, the PDF engine unreachable, and malformed AI output (an
  out-of-vocabulary line-item category) each produced exactly one alert naming the
  real failing node and error, logged the run as failed, and delivered zero
  partial output. The fourth (an Airtable credential revoked mid-flight) could not
  be independently re-verified in this pass — the tooling used to build this
  validates credential existence and type on every assignment, which made it
  impossible to fake a broken credential without touching the real token.
- **A real bug was found and fixed during testing**, not swept under the rug: a
  Config node placed on a parallel branch (to avoid clobbering the real webhook
  payload, unlike the first demo's simpler linear Config) broke `.item`-style
  node references, which needed `.first()` instead — a genuine n8n execution-model
  gotcha, not a typo.
- **The AI-drafted quotes were consistently well-grounded** across every test run
  — correct category usage, sensible system-sizing assumptions stated explicitly
  rather than baked silently into a line item, and no invented scope beyond what
  the property size and free-text request actually implied.

## Stack

n8n · Google Gemini · Gotenberg (with a built hosted-API alternative) · Google
Drive · Gmail · Airtable
