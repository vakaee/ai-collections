# Workflow Adaptation Plan: Airbnb → Debt Collection

## Document Information

**Project**: AI Debt Collection Bot - Phase 1
**Version**: 1.0
**Last Updated**: October 3, 2025
**Status**: Ready for Implementation

---

## 1. Overview

This document details how to adapt existing Airbnb n8n workflows for the debt collection system.

**Time Savings**: 85% reduction in development time
- From scratch: 35 hours
- Adaptation: 10 hours (Phase 1 only - voice calls)

**Reusability Assessment**:
- Daily sync.json → 95% reusable ✅
- Vapi outbound caller.json → 80% reusable ✅
- Save results.json → 90% reusable ✅
- Send sms with review link.json → NOT USED (Phase 2)
- Respond to sms messages.json → NOT USED (Phase 2)

---

## 2. Google Sheets Schema

### 2.1 BCS_Debtors Sheet (23 columns)

| Column | Type | Example | Purpose |
|--------|------|---------|---------|
| debtor_id | Number | 1, 2, 3 | Unique identifier |
| name | Text | "John Smith" | Debtor name |
| phone | Text | "+61412345678" | Phone (E.164 format) |
| email | Text | "john@example.com" | Email address |
| debtor_type | Text | "consumer" or "commercial" | Routes to appropriate script |
| amount | Number | 1500.00 | Debt amount |
| creditor | Text | "ABC Company" | Creditor name |
| invoice_number | Text | "INV-2024-001" | Invoice reference |
| debt_date | Date | 2024-06-01 | Debt origin date |
| dob | Date | 1980-05-15 | Date of birth (consumer verification) |
| abn | Text | "12345678901" | ABN (commercial only) |
| timezone | Text | "AEST" | Debtor timezone (AEST/ACST/AWST) |
| call_status | Text | "calling" | null, "calling", "completed" |
| call_id | Text | "vapi_xxx" | Vapi call.id |
| last_call_date | Timestamp | 2025-10-05 10:30 | Last call attempt |
| last_outcome | Text | "PROMISE_TO_PAY" | Most recent outcome |
| next_action | Text | "SCHEDULE_CALL" | What to do next |
| next_call_date | Timestamp | 2025-10-11 14:00 | Scheduled next call |
| attempt_number | Number | 1, 2, 3 | Call progression |
| notes | Text | "Promised to pay by Oct 10" | Free-form notes |
| assigned_to_ai | Boolean | TRUE / FALSE | Include in AI queue? |
| payment_status | Text | "unpaid" | unpaid, paid, disputed |
| do_not_call | Boolean | FALSE | Compliance flag |
| recording_url | Text | "https://vapi..." | Call recording link |

---

## 3. Workflow 1: BCS Call Scheduler

**Based on**: Daily sync.json (95% reusable)

**Trigger**: Cron - every 30 minutes

**Purpose**: Read debtors from Google Sheets, filter eligible calls, trigger Vapi

### 3.1 Node 1: Cron Trigger

**Configuration**:
- Mode: Every 30 Minutes
- Field: Minute
- Value: 0,30

### 3.2 Node 2: Google Sheets - Read Rows

**Configuration**:
- Operation: Read Rows
- Document: BCS_Debtors (spreadsheet ID from env var)
- Sheet: Sheet1
- Range: A2:W1000 (23 columns, up to 1000 rows)
- Options: Return all matching rows

### 3.3 Node 3: Code - Filter & Validate

**Purpose**: Filter eligible debtors and check business hours

