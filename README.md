# AI Invoice Automation

Automated invoice processing with n8n, Gmail, OpenRouter, and Google Sheets.

![n8n](https://img.shields.io/badge/n8n-Cloud-EA4B71?style=flat-square)
![OpenRouter](https://img.shields.io/badge/OpenRouter-AI_Extraction-6366F1?style=flat-square)
![OCR.space](https://img.shields.io/badge/OCR.space-OCR-2563EB?style=flat-square)
![Google Sheets](https://img.shields.io/badge/Google_Sheets-Storage-34A853?style=flat-square)

## Visual previews

![Illustrative architecture diagram of the invoice-processing workflow](assets/workflow-architecture.svg)

![Illustrative Google Sheets output using synthetic invoice data](assets/sheets-example.svg)

> **Illustrations, not application screenshots.** These images explain the workflow and the expected spreadsheet result; they were derived from the workflow structure and synthetic example data. For authentic screenshots, capture the imported workflow in your own n8n editor and the sample rows in your own Google Sheet, then replace these illustrations. Remove email addresses, credentials, and real invoice data before publishing screenshots.

## Overview

AI Invoice Automation is an n8n workflow that processes PDF invoices received through Gmail and stores validated invoice data in Google Sheets.

It combines AI-powered extraction, OCR fallback, data validation, and duplicate detection. Invoices that cannot be processed reliably are routed to manual review, with separate error logging for troubleshooting.

## Features

- Automatic Gmail PDF attachment processing
- Structured invoice extraction using OpenRouter
- OCR.space fallback for scanned or incomplete documents
- Validation of required fields, dates, currencies, and amounts
- Duplicate detection across emails and attachments
- Dedicated manual review and error logs
- Configurable processing limits, retries, and error handling

## Workflow

```mermaid
flowchart TD
    A["Gmail PDF Attachment"] --> B{"Already processed?"}
    B -->|Yes| C["Skip"]
    B -->|No| D["OpenRouter Extraction"]
    D --> E{"Valid data?"}
    E -->|Missing fields| F{"OCR attempted?"}
    F -->|No| G["OCR.space"]
    G --> D
    F -->|Yes| H["Manual Review"]
    E -->|Invalid| H
    E -->|Yes| I{"Duplicate invoice?"}
    I -->|Yes| H
    I -->|No| J["Google Sheets"]
```

The workflow checks duplicates at two levels:

- **Attachment level:** Gmail message ID and attachment identifier.
- **Invoice level:** Normalized supplier, invoice number, invoice date, and currency.

Required fields include invoice number, date, supplier, total amount, and currency. Optional fields remain `null` when unavailable.

## Validation & Error Handling

Extracted records are validated before being written to Google Sheets.

| Condition | Handling |
| --- | --- |
| Duplicate attachment | Skip processing |
| Duplicate invoice | Route to manual review |
| Missing required fields | Attempt OCR fallback |
| Invalid JSON or inconsistent amounts | Route to manual review |
| OCR or extraction failure | Log error and route to review |
| Temporary API failure | Retry with bounded backoff |
| Google Sheets failure | Stop processing and log error |
| Invalid, encrypted, or oversized PDF | Route to review |

Financial validation includes checking that net amount plus tax matches the total within a configurable tolerance.

## Example output

A successfully processed fictional invoice would create a row similar to this in the `Invoices` tab:

| Invoice Date | Invoice Number | Supplier | Net Amount | Tax Amount | Total Amount | Currency | Processing Status |
| --- | --- | --- | ---: | ---: | ---: | --- | --- |
| 2026-11-10 | DEMO-2026-0142 | Nordstern Office Services GmbH | 100.00 | 19.00 | 119.00 | EUR | `RECORDED` |

The complete, 17-column example is available in [`examples/invoices.csv`](examples/invoices.csv). All names, message IDs, and transactions are synthetic; **the rows are illustrative, not evidence of an executed workflow**. A successful run writes to `Invoices`; invalid or duplicate invoices go to `Manual_Review`, with technical/audit events in `Error_Log`. The `Confidence` value is the model's self-assessment, not a calibrated probability.

## Prerequisites

- **n8n:** An n8n Cloud or self-hosted instance with support for the nodes in `workflow.json`, including Gmail Trigger (node version 1.3), Gmail (2.2), Google Sheets (4.7), HTTP Request (4.2), and Code (2). **No minimum or tested n8n application version has been established for this export**; verify node availability and import compatibility on your instance before deployment.
- **Google account:** Access to a Gmail inbox and a Google Sheets spreadsheet, with permission to authorize both services through OAuth2 in n8n.
- **OpenRouter account and API key:** Used for PDF parsing and invoice extraction.
- **OCR.space API key:** Needed for the optional OCR fallback when `ocrEnabled` is `true` (the default in `Config`).
- **Potential costs:** n8n hosting and third-party AI/OCR requests may incur charges or have usage limits. Check the current pricing and quotas of each service before enabling automatic processing.

## Setup

### 1. Import

Import [`workflow.json`](workflow.json) into n8n (Workflow menu → **Import from File**).

### 2. Connect services

Configure OAuth2 credentials for Gmail and Google Sheets.

Create separate Header Auth credentials for the HTTP Request nodes:

| Service | Header | Value |
| --- | --- | --- |
| OpenRouter | `Authorization` | `Bearer YOUR_OPENROUTER_KEY` |
| OCR.space | `apikey` | `YOUR_OCR_KEY` |

### 3. Configure Google Sheets

Create a spreadsheet with three tabs:

- `Invoices` — validated invoice records
- `Manual_Review` — invoices requiring attention
- `Error_Log` — technical errors and execution details

<details>
<summary>Required spreadsheet headers</summary>

**Invoices (A–Q)**

```text
Recorded At	Invoice Date	Invoice Number	Supplier	Total Amount	Currency	Net Amount	Tax Amount	Due Date	VAT ID	Description	Sender Email	Gmail Message ID	Attachment ID	Processing Status	Confidence	Duplicate Key
```

**Manual_Review (A–W)**

```text
Recorded At	Invoice Date	Invoice Number	Supplier	Total Amount	Currency	Net Amount	Tax Amount	Due Date	VAT ID	Description	Sender Email	Gmail Message ID	Attachment ID	Processing Status	Confidence	Duplicate Key	Review Reason	File Name	Subject	Received At	Execution ID	Review Key
```

**Error_Log (A–G)**

```text
Timestamp	Workflow Step	Error Category	Technical Reference	Error Message	Status	Execution ID
```

</details>

Header names and column order must match the workflow configuration.

### 4. Configure and activate

1. Set the spreadsheet ID, OpenRouter model, and validation settings in the `Config` node.
2. Configure the Gmail trigger filter.
3. Verify the assigned credentials and spreadsheet mappings.
4. Activate the workflow.

For manual reprocessing, use `Replay - Manual Trigger` with a Gmail API message ID.

## Deployment Notes

- Store API keys in n8n Credentials.
- Remove account-specific identifiers and saved execution data before sharing workflow exports.
- Invoice content is sent to external AI providers and, when OCR is used, to OCR.space.
- PDF processing is limited to 10 MiB per file. OCR requests are limited to 1 MB and 3 pages.
- Avoid concurrent executions when using Google Sheets for duplicate detection. Sheets does not enforce atomic uniqueness.
- Use a database with unique constraints if stronger concurrency guarantees are required.

The workflow includes error handling and manual review paths to prevent incomplete or invalid invoice records from being accepted automatically.

## Repository files

| Path | Purpose |
| --- | --- |
| [`workflow.json`](workflow.json) | Importable n8n workflow export |
| [`examples/invoices.csv`](examples/invoices.csv) | Synthetic example of the complete `Invoices` tab row format |
| [`assets/workflow-architecture.svg`](assets/workflow-architecture.svg) | Illustrative workflow overview |
| [`assets/sheets-example.svg`](assets/sheets-example.svg) | Illustrative spreadsheet preview |
| [`LICENSE`](LICENSE) | MIT License |

## License

MIT — see [`LICENSE`](LICENSE). Review this license before merging if you do not intend to grant reuse and modification rights.

---

*Test data uses fictional companies and synthetic transactions.*
