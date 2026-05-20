# Error Handling and Reliability Guide

Last updated: Wednesday, May 20, 2026 8:30 PM

## Overview

This document describes the error handling and reliability features implemented in the AU10TIX ticket triage workflow. The system uses n8n's native workflow nodes (HTTP Request, IF, Wait) to handle API failures gracefully without crashing the workflow or losing ticket data.

## Key Design Principles

1. **Lightweight and Simple** - Uses n8n native nodes, no complex retry systems
2. **Single Retry Policy** - Retry once with a fixed delay, then fail gracefully
3. **Graceful Degradation** - Use fallback classification when API fails
4. **Workflow Stability** - Never crash or terminate unexpectedly
5. **Standard n8n Patterns** - HTTP Request nodes with credentials, not Code nodes

## Implementation Details

### 1. API Call with Error Detection

**Nodes:** `Call Gemini API` → `Check for Error`

The workflow uses a standard HTTP Request node with these key settings:

- **Continue On Fail:** Enabled (allows workflow to continue even if HTTP request fails)
- **Ignore HTTP Status Errors:** Enabled (treats all HTTP responses as non-errors for workflow routing)
- **Timeout:** 30 seconds

When the API call fails (rate limit, server error, timeout), the HTTP Request node sets an `error` field in the output. The `Check for Error` IF node detects this field and routes the workflow appropriately.

**Error Detection Logic:**

```
IF $json.error is not empty
  → TRUE: Route to retry path
  → FALSE: Route to parse response (success)
```

### 2. Retry Logic with Wait

**Nodes:** `Wait 7 Seconds` → `Retry Gemini API`

If an error is detected, the workflow:

1. Routes to `Wait 7 Seconds` node
2. Waits exactly 7 seconds
3. Makes a second API call using `Retry Gemini API` (identical HTTP Request node)
4. Both success and retry paths merge at `Parse Response with Fallback`

**No infinite retries:** The retry only happens once. After the retry, the workflow continues regardless of success or failure.

### 3. Response Validation and Fallback

**Node:** `Parse Response with Fallback`

This Code node receives the API response (from either the first call or retry) and:

1. Checks if `error` field exists → use fallback
2. Validates response structure (candidates array exists) → use fallback if missing
3. Extracts AI text from response → use fallback if empty
4. Removes markdown code blocks (```json)
5. Parses JSON → use fallback if parsing fails
6. Validates required fields → use fallback if fields missing
7. Returns valid classification OR fallback

**Fallback Classification:**

```json
{
  "urgency": "Low",
  "category": "General Inquiry",
  "summary": "Ticket could not be classified automatically due to an invalid AI response.",
  "flag_for_review": true
}
```

## Workflow Flow Diagram

```
┌─────────────────────────────────────────────────────────────┐
│ Limit to 1                                                   │
│  → Passes single ticket to API call                         │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│ Call Gemini API (HTTP Request)                              │
│  - Continue On Fail: true                                   │
│  - Ignore HTTP Status Errors: true                          │
│  - Timeout: 30 seconds                                       │
│  - Uses n8n credential system                               │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│ Check for Error (IF Node)                                   │
│  Condition: $json.error is not empty                        │
│                                                               │
│  TRUE (Error)  │  FALSE (Success)                           │
│       ↓        │        ↓                                    │
│   ┌─────────┐ │   ┌────────────────────────────┐          │
│   │ Retry   │ │   │ Parse Response (success)   │          │
│   │ Path    │ │   │ Skip retry, go directly    │          │
│   └─────────┘ │   │ to parsing                 │          │
└─────────────────────────────────────────────────────────────┘
       ↓                           ↓
┌──────────────────┐               │
│ Wait 7 Seconds   │               │
└──────────────────┘               │
       ↓                           │
┌──────────────────────────────────┐│
│ Retry Gemini API (HTTP Request) ││
│  - Same configuration as first  ││
│  - Continue On Fail: true       ││
│  - References ticket from       ││
│    Limit to 1 node              ││
└──────────────────────────────────┘│
       ↓                           │
       └───────────┬───────────────┘
                   ↓
┌─────────────────────────────────────────────────────────────┐
│ Parse Response with Fallback (Code Node)                    │
│                                                               │
│  1. Check if $json.error exists → use fallback             │
│  2. Validate response structure                             │
│  3. Remove markdown code blocks                             │
│  4. Parse JSON                                               │
│  5. Validate required fields                                │
│  6. Return valid classification OR fallback                 │
│                                                               │
│  Fallback: urgency=Low, category=General Inquiry,           │
│            flag_for_review=true                             │
└─────────────────────────────────────────────────────────────┘
                            ↓
                    Continue workflow...
```

## Testing Scenarios

### Test 1: Normal Operation (Expected: Success)

- API returns valid JSON with all required fields
- Classification should proceed normally
- No fallback should be triggered

### Test 2: Rate Limit Error (Expected: Retry → Success or Fallback)

- First call returns HTTP 429
- Wait 7 seconds
- Second call succeeds or fails
- If second call fails, use fallback classification

### Test 3: Temporary Failure (Expected: Retry → Success or Fallback)

- First call returns HTTP 500
- Wait 7 seconds
- Second call succeeds or fails
- If second call fails, use fallback classification

### Test 4: Empty Response (Expected: Fallback)

- API returns empty response
- Use fallback classification immediately
- `flag_for_review` should be `true`

### Test 5: Invalid JSON (Expected: Fallback)

- API returns non-JSON text or malformed JSON
- JSON parsing fails
- Use fallback classification
- `flag_for_review` should be `true`

### Test 6: Missing Required Fields (Expected: Fallback)

- API returns valid JSON but missing `urgency` or other required field
- Use fallback classification
- `flag_for_review` should be `true`

## Monitoring and Logging

Currently, the workflow does not log errors to external systems. To monitor failures:

1. Check the `flag_for_review` field in the output CSV
2. Look for tickets with summary: "Ticket could not be classified automatically due to an invalid AI response."
3. These tickets should be manually reviewed

**Recommended Enhancement:**

Add a logging step after `Parse Response with Fallback` to track:

- Total API calls
- Failed API calls (rate limits, errors)
- Fallback classification usage rate
- Average response time

## Configuration

### Retry Delay

Current: **7 seconds**

To change the retry delay, modify this line in `Call Gemini API with Retry`:

```javascript
await new Promise(resolve => setTimeout(resolve, 7000)); // 7000ms = 7 seconds
```

Recommended range: 5-10 seconds

### Retry Count

Current: **1 retry** (2 total attempts)

To change the retry count, modify the retry logic:

```javascript
if ((isRateLimit || isTemporaryFailure) && retryCount === 0) {
  // Change the retry limit here
}
```

**Warning:** More than 1 retry is not recommended to avoid API quota exhaustion.

## What Was NOT Changed

The following components remain unchanged as requested:

- ✅ Classification prompt (`classification_prompt.txt`)
- ✅ Fraud validation logic (keyword detection)
- ✅ CSV output structure (11 fields)
- ✅ Fraud keywords list
- ✅ Node positions and workflow structure

## Production Considerations

1. **API Quota Management**
   - Monitor daily API usage
   - Set up alerts for quota thresholds
   - Consider upgrading API plan if hitting limits frequently

2. **Fallback Classification Rate**
   - Track how often fallback is used
   - High fallback rate indicates API reliability issues
   - Investigate if fallback rate > 5%

3. **Manual Review Process**
   - All fallback classifications have `flag_for_review = true`
   - Set up a process to manually review these tickets
   - Prioritize based on ticket subject/body content

4. **Retry Delay Tuning**
   - 7 seconds is conservative
   - Can reduce to 5 seconds if API recovers quickly
   - Can increase to 10 seconds if rate limits are frequent

## Troubleshooting

### Issue: All tickets are getting fallback classification

**Possible Causes:**

1. API key is invalid or expired
2. Gemini API service is down
3. Network connectivity issues
4. API quota exhausted

**Solution:**

- Check API key in n8n credentials
- Verify Gemini API service status
- Check network connectivity
- Review API usage dashboard

### Issue: Workflow is still crashing

**Possible Causes:**

1. Error in a different node (not API or Parse nodes)
2. Invalid CSV data format
3. Memory limits exceeded

**Solution:**

- Check n8n execution logs
- Validate CSV data structure
- Review "Load and Parse CSV" node output
- Check "Fraud Validation Layer" logic

### Issue: Retry is not working

**Possible Causes:**

1. Error type is not recognized as retryable
2. Retry delay is too short
3. API is consistently failing

**Solution:**

- Review error messages in n8n logs
- Add more error patterns to `isRateLimit` or `isTemporaryFailure` checks
- Increase retry delay
- Check Gemini API status page

## Summary

The error handling implementation provides:

✅ **Lightweight retry logic** (7-second delay, 1 retry)  
✅ **Rate limit handling** (HTTP 429, quota messages)  
✅ **Temporary failure handling** (HTTP 500, timeout)  
✅ **Invalid response handling** (empty, malformed, missing fields)  
✅ **Graceful fallback** (safe classification with review flag)  
✅ **Workflow stability** (no crashes or unexpected terminations)  

The system is production-ready for handling real-world API failures while maintaining data integrity and workflow stability.