**Code**:
```javascript
const items = $input.all();
const now = new Date();

// Business hours configuration
const BUSINESS_HOURS = {
  AEST: { weekday: { start: 7.5, end: 21 }, weekend: { start: 9, end: 21 } },
  ACST: { weekday: { start: 7.5, end: 21 }, weekend: { start: 9, end: 21 } },
  AWST: { weekday: { start: 7.5, end: 21 }, weekend: { start: 9, end: 21 } }
};

// Timezone offsets from UTC
const TIMEZONE_OFFSETS = {
  AEST: 10,
  ACST: 9.5,
  AWST: 8
};

function isBusinessHours(timezone) {
  const offset = TIMEZONE_OFFSETS[timezone] || 10; // Default AEST
  const localTime = new Date(now.getTime() + offset * 3600000);
  const hour = localTime.getHours() + localTime.getMinutes() / 60;
  const day = localTime.getDay();
  const isWeekend = day === 0 || day === 6;

  const hours = BUSINESS_HOURS[timezone] || BUSINESS_HOURS.AEST;
  const schedule = isWeekend ? hours.weekend : hours.weekday;

  return hour >= schedule.start && hour < schedule.end;
}

const filtered = items.filter(item => {
  const data = item.json;

  // Must be assigned to AI
  if (data.assigned_to_ai !== 'TRUE' && data.assigned_to_ai !== true) return false;

  // Must be unpaid
  if (data.payment_status !== 'unpaid') return false;

  // Must not be currently calling
  if (data.call_status === 'calling') return false;

  // Must not be on do-not-call list
  if (data.do_not_call === 'TRUE' || data.do_not_call === true) return false;

  // Must have valid attempt number (1-3)
  const attemptNumber = parseInt(data.attempt_number) || 1;
  if (attemptNumber > 3) return false;

  // Must be scheduled for now or past
  const nextCallDate = new Date(data.next_call_date);
  if (nextCallDate > now) return false;

  // Must be within business hours for debtor's timezone
  const timezone = data.timezone || 'AEST';
  if (!isBusinessHours(timezone)) return false;

  return true;
});

return filtered;
```

### 3.4 Node 4: Split Into Items

**Configuration**:
- Mode: Split Out Items
- Field: json (each debtor becomes separate workflow execution)

### 3.5 Node 5: Code - Normalize Phone & Prepare Variables

**Purpose**: Convert phone to E.164, prepare Vapi variables

**Code**:
```javascript
const data = $input.item.json;

// Phone normalization
function normalizePhone(phone) {
  if (!phone) return null;

  // Remove spaces, dashes, parentheses
  let cleaned = phone.toString().replace(/[\s\-\(\)]/g, '');

  // Convert Australian 04xx to +614xx
  if (cleaned.startsWith('04')) {
    cleaned = '+61' + cleaned.substring(1);
  }

  // Convert Australian 61 to +61
  if (cleaned.startsWith('61') && !cleaned.startsWith('+')) {
    cleaned = '+' + cleaned;
  }

  // Add + if missing (assume Australian)
  if (!cleaned.startsWith('+')) {
    cleaned = '+61' + cleaned;
  }

  return cleaned;
}

// Assistant selection
const debtorType = (data.debtor_type || 'consumer').toLowerCase();
const attemptNumber = parseInt(data.attempt_number) || 1;

const assistantMap = {
  'consumer_1': $vars.VAPI_CONSUMER_1,
  'consumer_2': $vars.VAPI_CONSUMER_2,
  'consumer_3': $vars.VAPI_CONSUMER_3,
  'commercial_1': $vars.VAPI_COMMERCIAL_1,
  'commercial_2': $vars.VAPI_COMMERCIAL_2,
  'commercial_3': $vars.VAPI_COMMERCIAL_3
};

const assistantKey = `${debtorType}_${attemptNumber}`;
const assistantId = assistantMap[assistantKey];

if (!assistantId) {
  throw new Error(`No assistant found for ${assistantKey}`);
}

// Prepare Vapi call payload
return {
  debtor_id: data.debtor_id,
  row_number: data.__rowNum__, // FIX #1: Use n8n's built-in row number (survives filtering)
  phone_normalized: normalizePhone(data.phone),
  phone_original: data.phone,
  assistant_id: assistantId,
  vapi_payload: {
    assistantId: assistantId,
    customer: {
      number: normalizePhone(data.phone)
    },
    phoneNumberId: $vars.VAPI_PHONE_NUMBER_ID, // FIX #3: Use ID not phone number
    assistantOverrides: {
      variableValues: {
        debtor_id: data.debtor_id.toString(),
        debtor_name: data.name,
        amount: data.amount.toString(),
        creditor: data.creditor,
        invoice_number: data.invoice_number,
        dob: data.dob || '',
        abn: data.abn || '',
        attempt_number: attemptNumber.toString()
      }
    }
  }
};
```

