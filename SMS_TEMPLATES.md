# SMS Payment Templates

**⚠️ PHASE 2 REFERENCE ONLY**

This document contains SMS templates for **Phase 2** implementation ($2,000-[PRICE REDACTED]).

**Phase 1 scope** (current): Voice-only. Payment instructions are provided **verbally** during calls (see VAPI_ASSISTANT_CONFIGS.md for verbal payment instructions).

When Phase 2 is commissioned, these templates will be implemented in n8n SMS workflows.

---

## Overview

These SMS templates are sent to debtors after they agree to make payment. The AI bot asks for their preferred payment method (credit card, bank transfer, or cheque), then Workflow 3 (BCS Send Payment SMS) sends the appropriate template.

**Variables** (automatically filled by n8n):
- `{{amount}}` - Debt amount
- `{{invoice_number}}` - Invoice/account reference
- `{{phone}}` - BCS phone number for questions

---

## Template 1: Credit Card Payment

**Trigger**: Debtor says "I'd like to pay by credit card" or similar

**SMS Body**:
```
[COLLECTION AGENCY]

Pay ${{amount}} now via credit card:
{{STRIPE_PAYMENT_LINK}}

Questions? Call {{phone}}
```

**Example** (with actual values):
```
[COLLECTION AGENCY]

Pay $1,500.00 now via credit card:
https://buy.stripe.com/test_7sI00g0Ub3Fg8Te000

Questions? Call +61412345678
```

**Notes**:
- Link expires in 24 hours (configure in Stripe)
- Payment confirmation sent to [CLIENT]'s email automatically
- Debtor receives receipt via email from Stripe

---

## Template 2: Bank Transfer Payment

**Trigger**: Debtor says "I'd like to pay by bank transfer" or similar

**SMS Body**:
```
[COLLECTION AGENCY]

Bank Transfer Details:
BSB: {{BSB_NUMBER}}
Account: {{ACCOUNT_NUMBER}}
Name: {{ACCOUNT_NAME}}
Amount: ${{amount}}
Reference: {{invoice_number}}

Questions? Call {{phone}}
```

**Example** (with actual values):
```
[COLLECTION AGENCY]

Bank Transfer Details:
BSB: 123-456
Account: 123456789
Name: [COLLECTION AGENCY] Pty Ltd
Amount: $1,500.00
Reference: INV-2024-001

Questions? Call +61412345678
```

**Notes**:
- Reference number critical (helps [CLIENT] match payment to debtor)
- Payment typically takes 1-2 business days
- Debtor should email proof of payment to {{BCS_EMAIL}}

---

## Template 3: Cheque Payment

**Trigger**: Debtor says "I'd like to pay by cheque" or similar

**SMS Body**:
```
[COLLECTION AGENCY]

Mail cheque (${{amount}}) to:
{{BCS_MAILING_ADDRESS}}

Reference: {{invoice_number}}

Questions? Call {{phone}}
```

**Example** (with actual values):
```
[COLLECTION AGENCY]

Mail cheque ($1,500.00) to:
[COLLECTION AGENCY]
123 Main Street
Sydney NSW 2000

Reference: INV-2024-001

Questions? Call +61412345678
```

**Notes**:
- Cheque should be made payable to "[COLLECTION AGENCY] Pty Ltd"
- Include reference number on cheque memo line
- Payment takes 5-7 business days to process

---

## Template 4: Generic Payment (Fallback)

**Trigger**: Payment method unclear or not specified

**SMS Body**:
```
[COLLECTION AGENCY]

Payment arrangements for ${{amount}}:
Call {{phone}}

Reference: {{invoice_number}}
```

**Example** (with actual values):
```
[COLLECTION AGENCY]

Payment arrangements for $1,500.00:
Call +61412345678

Reference: INV-2024-001
```

**Notes**:
- Used when AI couldn't determine payment method
- [CLIENT] handles manually

---

