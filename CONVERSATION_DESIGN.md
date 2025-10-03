# Conversation Design & Call Flows

## Document Information

**Project**: AI Debt Collection Bot - Phase 1
**Version**: 1.0
**Last Updated**: October 3, 2025
**Status**: Awaiting scripts from Peter

---

## 1. Overview

This document defines the conversation structure, call flows, objection handling, and outcome taxonomy for the AI debt collection voice bot.

**Note**: Detailed scripts are pending from Peter. This document provides the framework and placeholders that will be populated once scripts are received.

---

## 2. Call Flow State Machine

```
START
  ↓
GREETING
  - Identity disclosure: "This is [Agent] from Brodie Collection Services"
  - Purpose statement: "This call is regarding a debt collection matter"
  - Recording notice: "This call may be recorded..."
  ↓
REQUEST_DEBTOR
  - "May I please speak with [DEBTOR_NAME]?"
  ↓
┌─────────────────┐
│ Is debtor?      │
└─────────────────┘
   ↓ NO              ↓ YES
THIRD_PARTY      VERIFY_IDENTITY
  - Privacy rules     - Request DOB / Last 4 phone / Address
  - Request callback  ↓
  - END_CALL      ┌─────────────────┐
                  │ Verified?       │
                  └─────────────────┘
                     ↓ NO (2x)          ↓ YES
                  ESCALATE_HUMAN     STATE_DEBT
                                       - Amount: $X
                                       - Creditor: Y
                                       - Date: Z
                                       ↓
                                    REQUEST_PAYMENT
                                       - 7-day deadline
                                       - Payment methods
                                       ↓
                                    HANDLE_RESPONSE
                                       ↓
              ┌────────────────────────┼────────────────────────┐
              ↓                        ↓                        ↓
        AGREES_TO_PAY          RAISES_OBJECTION         ESCALATION_TRIGGER
         - Promise to pay       - Dispute                - "Lawyer"
         - Payment plan         - Hardship               - "Ombudsman"
         - When?                - Already paid           - Abusive language
         ↓                      - Wrong amount           ↓
    LOG_OUTCOME                ↓                     TRANSFER_HUMAN
    (PROMISE_TO_PAY)      HANDLE_OBJECTION
    ↓                          ↓
END_CALL                  LOG_OUTCOME
                          (DISPUTE_RAISED,
                           HARDSHIP_CLAIMED, etc.)
                           ↓
                      SEND_EMAIL (if applicable)
                           ↓
                      END_CALL
```

---

## 3. Conversation Scripts

### 3.1 Consumer Debt - 1st Call

**Status**: AWAITING SCRIPTS FROM PETER

**Draft Script** (based on Letter of Demand template):

```
[GREETING]
Hello, this is [AGENT_NAME] calling from Brodie Collection Services. This call is regarding a debt collection matter and may be recorded for quality and compliance purposes.

[REQUEST_DEBTOR]
May I please speak with [DEBTOR_NAME]?

[If person confirms they are DEBTOR_NAME]

[VERIFY_IDENTITY]
Thank you. For security purposes, can you please confirm your date of birth?

[If DOB matches CRM record]

[STATE_DEBT]
I'm calling regarding an outstanding debt of $[AMOUNT] owed to [CREDITOR_NAME]. Our records show this account from [DATE] remains unpaid despite previous communication.

[REQUEST_PAYMENT]
We kindly request full payment within 7 days to avoid further action. You can make payment via:
- Bank transfer to BSB [BSB_NUMBER], Account [ACCOUNT_NUMBER], Reference [REFERENCE]
- Credit card by calling [BCS_PHONE_NUMBER]
- Cheque mailed to [BCS_ADDRESS]

May I confirm you'll be making this payment within the next 7 days?

[HANDLE_RESPONSE - see Objection Handling section]

[END_CALL]
Thank you for your time. If you have any questions, please contact us at [BCS_PHONE_NUMBER]. Goodbye.

[Log outcome via function call]
```

---

### 3.2 Consumer Debt - 2nd Call

**Status**: AWAITING SCRIPTS FROM PETER

**Draft Script**:

```
[GREETING - same as 1st call]

[VERIFY_IDENTITY - same as 1st call]

[STATE_DEBT]
I'm calling as a follow-up regarding the outstanding debt of $[AMOUNT] owed to [CREDITOR_NAME]. We haven't received payment since our previous contact on [LAST_CALL_DATE].

[CONSEQUENCES]
I want to remind you that continued non-payment may result in:
- Legal action and additional costs
- A payment default being listed on your credit file, which will affect your ability to obtain credit for up to 5 years

[REQUEST_PAYMENT]
To avoid these consequences, we urge you to make payment within the next 7 days.

[Payment methods - same as 1st call]

[HANDLE_RESPONSE]

[END_CALL]
```

