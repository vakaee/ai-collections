# Project Overview: AI Debt Collection Bot

## Executive Summary

Building an AI-powered voice bot for outbound debt collection calls on behalf of [COLLECTION AGENCY] (BCS), an Australian debt collection agency. The system will automate initial contact, follow-ups, and payment negotiations while maintaining strict compliance with Australian regulations.

---

## Project Details

**Client**: [CLIENT NAME]
**Company**: [COLLECTION AGENCY] (Australia)
**Developer**: [DEVELOPER NAME]
**Contract Platform**: Upwork
**Budget**: [PRICE REDACTED]
**Timeline**: September 23, 2025 - October 31, 2025 (5 weeks)
**Phase**: Phase 1 (Voice-first, outbound calls only)

---

## Stakeholders

### Primary Contact
- **Name**: [CLIENT NAME]
- **Email**: [CLIENT EMAIL]
- **Role**: Client, BCS Owner
- **Availability**: Limited due to work commitments, prefers email communication

### Technical Contact
- **Name**: [CLIENT]'s brother (name TBD)
- **Role**: IT support, built current CRM
- **Responsibility**: Provide CRM access/schema documentation

### Developer
- **Name**: [DEVELOPER NAME]
- **Email**: [DEVELOPER EMAIL]
- **Phone**: [DEVELOPER PHONE]

---

## Phase 1 Success Criteria

Per [CLIENT]'s definition, Phase 1 is complete when:

1. **Debt file allocation**: System can accept a debtor account for processing
2. **Automated calling**: Outbound calls are made automatically
3. **Outcome logging**: Each call is logged in the CRM with one of the following outcomes:
   - Repayment plan requested
   - Promise to pay
   - Dispute raised
   - Message left with callback details
   - Other structured outcomes (see CONVERSATION_DESIGN.md)

**Important constraint**: Default placement (personal or company) remains a manual decision - the AI will NOT execute defaults.

---

## Project Objectives

### Primary Goals
1. Reduce manual calling workload for BCS staff
2. Increase contact rate and payment recovery efficiency
3. Maintain 100% compliance with Australian debt collection regulations
4. Provide audit trail of all debtor interactions

### Secondary Goals
1. Gather data on common objections and payment patterns
2. Identify candidates for payment plans vs. escalation
3. Build foundation for Phase 2 (SMS/email automation)
4. Build foundation for Phase 3 (CRM rebuild if needed)

---

## Current Project Status

**Start Date**: September 23, 2025
**Current Date**: October 3, 2025
**Days Elapsed**: 10 days
**Days Remaining**: 28 days

### What We Have
- Contract signed and active
- Budget confirmed ([PRICE REDACTED] + [BONUS REDACTED])
- Two Letter of Demand templates (consumer + commercial) received and analyzed
- High-level requirements from [CLIENT]'s scoping document
- Proposal accepted
- Initial communication established

### What We're Waiting For (from [CLIENT])
- Test CRM database access (credentials or API documentation)
- Call scripts (1st/2nd/3rd call for consumer + commercial debtors)
- SMS templates (1st/2nd/3rd message)
- Registered email address for dispute submissions
- Hardship proof template
- Clarification on phone number to use for outbound calls

### Development Status
- **Phase 1a**: Not started (awaiting CRM access and scripts)
- **Documentation**: In progress (this file)
- **Technical research**: Complete (architecture decisions made)

---

## Budget Breakdown

| Item | Estimated Cost |
|------|----------------|
| Development labor | $2,100 |
| Vapi subscription (1 month) | $50 |
| Testing calls (200 × 3 min × $0.07/min) | $42 |
| OpenAI API (LLM calls) | $50 |
| Database hosting (if needed) | $20 |
| Contingency | $238 |
| **TOTAL** | **[PRICE REDACTED]** |

**Bonus**: $500 if Phase 1 is successful

---

## Future Phases (Out of Scope for Phase 1)

### Phase 2: Multi-Channel Expansion
- SMS automation (3-tier progression)
- Email automation (templated follow-ups)
- Estimated cost: $2,000-[PRICE REDACTED]

### Phase 3: CRM Rebuild (if required)
- Custom CRM mirroring existing model
- Template/attachment support
- Document management
- Estimated cost: $4,000-$6,000

---

## Key Constraints

1. **Time**: 28 days remaining to deliver working system
2. **Budget**: [PRICE REDACTED] fixed price
3. **Compliance**: Must adhere to Australian regulations (ASIC/ACCC)
4. **CRM**: Limited attachment support in current system
5. **Telephony**: All current calls made via mobile phones (no existing provider)
6. **Human oversight**: Defaults remain manual decision
7. **[CLIENT]'s availability**: Limited time for meetings/reviews

---

## Risk Assessment

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| CRM access delayed | High | High | Build with mock data, swap later |
| Scripts delayed | Medium | High | Use letter templates as basis |
| Compliance violations | Low | Critical | Hard-code mandatory disclosures |
| Budget overrun (API costs) | Low | Medium | Cap test calls at 200 |
| Debtor complaints | Medium | High | Human review of first 50 calls |
| Deadline slip | Medium | High | Weekly check-ins, start immediately |

---

## Communication Protocol

**Primary channel**: Email ([CLIENT]'s preference)
**Response time**: [CLIENT] may take 24-48 hours due to work commitments
**Meeting cadence**: As-needed basis (call when necessary for complex topics)
**Status updates**: Weekly progress emails recommended

---

## Success Metrics (Beyond Phase 1 Criteria)

Once deployed, track:
- **Call connection rate**: % of calls answered vs. no answer
- **Outcome distribution**: Payment commitments vs. disputes vs. no contact
- **Average call duration**: Target 2-4 minutes
- **Human escalation rate**: % requiring manual intervention
- **Compliance adherence**: 100% required (all mandatory disclosures made)
- **Cost per call**: Monitor Vapi usage to ensure scalability

---

## Document Version

**Version**: 1.0
**Last Updated**: October 3, 2025
**Author**: [DEVELOPER NAME]
**Next Review**: October 10, 2025 (weekly cadence)

---

## Related Documentation

- [REQUIREMENTS.md](./REQUIREMENTS.md) - Detailed functional and non-functional requirements
- [TECHNICAL_ARCHITECTURE.md](./TECHNICAL_ARCHITECTURE.md) - System design and tech stack
- [DEVELOPMENT_PLAN.md](./DEVELOPMENT_PLAN.md) - 4-week development timeline
- [COMPLIANCE.md](./COMPLIANCE.md) - Australian debt collection regulations
- [CONVERSATION_DESIGN.md](./CONVERSATION_DESIGN.md) - Call flows and scripts
- [PENDING_ITEMS.md](./PENDING_ITEMS.md) - Blockers and questions for Peter
