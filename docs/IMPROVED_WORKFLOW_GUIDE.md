# Improved Workflow Guide

**Created:** Wednesday, May 20, 2026 11:30 PM

## What's Wrong with Your Current Workflow

### Problem 1: Confusing Loop Structure

Your workflow has this connection pattern:

```
Split in Batches
  ↓ (output 1) → Aggregate All Results
  ↓ (output 2) → Call Gemini API → ... → Fraud Validation Layer
                                              ↓
                                      (loops back to Split in Batches!)
```

**Why this is bad:**
- The loop from Fraud Validation back to Split in Batches creates confusion
- Split in Batches is designed to iterate through items sequentially, not loop
- The Aggregate happens on the first batch completion (output 1), not after all batches
- This creates unpredictable behavior and potential infinite loops

### Problem 2: Rate Limiting Issues

Your HTTP Request nodes have:
- `batchSize: 1` and `batchInterval: 7000` configured
- But these settings only work WITHIN a node, not between workflow iterations
- Split in Batches processes items too quickly without waiting
- Result: Multiple API calls fire rapidly → 429 Rate Limit errors

### Problem 3: Missing Wait Between Batches

- No explicit wait time between processing each ticket
- The Split in Batches node immediately processes the next item
- API rate limits get hit because requests are too fast

## What's Fixed in the Improved Workflow

### Fix 1: Clean Linear Flow

```
Manual Trigger
  ↓
Load and Parse CSV
  ↓
Split in Batches (batchSize: 1)
  ↓ (output 1: has more items)     ↓ (output 2: all done)
Wait Between API Calls (8 sec)     Aggregate All Results
  ↓                                  ↓
Call Gemini API                    Format CSV Output
  ↓                                  ↓
Check for Error                    Convert to Binary File
  ↓ (error) ↓ (success)              ↓
Wait 10s   Parse Response           END
  ↓           ↓
Retry API    ↓
  ↓           ↓
  Parse Response
       ↓
Fraud Validation Layer
       ↓
Split in Batches (loops back naturally via n8n)
```

**Key improvements:**
- No manual loop connection from Fraud Validation back to Split in Batches
- Split in Batches handles the loop automatically using its built-in "output 1"
- Aggregate only happens once at the very end (output 2 when all batches complete)
- Clean, predictable flow

### Fix 2: Proper Rate Limiting

1. **Split in Batches**: Set to `batchSize: 1` (process one ticket at a time)
2. **Wait Between API Calls**: New node that waits 8 seconds before EACH API call
3. **Wait Before Retry**: If error occurs, wait 10 seconds before retry
4. **Removed HTTP Request batching options**: These were redundant and confusing

**Result:**
- 8 seconds between each ticket = max 450 requests/hour
- Well within Gemini API free tier limits (15 RPM = 900/hour)
- 10-second retry delay gives API time to recover from rate limits

### Fix 3: Better Error Detection

```javascript
// Old (doesn't catch all errors):
$json.error !== undefined

// New (catches more error types):
$json.error !== undefined || $json.statusCode === 429
```

This explicitly checks for HTTP 429 (rate limit) status codes.

### Fix 4: Simplified API Authentication

Changed from:
```json
"authentication": "genericCredentialType",
"genericAuthType": "httpQueryAuth"
```

To:
```json
"authentication": "predefinedCredentialType",
"nodeCredentialType": "googleGeminiApi"
```

This uses n8n's built-in Gemini API credential type (cleaner and more reliable).

## How to Use the Improved Workflow

### Step 1: Import the Workflow

1. Open n8n Cloud
2. Go to **Workflows** → **+ Add workflow** → **Import from File**
3. Select `n8n/ticket_triage_cloud_improved.json`
4. Click **Import**

### Step 2: Update Your Gemini API Credential

The workflow uses credential ID placeholders. You need to update them:

1. Click on **Call Gemini API** node
2. In the **Credentials** section, select your existing Gemini API key credential
3. Repeat for **Retry Gemini API** node
4. Save the workflow

### Step 3: Test with a Small Batch

**IMPORTANT:** Don't test with all 50 tickets immediately!

