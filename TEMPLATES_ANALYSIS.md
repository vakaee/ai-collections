# Letter Templates Analysis

## Document Information

**Project**: AI Debt Collection Bot - Phase 1
**Version**: 1.0
**Last Updated**: October 3, 2025
**Templates Analyzed**: 2 Letter of Demand templates (Consumer + Commercial)

---

## 1. Overview

This document analyzes the two Letter of Demand templates provided by Peter to extract tone, structure, language patterns, and compliance elements. These insights inform the AI voice bot's conversation scripts.

---

## 2. Template 1: Letter of Demand - Consumer Debt

**Source**: `Letter_of_Demand.pdf` (provided Oct 3, 2025)

### 2.1 Structure

1. **Header**:
   - Brodie Collection Services branding
   - Contact details (phone, email, address)
   - Date
   - Debtor's name and address
   - Account reference number

2. **Subject Line**:
   - "Letter of Demand - [Creditor Name]"

3. **Opening**:
   - Direct address: "Dear [Debtor Name]"

4. **Debt Statement**:
   - Creditor name
   - Amount owing
   - Invoice/account number(s)
   - Date(s) of debt origin

5. **Formal Demand**:
   - "We hereby demand payment in full within 7 days..."

6. **Payment Methods**:
   - Bank transfer (BSB, account, reference)
   - Credit card (phone number)
   - Cheque (mailing address)

7. **Consequences Section**:
   - Legal action
   - Additional costs (legal fees, interest)
   - Credit rating impact (5 years)

8. **Dispute Rights**:
   - 7-day window to dispute in writing
   - Email address for disputes

9. **Financial Hardship**:
   - Acknowledgment of hardship assistance available
   - Contact details for hardship inquiries

10. **Closing**:
    - "Yours faithfully"
    - BCS representative signature
    - Debt collection license number
    - Privacy Act compliance statement

---

### 2.2 Tone Analysis

**Characteristics**:
- **Formal but fair**: Uses legal language ("we hereby demand") but balanced with options (dispute, hardship)
- **Direct**: No euphemisms, clearly states "debt collection"
- **Consequence-focused**: Emphasizes negative outcomes (legal action, credit impact) to motivate payment
- **Compliant**: Includes all required disclosures (dispute rights, hardship, license number)

**Key Phrases**:
- "We hereby demand" (assertive)
- "To avoid further action" (urgency)
- "May result in legal proceedings" (consequence)
- "Affecting your ability to obtain credit for up to 5 years" (specific impact)
- "We understand financial circumstances can change" (empathy)

---

### 2.3 Compliance Elements

| Element | Included? | Notes |
|---------|-----------|-------|
| Debt collection license number | ✅ | Footer |
| Creditor name | ✅ | In debt statement |
| Amount owed | ✅ | Specific dollar amount |
| Dispute rights | ✅ | 7-day window, written submission required |
| Financial hardship option | ✅ | Mentioned with contact details |
| Privacy Act statement | ✅ | Footer ("Your privacy is important...") |
| Payment deadline | ✅ | 7 days from letter date |

---

### 2.4 Implications for Voice Bot

**Conversation script should**:
- Maintain formal but fair tone
- State debt amount, creditor, and date clearly
- Offer 7-day payment deadline
- Provide payment methods verbally
- Mention consequences (legal action, credit rating) on 2nd/3rd calls
- Include dispute and hardship options when debtor objects

**Avoid**:
- Overly aggressive language
- Empty threats
- Ignoring debtor's financial situation

---

## 3. Template 2: Letter of Demand - Commercial Debt

**Source**: `Letter_of_Demand-1.pdf` (provided Oct 3, 2025)

### 3.1 Structure

Same basic structure as consumer template, with these modifications:

1. **Subject Line**:
   - "Letter of Demand - Commercial Debt"

2. **Opening**:
   - Addresses company: "Dear [Company Name] / [Director/Contact Name]"

3. **Debt Statement**:
   - References "invoice" instead of "account"
   - May include ABN/ACN

4. **Formal Demand**:
   - Same 7-day demand

5. **Payment Methods**:
   - Same as consumer (bank transfer, credit card, cheque)

6. **Consequences Section** (KEY DIFFERENCES):
   - **ASIC payment default**: "will permanently affect your company's credit rating and ability to secure finance"
   - **Legal proceedings**: "Our client's solicitor has been briefed and is prepared to commence proceedings"
   - **Director liability**: "Directors may be personally liable for company debts under certain circumstances"
   - **Asset seizure**: "Judgment enforcement may include sheriff seizure of company assets"

7. **Dispute Rights**:
   - Same 7-day window
   - Same written submission requirement

