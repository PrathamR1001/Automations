# Quote to Signed Document Pipeline

An n8n automation that takes a service-business lead from inbound enquiry to a
sent, priced, branded quote — with the architecture already in place to track it
through to a signed document.

## The problem

Service businesses (the reference scenario here: a home-solar installer) handle
inbound leads by hand: read the enquiry, work out a rough scope and price, type it
into a document, export a PDF, email it, and then remember to follow up. Every step
is manual, the lead cools while it waits in a queue, and nothing tracks whether the
quote was ever signed.

## What this builds

Five n8n workflows:

| Workflow | Role |
|---|---|
| **Quote to Document Pipeline** | Main orchestrator: webhook intake, validation, idempotency, AI-drafted quote, pricing, delivery, follow-up |
| **QTD - Render Quote PDF** | Sub-workflow: branded HTML → PDF, swappable rendering engine (same pattern as the Client Report Generator demo) |
| **QTD - Log Run** | Sub-workflow: append-only run-history log (success / skipped / failed / reminder_sent / signed) |
| **QTD - Error Alert** | Error handler for the other workflows — one clear alert email per failure |
| **QTD - Signature Callback Receiver** | Placeholder webhook: marks a quote signed the moment a real e-sign provider is connected |

See [SETUP.md](SETUP.md) for exact credential/config steps and
[workflows/](workflows/) for the five importable JSON files (credentials stripped —
see SETUP.md for re-linking sub-workflow references after import).

## Architecture

```
Webhook (lead payload)  ──┐
Manual Trigger + sample ──┴──► Normalize & Validate Input
                                       │
                          invalid? ──► Stop and Error ──► Error Alert
                                       │ valid
                                       ▼
                          Check Existing Quote (idempotency, by enquiryId)
                                       │
                       already sent? ──► log "skipped" ──► stop
                                       │ no
                                       ▼
                          Gemini: draft quote (scope + priced-category line items)
                                       │
                        malformed? ──► Stop and Error ──► Error Alert
                                       │ valid
                                       ▼
                          Compute Quote Pricing (rates × size-tier × discount, from Config)
                                       │
                          Render PDF (sub-workflow, swappable engine)
                                  ┌────┴────┐
                                  ▼         ▼
                          Upload to Drive   Email to Prospect
                                  │
                          Create Airtable Record (status: sent)
                                  │
                          Send for E-Signature ── disabled placeholder
                                  │
                          Internal notice ──► log "success"
                                  │
                          Wait (follow-up delay)
                                  │
                          Re-check Airtable status
                                  │
                    still "sent"? ──► reminder email + log "reminder_sent"
                                  └──► (signed) do nothing
```

A signature, whenever it actually arrives, comes in on a **separate** workflow
(QTD - Signature Callback Receiver) — an e-sign provider calls back on its own
schedule, not as a continuation of the original execution.

## Production-grade features

- **Idempotency.** The caller supplies a stable `enquiryId`; a repeat call for the
  same one is detected before any external service runs and logged as `skipped` —
  no duplicate quote, no duplicate Airtable record.
- **Input validation before any external call.** Missing or malformed fields (bad
  email, non-numeric property size, missing free-text requirements) fail with a
  specific message before the AI, Drive, Airtable, or Gmail are ever touched.
- **AI output is untrusted until parsed.** The quote JSON is fence-stripped,
  `JSON.parse`'d in a try/catch, schema-checked, and — since pricing depends on it —
  every line item's category is checked against the fixed rate-table keys. An
  out-of-vocabulary category fails the run by name rather than silently mispricing
  or crashing downstream.
- **Retries on transient failures** across Airtable, the PDF engine, Drive, and
  Gmail.
- **Centralized error alerting.** Any failure, in any of the five workflows, is
  caught by n8n's Error Workflow mechanism and emailed with the failing node, the
  real error message, and the run ID — no per-node alert wiring needed.
- **Structured, append-only run logging.** Every event in a quote's life (sent,
  skipped, failed, reminder sent, signed) gets its own logged row rather than
  overwriting a shared mutable field — Airtable's `Status` column stays the single
  source of truth for "where is this quote right now."
- **Swappable PDF rendering engine.** Self-hosted Gotenberg by default; a hosted
  API alternative is fully built for platforms that can't run Docker (e.g. n8n
  Cloud) — one Config value switches it.
- **One Config node.** Company/brand name, pricing rates, size-tier and discount
  rules, Airtable location, follow-up delay, and PDF engine all live in one place.

## What's deliberately out of scope

- **E-signature is stubbed, not live.** No paid DocuSign/Dropbox Sign account is
  wired up. The "Send for E-Signature" node is a disabled placeholder marking
  exactly where that API call would go; the callback receiver is real and working,
  ready to flip a quote to `signed` the moment a provider is connected — see SETUP.md.
- **Follow-up delay is compressed for demos** (2 minutes) rather than the realistic
  production value (documented in-canvas as ~3 days / 4320 minutes). n8n's Wait
  node handles both the same way — it isn't a special demo mechanism.
- **Airtable credential-revoked failure mode** was not independently re-verified in
  the final testing pass in this reference instance (tooling validates credential
  existence/type, so it can't be faked without touching the real token) — the other
  three deliberate failure scenarios (malformed input, PDF engine down, malformed AI
  output) were all verified live.