---

### 3.3 Consumer Debt - 3rd Call (Final Warning)

**Status**: AWAITING SCRIPTS FROM PETER

**Draft Script**:

```
[GREETING - same as 1st call]

[VERIFY_IDENTITY - same as 1st call]

[STATE_DEBT]
This is a final call regarding the outstanding debt of $[AMOUNT] owed to [CREDITOR_NAME]. Despite our previous attempts to contact you on [DATES], this account remains unpaid.

[FINAL WARNING]
This is your final opportunity to resolve this matter before we proceed with:
- Referral to our client's solicitor for legal proceedings
- Listing a payment default on your credit file with credit reporting agencies
- Additional legal costs and interest being added to the debt

[REQUEST_PAYMENT]
If you make full payment within the next 7 days, we can avoid these actions.

[Payment methods - same as 1st call]

[HANDLE_RESPONSE]

[END_CALL]
This matter will now be flagged for escalation. You may still contact us at [BCS_PHONE_NUMBER] to discuss payment. Goodbye.

[Log outcome + flag for manual review]
```

---

### 3.4 Commercial Debt - 1st Call

**Status**: AWAITING SCRIPTS FROM PETER

**Key Differences from Consumer Script**:

1. **Tone**: More formal, address "your company" instead of "you"
2. **Consequences**: Emphasize ASIC payment default, director liability
3. **No hardship option**: Remove financial hardship objection handling
4. **Legal emphasis**: Reference "our client's solicitor" more prominently

**Draft Script**:

```
[GREETING]
Good morning/afternoon, this is [AGENT_NAME] calling from Brodie Collection Services on behalf of [CREDITOR_NAME]. This call is regarding a commercial debt collection matter and may be recorded.

[REQUEST_CONTACT]
May I please speak with [DEBTOR_NAME] or the accounts payable manager?

[VERIFY_IDENTITY]
For security purposes, can you please confirm your role at [COMPANY_NAME] and the company's ABN?

[STATE_DEBT]
I'm calling regarding an outstanding commercial debt of $[AMOUNT] owed to [CREDITOR_NAME] under invoice [INVOICE_NUMBER] dated [DATE].

[REQUEST_PAYMENT]
Full payment is requested within 7 days. You can remit payment via bank transfer to BSB [BSB], Account [ACCOUNT], Reference [REFERENCE].

[CONSEQUENCES]
Please note that failure to pay may result in:
- A payment default being listed with ASIC, which will permanently affect your company's credit rating and ability to secure finance
- Referral to our client's solicitor for legal proceedings, which may include director liability
- Additional legal costs and interest

[HANDLE_RESPONSE]

[END_CALL]
```

---

### 3.5 Commercial Debt - 2nd & 3rd Calls

**Status**: AWAITING SCRIPTS FROM PETER

**Follow same escalation pattern as consumer calls**, with commercial-specific consequences.

---

## 4. Objection Handling

### 4.1 Dispute

**Debtor says**: "I don't owe this" / "This debt is not mine" / "I dispute this"

**AI Response**:
```
I understand you're disputing this debt. To formally dispute it, you must submit your dispute in writing to [DISPUTES_EMAIL] within 7 days. Please include:
- Your name and account reference
- Details of why you believe the debt is incorrect
- Any supporting documentation

I'll note your dispute in our system and pause collection activity pending review.
```

**Action**:
- Log outcome: `DISPUTE_RAISED`
- Send email with dispute instructions
- Flag for manual review (no further calls until dispute resolved)

---

### 4.2 Financial Hardship (Consumer Only)

**Debtor says**: "I can't afford to pay" / "I lost my job" / "Financial hardship"

**AI Response**:
```
I understand you're experiencing financial difficulty. We can send you a financial hardship form to complete, which will allow us to assess potential payment arrangements.

What email address should I send the form to?
```

**Action**:
- Log outcome: `HARDSHIP_CLAIMED`
- Collect email address
- Send hardship form template (Peter to provide)
- Flag for manual review

---

### 4.3 Payment Plan Request

**Debtor says**: "Can I pay in installments?" / "I can pay $X per month"

**AI Response**:
```
I understand you'd like to arrange a payment plan. I can note your request for our team to review.

What amount can you afford to pay, and how frequently? (e.g., $100 per week, $500 per month)
```

**Action**:
- Log outcome: `PAYMENT_PLAN_REQUESTED`
- Capture proposed amount and frequency in notes
- Flag for manual review (human approves payment plans)

