# n8n Cloud Setup Guide

## Step 1: Sign Up for n8n Cloud
1. Go to https://n8n.io/cloud/
2. Click "Start Free Trial" or "Sign Up"
3. Create your account

## Step 2: Create Gemini API Credential

### Get Your API Key:
1. Go to https://aistudio.google.com/app/apikey
2. Sign in with your Google account
3. Click "Create API Key"
4. Copy the key (starts with `AIzaSy...`)

### Add Credential in n8n Cloud:
1. In n8n Cloud, click **Credentials** in the left sidebar
2. Click **Add Credential**
3. Search for "HTTP Query Auth" (or "Generic Credential Type")
4. Configure it:
   - **Credential Name**: `Gemini API Key`
   - **Credential Type**: Query Auth
   - **Query Parameter Name**: `key`
   - **Query Parameter Value**: Paste your Gemini API key
5. Click **Save**

## Step 3: Import the Workflow

1. Download or copy the file: `n8n/ticket_triage_cloud.json`
2. In n8n Cloud, click the **menu (⋮)** at the top
3. Select **Import from File**
4. Upload `ticket_triage_cloud.json`
5. Click **Import**

## Step 4: Link the Credential to the Workflow

1. Open the imported workflow
2. Click on the **"Call Gemini API"** node
3. In the node settings, find the **Credentials** section
4. Select your "Gemini API Key" credential
5. Click **Save**

## Step 5: Test the Workflow

1. Click **Execute Workflow** (or press the play button)
2. Check each node to see the data flow:
   - **Load and Parse CSV**: Should show 50 items (all tickets)
   - **Limit to 1**: Should show 1 item (first ticket)
   - **Call Gemini API**: Should show Gemini's raw response
   - **Parse Response**: Should show the classified ticket with urgency, category, summary, and flag_for_review

## Expected Output

The **Parse Response** node should output something like:

```json
{
  "ticket_id": "TKT-1031",
  "created_at": "2026-04-28 02:44:00",
  "customer_name": "Greta Lindberg",
  "customer_email": "glindberg@nordicfintech.example",
  "subject": "General inquiry",
  "body": "Looking for a reseller in Brazil. Do you have local partners?",
  "status": "pending",
  "urgency": "Low",
  "category": "General Inquiry",
  "summary": "Customer is looking for a reseller partner in Brazil.",
  "flag_for_review": false
}
```

## Key Differences from Local Docker Setup

✅ **No file system access needed** - CSV data embedded in workflow
✅ **No environment variables** - using n8n's secure credential system
✅ **No Docker** - fully cloud-managed
✅ **Better security** - credentials encrypted by n8n Cloud
✅ **Accessible anywhere** - no localhost limitations

## Next Steps

After Step 5 works successfully:
- **Step 6**: Process ALL 50 tickets (remove the "Limit to 1" node)
- **Step 7**: Add CSV export to save results to `output/`
- **Step 8**: Add fraud keyword matching logic

---

**Note**: The credential ID in the workflow file is a placeholder. n8n Cloud will automatically assign the correct credential ID when you link it in Step 4.
