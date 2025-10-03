# Implementation Checklist - BCS Debt Collection Bot

**Phase 1 Scope**: Voice-only outbound calling ($2,500 + $500 bonus)
**Phase 2 Scope**: SMS + Email automation ($2,000-$2,500) - Not included below

---

## Phase 1: Account Setup & Configuration (1.5 hours)

### 1.1 Vapi Account Setup (30 min)

- [ ] Go to https://vapi.ai and create account
- [ ] Navigate to Dashboard
- [ ] Click "Phone Numbers" → "Buy Number"
- [ ] Select Australia (+61) and purchase number
- [ ] Copy phone number ID to `.env` as `VAPI_PHONE_NUMBER`
- [ ] Click "Settings" → "API Keys" → "Create API Key"
- [ ] Copy API key to `.env` as `VAPI_API_KEY`
- [ ] Click "Webhooks" → "Generate Secret"
- [ ] Copy webhook secret to `.env` as `VAPI_WEBHOOK_SECRET`
- [ ] Save webhook URL (will add later after n8n setup)

**Validation**: Test API key with curl:
```bash
curl https://api.vapi.ai/assistant \
  -H "Authorization: Bearer $VAPI_API_KEY"
```

---

### 1.2 Google Sheets Setup (30 min)

- [ ] Go to https://sheets.google.com
- [ ] Click "Blank" to create new spreadsheet
- [ ] Rename to "BCS_Debtors"
- [ ] Open `BCS_Debtors_Template.csv` in this project
- [ ] Copy all content from CSV
- [ ] Paste into Google Sheet (cell A1)
- [ ] Format column L (timezone) as dropdown: AEST, ACST, AWST
- [ ] Format column M (call_status) as dropdown: (blank), calling, completed
- [ ] Format column U (assigned_to_ai) as checkbox (TRUE/FALSE)
- [ ] Format column W (do_not_call) as checkbox (TRUE/FALSE)
- [ ] Share with Peter (edit access): peter@brodiecollectionservices.com.au
- [ ] Copy spreadsheet ID from URL: `docs.google.com/spreadsheets/d/{SPREADSHEET_ID}/edit`
- [ ] Paste to `.env` as `GOOGLE_SHEETS_SPREADSHEET_ID`

**Validation**: Can you see 5 sample rows with data?

---

### 1.3 n8n Local Setup (30 min)

- [ ] Ensure Docker Desktop running
- [ ] Copy `.env.example` to `.env`
- [ ] Fill in all values from steps 1.1-1.3
- [ ] Create `docker-compose.yml` (if not exists):

```yaml
version: '3.8'

services:
  n8n:
    image: n8nio/n8n:latest
    ports:
      - "5678:5678"
    environment:
      - N8N_BASIC_AUTH_ACTIVE=true
      - N8N_BASIC_AUTH_USER=admin
      - N8N_BASIC_AUTH_PASSWORD=change_this_password
    env_file:
      - .env
    volumes:
      - n8n_data:/home/node/.n8n
    restart: unless-stopped

volumes:
  n8n_data:
```

- [ ] Run: `docker-compose up -d`
- [ ] Wait 30 seconds for n8n to start
- [ ] Open http://localhost:5678 in browser
- [ ] Login with credentials (admin / change_this_password)

**Validation**: Can you access n8n UI?

---

### 1.4 n8n Google Sheets Integration (30 min)

**Option A: OAuth (Recommended for single user)**
- [ ] In n8n: Click Settings → Credentials → Add Credential
- [ ] Select "Google Sheets OAuth2 API"
- [ ] Click "Connect my account"
- [ ] Authorize with your Google account
- [ ] Test connection

**Option B: Service Account**
- [ ] Go to https://console.cloud.google.com
- [ ] Create new project "BCS Debt Collection"
- [ ] Enable Google Sheets API
- [ ] Create Service Account
- [ ] Download JSON key file
- [ ] Copy JSON content to `.env` as `GOOGLE_SHEETS_CREDENTIALS`
- [ ] Share Google Sheet with service account email (edit access)

**Validation**: Create test workflow with Google Sheets node, try to read BCS_Debtors

---

## Phase 2: Vapi Assistant Configuration (2 hours)

### 2.1 Create 6 Vapi Assistants (1.5 hours)

**For each assistant** (repeat 6 times):

- [ ] Go to Vapi Dashboard → Assistants → Create Assistant
- [ ] Name: See `VAPI_ASSISTANT_CONFIGS.md` for names
- [ ] Model: OpenAI GPT-4o
- [ ] Voice: Azure `en-AU-NatashaNeural` (Australian female)
- [ ] System Prompt: Copy from `VAPI_ASSISTANT_CONFIGS.md`
- [ ] Add Functions: `log_outcome` and `verify_identity` (see configs)
- [ ] Add verbal payment instructions to system prompt (see VAPI_ASSISTANT_CONFIGS.md "Verbal Payment Instructions" section)
- [ ] Save assistant
- [ ] Copy Assistant ID to `.env` (VAPI_CONSUMER_1, VAPI_CONSUMER_2, etc.)