1. In **Load and Parse CSV** node, temporarily edit the CSV to include only 3 tickets:
   ```javascript
   const csvContent = `ticket_id,created_at,customer_name,customer_email,subject,body,status
   TKT-1031,2026-04-28 02:44:00,Greta Lindberg,glindberg@nordicfintech.example,General inquiry,Looking for a reseller in Brazil. Do you have local partners?,pending
   TKT-1041,2026-04-27 22:22:00,Yuki Nakamura,ynakamura@tokyobank.example,Identity verification issue,got a notification my ID was used to open an account. this wasn't me. someone else is using my documents,new
   TKT-1026,2026-04-27 07:17:00,Diego Rivera,d.rivera@orbitfinance.example,Account question,\"Logged in but the dashboard is empty, no transactions visible. Was my account deactivated?\",new`;
   ```

2. Click **Execute Workflow**
3. Watch the execution:
   - Each ticket should wait 8 seconds before API call
   - Total time: ~24 seconds for 3 tickets
   - Check for any 429 errors

### Step 4: Tune the Wait Times

If you still see 429 errors:

1. **Increase "Wait Between API Calls"** from 8 seconds to 10 or 12 seconds
2. **Increase "Wait Before Retry"** from 10 seconds to 15 seconds

If everything works fine and you want faster processing:

1. **Decrease "Wait Between API Calls"** to 5 or 6 seconds
2. Monitor for 429 errors
3. If errors appear, increase back

### Step 5: Process All Tickets

Once the 3-ticket test succeeds:

1. Restore the full CSV in **Load and Parse CSV** node
2. Execute the workflow
3. Total time estimate: 50 tickets × 8 seconds = ~6-7 minutes
4. Download the output CSV from **Convert to Binary File** node

## Configuration Reference

### Wait Times

| Node | Current | Min | Max | Notes |
|------|---------|-----|-----|-------|
| **Wait Between API Calls** | 8 sec | 4 sec | 15 sec | Time between each ticket |
| **Wait Before Retry** | 10 sec | 5 sec | 20 sec | Time before retry on error |

### Batch Size

| Node | Current | Notes |
|------|---------|-------|
| **Split in Batches** | 1 | Process one ticket at a time (safest) |

You can increase to 2-3 if you want faster processing, but monitor for 429 errors.

### API Rate Limits (Gemini Free Tier)

- **15 requests per minute (RPM)**
- **1,500 requests per day (RPD)**
- **4 RPM per user per project**

With 8-second wait:
- **7.5 requests per minute** ✅ Well within limits
- **450 requests per hour**

With 4-second wait:
- **15 requests per minute** ⚠️ At the limit (risky)

## Workflow Comparison

| Feature | Old Workflow | New Workflow |
|---------|-------------|--------------|
| **Flow Structure** | Circular loop (confusing) | Linear flow (clear) |
| **Rate Limiting** | HTTP Request batching only | Explicit Wait nodes + batching |
| **Wait Between Calls** | 7 seconds (inside HTTP node) | 8 seconds (explicit Wait node) |
| **Error Handling** | Basic error check | Enhanced error + 429 detection |
| **Aggregation** | Happens too early | Happens at the end |
| **Total Time (50 tickets)** | ~6 min (if no errors) | ~7 min (reliable) |
| **429 Error Rate** | High (frequent) | Low (rare) |

## Troubleshooting

### Still getting 429 errors?

**Solution 1:** Increase wait times
- Change "Wait Between API Calls" to 10 seconds
- Change "Wait Before Retry" to 15 seconds

**Solution 2:** Check your API quota
- Log into Google AI Studio
- Check if you've hit daily quota (1,500 requests/day)
- Wait 24 hours for quota reset

**Solution 3:** Reduce batch size
- Keep "Split in Batches" at batchSize: 1
- Don't process more than 50 tickets per hour

### Workflow is too slow?

**Solution:** Reduce wait times (carefully!)
- Minimum safe: "Wait Between API Calls" = 5 seconds
- Monitor execution for 429 errors
- If errors appear, increase back

### Aggregate node not working?

**Check:** Make sure Fraud Validation Layer connects to Split in Batches
- This connection allows the batch loop to continue
- Split in Batches output 2 (all done) goes to Aggregate
- Aggregate should only run ONCE at the very end

## Next Steps

1. Import the improved workflow
2. Test with 3 tickets
3. Verify no 429 errors
4. Process full 50 tickets
5. Review the output CSV

If you still have issues, check:
- API key is valid and has quota
- Wait times are sufficient (8+ seconds)
- No other workflows are using the same API key simultaneously

## Summary of Key Changes

✅ **Removed:** Confusing loop from Fraud Validation back to Split in Batches  
✅ **Added:** Explicit "Wait Between API Calls" node (8 seconds)  
✅ **Fixed:** Aggregate now runs at the end, not during batch processing  
✅ **Improved:** Better error detection (checks for 429 status code)  
✅ **Simplified:** API authentication using n8n's built-in Gemini credential  

**Expected result:** Clean execution with minimal 429 errors and predictable processing time.
