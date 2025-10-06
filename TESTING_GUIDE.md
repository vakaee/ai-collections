# Testing Guide - BCS Debt Collection Bot

## Document Information

**Project**: AI Debt Collection Bot - Phase 1 (Voice-Only)
**Version**: 1.0
**Last Updated**: October 3, 2025
**Status**: Ready for Testing

---

## Overview

This guide provides step-by-step instructions for testing the BCS Debt Collection Bot workflows before going live. Critical tests verify Vapi webhook payload structure, which couldn't be 100% confirmed from documentation alone.

**Phase 1 Scope**: Voice-only outbound calling with verbal payment instructions
**Phase 2 Scope** (not included): SMS + Email automation

**Purpose**: Validate all assumptions made during development, especially Vapi API integration

---

## Prerequisites

Before testing, ensure:

- [ ] All 3 n8n workflows imported (Call Scheduler, Vapi Webhook Handler, Test Webhook)
- [ ] All 14 environment variables configured in `.env`
- [ ] 6 Vapi assistants created with function definitions and verbal payment instructions
- [ ] Google Sheets created with 23 columns and test data
- [ ] Your personal phone number added as test debtor

**Phase 2 Prerequisites** (not needed for Phase 1):
- Twilio account with SMS capabilities
- Email server configuration

---

## Phase 1: Environment Setup Validation

### Test 1.1: Environment Variables Loaded

**Purpose**: Verify all Phase 1 env vars accessible in n8n (14 variables)

**Steps**:
1. Open n8n → Workflows → BCS Call Scheduler
2. Add temporary Code node with this code:
```javascript
return {
  vapi_api_key: $vars.VAPI_API_KEY ? 'SET' : 'MISSING',
  vapi_webhook_secret: $vars.VAPI_WEBHOOK_SECRET ? 'SET' : 'MISSING',
  vapi_phone_number_id: $vars.VAPI_PHONE_NUMBER_ID ? 'SET' : 'MISSING',
  vapi_consumer_1: $vars.VAPI_CONSUMER_1 ? 'SET' : 'MISSING',
  vapi_consumer_2: $vars.VAPI_CONSUMER_2 ? 'SET' : 'MISSING',
  vapi_consumer_3: $vars.VAPI_CONSUMER_3 ? 'SET' : 'MISSING',
  vapi_commercial_1: $vars.VAPI_COMMERCIAL_1 ? 'SET' : 'MISSING',
  vapi_commercial_2: $vars.VAPI_COMMERCIAL_2 ? 'SET' : 'MISSING',
  vapi_commercial_3: $vars.VAPI_COMMERCIAL_3 ? 'SET' : 'MISSING',
  google_spreadsheet_id: $vars.GOOGLE_SHEETS_SPREADSHEET_ID ? 'SET' : 'MISSING',
  stripe_payment_link: $vars.STRIPE_PAYMENT_LINK ? 'SET' : 'MISSING',
  bsb_number: $vars.BSB_NUMBER ? 'SET' : 'MISSING',
  account_number: $vars.ACCOUNT_NUMBER ? 'SET' : 'MISSING',
  webhook_vapi_handler: $vars.WEBHOOK_VAPI_HANDLER || 'MISSING'
};
```
3. Execute node
4. Verify all values show 'SET'

**Expected Result**: All 14 Phase 1 variables present

**Note**: Twilio variables not needed for Phase 1 (SMS is Phase 2)

**If Failed**: Check `.env` file location, restart n8n container

---

### Test 1.2: Google Sheets Connection

**Purpose**: Verify n8n can read Google Sheets

**Steps**:
1. Open Workflow 1 (BCS Call Scheduler)
2. Find "Google Sheets - Read Rows" node
3. Click "Execute Node"
4. Inspect output

**Expected Result**: Returns array with test debtors (5 rows from CSV template)

**If Failed**:
- Re-authenticate Google Sheets credentials in n8n
- Verify spreadsheet ID matches `.env`
- Check sheet shared with service account (if using service account auth)

---

### Test 1.3: Vapi API Key Valid

**Purpose**: Verify Vapi API key works

**Steps**:
1. Create temporary Code node:
```javascript
const response = await fetch('https://api.vapi.ai/assistant', {
  headers: {
    'Authorization': `Bearer ${$vars.VAPI_API_KEY}`
  }
});

return {
  status: response.status,
  ok: response.ok,
  message: response.ok ? 'API key valid' : 'API key invalid'
};
```
2. Execute node