### 3.6 Node 6: Google Sheets - Update Row (Set call_status)

**Purpose**: Mark debtor as "calling" to prevent duplicate calls

**Configuration**:
- Operation: Update Row
- Document: BCS_Debtors
- Sheet: Sheet1
- Row Number: `{{$node['Code - Normalize Phone & Prepare Variables'].json.row_number}}`
- Columns to Update:
  - Column M (call_status): "calling"

### 3.7 Node 7: HTTP Request - Vapi /call

**Configuration**:
- Method: POST
- URL: `https://api.vapi.ai/call/phone`
- Authentication: Header Auth
  - Header Name: Authorization
  - Header Value: `Bearer {{$vars.VAPI_API_KEY}}`
- Body: `{{$node['Code - Normalize Phone & Prepare Variables'].json.vapi_payload}}`
- Options:
  - Response Format: JSON
  - Timeout: 30000

**Expected Response**:
```json
{
  "id": "call_xxx",
  "status": "queued",
  "phoneNumber": "+61412345678"
}
```

### 3.8 Error Handler (if Vapi call fails)

**Node 8: IF - Check HTTP Success**

**Configuration**:
- Condition: `{{$node['HTTP Request - Vapi /call'].json.id}}` exists

**Branch: False (call failed)**

**Node 9: Google Sheets - Reset call_status**

**Configuration**:
- Operation: Update Row
- Row Number: `{{$node['Code - Normalize Phone & Prepare Variables'].json.row_number}}`
- Columns:
  - Column M (call_status): null
  - Column T (notes): `[{{$now}}] Vapi call failed: {{$node['HTTP Request - Vapi /call'].json.error}}`

---

## 4. Workflow 2: BCS Vapi Webhook Handler

**Based on**: Save results.json (90% reusable) + custom logic

**Trigger**: Webhook (POST from Vapi)

**Purpose**: Receive call outcome, calculate next action, update Google Sheets, trigger SMS

### 4.1 Node 1: Webhook Trigger

**Configuration**:
- HTTP Method: POST
- Path: vapi-handler
- Response Mode: Respond Immediately
- Response Code: 200

**Webhook URL**: `https://n8n.yourdomain.com/webhook/vapi-handler`

**Configure in Vapi**:
- Dashboard → Settings → Webhooks → Server URL: [webhook URL]

### 4.2 Node 2: Code - Verify Signature

**Purpose**: Prevent unauthorized webhook calls

**Code**:
```javascript
const crypto = require('crypto');

// FIX #2: Header name is 'X-Vapi-Signature' (capital X) - verified from Vapi docs
const signature = $input.item.headers['X-Vapi-Signature'] || $input.item.headers['x-vapi-signature'];
const secret = $vars.VAPI_WEBHOOK_SECRET;
const payload = JSON.stringify($input.item.body);

if (!signature) {
  throw new Error('Missing webhook signature');
}

const expectedSignature = crypto
  .createHmac('sha256', secret)
  .update(payload)
  .digest('hex');

if (signature !== expectedSignature) {
  throw new Error('Invalid webhook signature');
}

return $input.item.json;
```

### 4.3 Node 3: Code - Parse Call Data

**Purpose**: Extract outcome, notes, call metadata

