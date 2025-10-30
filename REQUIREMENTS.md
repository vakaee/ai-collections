# Requirements Specification

## Document Information

**Project**: AI Debt Collection Bot - Phase 1
**Version**: 1.0
**Last Updated**: October 3, 2025
**Status**: Draft (pending [CLIENT]'s CRM access and scripts)

---

## 1. Functional Requirements

### 1.1 Debt File Management

**REQ-001**: System shall accept debt file allocation from CRM
- Input: Debtor ID, account details, debt amount, creditor name
- Debtor type: Consumer or Commercial
- Priority level: Normal, High, Urgent
- Contact details: Phone number(s), email, address

**REQ-002**: System shall retrieve debtor information from CRM database
- Read debtor profile data
- Read debt history (prior calls, letters sent, payment history)
- Read strategy assignment (which stage of recovery)

**REQ-003**: System shall support manual allocation of specific accounts to AI bot
- BCS staff can assign individual accounts
- BCS staff can assign batch of accounts
- BCS staff can remove accounts from queue

---

### 1.2 Outbound Call Execution

**REQ-004**: System shall initiate outbound calls automatically
- Schedule calls based on priority and retry logic
- Respect time restrictions (7:30am-9pm weekdays, 9am-9pm weekends AEST/ACST)
- Retry logic: 3 attempts per day maximum, spaced 2+ hours apart

**REQ-005**: System shall handle call progression (no answer, busy, voicemail)
- Detect answering machine vs. human
- Leave voicemail message if no answer
- Mark call as "NO_ANSWER" and schedule retry
- Detect busy signal and retry later

**REQ-006**: System shall conduct conversation per scripted flow
- Consumer script for individual debtors
- Commercial script for business debtors
- Adaptive responses based on debtor objections

**REQ-007**: System shall handle debtor identity verification
- Request date of birth, address, or last 4 digits of phone number
- Proceed with debt discussion only after verification
- Escalate to human if verification fails 2+ times

---

### 1.3 Conversation Management

**REQ-008**: System shall execute 3-tier call progression strategy
- **1st call**: Introduction, debt notification, payment request
- **2nd call**: Follow-up, consequences reminder, escalation warning
- **3rd call**: Final demand, explicit consequences, escalation notice

**REQ-009**: System shall handle standard objections:
- **Dispute**: Request written submission to registered email
- **Financial hardship**: Offer to send hardship proof form
- **Payment negotiation**: Capture promise to pay date or payment plan request
- **Already paid**: Request proof of payment, log for manual review
- **Wrong person**: Verify identity, update CRM if incorrect contact

**REQ-010**: System shall provide payment information VERBALLY during call (**PHASE 1 PRIMARY METHOD**)
- **Phase 1 Scope**: ALL payment details delivered verbally (no SMS)
- AI asks debtor: "Do you have a pen and paper handy?"
- Bank transfer details: AI reads BSB, account number, account name, reference slowly and clearly
- Credit card payment: AI provides phone number for payments, office hours
- Cheque payment: AI reads mailing address and reference number
- AI repeats all payment details once
- AI confirms debtor has written down details before ending call
- Follow-up call scheduled in 3 days to confirm payment received
- **Note**: SMS delivery of payment details is Phase 2 feature (deferred to $2,000-3,000 contract, not yet commissioned)

**REQ-011**: System shall escalate to human in these scenarios:
- Debtor uses trigger words: "lawyer," "complaint," "ombudsman," "sue"
- Debtor becomes abusive or threatens violence
- Technical issues (call quality, system errors)
- Complex negotiation beyond script capabilities
- Deceased debtor notification

---

### 1.4 Compliance & Mandatory Disclosures

**REQ-012**: System shall make mandatory disclosures on EVERY call:
1. Identity disclosure: "This is [Agent Name] calling from [COLLECTION AGENCY]"
2. Purpose statement: "This call is regarding a debt collection matter"
3. Recording notice: "This call may be recorded for quality and compliance purposes"

**REQ-013**: System shall NOT discuss debt details before identity verification

**REQ-014**: System shall NOT call outside permitted hours:
- Weekdays: 7:30am - 9:00pm
- Weekends: 9:00am - 9:00pm
- Timezone: AEST or ACST (confirm with Peter)

**REQ-015**: System shall NOT:
- Make misleading or deceptive statements
- Harass or coerce debtor
- Disclose debt to third parties
- Threaten action not legally permitted (e.g., arrest, wage garnishment)

**REQ-016**: System shall record all calls for compliance audit

---

### 1.5 CRM Integration & Logging

**REQ-017**: System shall log every call with structured outcome:

**Primary outcomes**:
- PAYMENT_RECEIVED
- PAYMENT_PLAN_REQUESTED
- PROMISE_TO_PAY (with date)
- DISPUTE_RAISED
- HARDSHIP_CLAIMED
- NO_ANSWER (voicemail left)
- WRONG_NUMBER
- DECEASED
- CALLBACK_REQUESTED
- HUNG_UP
- ESCALATE_TO_HUMAN

**REQ-018**: System shall update CRM with:
- Call timestamp (date/time)
- Call duration
- Outcome category
- Next action required
- Next scheduled call (if applicable)
- Transcript or summary

**REQ-019**: System shall store call recordings:
- Audio file linked to call log
- Accessible for compliance review
- Retention period: TBD (Australian regulations)

**REQ-020**: System shall flag accounts for manual review when:
- 3rd call completed with no resolution
- Dispute raised
- Hardship claimed
- Payment deadline passed with no contact
- Any escalation trigger occurs

---

### 1.6 Follow-up Actions

**REQ-021**: System shall schedule next call based on outcome:
- Promise to pay → Schedule follow-up 1 day after promised date
- Ready to pay → Schedule follow-up 3 days later to confirm payment received
- No answer → Schedule retry same day (4 hours later) or next business day
- Callback requested → Schedule at requested time
- Hung up → Schedule retry next business day

**REQ-022**: System shall flag accounts for manual review when:
- Dispute raised (AI provides email address verbally for written submission)
- Hardship claimed (AI directs debtor to call back or email for hardship form)
- Payment arrangement needed
- 3rd call completed with no resolution

**REQ-023**: System shall NOT automatically proceed to default
- Default placement remains manual decision by BCS staff
- System flags account as "READY_FOR_DEFAULT" after 3rd unsuccessful call
- Human must review and execute default

**Note**: Automated email/SMS follow-ups (e.g., hardship forms, dispute confirmations) are deferred to Phase 2

---

## 2. Non-Functional Requirements

### 2.1 Performance

**NFR-001**: System shall initiate calls within 5 minutes of scheduled time

**NFR-002**: System shall handle concurrent calls (scalability TBD based on Vapi limits)

**NFR-003**: Call latency shall be < 500ms (natural conversation pacing)

**NFR-004**: System uptime shall be 99%+ during business hours

---

### 2.2 Security & Privacy

**NFR-005**: System shall encrypt all PII (debtor names, addresses, phone numbers) at rest

**NFR-006**: System shall transmit call audio over encrypted connections (TLS/SRTP)

**NFR-007**: Database access shall require authentication (credentials not hard-coded)

**NFR-008**: Call recordings shall be stored securely with access restricted to authorized BCS staff

**NFR-009**: System shall comply with Australian Privacy Act 1988

---

### 2.3 Compliance

**NFR-010**: System shall adhere to ASIC guidelines for debt collection

**NFR-011**: System shall adhere to ACCC consumer protection standards

**NFR-012**: System shall maintain audit trail of all debtor interactions (6+ months retention minimum)

**NFR-013**: System shall support compliance review (searchable transcripts, flagged violations)

---

### 2.4 Usability

**NFR-014**: BCS staff shall be able to allocate debt files without technical training

**NFR-015**: Call outcomes shall be visible in CRM within 1 minute of call completion

**NFR-016**: System shall provide dashboard showing:
- Calls made today/this week
- Outcome distribution
- Accounts pending manual review
- Error/escalation rate

---

### 2.5 Maintainability

**NFR-017**: Conversation scripts shall be editable without code changes (configuration-based)

**NFR-018**: System shall log errors with sufficient detail for debugging

**NFR-019**: Code shall be documented with inline comments and README

**NFR-020**: System shall support easy telephony provider migration (Vapi → Twilio if needed)

---

### 2.6 Scalability

**NFR-021**: System shall support 50+ concurrent calls (Vapi plan dependent)

**NFR-022**: Database shall handle 10,000+ debtor records

**NFR-023**: System shall support multi-timezone calling (future: expand beyond Australia)

---

## 3. Assumptions

**ASS-001**: CRM database is accessible via direct connection or API

**ASS-002**: [CLIENT] will provide all necessary scripts, templates, and compliance guidance

**ASS-003**: Payment processing exists externally (bot redirects, does not process payments)

**ASS-004**: Human agents are available for escalations during business hours

**ASS-005**: Debtor phone numbers in CRM are accurate and current

**ASS-006**: Australian debt collection regulations are stable (no pending changes)

**ASS-007**: Vapi.ai service is reliable and meets SLA requirements

**ASS-008**: OpenAI API is reliable and meets latency requirements

---

## 4. Constraints

**CON-001**: Budget is fixed at [PRICE REDACTED] (+[BONUS REDACTED])

**CON-002**: Deadline is October 31, 2025 (28 days remaining)

**CON-003**: [CLIENT] has limited availability for meetings/reviews

**CON-004**: Current CRM lacks attachment/template support (workaround required)

**CON-005**: No existing telephony infrastructure (greenfield deployment)

**CON-006**: AI cannot execute defaults (human decision required)

**CON-007**: Phase 1 is voice-only per original agreement ([PRICE REDACTED] + [BONUS REDACTED]). SMS/email automation deferred to Phase 2 ($2,000-[PRICE REDACTED])

---

## 5. User Stories

### 5.1 BCS Staff (Peter)

**US-001**: As BCS staff, I want to allocate a debt file to the AI bot so that it automatically attempts collection

**US-002**: As BCS staff, I want to see call outcomes in the CRM so I know what happened with each debtor

**US-003**: As BCS staff, I want accounts flagged for manual review when disputes or hardships are claimed

**US-004**: As BCS staff, I want to listen to call recordings so I can verify compliance and quality

**US-005**: As BCS staff, I want to manually escalate an account to default after reviewing the AI's call history

---

### 5.2 Debtor

**US-006**: As a debtor, I want to verify my identity before discussing my debt so my privacy is protected

**US-007**: As a debtor, I want to dispute the debt and receive instructions on how to submit my dispute in writing

**US-008**: As a debtor, I want to request a hardship form if I'm experiencing financial difficulty

**US-009**: As a debtor, I want to make a payment arrangement and have the bot record my promise to pay

**US-010**: As a debtor, I want to request a callback at a more convenient time

---

### 5.3 Developer (Vlad)

**US-011**: As the developer, I want modular code so I can easily update scripts without rewriting the system

**US-012**: As the developer, I want comprehensive logging so I can debug issues quickly

**US-013**: As the developer, I want mock data for testing so I don't need production CRM access initially

---

## 6. Out of Scope (Phase 1)

**Note**: Phase 1 is voice-only outbound calling per original agreement ([PRICE REDACTED] + [BONUS REDACTED]).

**OOS-001**: Inbound call handling (not in Phase 1 agreement)

**OOS-002**: SMS automation (Phase 2: $2,000-[PRICE REDACTED])

**OOS-003**: Email automation (Phase 2: $2,000-[PRICE REDACTED])

**OOS-004**: Management dashboard/analytics (Phase 2)

**OOS-005**: CRM rebuild (Phase 3 if needed: $4,000-$6,000)

**OOS-006**: Payment processing integration (payment instructions provided verbally)

**OOS-007**: Multi-language support

**OOS-008**: Integration with legal software (for default placement)

---

## 7. Acceptance Criteria (Phase 1 Completion)

System is considered complete when:

1. ✅ Debt file can be allocated to the platform (manually or via CRM integration)
2. ✅ Calls are made automatically at scheduled times
3. ✅ Each call is logged in CRM with structured outcome
4. ✅ All mandatory compliance disclosures are made on every call
5. ✅ Identity verification works correctly
6. ✅ Consumer vs. commercial scripts route correctly
7. ✅ Objection handling works (dispute, hardship, payment negotiation)
8. ✅ Human escalation triggers function properly
9. ✅ Call recordings are stored and accessible
10. ✅ [CLIENT] can review call outcomes and transcripts

**Bonus criteria** (for [BONUS REDACTED]):
- Zero compliance violations in first 50 calls
- 80%+ successful contact rate (excluding wrong numbers)
- Positive feedback from [CLIENT] on call quality

---

## 8. Dependencies

**DEP-001**: CRM database access (blocking development)

**DEP-002**: Call scripts from [CLIENT] (blocking conversation design)

**DEP-003**: SMS templates from [CLIENT] (for follow-up actions)

**DEP-004**: Registered email address for disputes

**DEP-005**: Hardship form template

**DEP-006**: Phone number for outbound calls (Vapi-provided or BCS existing)

**DEP-007**: Vapi account setup (Vlad's responsibility)

**DEP-008**: OpenAI API key (Vlad's responsibility)

---

## 9. Change Log

| Date | Version | Changes | Author |
|------|---------|---------|--------|
| 2025-10-03 | 1.0 | Initial requirements document | [DEVELOPER NAME] |

---

## 10. Approval

**Pending approval from**: [CLIENT NAME]

**Review date**: TBD (after [CLIENT] reviews this document)

---

## Related Documentation

- [PROJECT_OVERVIEW.md](./PROJECT_OVERVIEW.md) - Project summary and status
- [TECHNICAL_ARCHITECTURE.md](./TECHNICAL_ARCHITECTURE.md) - System design
- [COMPLIANCE.md](./COMPLIANCE.md) - Regulatory requirements
- [CONVERSATION_DESIGN.md](./CONVERSATION_DESIGN.md) - Call flows and scripts