## Configuration in n8n

### Workflow 3: BCS Send Payment SMS

**Code Node "Build SMS Body"**:

```javascript
const debtor = $node['Google Sheets - Lookup Debtor'].json;
const paymentMethod = $input.item.json.payment_method;

const name = debtor.name;
const amount = parseFloat(debtor.amount).toFixed(2);
const phone = $vars.BCS_PHONE_NUMBER;
const invoiceNumber = debtor.invoice_number;

let smsBody = '';

switch(paymentMethod) {
  case 'credit_card':
    smsBody = `[COLLECTION AGENCY]

Pay $${amount} now via credit card:
${$vars.STRIPE_PAYMENT_LINK}

Questions? Call ${phone}`;
    break;

  case 'bank_transfer':
    smsBody = `[COLLECTION AGENCY]

Bank Transfer Details:
BSB: ${$vars.BSB_NUMBER}
Account: ${$vars.ACCOUNT_NUMBER}
Name: ${$vars.ACCOUNT_NAME}
Amount: $${amount}
Reference: ${invoiceNumber}

Questions? Call ${phone}`;
    break;

  case 'cheque':
    smsBody = `[COLLECTION AGENCY]

Mail cheque ($${amount}) to:
${$vars.BCS_MAILING_ADDRESS}

Reference: ${invoiceNumber}

Questions? Call ${phone}`;
    break;

  default:
    smsBody = `[COLLECTION AGENCY]

Payment arrangements for $${amount}:
Call ${phone}

Reference: ${invoiceNumber}`;
}

return {
  to: debtor.phone,
  body: smsBody,
  debtor_id: debtor.debtor_id,
  row_number: debtor.__rowNum__
};
```

---

## Testing SMS Templates

### Test Credit Card SMS
1. Use Workflow 5 (test webhook) with outcome = "READY_TO_PAY" and payment_method = "credit_card"
2. Check your phone for SMS
3. Click link and verify Stripe page loads

### Test Bank Transfer SMS
1. Use Workflow 5 with payment_method = "bank_transfer"
2. Verify all bank details visible and correct
3. Verify reference number matches invoice_number

### Test Cheque SMS
1. Use Workflow 5 with payment_method = "cheque"
2. Verify mailing address correct
3. Verify reference number included

---

## Character Count (SMS Pricing)

**Standard SMS**: Up to 160 characters = 1 SMS segment = $0.08
**Long SMS**: 161-306 characters = 2 segments = $0.16

**Current template lengths**:
- Credit card: ~120 characters (1 segment)
- Bank transfer: ~180 characters (2 segments)
- Cheque: ~140 characters (1 segment)

**Cost per payment SMS**:
- Credit card: $0.08
- Bank transfer: $0.16
- Cheque: $0.08
- Average: ~$0.11 per payment SMS

---

## Customization

To customize templates:
1. Open n8n → Workflow 3 (BCS Send Payment SMS)
2. Find Code node "Build SMS Body"
3. Edit smsBody strings
4. Save workflow
5. Test with Workflow 5

**Tips**:
- Keep under 160 characters to save cost (1 segment)
- Include BCS name for legitimacy
- Always include reference number
- Always include contact phone number

---

## Compliance Notes

**Australian SMS Regulations**:
- Must identify sender ([COLLECTION AGENCY])
- Must provide opt-out method (reply STOP - handled by Twilio)
- Must be sent during reasonable hours (7:30am-9pm)
- Must be relevant to debt collection activity

**Current templates**: ✅ Compliant

---

## Future Enhancements (Phase 2)

- **Payment reminders**: Automated follow-up SMS if payment not received
- **Payment confirmation**: SMS confirming payment received
- **Two-way SMS**: Debtor can reply with questions, routed to [CLIENT]
- **Multi-language**: Templates in languages other than English

---

**End of SMS Templates**

**Status**: Ready for use
**Last Updated**: October 3, 2025