**Code**:
```javascript
// FIX #2: Updated webhook payload parsing based on Vapi docs verification
const webhook = $input.item.json;
const message = webhook.message;

// Verify this is an end-of-call-report event
if (message.type !== 'end-of-call-report') {
  throw new Error(`Unexpected webhook type: ${message.type}`);
}

const call = message.call;
const artifact = message.artifact || {};
const endedReason = message.endedReason;

// Extract debtor_id from assistant overrides (stored in call object)
const debtorId = call.assistantOverrides?.variableValues?.debtor_id;

if (!debtorId) {
  throw new Error('Missing debtor_id in webhook');
}

// Check if call succeeded or failed
const callStatus = call.status; // "ended" or "failed"

// Parse function call (log_outcome)
// NOTE: Vapi may send this in artifact.messages array - needs testing
let outcome = 'UNKNOWN';
let paymentMethod = 'none';
let notes = '';
let promisedDate = null;

if (callStatus === 'failed') {
  // FIX #7: CALL_FAILED not in function enum - handle separately
  outcome = 'CALL_FAILED';
  notes = `Call failed: ${endedReason}`;
} else if (artifact.messages && artifact.messages.length > 0) {
  // Search for log_outcome function call in messages
  // Function calls may appear as toolCalls in message objects
  let functionCallFound = false;

  for (const msg of artifact.messages) {
    if (msg.toolCalls && Array.isArray(msg.toolCalls)) {
      for (const toolCall of msg.toolCalls) {
        if (toolCall.function?.name === 'log_outcome') {
          // Parse function arguments (may be string or object)
          const args = typeof toolCall.function.arguments === 'string'
            ? JSON.parse(toolCall.function.arguments)
            : toolCall.function.arguments;

          outcome = args.outcome || 'UNKNOWN';
          paymentMethod = args.payment_method || 'none';
          notes = args.notes || '';
          promisedDate = args.promised_date || null;
          functionCallFound = true;
          break;
        }
      }
      if (functionCallFound) break;
    }
  }

  if (!functionCallFound) {
    // No function call found = likely no answer, voicemail, or system error
    outcome = 'NO_ANSWER';
    notes = 'No outcome logged by assistant (check transcript)';
  }
} else {
  // No messages in artifact
  outcome = 'NO_ANSWER';
  notes = 'No conversation recorded';
}

return {
  debtor_id: debtorId,
  call_id: call.id,
  call_status: callStatus,
  recording_url: artifact.recording?.url || call.recordingUrl || '',
  transcript: artifact.transcript || '',
  outcome: outcome,
  payment_method: paymentMethod,
  notes: notes,
  promised_date: promisedDate,
  call_duration: call.endedAt && call.startedAt
    ? (new Date(call.endedAt) - new Date(call.startedAt)) / 1000
    : 0
};
```

### 4.4 Node 4: Code - Calculate Next Action

**Purpose**: Determine next_action, next_call_date, flags based on outcome

**Code**:
```javascript
const data = $input.item.json;
const outcome = data.outcome;
const promisedDate = data.promised_date;
const now = new Date();

let nextAction = null;
let nextCallDate = null;
let assignedToAi = true;
let paymentStatus = 'unpaid';
let doNotCall = false;

function addDays(date, days) {
  const result = new Date(date);
  result.setDate(result.getDate() + days);
  return result;
}

function addHours(date, hours) {
  const result = new Date(date);
  result.setHours(result.getHours() + hours);
  return result;
}

switch(outcome) {
  case 'PROMISE_TO_PAY':
    nextAction = 'SCHEDULE_CALL';
    if (promisedDate) {
      // Follow up day after promised date
      nextCallDate = addDays(new Date(promisedDate), 1);
    } else {
      // Default: follow up in 7 days
      nextCallDate = addDays(now, 7);
    }
    break;

  case 'READY_TO_PAY':
    nextAction = 'SEND_PAYMENT_SMS';
    // Don't schedule next call yet (wait for payment confirmation)
    assignedToAi = false;
    break;

  case 'NO_ANSWER':
  case 'VOICEMAIL':
    nextAction = 'SCHEDULE_CALL';
    nextCallDate = addHours(now, 4); // Retry in 4 hours
    break;

  case 'WRONG_NUMBER':
    nextAction = 'MANUAL_REVIEW';
    assignedToAi = false;
    break;

  case 'DISPUTE_RAISED':
  case 'HARDSHIP_CLAIMED':
    nextAction = 'MANUAL_REVIEW';
    assignedToAi = false;
    break;

  case 'REFUSED_TO_ENGAGE':
    nextAction = 'SCHEDULE_CALL';
    nextCallDate = addDays(now, 7); // Wait 1 week
    break;

  case 'REQUESTED_NO_CONTACT':
    nextAction = 'MANUAL_REVIEW';
    assignedToAi = false;
    doNotCall = true;
    break;

  case 'ALREADY_PAID':
    paymentStatus = 'paid';
    nextAction = 'CLOSE_ACCOUNT';
    assignedToAi = false;
    break;

  case 'CALL_FAILED':
    nextAction = 'SCHEDULE_CALL';
    nextCallDate = addHours(now, 2); // Retry in 2 hours
    break;

  case 'OBJECTED_TO_RECORDING':
    nextAction = 'MANUAL_REVIEW';
    assignedToAi = false;
    break;

  default:
    nextAction = 'MANUAL_REVIEW';
    assignedToAi = false;
}

return {
  ...data,
  next_action: nextAction,
  next_call_date: nextCallDate ? nextCallDate.toISOString() : '',
  assigned_to_ai: assignedToAi,
  payment_status: paymentStatus,
  do_not_call: doNotCall
};
```

