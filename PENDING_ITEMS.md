# Pending Items & Blockers

## Document Information

**Project**: AI Debt Collection Bot - Phase 1
**Version**: 2.0 (Added 3 new decision items from workflow adaptation analysis)
**Last Updated**: October 3, 2025
**Status**: Awaiting multiple inputs from Peter

---

## 1. ✅ BLOCKERS RESOLVED

### 1.1 CRM Database Access - ✅ SOLVED

**Original Issue**: Needed Peter's CRM database access to store/retrieve debtor records

**Solution**: **Google Sheets Interim CRM**
- Created "BCS_Debtors" Google Sheet with full schema
- n8n native integration (read/write via Google Sheets node)
- Peter can view/edit data in real-time
- Easy migration to real CRM later (swap n8n node)

**Status**: ✅ NO LONGER BLOCKING

**Future**: When Peter provides CRM access, will migrate from Google Sheets to real database (2-4 hour migration)

---

### 1.2 Call Scripts - ✅ SOLVED

**Original Issue**: Needed finalized call scripts from Peter for Vapi configuration

**Solution**: **Draft Scripts from Letter Templates**
- Using Letter of Demand templates as basis
- Drafting consumer + commercial scripts (1st/2nd/3rd calls)
- Vapi assistants can be updated in 5 minutes (scripts are just configuration)
- Will refine scripts with Peter's feedback after first test calls

**Status**: ✅ NO LONGER BLOCKING

**Future**: Peter can provide finalized scripts anytime, easy to update in Vapi

---

## 2. High Priority Items (Nice to Have, Not Blocking)

### 2.1 Payment Information for SMS Delivery

**Needed from**: Peter

**Required for**:
- SMS payment delivery (via Twilio)
- Conversation scripts

**Details needed**:

**For bank transfer SMS**:
- BSB number
- Account number
- Account name
- BCS phone number (for questions)

**For credit card SMS**:
- Stripe payment link (or Vlad can set up)
- OR generic payment portal URL
- BCS phone number (for questions)

**For cheque SMS**:
- BCS mailing address
- BCS phone number (for questions)

**Urgency**: 🟡 Medium - Need by Oct 14 (when SMS workflows built)

**Workaround**: Use placeholder text in SMS templates, update later

**Status**: PENDING

---

### 2.2 Registered Email Addresses

**Needed from**: Peter

**Required for**:
- Dispute submissions
- Hardship form delivery
- Account-related inquiries

**Email addresses needed**:
- **Disputes email**: e.g., `disputes@brodiecollectionservices.com.au`
- **Hardship email**: e.g., `hardship@brodiecollectionservices.com.au` (or same as disputes)
- **General inquiries**: e.g., `accounts@brodiecollectionservices.com.au`

**Urgency**: 🟡 Medium - Need by Oct 12 (when email workflows built)

**Status**: PENDING

---

### 2.3 Hardship Form Template

**Needed from**: Peter

**Required for**:
- Email attachment when debtor claims financial hardship
- Must comply with Australian hardship assessment regulations

**Format**: PDF preferred (will be attached to email)

**Contents** (typical hardship form):
- Debtor personal details
- Income and expenses statement
- Assets and liabilities
- Reason for hardship
- Proposed payment arrangement

**Urgency**: 🟡 Medium - Need by Oct 12

**Status**: PENDING

---

### 2.4 SMS Templates

**Needed from**: Peter

**Required for**:
- Future Phase 2 (SMS automation)
- Reference for tone and language

**Templates needed**:
1. **1st SMS**: Notification of action taken
2. **2nd SMS**: Escalation/consequences
3. **3rd SMS**: Confirmation of default recorded

**Urgency**: 🟢 Low - Not needed for Phase 1 (voice only)

**Status**: PENDING (nice to have for future reference)

---

## 3. Decisions Needed from Peter

### 3.1 Phone Number for Outbound Calls

**Question**: Should the AI bot use:
- **Option A**: Vapi-provided Australian phone number (instant setup, $10/month)
- **Option B**: Existing BCS phone number (requires porting, 2-4 weeks, may disrupt current usage)

**Recommendation**: Use Vapi number for Phase 1 (faster, safer)

**Urgency**: 🔴 High - Need decision by Oct 5

**Status**: PENDING

---

### 3.2 Human Escalation Protocol

**Question**: When AI bot needs to escalate to a human, what should happen?

**Options**:
- **Option A**: Transfer call to Peter's mobile (requires Vapi call forwarding setup)
- **Option B**: End call and leave callback number for BCS
- **Option C**: Flag for manual review, BCS calls debtor back later

**Recommendation**: Option C for Phase 1 (simpler, no real-time human availability needed)