---

### 4.4 Already Paid

**Debtor says**: "I already paid this" / "I paid last month"

**AI Response**:
```
Thank you for letting me know. Can you please provide details of your payment, such as:
- Date of payment
- Payment method (bank transfer, credit card, cheque)
- Transaction reference number or receipt

Once we verify your payment, we'll update our records. You can also email proof of payment to [ACCOUNTS_EMAIL].
```

**Action**:
- Log outcome: `CLAIMS_PAID`
- Request proof of payment
- Flag for manual review (human checks payment records)

---

### 4.5 Wrong Amount

**Debtor says**: "The amount is wrong" / "I only owe $X, not $Y"

**AI Response**:
```
I understand there's a discrepancy. Our records show $[AMOUNT] owed under [INVOICE/ACCOUNT]. If you believe this is incorrect, please submit details in writing to [DISPUTES_EMAIL] with supporting documentation.
```

**Action**:
- Log outcome: `AMOUNT_DISPUTED`
- Same as DISPUTE_RAISED
- Flag for manual review

---

### 4.6 Callback Request

**Debtor says**: "Can you call back later?" / "I'm busy right now"

**AI Response**:
```
Of course. When would be a convenient time for me to call back?

[Debtor provides time]

Thank you. I'll schedule a callback for [DATE/TIME]. If you'd prefer to call us directly, our number is [BCS_PHONE].
```

**Action**:
- Log outcome: `CALLBACK_REQUESTED`
- Schedule next call at requested time
- Ensure it's within business hours

---

### 4.7 "Stop Calling Me" / Cease Contact Request

**Debtor says**: "Stop calling me" / "Don't contact me again" / "Remove me from your list"

**AI Response**:
```
I understand. I've noted your request to cease phone contact. You may still receive written correspondence regarding this matter. If you wish to discuss payment or dispute the debt, you can contact us at [BCS_PHONE].
```

**Action**:
- Log outcome: `CEASE_CONTACT_REQUESTED`
- Flag for manual review
- **IMPORTANT**: Do NOT make further calls to this debtor (legal requirement)
- Note: Written contact (letters) may still be permitted

---

## 5. Escalation Triggers (Transfer to Human)

### 5.1 Legal Representation

**Trigger words**: "lawyer," "solicitor," "attorney," "legal representative"

**AI Response**:
```
I understand you have legal representation. I'll note this and transfer you to a supervisor who can discuss next steps.

[If no human available immediately]

Please provide your lawyer's contact details, and we'll correspond directly with them.
```

**Action**:
- Log outcome: `LEGAL_REPRESENTATION`
- Attempt human transfer (if available)
- If not available, collect lawyer details and end call
- Flag for urgent manual review

---

### 5.2 Regulatory Complaint

**Trigger words**: "complaint," "ombudsman," "ASIC," "ACCC," "regulator"

**AI Response**:
```
I understand you'd like to make a complaint. I'll transfer you to a supervisor who can assist you.
```

**Action**:
- Log outcome: `COMPLAINT_THREATENED`
- Escalate to human immediately
- High priority for manual review

---

### 5.3 Abusive Language / Threats

**Trigger**: Debtor swears, threatens violence, uses abusive language

**AI Response**:
```
I'm sorry, but I'm unable to continue this call. If you'd like to discuss this matter calmly, please call us at [BCS_PHONE]. Goodbye.
```

**Action**:
- Log outcome: `ABUSIVE_DEBTOR`
- End call immediately
- Flag for manual review
- Consider cease contact

---

### 5.4 Deceased Debtor

**Debtor/third party says**: "They passed away" / "He/she is deceased"

**AI Response**:
```
I'm very sorry to hear that. I'll note this immediately and remove this number from our system. Can you please confirm the deceased's full name and date of passing?

We may need to contact the estate executor. If you have their contact details, please provide them, or you can email us at [ESTATES_EMAIL].
```

**Action**:
- Log outcome: `DECEASED`
- Escalate to human immediately
- Cease all contact to this number
- Flag for estate administration process

---

### 5.5 Bankruptcy / Insolvency

**Debtor says**: "I'm bankrupt" / "The company is in liquidation" / "Administrators appointed"

**AI Response**:
```
Thank you for informing me. I'll note this and transfer you to a supervisor. We'll need details of your bankruptcy trustee or administrator.
```

**Action**:
- Log outcome: `BANKRUPTCY_DECLARED`
- Escalate to human
- Legal requirement: Cease collection if bankruptcy confirmed

---

## 6. Outcome Taxonomy

### 6.1 Primary Outcomes

