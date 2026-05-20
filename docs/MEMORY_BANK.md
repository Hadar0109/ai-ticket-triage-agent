# AU10TIX Assignment Progress

Last updated: Wednesday, May 20, 2026 2:54 PM

## Current Status
Step 5 adapted for n8n Cloud - Testing with ONE ticket to Gemini API

## Completed Milestones
- Step 1: Folders created (input/, output/, prompts/, n8n/, docs/)
- Step 2: Prompt files created with strengthened fraud detection logic
- Step 3: n8n setup (switched from Docker to n8n Cloud for better credential management)
- Step 4: CSV parsing workflow working - returns 50 items ✓
- Step 5: Created cloud-optimized workflow (ticket_triage_cloud.json)
  - CSV data embedded in Code node (no file system dependency)
  - Using n8n credential system instead of environment variables
  - Ready to test with Gemini API

## Key Decisions
- **Switched to n8n Cloud** (better credential management, no Docker complexity, more production-like)
- CSV data embedded in Code node (works in cloud without file system access)
- Using n8n's secure credential system for Gemini API key (instead of environment variables)
- Using gemini-2.5-flash-lite model
- Dual-layer fraud detection (LLM + keywords)
- Conservative fallback (flag for review on failure)
- Strengthened category distinctions (Document Issue vs Fraud Concern)
- Enhanced fraud keywords with user behavior phrases ("not me", "wasn't me", etc.)
- Manual trigger for workflow execution (easier testing and control)
- Professional naming convention (e.g., "Ticket Triage - Cloud Version")

## Issues Resolved
- Removed weak fraud keywords (e.g., "suspicious verification", "same selfie")
- Added clearer distinction between legitimate document issues and fraud concerns
- Added edge case handling for empty/vague tickets
- Fixed n8n CSV parsing: Used `this.helpers.getBinaryDataBuffer()` instead of manual Buffer decoding
- Parse CSV now returns 50 items correctly ✓
- Switched from Docker to n8n Cloud (solved environment variable access issues)
- Embedded CSV data in Code node (no file system dependencies)
- Migrated to n8n credential system (better security than env vars)

## Next Step
Step 5: Test the workflow in n8n Cloud
1. Sign up for n8n Cloud (https://n8n.io/cloud/)
2. Create Gemini API credential (using "HTTP Query Auth")
3. Import `n8n/ticket_triage_cloud.json`
4. Link credential to "Call Gemini API" node
5. Execute and verify one ticket gets classified

See: `docs/N8N_CLOUD_SETUP.md` for detailed instructions
