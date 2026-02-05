# Receipt Processing Automation

A webhook-triggered n8n workflow that automatically processes expense receipts from Airtable: downloads attachments, converts images to PDF, uploads to Dropbox, and updates record status - all with comprehensive error handling and notifications.

## What This Does

When a new expense is added to Airtable (or an existing expense is marked for processing), Airtable sends a webhook to n8n. This workflow automatically:

1. Retrieves the expense record and attachment from Airtable
2. Downloads the receipt file
3. Converts images (JPG/PNG) to PDF using CloudConvert
4. Uploads the PDF to Dropbox with a structured filename
5. Updates the Airtable record to mark it as processed
6. Sends email notifications if any step fails

Perfect for businesses tracking expenses, consultants managing client receipts, or teams needing organized receipt archiving.

## Key Features

- **Webhook-Triggered**: Instant processing when expenses are added
- **Smart File Handling**: Converts images to PDF, passes through existing PDFs unchanged
- **Organized Storage**: Files saved with consistent naming: `ExpenseName YYYYMMDD.pdf`
- **Comprehensive Error Handling**: Email notifications at every failure point
- **Idempotent**: Safe to retry - won't duplicate files or break on re-runs
- **Status Tracking**: Updates Airtable fields to track processing state

## Use Cases

- Automated expense receipt archiving for accounting/tax purposes
- Client reimbursement documentation management
- Receipt collection for credit card reconciliation
- Automated backup of financial documents to cloud storage

## Data Flow

```
Airtable Automation triggers webhook
  → n8n receives webhook with record ID
  → Fetch Airtable record
  → Download receipt attachment
  → Check file type
    ├─ Image (jpg/png): Convert to PDF via CloudConvert
    └─ Already PDF: Skip conversion
  → Upload to Dropbox with structured filename
  → Update Airtable record status
  → Send success/failure notifications
```

## Architecture Overview

This workflow demonstrates several production patterns:

- **Graceful degradation**: Each step validates success before proceeding
- **Error isolation**: Failures at any step trigger specific notifications
- **Atomic operations**: Dropbox upload and Airtable update are independent
- **Clear audit trail**: Email notifications provide complete failure context

## Airtable Setup

### Required Fields

Your Airtable Expenses table needs these fields:

| Field Name | Type | Description |
|------------|------|-------------|
| Expense Name | Single line text | Description of the expense |
| Date | Date | When expense occurred |
| Amount | Currency | Expense amount |
| Category | Single select | Expense category |
| Receipt | Attachment | The receipt file (image or PDF) |
| Submitted | Checkbox | Marks expense as ready for processing |
| Synced to Dropbox | Checkbox | Workflow sets this when complete |

### Airtable Automation Setup

Create an Airtable automation:

**Trigger**: When record matches conditions
- Conditions: `Submitted = checked` AND `Synced to Dropbox = unchecked`

**Action**: Send webhook
- URL: `https://your-n8n-instance.com/webhook/airtable-receipt`
- Method: POST
- Body:
```json
{
  "recordId": "AIRTABLE_RECORD_ID()"
}
```

Replace `your-n8n-instance.com` with your actual n8n webhook URL (found in the Webhook node after importing).

## n8n Setup

### 1. Prerequisites

- n8n instance (cloud or self-hosted)
- Airtable Personal Access Token
- Dropbox OAuth2 credentials
- CloudConvert API account (free tier available)
- Gmail OAuth2 for notifications (or another email service)

### 2. Import the Workflow

1. Download `workflow.json` from this repository
2. In n8n: **Import from File**
3. Select the downloaded JSON

### 3. Configure Credentials

Replace placeholder credential IDs:

**Airtable Personal Access Token** (`YOUR_AIRTABLE_CREDENTIAL_ID`)
- Scopes needed: `data.records:read`, `data.records:write`, `schema.bases:read`
- Used in: Get Airtable record, Update Airtable record