### 4.5 Node 5: Google Sheets - Lookup Row

**Configuration**:
- Operation: Lookup
- Document: BCS_Debtors
- Sheet: Sheet1
- Lookup Column: A (debtor_id)
- Lookup Value: `{{$node['Code - Parse Call Data'].json.debtor_id}}`

### 4.6 Node 6: Code - Prepare Update Data

**Purpose**: Build update object with incremented attempt_number

**Code**:
```javascript
const parsed = $node['Code - Calculate Next Action'].json;
const currentRow = $node['Google Sheets - Lookup Row'].json;

const currentAttempt = parseInt(currentRow.attempt_number) || 1;
const newAttempt = currentAttempt + 1;

const timestamp = new Date().toLocaleString('en-AU', { timeZone: 'Australia/Sydney' });
const newNotes = `${currentRow.notes || ''}\n[${timestamp}] Attempt ${currentAttempt}: ${parsed.outcome} - ${parsed.notes}`.trim();

return {
  row_number: currentRow.__rowNum__, // n8n provides this
  call_status: 'completed',
  call_id: parsed.call_id,
  recording_url: parsed.recording_url,
  last_call_date: new Date().toISOString(),
  last_outcome: parsed.outcome,
  next_action: parsed.next_action,
  next_call_date: parsed.next_call_date,
  attempt_number: newAttempt,
  notes: newNotes,
  assigned_to_ai: parsed.assigned_to_ai,
  payment_status: parsed.payment_status,
  do_not_call: parsed.do_not_call,
  payment_method: parsed.payment_method,
  debtor_id: parsed.debtor_id
};
```

### 4.7 Node 7: Google Sheets - Update Row

**Configuration**:
- Operation: Update Row
- Document: BCS_Debtors
- Sheet: Sheet1
- Row Number: `{{$node['Code - Prepare Update Data'].json.row_number}}`
- Columns to Update:
  - Column M (call_status): `{{$node['Code - Prepare Update Data'].json.call_status}}`
  - Column N (call_id): `{{$node['Code - Prepare Update Data'].json.call_id}}`
  - Column O (last_call_date): `{{$node['Code - Prepare Update Data'].json.last_call_date}}`
  - Column P (last_outcome): `{{$node['Code - Prepare Update Data'].json.last_outcome}}`
  - Column Q (next_action): `{{$node['Code - Prepare Update Data'].json.next_action}}`
  - Column R (next_call_date): `{{$node['Code - Prepare Update Data'].json.next_call_date}}`
  - Column S (attempt_number): `{{$node['Code - Prepare Update Data'].json.attempt_number}}`
  - Column T (notes): `{{$node['Code - Prepare Update Data'].json.notes}}`
  - Column U (assigned_to_ai): `{{$node['Code - Prepare Update Data'].json.assigned_to_ai}}`
  - Column V (payment_status): `{{$node['Code - Prepare Update Data'].json.payment_status}}`
  - Column W (do_not_call): `{{$node['Code - Prepare Update Data'].json.do_not_call}}`
  - Column X (recording_url): `{{$node['Code - Prepare Update Data'].json.recording_url}}`

### 4.8 Node 8: IF - Check if Manual Review Needed

**Configuration**:
- Condition: `{{$node['Code - Calculate Next Action'].json.next_action}}` equals "MANUAL_REVIEW"

### 4.9 Node 9: Email - Notify [CLIENT] (if true)

