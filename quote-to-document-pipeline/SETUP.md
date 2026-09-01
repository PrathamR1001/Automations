# Setup Guide — Quote to Document Pipeline

This guide assumes you have never seen this workflow before. It covers: what to
import, what credentials to create, the Airtable structure required, both PDF
rendering options, how to connect a real e-signature provider, which node to edit
for a new client, and how to confirm your first run worked.

The system is five n8n workflows, no Data Table beyond what's noted below:

| Workflow | Role |
|---|---|
| **Quote to Document Pipeline** | Main orchestrator — webhook + manual trigger, validation, idempotency, AI quote draft, pricing, delivery, follow-up |
| **QTD - Render Quote PDF** | Sub-workflow — builds the branded HTML quote and converts it to PDF |
| **QTD - Log Run** | Sub-workflow — appends one row per event to the run log Data Table |
| **QTD - Error Alert** | Error handler — set as the Error Workflow on the other four |
| **QTD - Signature Callback Receiver** | Placeholder webhook for a future e-sign provider callback |

---

## 0. Importing the workflow files

The five JSON files in [workflows/](workflows/) are sanitized exports — credential
IDs, the Airtable base/table IDs, the run-log Data Table ID, and email addresses
have all been replaced with placeholders. After importing all five:

1. **Create the QTD_RunLog Data Table** — columns `runId` (string), `timestamp`
   (date), `prospectName` (string), `quoteTotal` (number), `status` (string),
   `durationMs` (number), `error` (string). Then open **Upsert Run Log Row** (in
   QTD - Log Run) and point `dataTableId` at the one you just created.
2. **Re-link the sub-workflow references.** Execute Workflow nodes store an ID
   that doesn't carry over on import. Re-pick the correct workflow in each of:
   **Render PDF** and **Log Skipped Run** / **Log Successful Run** / **Log Reminder
   Sent** (all in Quote to Document Pipeline) → point at QTD - Render Quote PDF /
   QTD - Log Run respectively; **Log Failed Run** (in QTD - Error Alert) and
   **Log Signed Event** (in QTD - Signature Callback Receiver) → QTD - Log Run.
3. **Set the Error Workflow.** In Quote to Document Pipeline, QTD - Render Quote
   PDF, and QTD - Signature Callback Receiver's Settings, set **Error Workflow** to
   QTD - Error Alert. This is a workflow-level setting, not a node — easy to miss
   after import.
4. Continue with credentials, Airtable, and Config below.

---

## 1. Credentials to create

| # | Credential | Type | Used by |
|---|---|---|---|
| 1 | Google Gemini(PaLM) Api account | Google Gemini (PaLM) API | Drafts the quote scope and line items |
| 2 | Google Drive account | Google Drive OAuth2 | Uploads the PDF, needs full Drive access (not the restricted `drive.file` scope) |
| 3 | Gmail account | Gmail OAuth2 | Sends the quote to the prospect, the internal notice, the reminder, and (from QTD - Error Alert) the failure alert |
| 4 | Airtable Personal Access Token account | Airtable Token API | Idempotency check, record create/update, follow-up re-check, signature update |

The Airtable PAT needs, at minimum, the scopes `data.records:read`,
`data.records:write`, and `schema.bases:read`, **and** the specific base added
under the token's Access list — a PAT with the right scopes but no base granted
still fails every call with a generic 403.

---

## 2. Airtable structure

Create a base with a table containing exactly these fields:

| Field | Type |
|---|---|
| `EnquiryId` | Single line text (primary) |
| `ProspectName` | Single line text |
| `Email` | Single line text |
| `Phone` | Single line text |
| `PropertyAddress` | Long text |
| `PropertySizeSqFt` | Number |
| `RequirementsText` | Long text |
| `ScopeSummary` | Long text |
| `LineItemsJSON` | Long text |
| `ClarifyingAssumptionsJSON` | Long text |
| `QuoteTotal` | Number |
| `PDFLink` | URL |
| `Status` | Single select — options `sent`, `signed` |
| `CreatedAt` | Date |
| `SignedAt` | Date |
| `ReminderSentAt` | Date |

Getting a field type wrong here is the single most common setup mistake — e.g. a
`Single Collaborator` field where `Email` should be plain text, or a `Single
Select` field where `CreatedAt` should be a date, will cause a specific, readable
error on the affected node rather than a silent failure. If you see an Airtable
422 error naming a field, re-check that field's type against this table first.

---

## 3. PDF rendering: two options

Identical mechanism to the Client Report Generator demo — switching is a single
Config value (`pdfEngine`, `"gotenberg"` or `"api"`), nothing else changes.

### Option A — Self-hosted Gotenberg (default)

No per-document cost, HTML never leaves your infrastructure, requires Docker.
Does not work on n8n Cloud.

```bash
docker run -d --name gotenberg --network <your-n8n-docker-network> gotenberg/gotenberg:8
```

`Config.gotenbergUrl` defaults to `http://gotenberg:3000` (the container name on
the shared Docker network). No credential needed — the call is unauthenticated by
design, since it stays inside your network.