**Assistants to create**:
1. [ ] BCS Consumer - First Contact → `VAPI_CONSUMER_1`
2. [ ] BCS Consumer - Follow-up → `VAPI_CONSUMER_2`
3. [ ] BCS Consumer - Final Warning → `VAPI_CONSUMER_3`
4. [ ] BCS Commercial - First Contact → `VAPI_COMMERCIAL_1`
5. [ ] BCS Commercial - Follow-up → `VAPI_COMMERCIAL_2`
6. [ ] BCS Commercial - Final Warning → `VAPI_COMMERCIAL_3`

**Validation**: All 6 assistant IDs saved in `.env`?

---

### 2.2 Test One Assistant (30 min)

- [ ] Go to Vapi Dashboard → Test
- [ ] Select "BCS Consumer - First Contact"
- [ ] Click "Call my phone"
- [ ] Enter your phone number
- [ ] Answer call and test conversation
- [ ] Verify AI asks for date of birth (identity verification)
- [ ] Verify AI mentions debt amount and creditor
- [ ] End call
- [ ] Check if `log_outcome` function was called

**Validation**: Did you receive a call? Did AI sound natural?

---

## Phase 3: Build n8n Workflows (5.5 hours)

### 3.1 Workflow 1: BCS Call Scheduler (2 hours)

- [ ] In n8n: Click Workflows → Add Workflow
- [ ] Name: "BCS Call Scheduler"
- [ ] Add Cron Trigger node
  - Mode: Every 30 Minutes
- [ ] Add Google Sheets node "Read Rows"
  - Document: BCS_Debtors (select from dropdown)
  - Range: A2:W1000
- [ ] Add Code node "Filter & Validate"
  - Copy code from `WORKFLOW_ADAPTATION_PLAN.md` Section 3.3
- [ ] Add Split Into Items node
- [ ] Add Code node "Normalize Phone & Prepare Variables"
  - Copy code from Section 3.5
- [ ] Add Google Sheets node "Update call_status"
  - Operation: Update Row
  - Column M: "calling"
- [ ] Add HTTP Request node "Vapi API Call"
  - Method: POST
  - URL: https://api.vapi.ai/call/phone
  - Authentication: Header Auth
  - Header Name: Authorization
  - Header Value: `Bearer {{$vars.VAPI_API_KEY}}`
  - Body: `{{$node['Normalize Phone & Prepare Variables'].json.vapi_payload}}`
- [ ] Add IF node "Check HTTP Success"
- [ ] Add Google Sheets node "Reset call_status" (on error branch)
- [ ] Save workflow
- [ ] Test: Click "Execute Workflow" button

**Validation**: Check Google Sheets - did call_status update to "calling"?

---

### 3.2 Workflow 2: BCS Vapi Webhook Handler (3 hours)

- [ ] Create new workflow: "BCS Vapi Webhook Handler"
- [ ] Add Webhook Trigger node
  - HTTP Method: POST
  - Path: vapi-handler
