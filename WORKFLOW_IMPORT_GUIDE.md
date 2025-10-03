# n8n Workflow Import Guide

**Phase 1 Scope**: 3 workflows (voice-only)
**Phase 2 Scope**: +2 workflows (SMS automation) - not included in this guide

This guide provides step-by-step instructions for importing the 3 Phase 1 BCS debt collection workflows into your n8n instance.

## Prerequisites

Before importing workflows, ensure you have completed:

1. Set up Google Sheets with 23-column schema (see `BCS_DEBTORS_SCHEMA.csv`)
2. Created Vapi account with 6 assistants configured with verbal payment instructions (see `VAPI_ASSISTANT_CONFIG.md`)
3. Set all required environment variables in n8n (14 variables - see `.env.example`)
4. Payment information ready (BSB, account, Stripe link, mailing address) for verbal delivery

**Phase 2 Prerequisites** (not needed for Phase 1):
- Twilio account with SMS capabilities
- Email server configuration

## Workflow Import Order

Import workflows in this specific order to satisfy dependencies:

### 1. BCS_Call_Scheduler.json
**Purpose**: Initiates outbound calls every 30 minutes
**Dependencies**: None
**Priority**: Import first

### 2. BCS_Vapi_Webhook_Handler.json
**Purpose**: Processes call results from Vapi
**Dependencies**: None (payment instructions delivered verbally during call)
**Priority**: Import second

### 3. BCS_Test_Webhook.json
**Purpose**: Testing tool for Vapi webhook
**Dependencies**: BCS_Vapi_Webhook_Handler (must be imported and activated)
**Priority**: Import third

**Phase 2 Workflows** (not included in Phase 1):
- BCS_Send_Payment_SMS.json - Payment SMS automation
- BCS_SMS_Status_Handler.json - SMS delivery tracking

## Import Steps

### Step 1: Access n8n Workflows Page

1. Log into your n8n instance
2. Navigate to **Workflows** in the left sidebar
3. Click **Add Workflow** dropdown
4. Select **Import from File**

### Step 2: Import Each Workflow

For each workflow JSON file:

1. Click **Import from File**
2. Select the workflow JSON file (e.g., `BCS_Call_Scheduler.json`)
3. n8n will display a preview - review the nodes
4. Click **Import**
5. The workflow will appear in "Inactive" state

### Step 3: Configure Credentials

After importing each workflow, configure the required credentials:

#### Google Sheets Credentials

1. Open any workflow with Google Sheets nodes
2. Click on a Google Sheets node (e.g., "Read BCS Debtors")
3. In the **Credentials** section, click **Create New Credential**
4. Select **Google Sheets OAuth2 API**
5. Follow the OAuth2 flow to authorize n8n
6. Name the credential "Google Sheets account"
7. Copy the credential ID and update `.env`:
   ```
   GOOGLE_SHEETS_CREDENTIAL_ID=<credential_id>
   ```

**Note**: Twilio credentials not needed for Phase 1 (SMS is Phase 2)

### Step 4: Update Credential Placeholders

For each imported workflow, replace credential placeholder IDs:

1. Open the workflow in n8n editor
2. Click on each node with credentials (Google Sheets only for Phase 1)
3. Select the correct credential from the dropdown
4. Click **Save** in the top right

**Alternative (Manual JSON Edit)**:

1. Export the workflow after import
2. Search for `{{GOOGLE_SHEETS_CREDENTIAL_ID}}`
3. Replace with actual credential ID from Step 3
4. Re-import the updated workflow

### Step 5: Configure Environment Variables

In n8n, navigate to **Settings** > **Variables** and add all required variables from `.env.example`:

#### Required Variables (14 total - Phase 1)

```
GOOGLE_SHEETS_SPREADSHEET_ID=<your_spreadsheet_id>
VAPI_API_KEY=<your_vapi_api_key>
VAPI_WEBHOOK_SECRET=<your_vapi_webhook_secret>
VAPI_PHONE_NUMBER_ID=ph_<your_phone_number_id>
VAPI_CONSUMER_1=<assistant_id>
VAPI_CONSUMER_2=<assistant_id>
VAPI_CONSUMER_3=<assistant_id>
VAPI_COMMERCIAL_1=<assistant_id>
VAPI_COMMERCIAL_2=<assistant_id>
VAPI_COMMERCIAL_3=<assistant_id>
STRIPE_PAYMENT_LINK=<your_stripe_link>
BSB_NUMBER=<bank_bsb>
ACCOUNT_NUMBER=<bank_account>
ACCOUNT_NAME=<bank_account_name>
BCS_PHONE_NUMBER=<your_business_phone>
BCS_MAILING_ADDRESS=<your_mailing_address>
WEBHOOK_VAPI_HANDLER=<vapi_webhook_url>
PETER_EMAIL=<your_email>
PETER_NAME=<your_name>
```

**Note**: Payment information (Stripe link, BSB, account, etc.) is used by AI to provide details **verbally** during calls.