**Urgency**: 🟡 Medium - Need decision by Oct 14

**Status**: PENDING

---

### 3.3 Business Hours & Timezone

**Question**: Confirm calling hours and timezone

**Assumptions**:
- Weekdays: 7:30am - 9:00pm
- Weekends: 9:00am - 9:00pm
- Timezone: AEST (or ACST depending on debtor location)

**Need to confirm**:
- Are debtors across multiple Australian timezones?
- Should system detect debtor timezone from phone area code?
- Any exceptions (e.g., no Sunday calls)?

**Urgency**: 🟡 Medium - Need confirmation by Oct 9

**Status**: PENDING

---

### 3.4 Call Volume Estimate

**Question**: How many calls per day/week expected once system is live?

**Needed for**:
- Vapi plan selection (concurrent call limits)
- Cost estimation (per-minute charges)
- Database scaling

**Urgency**: 🟢 Low - Nice to have for capacity planning

**Status**: PENDING

---

### 3.5 Timezone Handling Approach (NEW)

**Question**: How should the system handle debtors across multiple Australian timezones?

**Background**: Australia has 3 main timezones (AEST, ACST, AWST). Calling 7:30am Sydney time = 5:00am Perth time (non-compliant).

**Options**:
- **Option A**: Add `timezone` column to Google Sheets (Peter manually sets for each debtor)
- **Option B**: Auto-detect from phone area code (complex, error-prone)
- **Option C**: Assume all debtors in single timezone (risky if debtors nationwide)

**Recommendation**: Option A (explicit timezone column)

**Urgency**: 🟡 Medium - Need decision by Oct 9

**Status**: PENDING

---

### 3.6 Attempt Number Increment Policy (NEW)

**Question**: Should attempt_number increment on every call or only successful contacts?

**Scenario**: 1st call → NO_ANSWER → retry in 4 hours → is this still attempt 1 or attempt 2?

**Options**:
- **Option A**: Every call counts (NO_ANSWER increments attempt_number)
  - **Pros**: Safer, limits total calls to debtor (max 3 calls regardless of outcome)
  - **Cons**: Debtor with genuine unavailability gets fewer chances
- **Option B**: Only answered calls count
  - **Pros**: More persistent, better contact rate
  - **Cons**: Risk of harassment if debtor genuinely unavailable

**Recommendation**: Option A (every call counts - safer for compliance)

**Urgency**: 🟡 Medium - Need decision by Oct 9

**Status**: PENDING

---

### 3.7 Call Recording Retention (NEW)

**Question**: How long should call recordings be kept?

**Options**:
- **Vapi default**: 7-30 days (included in plan)
- **Extended retention**: Download recordings to DigitalOcean Spaces (Phase 2 feature)
- **Compliance requirement**: May require 7 years for Australian debt collection

**Recommendation**: Use Vapi 30-day default for Phase 1, implement long-term archival in Phase 2 if required

**Urgency**: 🟢 Low - Nice to have for Phase 1

**Status**: PENDING

---

## 4. Optional / Nice to Have

### 4.1 CRM Screenshots or Demo

**Request**: Screenshots or demo of current CRM interface

**Purpose**:
- Understand workflow
- Design better integration
- Plan future CRM rebuild (Phase 3) if needed

**Urgency**: 🟢 Low - Not critical

**Status**: PENDING

---

### 4.2 Sample Debtor Records (Anonymized)

**Request**: 5-10 sample debtor records (names/details anonymized)

**Purpose**:
- Test data structure
- Build realistic test scenarios
- Validate database integration

**Format**: CSV, Excel, or database export

**Urgency**: 🟢 Low - Can use synthetic data

**Status**: PENDING

---

### 4.3 Existing Call Recordings (if any)

**Request**: Recordings of actual debt collection calls (if BCS staff have made any)

**Purpose**:
- Understand tone and pacing
- Identify common objections
- Improve AI bot realism

**Urgency**: 🟢 Low - Nice to have for refinement

**Status**: PENDING

---

## 5. Information Provided by Peter

### 5.1 Received ✅

| Item | Date Received | Source |
|------|---------------|--------|
| Scoping/requirements document (PDF) | Sep 10, 2025 | Upwork message |
| Letter of Demand (Consumer) template | Sep 26, 2025 | Email (found in spam) |
| Letter of Demand (Commercial) template | Sep 26, 2025 | Email (found in spam) |
| Test database link (mentioned) | Sep 26, 2025 | Email (link TBD) |
| Contact information (email) | Sep 24, 2025 | Upwork message |

---

### 5.2 Clarifications Provided ✅

