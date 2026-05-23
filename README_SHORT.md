AI Support Ticket Triage Agen- An AI-powered support ticket classification workflow built with **n8n Cloud** and **Gemini 2.5 Flash-Lite** for the AU10TIX AI Agent & Automation Builder home assignment.

I built an automated workflow that:
- receives a CSV file containing support tickets
- classifies each ticket by urgency and category
- generates a one-sentence summary
- sets a `flag_for_review` when fraud or security signals are detected
- exports the final classified results as a CSV file

The system combines:
- LLM-based classification using Gemini
- additional rule-based fraud detection
- structured validation and fallback handling

The workflow was designed to be reliable even when the LLM returns malformed output or API failures occur.

## Tools & Technologies
- n8n Cloud
- Google Gemini 2.5 Flash-Lite
- JavaScript (inside n8n Code nodes)
- Google Drive API

## Workflow Design
The workflow is divided into several layers:
1. **Input Layer**  
   Receives and parses the uploaded CSV file.
2. **LLM Processing Layer**  
   Sends tickets to Gemini in controlled batches (4 per batch) with rate limiting and retry handling.
3. **Validation & Error Handling Layer**  
   Validates Gemini responses and generates safe fallback outputs if parsing or API failures occur.
4. **Fraud Detection Layer**  
   Runs additional regex-based fraud detection rules to identify forged documents, identity theft, impersonation, and suspicious activity.
5. **Export Layer**  
   Generates the final CSV output and uploads it to Google Drive.

## Prompt Engineering
The prompt was intentionally designed to be strict, structured, deterministic, and token-efficient.

I used explicit category definitions, urgency rules, fraud escalation rules, JSON-only output instructions, Gemini `responseSchema` enforcement.
This improves output consistency and prevents malformed responses.

## Edge Cases & Reliability
The workflow handles several failure scenarios:
- Empty tickets are skipped and not sent to Gemini
- API rate limits are handled with batching and retry delays
- Invalid JSON responses are safely parsed and validated
- Missing classifications generate fallback outputs
- API failures and timeouts do not crash the workflow
- CSV injection protection is applied during export

The workflow never blindly trusts the LLM output.

## Security Considerations
- API keys are stored securely in n8n credentials
- No secrets are hardcoded in the workflow export
- Only required ticket fields are sent to the LLM
- All LLM outputs are validated before use

## Run with Hosted Cloud Version
Use the shared n8n Cloud form:
**https://hadar.app.n8n.cloud/form/7ea19070-6c8d-47eb-b593-9cc91cc864dd**

The workflow is already connected to Gemini API, Google Drive, and workflow credentials.
Upload a tickets CSV and download the classified CSV when processing finishes.

## Import and run yourself
Requirements:
- [n8n Cloud](https://n8n.io/cloud/) account (or local n8n via Docker)
- [Gemini API key](https://aistudio.google.com/app/apikey)
- Google Drive connection *(optional, for cloud upload)*

#### Setup Steps
1. Import workflow
In n8n: **Menu (⋮) → Import from File** → select the submitted workflow file

2. Configure Gemini API credential
Create an **HTTP Header Auth** credential in n8n:
| Field | Value |
|-------|-------|
| Header name | `x-goog-api-key` |
| Header value | Your Gemini API key |
Link it to the **Gemini Batch Call** and **Gemini Batch Retry Call** nodes.

3. Configure Google Drive credential *(optional)*
Connect Google Drive OAuth in n8n if you add upload/share nodes after CSV generation.

4. Run the workflow
- Activate the workflow and open the **Form Trigger** URL
- Upload a CSV file with columns: `ticket_id`, `created_at`, `customer_name`, `customer_email`, `subject`, `body`, `status`
- Wait for processing to complete
- Download **`classified_tickets.csv`** from the **Generate CSV File** node output

