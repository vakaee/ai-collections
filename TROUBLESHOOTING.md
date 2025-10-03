# Troubleshooting Guide - BCS Debt Collection Bot

**Phase 1 Scope**: Voice-only outbound calling
**Phase 2 Scope** (not included): SMS + Email automation

## Common Issues and Solutions

---

## 1. Vapi Issues

### Issue 1.1: Vapi Call Not Initiated

**Symptoms**:
- Workflow 1 executes successfully
- No error in n8n
- Debtor's phone doesn't ring

**Possible Causes**:
1. Phone number format incorrect (not E.164)
2. Vapi API key invalid
3. Assistant ID incorrect
4. Phone number not purchased/active

**Solutions**:
```javascript
// Check phone normalization
console.log($node['Normalize Phone'].json.phone_normalized);
// Should output: +61412345678

// Test Vapi API key
curl https://api.vapi.ai/assistant \
  -H "Authorization: Bearer $VAPI_API_KEY"
// Should return 200 OK

// Verify assistant ID exists
curl https://api.vapi.ai/assistant/{ASSISTANT_ID} \
  -H "Authorization: Bearer $VAPI_API_KEY"
```

**Fix**:
- Check phone format in Google Sheets
- Verify `.env` has correct `VAPI_API_KEY`
- Verify assistant IDs copied correctly to `.env`
- Check Vapi dashboard for phone number status

---

### Issue 1.2: Vapi Webhook Not Received

**Symptoms**:
- Call completes
- Debtor interacts with AI
- Workflow 2 never triggers
- Google Sheets not updated

**Possible Causes**:
1. Webhook URL incorrect in Vapi
2. n8n not accessible from internet (if production)
3. Webhook node not active

**Solutions**:
```bash
# Test webhook locally
curl -X POST http://localhost:5678/webhook/vapi-handler \
  -H "Content-Type: application/json" \
  -d '{"test": "data"}'

# Should return 200 OK
```

**Fix**:
- Go to Vapi Dashboard → Settings → Webhooks
- Verify Server URL matches n8n webhook URL
- If local: Use ngrok for public URL: `ngrok http 5678`
- Verify Workflow 2 is "Active" (not "Inactive")

---

### Issue 1.3: Signature Verification Fails

**Symptoms**:
- Error in Workflow 2: "Invalid webhook signature"
- Call completes but outcome not logged

**Possible Causes**:
1. `VAPI_WEBHOOK_SECRET` incorrect
2. Signature header name wrong
3. Payload modification before verification

**Solutions**:
```javascript
// Debug signature (UPDATED Oct 2025 - header name is capital X)
console.log('Received signature (capital):', $input.headers['X-Vapi-Signature']);
console.log('Received signature (lowercase):', $input.headers['x-vapi-signature']);
console.log('Webhook secret:', $vars.VAPI_WEBHOOK_SECRET);
console.log('Payload:', JSON.stringify($input.body));
```

**Fix**:
- **VERIFIED**: Header name is `X-Vapi-Signature` (capital X)
- Update code to use: `$input.headers['X-Vapi-Signature']`
- Regenerate webhook secret in Vapi Dashboard if needed
- Update `.env` with new `VAPI_WEBHOOK_SECRET`
- Restart n8n: `docker-compose restart`

---

### Issue 1.4: AI Doesn't Call Functions

**Symptoms**:
- Call completes normally
- No `log_outcome` function called
- Webhook receives incomplete data

**Possible Causes**:
1. Function definition missing or incorrect
2. System prompt doesn't instruct to call function
3. LLM doesn't understand when to call

**Solutions**:
- Review call transcript in Vapi Dashboard
- Check function definition JSON (valid syntax?)
- Check system prompt mentions calling log_outcome

**Fix**:
- Go to Vapi Dashboard → Assistant → Functions
- Verify `log_outcome` and `verify_identity` both added
- Update system prompt: "ALWAYS call log_outcome function before ending"
- Test with Test Assistant feature in Vapi

---

### Issue 1.5: Webhook Payload Parsing Fails (NEW - Oct 2025)

**Symptoms**:
- Webhook received successfully
- Error: "Cannot read property 'xxx' of undefined"
- Outcome not extracted from webhook

**Possible Causes**:
1. Vapi webhook payload structure different than expected
2. Function call results in unexpected location
3. Webhook type not "end-of-call-report"

