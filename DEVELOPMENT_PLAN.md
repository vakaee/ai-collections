# Development Plan - Phase 1

## Document Information

**Project**: AI Debt Collection Bot - Phase 1
**Version**: 2.0 (Updated with workflow adaptation timeline: 13.5h vs 40h from scratch)
**Last Updated**: October 3, 2025
**Timeline**: 4 weeks (October 3 - October 31, 2025)
**Budget**: [PRICE REDACTED] (+[BONUS REDACTED])

---

## 1. Timeline Overview

**Total Duration**: 28 days (4 weeks)
**Current Date**: October 3, 2025
**Deadline**: October 31, 2025
**Days Remaining**: 28 days

---

## 2. Development Phases

### Week 1: Foundation (Oct 3-9, 2025)

**Goal**: Set up infrastructure, configure Vapi, build first n8n workflow

#### Day 1-2 (Oct 3-4): Environment Setup
- [x] Create project documentation (this file and related docs)
- [ ] Set up Vapi account
- [ ] Purchase/configure phone number for outbound calls
- [ ] Create 6 Vapi assistants (consumer 1/2/3, commercial 1/2/3)
- [ ] Set up OpenAI API key
- [ ] Configure local n8n instance (Docker Compose)

**Deliverable**: Vapi account configured, local n8n running

---

#### Day 3-4 (Oct 5-6): Database & Interim CRM Setup
- [ ] Create PostgreSQL database (for AI call logs only)
- [ ] Implement database schema (ai_call_logs table)
- [ ] **Create Google Sheet** "BCS_Debtors" (interim CRM - ⚠️ STOPGAP)
- [ ] Set up n8n Google Sheets integration (OAuth or service account)
- [ ] Add 3-5 sample debtor rows for testing
- [ ] Test read/write to Google Sheets from n8n

**Deliverable**: PostgreSQL deployed, Google Sheets CRM ready, n8n can read/write

---

#### Day 5-7 (Oct 7-9): Adapt Existing n8n Workflows (10.5h total - PHASE 1)
- [ ] **Adapt Workflow 1: BCS Call Scheduler** (2h) - from "Daily sync.json"
  - Update Google Sheets schema reference (23 columns)
  - Add phone normalization (E.164 format)
  - Add business hours check with timezone
  - Add duplicate call prevention (call_status column)
  - Update assistant selection logic (6 assistants vs 2)
- [ ] **Adapt Workflow 2: BCS Vapi Webhook Handler** (3h) - from "Save results.json" + new logic
  - Add webhook signature verification (HMAC SHA-256)
  - Add complete outcome routing (11 outcome types)
  - Add call ID and recording URL tracking
  - Add manual review email notifications
  - READY_TO_PAY schedules follow-up call (no SMS in Phase 1)
- [ ] **Build Workflow 3: BCS Test Webhook** (0.5h) - new workflow
  - Mock Vapi payload for testing
- [ ] Configure 6 Vapi Assistants with verbal payment instructions (2h)
  - Add "pen and paper" confirmation scripts
  - Add slow, clear verbal delivery of payment details
  - Add confirmation that debtor wrote down information
- [ ] Integration testing (2h)
- [ ] Compliance testing (1h)
- [ ] Test with 1 sample call to test phone number

**Deliverable**: 3 workflows operational (Phase 1 scope), first successful AI call made with verbal payment delivery

**Time Savings**: 10.5h vs 40h from scratch (74% faster using Airbnb boilerplate)

**Phase 2 Workflows** (deferred to Phase 2 contract: $2,000-3,000):
- BCS Send Payment SMS (Twilio) - 1.5h
- BCS SMS Status Handler - 0.5h

---

### Week 2: Conversation Logic (Oct 10-16, 2025)

**Goal**: Implement conversation scripts, objection handling, outcome logging

#### Day 8-9 (Oct 10-11): Vapi Assistant Configuration
- [ ] **Draft scripts** based on Letter of Demand templates (will refine with [CLIENT] later)
- [ ] Configure Consumer 1st Call assistant (system prompt, functions)
- [ ] Configure Consumer 2nd Call assistant
- [ ] Configure Consumer 3rd Call assistant
- [ ] Configure Commercial 1st Call assistant
- [ ] Configure Commercial 2nd Call assistant
- [ ] Configure Commercial 3rd Call assistant
- [ ] Implement `log_outcome` function
- [ ] Implement `verify_identity` function
- [ ] ~~Implement `send_payment_sms` function~~ (PHASE 2 ONLY - deferred)

