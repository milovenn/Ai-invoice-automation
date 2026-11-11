<div align="center">

<img src="./assets/ai-invoice-automation-banner.svg" alt="AI Invoice Automation banner" width="1200">

# AI Invoice Automation

**Process Gmail PDF invoices with n8n and OpenRouter, then record validated fields in Google Sheets.**

[![MIT License](https://img.shields.io/badge/License-MIT-6366f1?style=for-the-badge&labelColor=18181b)](./LICENSE)
[![n8n workflow](https://img.shields.io/badge/n8n-Workflow-6366f1?style=for-the-badge&labelColor=18181b&logo=n8n&logoColor=white)](https://docs.n8n.io/build/manage-workflows/export-and-import/)
[![OpenRouter invoice extraction](https://img.shields.io/badge/OpenRouter-Extraction-6366f1?style=for-the-badge&labelColor=18181b)](#-configuration)

[Quickstart](#-quickstart) · [Demo](#-demo) · [Configuration](#-configuration)

</div>

## ✨ Why AI Invoice Automation

Invoice attachments arrive in email, but review and reporting happen in a spreadsheet. This workflow extracts invoice fields, checks them, and sends exceptions to a separate review queue.

## 🧩 Features

- Process PDF attachments from Gmail and replay a specific message by Gmail API message ID.
- Validate the invoice identity, currency, dates, and amounts before writing a row.
- Skip attachments already recorded in the ledger and send duplicate invoice identities to manual review.
- Retry eligible API failures and use OCR.space as a fallback for eligible PDFs.

## 🎬 Demo

![Illustration of a Google Sheets invoice ledger populated with synthetic invoice rows](./assets/sheets-example.svg)

See the [synthetic invoice CSV](./examples/invoices.csv) for a complete example row.

## 🚀 Quickstart

Use an n8n instance with the Gmail, Google Sheets, HTTP Request, and Code nodes in this export. Prepare a test inbox, a Google spreadsheet, and an OpenRouter API key. Add an OCR.space key if using the fallback.

1. In n8n, select **Import from File** and open [workflow.json](./workflow.json).
2. Create the three tabs and paste the headers from [Configuration](#-configuration) into row 1.
3. Assign Gmail and Google Sheets OAuth2 credentials to their respective nodes. Add Header Auth credentials to the HTTP Request nodes using the headers below.
4. Set `spreadsheetId`, `model`, and `pdfEngine` in **Config**. Set `ocrEnabled` to `false` if you are testing without OCR.space.
5. Email a synthetic PDF invoice to the test inbox. Enter its Gmail API message ID in **Replay - Message ID**, then run **Replay - Manual Trigger**. Look for a row in `Invoices` or check `Manual_Review` and `Error_Log`.

## 📖 Usage

Set the query in **Gmail - New Invoice** to match the messages you want to process, then publish the workflow to enable polling. `Invoices` holds accepted records, `Manual_Review` holds invoices that need a decision, and `Error_Log` tracks technical and review events.

> [!WARNING]
> OpenRouter receives the PDF or extracted OCR text. With fallback enabled, OCR.space also receives the PDF. Use synthetic invoices for setup. Provider calls may incur charges.

## 🏗️ Architecture

![Invoice processing flow with Gmail intake, OpenRouter extraction, validation, OCR fallback and Google Sheets outputs](./assets/workflow-architecture.svg)

The attachment check uses the Gmail message ID and attachment identifier. The business duplicate key combines the normalized supplier, invoice number, invoice date, and currency. Google Sheets has no atomic uniqueness check for concurrent executions.

## ⚙️ Configuration

| Setting | Where | Value |
| --- | --- | --- |
| Spreadsheet ID | **Config** | ID of the spreadsheet with the three output tabs |
| Model and PDF engine | **Config** | Select a model and engine that support the workflow's PDF and structured-output request |
| OpenRouter key | **OpenRouter - Extract Invoice** | Header `Authorization` with value `Bearer YOUR_OPENROUTER_KEY` |
| OCR.space key | **OCR.space - Read PDF** | Header `apikey` with your key when OCR is enabled |
| PDF limits | **Config** | 10 MiB per PDF by default, with a separate 1 MB and 3-page cap for OCR |

<details>
<summary>Spreadsheet headers</summary>

**Invoices**

```text
Recorded At	Invoice Date	Invoice Number	Supplier	Total Amount	Currency	Net Amount	Tax Amount	Due Date	VAT ID	Description	Sender Email	Gmail Message ID	Attachment ID	Processing Status	Confidence	Duplicate Key
```

**Manual_Review**

```text
Recorded At	Invoice Date	Invoice Number	Supplier	Total Amount	Currency	Net Amount	Tax Amount	Due Date	VAT ID	Description	Sender Email	Gmail Message ID	Attachment ID	Processing Status	Confidence	Duplicate Key	Review Reason	File Name	Subject	Received At	Execution ID	Review Key
```

**Error_Log**

```text
Timestamp	Workflow Step	Error Category	Technical Reference	Error Message	Status	Execution ID
```

</details>

## 🛠️ Tech Stack

n8n, JavaScript Code nodes, Gmail, OpenRouter, OCR.space, and Google Sheets.

## 📄 License

MIT License. See [LICENSE](./LICENSE).

---

<div align="center">
  <sub>Released under the <a href="./LICENSE">MIT License</a>. If this saved you time, a ⭐️ helps others find it.</sub>
</div>
