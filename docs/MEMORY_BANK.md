# AU10TIX Assignment Progress

Last updated: Tuesday, May 19, 2026 6:09 PM

## Current Status
Step 4 complete - CSV parsing workflow created and ready for import into n8n.

## Completed Milestones
- Step 1: Folders created (input/, output/, prompts/, n8n/, docs/)
- Step 2: Prompt files created with strengthened fraud detection logic
- Step 3: n8n running in Docker with workspace mounted at /data
- Step 4: CSV ingestion workflow created (n8n/ticket_triage_csv_ingestion.json)

## Key Decisions
- Using Docker Compose for n8n (consistent environment)
- API key in .env file (security best practice, never committed)
- Environment variable {{$env.GEMINI_API_KEY}} in workflow (no hardcoded secrets)
- Using gemini-2.5-flash-lite model
- Dual-layer fraud detection (LLM + keywords)
- Conservative fallback (flag for review on failure)
- Strengthened category distinctions (Document Issue vs Fraud Concern)
- Enhanced fraud keywords with user behavior phrases ("not me", "wasn't me", etc.)
- Manual trigger for workflow execution (easier testing and control)
- Professional naming convention (e.g., "Ticket Triage - CSV Ingestion" vs "Step 4")

## Issues Resolved
- Removed weak fraud keywords (e.g., "suspicious verification", "same selfie")
- Added clearer distinction between legitimate document issues and fraud concerns
- Added edge case handling for empty/vague tickets
- Fixed n8n file access: replaced Read Binary File node with Code node (bypasses security restrictions)
- Code node uses Node.js fs module to read CSV directly from `/files/input/tickets.csv`

## Next Step
Step 5: Test with ONE ticket to Gemini
- Use {{$env.GEMINI_API_KEY}} in HTTP Request node
- Model: gemini-2.5-flash-lite
- API key is now in .env file (secure)
