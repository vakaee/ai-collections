# Technical Architecture

## Document Information

**Project**: AI Debt Collection Bot - Phase 1
**Version**: 2.0 (Updated with 5 workflows, 23-column schema, security measures)
**Last Updated**: October 3, 2025
**Author**: Vlad Petoukhov

---

## 1. System Overview

### 1.1 High-Level Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    CRM DATABASE                             │
│  ┌──────────────┐  ┌────────────────┐  ┌─────────────────┐ │
│  │ Debtor       │  │ Debt Accounts  │  │ Payment History │ │
│  │ Profiles     │  │                │  │                 │ │
│  └──────────────┘  └────────────────┘  └─────────────────┘ │
└──────────────────┬──────────────────────────────────────────┘
                   │ (Read/Write)
                   ↓
┌─────────────────────────────────────────────────────────────┐
│              BACKEND ORCHESTRATION LAYER                    │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  API Server (Node.js + Express)                        │ │
│  │  - Call scheduling & queue management                  │ │
│  │  - Strategy router (consumer vs commercial)            │ │
│  │  - Webhook handler (Vapi callbacks)                    │ │
│  │  - CRM sync (read debtors, write outcomes)             │ │
│  └────────────────────────────────────────────────────────┘ │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  Database (PostgreSQL)                                 │ │
│  │  - ai_call_logs (call history, transcripts)            │ │
│  │  - call_queue (scheduled calls, retries)               │ │
│  │  - system_config (scripts, templates)                  │ │
│  └────────────────────────────────────────────────────────┘ │
└──────────────────┬──────────────────────────────────────────┘
                   │ (API calls via HTTPS)
                   ↓
┌─────────────────────────────────────────────────────────────┐
│              VAPI AI VOICE PLATFORM                         │
│  - Telephony (outbound calling via Twilio/Telnyx)          │
│  - Speech-to-Text (real-time transcription)                │
│  - LLM Integration (OpenAI GPT-4o)                         │
│  - Text-to-Speech (natural voice synthesis)                │
│  - Call recording & storage                                │
│  - Function calling (structured outcomes)                  │
└──────────────────┬──────────────────────────────────────────┘
                   │ (Phone call via PSTN)
                   ↓