**Expected Result**: `{ status: 200, ok: true, message: 'API key valid' }`

**If Failed**: Regenerate API key in Vapi Dashboard

---

## Phase 2: Workflow Unit Tests

### Test 2.1: Phone Normalization

**Purpose**: Verify phone number conversion to E.164 format

**Steps**:
1. Open Workflow 1 → "Code - Normalize Phone & Prepare Variables" node
2. Create test input:
```javascript
// In previous node, create test data:
return [
  { phone: '0412 345 678' },
  { phone: '+61 412 345 678' },
  { phone: '61412345678' },
  { phone: '+61412345678' }
];
```
3. Execute normalization node
4. Check output

**Expected Result**: All phones normalized to `+61412345678`

**If Failed**: Debug normalization function logic

---

### Test 2.2: Assistant Selection Logic

**Purpose**: Verify correct assistant chosen based on debtor type and attempt number

**Test Cases**:

| Debtor Type | Attempt # | Expected Assistant ID |
|-------------|-----------|----------------------|
| consumer | 1 | VAPI_CONSUMER_1 |
| consumer | 2 | VAPI_CONSUMER_2 |
| consumer | 3 | VAPI_CONSUMER_3 |
| commercial | 1 | VAPI_COMMERCIAL_1 |
| commercial | 2 | VAPI_COMMERCIAL_2 |
| commercial | 3 | VAPI_COMMERCIAL_3 |

**Steps**:
1. Manually execute "Code - Normalize Phone & Prepare Variables" with test data
2. Verify `assistant_id` matches expected value

**If Failed**: Check assistant ID mapping in code

---

### Test 2.3: Business Hours Check

**Purpose**: Verify debtors filtered outside business hours

**Test Cases**:

| Timezone | Current Time (Local) | Should Call? |
|----------|---------------------|--------------|
| AEST | 7:00am | NO (before 7:30am) |
| AEST | 7:30am | YES |
| AEST | 8:59pm | YES |
| AEST | 9:00pm | NO (after 9pm) |
| AWST | 7:00am | NO (3hr offset = 10am AEST) |

**Steps**:
1. Add test debtor with specific timezone
2. Modify system time or hardcode test time in filter logic
3. Execute filter node
4. Verify debtor included/excluded correctly

**If Failed**: Review business hours calculation and timezone offsets

---

## Phase 3: CRITICAL - Vapi Webhook Payload Verification

This is the most important test. We need to verify the actual Vapi webhook payload structure.

### Test 3.1: Capture Real Webhook Payload

**Purpose**: Inspect actual Vapi webhook to verify our parsing logic

**Steps**:

1. **Prepare Test Debtor**:
   - Add your phone number to Google Sheets as test debtor
   - Set `debtor_id = 999` (easy to identify)
   - Set `next_call_date = now`
   - Set `assigned_to_ai = TRUE`
   - Set `payment_status = unpaid`
   - Set `attempt_number = 1`
   - Set `debtor_type = consumer`

2. **Set Up Webhook Logging**:
   - Open Workflow 2 (BCS Vapi Webhook Handler)
   - Add new Code node BEFORE "Verify Signature" node
   - Code:
```javascript
// LOG FULL WEBHOOK PAYLOAD
console.log('===== FULL VAPI WEBHOOK PAYLOAD =====');
console.log(JSON.stringify($input.item, null, 2));
console.log('===== END PAYLOAD =====');

// Pass through unchanged
return $input.item.json;
```

3. **Make Test Call**:
   - Execute Workflow 1 manually (or wait for cron)
   - Answer your phone
   - When AI asks for DOB, provide fake DOB
   - Listen to debt details
   - Say "I'd like to pay by credit card"
   - End call

4. **Capture Webhook**:
   - Check n8n logs: `docker logs n8n | grep "FULL VAPI WEBHOOK"`
   - Copy entire JSON payload
   - Save to file `vapi_webhook_sample.json`

5. **Verify Structure**:
   Compare actual payload against our expected structure:

**Expected**:
```json
{
  "message": {
    "type": "end-of-call-report",
    "endedReason": "...",
    "call": {
      "id": "call_xxx",
      "status": "ended",
      "startedAt": "...",
      "endedAt": "...",
      "assistantOverrides": {
        "variableValues": {
          "debtor_id": "999"
        }
      }
    },
    "artifact": {
      "recording": {
        "url": "https://..."
      },
      "transcript": "...",
      "messages": [...]
    }
  }
}
```