- [ ] Copy webhook URL (e.g., http://localhost:5678/webhook/vapi-handler)
- [ ] Go to Vapi Dashboard → Settings → Webhooks
- [ ] Set Server URL to webhook URL
- [ ] Add Code node "Verify Signature"
  - Copy code from `WORKFLOW_ADAPTATION_PLAN.md` Section 4.2
- [ ] Add Code node "Parse Call Data"
  - Copy code from Section 4.3
- [ ] Add Code node "Calculate Next Action"
  - Copy code from Section 4.4 (Note: READY_TO_PAY now schedules follow-up in 3 days)
- [ ] Add Google Sheets node "Lookup Row"
  - Operation: Lookup
  - Lookup Column: A (debtor_id)
- [ ] Add Code node "Prepare Update Data"
  - Copy code from Section 4.6
- [ ] Add Google Sheets node "Update Row"
  - Update 12 columns (call_status through recording_url)
- [ ] Add IF node "Check if Manual Review"
- [ ] Add Email node "Notify Peter"
  - To: `{{$vars.BCS_EMAIL}}`
  - Subject: "BCS - Manual Review Required"
- [ ] Save workflow
- [ ] Mark as "Active" (production mode - listening for webhooks)

**Validation**: Use Workflow 3 (test webhook) to send mock payload

---

### 3.3 Workflow 3: BCS Test Webhook (30 min)

- [ ] Create new workflow: "BCS Test Webhook"
- [ ] Add Manual Trigger node
- [ ] Add Code node "Mock Vapi Payload"
  - Copy sample payload from `WORKFLOW_ADAPTATION_PLAN.md` Section 5.2
- [ ] Add HTTP Request node
  - Method: POST
  - URL: `{{$vars.WEBHOOK_VAPI_HANDLER}}`
  - Body: `{{$node['Mock Vapi Payload'].json}}`
- [ ] Save workflow
- [ ] Test: Click "Execute Workflow"
- [ ] Check Google Sheets - verify outcome logged

**Validation**: Test with multiple outcome types (PROMISE_TO_PAY, READY_TO_PAY, NO_ANSWER)

**Note**: SMS workflows (Payment SMS + SMS Status Handler) deferred to Phase 2

---

## Phase 4: Integration Testing (2 hours)

### 4.1 Unit Tests (30 min)

- [ ] Test phone normalization
  - Input: "0412 345 678" → Expected: "+61412345678"
  - Input: "+61 412 345 678" → Expected: "+61412345678"
  - Input: "61412345678" → Expected: "+61412345678"
- [ ] Test assistant selection
  - consumer + attempt 1 → VAPI_CONSUMER_1
  - commercial + attempt 3 → VAPI_COMMERCIAL_3
- [ ] Test outcome routing
  - PROMISE_TO_PAY → next_action = SCHEDULE_CALL, next_call_date = +7 days
  - READY_TO_PAY → next_action = AWAIT_PAYMENT, next_call_date = +3 days
  - NO_ANSWER → next_call_date = +4 hours

**Validation**: All tests pass?

---

### 4.2 Workflow Tests (1 hour)

- [ ] Test Workflow 1 (Call Scheduler)
  - Add test debtor with your phone number
  - Set next_call_date to current time
  - Execute workflow manually
  - Verify: Vapi call initiated
  - Verify: call_status updated to "calling"
- [ ] Test Workflow 2 (Webhook Handler)
  - Use Workflow 3 to send mock webhook
  - Verify: Signature validated
  - Verify: Outcome logged in Google Sheets
  - Verify: call_status updated to "completed"
- [ ] Test Workflow 3 (Test Webhook)
  - Test all 11 outcome types
  - Verify: Each outcome routed correctly

**Validation**: All workflows execute without errors?

---

### 4.3 End-to-End Test (30 min)

- [ ] Add real test debtor with your phone number
- [ ] Set next_call_date to current time + 5 minutes
- [ ] Wait for Workflow 1 cron to trigger (or execute manually)
- [ ] Answer call
- [ ] Interact with AI:
  - Verify identity (provide fake DOB)
  - Listen to debt details
  - Say "I'd like to pay by bank transfer"
  - Confirm you wrote down BSB, account, reference
- [ ] End call
- [ ] Check Google Sheets:
  - [ ] last_call_date populated
  - [ ] last_outcome = "READY_TO_PAY"
  - [ ] call_status = "completed"
  - [ ] attempt_number incremented
  - [ ] notes contain "payment details provided verbally"
  - [ ] next_action = "AWAIT_PAYMENT"
  - [ ] next_call_date = +3 days from now

**Validation**: Full flow works end-to-end?

---

## Phase 5: Compliance Testing (1 hour)

### 5.1 Business Hours Validation (20 min)

- [ ] Add test debtor with timezone = AWST
- [ ] Set current time to 6:00am AWST (before 7:30am)
- [ ] Execute Workflow 1
- [ ] Verify: Debtor NOT called (filtered out by business hours check)

**Validation**: Business hours enforced?

---

### 5.2 Do-Not-Call Validation (20 min)

- [ ] Add test debtor
- [ ] Set do_not_call = TRUE
- [ ] Execute Workflow 1
- [ ] Verify: Debtor NOT called (filtered out by do_not_call check)

**Validation**: Do-not-call respected?

---

### 5.3 Max Attempts Validation (20 min)

- [ ] Add test debtor
- [ ] Set attempt_number = 4
- [ ] Execute Workflow 1
- [ ] Verify: Debtor NOT called (filtered out by max attempts check)

**Validation**: Max 3 attempts enforced?

---

## Phase 6: Production Deployment (Optional)

### 6.1 DigitalOcean Deployment

- [ ] Create DigitalOcean Droplet (Ubuntu 22.04, $12/month)
- [ ] SSH into droplet
- [ ] Install Docker and Docker Compose
- [ ] Clone project repository
- [ ] Copy `.env` file
- [ ] Update webhook URLs to production domain
- [ ] Run `docker-compose up -d`
- [ ] Configure firewall (UFW)
- [ ] Set up SSL with Let's Encrypt
- [ ] Update Vapi webhook URL to production

---

## Final Checklist (Phase 1)

- [ ] All 3 workflows active and working
- [ ] 6 Vapi assistants configured with verbal payment instructions
- [ ] Google Sheet with 23 columns
- [ ] All environment variables set (14 variables)
- [ ] First test call completed successfully
- [ ] Payment details delivered verbally and confirmed by tester
- [ ] Compliance checks validated
- [ ] Peter has edit access to Google Sheet
- [ ] Documentation shared with Peter

**Phase 2 Additions (When Commissioned)**:
- [ ] Twilio account setup
- [ ] SMS workflows (Payment SMS + Status Handler) added
- [ ] Email automation workflows added
- [ ] Management dashboard created

---

## Troubleshooting

If you encounter issues, see `TROUBLESHOOTING.md` for common errors and solutions.

---

**Estimated Total Time (Phase 1)**: 10.5 hours

**Status**: Ready for implementation
**Last Updated**: October 3, 2025