| Question | Answer | Date |
|----------|--------|------|
| Integrate into existing CRM or build custom? | Existing CRM is basic, lacks attachment support; may need rebuild | Sep 16, 2025 |
| Current telephony provider? | All calls made via mobile phones; inbound handled by virtual reception | Sep 16, 2025 |
| Compliance scripts in scope? | Have letter/email/SMS templates; need conversational scripts drafted for debtor questions | Sep 16, 2025 |
| Initial channel priority? | Voice first, SMS/email to follow (staged approach) | Sep 16, 2025 |

---

## 6. Action Items for Vlad

### 6.1 Immediate (This Week)

- [ ] Set up Vapi account
- [ ] Create PostgreSQL database schema
- [ ] Build n8n local development environment
- [ ] Draft consumer/commercial scripts based on letter templates (for Peter's review)
- [ ] Create mock debtor data for testing

---

### 6.2 Waiting on Peter

- [ ] Receive CRM database access
- [ ] Receive call scripts (or approve drafted scripts)
- [ ] Receive payment information
- [ ] Receive registered email addresses
- [ ] Receive hardship form template
- [ ] Decide on phone number approach (Vapi vs. existing)
- [ ] Confirm business hours and timezone

---

## 7. Communication Log

| Date | Method | Topic | Outcome |
|------|--------|-------|---------|
| Sep 10 | Upwork | Project offer and scoping doc | Accepted |
| Sep 11 | Google Docs | Proposal submitted | Positive response |
| Sep 21 | Upwork | Budget negotiation | $2.5k + $500 bonus agreed |
| Sep 22 | Upwork | Contract started | Active |
| Sep 24 | Upwork | Email address exchange | peter@brodiecollectionservices.com.au |
| Sep 25 | Email | Information request sent | Awaiting response |
| Sep 26 | Email | Templates and database link sent | Received (in spam folder) |
| Oct 1 | Upwork | Progress update | Scaffolding complete |
| Oct 2 | Upwork | Peter confirms receiving email | Resolved |
| Oct 3 | — | Documentation creation | In progress |

---

## 8. Next Steps - ✅ NO BLOCKERS

### For Peter (Optional - Not Blocking Development):

**Nice to have by Oct 14**:
1. Payment information (BSB, account number, Stripe link, mailing address)
2. Registered email addresses (disputes, hardship)
3. BCS phone number (for SMS/scripts)

**Nice to have by Oct 16**:
1. Hardship form template
2. Finalized call scripts (can refine Vlad's drafts)
3. Confirm business hours and timezone

**Note**: ✅ None of these are blocking. Vlad can proceed immediately with Google Sheets CRM and drafted scripts.

---

### For Vlad (Starting Immediately):

**This week** (Oct 3-9):
1. ✅ Complete documentation (DONE)
2. Set up Vapi account + Australian phone number
3. Set up Twilio account
4. Set up n8n locally (Docker Compose)
5. **Create Google Sheet "BCS_Debtors"** with sample data
6. Build PostgreSQL database (for call logs only)
7. Draft consumer/commercial scripts from letter templates
8. Build first n8n workflow (Call Scheduler with Google Sheets)

**Next week** (Oct 10-16):
1. Configure 6 Vapi assistants with draft scripts
2. Build SMS payment workflows (Twilio)
3. Test end-to-end with sample calls

---

## 9. Risk Assessment - ✅ ALL RISKS MITIGATED

| Original Risk | Status | Solution |
|---------------|--------|----------|
| CRM access delayed | ✅ RESOLVED | Google Sheets interim CRM |
| Call scripts delayed | ✅ RESOLVED | Draft from letter templates |
| Payment info delayed | 🟢 LOW RISK | Use placeholders in SMS, update later |
| Email addresses delayed | 🟢 LOW RISK | Use generic BCS email initially |
| Hardship form delayed | 🟢 LOW RISK | Send generic template, replace later |
| Phone number decision | ✅ RESOLVED | Use Vapi-provided Australian number |

**Overall risk level**: 🟢 LOW - All major blockers resolved. Development can proceed immediately with no dependencies on Peter.

---

## 10. Related Documentation

- [PROJECT_OVERVIEW.md](./PROJECT_OVERVIEW.md) - Project status and timeline
- [DEVELOPMENT_PLAN.md](./DEVELOPMENT_PLAN.md) - Week-by-week deliverables and dependencies
- [CONVERSATION_DESIGN.md](./CONVERSATION_DESIGN.md) - Placeholder scripts awaiting Peter's input
- [TEMPLATES_ANALYSIS.md](./TEMPLATES_ANALYSIS.md) - Analysis of provided letter templates

---

**End of Pending Items Document**

**Last Updated**: October 3, 2025
**Next Review**: October 6, 2025 (after expected CRM access)