| Outcome Code | Definition | Next Action |
|--------------|------------|-------------|
| `PROMISE_TO_PAY` | Debtor commits to pay by specific date | Schedule follow-up call 1 day after promised date |
| `PAYMENT_PLAN_REQUESTED` | Debtor wants installment arrangement | Flag for manual review (human approves) |
| `PAYMENT_RECEIVED` | Debtor paid during/immediately after call | Update CRM, close account |
| `DISPUTE_RAISED` | Debtor disputes debt validity | Send dispute email, pause collection |
| `HARDSHIP_CLAIMED` | Debtor claims financial hardship | Send hardship form, pause collection |
| `NO_ANSWER` | Call went to voicemail | Leave message, schedule retry (2+ hours later) |
| `WRONG_NUMBER` | Not debtor's number | Flag for CRM update, cease calls to this number |
| `DECEASED` | Debtor reported deceased | Escalate, cease calls, contact estate |
| `CALLBACK_REQUESTED` | Debtor requests callback at specific time | Schedule call at requested time |
| `HUNG_UP` | Debtor hung up during call | Schedule retry next business day |
| `ESCALATE_TO_HUMAN` | Complex issue requiring human | Transfer or flag for urgent manual review |
| `CEASE_CONTACT_REQUESTED` | Debtor requested no more calls | Cease calls, written contact only |
| `LEGAL_REPRESENTATION` | Debtor has lawyer | Cease debtor contact, contact lawyer |
| `COMPLAINT_THREATENED` | Debtor threatens complaint | Escalate to human, high priority review |
| `ALREADY_PAID` | Debtor claims payment made | Request proof, flag for payment verification |
| `AMOUNT_DISPUTED` | Debtor disputes amount owed | Same as DISPUTE_RAISED |
| `ABUSIVE_DEBTOR` | Debtor was abusive/threatening | End call, flag for review |
| `BANKRUPTCY_DECLARED` | Debtor/company bankrupt | Cease collection, contact trustee |

---

## 7. Voicemail Scripts

### 7.1 Consumer Voicemail

```
Hello, this is [AGENT_NAME] from Brodie Collection Services calling for [DEBTOR_NAME]. This is regarding an important matter. Please return our call at [BCS_PHONE_NUMBER]. Again, that's [BCS_PHONE_NUMBER]. Thank you.
```

**Note**: Do NOT mention "debt" or specific amount (privacy protection).

---

### 7.2 Commercial Voicemail

```
Good morning/afternoon, this is [AGENT_NAME] from Brodie Collection Services calling for [CONTACT_NAME] at [COMPANY_NAME]. Please return our call regarding account [ACCOUNT_NUMBER] at [BCS_PHONE_NUMBER]. Thank you.
```

---

## 8. Payment Information & SMS Delivery

### 8.1 Payment Delivery Method: SMS (Preferred)

**Approach**: Instead of providing payment details verbally (prone to transcription errors), send payment instructions via SMS.

**Benefits**:
- ✅ Higher conversion (debtor can act immediately)
- ✅ No transcription errors
- ✅ Persistent reference (debtor can revisit SMS)
- ✅ Professional, modern approach

---

### 8.2 AI Bot Script (When Debtor Agrees to Pay)

**AI Bot**: "Great! How would you like to make payment today? I can send you details via text message for:
- Credit card payment (secure online link)
- Bank transfer
- Cheque mailing address"

**Debtor chooses payment method**

**AI Bot**: "Perfect. I'll send you the [payment method] details via SMS to this number ending in [last 4 digits] right now. You should receive it within the next minute. Is there anything else I can help you with?"

**Then**: n8n workflow sends SMS via Twilio

---

### 8.3 SMS Templates

#### Credit Card Payment SMS
```
Brodie Collection Services

Pay $1,500 now via credit card:
[STRIPE_PAYMENT_LINK]

Questions? Call [BCS_PHONE]
```

**Notes**:
- Requires Stripe Payment Link setup (Peter to provide or Vlad to set up)
- Link can be generic (manual amount entry) or dynamic (pre-filled amount per debtor)

---

#### Bank Transfer SMS
```
Brodie Collection Services
Bank Transfer Details:

BSB: XXX-XXX
Account: XXXXXXXXX
Account Name: Brodie Collection Services
Reference: [DEBTOR_ID]
Amount: $1,500

Questions? Call [BCS_PHONE]
```

**To be provided by Peter**:
- BSB number
- Account number
- Account name
- BCS phone number

---

