# Client Report Generator

An n8n automation that replaces a marketing agency's manual monthly client-reporting
process: pulling metrics, writing a summary, formatting a PDF, and sending it out —
work that otherwise eats an afternoon per client, every month.

## The problem

A digital marketing agency reports monthly on each client's performance (traffic,
leads, conversion rate, ad spend, backlinks, revenue, site speed). Every month, for
every client, someone has to:

1. Pull the latest numbers together.
2. Compare them to the prior month and decide what's actually worth calling out.
3. Write a short, sensible executive summary.
4. Format it into something client-ready.
5. File it and send it.

None of that is hard — it's just repetitive, and it's exactly the kind of task where
a tired person on the fifteenth client of the day starts skipping steps, or sends the
wrong month's numbers, or forgets to attach the file.

## What this builds

Four n8n workflows plus one Data Table, wired so the whole thing runs unattended:

| Workflow | Role |
|---|---|
| **Client Report Generator** | Main orchestrator: schedule trigger, idempotency check, data fetch (Sheets + PageSpeed Insights), validation, AI executive summary, delivery |
| **CRG - Render Report PDF** | Sub-workflow: builds the branded HTML report and converts it to PDF — swappable rendering engine (see below) |
| **CRG - Log Run** | Sub-workflow: upserts one row per run into the run log, keyed by run ID |
| **CRG - Error Alert** | Set as the Error Workflow on the other two — catches any failure, emails a clear alert, logs it as failed |
| **CRG_RunLog** (Data Table) | Structured history: run ID, timestamp, client, period, status, duration, error |

See [SETUP.md](SETUP.md) for exact credential and configuration steps, and
[workflows/](workflows/) for the four importable JSON files (credentials stripped —
see the note in SETUP.md on reconnecting sub-workflow references after import).

## Architecture

```
Schedule / Manual Trigger
        │
        ▼
     Config  ──────────────────────────────► (single node holding every
        │                                      per-client value: names,
        ▼                                      sheet, thresholds, emails,
Determine Reporting Period                     PDF engine choice)
        │
        ▼
Check Existing Report (idempotency) ──► already delivered? ──► log "skipped" ──► stop
        │ no
        ▼
   ┌────┴────┐
   ▼         ▼
Sheets    PageSpeed Insights
   │         │
   └────┬────┘
        ▼
  Validate Source Data ──► invalid? ──► Stop and Error ──► Error Alert
        │ valid
        ▼
  Compute Deltas & Flag Changes (>threshold%)
        │
        ▼
  Gemini: Executive Summary (JSON) ──► malformed? ──► Stop and Error ──► Error Alert
        │ valid
        ▼
  Create Dated Drive Folder
        │
        ▼
  Render PDF (sub-workflow) ──► Gotenberg or hosted API, by Config value
        │
   ┌────┴────┐
   ▼         ▼
Upload to   Email to
 Drive       Client
   │         │
   └────┬────┘
        ▼
Email internal notice ──► log "success"
```

Any failure anywhere in this chain — bad data, a down PDF engine, malformed AI
output — routes to **CRG - Error Alert** automatically via n8n's Error Workflow
setting, rather than needing to be wired node-by-node.

## Production-grade features

This isn't a happy-path demo. It was built and adversarially tested to survive the
failures a real unattended monthly job will eventually hit:

- **Idempotency.** Running the workflow twice for the same client/period is safe —
  the second run detects the existing successful delivery and stops before touching
  any external service, logging `status: skipped` instead of sending a duplicate
  report. Uses a zero-item-safety pattern (`alwaysOutputData: true` + a real-field
  check) rather than trusting row count, which is a common source of false negatives
  in "does this already exist" checks.
- **Input validation before any external call.** Missing metrics, non-numeric
  values, duplicate/ambiguous rows for a period, and a failed PageSpeed lookup are
  all caught by an explicit validation step — with a specific, human-readable error
  — before Drive, Gmail, or the AI call ever run. No half-built report ever reaches
  a client.
- **AI output is untrusted until parsed.** The executive-summary JSON is
  fence-stripped, `JSON.parse`'d in a try/catch, and schema-checked before use.
  Malformed AI output fails the run cleanly instead of shipping a broken report.
- **Retries on transient failures.** Sheets, PageSpeed, the PDF engine, Drive, and
  Gmail all retry (3 attempts, backed off) before being treated as a real failure.
- **Centralized, informative error alerting.** Every failure — anywhere in the
  chain, in either workflow — triggers one alert email naming the failing node, the
  actual error message, and the run/execution ID, plus a link straight to the failed
  execution. No silent failures, no digging through logs to find out what broke.
- **Every run is logged**, success or failure, with duration and error detail, to a
  structured Data Table — a queryable audit trail without a separate database.
- **Swappable PDF rendering engine.** Self-hosted Gotenberg (default: no per-document
  cost, data never leaves your infrastructure, requires Docker) or a hosted API
  (works on n8n Cloud, no infrastructure, small per-document cost after a free tier)
  — selected by a single Config value, no rewiring. See SETUP.md section 2.
- **One Config node per client.** Every per-client value (name, thresholds, sheet,
  recipient emails, PDF engine) lives in one Set node at the top of the main
  workflow. Onboarding a new client means editing that node, not hunting through
  the canvas.

## What's deliberately out of scope

- Multi-client fan-out (this runs one client per execution; batching many clients
  is a natural next step — trigger this workflow per client, e.g. from a small
  client-list loop).
- The hosted PDF-API path (PDFShift) is fully built and documented but not
  credentialed or live-tested in the reference instance — see SETUP.md.
- Google Drive folder reuse is temporarily disabled due to a restricted OAuth scope
  in the reference instance (documented in-canvas via a sticky note); every run
  currently creates a fresh dated folder rather than reusing one.
