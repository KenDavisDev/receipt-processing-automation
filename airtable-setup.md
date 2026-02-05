# Airtable Automation Setup Guide

This guide walks you through setting up the Airtable automation that triggers the n8n workflow.

## Overview

The automation watches for new expense records (or existing records being marked as ready) and sends a webhook to n8n with the record ID.

## Step-by-Step Setup

### 1. Open Your Airtable Base

Navigate to your Expenses table where you track receipts.

### 2. Create an Automation

1. Click the **"Automations"** button (top right in Airtable)
2. Click **"Create automation"**
3. Name it: `Process Receipt to Dropbox`

### 3. Configure the Trigger

**Select trigger type:** When record matches conditions

**Settings:**
- **Table:** Expenses
- **View:** All Expenses (or create a specific view)
- **When:** Conditions are met
- **Conditions:**
  - Field: `Submitted` | equals | ☑ (checked)
  - AND
  - Field: `Synced to Dropbox` | equals | ☐ (unchecked)

This ensures only new submissions are processed, and already-synced records are ignored.

### 4. Add Webhook Action

**Action type:** Send webhook request

**Configuration:**

- **URL:** `https://your-n8n-instance.com/webhook/airtable-receipt`
  - Get this URL from your n8n Webhook node after importing the workflow
  - If using n8n.cloud, it looks like: `https://yourname.app.n8n.cloud/webhook/airtable-receipt`
  - If self-hosted, use your domain: `https://n8n.yourdomain.com/webhook/airtable-receipt`

- **Method:** POST

- **Content type:** application/json

- **Body:**
```json
{
  "recordId": "AIRTABLE_RECORD_ID()"
}
```

**Important:** Use `AIRTABLE_RECORD_ID()` as a dynamic field, not literal text. In Airtable's webhook body editor, click the **+** button and insert **"Record ID"** from the dynamic fields.

### 5. Test the Automation

1. Click **"Run test"** in Airtable
2. Select a test record (or create one)
3. Airtable will show you the webhook request/response
4. Check n8n execution log to verify it triggered

### 6. Turn On the Automation

Toggle the automation to **"On"** in Airtable.

## Alternative Trigger Configurations

### Option 1: Trigger on Record Creation Only

If you want the workflow to run immediately when a record is created:

**Trigger:** When record is created
- **Table:** Expenses
- No additional conditions needed

**Pros:** Instant processing upon creation
**Cons:** Records without attachments will fail and send error emails

### Option 2: Manual Button Trigger

If you prefer manual control:

**Trigger:** When button clicked
- Add a button field to your Expenses table
- Configure button to run the automation
- Users click button when ready to process

**Pros:** Full control over when processing happens
**Cons:** Requires manual action, not automatic

### Option 3: Scheduled Batch Processing

If you prefer processing receipts in batches:

Replace the Webhook trigger in n8n with:
- **Schedule Trigger** (e.g., daily at 6 PM)
- **Airtable Search** node to find all unprocessed records
- Loop through and process each

## Webhook Payload Reference

The webhook sends this JSON structure:

```json
{
  "recordId": "recABC123XYZ"
}
```

The n8n workflow uses this `recordId` to fetch the full record details from Airtable.

## Security Considerations

### Basic Security (Recommended)

The webhook URL is the only "security" by default - knowing the URL gives access.

**Best practices:**
- Don't publish your webhook URL publicly
- Consider rotating the URL periodically (regenerate in n8n)

### Enhanced Security (Optional)

Add authentication to the webhook:

**In Airtable webhook body, add:**
```json
{
  "recordId": "AIRTABLE_RECORD_ID()",
  "authToken": "your-secret-token-here"
}
```

**In n8n, add an IF node after Webhook:**
```javascript
{{ $json.body.authToken === "your-secret-token-here" }}
```

Only proceed if the token matches.

## Troubleshooting Automation Issues

**Automation not triggering:**
- Check conditions match your data (checkbox fields must be exactly ☑/☐)
- Verify automation is toggled "On"
- Check Airtable automation run history for errors

**Webhook fails in Airtable:**
- Verify n8n workflow is active
- Check webhook URL is correct (no trailing slash, correct protocol https://)
- Test webhook URL with curl or Postman first

**Multiple triggers for same record:**
- Ensure `Synced to Dropbox` condition is included
- Once n8n marks record complete, it won't trigger again

**Webhook times out:**
- Airtable has a 30-second timeout for webhooks
- If n8n takes longer, Airtable marks it as failed (but n8n continues processing)
- Use asynchronous processing: webhook responds immediately, processing happens after

## Testing Checklist

Before going live, test these scenarios:

- ✅ New record with JPG attachment
- ✅ New record with PNG attachment  
- ✅ New record with PDF attachment (should skip conversion)
- ✅ New record without attachment (should fail gracefully with email)
- ✅ Re-checking `Submitted` on already-processed record (should not retrigger)
- ✅ Manually un-checking `Synced to Dropbox` (should allow reprocessing)

## Example Airtable View

Create a filtered view for monitoring:

**View name:** Pending Processing
**Filter:**
- `Submitted` is ☑ checked
- `Synced to Dropbox` is ☐ unchecked

This shows all expenses waiting to be processed. Once they process successfully, they disappear from this view.

## Field Setup Summary

| Field | Type | Purpose |
|-------|------|---------|
| Expense Name | Single line text | Description for filename |
| Date | Date | For filename and organization |
| Amount | Currency | Tracking expense amount |
| Category | Single select | Optional categorization |
| Receipt | Attachment | The file to process |
| Submitted | Checkbox | User marks when ready to process |
| Synced to Dropbox | Checkbox | Workflow marks when complete |

## Next Steps

1. Set up the fields in your Expenses table
2. Create the automation following this guide
3. Import the n8n workflow
4. Configure all credentials in n8n
5. Test with a sample expense
6. Monitor for a few days before scaling up

Need help? Open an issue in this repository or reach out at [lodgepole.io](https://lodgepole.io).