**Solutions**:
```javascript
// Add logging to "Parse Call Data" node to inspect actual structure
console.log('===== FULL WEBHOOK PAYLOAD =====');
console.log(JSON.stringify($input.item.json, null, 2));
console.log('===== END PAYLOAD =====');

// Check expected structure
const message = $input.item.json.message;
console.log('Message type:', message?.type);
console.log('Call object exists:', !!message?.call);
console.log('Artifact exists:', !!message?.artifact);
console.log('Messages array length:', message?.artifact?.messages?.length);
```

**Expected Payload Structure** (verified Oct 2025):
```json
{
  "message": {
    "type": "end-of-call-report",
    "endedReason": "assistant-ended-call",
    "call": {
      "id": "call_xxx",
      "status": "ended",
      "startedAt": "...",
      "endedAt": "...",
      "assistantOverrides": {
        "variableValues": {
          "debtor_id": "123"
        }
      }
    },
    "artifact": {
      "recording": {
        "url": "https://..."
      },
      "transcript": "...",
      "messages": [
        {
          "role": "assistant",
          "message": "...",
          "toolCalls": [
            {
              "function": {
                "name": "log_outcome",
                "arguments": {
                  "outcome": "PROMISE_TO_PAY",
                  "payment_method": "credit_card",
                  "notes": "...",
                  "promised_date": "2025-10-10"
                }
              }
            }
          ]
        }
      ]
    }
  }
}
```

**Fix**:
- Compare actual payload against expected structure above
- Update parsing code in "Parse Call Data" node if structure differs
- See `VAPI_API_VERIFICATION.md` for latest verified structure
- Run Test 3.1 in `TESTING_GUIDE.md` to capture real payload

**Common Issues**:
- Function call in `artifact.messages[].toolCalls` (correct)
- NOT in `call.messages` or `message.toolCalls`
- `arguments` may be string (need JSON.parse) or object
- Check `message.type` === "end-of-call-report" before parsing

---

### Issue 1.6: Phone Number ID vs Phone Number (NEW - Oct 2025)

**Symptoms**:
- Error: "Invalid phoneNumberId"
- Vapi API rejects call creation

**Cause**:
- Using actual phone number (+61...) instead of phone number ID (ph_xxx)

**Fix**:
- Go to Vapi Dashboard → Phone Numbers
- Copy the **ID** (starts with `ph_`), not the phone number
- Update `.env`: `VAPI_PHONE_NUMBER_ID=ph_abc123...`
- Use `$vars.VAPI_PHONE_NUMBER_ID` in API call, not `VAPI_PHONE_NUMBER`

---

## 2. Google Sheets Issues

### Issue 2.1: Can't Read Google Sheets

**Symptoms**:
- Workflow 1 error: "Could not read from Google Sheets"
- Authentication failed

**Possible Causes**:
1. Google Sheets credentials expired
2. Spreadsheet ID incorrect
3. Sheet not shared with service account
4. Sheet range incorrect

**Solutions**:
```bash
# Verify spreadsheet ID from URL
# https://docs.google.com/spreadsheets/d/{SPREADSHEET_ID}/edit
# Copy ID to .env
```

**Fix**:
- Re-authenticate Google Sheets in n8n (Settings → Credentials)
- Verify `GOOGLE_SHEETS_SPREADSHEET_ID` in `.env`
- If using service account: Share sheet with service account email
- Check range: Should be `A2:W1000` (23 columns)

---

### Issue 2.2: Can't Update Google Sheets

**Symptoms**:
- Workflow 2 error: "Could not update row"
- Read works but write fails

**Possible Causes**:
1. Row number incorrect
2. Column references wrong (A-W)
3. Write permissions missing

**Solutions**:
```javascript
// Debug row lookup
console.log('Row number:', $node['Lookup Row'].json.__rowNum__);
// Should output: 2, 3, 4, etc. (never 1 - that's the header)

// Verify column positions
// Column M = call_status (13th column)
// Column N = call_id (14th column)
// ...
```

**Fix**:
- Verify Google Sheets node has "Update Row" operation
- Check row number expression: `{{$node['Lookup Row'].json.__rowNum__}}`
- Verify credentials have edit access
- Test with Google Sheets "Append" instead of "Update" first

---

### Issue 2.3: Duplicate Calls (call_status Not Working)

**Symptoms**:
- Debtor called multiple times in short period
- call_status column not updating

**Possible Causes**:
1. call_status update node missing
2. Filter logic incorrect
3. Race condition (cron too frequent)

**Solutions**:
```javascript
// Check filter logic in Workflow 1
const filtered = items.filter(item => {
  console.log('call_status:', item.json.call_status);
  return item.json.call_status !== 'calling';
});
```