#### Cheque Payment SMS
```
Brodie Collection Services
Mail cheque to:

[BCS_ADDRESS]
[SUBURB, STATE, POSTCODE]

Payable to: Brodie Collection Services
Amount: $1,500
Reference: [DEBTOR_ID]

Questions? Call [BCS_PHONE]
```

**To be provided by Peter**:
- BCS mailing address
- BCS phone number

---

### 8.4 Fallback: Verbal Payment Instructions (If SMS Fails)

**If debtor says "I don't have SMS"** or Twilio send fails:

**Bank transfer (verbal)**:
"You can transfer funds to BSB [XXX-XXX], account number [XXXXXXXXX], account name Brodie Collection Services. Please use reference [DEBTOR_ID] so we can identify your payment. Would you like me to repeat that?"

**Credit card (verbal)**:
"You can call our office at [BCS_PHONE] to process a credit card payment over the phone."

**Cheque (verbal)**:
"You can mail a cheque to [BCS_ADDRESS]. Please make it payable to Brodie Collection Services and include reference [DEBTOR_ID]."

---

## 9. Function Calls (Vapi)

### 9.1 log_outcome

**When to call**: At the end of every call

**Parameters**:
```json
{
  "outcome": "PROMISE_TO_PAY | DISPUTE_RAISED | NO_ANSWER | etc.",
  "next_action": "SCHEDULE_CALL | SEND_EMAIL | SEND_PAYMENT_SMS | MANUAL_REVIEW | null",
  "notes": "Optional string (e.g., 'Promised to pay by Oct 10')"
}
```

---

### 9.2 verify_identity

**When to call**: After debtor confirms they are the person named

**Parameters**:
```json
{
  "dob": "YYYY-MM-DD | null",
  "last_4_phone": "1234 | null",
  "address": "Street address | null"
}
```

**Response**:
```json
{
  "verified": true | false,
  "message": "Verification successful" | "Verification failed, please try again"
}
```

---

### 9.3 send_payment_sms

**When to call**: When debtor agrees to pay and chooses payment method

**Parameters**:
```json
{
  "payment_method": "credit_card | bank_transfer | cheque",
  "phone_number": "debtor's phone number (E.164 format)"
}
```

**Response**:
```json
{
  "sms_sent": true | false,
  "message": "SMS sent successfully" | "SMS failed, please provide details verbally"
}
```

**Implementation**: This function triggers n8n workflow to send Twilio SMS with appropriate template

---

## 10. Conversation Tone Guidelines

### 10.1 General Principles

- **Professional but empathetic**: Not robotic, not overly friendly
- **Concise**: Keep explanations brief (2-4 sentences max)
- **Clear**: Use simple language, avoid legal jargon
- **Respectful**: Maintain respectful tone even if debtor is upset
- **Non-judgmental**: No assumptions about debtor's financial situation

---

### 10.2 Voice Selection (Vapi)

**Recommended voice**: Azure "en-AU-NatashaNeural" (female Australian accent)

**Alternative**: Azure "en-AU-WilliamNeural" (male Australian accent)

**Why Australian accent**: Builds trust, sounds local, reduces "scam call" perception

---

## 11. Call Duration Targets

| Call Type | Target Duration |
|-----------|----------------|
| 1st call (no objections) | 2-3 minutes |
| 2nd call (follow-up) | 1.5-2.5 minutes |
| 3rd call (final warning) | 2-3 minutes |
| Objection handling (dispute, hardship) | 3-4 minutes |
| Escalation (transfer to human) | 1-2 minutes (before transfer) |

**If call exceeds 5 minutes**: May indicate complex situation, consider human escalation.

---

## 12. Testing Scenarios

See [DEVELOPMENT_PLAN.md Section 4](./DEVELOPMENT_PLAN.md#4-testing-strategy) for full synthetic test scenario list.

---

## 13. Updates and Refinements

**After receiving Peter's scripts**:
1. Replace all [PLACEHOLDER] text with actual details
2. Update Vapi system prompts with finalized scripts
3. Test all scripts with synthetic calls
4. Iterate based on Peter's feedback

**Version history**:
| Version | Date | Changes | Author |
|---------|------|---------|--------|
| 1.0 | 2025-10-03 | Initial framework, awaiting Peter's scripts | Vlad Petoukhov |

---

## 14. Related Documentation

- [COMPLIANCE.md](./COMPLIANCE.md) - Regulatory requirements that scripts must adhere to
- [TEMPLATES_ANALYSIS.md](./TEMPLATES_ANALYSIS.md) - Letter templates that inform call tone
- [TECHNICAL_ARCHITECTURE.md](./TECHNICAL_ARCHITECTURE.md) - Vapi configuration and system prompts

---

**End of Conversation Design Document**