6. **Check Key Fields**:
   - [ ] `message.type` exists and equals "end-of-call-report"
   - [ ] `message.call` exists
   - [ ] `message.call.id` exists
   - [ ] `message.artifact` exists
   - [ ] `message.artifact.messages` is an array
   - [ ] Function call (`log_outcome`) found in messages array
   - [ ] Function call has `toolCalls` or similar structure

7. **Update Code If Needed**:
   If payload structure differs from expected:
   - Update Workflow 2 "Parse Call Data" code
   - Update VAPI_API_VERIFICATION.md with actual structure
   - Update WORKFLOW_ADAPTATION_PLAN.md

---

### Test 3.2: Signature Verification

**Purpose**: Verify webhook signature works

**Steps**:

1. Check webhook received from Test 3.1
2. Inspect headers in n8n webhook trigger node output
3. Verify `X-Vapi-Signature` header present (or check exact header name)
4. Temporarily disable signature check:
```javascript
// Comment out signature verification temporarily
// const signature = ...
// if (signature !== expectedSignature) { throw ... }

return $input.item.json;
```
5. Verify webhook processes successfully
6. Re-enable signature check
7. Make another test call
8. Verify signature passes validation

**If Failed**:
- Check header name (capital X vs lowercase x)
- Verify webhook secret matches Vapi dashboard
- Check payload format (may need to use raw body, not JSON)

---

### Test 3.3: Function Call Result Parsing

**Purpose**: Verify we can extract `log_outcome` results from webhook

**Steps**:

1. Using captured webhook from Test 3.1
2. Search for `log_outcome` function call in payload
3. Verify parsing code extracts:
   - `outcome` = "READY_TO_PAY" (or whatever you said)
   - `payment_method` = "credit_card"
   - `notes` = conversation summary
   - `promised_date` = null (or date if promised)

4. Add debug logging to "Parse Call Data" node:
```javascript
// After parsing function call
console.log('Extracted outcome:', outcome);
console.log('Extracted payment method:', paymentMethod);
console.log('Extracted notes:', notes);

return { /* ... */ };
```

5. Make another test call
6. Check n8n logs for extracted values
7. Verify values match what you said on the call

**If Failed**:
- Update parsing logic to match actual payload structure
- Check if function calls are in different location (e.g., `call.analysis.structuredData`)

---

### Test 3.4: No-Answer Scenario

**Purpose**: Verify behavior when debtor doesn't answer

**Steps**:

1. Add test debtor with WRONG phone number (123456789)
2. Execute Workflow 1
3. Call will fail or go to voicemail
4. Capture webhook payload
5. Verify:
   - Webhook still received (or check if Vapi sends webhook for unanswered calls)
   - `outcome` set to "NO_ANSWER" correctly
   - `next_call_date` set to +4 hours

**If Failed**:
- Vapi may not send `end-of-call-report` for unanswered calls
- Need to use `status-update` event instead
- Update Workflow 2 to handle both event types

---

## Phase 4: Integration Tests

### Test 4.1: Full End-to-End Call Flow

**Purpose**: Verify complete flow from scheduling to verbal payment delivery

**Steps**:

1. Add test debtor with your phone number
2. Set `next_call_date = now`
3. Execute Workflow 1 (Call Scheduler)
4. Answer call
5. Interact with AI:
   - Verify identity (provide fake DOB)
   - Listen to debt details
   - Say "I'd like to pay by bank transfer"
   - **CRITICAL**: Verify AI asks if you have pen and paper ready
   - **CRITICAL**: Verify AI reads BSB, account number, account name slowly
   - **CRITICAL**: Verify AI repeats payment details
   - **CRITICAL**: Verify AI confirms you wrote down all details
   - Write down the payment details AI provides
6. End call
7. Wait 30 seconds
8. Check Google Sheets:
   - [ ] `call_status` = "completed"
   - [ ] `last_call_date` populated
   - [ ] `last_outcome` = "READY_TO_PAY"
   - [ ] `attempt_number` incremented to 2
   - [ ] `notes` contains "payment details provided verbally"
   - [ ] `notes` contains payment method = "bank_transfer"
   - [ ] `recording_url` populated
   - [ ] `assigned_to_ai` = FALSE (waiting for payment)
   - [ ] `next_action` = "AWAIT_PAYMENT"
   - [ ] `next_call_date` = +3 days from now