**Fix**:
- Verify "Update call_status" node BEFORE Vapi call
- Check filter excludes `call_status = "calling"`
- Increase cron frequency (e.g., every 60 min instead of 30 min)

---

## 3. n8n Issues

### Issue 3.1: Workflow Not Executing

**Symptoms**:
- Cron trigger not firing
- Manual execution works

**Possible Causes**:
1. Workflow not activated
2. Timezone issue (cron in UTC)
3. n8n container restarted

**Solutions**:
```bash
# Check n8n logs
docker logs n8n

# Check workflow status
# In n8n UI, verify "Active" toggle is ON
```

**Fix**:
- Click "Active" toggle in workflow (top right)
- Set timezone in `.env`: `TZ=Australia/Sydney`
- Restart n8n: `docker-compose restart`

---

### Issue 3.2: Code Node Error

**Symptoms**:
- Workflow stops at Code node
- Error: "ReferenceError: X is not defined"

**Possible Causes**:
1. Variable not defined in code
2. Syntax error in JavaScript
3. Node reference incorrect

**Solutions**:
```javascript
// Debug variables
console.log('Input:', $input.item.json);
console.log('Previous node:', $node['Previous Node Name'].json);
```

**Fix**:
- Check JavaScript syntax (missing semicolons, brackets)
- Verify node names match exactly (case-sensitive)
- Use `$vars` for environment variables: `$vars.VAPI_API_KEY`

---

### Issue 3.3: Environment Variables Not Loaded

**Symptoms**:
- Error: "$vars.VAPI_API_KEY is undefined"
- Variables work locally but not in Docker

**Possible Causes**:
1. `.env` file not in correct location
2. Docker Compose not loading `.env`
3. Typo in variable name

**Solutions**:
```bash
# Check .env file location
ls -la .env
# Should be in same directory as docker-compose.yml

# Check Docker environment
docker exec n8n env | grep VAPI
# Should show VAPI_API_KEY value
```

**Fix**:
- Verify `.env` file exists
- Add to `docker-compose.yml`: `env_file: - .env`
- Restart: `docker-compose down && docker-compose up -d`
- Wait 30 seconds for n8n to restart

---

## 4. Business Logic Issues

### Issue 4.1: Business Hours Check Not Working

**Symptoms**:
- Calls made outside 7:30am-9pm
- Timezone incorrect

**Possible Causes**:
1. Timezone column empty
2. Timezone calculation wrong
3. System time incorrect

**Solutions**:
```javascript
// Debug business hours
const timezone = data.timezone || 'AEST';
const now = new Date();
console.log('Current time:', now.toISOString());
console.log('Debtor timezone:', timezone);
console.log('Is business hours:', isBusinessHours(timezone));
```

**Fix**:
- Fill `timezone` column in Google Sheets for all debtors
- Verify timezone offsets in code:
  - AEST: UTC+10
  - ACST: UTC+9.5
  - AWST: UTC+8
- Set container timezone: `TZ=Australia/Sydney` in `.env`

---

### Issue 4.2: Wrong Assistant Selected

**Symptoms**:
- Consumer gets commercial script
- 1st call gets 3rd call script

**Possible Causes**:
1. debtor_type column incorrect
2. attempt_number not incrementing
3. Assistant ID mapping wrong

**Solutions**:
```javascript
// Debug assistant selection
const debtorType = data.debtor_type; // Should be "consumer" or "commercial"
const attemptNumber = data.attempt_number; // Should be 1, 2, or 3
const key = `${debtorType}_${attemptNumber}`; // Should be "consumer_1", etc.
console.log('Assistant key:', key);
console.log('Assistant ID:', assistantMap[key]);
```

**Fix**:
- Verify `debtor_type` in Google Sheets (lowercase: "consumer" or "commercial")
- Check `attempt_number` starts at 1
- Verify all 6 assistant IDs in `.env`

---

### Issue 4.3: Payment Details Not Provided Verbally

**Symptoms**:
- Call outcome = "READY_TO_PAY"
- AI didn't read payment details to debtor

**Possible Causes**:
1. Vapi assistant missing verbal payment instructions
2. AI didn't ask for pen and paper
3. payment_method not set correctly

**Solutions**:
- Review call recording/transcript in Vapi Dashboard
- Check if AI asked "Do you have pen and paper ready?"
- Verify payment method was identified (bank_transfer, credit_card, cheque)

**Fix**:
- Review VAPI_ASSISTANT_CONFIGS.md "Verbal Payment Instructions" section
- Update assistant system prompts to include payment delivery scripts
- Ensure AI confirms debtor wrote down all details before ending call
- Test with different payment methods