### Option B — Hosted API (PDFShift)

Works anywhere including n8n Cloud, no infrastructure, small per-document cost
after a 50-document/month free tier.

1. Sign up free at pdfshift.io — no card required.
2. Create a **Header Auth** credential in n8n: Header Name `X-API-Key`, Header
   Value your PDFShift key.
3. Assign that credential to the **Convert HTML to PDF (PDFShift API)** node (in
   QTD - Render Quote PDF).
4. Set `Config.pdfEngine` to `"api"`.

If `pdfEngine` is set to `"api"` without step 2–3 done, the run fails cleanly:
QTD - Error Alert recognizes the missing-credential case and sends a plain-language
message telling you to add it, rather than a cryptic auth error. No partial PDF is
produced either way.

---

## 4. Connecting a real e-signature provider

The pipeline currently **stubs** e-signature — no live DocuSign/Dropbox Sign
account is wired up, by design, so this demo never sends a real signature request.
To connect one:

1. In the provider's dashboard, create the "send for signature" API call you want
   (send the rendered PDF, or re-render it from the same HTML this pipeline
   already builds).
2. Enable the **Send for E-Signature (PLACEHOLDER)** node in Quote to Document
   Pipeline and replace it with that real API call — it currently sits disabled
   exactly where that call belongs in the flow (after the Airtable record is
   created, before the internal notice).
3. Point the provider's webhook/callback configuration at **QTD - Signature
   Callback Receiver**'s webhook URL (path `/quote-signed-callback`).
4. Open **Normalize Signature Payload** in that workflow and adjust the field
   mapping to match the provider's real callback payload shape — the current code
   assumes an illustrative `{enquiryId, signedAt}` shape, which is a placeholder,
   not any real provider's actual schema.

Everything downstream of that (finding the Airtable record by `enquiryId`, marking
it `signed`, logging the event, and the main pipeline's follow-up check correctly
seeing it as signed and skipping the reminder) already works and needs no changes.

---

## 5. Config values to change for a new client

Open the **Config** node (in Quote to Document Pipeline) and edit its fields
directly:

| Field | What it controls |
|---|---|
| `companyName` / `brandName` | Shown on the quote PDF header and in email copy |
| `pricingRatesJson` | JSON string mapping category → base rate per unit (`panels`, `inverter`, `mounting`, `electrical`, `permit`, `labor`, `other`) |
| `sizeTierThresholdSqFt` / `sizeTierMultiplier` | Above this property size, rates are multiplied by this factor |
| `discountThresholdTotal` / `discountPct` | Above this subtotal, this discount percentage is applied |
| `airtableBaseId` / `airtableTableId` | Where quote records are stored |
| `driveParentFolderId` | Where quote PDFs are uploaded (`root` = My Drive) |
| `internalNotifyEmail` | Who receives the internal "quote sent" notice |
| `followUpDelayMinutes` | How long to wait before checking whether a reminder is needed — set to something like `4320` (3 days) in production |
| `pdfEngine` / `gotenbergUrl` / `pdfApiEndpoint` | See section 3 |

The AI prompt (in **Generate Quote Content (Gemini)**) is written for a solar
installer specifically — the category list, the system message's framing, and the
sample enquiry are all solar-specific. Adapting this pipeline to a different
service business means rewriting that prompt and the `pricingRatesJson` categories
to match, not just editing Config.

---

## 6. Verifying a first successful run

1. Complete sections 1–5 above.
2. POST a sample enquiry to the webhook:
   ```bash
   curl -X POST http://localhost:5678/webhook/quote-enquiry \
     -H "Content-Type: application/json" \
     -d '{
       "enquiryId": "ENQ-TEST-001",
       "name": "Test Prospect",
       "email": "you@example.com",
       "phone": "+1-555-0100",
       "propertyAddress": "1 Test St",
       "propertySizeSqFt": 2000,
       "requirementsText": "Solar panels to reduce our monthly electric bill, standard asphalt roof, no battery needed."
     }'
   ```
   (Or use the **Manual Demo Trigger**, which feeds a hardcoded sample payload
   through the same path.)
3. Watch it complete. A successful run:
   - Creates one Airtable record with `Status: sent`.
   - Delivers the PDF quote by email to the address you supplied, plus an internal
     notice to `internalNotifyEmail`.
   - Adds a `success` row to QTD_RunLog.
   - After `followUpDelayMinutes` elapses, re-checks the Airtable status; since
     it's still `sent`, sends a reminder email, sets `ReminderSentAt`, and logs a
     `reminder_sent` row.
4. Run it again immediately with the **same** `enquiryId`. It should skip cleanly
   — no second email, no second Airtable record — logging a `skipped` row instead.
5. To see the failure path work, POST a payload missing `email`, or stop Gotenberg
   first — confirm an alert email arrives naming the failing node, the error, and
   the run ID, and that QTD_RunLog gets a `failed` row. Undo/restart afterward.
