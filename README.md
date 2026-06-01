# AI Support Ticket Triage Agent

An AI-powered support ticket classification workflow built with n8n Cloud and Gemini 2.5 Flash-Lite.

## Run the Workflow

Use the shared n8n Cloud form:

https://hadar.app.n8n.cloud/form/7ea19070-6c8d-47eb-b593-9cc91cc864dd

The hosted version is already configured and ready to run.
Upload a tickets CSV file and download the classified CSV when processing finishes.

The submitted workflow JSON file can also be imported into the n8n editor in order to review the workflow structure, nodes, logic, validation, retry handling, and fraud-detection pipeline.

---

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

The workflow was designed with reliability, validation, and graceful failure handling in mind.

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
   Sends tickets to Gemini in controlled batches with rate limiting and retry handling.

3. **Validation & Error Handling Layer**  
   Validates Gemini responses and generates safe fallback outputs if parsing or API failures occur.

4. **Fraud Detection Layer**  
   Runs additional regex-based fraud detection rules to identify forged documents, identity theft, impersonation, and suspicious activity.

5. **Export Layer**  
   Generates the final CSV output and uploads it to Google Drive.

## Prompt Engineering
The prompt was intentionally designed to be strict, structured, deterministic, and token-efficient.

I used:
- explicit category definitions
- urgency rules
- fraud escalation rules
- JSON-only output instructions
- Gemini `responseSchema` enforcement

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