# Compliance Framework - Australian Debt Collection

## Document Information

**Project**: AI Debt Collection Bot - Phase 1
**Version**: 1.0
**Last Updated**: October 3, 2025
**Regulatory Scope**: Australia (ASIC, ACCC, Privacy Act 1988)

---

## 1. Regulatory Overview

### 1.1 Governing Bodies

| Organization | Jurisdiction | Relevance |
|--------------|--------------|-----------|
| **ASIC** (Australian Securities and Investments Commission) | Debt collection licensing | BCS must hold Australian Credit License |
| **ACCC** (Australian Competition and Consumer Commission) | Consumer protection | Prohibits misleading/deceptive conduct |
| **OAIC** (Office of the Australian Information Commissioner) | Privacy regulations | Privacy Act 1988 compliance |

---

## 2. Mandatory Disclosures (EVERY Call)

### 2.1 Required Statements

**These must be made on EVERY call, verbatim or substantially similar:**

1. **Identity Disclosure**:
   - "This is [Agent Name] calling from Brodie Collection Services."
   - Must include: Agent name + Company name

2. **Purpose Statement**:
   - "This call is regarding a debt collection matter."
   - Or: "I'm calling about an outstanding debt."

3. **Recording Notice** (if recording):
   - "This call may be recorded for quality and compliance purposes."

**Implementation**: Hard-code these into Vapi system prompts (cannot be skipped).

---

### 2.2 Debtor Verification Before Debt Discussion

**CRITICAL**: Cannot discuss debt details before confirming debtor identity.

**Acceptable verification methods**:
- Date of birth
- Last 4 digits of phone number
- Residential address
- Last 4 of account/invoice number

**If verification fails 2+ times**: Escalate to human.

---

## 3. Prohibited Conduct

### 3.1 Misleading or Deceptive Conduct (ACCC)

**AI bot must NOT**:
- Claim legal action will be taken if not authorized
- Imply debtor will be arrested (not possible for civil debt)
- Falsely claim to be a government agency or lawyer
- Misrepresent the amount owed
- Threaten to contact employer (unless legally permitted)

---

### 3.2 Harassment or Coercion (ASIC/ACCC)

**AI bot must NOT**:
- Use abusive, intimidating, or threatening language
- Call excessively (max 3 attempts per day, 2+ hours apart)
- Apply undue pressure or psychological coercion
- Contact debtor at unreasonable hours (see time restrictions)

---

### 3.3 Privacy Violations (Privacy Act 1988)

**AI bot must NOT**:
- Disclose debt details to third parties (family, roommates, colleagues)
- Leave voicemails with specific debt amounts or creditor names
- Send emails/SMS to unverified addresses

**If third party answers**:
- "May I please speak with [Debtor Name]?"
- If not available: "Please ask them to call us at [BCS Number]. This is a personal matter."
- Do NOT state it's a debt collection matter to third party.

---

## 4. Time Restrictions

### 4.1 Permitted Calling Hours