**Deliverable**: 6 Vapi assistants configured with draft scripts and verbal payment instructions (can swap scripts later)

---

#### Day 10-11 (Oct 12-13): Webhook Handler Workflow
- [ ] Build Workflow 2: Vapi Webhook Handler
  - Webhook trigger for call.ended event
  - Parse function_call for outcome
  - Route based on outcome (switch node)
  - Insert into ai_call_logs
  - Update CRM debtor record
  - Schedule next action (call, email, manual review)
- [ ] Test with various outcomes (promise to pay, dispute, no answer)

**Deliverable**: Webhook handling working, outcomes logged correctly

---

#### Day 12-14 (Oct 14-16): Testing & Refinement (PHASE 1)
- [ ] Test verbal payment delivery with multiple test calls
  - Verify AI asks for "pen and paper"
  - Verify AI reads payment details slowly and clearly
  - Verify AI confirms debtor wrote down information
- [ ] Test all 11 outcome types with various scenarios
- [ ] Test READY_TO_PAY outcome schedules follow-up call (3 days later)
- [ ] Refine Vapi assistant prompts based on test results
- [ ] Document any issues or edge cases discovered

**Deliverable**: Verbal payment instructions working reliably, all outcomes routing correctly

**PHASE 2 ONLY** (deferred to $2,000-3,000 contract):
- ~~Set up Twilio account~~ (SMS delivery)
- ~~Build SMS Payment Dispatcher~~ (credit card/bank/cheque SMS)
- ~~Build SMS Status Handler~~ (delivery confirmations)
- ~~Build Email Dispatcher~~ (dispute/hardship forms)

---

### Week 3: Testing & Refinement (Oct 17-23, 2025)

**Goal**: Rigorous testing, compliance validation, edge case handling

#### Day 15-16 (Oct 17-18): Synthetic Testing
- [ ] Create 20+ test scenarios (see Testing Strategy section)
- [ ] Role-play debtor responses (Vlad acts as debtor)
- [ ] Record and review all test calls
- [ ] Check compliance (mandatory disclosures, identity verification)
- [ ] Test edge cases:
  - Debtor hangs up mid-call
  - Wrong number
  - Third party answers
  - Angry debtor (escalation trigger)

**Deliverable**: 50+ test calls completed, transcripts reviewed

---

#### Day 17-18 (Oct 19-20): Compliance Validation
- [ ] Review all mandatory disclosures (identity, purpose, recording)
- [ ] Verify time-gating (no calls outside 7:30am-9pm)
- [ ] Test identity verification failures
- [ ] Test escalation triggers (lawyer, ombudsman)
- [ ] Ensure zero compliance violations

**Deliverable**: 100% compliance adherence verified

---

#### Day 19-21 (Oct 21-23): Refinement & UAT with Peter
- [ ] Share call recordings and transcripts with Peter
- [ ] Adjust tone/pacing based on feedback
- [ ] Refine scripts if needed
- [ ] Fix any bugs or edge cases
- [ ] Optimize call duration (target 2-4 minutes)

**Deliverable**: [CLIENT] approves call quality and compliance

---

### Week 4: Production Deployment (Oct 24-31, 2025)

**Goal**: Deploy to DigitalOcean, production pilot, final handover

