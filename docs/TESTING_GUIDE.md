# Testing Guide for Error Handling

Last updated: Wednesday, May 20, 2026 8:15 PM

## Quick Start

The error handling has been implemented. Here's how to test it with **one ticket**.

## What Was Implemented

### 1. Call Gemini API with Retry Node

**Previous:** HTTP Request node with no error handling  
**Now:** Code node with comprehensive error handling

**Features:**

- ✅ Detects rate limit errors (HTTP 429, "quota exceeded", "rate limit")
- ✅ Detects temporary failures (HTTP 500+, timeout)
- ✅ Waits 7 seconds before retry
- ✅ Retries exactly once
- ✅ Returns fallback indicator if both attempts fail

### 2. Parse Response with Fallback Node

**Previous:** Simple JSON parsing that could crash on errors  
**Now:** Comprehensive validation with graceful fallback

**Features:**

- ✅ Handles empty or null responses
- ✅ Handles invalid JSON
- ✅ Handles missing required fields
- ✅ Returns safe fallback classification on any error
- ✅ Never crashes the workflow

## Fallback Classification

When any error occurs, the system uses this safe classification:

```json
{
  "urgency": "Low",
  "category": "General Inquiry",
  "summary": "Ticket could not be classified automatically due to an invalid AI response.",
  "flag_for_review": true
}
```

**Key Point:** `flag_for_review` is set to `true` so you can identify failed classifications.

## Testing Instructions

### Step 1: Import Updated Workflow to n8n Cloud

1. Go to your n8n Cloud dashboard
2. Open your "Ticket Triage - Cloud Version" workflow
3. Click the three-dot menu → "Import from File"
4. Select: `c:\Users\Lenovo\Desktop\au10tix-home-test\n8n\ticket_triage_cloud.json`
5. Confirm to replace the existing workflow

### Step 2: Verify Node Names Changed

You should see these updated node names:

- ✅ "Call Gemini API with Retry" (was "Call Gemini API")
- ✅ "Parse Response with Fallback" (was "Parse Response")

### Step 3: Test Normal Operation (Expected: Success)

1. Make sure "Limit to 1" node is active (should process only TKT-1031)
2. Click "Execute Workflow"
3. Wait for completion (should take 5-10 seconds)
4. Check "Format CSV Output" node results

**Expected Output:**

```csv
ticket_id,created_at,customer_name,customer_email,subject,body,status,urgency,category,summary,flag_for_review
TKT-1031,2026-04-28 02:44:00,Greta Lindberg,glindberg@nordicfintech.example,General inquiry,Looking for a reseller in Brazil. Do you have local partners?,pending,Low,General Inquiry,Customer is requesting information about local reseller partners in Brazil,false
```

**Verification:**

- ✅ `urgency`: Low
- ✅ `category`: General Inquiry
- ✅ `flag_for_review`: false (normal classification, no errors)
- ✅ Summary describes the ticket content

### Step 4: Test Error Handling (Simulated)

To test error handling, you would need to:

**Option A: Temporarily break the API key**

1. Go to Credentials → "Gemini API Key"
2. Add a random character to the end of the API key (e.g., add "X")
3. Save the credential
4. Execute workflow
5. Expected: Workflow completes but returns fallback classification
6. Restore the correct API key after testing

**Expected Output:**

```csv
ticket_id,created_at,customer_name,customer_email,subject,body,status,urgency,category,summary,flag_for_review
TKT-1031,2026-04-28 02:44:00,Greta Lindberg,glindberg@nordicfintech.example,General inquiry,Looking for a reseller in Brazil. Do you have local partners?,pending,Low,General Inquiry,Ticket could not be classified automatically due to an invalid AI response.,true
```

**Verification:**

- ✅ `flag_for_review`: true (indicates fallback was used)
- ✅ Summary: "Ticket could not be classified automatically due to an invalid AI response."
- ✅ Workflow did not crash

**Option B: Review n8n Execution Logs**

1. Click on the workflow execution in the history
2. Check "Call Gemini API with Retry" node output
3. Look for `isFallback` field:
   - `isFallback: false` = API call succeeded
   - `isFallback: true` = API call failed, retry failed, using fallback

## How to Identify Failed Classifications in Output

In the CSV output, look for these indicators:

1. **`flag_for_review` = true** AND
2. **`summary` = "Ticket could not be classified automatically due to an invalid AI response."**

These tickets need manual review because the AI classification failed.

## Error Scenarios Handled

| Error Type | Detection | Action | Outcome |
|------------|-----------|--------|---------|
| HTTP 429 (Rate Limit) | Status code 429 | Wait 7s, retry once | Success or fallback |
| Quota Exceeded | Error message contains "quota exceeded" | Wait 7s, retry once | Success or fallback |
| HTTP 500 (Server Error) | Status code >= 500 | Wait 7s, retry once | Success or fallback |
| Timeout | Error message contains "timeout" | Wait 7s, retry once | Success or fallback |
| Empty Response | No candidates array | Immediate fallback | Fallback classification |
| Invalid JSON | JSON parse error | Immediate fallback | Fallback classification |
| Missing Fields | Required fields missing/null | Immediate fallback | Fallback classification |

## Monitoring Recommendations

After processing tickets, check:

1. **Total tickets processed**: Count rows in CSV output
2. **Fallback count**: Count rows where `flag_for_review = true` AND summary contains "invalid AI response"
3. **Fallback rate**: (Fallback count / Total tickets) × 100%

**Healthy System:**

- Fallback rate < 5%
- Most tickets have `flag_for_review = false`
- Summaries are descriptive and content-specific

**Problem Indicators:**

- Fallback rate > 10%
- Many tickets with fallback classification
- Check API quota, API key, network connectivity

## What Did NOT Change

The following remain unchanged:

- ✅ Classification prompt (128 lines)
- ✅ Fraud validation logic (15 keywords)
- ✅ CSV output structure (11 fields)
- ✅ Workflow architecture (6 nodes)
- ✅ "Limit to 1" node (still testing with 1 ticket)

## Next Steps After Testing

1. **If test succeeds:**
   - Proceed to test with 3-5 tickets
   - Then process all 50 tickets
   - Export results to permanent storage

2. **If test fails:**
   - Check n8n execution logs
   - Verify API key is correct
   - Check Gemini API service status
   - Review ERROR_HANDLING.md troubleshooting section

## File Changes Summary

- ✅ `n8n/ticket_triage_cloud.json` - Updated workflow (177 lines)
- ✅ `docs/MEMORY_BANK.md` - Updated with Stage 3 completion
- ✅ `docs/ERROR_HANDLING.md` - New comprehensive error handling documentation
- ✅ `docs/TESTING_GUIDE.md` - This file

## Questions?

Refer to:

- `docs/ERROR_HANDLING.md` - Detailed technical documentation
- `docs/MEMORY_BANK.md` - Project progress and architecture
- `docs/N8N_CLOUD_SETUP.md` - n8n Cloud setup instructions
