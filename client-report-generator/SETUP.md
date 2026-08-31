# Setup Guide — Client Report Generator

This guide assumes you have never seen this workflow before. It covers: what to
install, what credentials to create, what to put in the Google Sheet, which single
node to edit for a new client, and how to confirm your first run worked.

The system is four n8n workflows plus one Data Table:

| Workflow | Role |
|---|---|
| **Client Report Generator** | Main orchestrator — schedule/manual trigger, data fetch, validation, AI summary, delivery |
| **CRG - Render Report PDF** | Sub-workflow — builds the branded HTML and converts it to PDF |
| **CRG - Log Run** | Sub-workflow — writes one row per run to the run log |
| **CRG - Error Alert** | Error handler — set as the "Error Workflow" on the other three |
| **CRG_RunLog** (Data Table) | Structured log: run ID, timestamp, client, period, status, duration, error |

---

## 0. Importing the workflow files

The four JSON files in [workflows/](workflows/) are sanitized exports — credential
IDs, the sample Google Sheet ID, the run-log Data Table ID, and email addresses have
all been replaced with placeholders. After importing all four into your n8n
instance:

1. **Create the CRG_RunLog Data Table** (see column list in the original build, or
   just recreate: `runId` string, `timestamp` date, `clientName` string, `period`
   string, `status` string, `durationMs` number, `error` string). Then open
   **Check Existing Report** (in Client Report Generator) and **Upsert Run Log Row**
   (in CRG - Log Run) and point their `dataTableId` at the one you just created —
   the placeholder `YOUR_RUNLOG_DATA_TABLE_ID` won't resolve on its own.
2. **Re-link the sub-workflow references.** n8n's Execute Workflow nodes store a
   workflow ID, which doesn't carry over on import. Open each of these and re-pick
   the correct workflow from the dropdown: **Render PDF** (in Client Report
   Generator) → CRG - Render Report PDF; **Log Skipped Run** and **Log Successful
   Run** (in Client Report Generator) → CRG - Log Run; **Log Failed Run** (in
   CRG - Error Alert) → CRG - Log Run.
3. **Set the Error Workflow.** In both Client Report Generator's and CRG - Render
   Report PDF's workflow Settings, set **Error Workflow** to CRG - Error Alert. This
   is what routes any failure to the alert email — it's a workflow-level setting,
   not a node, so it doesn't show up on the canvas and is easy to miss after import.
4. Continue with credentials and Config below.

---

## 1. Credentials to create

Create these in n8n's **Credentials** store (Settings → Credentials). Never enter
API keys directly into a node — every node in this build references a credential
by name.

| # | Credential | Type | Used by | Scope needed |
|---|---|---|---|---|
| 1 | Google Sheets account | Google Sheets OAuth2 | Reads the metrics sheet | Read access to the target spreadsheet |
| 2 | Google Drive account | Google Drive OAuth2 | Creates the dated report folder, uploads the PDF | **Full Drive access** (`https://www.googleapis.com/auth/drive`), not the restricted "see only files this app created" (`drive.file`) option — the restricted scope blocks folder search/listing and will throw `"granted scopes do not give access to all of the requested spaces"` |
| 3 | Gmail account | Gmail OAuth2 | Sends the client email, the internal notice, and (from CRG - Error Alert) the failure alert | Send-mail scope (default Gmail OAuth2 scope in n8n) |
| 4 | Google Gemini(PaLM) Api account | Google Gemini (PaLM) API | Generates the executive summary | A Gemini API key from Google AI Studio |
| 5 | PageSpeed Query Auth *(optional)* | Generic — Query Auth | PageSpeed Insights external call | A free PageSpeed Insights API key from Google Cloud Console — the call works keyless at low volume, but the **unauthenticated quota is shared globally and exhausts fast** in testing. A free key removes that risk. |

You can use the same Google account for credentials 1–3.

---

## 2. PDF rendering: two options

The workflow can produce the PDF two ways. **Switching between them is a single
value**: the `pdfEngine` field on the **Config** node (first node after the triggers
in "Client Report Generator"). Set it to `"gotenberg"` or `"api"`. Nothing else
needs to change — the HTML report is built identically either way; only the final
conversion step differs.

### Option A — Self-hosted Gotenberg (default)

- **Cost:** none per document.
- **Data handling:** the HTML never leaves your infrastructure — Gotenberg runs in
  your own Docker network.
