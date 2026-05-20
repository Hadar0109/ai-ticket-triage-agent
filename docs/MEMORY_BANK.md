# AU10TIX Assignment Progress

Last updated: Wednesday, May 20, 2026 7:48 PM

## Current Status
✅ **Stage 2 Complete** - Fraud Validation Layer implemented and tested
🔄 **Testing Phase** - Running with 1 ticket to conserve API quota

## Completed Milestones
- **Step 1**: Folders created (input/, output/, prompts/, n8n/, docs/)
- **Step 2**: Prompt files created with strengthened fraud detection logic
- **Step 3**: n8n setup (switched from Docker to n8n Cloud for better credential management)
- **Step 4**: CSV parsing workflow working - returns 50 items ✓
- **Step 5**: Single-ticket classification tested successfully ✓
  - CSV data embedded in Code node (no file system dependency)
  - Using n8n credential system instead of environment variables
  - Gemini API integration working correctly
- **Stage 1**: CSV Output implemented ✓
  - Format CSV Output node added (position 1560)
  - Proper CSV escaping (handles commas, quotes, newlines)
  - Outputs all 11 fields (original + classification)
  - Scalable design (works for 1 or 50 tickets without changes)
- **Stage 2**: Independent Fraud Validation Layer ✓
  - Rule-based keyword detection (15 fraud indicators)
  - Overrides AI classification when fraud detected
  - Sets category = "Fraud Concern" and flag_for_review = true
  - Smart urgency elevation (High only for active threats)
  - Position: Between Parse Response and Format CSV Output

## Workflow Architecture (6 Nodes)
```
Manual Trigger 
  → Load and Parse CSV (50 tickets embedded)
    → Limit to 1 (testing mode - preserves API quota)
      → Call Gemini API (gemini-2.5-flash-lite)
        → Parse Response (extracts JSON classification)
          → Fraud Validation Layer (rule-based override)
            → Format CSV Output (exports results)
```

## Key Decisions
- **Switched to n8n Cloud** (better credential management, no Docker complexity, more production-like)
- **CSV data embedded in Code node** (works in cloud without file system access)
- **Using n8n's secure credential system** for Gemini API key (instead of environment variables)
- **Using gemini-2.5-flash-lite model** (fast, cost-effective)
- **Dual-layer fraud detection** (AI + deterministic keyword matching)
- **Content-only urgency classification** (removed time-based escalation for historical data)
- **Conservative fallback** (flag for review on failure)
- **Strengthened category distinctions** (Document Issue vs Fraud Concern)
- **Enhanced fraud keywords** with user behavior phrases ("not me", "someone else using", etc.)
- **Manual trigger for workflow execution** (easier testing and control)
- **Professional naming convention** (e.g., "Ticket Triage - Cloud Version", "Fraud Validation Layer")

## Issues Resolved
- Removed weak fraud keywords (e.g., "suspicious verification", "same selfie")
- Added clearer distinction between legitimate document issues and fraud concerns
- Added edge case handling for empty/vague tickets
- Fixed n8n CSV parsing: Used `this.helpers.getBinaryDataBuffer()` instead of manual Buffer decoding
- Parse CSV now returns 50 items correctly ✓
- Switched from Docker to n8n Cloud (solved environment variable access issues)
- Embedded CSV data in Code node (no file system dependencies)
- Migrated to n8n credential system (better security than env vars)
- Fixed URL in Call Gemini API node (removed duplicate authentication)
- Fixed Parse Response to handle Gemini markdown formatting (strips ```json wrappers)
- **Removed time-based urgency escalation** (historical data was causing incorrect classifications)
- **Updated prompt to 128 lines** with comprehensive rules and examples
- **Removed hours_since_created and status from LLM input** (content-only classification)

## Prompt Engineering Highlights
- **128-line comprehensive prompt** (`classification_prompt.txt`)
- 6 numbered rules (General Principles, Hierarchy, Urgency, Categories, Summary, Fraud)
- Explicit output format specification (no markdown, no preambles)
- 4 good summary examples + 4 bad examples with explanations
- Content-only urgency rule (critical constraint added)
- Edge case handling for empty tickets
- Fraud detection guidelines (8 true conditions + 8 false conditions)

## Testing Results
✅ **TKT-1031** (General Inquiry): Correctly classified
- Urgency: Low
- Category: General Inquiry
- Summary: "Customer is requesting information about local reseller partners in Brazil"
- flag_for_review: false

## Next Steps
1. **Add Error Handling and Reliability** -  API Rate Limit / Quota Handling, Temporary API Failures, Invalid or Empty AI Response
5. **Implement JSON schema enforcement** via `generationConfig.response_schema` (optional enhancement)
2. **Export results** to permanent storage (Google Sheets, Database, or local file)
3. **Remove "Limit to 1" node** and process all 50 tickets (when API quota permits)
4. **Performance analysis**: Review classification accuracy and fraud detection rate

## Production Readiness
✅ Comprehensive classification prompt  
✅ Dual-layer fraud detection (AI + rules)  
✅ CSV input/output handling  
✅ Secure credential management  
✅ No file system dependencies (cloud-ready)  
✅ Scalable architecture (1 to 50+ tickets)  
✅ Proper error handling in parse logic  
🔄 Pending: Full batch testing (50 tickets)  
🔄 Pending: Results export to permanent storage  

## Documentation
- `docs/N8N_CLOUD_SETUP.md` - Complete setup guide with step-by-step instructions
- `prompts/classification_prompt.txt` - 128-line AI prompt with 6 rules
- `prompts/fraud_keywords.txt` - 25 fraud indicator keywords
- `n8n/ticket_triage_cloud.json` - 187-line workflow definition (6 nodes)