#### Day 22-23 (Oct 24-25): DigitalOcean Deployment
- [ ] Provision DigitalOcean Droplet ($12/month, 2GB RAM)
- [ ] Install Docker and Docker Compose
- [ ] Configure firewall (SSH, HTTP, HTTPS)
- [ ] Deploy n8n + PostgreSQL via Docker Compose
- [ ] Set up SSL certificate (Let's Encrypt)
- [ ] Migrate n8n workflows from local to production
- [ ] Test webhook URL (ensure Vapi can reach server)

**Deliverable**: Production environment live, workflows deployed

---

#### Day 24-26 (Oct 26-28): Production Pilot
- [ ] Load 10-20 real debtor accounts (low-value, non-critical)
- [ ] Monitor calls in real-time
- [ ] Review outcomes immediately after each call
- [ ] Check CRM updates
- [ ] Iterate on any issues
- [ ] Scale to 50-100 accounts if successful

**Deliverable**: Successful production calls made, zero major issues

---

#### Day 27-28 (Oct 29-31): Final Handover
- [ ] Document all workflows (screenshots, descriptions)
- [ ] Create user guide for [CLIENT] (how to allocate accounts, review outcomes)
- [ ] Train [CLIENT] on accessing n8n dashboard
- [ ] Provide credentials (Vapi, n8n, database)
- [ ] Final review call with Peter
- [ ] Address any last-minute feedback

**Deliverable**: Project handed over, [CLIENT] can operate system independently

---

## 3. Success Criteria Checklist

Phase 1 is complete when:

- [x] Documentation complete (this file and related docs)
- [ ] Debt file can be allocated to platform
- [ ] Calls are made automatically
- [ ] Each call is logged in CRM with outcome
- [ ] All mandatory compliance disclosures made
- [ ] Identity verification works
- [ ] Consumer vs commercial scripts route correctly
- [ ] Objection handling works (dispute, hardship, payment)
- [ ] Human escalation triggers function
- [ ] Call recordings stored and accessible
- [ ] [CLIENT] can review outcomes and transcripts
- [ ] System deployed to DigitalOcean
- [ ] 10+ successful production calls made

**Bonus Criteria** ([BONUS REDACTED]):
- [ ] Zero compliance violations in first 50 calls
- [ ] 80%+ successful contact rate (excluding wrong numbers)
- [ ] Positive feedback from [CLIENT] on call quality

---

## 4. Testing Strategy

### 4.1 Synthetic Test Scenarios

| Scenario | Debtor Response | Expected Outcome |
|----------|----------------|------------------|
| Happy path | Agrees to pay by Oct 10 | Log PROMISE_TO_PAY, schedule follow-up Oct 11 |
| Dispute | "I don't owe this" | Log DISPUTE_RAISED, send email |
| Hardship | "I lost my job" | Log HARDSHIP_CLAIMED, send form |
| Payment plan | "Can I pay $100/month?" | Log PAYMENT_PLAN_REQUESTED, flag for manual review |
| Already paid | "I paid this last month" | Log MANUAL_REVIEW, request proof |
| Wrong person | "I'm not [debtor name]" | Log WRONG_NUMBER, end call |
| Third party | "He's not here" | Comply with privacy rules, request callback |
| Hang up | Debtor hangs up | Log HUNG_UP, schedule retry |
| No answer | Voicemail | Log NO_ANSWER, leave message, retry later |
| Angry debtor | "I'm calling my lawyer!" | ESCALATE_TO_HUMAN immediately |
| Deceased | "They passed away" | ESCALATE_TO_HUMAN, flag for removal |

**Test each scenario for both consumer and commercial scripts (22 total tests)**

---

### 4.2 Compliance Checks

| Check | Pass/Fail |
|-------|-----------|
| Identity disclosure made on every call | |
| Purpose statement made on every call | |
| Recording notice made on every call | |
| No debt details before identity verification | |
| No calls outside permitted hours (7:30am-9pm) | |
| No misleading statements | |
| No harassment or coercion | |
| Escalation triggers work (lawyer, ombudsman) | |
| Call recordings stored correctly | |
| Transcripts accessible for review | |

---

### 4.3 Integration Tests

| Test | Description | Pass/Fail |
|------|-------------|-----------|
| CRM Read | n8n can read debtor records from CRM | |
| CRM Write | n8n can update debtor records with outcomes | |
| Vapi Call | n8n can trigger Vapi outbound call | |
| Vapi Webhook | n8n receives and parses Vapi webhook | |
| Email Send | Dispute/hardship emails sent correctly | |
| Retry Logic | NO_ANSWER calls retry after 2 hours | |
| Max Attempts | Calls stop after 3 attempts | |
| Business Hours | Calls only scheduled within permitted hours | |

---

## 5. Risk Mitigation Plan

| Risk | Likelihood | Impact | Mitigation | Owner |
|------|------------|--------|------------|-------|
| CRM access delayed | High | High | Build with mock data, swap later | Vlad |
| Scripts delayed | Medium | High | Use letter templates as basis | Vlad |
| Vapi API issues | Low | High | Test thoroughly, have Twilio backup plan | Vlad |
| Compliance violations | Medium | Critical | Hard-code disclosures, manual review first 50 calls | Vlad |
| Debtor complaints | Medium | High | Human review of first 50 calls, adjust tone | [CLIENT] |
| Budget overruns (API) | Low | Medium | Cap test calls at 200, monitor usage | Vlad |
| Deadline slip | Medium | High | Start immediately, weekly check-ins | Vlad/[CLIENT] |
| Deployment issues | Medium | Medium | Test Docker locally first, have backup Droplet | Vlad |

---

## 6. Dependencies & Blockers

### ✅ BLOCKERS RESOLVED

**Original Blocker #1**: CRM Database Access
- **Status**: ✅ SOLVED
- **Solution**: Google Sheets interim CRM (no waiting for credentials)
- **Migration path**: Will swap to real CRM post-Phase 1

**Original Blocker #2**: Call Scripts
- **Status**: ✅ SOLVED
- **Solution**: Draft scripts from Letter of Demand templates, refine with [CLIENT] later
- **Note**: Vapi scripts can be updated in 5 minutes (not blocking)

### Remaining Dependencies (Not Blocking)

1. **Payment Information** (Nice to have by Oct 14)
   - **For bank transfer SMS**: BSB, account number, account name
   - **For credit card SMS**: Stripe payment link (or Vlad sets up)
   - **For cheque SMS**: BCS mailing address
   - **Workaround**: Use placeholders, update when provided

2. **Registered Email Address** (Nice to have by Oct 14)
   - Needed for: Dispute submissions
   - Example: disputes@brodiecollectionservices.com.au
   - **Workaround**: Use generic BCS email initially

3. **Hardship Form Template** (Nice to have by Oct 14)
   - Needed for: Email attachment
   - **Workaround**: Send generic template, replace later

4. **Phone Number for Outbound Calls** (Decision by Oct 5)
   - **Recommendation**: Use Vapi-provided Australian number
   - **Alternative**: Port existing BCS number (takes 2-4 weeks)
   - **Decision**: Vapi number (instant setup)

---

## 7. Budget Breakdown (Phase 1: Voice-Only)

| Item | Estimated Cost | Notes |
|------|----------------|-------|
| Development labor | $2,066 | Vlad's time (10.5h implementation) |
| Vapi subscription | $50 | 1 month, base plan |
| Vapi phone number | $10 | Australian number rental |
| Testing calls | $42 | 200 calls × 3 min × $0.07/min |
| OpenAI API | $30 | GPT-4o usage for 200+ calls |
| ~~Twilio SMS~~ | ~~$16~~ | **PHASE 2 ONLY** (deferred) |
| DigitalOcean Droplet | $12 | 1 month, $12/month plan |
| Domain name (optional) | $12 | If custom domain needed |
| SSL certificate | $0 | Let's Encrypt (free) |
| n8n license | $0 | Self-hosted, no cost |
| Google Sheets | $0 | Free (interim CRM) |
| Contingency | $278 | Buffer for overages |
| **TOTAL** | **$2,500** | Phase 1 contract amount |

**Savings from Phase 1 scope**: $16 Twilio SMS testing costs deferred to Phase 2

**Post-delivery ongoing costs** ([CLIENT]'s responsibility after handover):
- Vapi: $50/month + per-minute charges (~$0.05-0.09/min)
- DigitalOcean: $12/month
- OpenAI: Variable based on call volume (minimal, ~$0.01-0.03/call)
- ~~Twilio SMS~~ (Phase 2 only - not needed for Phase 1)

**Phase 2 Budget** (if commissioned: $2,000-3,000):
- SMS automation implementation: 2h ($200)
- Twilio setup and testing: $16
- Email automation: 1h ($100)
- Additional testing: $50

---

## 8. Delivery Milestones & Payment Schedule

**Milestone 1** (Oct 9): Foundation complete
- Vapi configured, n8n running, first test call successful
- Payment: Not applicable (fixed price contract)

**Milestone 2** (Oct 16): Workflows complete
- All 3 n8n workflows functional (Phase 1 scope), conversation logic working with verbal payment delivery
- Payment: Not applicable

**Milestone 3** (Oct 23): Testing complete
- 50+ test calls, compliance verified, [CLIENT] approval
- Payment: Not applicable

**Milestone 4** (Oct 31): Production deployment
- System live on DigitalOcean, production calls successful, handover complete
- Payment: [PRICE REDACTED] (via Upwork contract)
- Bonus: $500 (if success criteria met)

---

## 9. Communication Plan

### Weekly Check-ins with Peter

**Week 1** (Oct 9): Email update
- Progress: Vapi setup, n8n foundation
- Blockers: CRM access needed
- Next week: Build workflows, need scripts

**Week 2** (Oct 16): Email update + optional call
- Progress: Workflows complete
- Demo: Share test call recordings
- Next week: Testing begins

**Week 3** (Oct 23): Call with [CLIENT] (30-60 min)
- Review: Test call results
- Approval: Confirm quality and compliance
- Next week: Production deployment

**Week 4** (Oct 30): Final handover call
- Training: How to use the system
- Documentation: User guide walkthrough
- Credentials: Vapi, n8n, database access

---

## 10. Post-Delivery Support

**Included in [PRICE REDACTED]**:
- 7 days of bug fixes (Oct 31 - Nov 7)
- Minor adjustments to scripts/workflows

**Not included** (separate quote if needed):
- New features (SMS, email automation)
- CRM rebuild
- Scaling beyond 100 calls/day
- Ongoing maintenance after Nov 7

---

## 11. Knowledge Transfer

### Documentation to Provide

1. **User Guide** (for Peter):
   - How to allocate debt files to AI bot
   - How to review call outcomes in CRM
   - How to access n8n dashboard
   - How to listen to call recordings
   - How to manually escalate accounts

2. **Technical Documentation** (for [CLIENT]'s IT):
   - n8n workflow diagrams
   - Database schema
   - Vapi configuration
   - Docker Compose setup
   - Deployment instructions

3. **Credentials**:
   - n8n dashboard login
   - Vapi account access
   - Database credentials
   - DigitalOcean Droplet SSH key

---

## 12. Success Metrics (Track Weekly)

| Week | Metric | Target | Actual |
|------|--------|--------|--------|
| 1 | Vapi setup complete | Yes | |
| 1 | First test call successful | Yes | |
| 2 | All workflows functional | Yes | |
| 2 | Test calls made | 10+ | |
| 3 | Test calls made (cumulative) | 50+ | |
| 3 | Compliance violations | 0 | |
| 3 | [CLIENT] approval | Yes | |
| 4 | Production deployment | Yes | |
| 4 | Production calls successful | 10+ | |
| 4 | Handover complete | Yes | |

---

## 13. Contingency Plans

**If CRM access is delayed beyond Oct 6**:
- Proceed with mock data
- Build all workflows
- Swap in real CRM connection when available

**If scripts are delayed beyond Oct 11**:
- Draft scripts based on letter templates
- Get [CLIENT]'s approval via email
- Adjust in Vapi after feedback

**If Vapi has outages**:
- Have Twilio account ready as backup (not ideal, but functional)
- Notify [CLIENT] of delay

**If deadline is at risk (Oct 25+)**:
- Reduce testing to 25 calls instead of 50
- Deploy with smaller production pilot (5 accounts)
- Extend support period to compensate

---

## 14. Related Documentation

- [PROJECT_OVERVIEW.md](./PROJECT_OVERVIEW.md) - Project summary and status
- [REQUIREMENTS.md](./REQUIREMENTS.md) - Functional requirements
- [TECHNICAL_ARCHITECTURE.md](./TECHNICAL_ARCHITECTURE.md) - System design (n8n, Vapi, DigitalOcean)
- [COMPLIANCE.md](./COMPLIANCE.md) - Australian regulations
- [CONVERSATION_DESIGN.md](./CONVERSATION_DESIGN.md) - Call flows and scripts
- [PENDING_ITEMS.md](./PENDING_ITEMS.md) - Current blockers and questions

---

**End of Development Plan**