- **Requirement:** Docker. Works on self-hosted n8n. **Does not work on n8n Cloud**
  (there's no container for n8n Cloud to reach).
- **Best for:** self-hosted n8n instances, and any client sensitive about where
  their data is processed.

Setup:
```bash
docker run -d --name gotenberg --network <your-n8n-docker-network> gotenberg/gotenberg:8
```
Find your n8n container's network with `docker inspect <n8n-container> --format '{{json .NetworkSettings.Networks}}'`.
No port publish is required — n8n reaches Gotenberg by container name on the shared
network at `http://gotenberg:3000` (this is `Config.gotenbergUrl`; change it if you
name the container differently).

No credential needed for this path — the Gotenberg call is unauthenticated, by
design, since it's inside your own network.

### Option B — Hosted API (PDFShift)

- **Cost:** per document, after a free tier (50 documents/month, no card required
  to start; paid beyond that).
- **Data handling:** your rendered HTML is sent to PDFShift's servers to be
  converted. It's a third party in the document's path.
- **Requirement:** none — works anywhere, including n8n Cloud.
- **Best for:** clients on n8n Cloud, or anyone who'd rather pay per document than
  run infrastructure.

Setup:
1. Sign up free at [pdfshift.io](https://pdfshift.io) — no card required.
2. Copy your API key from the PDFShift dashboard.
3. In n8n, create a credential of type **Header Auth** (n8n's generic
   `httpHeaderAuth` type):
   - **Header Name:** `X-API-Key`
   - **Header Value:** your PDFShift API key
   - Name it "PDFShift API" (or reassign the credential dropdown on the
     "Convert HTML to PDF (PDFShift API)" node in **CRG - Render Report PDF** to
     whatever you name it).
4. Set `Config.pdfEngine` to `"api"`.

**If you set `pdfEngine` to `"api"` without doing step 3 first:** the run fails
cleanly. CRG - Error Alert recognizes this specific failure and sends an alert that
says plainly that the PDFShift credential is missing and how to add it — not a raw
n8n "invalid credential" error. No partial or broken PDF is produced or delivered
in this case.

---

## 3. Google Sheet structure

Create a Google Sheet with a tab named **Metrics** (or whatever you set
`Config.sheetTab` to) with exactly these columns, one row per client per month:

| Period | ClientName | WebsiteURL | Website Sessions | Leads Generated | Conversion Rate (%) | Ad Spend (USD) | New Backlinks | Revenue Attributed (USD) |
|---|---|---|---|---|---|---|---|---|
| 2026-07 | Rivertown Fitness Co | https://example.com | 5200 | 79 | 3.4 | 1840 | 6 | 11200 |

Notes:
- `Period` must be `YYYY-MM` and match a real calendar month.
- Keep at least two consecutive months present at all times (the workflow always
  compares "most recently complete month" against the month before it) — including
  a row for the current, still-in-progress month is fine; the workflow ignores it
  until it becomes the most recently *complete* month.
- Never put two rows with the same `Period` for the same client — the workflow
  detects this and fails the run with a clear "ambiguous, expected exactly one"
  message rather than guessing which row to use.
- All six metric columns must be numeric. A non-numeric or missing value fails
  validation before any external call is made (Drive, email, AI) — no broken report
  is ever generated from bad input.

---

## 4. Config values to change for a new client

Open the **Config** node (first node in "Client Report Generator", right after the
triggers) and edit its fields directly — nothing else in the workflow needs to
change:

| Field | What it controls |
|---|---|
| `clientName` | Must exactly match the `ClientName` value in the sheet |
| `agencyName` | Your agency/brand name, shown in the report header |
| `reportTitleFormat` | The subtitle line under the client name in the report |
| `thresholdPct` | % change that flags a metric as notable |
| `websiteUrl` | The site PageSpeed Insights checks |
| `sheetId` / `sheetTab` | The Google Sheet and tab to read |
| `driveParentFolderId` | Where dated report folders are created (`root` = My Drive) |
| `clientRecipientEmail` | Who receives the report PDF |
| `internalNotifyEmail` | Who receives the internal "run complete" notice |
| `pdfEngine` | `"gotenberg"` or `"api"` — see section 2 |
| `gotenbergUrl` / `pdfApiEndpoint` | Only relevant to their respective engine |

---

## 5. Verifying a first successful run

1. Complete sections 1–4 above.
2. Open **Client Report Generator** in n8n and click the **Manual Demo Trigger**
   node, then "Execute workflow" (or use "Test workflow").
3. Watch it complete. A successful run:
   - Ends at **Log Successful Run** with no red (error) nodes anywhere on the canvas.
   - Creates a `Reports - <period>` folder in the configured Drive location
     containing one PDF.
   - Delivers two emails: the report to `clientRecipientEmail`, and a short
     completion notice to `internalNotifyEmail`.
   - Adds one row to **CRG_RunLog** with `status = success`.
4. Run it again immediately for the same period. It should skip cleanly — no
   second email, no second file — and add a `status = skipped` row to
   **CRG_RunLog** instead. This confirms the idempotency guard is working.
5. To see the error path work, temporarily break something on purpose — stop
   Gotenberg, or put a non-numeric value in the sheet — and confirm an alert email
   arrives naming the failing node, the error, and the run ID, and that
   **CRG_RunLog** gets a `status = failed` row. Undo the change afterward.
