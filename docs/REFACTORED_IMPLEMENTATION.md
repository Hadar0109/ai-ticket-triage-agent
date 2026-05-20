# Refactored Error Handling - Implementation Update

**Date:** Wednesday, May 20, 2026 8:30 PM  
**Status:** ✅ Refactored and Fixed

## Issue Identified

The initial error handling implementation used a Code node to call the Gemini API with credential access via `$credentials`. However, **Code nodes in n8n do not have direct access to credentials** like HTTP Request nodes do.

**Error:** `$credentials is not defined [line 5]`

## Solution

Refactored the workflow to use **n8n's native workflow logic** with standard HTTP Request nodes and workflow routing:

### Before (Broken)
```
Call Gemini API with Retry (Code Node)
  └─ Tried to access $credentials.geminiApiKey.apiKey
  └─ ❌ Failed: $credentials not available in Code nodes
```

### After (Fixed)
```
Call Gemini API (HTTP Request)
  → Check for Error (IF Node)
    ├─ TRUE: Wait 7 Seconds → Retry Gemini API (HTTP Request)
    └─ FALSE: Direct to Parse Response
```

## Architecture Changes

### Old Workflow (6 Nodes)
1. Manual Trigger
2. Load and Parse CSV
3. Limit to 1
4. **Call Gemini API with Retry** (Code node - ❌ broken)
5. **Parse Response with Fallback**
6. Fraud Validation Layer
7. Format CSV Output

### New Workflow (9 Nodes)
1. Manual Trigger
2. Load and Parse CSV
3. Limit to 1
4. **Call Gemini API** (HTTP Request - ✅ uses n8n credentials)
5. **Check for Error** (IF node - ✅ routes workflow)
6. **Wait 7 Seconds** (Wait node - ✅ delay before retry)
7. **Retry Gemini API** (HTTP Request - ✅ second attempt)
8. **Parse Response with Fallback** (Code node - ✅ handles all responses)
9. Fraud Validation Layer
10. Format CSV Output

## Key Implementation Details

### 1. HTTP Request Node Configuration

Both `Call Gemini API` and `Retry Gemini API` use:

```json
{
  "type": "n8n-nodes-base.httpRequest",
  "parameters": {
    "method": "POST",
    "url": "https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash-lite:generateContent",
    "continueOnFail": true,
    "options": {
      "ignoreHttpStatusErrors": true,
      "timeout": 30000
    }
  },
  "credentials": {
    "geminiApiKey": {
      "id": "PLACEHOLDER_CREDENTIAL_ID",
      "name": "Gemini API Key"
    }
  }
}
```

**Key Settings:**
- `continueOnFail: true` - Workflow continues even if HTTP request fails
- `ignoreHttpStatusErrors: true` - Treats all HTTP responses as non-errors for routing
- Credentials accessed via n8n's native credential system

### 2. IF Node Error Detection

```json
{
  "type": "n8n-nodes-base.if",
  "parameters": {
    "conditions": {
      "conditions": [
        {
          "leftValue": "={{ $json.error }}",
          "rightValue": "",
          "operator": {
            "type": "object",
            "operation": "notEmpty"
          }
        }
      ]
    }
  }
}
```

**Logic:**
- Checks if `$json.error` field exists and is not empty
- TRUE output → Route to retry path (Wait → Retry)
- FALSE output → Route directly to Parse Response

### 3. Wait Node

```json
{
  "type": "n8n-nodes-base.wait",
  "parameters": {
    "amount": 7,
    "unit": "seconds"
  }
}
```

Simple 7-second delay before retry attempt.

### 4. Retry Node

Second HTTP Request node with **identical configuration** to the first, except it references the original ticket from `Limit to 1` node:

```
{{ $('Limit to 1').item.json.subject }}
{{ $('Limit to 1').item.json.body }}
```

This ensures the retry uses the same ticket data even though the workflow path diverged.

### 5. Parse Response Node

Updated to handle responses from **both** the first call and the retry:

```javascript
const apiResponse = $input.item.json;
const originalTicket = $('Limit to 1').item.json;

// Check if response has error field
if (apiResponse.error) {
  classification = fallbackClassification;
} else {
  // Validate and parse response
  // ...
}
```

## Error Handling Flow

```
Ticket → Call API
           ↓
     Has error field?
     ├─ YES → Wait 7s → Retry API → Parse
     └─ NO  → Parse (skip retry)
                ↓
          Valid response?
          ├─ YES → Use classification
          └─ NO  → Use fallback
```