---

## 5. Compliance Issues

### Issue 5.1: Calling Outside Business Hours

**Symptoms**:
- Compliance violation
- Calls at night or early morning

**Fix**:
- Verify business hours check in Workflow 1
- Set correct timezones for all debtors
- Test with different timezones (AEST, ACST, AWST)

---

### Issue 5.2: Calling After "Do Not Call" Request

**Symptoms**:
- Debtor requested no contact
- Still receiving calls

**Fix**:
- Verify `do_not_call` flag set after REQUESTED_NO_CONTACT outcome
- Check filter excludes `do_not_call = TRUE`
- Manually verify Google Sheets row updated

---

### Issue 5.3: More Than 3 Attempts

**Symptoms**:
- Debtor called 4+ times

**Fix**:
- Verify attempt_number increments after each call
- Check filter excludes `attempt_number > 3`
- Add validation in Code node

---

## 6. Testing Issues

### Issue 6.1: Test Webhook Not Working

**Symptoms**:
- Workflow 3 (Test Webhook) executes
- Workflow 2 (Vapi Webhook Handler) not triggered

**Fix**:
- Verify webhook URL in Workflow 3 HTTP Request node
- Check Workflow 2 is active
- Use correct payload structure (see `WORKFLOW_ADAPTATION_PLAN.md` Section 5.2)

---

### Issue 6.2: Can't Test Locally

**Symptoms**:
- Vapi can't reach local n8n webhook

**Solution**: Use ngrok
```bash
# Install ngrok
brew install ngrok  # Mac
# OR download from ngrok.com

# Create tunnel
ngrok http 5678

# Copy public URL (e.g., https://abc123.ngrok.io)
# Update Vapi webhook URL to: https://abc123.ngrok.io/webhook/vapi-handler
```

---

## 7. Data Issues

### Issue 7.1: Debtor Not Called

**Symptoms**:
- Debtor in Google Sheets
- next_call_date in past
- Not being called

**Checklist**:
- [ ] `assigned_to_ai = TRUE`?
- [ ] `payment_status = "unpaid"`?
- [ ] `call_status != "calling"`?
- [ ] `do_not_call != TRUE`?
- [ ] `attempt_number <= 3`?
- [ ] Within business hours for timezone?

**Fix**: Check all filter conditions in Workflow 1

---

### Issue 7.2: Outcome Not Logged

**Symptoms**:
- Call completes
- Google Sheets not updated

**Debug Steps**:
1. Check Vapi Dashboard → Call Logs → Find call
2. Review transcript - was `log_outcome` called?
3. Check Workflow 2 execution history
4. Check Workflow 2 "Verify Signature" node - passed?

---

## 8. Performance Issues

### Issue 8.1: Workflows Slow

**Symptoms**:
- Workflows take > 10 seconds

**Possible Causes**:
1. Large Google Sheets (1000+ rows)
2. Complex Code nodes
3. n8n resource limits

**Solutions**:
- Reduce Google Sheets range (e.g., A2:W500)
- Optimize filter logic
- Increase Docker resources (CPU/RAM)

---

### Issue 8.2: Concurrent Calls Failing

**Symptoms**:
- Multiple calls scheduled at same time
- Some fail

**Possible Causes**:
1. Vapi plan concurrent call limit
2. Rate limiting

**Solutions**:
- Upgrade Vapi plan
- Add delay between calls (Wait node in n8n)
- Stagger next_call_date times

---

## 9. Getting Help

### Debug Checklist
- [ ] Check n8n workflow execution history
- [ ] Check Vapi Dashboard call logs
- [ ] Check Google Sheets data
- [ ] Check Docker logs: `docker logs n8n`
- [ ] Check `.env` variables loaded

### Logs to Check
```bash
# n8n logs
docker logs n8n --tail 100

# n8n exec logs (for workflow errors)
docker exec n8n cat /home/node/.n8n/logs/n8n.log

# System time (for business hours)
date
```

### Contact Support
- Vapi support: support@vapi.ai
- n8n community: https://community.n8n.io

**Phase 2 Support** (when commissioned):
- Twilio support: https://support.twilio.com

---

## Related Documentation

- `WORKFLOW_ADAPTATION_PLAN.md` - Complete workflow code
- `IMPLEMENTATION_CHECKLIST.md` - Setup steps
- `VAPI_ASSISTANT_CONFIGS.md` - Assistant configuration
- `COMPLIANCE.md` - Australian regulations

---

**End of Troubleshooting Guide**

**Last Updated**: October 3, 2025