**Important**:
- `VAPI_PHONE_NUMBER_ID` must be the ID (starts with `ph_`), not the actual phone number
- Find this in Vapi Dashboard → Phone Numbers → Copy ID

### Step 6: Get Webhook URLs

After importing webhook-based workflows, you need to get their URLs:

#### BCS_Vapi_Webhook_Handler

1. Open `BCS_Vapi_Webhook_Handler.json` in n8n
2. Click on the "Vapi End-of-Call Webhook" node (first node)
3. Click **Test** or **Production** tab
4. Copy the webhook URL (e.g., `https://your-n8n.com/webhook/vapi-handler`)
5. Update n8n variable:
   ```
   WEBHOOK_VAPI_HANDLER=https://your-n8n.com/webhook/vapi-handler
   ```
6. Add this URL to Vapi Dashboard → Assistants → Server URL

**Note**: SMS webhook URLs (send-payment-sms, sms-status) are Phase 2 features

### Step 6: Activate Workflows

Activate workflows in this order:

1. **BCS_Vapi_Webhook_Handler** - Activate first (passive listener)
2. **BCS_Call_Scheduler** - Activate second (will start calling every 30 min!)
3. **BCS_Test_Webhook** - Keep inactive (manual testing only)

**To activate**:
1. Open the workflow
2. Toggle the switch in the top right from "Inactive" to "Active"
3. Confirm activation

**Warning**: Once `BCS_Call_Scheduler` is active, it will start making calls every 30 minutes to eligible debtors!

### Step 7: Test with BCS_Test_Webhook

Before activating `BCS_Call_Scheduler`, test the Vapi webhook handler:

1. Ensure `BCS_Vapi_Webhook_Handler` is active
2. Open `BCS_Test_Webhook` (keep it inactive)
3. Click **Execute Workflow** (manual test button)
4. Check the execution log for:
   - Status code 200
   - "Test webhook - debtor ready to pay via credit card" in notes
5. Verify in Google Sheets that test debtor (TEST-001) was updated:
   - last_outcome = "PROMISE_TO_PAY"
   - call_status = "completed"
   - next_action = "SCHEDULE_CALL"

## Post-Import Checklist (Phase 1)

- [ ] All 3 workflows imported successfully
- [ ] Google Sheets credential configured in workflows
- [ ] All 14 environment variables set in n8n
- [ ] Webhook URL configured in Vapi Dashboard
- [ ] BCS_Test_Webhook executed successfully
- [ ] Test debtor (TEST-001) exists in Google Sheets
- [ ] Manual test call completed (see TESTING_GUIDE.md)
- [ ] Payment details delivered verbally and confirmed
- [ ] All workflows activated in correct order

**Phase 2 Additions (when commissioned)**:
- [ ] Twilio account setup
- [ ] SMS workflows imported (Payment SMS + Status Handler)
- [ ] Email automation workflows imported

## Common Import Issues

### Issue 1: "Credential not found"

**Cause**: Placeholder credential IDs in JSON not replaced

**Fix**:
1. Open workflow editor
2. Click on node with red warning icon
3. Select credential from dropdown
4. Save workflow

### Issue 2: "Variable not found"

**Cause**: Environment variables not set in n8n

**Fix**:
1. Go to Settings → Variables
2. Add all 21 variables from `.env.example`
3. Re-execute workflow

### Issue 3: Webhook URL returns 404

**Cause**: Workflow not activated

**Fix**:
1. Open the workflow
2. Toggle to "Active"
3. Refresh webhook URL

### Issue 4: Google Sheets node fails with "Spreadsheet not found"

**Cause**: Incorrect `GOOGLE_SHEETS_SPREADSHEET_ID`

**Fix**:
1. Open Google Sheet in browser
2. Copy ID from URL: `https://docs.google.com/spreadsheets/d/{SPREADSHEET_ID}/edit`
3. Update n8n variable
4. Re-execute workflow

### Issue 5: Vapi API returns "phoneNumberId not found"

**Cause**: Using phone number instead of ID

**Fix**:
1. Go to Vapi Dashboard → Phone Numbers
2. Click on your number
3. Copy the ID (starts with `ph_`)
4. Update `VAPI_PHONE_NUMBER_ID` variable
5. Re-execute workflow

## Next Steps

After successful import:

1. Follow **TESTING_GUIDE.md** for comprehensive testing (7 phases, 40+ tests)
2. Review **TROUBLESHOOTING.md** for common runtime issues
3. Monitor first 10 calls closely using n8n execution logs
4. Set up error notifications (see WORKFLOW_ADAPTATION_PLAN.md Section 8)

## Support

If you encounter issues not covered in this guide:

1. Check **TROUBLESHOOTING.md** for detailed error solutions
2. Review n8n execution logs for specific error messages
3. Verify all prerequisites are met
4. Test each workflow independently before activating Call Scheduler

## Version Information

- Workflows Version: v1
- n8n Compatibility: v1.0+
- Google Sheets API: v4
- Twilio API: 2010-04-01
- Vapi API: Current (2025)
