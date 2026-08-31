# Case Study: Client Report Generator

**A reference implementation** — built end-to-end in n8n, tested against sample
agency data and a battery of deliberate failure scenarios. Not a deployed client
project; no client name, quote, or result below is invented or implied to be real.

## Problem

Digital marketing agencies typically report to each client monthly: current traffic,
leads, conversion rate, ad spend, backlinks, revenue, and site performance, compared
to the prior month, with commentary on what moved and why. Producing one of these by
hand — pulling the numbers, deciding what's notable, writing it up, formatting a PDF,
filing it, sending it — is a repetitive, error-prone task that scales linearly with
client count and is exactly the kind of work automation is suited to, provided the
automation is trustworthy enough to run completely unattended.

## Built

A 4-workflow n8n pipeline (plus one Data Table for run history):

- **Data ingestion** from Google Sheets (client metrics) and Google PageSpeed
  Insights (site performance), merged and validated before anything downstream runs.
- **AI-generated executive summary** (Google Gemini) constrained to a strict JSON
  schema, grounded only in the supplied numbers — the prompt explicitly forbids
  inventing causes not present in the data.
- **Branded PDF rendering** via a swappable engine: self-hosted Gotenberg by
  default, or a hosted HTML-to-PDF API for clients on n8n Cloud, switched with a
  single config value.
- **Delivery** to Google Drive (dated folder) and Gmail (client + internal notice).
- **Idempotency, retries, input/output validation, centralized error alerting, and
  structured run logging** — see the README for the full list. These aren't
  incidental — they were the majority of the build effort, and the point of the
  exercise: an unattended monthly job has to survive bad data, a down dependency,
  and a malformed AI response without silently producing a broken or duplicate
  report.

## Result

Verified through deliberate testing rather than claimed from production use:

- **Idempotency confirmed** — re-running the workflow for an already-delivered
  client/period skips cleanly (no duplicate email, no duplicate file, logged as
  `skipped`) in under half a second, without touching any external service.
- **Four failure scenarios deliberately induced and confirmed handled correctly**:
  the PDF engine unreachable, non-numeric source data, an ambiguous duplicate data
  row, and (documented, not independently re-verified in this pass) a bad AI
  credential. Each produced exactly one alert email naming the real failing node,
  the real error message, and the run ID — and zero partial or broken output ever
  reached Drive or the client inbox.
- **Two data scenarios tested end-to-end**: an across-the-board decline (every
  metric flagged, sentiment correctly negative) and modest broad improvement with
  nothing crossing the notability threshold (sentiment correctly positive, the
  "no notable changes" case rendering as a clear sentence rather than an empty or
  broken section).
- **Both PDF paths built**; the default (Gotenberg) is fully tested live. The
  hosted-API alternative is complete and documented, with a specific, non-cryptic
  failure message if selected before it's credentialed, but is not itself
  credentialed or live-tested in this reference instance.

What this replaces, for scale: producing one client's monthly report by hand — pulling
metrics, judging what's notable, writing the summary, formatting the document, filing
and sending it — is commonly a 30–45 minute task per client. This pipeline runs that
same work unattended in roughly a minute per client, end to end, with the specific
failure modes above tested and handled rather than assumed away.

## Stack

n8n · Google Sheets · Google PageSpeed Insights · Google Gemini · Gotenberg (with a
built hosted-API alternative) · Google Drive · Gmail · n8n Data Tables (run log)