9. Verify payment details you wrote down match:
   - [ ] BSB number from `.env`
   - [ ] Account number from `.env`
   - [ ] Account name from `.env`

**Expected Duration**: 3-4 minutes

**Note**: No SMS sent in Phase 1. Payment instructions delivered verbally during call.

**If Failed**: Check execution history for each workflow, review errors, verify verbal payment instructions in Vapi assistant configs

---

### Test 4.2: Promise to Pay Flow

**Purpose**: Verify scheduled follow-up for promise to pay

**Steps**:

1. Make test call
2. Say "I'll pay by Friday" (October 10)
3. End call
4. Check Google Sheets:
   - [ ] `last_outcome` = "PROMISE_TO_PAY"
   - [ ] `next_call_date` = October 11 (day after promised date)
   - [ ] `notes` contains "promised to pay by Oct 10"
   - [ ] `assigned_to_ai` = TRUE (still in queue for follow-up)
   - [ ] `next_action` = "SCHEDULE_CALL"
5. Verify:
   - [ ] NO SMS sent (Phase 1 is voice-only)
   - [ ] NO payment details provided verbally (only for READY_TO_PAY)

**If Failed**: Review outcome routing logic in "Calculate Next Action" node

---

### Test 4.3: Dispute Raised Flow

**Purpose**: Verify manual review triggered

**Steps**:

1. Make test call
2. Say "I don't owe this money"
3. End call
4. Check Google Sheets:
   - [ ] `last_outcome` = "DISPUTE_RAISED"
   - [ ] `next_action` = "MANUAL_REVIEW"
   - [ ] `assigned_to_ai` = FALSE (removed from AI queue)
5. Check email:
   - [ ] [CLIENT] received "Manual Review Required" email
   - [ ] Email contains debtor ID and dispute details

**If Failed**: Check email node configuration, verify SMTP settings

---

### Test 4.4: Do Not Call Request

**Purpose**: Verify compliance with do-not-call requests

**Steps**:

1. Make test call
2. Say "Stop calling me"
3. End call
4. Check Google Sheets:
   - [ ] `last_outcome` = "REQUESTED_NO_CONTACT"
   - [ ] `do_not_call` = TRUE
   - [ ] `assigned_to_ai` = FALSE
5. Set `next_call_date = now` (try to trigger another call)
6. Execute Workflow 1
7. Verify:
   - [ ] Debtor NOT called (filtered out by do_not_call check)

**Expected Result**: AI respects do-not-call flag

**If Failed**: Review filter logic in Workflow 1

---

## Phase 5: Edge Cases & Error Handling

### Test 5.1: Invalid Phone Number

**Purpose**: Verify graceful handling of invalid phones

**Test Cases**:
- Debtor with phone = ""
- Debtor with phone = "invalid"
- Debtor with phone = "+1234" (too short)

**Expected Result**: Skipped or logged error, no crash

---

### Test 5.2: Duplicate Call Prevention

**Purpose**: Verify `call_status` lock prevents duplicate calls

**Steps**:

1. Add test debtor with `next_call_date = now`
2. Execute Workflow 1
3. Verify `call_status` = "calling"
4. Immediately execute Workflow 1 again (before call ends)
5. Verify:
   - [ ] Debtor NOT called twice (filtered out)

**Expected Result**: Only one call initiated

**If Failed**: Check filter logic for `call_status === 'calling'`

---

### Test 5.3: Vapi API Call Failure

**Purpose**: Verify error handling when Vapi API fails

**Steps**:

1. Temporarily use invalid Vapi API key
2. Execute Workflow 1
3. Verify:
   - [ ] Workflow doesn't crash
   - [ ] `call_status` reset to null
   - [ ] Error logged in notes

**Expected Result**: Graceful failure

**If Failed**: Add try/catch in error handler

---

### Test 5.4: Max Attempts Reached

**Purpose**: Verify debtor stopped after 3 attempts

**Steps**:

1. Set `attempt_number = 4` for test debtor
2. Execute Workflow 1
3. Verify debtor NOT called (filtered out)

**Expected Result**: Respects max 3 attempts rule

**If Failed**: Check filter logic for `attempt_number > 3`

---

## Phase 6: Compliance Tests

### Test 6.1: Business Hours Enforcement

**Test Cases**:

| Scenario | Time (AEST) | Should Call? |
|----------|-------------|--------------|
| Weekday morning | 7:00am | NO |
| Weekday morning | 7:30am | YES |
| Weekday evening | 8:59pm | YES |
| Weekday night | 9:00pm | NO |
| Saturday morning | 8:30am | NO |
| Saturday morning | 9:00am | YES |
| Sunday afternoon | 2:00pm | YES |
| Sunday night | 9:00pm | NO |

**Steps**: Modify system time or hardcode test times, verify filtering

---

### Test 6.2: Recording Consent

**Purpose**: Verify AI announces call recording

**Steps**:

1. Make test call
2. Listen to opening script
3. Verify AI says: "This call may be recorded for quality and compliance purposes"

**Expected Result**: Compliance disclosure made

**If Failed**: Update assistant system prompt

---

### Test 6.3: Identity Verification

**Purpose**: Verify AI doesn't discuss debt before identity confirmed

**Steps**:

1. Make test call
2. When AI asks for DOB, provide WRONG DOB
3. Verify AI refuses to proceed with debt discussion
4. End call
5. Make another call
6. Provide CORRECT DOB
7. Verify AI proceeds to discuss debt

**Expected Result**: Identity verification enforced

**If Failed**: Update assistant system prompt

---

## Phase 7: Load & Performance Tests

### Test 7.1: Multiple Concurrent Calls

**Purpose**: Verify system handles multiple calls

**Steps**:

1. Add 10 test debtors with `next_call_date = now`
2. Execute Workflow 1
3. Verify all 10 calls initiated
4. Check for errors or failed calls

**Expected Result**: All calls succeed

**If Failed**: Check Vapi concurrent call limits, add rate limiting

---

### Test 7.2: Large Google Sheets

**Purpose**: Verify performance with 1000 debtors

**Steps**:

1. Generate 1000 test rows in Google Sheets
2. Execute Workflow 1
3. Measure execution time
4. Verify workflow completes in <30 seconds

**Expected Result**: Reasonable performance

**If Failed**: Consider pagination or caching

---

## Test Results Tracking

### Test Execution Log

| Test ID | Test Name | Date | Result | Notes |
|---------|-----------|------|--------|-------|
| 1.1 | Env vars loaded (14 vars) | | ⬜ Pass / ❌ Fail | |
| 1.2 | Google Sheets connection | | ⬜ Pass / ❌ Fail | |
| 1.3 | Vapi API key valid | | ⬜ Pass / ❌ Fail | |
| 2.1 | Phone normalization | | ⬜ Pass / ❌ Fail | |
| 2.2 | Assistant selection | | ⬜ Pass / ❌ Fail | |
| 2.3 | Business hours | | ⬜ Pass / ❌ Fail | |
| 3.1 | Webhook payload capture | | ⬜ Pass / ❌ Fail | |
| 3.2 | Signature verification | | ⬜ Pass / ❌ Fail | |
| 3.3 | Function call parsing | | ⬜ Pass / ❌ Fail | |
| 3.4 | No-answer scenario | | ⬜ Pass / ❌ Fail | |
| 4.1 | Full E2E verbal payment | | ⬜ Pass / ❌ Fail | |
| 4.2 | Promise to pay | | ⬜ Pass / ❌ Fail | |
| 4.3 | Dispute raised | | ⬜ Pass / ❌ Fail | |
| 4.4 | Do not call | | ⬜ Pass / ❌ Fail | |

---

## Critical Issues Found During Testing

Document any issues discovered:

### Issue Template

**Issue #**:
**Test ID**:
**Description**:
**Actual Behavior**:
**Expected Behavior**:
**Root Cause**:
**Fix Applied**:
**Re-test Result**:

---

## Sign-Off

### Testing Complete Checklist

- [ ] All Phase 1 tests passed
- [ ] All Phase 2 tests passed
- [ ] All Phase 3 tests passed (CRITICAL)
- [ ] All Phase 4 tests passed
- [ ] All Phase 5 tests passed
- [ ] All Phase 6 tests passed
- [ ] Test results documented
- [ ] All critical issues resolved
- [ ] Re-tested after fixes
- [ ] System ready for production

### Tested By

**Name**: _________________
**Date**: _________________
**Signature**: _________________

### Approved By

**Name**: [CLIENT NAME]
**Date**: _________________
**Signature**: _________________

---

**End of Testing Guide**

**Last Updated**: October 3, 2025
**Next Action**: Execute Phase 1 tests