┌─────────────────────────────────────────────────────────────┐
│                    DEBTOR                                   │
│  (Receives phone call, interacts with AI voice agent)      │
└─────────────────────────────────────────────────────────────┘
```

---

## 2. Technology Stack

### 2.1 Core Components

| Component | Technology | Rationale |
|-----------|-----------|-----------|
| **Orchestration** | n8n (workflow automation) | Visual workflows, faster development, existing boilerplate available |
| **Database** | PostgreSQL | Reliable, ACID-compliant, good for structured data |
| **Telephony** | Vapi.ai | Native AI voice agent platform, faster development |
| **LLM** | OpenAI GPT-4o | Low latency, proven voice conversation handling |
| **SMS** | Twilio | Payment details delivery, native n8n integration |
| **Deployment** | Docker Compose | Containerized deployment, easy to replicate and scale |
| **Hosting** | DigitalOcean Droplet | Affordable ($6-12/month), supports Docker, full control |
| **CRM (Interim)** | Google Sheets | **STOPGAP**: Quick start, migrate to real CRM later |

### 2.2 Alternative Considerations

**Telephony: Twilio vs. Vapi**
- **Twilio**: Lower-level control, $0.013/min, requires custom conversation layer
- **Vapi**: AI-native, $0.05-0.09/min, built-in conversation management
- **Decision**: Use Vapi for faster development (Phase 1 timeline is tight)

**LLM: OpenAI vs. Anthropic Claude**
- **OpenAI GPT-4o**: Faster response times, proven voice latency
- **Claude 3.5 Sonnet**: Better reasoning, but higher latency
- **Decision**: GPT-4o primary, Claude for complex edge cases if needed

**Orchestration: n8n vs. Custom Node.js Backend**
- **n8n**: Visual workflow builder, built-in integrations, faster development
- **Custom Node.js**: More control, but slower development
- **Decision**: n8n (developer has existing boilerplate, timeline is tight, client can maintain workflows)
- **Note**: This project is primarily orchestration (schedule → call → log → update CRM), which is n8n's core strength

**Hosting: DigitalOcean vs. Railway/Render**
- **DigitalOcean Droplet**: Cheap ($6/month), Docker support, full control
- **Railway/Render**: Easier deployment, auto-scaling, more expensive
- **Decision**: DigitalOcean with Docker Compose (cost-effective, suitable for this scale)

**CRM: Google Sheets (Interim) vs. Real Database**
- **Google Sheets**: Zero setup time, easy for client to view/edit, n8n native integration
- **Real CRM Database**: Requires access credentials, schema documentation, potential delays
- **Decision**: Google Sheets for Phase 1 (STOPGAP), migrate to real CRM later
- **Migration path**: Swap Google Sheets node with PostgreSQL/MySQL node in n8n workflows

**SMS: Twilio for Payment Delivery**
- **Why**: Send payment links/details via SMS instead of verbal-only instructions
- **Benefits**: Higher conversion (immediate action), no transcription errors, persistent reference
- **Cost**: ~$0.08 per SMS to Australia
- **Integration**: Native n8n Twilio node

---

## 3. Data Architecture

### 3.1 Google Sheets Schema (Interim CRM) ⚠️ STOPGAP SOLUTION

**Sheet Name**: "BCS_Debtors"

**Total: 23 columns** (includes call tracking, compliance, and security fields)

| Column | Data Type | Purpose |
|--------|-----------|---------|
| debtor_id | Number | Unique identifier (auto-increment) |
| name | Text | Debtor name (individual or company) |
| phone | Text | Phone number (E.164 format: +61...) |
| email | Text | Email address |
| debtor_type | Text | "consumer" or "commercial" |
| amount | Number | Debt amount (dollars) |
| creditor | Text | Creditor name |
| invoice_number | Text | Invoice/account reference |
| debt_date | Date | Debt origin date |
| dob | Date | Date of birth (for consumer verification) |
| abn | Text | ABN (for commercial debtors) |
| timezone | Text | AEST/ACST/AWST (for business hours compliance) |
| call_status | Text | null/"calling"/"completed" (prevents duplicate calls) |
| call_id | Text | Vapi call ID (for tracking/debugging) |
| last_call_date | Timestamp | Last call attempt |
| last_outcome | Text | PROMISE_TO_PAY, DISPUTE_RAISED, etc. |
| next_action | Text | SCHEDULE_CALL, SEND_EMAIL, MANUAL_REVIEW |
| next_call_date | Timestamp | Scheduled next call |
| attempt_number | Number | 1, 2, or 3 (call progression) |
| notes | Text | Free-form notes from calls |
| assigned_to_ai | Boolean | TRUE = AI should call, FALSE = exclude |
| payment_status | Text | unpaid, paid, disputed, etc. |
| do_not_call | Boolean | TRUE = debtor requested no contact (compliance) |
| recording_url | Text | Vapi call recording link |

**n8n Integration**:
- **Read**: Google Sheets node queries rows (A2:W1000) where `assigned_to_ai=TRUE`, `payment_status=unpaid`, `call_status!="calling"`, `do_not_call!=TRUE`
- **Write**: Google Sheets node updates 12 columns after call with outcome, call_id, recording_url, next_action, notes, etc.
- **Security**: Webhook signature verification prevents unauthorized data updates

**Migration Path**:
When Peter provides CRM access, replace Google Sheets node with PostgreSQL/MySQL node. Data can be exported to CSV and imported to real CRM.

---

### 3.2 PostgreSQL Database Schema (AI Call Logs)

#### **Table: ai_call_logs**
Stores every call made by the AI system.

```sql
CREATE TABLE ai_call_logs (
  id SERIAL PRIMARY KEY,
  call_id VARCHAR(255) UNIQUE NOT NULL,  -- Vapi call ID
  debtor_id INTEGER NOT NULL,            -- Foreign key to CRM debtor table
  call_timestamp TIMESTAMP NOT NULL,
  call_duration INTEGER,                 -- seconds
  outcome VARCHAR(50) NOT NULL,          -- PAYMENT_RECEIVED, DISPUTE_RAISED, etc.
  next_action VARCHAR(50),               -- SEND_EMAIL, SCHEDULE_CALL, MANUAL_REVIEW
  next_call_date TIMESTAMP,
  transcript TEXT,
  recording_url VARCHAR(500),
  metadata JSONB,                        -- Additional structured data
  created_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_debtor_id ON ai_call_logs(debtor_id);
CREATE INDEX idx_call_timestamp ON ai_call_logs(call_timestamp);
CREATE INDEX idx_outcome ON ai_call_logs(outcome);
```

#### **Table: call_queue**
Manages scheduled and pending calls.

```sql
CREATE TABLE call_queue (
  id SERIAL PRIMARY KEY,
  debtor_id INTEGER NOT NULL,
  scheduled_time TIMESTAMP NOT NULL,
  attempt_number INTEGER DEFAULT 1,      -- 1st, 2nd, 3rd attempt
  status VARCHAR(20) DEFAULT 'pending',  -- pending, in_progress, completed, failed
  priority VARCHAR(20) DEFAULT 'normal', -- normal, high, urgent
  debtor_type VARCHAR(20) NOT NULL,      -- consumer, commercial
  last_outcome VARCHAR(50),
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_scheduled_time ON call_queue(scheduled_time);
CREATE INDEX idx_status ON call_queue(status);
```

#### **Table: system_config**
Stores conversation scripts, templates, and settings (editable without code changes).

```sql
CREATE TABLE system_config (
  id SERIAL PRIMARY KEY,
  config_key VARCHAR(100) UNIQUE NOT NULL,
  config_value TEXT NOT NULL,
  description TEXT,
  updated_at TIMESTAMP DEFAULT NOW()
);

-- Example rows:
-- config_key = 'script_consumer_call_1', config_value = '{script content}'
-- config_key = 'script_commercial_call_2', config_value = '{script content}'
-- config_key = 'business_hours_start', config_value = '07:30'
-- config_key = 'business_hours_end', config_value = '21:00'
```

---

### 3.2 CRM Integration

**Approach 1: Direct Database Access**
- Read debtor records from existing CRM database
- Write call outcomes to new `ai_call_logs` table in CRM
- Pros: Simplest, no API development needed
- Cons: Tight coupling, requires DB credentials

**Approach 2: REST API**
- Peter's brother builds lightweight API endpoints:
  - `GET /debtors/:id` - Fetch debtor details
  - `POST /calls` - Log call outcome
- Pros: Decoupled, more secure
- Cons: Requires Peter's brother's time

**Decision**: Approach 1 initially (pending CRM access), migrate to Approach 2 if needed.

---

## 4. Conversation Engine

### 4.1 Vapi Configuration

**Vapi Assistant Schema**:

```json
{
  "name": "BCS Debt Collection Agent",
  "voice": {
    "provider": "azure",
    "voiceId": "en-AU-NatashaNeural"  // Australian accent
  },
  "model": {
    "provider": "openai",
    "model": "gpt-4o",
    "temperature": 0.3,  // Low temperature for consistency
    "systemPrompt": "You are calling on behalf of Brodie Collection Services..."
  },
  "functions": [
    {
      "name": "log_outcome",
      "description": "Log the outcome of the call",
      "parameters": {
        "outcome": "string (enum)",
        "next_action": "string (optional)",
        "notes": "string (optional)"
      }
    },
    {
      "name": "verify_identity",
      "description": "Verify debtor identity before proceeding",
      "parameters": {
        "dob": "string",
        "last_4_phone": "string"
      }
    }
  ],
  "endCallPhrases": ["goodbye", "have a good day"],
  "recordingEnabled": true
}
```

---

### 4.2 System Prompts

**Consumer Debtor (1st Call)**:

```
You are calling on behalf of Brodie Collection Services, a licensed debt collection agency in Australia. Your name is Sarah.

MANDATORY OPENING (say this verbatim):
"Hello, this is Sarah calling from Brodie Collection Services. This call is regarding a debt collection matter and may be recorded for quality and compliance purposes. May I please speak with [DEBTOR_NAME]?"

If the person confirms they are [DEBTOR_NAME], proceed with identity verification:
"For security purposes, can you please confirm your date of birth?"

If they provide the correct DOB ([DOB_FROM_CRM]), proceed:

"Thank you. I'm calling regarding an outstanding debt of $[AMOUNT] owed to [CREDITOR_NAME] from [DATE]. Our records show this account remains unpaid. We kindly request full payment within 7 days to avoid further action."

"You can make payment via:
- Bank transfer to [BSB] [ACCOUNT_NUMBER], reference [REFERENCE]
- Credit card by calling [PHONE_NUMBER]
- Cheque mailed to [ADDRESS]"

OBJECTION HANDLING:
- If they say "I dispute this debt": "I understand. To formally dispute this, please submit your dispute in writing to disputes@brodiecollectionservices.com.au within 7 days. I'll make a note of your dispute in our system."
- If they say "I can't afford to pay" or "financial hardship": "I understand you're experiencing financial difficulty. I can send you a hardship form to complete. What email address should I send it to?"
- If they ask for a payment plan: "I can note your request for a payment arrangement. What amount can you afford to pay, and how frequently?"
- If they promise to pay: "Thank you. When can we expect your payment?" [Log the date]

ESCALATION TRIGGERS (transfer to human immediately):
- "lawyer", "solicitor", "complaint", "ombudsman"
- Abusive language or threats
- Deceased debtor notification

End the call politely: "Thank you for your time. If you have any questions, please contact us at [PHONE_NUMBER]. Goodbye."

Call the function `log_outcome` with the appropriate outcome before ending the call.
```

**Commercial Debtor (1st Call)**:

```
[Similar structure, but with these changes:]

- More formal tone
- Reference "your company" instead of "you"
- Consequences: "Failure to pay may result in a payment default being listed with ASIC, which will permanently affect your company's credit rating and ability to secure finance."
- No hardship option (remove that objection handler)
- Emphasize director liability: "Please note that directors may be personally liable for company debts in certain circumstances."
```

---

### 4.3 Conversation Flow State Machine

```
START
  ↓
GREETING (identity disclosure + purpose statement)
  ↓
REQUEST_DEBTOR (ask for debtor by name)
  ↓
[If wrong person] → END_CALL (log WRONG_NUMBER)
[If correct person] → VERIFY_IDENTITY
  ↓
VERIFY_IDENTITY (request DOB or last 4 of phone)
  ↓
[If verification fails 2x] → ESCALATE_HUMAN
[If verification succeeds] → STATE_DEBT
  ↓
STATE_DEBT (amount owed, creditor, date)
  ↓
REQUEST_PAYMENT (7-day deadline, payment methods)
  ↓
HANDLE_RESPONSE
  ↓
[Agrees to pay] → LOG_OUTCOME (PROMISE_TO_PAY) → END_CALL
[Disputes] → SEND_DISPUTE_EMAIL → LOG_OUTCOME (DISPUTE_RAISED) → END_CALL
[Hardship] → SEND_HARDSHIP_FORM → LOG_OUTCOME (HARDSHIP_CLAIMED) → END_CALL
[Negotiates] → LOG_OUTCOME (PAYMENT_PLAN_REQUESTED) → END_CALL
[Escalation trigger] → TRANSFER_HUMAN
[Hangs up] → LOG_OUTCOME (HUNG_UP)
```

---

## 5. n8n Workflow Design

### 5.1 Workflow Overview

n8n will handle all orchestration via visual workflows. No custom API code required.

**5 workflows total** (Phase 1) - adapted from existing Airbnb workflows:
1. BCS Call Scheduler (every 30 min, Google Sheets → Vapi)
2. BCS Vapi Webhook Handler (processes outcomes, routes actions)
3. BCS Send Payment SMS (Twilio SMS with payment details)
4. BCS SMS Status Handler (tracks delivery confirmations)
5. BCS Test Webhook (testing tool for development)

**Key Security & Reliability Features**:
- Webhook signature verification (HMAC SHA-256)
- Duplicate call prevention (`call_status` column)
- Phone normalization (E.164 format)
- Business hours enforcement (timezone-aware)
- Compliance flags (`do_not_call`, max 3 attempts)

---

**Workflow 1: BCS Call Scheduler (Cron-based)**
```
[Cron Trigger - Every 30 minutes]
  ↓
[Google Sheets - Read BCS_Debtors (A2:W1000)]
  ↓
[Code Node - Filter & Validate]
  - assigned_to_ai = TRUE
  - payment_status = unpaid
  - call_status != "calling" (prevents duplicates)
  - do_not_call != TRUE (compliance)
  - attempt_number <= 3 (max attempts)
  - next_call_date <= now
  - Business hours check (7:30am-9pm, timezone-aware)
  ↓
[Split Into Items]
  ↓
[Code Node - Normalize Phone & Prepare Variables]
  - Convert phone to E.164 (+61...)
  - Select assistant ID (6 options: consumer/commercial × 1st/2nd/3rd)
  - Build variableValues: debtor_id, name, amount, creditor, dob, abn
  ↓
[Google Sheets - Update call_status = "calling"]
  ↓
[HTTP Request - Vapi API: POST /call]
  ↓
[Error Handler - If call fails]
  └── [Google Sheets - Reset call_status to null]
```

---

**Workflow 2: BCS Vapi Webhook Handler**
```
[Webhook Trigger - POST from Vapi]
  ↓
[Code Node - Verify Signature]
  - HMAC SHA-256 with VAPI_WEBHOOK_SECRET
  - Reject unauthorized requests
  ↓
[Code Node - Parse Call Data]
  - Extract: debtor_id, call_id, outcome, payment_method, notes, recording_url
  - Handle failed calls (status="failed")
  ↓
[Code Node - Calculate Next Action]
  - 11 outcome types with routing logic:
    * PROMISE_TO_PAY → schedule follow-up (7 days)
    * READY_TO_PAY → trigger payment SMS
    * NO_ANSWER / VOICEMAIL → retry in 4 hours
    * WRONG_NUMBER → manual review, stop AI
    * DISPUTE_RAISED / HARDSHIP_CLAIMED → manual review
    * REFUSED_TO_ENGAGE → wait 1 week
    * REQUESTED_NO_CONTACT → set do_not_call=TRUE
    * ALREADY_PAID → mark paid
    * CALL_FAILED → retry in 2 hours
    * OBJECTED_TO_RECORDING → manual review
  ↓
[Google Sheets - Lookup Row by debtor_id]
  ↓
[Code Node - Prepare Update Data]
  - Increment attempt_number
  - Append timestamped notes
  ↓
[Google Sheets - Update Row]
  - 12 columns: call_status, call_id, recording_url, last_call_date,
    last_outcome, next_action, next_call_date, attempt_number, notes,
    assigned_to_ai, payment_status, do_not_call
  ↓
[IF - outcome = READY_TO_PAY or PROMISE_TO_PAY?]
  ↓ (yes)
  [HTTP Request - Trigger BCS Send Payment SMS]
  ↓
[IF - next_action = MANUAL_REVIEW?]
  ↓ (yes)
  [Email - Notify Peter]
```

---

**Workflow 3: BCS Send Payment SMS**
```
[Webhook Trigger - POST from Workflow 2]
  - Payload: { debtor_id, payment_method }
  ↓
[Google Sheets - Lookup Debtor]
  ↓
[Code Node - Build SMS Body]
  - credit_card → Stripe payment link
  - bank_transfer → BSB, account, reference
  - cheque → Mailing address
  ↓
[Twilio - Send SMS]
  - Status Callback: WEBHOOK_SMS_STATUS
  ↓
[Google Sheets - Update Notes]
  - Log: "Payment SMS sent via {method}"
```

---

**Workflow 4: BCS SMS Status Handler**
```
[Webhook Trigger - POST from Twilio]
  - Payload: { MessageStatus, To, ErrorCode }
  ↓
[Code Node - Parse Status]
  - Extract: phone, status (delivered/failed), error
  ↓
[Google Sheets - Lookup Row by Phone]
  ↓
[Google Sheets - Update Notes]
  - Log: "SMS delivered" or "SMS failed (Error: {code})"
```

---

**Workflow 5: BCS Test Webhook (Testing)**
```
[Manual Trigger]
  ↓
[Code Node - Mock Vapi Payload]
  - Sample outcomes (PROMISE_TO_PAY, READY_TO_PAY, etc.)
  ↓
[HTTP Request - POST to Workflow 2]
  - Test outcome logic without real calls
```

---

### 5.2 Vapi API Integration (n8n HTTP Request Node)

**Initiate Call** (n8n HTTP Request node configuration):

```json
{
  "method": "POST",
  "url": "https://api.vapi.ai/call",
  "authentication": "headerAuth",
  "headers": {
    "Authorization": "Bearer {{$env.VAPI_API_KEY}}"
  },
  "body": {
    "assistantId": "{{$node['Determine Assistant'].json.assistantId}}",
    "phoneNumberId": "{{$env.VAPI_PHONE_NUMBER_ID}}",
    "customer": {
      "number": "{{$node['Get Debtor'].json.phone}}"
    },
    "assistantOverrides": {
      "variableValues": {
        "debtor_name": "{{$node['Get Debtor'].json.name}}",
        "amount": "{{$node['Get Debtor'].json.amount}}",
        "creditor": "{{$node['Get Debtor'].json.creditor}}",
        "dob": "{{$node['Get Debtor'].json.dob}}"
      }
    }
  }
}
```

---

### 5.3 n8n Code Nodes (Custom Logic)

**Parse Vapi Outcome** (Code node in Workflow 2):

```javascript
// Extract outcome from Vapi function call
const functionCalls = $input.item.json.message?.functionCall || [];
const logOutcomeCall = functionCalls.find(fc => fc.name === 'log_outcome');

if (!logOutcomeCall) {
  return { outcome: 'UNKNOWN', next_action: 'MANUAL_REVIEW' };
}

const { outcome, next_action, notes } = logOutcomeCall.arguments;

return {
  outcome,
  next_action,
  notes,
  call_id: $input.item.json.call.id,
  debtor_id: $input.item.json.call.customer.number,
  duration: $input.item.json.call.endedAt - $input.item.json.call.startedAt,
  transcript: $input.item.json.transcript,
  recording_url: $input.item.json.recordingUrl
};
```

---

**Determine Assistant ID** (Code node in Workflow 1):

```javascript
// Select correct Vapi assistant based on debtor type and attempt number
const debtorType = $input.item.json.debtor_type; // 'consumer' or 'commercial'
const attemptNumber = $input.item.json.attempt_number; // 1, 2, or 3

const assistantMap = {
  'consumer_1': process.env.VAPI_ASSISTANT_CONSUMER_1,
  'consumer_2': process.env.VAPI_ASSISTANT_CONSUMER_2,
  'consumer_3': process.env.VAPI_ASSISTANT_CONSUMER_3,
  'commercial_1': process.env.VAPI_ASSISTANT_COMMERCIAL_1,
  'commercial_2': process.env.VAPI_ASSISTANT_COMMERCIAL_2,
  'commercial_3': process.env.VAPI_ASSISTANT_COMMERCIAL_3
};

const key = `${debtorType}_${attemptNumber}`;
return { assistantId: assistantMap[key] };
```

---

**Business Hours Check** (Code node in Workflow 1):

```javascript
// Check if current time is within business hours
const now = new Date();
const day = now.getDay(); // 0 = Sunday, 6 = Saturday
const hour = now.getHours();
const minute = now.getMinutes();

const isWeekday = day >= 1 && day <= 5;
const isWeekend = day === 0 || day === 6;

const weekdayStart = 7.5; // 7:30am
const weekdayEnd = 21; // 9:00pm
const weekendStart = 9;
const weekendEnd = 21;

const currentTime = hour + minute / 60;

let withinHours = false;

if (isWeekday) {
  withinHours = currentTime >= weekdayStart && currentTime < weekdayEnd;
} else if (isWeekend) {
  withinHours = currentTime >= weekendStart && currentTime < weekendEnd;
}

return { withinHours };
```

---

### 5.4 Twilio SMS Integration (Payment Delivery)

**SMS Templates** (n8n Twilio node configuration):

**Credit Card Payment SMS**:
```javascript
// Code node to build SMS body
const debtor = $input.item.json;
const paymentLink = process.env.STRIPE_PAYMENT_LINK; // or dynamic link per debtor

return {
  to: debtor.phone,
  body: `Brodie Collection Services

Pay $${debtor.amount} now via credit card:
${paymentLink}

Questions? Call ${process.env.BCS_PHONE}`
};
```

**Bank Transfer SMS**:
```javascript
const debtor = $input.item.json;

return {
  to: debtor.phone,
  body: `Brodie Collection Services
Bank Transfer Details:

BSB: ${process.env.BSB}
Account: ${process.env.ACCOUNT_NUMBER}
Account Name: ${process.env.ACCOUNT_NAME}
Reference: ${debtor.debtor_id}
Amount: $${debtor.amount}

Questions? Call ${process.env.BCS_PHONE}`
};
```

**Cheque Payment SMS**:
```javascript
const debtor = $input.item.json;

return {
  to: debtor.phone,
  body: `Brodie Collection Services
Mail cheque to:

${process.env.BCS_ADDRESS}

Payable to: Brodie Collection Services
Amount: $${debtor.amount}
Reference: ${debtor.debtor_id}

Questions? Call ${process.env.BCS_PHONE}`
};
```

**n8n Twilio Node Setup**:
- **Account SID**: From Twilio dashboard
- **Auth Token**: From Twilio dashboard
- **From Number**: Twilio-provided Australian number
- **To Number**: `{{$node['Previous'].json.phone}}`
- **Message Body**: `{{$node['Build SMS'].json.body}}`

**Cost**: ~$0.08 per SMS to Australian mobile numbers

---

## 6. Deployment Architecture

### 6.1 Docker Deployment on DigitalOcean

**Deployment Strategy**: Docker Compose on DigitalOcean Droplet

**Why DigitalOcean**:
- Cost-effective ($6/month for Basic Droplet, $12/month for better performance)
- Full control over infrastructure
- Supports Docker natively
- Easy to scale vertically (upgrade Droplet) or horizontally (add load balancer)
- Simple firewall and networking configuration

**Docker Compose Stack**:
```yaml
version: '3.8'

services:
  n8n:
    image: n8nio/n8n:latest
    ports:
      - "5678:5678"
    environment:
      - N8N_BASIC_AUTH_ACTIVE=true
      - N8N_BASIC_AUTH_USER=${N8N_USER}
      - N8N_BASIC_AUTH_PASSWORD=${N8N_PASSWORD}
      - N8N_HOST=${N8N_HOST}
      - N8N_PORT=5678
      - N8N_PROTOCOL=https
      - WEBHOOK_URL=https://${N8N_HOST}
      - DB_TYPE=postgresdb
      - DB_POSTGRESDB_HOST=postgres
      - DB_POSTGRESDB_PORT=5432
      - DB_POSTGRESDB_DATABASE=${POSTGRES_DB}
      - DB_POSTGRESDB_USER=${POSTGRES_USER}
      - DB_POSTGRESDB_PASSWORD=${POSTGRES_PASSWORD}
      - VAPI_API_KEY=${VAPI_API_KEY}
      - VAPI_ASSISTANT_CONSUMER_1=${VAPI_ASSISTANT_CONSUMER_1}
      - VAPI_ASSISTANT_CONSUMER_2=${VAPI_ASSISTANT_CONSUMER_2}
      - VAPI_ASSISTANT_CONSUMER_3=${VAPI_ASSISTANT_CONSUMER_3}
      - VAPI_ASSISTANT_COMMERCIAL_1=${VAPI_ASSISTANT_COMMERCIAL_1}
      - VAPI_ASSISTANT_COMMERCIAL_2=${VAPI_ASSISTANT_COMMERCIAL_2}
      - VAPI_ASSISTANT_COMMERCIAL_3=${VAPI_ASSISTANT_COMMERCIAL_3}
      - SMTP_HOST=${SMTP_HOST}
      - SMTP_USER=${SMTP_USER}
      - SMTP_PASSWORD=${SMTP_PASSWORD}
    volumes:
      - n8n_data:/home/node/.n8n
    depends_on:
      - postgres
    restart: unless-stopped

  postgres:
    image: postgres:15-alpine
    environment:
      - POSTGRES_USER=${POSTGRES_USER}
      - POSTGRES_PASSWORD=${POSTGRES_PASSWORD}
      - POSTGRES_DB=${POSTGRES_DB}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    restart: unless-stopped

  # Optional: Nginx reverse proxy for SSL
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf
      - ./ssl:/etc/nginx/ssl
    depends_on:
      - n8n
    restart: unless-stopped

volumes:
  n8n_data:
  postgres_data:
```

**Deployment Steps**:

1. **Provision DigitalOcean Droplet**:
   ```bash
   # Create Droplet via DO dashboard or API
   # Ubuntu 22.04 LTS, $12/month (2 GB RAM, 1 CPU)
   # Enable monitoring and backups
   ```

2. **SSH into Droplet and install Docker**:
   ```bash
   ssh root@your-droplet-ip

   # Install Docker
   curl -fsSL https://get.docker.com -o get-docker.sh
   sh get-docker.sh

   # Install Docker Compose
   apt install docker-compose -y
   ```

3. **Clone repository and configure**:
   ```bash
   git clone https://github.com/yourusername/bcs-debt-bot.git
   cd bcs-debt-bot

   # Create .env file
   cp .env.example .env
   nano .env  # Fill in secrets
   ```

4. **Start services**:
   ```bash
   docker-compose up -d

   # Verify services running
   docker-compose ps
   ```

5. **Configure firewall**:
   ```bash
   ufw allow 22    # SSH
   ufw allow 80    # HTTP
   ufw allow 443   # HTTPS
   ufw enable
   ```

6. **Set up SSL** (Let's Encrypt via Certbot):
   ```bash
   apt install certbot python3-certbot-nginx -y
   certbot --nginx -d yourdomain.com
   ```

---

### 6.2 Alternative: DigitalOcean App Platform

If you prefer managed deployment (more expensive but easier):

**Pros**:
- No server management
- Auto-scaling
- Built-in SSL
- GitHub auto-deploy

**Cons**:
- $15-30/month (vs $6-12 for Droplet)
- Less control

**When to use**: If budget allows and you want minimal DevOps

---

### 6.2 Environment Variables

```
# Database
DATABASE_URL=postgresql://user:pass@host:5432/db

# Vapi
VAPI_API_KEY=xxx
VAPI_ASSISTANT_ID_CONSUMER_1=xxx
VAPI_ASSISTANT_ID_CONSUMER_2=xxx
VAPI_ASSISTANT_ID_CONSUMER_3=xxx
VAPI_ASSISTANT_ID_COMMERCIAL_1=xxx
VAPI_ASSISTANT_ID_COMMERCIAL_2=xxx
VAPI_ASSISTANT_ID_COMMERCIAL_3=xxx

# OpenAI (if direct calls needed)
OPENAI_API_KEY=xxx

# CRM
CRM_DB_HOST=xxx
CRM_DB_USER=xxx
CRM_DB_PASSWORD=xxx

# Email (for sending dispute/hardship forms)
SMTP_HOST=xxx
SMTP_USER=xxx
SMTP_PASSWORD=xxx
EMAIL_FROM=noreply@brodiecollectionservices.com.au
DISPUTES_EMAIL=disputes@brodiecollectionservices.com.au

# Twilio (for SMS payment delivery)
TWILIO_ACCOUNT_SID=xxx
TWILIO_AUTH_TOKEN=xxx
TWILIO_FROM_NUMBER=+61xxxxxxxxx

# Payment Information
STRIPE_PAYMENT_LINK=https://buy.stripe.com/xxxxx  # or dynamic
BSB=XXX-XXX
ACCOUNT_NUMBER=XXXXXXXXX
ACCOUNT_NAME=Brodie Collection Services
BCS_PHONE=+61x xxxx xxxx
BCS_ADDRESS=123 Example St, Sydney NSW 2000

# Business hours (AEST/ACST)
BUSINESS_HOURS_START=07:30
BUSINESS_HOURS_END=21:00
TIMEZONE=Australia/Sydney

# Compliance
MAX_CALLS_PER_DAY=3
MIN_HOURS_BETWEEN_CALLS=2
```

---

## 7. Security & Compliance

### 7.1 Data Protection

**Encryption at Rest**:
- PostgreSQL with encrypted volumes (Railway/Render default)
- Call recordings stored in Vapi (encrypted by default)

**Encryption in Transit**:
- HTTPS for all API calls (TLS 1.3)
- SRTP for voice calls (Vapi default)

**Access Control**:
- Database credentials stored in environment variables (not in code)
- API authentication (JWT tokens for future dashboard)
- Vapi API key restricted to backend server IP

---

### 7.2 Compliance Logging

**Audit Trail Requirements**:
1. Every call recorded (audio + transcript)
2. Timestamp of all mandatory disclosures
3. Identity verification result logged
4. Outcome categorized
5. Retention: 7 years (Australian standard for financial records)

**Compliance Checks**:
- Time-gating (no calls outside 7:30am-9pm)
- Hard-coded mandatory disclosures (cannot be skipped)
- Automatic escalation on trigger words

---

## 8. Monitoring & Observability

### 8.1 Logging

**Backend Logs**:
- Request/response for all API calls
- Errors with stack traces
- Call scheduling events
- CRM sync results

**Tool**: Winston (Node.js logger) or Railway/Render built-in logs

---

### 8.2 Metrics to Track

| Metric | Definition | Target |
|--------|------------|--------|
| Call connection rate | % calls answered (excluding wrong numbers) | > 70% |
| Average call duration | Mean duration of answered calls | 2-4 minutes |
| Outcome distribution | % payment vs dispute vs no answer | TBD after first week |
| Human escalation rate | % calls escalated to human | < 10% |
| Compliance violation rate | % calls missing mandatory disclosures | 0% |
| API latency | Time from webhook to CRM update | < 5 seconds |

**Dashboard**: Build simple admin panel showing these metrics (Phase 1.5 or Phase 2)

---

## 9. Error Handling

### 9.1 Failure Scenarios

| Failure | Detection | Recovery |
|---------|-----------|----------|
| Vapi API down | HTTP 5xx errors | Retry after 5 min, alert developer |
| CRM database unreachable | Connection timeout | Queue writes, sync when online |
| Debtor phone invalid | Vapi returns "invalid number" | Log as WRONG_NUMBER, flag for manual review |
| Call drops mid-conversation | Vapi webhook "call_ended" early | Log partial transcript, retry next day |
| LLM exceeds token limit | Vapi error | Escalate to human immediately |

---

### 9.2 Retry Logic

**Call retries**:
- 1st attempt: Scheduled time
- 2nd attempt: Same day, 2+ hours later
- 3rd attempt: Next business day
- Max: 3 attempts per debt file

**API retries**:
- Exponential backoff for transient errors (5xx)
- Max 3 retries, then log error and alert

---

## 10. Scalability

### 10.1 Current Limitations

**Phase 1**:
- Single server (Railway/Render)
- Vapi concurrent call limit: 50-100 (plan dependent)
- Database: 10,000 debtor records (PostgreSQL handles easily)

---

### 10.2 Future Scaling

**If BCS scales to 1000+ calls/day**:
- Upgrade Vapi plan (higher concurrency)
- Add database read replicas
- Separate queue processing into background workers (Bull MQ)
- Load balancer for API servers

---

## 11. Technology Versions

| Technology | Version | Notes |
|------------|---------|-------|
| Node.js | 20.x LTS | Latest stable |
| Express | 4.x | Web framework |
| PostgreSQL | 15.x | Database |
| Vapi SDK | Latest | `@vapi-ai/server-sdk` |
| OpenAI SDK | Latest | `openai` (if needed) |

---

## 12. Development Tools

**Local Development**:
- IDE: VS Code
- Database: PostgreSQL (Docker container or local install)
- API testing: Postman or Insomnia
- Mock Vapi: ngrok for webhook testing

**Version Control**:
- Git + GitHub
- Branching strategy: `main` (production), `dev` (development)

**CI/CD**:
- Railway/Render auto-deploy on push to `main`

---

## 13. Migration Strategy (CRM Integration)

**Phase 1a**: Mock data (no CRM dependency)
- Hard-code sample debtor records in code
- Prove call flow works end-to-end

**Phase 1b**: Direct DB connection
- Connect to Peter's CRM database (read-only initially)
- Write outcomes to new table

**Phase 1c**: Production
- Full read/write access
- Monitor for conflicts with existing CRM usage

---

## 14. Related Documentation

- [REQUIREMENTS.md](./REQUIREMENTS.md) - Functional and non-functional requirements
- [DEVELOPMENT_PLAN.md](./DEVELOPMENT_PLAN.md) - 4-week timeline
- [DECISIONS_LOG.md](./DECISIONS_LOG.md) - Technical decisions and rationale
- [CRM_INTEGRATION.md](./CRM_INTEGRATION.md) - Database schema (TBD)

---

## 15. Architecture Decision Records (ADRs)

**ADR-001**: Use Vapi instead of Twilio
- **Context**: Need fast development with limited budget
- **Decision**: Vapi.ai for AI-native telephony
- **Consequences**: Higher per-minute cost ($0.05-0.09 vs $0.013) but 50% faster development

**ADR-002**: Use n8n instead of custom Node.js backend
- **Context**: Project is primarily orchestration (schedule, call, webhook, log, update)
- **Decision**: n8n workflow automation platform
- **Consequences**:
  - Faster development (visual workflows vs code)
  - Developer has existing boilerplate
  - Client can maintain workflows without hiring developer
  - Slightly less control but sufficient for requirements

**ADR-003**: Use PostgreSQL instead of MongoDB
- **Context**: Call logs and queue need ACID transactions, structured data
- **Decision**: PostgreSQL 15
- **Consequences**: Relational model fits structured data better, easier compliance auditing

**ADR-004**: Use OpenAI GPT-4o instead of Claude
- **Context**: Voice calls require low latency (<500ms response time)
- **Decision**: GPT-4o for primary LLM
- **Consequences**: Faster response times, proven voice handling, lower latency than Claude

**ADR-005**: Deploy to DigitalOcean Droplet instead of managed PaaS
- **Context**: Need cost-effective hosting with Docker support
- **Decision**: DigitalOcean Droplet ($6-12/month) with Docker Compose
- **Consequences**:
  - More affordable than Railway/Render ($20-30/month)
  - Full control over infrastructure
  - Requires basic DevOps (Docker, SSH, firewall)
  - n8n and PostgreSQL run in containers

**ADR-006**: Use Google Sheets as interim CRM (Stopgap)
- **Context**: Peter's CRM access delayed, timeline is tight (28 days)
- **Decision**: Use Google Sheets for Phase 1 debtor tracking, migrate to real CRM later
- **Consequences**:
  - ✅ Zero setup time (no waiting for credentials)
  - ✅ Easy for Peter to view/edit debtor data
  - ✅ n8n native integration (Google Sheets node)
  - ✅ Simple migration path (swap node, export/import data)
  - ❌ Limited scalability (fine for <1000 records)
  - ❌ No relational integrity or constraints
- **Status**: INTERIM - Will migrate to real CRM post-Phase 1

**ADR-007**: Use Twilio SMS for payment delivery
- **Context**: Need to deliver payment details to debtors who agree to pay
- **Decision**: Send payment links/details via SMS (Twilio) instead of verbal-only
- **Rationale**:
  - Higher conversion (debtor can click link immediately)
  - No transcription errors (written BSB/account vs. mishearing)
  - Persistent reference (debtor can revisit SMS)
  - Modern, professional approach
- **Consequences**:
  - ✅ Improved payment completion rate
  - ✅ Better debtor experience
  - ✅ Trackable (delivery confirmations)
  - ❌ Additional cost (~$0.08 per SMS)
  - ❌ Requires Twilio account setup
- **Cost impact**: ~$8 for 100 payment SMS (acceptable)

---

**End of Technical Architecture Document**