## What Changed from Original Implementation

| Aspect | Old (Broken) | New (Fixed) |
|--------|--------------|-------------|
| **API Call** | Code node with $credentials | HTTP Request node with n8n credentials |
| **Retry Logic** | JavaScript async/await in Code | n8n IF/Wait/HTTP workflow nodes |
| **Error Detection** | JavaScript try/catch | IF node checking $json.error |
| **Delay** | setTimeout() in Code | Wait node (7 seconds) |
| **Credential Access** | $credentials.geminiApiKey.apiKey | n8n native credential system |
| **Workflow Nodes** | 6 nodes | 9 nodes |
| **Total Lines** | 177 lines | 303 lines |

## Benefits of Refactored Approach

### ✅ Advantages

1. **Uses n8n Native Features** - No custom credential handling
2. **Visual Workflow** - Error/retry path visible in n8n canvas
3. **Standard Pattern** - Follows n8n best practices
4. **Debugging** - Can see intermediate node outputs
5. **Maintainable** - Clear separation of concerns

### ⚠️ Trade-offs

1. **More Nodes** - 9 nodes instead of 6 (but clearer)
2. **More Connections** - More complex routing (but more transparent)
3. **File Size** - 303 lines vs 177 lines (but more readable)

## Testing Requirements

### Before Testing

1. Ensure Gemini API credential is configured in n8n
2. Verify credential name matches workflow reference
3. Check that "Limit to 1" node is active

### Test Scenarios

#### Test 1: Normal Operation
- **Expected:** API succeeds on first call
- **Path:** Call API → Check for Error (FALSE) → Parse Response
- **Result:** `flag_for_review: false`

#### Test 2: API Error with Successful Retry
- **Expected:** First call fails, retry succeeds
- **Path:** Call API → Check for Error (TRUE) → Wait → Retry API → Parse Response
- **Result:** Valid classification (depends on ticket content)

#### Test 3: Both Attempts Fail
- **Expected:** First call fails, retry fails
- **Path:** Call API → Check for Error (TRUE) → Wait → Retry API → Parse Response
- **Result:** Fallback classification with `flag_for_review: true`

#### Test 4: Invalid Response
- **Expected:** API returns malformed JSON
- **Path:** Call API → Check for Error (FALSE) → Parse Response detects invalid JSON
- **Result:** Fallback classification with `flag_for_review: true`

## Configuration Notes

### Retry Delay

To change the retry delay, update the `Wait 7 Seconds` node:

```json
{
  "parameters": {
    "amount": 7,  // Change this value
    "unit": "seconds"
  }
}
```

### Timeout

To change the API timeout, update both HTTP Request nodes:

```json
{
  "options": {
    "timeout": 30000  // milliseconds (30 seconds)
  }
}
```

### Retry Behavior

Currently: **1 retry** (2 total attempts)

To disable retry completely:
- Delete `Wait 7 Seconds` node
- Delete `Retry Gemini API` node
- Connect `Check for Error` TRUE output directly to `Parse Response with Fallback`

To add more retries (not recommended):
- Add another IF node after retry to check for error again
- Add another Wait node
- Add another HTTP Request node
- Becomes complex quickly (exponential backoff would be better)

## Files Updated

| File | Lines | Status |
|------|-------|--------|
| `n8n/ticket_triage_cloud.json` | 303 | ✅ Refactored |
| `docs/MEMORY_BANK.md` | Updated | ✅ Architecture updated |
| `docs/ERROR_HANDLING.md` | Updated | ✅ Flow diagram updated |
| `docs/REFACTORED_IMPLEMENTATION.md` | New | ✅ This file |

## Summary

The error handling implementation has been **successfully refactored** to use n8n's native HTTP Request nodes with the credential system instead of trying to access credentials in Code nodes.

**Key Changes:**
- ✅ HTTP Request nodes for API calls (not Code nodes)
- ✅ IF node for error detection (not JavaScript try/catch)
- ✅ Wait node for delay (not setTimeout)
- ✅ Workflow routing for retry logic (not async/await)
- ✅ n8n native credentials (not $credentials variable)

**Status:** Ready for testing in n8n Cloud

**Next Step:** Import `n8n/ticket_triage_cloud.json` to n8n Cloud and test with 1 ticket