**Dropbox OAuth2** (`YOUR_DROPBOX_CREDENTIAL_ID`)
- Scopes needed: `files.content.write`
- Used in: Add file to Dropbox

**CloudConvert OAuth2** (`YOUR_CLOUDCONVERT_CREDENTIAL_ID`)
- Used in: CloudConvert node
- Free tier: 25 conversions/day

**Gmail OAuth2** (`YOUR_GMAIL_CREDENTIAL_ID`)
- Used in: All notification nodes
- Alternative: Replace with email service of choice (SendGrid, Mailgun, etc.)

### 4. Configure Airtable Base & Table

Replace these placeholders:

- `YOUR_AIRTABLE_BASE_ID` - Your base ID (starts with `app...`)
- `YOUR_AIRTABLE_TABLE_ID` - Your table ID (starts with `tbl...`)

**Finding IDs**: Airtable Help → API Documentation → locate IDs in the docs

### 5. Configure Dropbox Path

In the **"Add file to Dropbox"** node, update the path:

```javascript
path: "=/Receipts/{{ $('Get Airtable record').item.json['Expense Name'] }} {{ DateTime.fromISO($('Get Airtable record').item.json.Date).toFormat('yyyyMMdd') }}.pdf"
```

Change `/Receipts/` to your desired Dropbox folder path.

### 6. Configure Email Notifications

Replace `YOUR_EMAIL@example.com` in all Gmail notification nodes with your actual email address.

**Notification types:**
- Invalid webhook data
- Download failure
- Conversion failure  
- Upload failure
- Airtable update failure

### 7. Get Webhook URL

1. Open the **Webhook** node
2. Copy the **Production URL**
3. Use this URL in your Airtable automation

### 8. Test the Workflow

1. Add a test expense in Airtable with a receipt attachment
2. Check the `Submitted` checkbox
3. Monitor n8n execution log
4. Verify:
   - File appears in Dropbox
   - `Synced to Dropbox` is checked in Airtable
   - No error emails received

### 9. Activate

Click **Active** toggle in n8n - workflow is now live.

## File Naming Convention

Files are saved to Dropbox with this format:

```
ExpenseName YYYYMMDD.pdf
```

Examples:
- `Office Supplies 20240115.pdf`
- `Client Lunch 20240120.pdf`
- `Software Subscription 20240201.pdf`

This ensures:
- Chronological sorting
- Human-readable names
- No duplicate filenames (Date + Expense Name is unique)

## Customization Options

### Support Additional File Types

To handle more image formats, modify the **"Is Image (jpg/png)?"** condition:

```javascript
conditions: [
  { "leftValue": "={{ $json.Receipt[0].type }}", "rightValue": "image/jpeg" },
  { "leftValue": "={{ $json.Receipt[0].type }}", "rightValue": "image/png" },
  { "leftValue": "={{ $json.Receipt[0].type }}", "rightValue": "image/webp" }  // Add this
]
```

CloudConvert supports: jpg, png, gif, bmp, webp, tiff, and many more.

### Change Dropbox Folder Structure

Organize by month, client, or category:

```javascript
// By month
path: "=/Receipts/{{ DateTime.fromISO($('Get Airtable record').item.json.Date).toFormat('yyyy-MM') }}/{{ ... }}"

// By category
path: "=/Receipts/{{ $('Get Airtable record').item.json.Category }}/{{ ... }}"

// By client (if you have a Client field)
path: "=/Receipts/{{ $('Get Airtable record').item.json.Client }}/{{ ... }}"
```

### Use Different Cloud Storage

Replace the Dropbox node with:
- **Google Drive**: Use Google Drive node
- **OneDrive**: Use Microsoft OneDrive node
- **S3**: Use AWS S3 node
- **SFTP**: Use FTP node for self-hosted storage

### Alternative to CloudConvert

CloudConvert has usage limits on the free tier. Alternatives:

**ImageMagick** (self-hosted n8n only):
- Install ImageMagick on your n8n server
- Replace CloudConvert node with Execute Command node:
```bash
convert input.jpg -quality 100 output.pdf
```

**Python script** (self-hosted):
- Use n8n's Code node with Python
- Libraries: PIL (Pillow), img2pdf

### Batch Processing

To process multiple unsynced expenses at once:

1. Replace Webhook trigger with **Schedule Trigger**
2. Add **Airtable Search** node: Find records where `Submitted = checked` AND `Synced to Dropbox = unchecked`
3. Loop through results

This is useful for bulk backfills or if the webhook misses records.

## Error Handling Details

The workflow includes 5 error notification points:

| Failure Point | Email Subject | Likely Cause |
|---------------|---------------|--------------|
| Invalid webhook | Invalid Webhook Data | Airtable automation misconfigured |
| Record not found | Receipt Download Failed | Record deleted before processing |
| Download failed | Receipt Download Failed | Attachment URL expired or invalid |
| Conversion failed | Image Conversion Failed | CloudConvert service down or quota exceeded |
| Upload failed | Dropbox Upload Failed | Dropbox authentication expired or storage full |
| Update failed | Airtable Update Failed | Airtable token expired (file uploaded successfully) |

Each email includes:
- Record ID
- Expense Name
- Date
- Specific failure context

This makes manual intervention straightforward when issues occur.

## Troubleshooting

**Webhook not triggering:**
- Verify webhook URL is correct in Airtable automation
- Check n8n webhook node is active
- Test webhook manually with Postman/curl

**CloudConvert quota exceeded:**
- CloudConvert free tier: 25 conversions/day
- Upgrade account or implement daily batch processing
- Alternative: Use ImageMagick for unlimited conversions (self-hosted)

**Dropbox upload fails with "invalid path":**
- Check Dropbox folder exists
- Ensure Expense Name doesn't contain illegal characters (`/`, `\`, `:`, `*`, `?`, `"`, `<`, `>`, `|`)
- Add sanitization in the path expression if needed

**Airtable update fails after successful upload:**
- File is in Dropbox but record not marked complete
- Check Airtable token permissions
- Manually update record and investigate token expiry

**Duplicate files in Dropbox:**
- Workflow uses file naming based on Expense Name + Date
- If same expense + date combination exists, Dropbox will append (1), (2), etc.
- Ensure Expense Name + Date combination is unique per receipt

## Production Best Practices

1. **Monitor error emails daily** - Set up email filters/labels for quick review
2. **Test with all file types** - JPG, PNG, PDF before going live
3. **Backup Airtable regularly** - Export expenses monthly as CSV backup
4. **CloudConvert alternatives** - Have ImageMagick ready if you hit API limits
5. **Webhook security** - Consider adding authentication to webhook endpoint
6. **Rate limiting** - Airtable automation may batch webhooks; ensure n8n can handle volume

## Cost Considerations

**Free tier limits:**
- n8n.cloud: 5,000 workflow executions/month
- CloudConvert: 25 conversions/day
- Airtable: 50,000 records per base
- Dropbox: 2GB storage (Basic plan)

**Scaling:**
- Self-host n8n for unlimited executions
- CloudConvert paid plan: $9/month for 500 conversions
- Business Airtable for higher limits
- Dropbox Plus: 2TB storage

## License

MIT - Feel free to use, modify, and distribute. Attribution appreciated but not required.

## About

Created by [Ken Thompson](https://github.com/your-username) at [Lodgepole I/O](https://lodgepole.io) - Workflow design and operational systems consulting.

This workflow is based on production use for expense tracking and receipt management. It processes hundreds of receipts per month reliably since 2024.

## Related Workflows

Check out my other n8n templates:
- [Email-Based Time Tracking](../gmail-to-airtable-time-tracking) - Automated time logging for invoicing
- More coming soon!