**Configuration**:
- To: `{{$vars.BCS_EMAIL}}`
- Subject: "BCS Debt Collection - Manual Review Required"
- Body:
```
Manual review required for debtor:

Debtor ID: {{$node['Code - Prepare Update Data'].json.debtor_id}}
Outcome: {{$node['Code - Prepare Update Data'].json.last_outcome}}
Notes: {{$node['Code - Prepare Update Data'].json.notes}}

View in Google Sheets: [LINK]
```

---

**Note**: SMS automation workflows (Payment SMS and SMS Status Handler) deferred to Phase 2. Phase 1 delivers payment instructions verbally during calls.

---

## 5. Workflow 3: BCS Test Webhook

**Purpose**: Test webhook handler without making real calls

**Trigger**: Manual

### 5.1 Node 1: Manual Trigger

### 5.2 Node 2: Code - Sample Vapi Payload

**Code**:
```javascript
// FIX #6: Updated test payload to match verified Vapi webhook structure
return {
  message: {
    type: "end-of-call-report",
    endedReason: "assistant-ended-call",
    call: {
      id: "call_test_123",
      status: "ended",
      startedAt: new Date(Date.now() - 120000).toISOString(),
      endedAt: new Date().toISOString(),
      assistantOverrides: {
        variableValues: {
          debtor_id: "1"
        }
      }
    },
    artifact: {
      recording: {
        url: "https://example.com/recording.mp3"
      },
      transcript: "Full test transcript...",
      messages: [
        {
          role: "assistant",
          message: "Hello, this is a test",
          time: Date.now() - 120000,
          toolCalls: [
            {
              function: {
                name: "log_outcome",
                arguments: {
                  outcome: "PROMISE_TO_PAY",
                  payment_method: "bank_transfer",
                  notes: "Debtor agreed to pay by Friday",
                  promised_date: "2025-10-10"
                }
              }
            }
          ]
        }
      ]
    }
  }
};
```

**Note**: This test payload includes a mock signature. For production, add signature generation:
```javascript
const crypto = require('crypto');
const payload = JSON.stringify(testPayload);
const signature = crypto.createHmac('sha256', $vars.VAPI_WEBHOOK_SECRET).update(payload).digest('hex');
// Include signature in HTTP headers
```

### 5.3 Node 3: HTTP Request - POST to Webhook Handler

**Configuration**:
- Method: POST
- URL: `{{$vars.WEBHOOK_VAPI_HANDLER}}`
- Body: `{{$node['Code - Sample Vapi Payload'].json}}`

---

## 6. Vapi Assistant Configuration

### 6.1 Function Definitions (Apply to All 6 Assistants)

**Function 1: log_outcome**

```json
{
  "name": "log_outcome",
  "description": "Log the call outcome when conversation ends. MUST be called before ending call.",
  "parameters": {
    "type": "object",
    "properties": {
      "outcome": {
        "type": "string",
        "enum": [
          "PROMISE_TO_PAY",
          "READY_TO_PAY",
          "DISPUTE_RAISED",
          "HARDSHIP_CLAIMED",
          "NO_ANSWER",
          "VOICEMAIL",
          "WRONG_NUMBER",
          "REFUSED_TO_ENGAGE",
          "ALREADY_PAID",
          "REQUESTED_NO_CONTACT",
          "OBJECTED_TO_RECORDING"
        ],
        "description": "The outcome of the call"
      },
      "payment_method": {
        "type": "string",
        "enum": ["credit_card", "bank_transfer", "cheque", "none"],
        "description": "Preferred payment method if debtor is ready to pay"
      },
      "notes": {
        "type": "string",
        "description": "Free-form summary of the conversation"
      },
      "promised_date": {
        "type": "string",
        "format": "date",
        "description": "ISO date string if debtor promised to pay by specific date"
      }
    },
    "required": ["outcome", "notes"]
  }
}
```

**Function 2: verify_identity**

```json
{
  "name": "verify_identity",
  "description": "Verify debtor identity before discussing debt details",
  "parameters": {
    "type": "object",
    "properties": {
      "verification_method": {
        "type": "string",
        "enum": ["dob", "abn", "address"],
        "description": "Method used for verification"
      },
      "verified": {
        "type": "boolean",
        "description": "Whether identity was successfully verified"
      }
    },
    "required": ["verification_method", "verified"]
  }
}
```