8. **Financial Hardship**:
   - **NOT INCLUDED** (commercial debtors don't have consumer protections)

9. **Closing**:
   - Same as consumer template

---

### 3.2 Tone Analysis

**Characteristics**:
- **More formal and assertive**: Uses stronger legal language
- **Consequence-heavy**: Emphasizes ASIC record, director liability, asset seizure
- **Business-to-business**: Assumes debtor understands commercial consequences
- **Legal backing**: References "our client's solicitor" to imply readiness for litigation

**Key Phrases**:
- "Our client's solicitor has been briefed" (escalation threat)
- "Payment default will be listed with ASIC" (regulatory consequence)
- "Permanently affect your company's credit rating" (long-term impact)
- "Directors may be personally liable" (personal risk for company officers)
- "Sheriff seizure of company assets" (enforcement action)

---

### 3.3 Compliance Elements

| Element | Included? | Notes |
|---------|-----------|-------|
| Debt collection license number | ✅ | Footer |
| Creditor name | ✅ | In debt statement |
| Amount owed | ✅ | Specific dollar amount |
| Dispute rights | ✅ | 7-day window, written submission |
| Financial hardship option | ❌ | Not applicable to commercial debtors |
| Privacy Act statement | ✅ | Footer |
| Payment deadline | ✅ | 7 days from letter date |
| ASIC default warning | ✅ | Specific to commercial debt |

---

### 3.4 Implications for Voice Bot

**Commercial script should**:
- Use more formal language ("your company," "the account")
- Emphasize ASIC payment default as primary consequence
- Mention director liability (if legally accurate)
- Reference "our client's solicitor" to establish legal backing
- **Omit financial hardship** option

**Differences from consumer script**:
- No "we understand financial circumstances can change"
- Stronger emphasis on legal action
- Focus on business reputation and credit rating impact

---

## 4. Comparative Analysis

| Element | Consumer Template | Commercial Template |
|---------|-------------------|---------------------|
| **Tone** | Formal but empathetic | Formal and assertive |
| **Primary Consequence** | Credit rating impact | ASIC payment default |
| **Legal Threat** | "May result in legal action" | "Solicitor has been briefed" |
| **Hardship Option** | ✅ Included | ❌ Not included |
| **Target** | Individual debtor | Company/directors |
| **Personal Liability** | Not mentioned | ✅ Director liability mentioned |
| **Enforcement** | General ("legal costs") | Specific ("sheriff seizure") |

---

## 5. Payment Information Extracted

### 5.1 Payment Methods (Both Templates)

**Bank Transfer**:
- BSB: [AWAITING DETAILS FROM PETER]
- Account Number: [AWAITING DETAILS FROM PETER]
- Account Name: Likely "Brodie Collection Services" or "[Creditor Name] c/o BCS"
- Reference: [AWAITING FORMAT - likely debtor ID or invoice number]

**Credit Card**:
- Phone number: [AWAITING BCS PHONE NUMBER]
- Processing: Likely manual (call BCS to process)

**Cheque**:
- Payable to: "Brodie Collection Services"
- Mailing address: [AWAITING BCS ADDRESS]

**Note**: Peter has not yet provided exact payment details. These will be added to conversation scripts once received.

---

## 6. Common Language Patterns

### 6.1 Phrases to Use in Voice Scripts

**Debt statement**:
- "Our records show an outstanding debt of $[AMOUNT] owed to [CREDITOR]"
- "This account from [DATE] remains unpaid"

**Payment request**:
- "We kindly request payment in full within 7 days"
- "To avoid further action, we urge you to make payment by [DATE]"

**Consequences (consumer)**:
- "Continued non-payment may result in legal action and additional costs"
- "A payment default may be listed on your credit file, affecting your ability to obtain credit for up to 5 years"

**Consequences (commercial)**:
- "Failure to pay will result in a payment default being listed with ASIC"
- "This will permanently affect your company's credit rating and ability to secure finance"
- "Our client's solicitor is prepared to commence legal proceedings"

**Dispute handling**:
- "If you dispute this debt, please submit your dispute in writing to [EMAIL] within 7 days"

**Hardship (consumer only)**:
- "If you're experiencing financial difficulty, we can provide a hardship form"

---

## 7. Compliance Alignment

Both templates demonstrate strong compliance with Australian debt collection regulations:

✅ Clear identification of collector (BCS)
✅ License number disclosed
✅ Debt details specified (amount, creditor, date)
✅ Payment methods provided
✅ Dispute rights clearly stated
✅ Hardship option mentioned (consumer)
✅ Privacy Act compliance statement
✅ No misleading or deceptive language
✅ No harassment or undue pressure

**Voice bot must maintain this same level of compliance.**

---

## 8. Recommended Script Adjustments

Based on template analysis, voice scripts should:

1. **Opening**: Mirror the formal but clear tone
   - "This is [Agent] from Brodie Collection Services"
   - Not: "Hey, it's [Agent] calling about a payment"

2. **Debt Statement**: Be as specific as the letters
   - Include creditor name, amount, date
   - Not: "You owe money to one of our clients"

3. **Consequences**: Escalate across 3 calls (same as letters)
   - 1st call: Mention consequences briefly
   - 2nd call: Emphasize consequences (credit rating / ASIC)
   - 3rd call: Final warning with specific actions (solicitor referral)

4. **Options**: Always provide dispute and hardship options when appropriate
   - Consumer: Both dispute and hardship
   - Commercial: Dispute only

5. **Closing**: Professional and respectful
   - "Thank you for your time. If you have questions, call [BCS NUMBER]."
   - Not: "You need to pay or else we'll take action."

---

## 9. Gaps / Questions for Peter

1. **Payment details**: Need exact BSB, account number, reference format
2. **BCS phone number**: For payment and callback inquiries
3. **BCS address**: For cheque payments
4. **Disputes email**: Specific email address for dispute submissions
5. **Hardship email/phone**: Contact for hardship inquiries
6. **License number**: For verbal disclosure (if required by regulations)
7. **Voice vs. letter differences**: Should voice scripts be softer than written letters?

---

## 10. Related Documentation

- [CONVERSATION_DESIGN.md](./CONVERSATION_DESIGN.md) - Call scripts informed by these templates
- [COMPLIANCE.md](./COMPLIANCE.md) - Regulatory requirements
- [PENDING_ITEMS.md](./PENDING_ITEMS.md) - Information still needed from Peter

---

**End of Templates Analysis Document**