**Weekdays (Monday-Friday)**:
- 7:30 AM - 9:00 PM (debtor's local time)

**Weekends (Saturday-Sunday)**:
- 9:00 AM - 9:00 PM (debtor's local time)

**Public Holidays**:
- No calls (or same as weekends, confirm with ASIC guidelines)

**Time Zone**: AEST or ACST (confirm with Peter which states debtors are located in)

**Implementation**: n8n "Business Hours Check" code node (see TECHNICAL_ARCHITECTURE.md)

---

## 5. Frequency Limitations

**Maximum call attempts**:
- 3 per day
- Minimum 2 hours between attempts
- 9 total attempts per debt file (3 per week for 3 weeks)

**After 9 attempts with no contact**: Flag for manual review or alternative contact method (letter, email).

---

## 6. Debtor Rights

### 6.1 Right to Dispute

**If debtor says**: "I don't owe this" or "This debt is not mine"

**AI bot response**:
- "I understand you're disputing this debt. To formally dispute it, please submit your dispute in writing to [DISPUTES_EMAIL] within 7 days."
- Log outcome: DISPUTE_RAISED
- No further collection attempts until dispute resolved (manual review)

---

### 6.2 Right to Request Hardship Assistance

**If debtor says**: "I can't afford to pay" or "I'm experiencing financial hardship"

**AI bot response**:
- "I understand you're experiencing financial difficulty. I can send you a financial hardship form to complete. What email address should I send it to?"
- Log outcome: HARDSHIP_CLAIMED
- Send hardship form (Peter to provide template)
- Pause collection until hardship assessment complete (manual review)

---

### 6.3 Right to Cease Contact

**If debtor says**: "Stop calling me" or "Don't contact me again"

**AI bot response**:
- "I understand. I've noted your request. You may still receive written correspondence. If you wish to discuss this matter, please contact us at [BCS Number]."
- Log outcome: CEASE_CONTACT_REQUESTED
- Flag for manual review (legal team decides next steps)

---

## 7. Escalation Triggers (Transfer to Human Immediately)

**AI bot must escalate if debtor says**:
- "Lawyer" / "Solicitor" / "Legal representation"
- "Complaint" / "Ombudsman" / "Regulator"
- "Harassment" / "I'm recording this call"
- "Deceased" / "They passed away"
- "Bankruptcy" / "Insolvency"

**Also escalate for**:
- Abusive language or threats toward AI agent
- Multiple verification failures
- Complex payment negotiation beyond script capability
- Technical issues (call quality, system errors)

---

## 8. Call Recording & Audit Trail

### 8.1 Recording Requirements

**All calls must be recorded** for:
- Compliance review
- Dispute resolution
- Training and quality assurance

**Storage**:
- Vapi stores recordings automatically
- Accessible via Vapi dashboard or API
- Linked to call log in database (ai_call_logs.recording_url)

**Retention period**: 7 years (Australian financial record standard)

---

### 8.2 Transcript Requirements

**All calls must be transcribed**:
- Vapi provides automatic transcription
- Store in database (ai_call_logs.transcript)
- Searchable for compliance review

**Transcript should include**:
- Timestamp
- Full conversation
- Function calls made (log_outcome, verify_identity)
- Escalation events

---

## 9. Credit Reporting (Default Listings)

**CRITICAL**: AI bot CANNOT execute defaults.

**Default placement requirements** (manual human decision):
- Debt must be overdue 60+ days (consumer) or per contract (commercial)
- Debtor must be notified in writing 14+ days before default listed
- Notice must include right to dispute and hardship options
- Default can only be listed by licensed credit reporter

**AI bot role**:
- Flag account as "READY_FOR_DEFAULT" after 3rd unsuccessful call
- Human (Peter) reviews and decides whether to proceed

---

## 10. Consequences Statements (What AI Can Say)

### 10.1 Consumer Debt

**Allowed**:
- "Failure to pay may result in legal action and additional costs."
- "This debt may be reported to credit bureaus, affecting your credit rating."
- "Continued non-payment may lead to a payment default being listed on your credit file."

**Not allowed**:
- "You will be arrested if you don't pay." (FALSE)
- "We will garnish your wages." (Requires court order)
- "We will seize your assets." (Requires court order)

---

### 10.2 Commercial Debt

**Allowed**:
- "Failure to pay may result in a payment default being listed with ASIC, which will permanently affect your company's credit rating."
- "This matter may be referred to our client's solicitor for legal proceedings."
- "Directors may be personally liable for company debts in certain circumstances." (if legally accurate)

**Not allowed**:
- "We will shut down your business." (Requires court order)
- "We will report you to the tax office." (Not debt collector's role)

---

## 11. Compliance Checklist (Every Call)

| Requirement | Checked? |
|-------------|----------|
| Identity disclosure made | ☐ |
| Purpose statement made | ☐ |
| Recording notice made (if recording) | ☐ |
| Debtor identity verified before debt discussed | ☐ |
| Call made within permitted hours (7:30am-9pm weekdays, 9am-9pm weekends) | ☐ |
| No misleading statements made | ☐ |
| No harassment or coercion | ☐ |
| Third-party privacy protected (if applicable) | ☐ |
| Escalation triggers handled correctly | ☐ |
| Call recorded and transcript stored | ☐ |
| Outcome logged in CRM | ☐ |

---

## 12. Compliance Review Process

### 12.1 Initial Review (First 50 Calls)

**Who**: Vlad (developer) + Peter (client)

**Process**:
1. Listen to all call recordings
2. Review transcripts for mandatory disclosures
3. Check for any prohibited statements
4. Verify escalation triggers worked
5. Confirm outcomes logged correctly

**Target**: 100% compliance (zero violations)

---

### 12.2 Ongoing Monitoring

**Frequency**: Weekly spot checks (10% of calls)

**Who**: Peter or designated BCS staff

**Red flags to watch for**:
- Missing mandatory disclosures
- Threatening or harassing language
- Privacy violations
- Calls outside permitted hours
- Escalation triggers not handled

**Action if violation found**:
- Immediate manual review of all recent calls
- Adjust Vapi system prompts
- Retrain AI if needed
- Notify affected debtors if required

---

## 13. Debtor Complaint Handling

**If debtor complains** (to BCS, ASIC, or Ombudsman):

1. **Immediate action**:
   - Pause all AI calls to that debtor
   - Flag account for manual review
   - Retrieve call recording and transcript

2. **Investigation**:
   - Review compliance with mandatory disclosures
   - Check for prohibited conduct
   - Verify call times and frequency

3. **Remediation**:
   - Apologize if violation occurred
   - Correct the record (if debt amount disputed)
   - Remove default listing if improperly placed

4. **Preventive action**:
   - Update Vapi prompts to prevent recurrence
   - Document in DECISIONS_LOG.md

---

## 14. License and Registration

**BCS Requirements**:
- Australian Credit License (held by Peter/BCS)
- Membership in industry association (if applicable)
- Compliance with ASIC regulatory guides (e.g., RG 96)

**AI bot as "agent" of BCS**:
- BCS remains responsible for AI bot's conduct
- AI bot acts under BCS's license
- BCS liable for compliance violations

---

## 15. References & Resources

**ASIC Regulatory Guides**:
- RG 96: Debt collection guideline for collectors and creditors
- RG 255: Providing digital financial product advice

**ACCC Resources**:
- Debt collection and the Australian Consumer Law

**Privacy Act 1988**:
- Australian Privacy Principles (APPs)

**Industry Codes**:
- Debt Collection Guideline (if BCS is member of industry body)

---

## 16. Updates and Changes

**Monitoring**:
- Check ASIC/ACCC websites quarterly for regulatory changes
- Subscribe to compliance newsletters

**When regulations change**:
- Update Vapi system prompts immediately
- Document in DECISIONS_LOG.md
- Notify Peter of any impacts

---

## 17. Related Documentation

- [CONVERSATION_DESIGN.md](./CONVERSATION_DESIGN.md) - Call flows and scripts (must align with compliance)
- [REQUIREMENTS.md](./REQUIREMENTS.md) - Functional requirements (REQ-012 to REQ-016)
- [TECHNICAL_ARCHITECTURE.md](./TECHNICAL_ARCHITECTURE.md) - Implementation of compliance safeguards

---

**End of Compliance Framework Document**