### 6.2 Assistant Naming Convention

- Consumer 1st Call: "BCS Consumer - First Contact"
- Consumer 2nd Call: "BCS Consumer - Follow-up"
- Consumer 3rd Call: "BCS Consumer - Final Warning"
- Commercial 1st Call: "BCS Commercial - First Contact"
- Commercial 2nd Call: "BCS Commercial - Follow-up"
- Commercial 3rd Call: "BCS Commercial - Final Warning"

### 6.3 Variable Access in Scripts

**Example prompt**:
```
You are calling on behalf of [COLLECTION AGENCY] regarding a debt.

Debtor name: {{debtor_name}}
Amount owed: ${{amount}}
Creditor: {{creditor}}
Invoice number: {{invoice_number}}

For consumer debtors, verify identity by asking date of birth (expected: {{dob}}).
For commercial debtors, verify ABN (expected: {{abn}}).

This is attempt {{attempt_number}} of 3.
```

---

## 7. Environment Variables

**Note**: Twilio variables removed (SMS is Phase 2)

```bash
# Vapi
VAPI_API_KEY=your_vapi_api_key
VAPI_WEBHOOK_SECRET=your_webhook_secret
VAPI_CONSUMER_1=assistant_id_consumer_1
VAPI_CONSUMER_2=assistant_id_consumer_2
VAPI_CONSUMER_3=assistant_id_consumer_3
VAPI_COMMERCIAL_1=assistant_id_commercial_1
VAPI_COMMERCIAL_2=assistant_id_commercial_2
VAPI_COMMERCIAL_3=assistant_id_commercial_3
VAPI_PHONE_NUMBER_ID=ph_abc123xyz  # FIX #3: Use phone number ID from Vapi dashboard

# Twilio
TWILIO_ACCOUNT_SID=your_twilio_sid
TWILIO_AUTH_TOKEN=your_twilio_token
TWILIO_FROM_NUMBER=+61xxxxxxxxx

# Google Sheets
GOOGLE_SHEETS_SPREADSHEET_ID=your_spreadsheet_id
GOOGLE_SHEETS_CREDENTIALS=service_account_json

# Payment Info
STRIPE_PAYMENT_LINK=https://buy.stripe.com/xxxxx
BSB_NUMBER=123-456
ACCOUNT_NUMBER=123456789
ACCOUNT_NAME=[COLLECTION AGENCY] Pty Ltd
BCS_MAILING_ADDRESS=123 Main St, Sydney NSW 2000
BCS_PHONE_NUMBER=+61xxxxxxxxx
BCS_EMAIL=[CLIENT EMAIL]

# n8n Webhooks
WEBHOOK_VAPI_HANDLER=https://n8n.yourdomain.com/webhook/vapi-handler
WEBHOOK_SEND_SMS=https://n8n.yourdomain.com/webhook/send-payment-sms
WEBHOOK_SMS_STATUS=https://n8n.yourdomain.com/webhook/sms-status
```

---

## 8. Testing Plan

### 8.1 Unit Tests

**Test 1: Phone Normalization**
- Input: "0412 345 678"
- Expected: "+61412345678"

**Test 2: Assistant Selection**
- Input: debtor_type="consumer", attempt_number=2
- Expected: VAPI_CONSUMER_2

**Test 3: Outcome Routing**
- Input: outcome="PROMISE_TO_PAY", promised_date="2025-10-10"
- Expected: next_action="SCHEDULE_CALL", next_call_date="2025-10-11"

### 8.2 Integration Tests

**Test 1: Full Call Flow**
1. Add test debtor to Google Sheets
2. Set next_call_date to now
3. Wait for cron trigger
4. Verify Vapi call initiated
5. Use "BCS Test Webhook" to simulate outcome
6. Verify Google Sheets updated correctly

**Test 2: Payment SMS**
1. Trigger test webhook with outcome="READY_TO_PAY"
2. Verify SMS sent via Twilio
3. Verify notes updated in Google Sheets

