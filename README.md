# Translation-automatization

End-to-end automation of sworn document translation orders across 4 linked n8n workflows. From the client's form submission to delivery of the certified PDF — no manual steps except the final manager approval.

> ⚠️ This README is kept local. Only screenshots of the workflows are pushed to GitHub, as the JSON files contain internal IDs, webhook URLs, and credential references.

---

## Architecture

```
[Client form]      ──►  [1. Client fills form]       ──►  creates order, Drive folder, emails translator
                                                          │
[Translator form]  ──►  [2. Cost estimation]         ──►  Stripe Checkout + email to client with payment link
                                                          │
[Stripe webhook]   ──►  [3. Payment trigger]         ──►  status "paid", unlocks translator to start
                                                          │
[Gmail from SATEO] ──►  [4. Sending translation]     ──►  manager approval → email client + cleanup
```

**State machine** on top of Google Sheets (`orderId` is the primary key):
`awaiting_price` → `awaiting_payment` → `paid` → `Ready`

---

## Workflows

### 1 — Client fills form (`7X9kFzOwWjo5akP1`)
- **Trigger:** public n8n form (`Client form`, webhook `ab47042d-...`) with Name, Email, Documents (file upload), Source/Target language, Additional comments.
- **What it does:**
  1. JS node generates `orderId` in the form `ORD-{yyyyLLdd-HHmmss}-{safeEmail}`.
  2. Creates a Google Drive folder `TranslationOrders/{clientName} - {orderId}`.
  3. Uploads the client's files into that folder.
  4. OpenAI (`gpt-4o-mini`) detects the language of the client comment and translates it into English (JSON: `detectedLanguage`, `englishVersion`).
  5. Emails the translator (`valeria@fenchell-team.com`, `office@sateo.net`) with attachments and a link to the quotation form.
  6. Appends a row to Google Sheets with status `awaiting_price`.
- **Redirect after submit:** `https://www.fenchell.com/en/translation-request-accepted/`

### 2 — Cost estimation (`WB2IkJg5cdDH5ioI`)
- **Trigger:** translator-facing form (webhook `b0d91298-...`) — takes `Total price (EUR)` and `Estimated turnaround time`.
- **Query params** passed from the email: `orderId`, `clientEmail`.
- **What it does:**
  1. Adds +20 EUR to the translator's price (Fenchell margin), converts to cents for Stripe.
  2. Creates a Stripe Checkout Session (`mode=payment`, automatic tax, invoice creation, `tax_id_collection`).
  3. Emails the client the payment link and estimated turnaround time.
  4. Updates the Sheets status to `awaiting_payment`.

### 3 — Payment trigger (`RLysuiZ4WUMsuXmh`)
- **Trigger:** Stripe webhook — `checkout.session.completed`.
- **What it does:**
  1. Extracts `orderId` from `metadata`, filters out unrelated events.
  2. Updates Sheets: `status = paid`, `paidAt = now()`.
  3. Replies in the original Gmail thread (using `threadId_translator`) to tell the translator: "payment received, you may begin the translation".

### 4 — Sending translation to client (`SO2n02zq1nrCmJWE`)
- **Trigger:** Gmail poller (every 15 min, Mon-Fri 9-18) — emails from `office@sateo.net` with attachments.
- **What it does:**
  1. Extracts `orderId` from the email subject (`/ORD-[\w.-]+/`).
  2. Uploads the received PDFs into the order's Drive folder.
  3. Sends the manager (`valeria@...`) a **Send-and-Wait** email with approve/reject buttons.
  4. On **approve** → forwards the final PDFs to the client, sets status `Ready`, deletes the Drive folder.
  5. On **reject** → the chain halts (manual handling required).

---

## External dependencies

| Service | Usage | n8n Credential |
|---|---|---|
| **Google Sheets** | Orders DB, spreadsheet `1ZY78kZdRs6b-QsXunBFfLC1rPOPqLCHRnXJcCFFOdd8` / `Sheet1` | `Conformity Google Sheets ACTION NODES` |
| **Google Drive** | Storage for source docs and translations (root folder `1smHpaCPoxQZ7NPptkJ__l72MF8XYPSHB`) | `EUROTRADE Invoice Access` |
| **Gmail** | Communication with translator and client | `Conformity Gmail` |
| **Stripe** | Checkout Sessions + webhook | `Stripe account`, `Header Auth account 2` |
| **OpenAI** | Normalising client comments to English | `OpenAi account` |

---

## `Sheet1` schema

| Column | Written by WF | Description |
|---|---|---|
| `orderId` | 1 | Primary key, `ORD-YYYYMMDD-HHMMSS-email` |
| `clientEmail`, `clientName`, `comment` | 1 | From the form |
| `documentLanguage`, `targetLanguage` | 1 | Language pair |
| `folderId`, `folderLink` | 1 | Order folder in Drive |
| `threadId_translator` | 1 | Gmail thread ID — used by WF3 to reply in-thread |
| `createdAt` | 1 | ISO timestamp |
| `status` | 1→2→3→4 | State machine |
| `paidAt` | 3 | Moment payment succeeded |

---

## Setup

1. Import the 4 JSON files into n8n (`Workflows → Import from file`).
2. Configure the credentials from the table above.
3. Create a Google Sheet with the columns listed, and put its `documentId` into workflows 1/2/3/4.
4. Create a root folder in Drive, put its ID into workflow 1 (`Create folder` node).
5. Configure the Stripe webhook on `n8n.fenchell.com` (the Stripe Trigger in WF3 provisions it automatically).
6. Update the quotation-form link inside WF1's translator email if the WF2 webhook changes.
7. `errorWorkflow: To7O7miuQAurOzoX` — ensure the error workflow exists, or remove the setting.

---

## Important details

- **Timezone:** `Europe/Sofia` across all workflows.
- **Binary mode:** `separate` — files travel separately from JSON (matters for large PDFs).
- **Fenchell margin:** +20 EUR added on top of the translator's price (hard-coded in WF2's `Code` node).
- **Gmail polling:** business hours Mon-Fri 9-18, every 15 minutes — emails arriving overnight are picked up in the morning.
- **Approval flow in WF4:** uses Gmail `sendAndWait` — the manager clicks Approve/Reject straight from the inbox; n8n waits on the callback.
- **Branding/CSS:** WF1's form ships custom CSS (classic heritage palette — parchment + bottle green) for the Fenchell look.

---

## Failure points to watch

1. **OpenAI quota** — if it fails, WF1 won't send the translator email (Merge1 waits for 3 inputs).
2. **Stripe webhook signature** — if the secret rotates, WF3 stops firing.
3. **Gmail OAuth expiry** — 3 of 4 workflows depend on a single credential.
4. **Google Drive quota** — folders are not cleaned up on reject (only on approve in WF4).
5. **`threadId_translator`** — if the original email in WF1 failed to send, WF3 can't reply into the thread.
