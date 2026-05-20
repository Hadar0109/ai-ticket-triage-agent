# AU10TIX Assignment Progress

Last updated: Wednesday, May 20, 2026 2:54 PM

## Current Status
Step 5 in progress - Testing with ONE ticket to Gemini API

## Completed Milestones
- Step 1: Folders created (input/, output/, prompts/, n8n/, docs/)
- Step 2: Prompt files created with strengthened fraud detection logic
- Step 3: n8n running in Docker with workspace mounted at /data
- Step 4: CSV parsing workflow working - returns 50 items ✓
- Step 5: Added Gemini API test nodes (Limit to 1, Call Gemini API, Parse Response)

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
- Fixed n8n CSV parsing: Used `this.helpers.getBinaryDataBuffer()` instead of manual Buffer decoding
- Parse CSV now returns 50 items correctly ✓
- Created .env file with GEMINI_API_KEY placeholder

## Next Step
Step 5: Test the workflow
1. Add your real Gemini API key to .env file (get from https://aistudio.google.com/app/apikey)
2. Restart Docker container to load environment variables: `docker-compose restart`
3. Import updated workflow into n8n
4. Execute and verify one ticket gets classified by Gemini