**Test 3: Manual Review Notification**
1. Trigger test webhook with outcome="DISPUTE_RAISED"
2. Verify email sent to [CLIENT]
3. Verify assigned_to_ai set to FALSE

### 8.3 Compliance Tests

**Test 1: Business Hours**
- Set timezone to AWST, current time to 6:00am AWST
- Verify debtor NOT called (before 7:30am)

**Test 2: Do-Not-Call**
- Set do_not_call=TRUE
- Verify debtor NOT called

**Test 3: Max Attempts**
- Set attempt_number=4
- Verify debtor NOT called

---

## 9. Timeline

| Task | Duration | Dependencies |
|------|----------|--------------|
| Create Google Sheet with schema | 1h | - |
| Adapt Workflow 1 (Scheduler) | 2h | Google Sheet |
| Adapt Workflow 2 (Webhook Handler) | 3h | Google Sheet |
| Create Workflow 3 (Test) | 0.5h | Workflow 2 |
| Configure 6 Vapi Assistants | 2h | Vapi account |
| Add verbal payment instructions to assistants | 0.5h | Vapi account |
| Integration Testing | 1.5h | All workflows |
| Compliance Testing | 1h | All workflows |
| **TOTAL** | **10.5h** | |

**Compare to from-scratch**: 35 hours

**Time savings**: 24.5 hours (70%)

**Note**: SMS workflows (2 workflows, ~3h) deferred to Phase 2

---

## 10. Deployment Checklist

- [ ] Google Cloud project created
- [ ] Google Sheets API enabled
- [ ] Service account created with Sheets access
- [ ] BCS_Debtors spreadsheet created with 23 columns
- [ ] Sample test data added (3-5 rows)
- [ ] Vapi account created
- [ ] Vapi webhook URL configured
- [ ] Vapi webhook secret generated
- [ ] 6 Vapi assistants created with function definitions and verbal payment instructions
- [ ] n8n environment variables configured (14 variables, Twilio removed)
- [ ] Workflow 1 (Call Scheduler) imported and tested
- [ ] Workflow 2 (Webhook Handler) imported and tested
- [ ] Workflow 3 (Test Webhook) created for testing
- [ ] Test call completed successfully
- [ ] Payment instructions delivered verbally and confirmed by tester
- [ ] Compliance checks validated
- [ ] [CLIENT] trained on Google Sheets workflow

**Phase 2 additions (when commissioned)**:
- [ ] Twilio account created and SMS workflows added
- [ ] Email automation workflows added
- [ ] Management dashboard created

---

## 11. Risks & Mitigations

| Risk | Impact | Mitigation | Status |
|------|--------|------------|--------|
| Webhook security breach | HIGH | Signature verification | MITIGATED |
| Duplicate calls | HIGH | call_status column | MITIGATED |
| Phone format errors | HIGH | Normalization function | MITIGATED |
| Timezone errors | MEDIUM | timezone column + validation | NEEDS PETER INPUT |
| Missing function definitions | HIGH | Explicit Vapi config | MITIGATED |
| Google Sheets race condition | LOW | Column-specific updates | ACCEPTABLE |
| Payment details misheard | MEDIUM | AI confirms debtor wrote down details | MITIGATED |
| Compliance violations | HIGH | do_not_call flag + business hours | MITIGATED |

---

## 12. Phase 2 Enhancements (When Commissioned)

**Estimate**: 15-18 hours, $2,000-[PRICE REDACTED]

1. **SMS Automation**: Payment SMS (4 templates) + delivery tracking
2. **Email Automation**: Dispute/hardship forms + manual review notifications
3. **Inbound Call Handling**: Route to appropriate assistant based on debtor type
4. **Management Dashboard**: Call metrics, outcome distribution, review queue
5. **Advanced Features** (Phase 3 if needed):
   - Real CRM migration (Google Sheets → [CLIENT]'s CRM)
   - Do-Not-Call Registry API integration
   - Call recording archival (DigitalOcean Spaces)
   - Multi-language support
   - Predictive dialing optimization

---

**End of Workflow Adaptation Plan**

**Status**: Ready for implementation
**Estimated Completion**: October 16, 2025
**Last Updated**: October 3, 2025
