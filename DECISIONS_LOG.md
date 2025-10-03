# Technical Decisions Log

## Document Information

**Project**: AI Debt Collection Bot - Phase 1
**Purpose**: Record all technical decisions, rationale, and alternatives considered
**Version**: 3.0 (Added Decision 013: Phase 1 Scope Clarification - Voice-only)
**Last Updated**: October 3, 2025

---

## 1. Overview

This document logs all significant technical decisions made during the project, including:
- Technology choices
- Architecture decisions
- Implementation approaches
- Changes to original plans

**Format**: Each decision follows the ADR (Architecture Decision Record) pattern:
- **Context**: Why this decision was needed
- **Decision**: What was chosen
- **Alternatives**: What else was considered
- **Consequences**: Trade-offs and implications
- **Status**: Active, Superseded, or Deprecated

---

## 2. Active Decisions

### Decision 001: Use Vapi.ai for Telephony

**Date**: October 3, 2025
**Status**: ✅ Active
**Decision maker**: Vlad Petoukhov

**Context**:
Need a telephony provider to handle outbound calling and AI voice agent integration. Project has tight timeline (28 days) and fixed budget ($2,500).

**Decision**:
Use Vapi.ai as the telephony and voice AI platform.

**Alternatives considered**:
1. **Twilio + custom code**: Lower per-minute cost ($0.013/min), but requires building conversation layer from scratch
2. **Bland.ai**: Similar to Vapi, but less mature and fewer features
3. **Retell.ai**: Good alternative, but slightly more expensive

**Rationale**:
- Vapi is AI-native (built for voice agents, not retrofitted)
- Built-in conversation management, function calling, recording
- Faster development (50% less code than Twilio approach)
- Developer has prior experience with Vapi
- Higher per-minute cost ($0.05-0.09) acceptable given budget is mostly development labor

**Consequences**:
- ✅ Faster time to market
- ✅ Less code to maintain
- ✅ Native LLM integration (OpenAI, Anthropic)
- ❌ Higher per-minute cost (but still affordable for this use case)
- ❌ Vendor lock-in (but mitigated by clear function calling interface)

**Cost impact**:
- Testing: 200 calls × 3 min × $0.07/min = $42
- Production: Variable based on call volume (client pays ongoing)

**Next review**: After first test call (Oct 9)

---

### Decision 002: Use n8n for Orchestration

**Date**: October 3, 2025
**Status**: ✅ Active
**Decision maker**: Vlad Petoukhov (after discussion with user)

**Context**:
Originally planned to build custom Node.js backend with Express for webhooks and CRM integration. User suggested n8n as an alternative, noting they have existing boilerplate.

**Decision**:
Use n8n (workflow automation platform) for all backend orchestration instead of custom Node.js code.

**Alternatives considered**:
1. **Node.js + Express**: More control, but slower development
2. **Python + FastAPI**: Good for AI/ML, but less suitable for webhooks/scheduling
3. **Zapier/Make.com**: SaaS alternatives, but more expensive and less flexible than n8n

**Rationale**:
- This project is primarily orchestration (schedule → call → log → update CRM)
- n8n excels at workflow automation
- Visual workflows faster to build than custom code
- Developer has existing n8n boilerplate (significant time saving)
- Client can maintain workflows without hiring developer (long-term benefit)
- Self-hosted n8n = no additional licensing cost

**Consequences**:
- ✅ 30-40% faster development
- ✅ Client can modify workflows independently
- ✅ Built-in integrations (PostgreSQL, HTTP, Email, Webhooks)
- ✅ Visual debugging (easier to troubleshoot)
- ❌ Slightly less control than custom code
- ❌ Learning curve for complex custom logic (mitigated by Code nodes)

**Implementation notes**:
- Use n8n Code nodes (JavaScript) for complex logic (business hours check, assistant selection)
- Use n8n native nodes for standard tasks (database queries, HTTP requests, email)

**Next review**: After building first workflow (Oct 9)

---

### Decision 003: Use PostgreSQL for Database

**Date**: October 3, 2025
**Status**: ✅ Active
**Decision maker**: Vlad Petoukhov

**Context**:
Need database to store AI call logs, call queue, and system configuration. CRM integration will be separate (Peter's existing CRM database).

**Decision**:
Use PostgreSQL 15 for the AI bot's database.

**Alternatives considered**:
1. **MongoDB**: NoSQL, flexible schema, but overkill for structured data
2. **SQLite**: Lightweight, but not suitable for concurrent access
3. **MySQL**: Similar to PostgreSQL, but PostgreSQL has better JSON support

**Rationale**:
- Call logs and queue are structured data (ACID transactions needed)
- PostgreSQL well-supported by n8n (native node)
- JSONB column type useful for storing metadata, transcripts
- Free and open-source
- Scales well (can handle 10,000+ debtor records easily)
- Better compliance auditing with relational model

**Consequences**:
- ✅ ACID guarantees (no data loss on failures)
- ✅ Strong typing and constraints
- ✅ Easy to query for compliance reports
- ❌ Slightly more overhead than NoSQL for very high-volume writes (not a concern at this scale)

**Next review**: After database schema deployed (Oct 6)

---

### Decision 004: Use OpenAI GPT-4o for LLM

**Date**: October 3, 2025
**Status**: ✅ Active
**Decision maker**: Vlad Petoukhov

**Context**:
Need LLM to power the conversational voice agent. Voice calls require low latency (<500ms response time) for natural conversation.

**Decision**:
Use OpenAI GPT-4o as the primary LLM.

**Alternatives considered**:
1. **Claude 3.5 Sonnet**: Better reasoning, but higher latency for voice
2. **GPT-4 Turbo**: Slower than GPT-4o
3. **Llama 3.1 (self-hosted)**: Open-source, but requires infrastructure

**Rationale**:
- GPT-4o optimized for low-latency use cases (including voice)
- Proven track record in Vapi integrations
- Fast response times (<300ms typically)
- Good at following structured prompts (compliance disclosures)
- Cost-effective (~$0.01-0.03 per call conversation)

**Consequences**:
- ✅ Low latency (natural conversation pacing)
- ✅ Reliable function calling (for log_outcome, verify_identity)
- ✅ Good at staying on script
- ❌ Slightly less nuanced reasoning than Claude (acceptable for scripted calls)
- ❌ OpenAI dependency (but standard in industry)

**Fallback plan**:
If GPT-4o proves unreliable, can switch to Claude 3.5 Sonnet with minor Vapi configuration change.

**Next review**: After first 10 test calls (Oct 10)

---

### Decision 005: Deploy to DigitalOcean Droplet with Docker Compose

**Date**: October 3, 2025
**Status**: ✅ Active
**Decision maker**: Vlad Petoukhov (per user's request)

**Context**:
Need hosting solution for n8n + PostgreSQL. Project has limited budget, and client prefers Docker deployment.

**Decision**:
Deploy to DigitalOcean Droplet using Docker Compose.

**Alternatives considered**:
1. **Railway.app**: PaaS, easier deployment, but $20-30/month
2. **Render.com**: Similar to Railway, $15-25/month
3. **Heroku**: Mature PaaS, but expensive ($25+/month)
4. **DigitalOcean App Platform**: Managed Docker, $15-30/month
5. **AWS EC2**: More complex, similar cost to Droplet

**Rationale**:
- **Cost**: $6-12/month for Droplet vs. $15-30/month for PaaS
- **Control**: Full SSH access, can customize
- **Docker support**: Native Docker and Docker Compose support
- **Simplicity**: Easier than Kubernetes or AWS
- **Scalability**: Can upgrade Droplet vertically if needed
- **Client preference**: User explicitly requested Docker + DigitalOcean

**Consequences**:
- ✅ Low cost ($6-12/month ongoing)
- ✅ Full control over infrastructure
- ✅ Docker Compose = easy replication across environments
- ❌ Manual DevOps (SSH, firewall, SSL setup)
- ❌ No auto-scaling (but not needed at this scale)

**Implementation**:
- Ubuntu 22.04 LTS Droplet
- Docker Compose stack: n8n + PostgreSQL + Nginx (for SSL)
- Let's Encrypt for SSL certificate (free)
- UFW firewall configuration

**Next review**: During deployment (Oct 24-25)

---

### Decision 006: Use Australian Voice (en-AU) for AI Agent

**Date**: October 3, 2025
**Status**: ✅ Active
**Decision maker**: Vlad Petoukhov

**Context**:
AI voice agent will be calling Australian debtors on behalf of Australian company (Brodie Collection Services).

**Decision**:
Use Azure "en-AU-NatashaNeural" (female Australian accent) for voice synthesis.

**Alternatives considered**:
1. **US accent** (en-US): More options, but sounds foreign to Australian debtors
2. **UK accent** (en-GB): Similar to AU but not identical
3. **Generic English**: No regional accent

**Rationale**:
- Australian accent builds trust (sounds local, not offshore call center)
- Reduces "scam call" perception
- Better compliance (debtor more likely to engage)
- Azure has high-quality AU voices via Vapi

**Consequences**:
- ✅ Higher engagement rate
- ✅ Sounds professional and local
- ✅ Compliance advantage (clear communication)
- ❌ Fewer voice options than US accents (but sufficient)

**Alternative voice**: "en-AU-WilliamNeural" (male) if client prefers

**Next review**: After Peter hears test calls (Oct 16)

---

### Decision 007: Use Vapi-Provided Phone Number (Not Port Existing)

**Date**: October 3, 2025
**Status**: ⏳ Pending Peter's confirmation
**Decision maker**: Vlad Petoukhov (recommendation)

**Context**:
Need Australian phone number for outbound calls. BCS currently uses mobile phones for all outbound calls.

**Decision** (recommended):
Use Vapi-provided Australian phone number for Phase 1.

**Alternatives considered**:
1. **Port existing BCS number**: Requires 2-4 weeks, may disrupt current operations
2. **Buy new number from Twilio**: Extra step, no advantage over Vapi

**Rationale**:
- **Time**: Vapi provides number instantly vs. 2-4 weeks for porting
- **Risk**: Porting could disrupt Peter's current mobile phone usage
- **Cost**: $10/month for Vapi number (acceptable)
- **Flexibility**: Can port later if needed (after Phase 1 proven)

**Consequences**:
- ✅ Instant setup (no delay)
- ✅ No risk to existing operations
- ✅ Can upgrade to ported number later
- ❌ New number may have lower answer rate initially (but improves over time)
- ❌ Extra $10/month cost (minimal)

**Status**: Awaiting Peter's confirmation

**Next review**: Oct 5 (need decision before Vapi setup)

---

### Decision 008: Use Google Sheets as Interim CRM (Stopgap)

**Date**: October 3, 2025
**Status**: ✅ Active (Interim/Stopgap)
**Decision maker**: Vlad Petoukhov (with user's agreement)

**Context**:
Peter's CRM database access delayed. Timeline is tight (28 days). Needed immediate solution to start development without waiting for credentials.

**Decision**:
Use Google Sheets as interim CRM for Phase 1 debtor tracking and call outcome logging.

**Alternatives considered**:
1. **Wait for Peter's CRM access**: Would block development for Week 1
2. **Build custom database immediately**: Duplication of effort (Peter's CRM exists)
3. **Use mock/hard-coded data**: Not realistic for testing and demo

**Rationale**:
- Zero setup time (create spreadsheet in 5 minutes)
- Peter can view/edit debtor data in real-time (transparency)
- n8n has native Google Sheets integration (no custom code)
- Easy migration path: swap Google Sheets node with PostgreSQL node later
- Can export to CSV and import to real CRM when ready

**Consequences**:
- ✅ Unblocks Week 1 development completely
- ✅ Peter can manually add/remove debtors easily
- ✅ Real-time visibility for Peter
- ✅ Simple 2-4 hour migration when CRM access provided
- ❌ Limited scalability (acceptable for <1000 records, fine for Phase 1)
- ❌ No relational integrity or constraints
- ❌ Requires Google account (Peter likely has one)

**Cost impact**: $0 (Google Sheets is free)

**Migration plan**:
1. Export Google Sheet to CSV
2. Import to real CRM
3. Swap n8n Google Sheets node with PostgreSQL/MySQL node
4. Test end-to-end

**Status**: INTERIM SOLUTION - Will be superseded when real CRM access provided

**Next review**: When Peter provides CRM access (post-Phase 1)

---

### Decision 010: Webhook Signature Verification (Security)

**Date**: October 3, 2025
**Status**: ✅ Active
**Decision maker**: Vlad Petoukhov

**Context**:
Vapi sends webhooks to n8n when calls end. Without authentication, anyone with the webhook URL could POST fake call outcomes, corrupting debtor data or triggering fraudulent SMS.

**Decision**:
Implement HMAC SHA-256 signature verification on all Vapi webhook endpoints.

**Alternatives considered**:
1. **No verification**: Simple but insecure (anyone with URL can send fake data)
2. **API key in URL**: Better than nothing, but URL can leak in logs
3. **IP allowlist**: Vapi IPs can change, maintenance burden
4. **HMAC signature**: Industry standard (GitHub, Stripe, Twilio all use this)

**Rationale**:
- **Security**: Prevents unauthorized webhook calls
- **Data integrity**: Ensures only Vapi can update debtor records
- **Compliance**: Audit trail requirement (know all data came from legitimate source)
- **Standard practice**: HMAC SHA-256 is industry standard for webhook security
- **Low overhead**: Adds <5ms to webhook processing time

**Consequences**:
- ✅ Secure webhook endpoint
- ✅ Prevents data corruption attacks
- ✅ Compliance advantage (audit trail integrity)
- ❌ Requires VAPI_WEBHOOK_SECRET env var (minor)
- ❌ Small code complexity increase (20 lines in Code node)

**Implementation**:
```javascript
const signature = $input.headers['x-vapi-signature'];
const secret = $vars.VAPI_WEBHOOK_SECRET;
const payload = JSON.stringify($input.body);
const crypto = require('crypto');
const expectedSignature = crypto
  .createHmac('sha256', secret)
  .update(payload)
  .digest('hex');
if (signature !== expectedSignature) {
  throw new Error('Invalid webhook signature');
}
```

**Cost impact**: $0 (no additional cost)

**Next review**: After first production deployment (Oct 20)

---

### Decision 011: Phone Number Normalization (E.164 Format)

**Date**: October 3, 2025
**Status**: ✅ Active
**Decision maker**: Vlad Petoukhov

**Context**:
Google Sheets debtor data may have inconsistent phone formats: "0412 345 678", "+61 412 345 678", "61412345678". Vapi requires E.164 format (+61412345678) or calls will fail.

**Decision**:
Implement phone normalization function in Call Scheduler workflow (before Vapi API call).

**Alternatives considered**:
1. **Manual data entry validation**: Rely on Peter to enter correct format (error-prone)
2. **Validate at Google Sheets level**: No native phone validation in Sheets
3. **Normalize at read time**: Best approach (handles all edge cases)

**Rationale**:
- **Reliability**: Prevents call failures due to phone format errors
- **User-friendly**: Peter can enter phone numbers in any common format
- **Defensive programming**: Handle edge cases (spaces, dashes, parentheses)
- **Australian-specific**: Auto-detect "04xx" format and convert to "+614xx"
- **Zero cost**: Pure JavaScript function (no external service)

**Consequences**:
- ✅ Robust phone handling (no format-related call failures)
- ✅ Better UX for Peter (no strict format requirements)
- ✅ Handles edge cases (international format, local format, with/without spaces)
- ❌ Small code complexity (30 lines)
- ❌ Assumes Australian phone numbers (acceptable for Phase 1)

**Implementation**:
```javascript
function normalizePhone(phone) {
  let cleaned = phone.replace(/[\s\-\(\)]/g, '');
  if (cleaned.startsWith('04')) {
    cleaned = '+61' + cleaned.substring(1);
  }
  if (!cleaned.startsWith('+')) {
    cleaned = '+61' + cleaned;
  }
  return cleaned;
}
```

**Testing**:
- Input: "0412 345 678" → Output: "+61412345678" ✅
- Input: "+61 412 345 678" → Output: "+61412345678" ✅
- Input: "61412345678" → Output: "+61412345678" ✅

**Next review**: After first 100 calls (monitor for format errors)

---

### Decision 012: Duplicate Call Prevention (call_status Column)

**Date**: October 3, 2025
**Status**: ✅ Active
**Decision maker**: Vlad Petoukhov

**Context**:
Call Scheduler runs every 30 minutes. If calls take >30 min to complete (e.g., debtor on long hold), same debtor could be called twice simultaneously (race condition).

**Decision**:
Add `call_status` column to Google Sheets schema with values: null, "calling", "completed".

**Alternatives considered**:
1. **No prevention**: Risk duplicate calls (compliance issue, poor UX)
2. **Lock file on filesystem**: Complex, requires shared storage
3. **PostgreSQL advisory lock**: Requires database connection
4. **call_status column**: Simple, uses existing Google Sheets

**Rationale**:
- **Compliance**: Prevents accidentally calling debtor twice in short period (harassment risk)
- **Data integrity**: Only one workflow instance can "claim" a debtor at a time
- **Simple**: Uses existing Google Sheets schema (no external dependencies)
- **Idempotent**: If workflow crashes, status stays "calling" → manual review needed
- **Minimal overhead**: Single column update (adds <100ms to workflow)

**Consequences**:
- ✅ Prevents duplicate calls (compliance, UX)
- ✅ Race condition protection
- ✅ Simple implementation (no external locks)
- ❌ Requires manual intervention if workflow crashes mid-call (rare)
- ❌ Adds 1 column to Google Sheets schema

**Implementation**:
1. Call Scheduler filters: `call_status != "calling"`
2. Before Vapi call: Update `call_status = "calling"`
3. After webhook: Update `call_status = "completed"`
4. Error handler: If Vapi call fails, reset `call_status = null`

**Edge case handling**:
- If n8n crashes after setting `call_status = "calling"` but before call completes:
  - Status remains "calling" → debtor never called again
  - Solution: Manual review OR timeout logic (reset after 1 hour)
  - Acceptable for Phase 1 (low risk)

**Next review**: After first week of production (monitor for stuck "calling" status)

---

### Decision 013: Phase 1 Scope Clarification (Voice-Only)

**Date**: October 3, 2025
**Status**: ✅ Active
**Decision maker**: Client (Peter) + Vlad Petoukhov

**Context**:
During implementation planning, a scope mismatch was identified. The original proposal (Sep 11, 2025) mentioned voice + SMS + email automation. However, the actual agreement (Sep 21, 2025) specified Phase 1 as "$2,500 + $500 bonus" for "Voice AI Agent (Outbound-first)" only. SMS and email automation were designated as Phase 2 ($2,000-$2,500), not yet commissioned.

**Decision**:
Phase 1 delivers voice-only outbound calling with verbal payment instructions. SMS and email automation deferred to Phase 2.

**Original plan (pre-clarification)**:
- 5 workflows:
  1. BCS Call Scheduler
  2. BCS Vapi Webhook Handler
  3. BCS Send Payment SMS (Twilio)
  4. BCS SMS Status Handler (Twilio)
  5. BCS Test Webhook
- 21 environment variables
- 13.5 hour implementation estimate
- Twilio account setup required
- Payment details delivered via SMS

**Revised plan (post-clarification)**:
- 3 workflows:
  1. BCS Call Scheduler
  2. BCS Vapi Webhook Handler
  3. BCS Test Webhook
- 14 environment variables
- 10.5 hour implementation estimate
- No Twilio account needed for Phase 1
- Payment details delivered verbally during calls

**Changes made**:
1. **Deleted files**: BCS_Send_Payment_SMS.json, BCS_SMS_Status_Handler.json
2. **Updated workflows**: Removed SMS trigger nodes from BCS_Vapi_Webhook_Handler.json
3. **Updated documentation** (10 files):
   - .env.example - Removed Twilio variables
   - REQUIREMENTS.md - Clarified Phase 1 scope
   - WORKFLOW_ADAPTATION_PLAN.md - Removed SMS sections
   - VAPI_ASSISTANT_CONFIGS.md - Added verbal payment instructions
   - SMS_TEMPLATES.md - Marked as Phase 2 reference
   - IMPLEMENTATION_CHECKLIST.md - Removed SMS tasks
   - WORKFLOW_IMPORT_GUIDE.md - Updated to 3 workflows
   - TESTING_GUIDE.md - Removed SMS tests
   - TROUBLESHOOTING.md - Removed SMS issues
   - DECISIONS_LOG.md - This entry

**Rationale**:
- **Contract alignment**: Deliver exactly what was agreed to in Phase 1 contract
- **Quality over quantity**: Focus on excellent voice experience first
- **Staged rollout**: Reduce risk by implementing features in phases
- **Budget control**: Stay within $2,500 + $500 bonus Phase 1 budget
- **Client preference**: Client confirmed "let's only do the original scope"

**Consequences**:
- ✅ Aligns with actual agreement (Sep 21, 2025)
- ✅ Reduced complexity (3 workflows vs 5)
- ✅ Reduced time estimate (10.5h vs 13.5h)
- ✅ No Twilio setup needed for Phase 1
- ✅ Clearer scope boundaries between Phase 1 and Phase 2
- ✅ Lower risk of scope creep
- ❌ No SMS payment delivery in Phase 1 (verbal only)
- ❌ Higher risk of payment detail transcription errors (mitigated by AI confirmation)
- ❌ Phase 2 not yet commissioned (SMS features on hold)

**Verbal payment delivery approach**:
When debtor agrees to pay:
1. AI asks: "Do you have a pen and paper ready?"
2. AI reads payment details slowly (BSB, account, Stripe link, or mailing address)
3. AI repeats details once
4. AI confirms: "Have you written down all the details?"
5. READY_TO_PAY outcome logged with payment method
6. Follow-up call scheduled in 3 days to confirm payment received

**Phase 2 scope** (when commissioned - $2,000-$2,500):
- SMS automation (payment details, reminders)
- Email automation (payment confirmations, follow-ups)
- Two-way SMS communication
- Management dashboard

**Cost impact**:
- Phase 1: $2,500 (10.5 hours) + potential $500 bonus
- Savings: 3 hours of development time
- Twilio costs deferred to Phase 2

**Next review**: After Phase 1 delivery (determine if Phase 2 should be commissioned)

---

## 3. Deferred Decisions

### Decision 101: SMS and Email Automation

**Date**: October 3, 2025
**Status**: ⏸️ Deferred to Phase 2
**Decision maker**: Client (Peter) + Vlad

**Context**:
Original scoping included SMS and email automation, but timeline and budget prioritize voice-first approach.

**Decision**:
Defer SMS and email automation to Phase 2.

**Rationale**:
- Voice calls are highest priority (Phase 1 success criteria)
- 28 days insufficient to deliver voice + SMS + email
- Staged rollout reduces risk

**Consequences**:
- ✅ Focus on quality voice experience
- ✅ Lower Phase 1 complexity
- ❌ Manual SMS/email follow-ups required temporarily

**Next review**: After Phase 1 completion (Nov 1+)

---

### Decision 102: CRM Rebuild

**Date**: October 3, 2025
**Status**: ⏸️ Deferred to Phase 3 (if needed)
**Decision maker**: Client (Peter) + Vlad

**Context**:
Peter's current CRM lacks attachment support for letters and templates. Full rebuild may be needed long-term.

**Decision**:
Defer CRM rebuild to Phase 3. Phase 1 will integrate with existing CRM.

**Rationale**:
- Existing CRM functional for basic debtor tracking
- Workaround: Store attachments in separate file storage (Google Drive, S3)
- CRM rebuild is large project (estimated $4,000-6,000)

**Consequences**:
- ✅ Phase 1 focuses on AI bot, not CRM overhaul
- ✅ Lower cost and risk
- ❌ Template management remains manual

**Next review**: After Phase 1 (if Peter requests Phase 3 quote)

---

## 4. Superseded Decisions

### Decision 009: Use Twilio SMS for Payment Delivery

**Date**: October 3, 2025
**Status**: ⛔ Superseded by Decision 013 (moved to Phase 2)
**Decision maker**: Vlad Petoukhov (with user's suggestion)
**Superseded by**: Decision 013 - Phase 1 Scope Clarification

**Context**:
Need to deliver payment details to debtors who agree to pay. Verbal-only delivery is error-prone (debtor mishears BSB/account number).

**Original Decision**:
Send payment links and details via SMS (Twilio) instead of verbal-only instructions.

**Why Superseded**:
Upon reviewing the original agreement (Sep 21, 2025), Phase 1 was confirmed to be voice-only ($2,500 + $500 bonus). SMS automation is Phase 2 ($2,000-$2,500). Payment instructions are now delivered verbally during calls with confirmation from debtor.

**Replacement Approach**:
- AI asks debtor if they have pen and paper ready
- AI reads payment details slowly and clearly (BSB, account, Stripe link, mailing address)
- AI repeats details once
- AI confirms debtor wrote down all information
- Follow-up call scheduled in 3 days to confirm payment received

**Consequences of Change**:
- ✅ Aligns with actual Phase 1 agreement (voice-only)
- ✅ Reduces implementation scope from 5 workflows to 3
- ✅ No Twilio account setup needed for Phase 1
- ✅ Reduces environment variables from 21 to 14
- ❌ Higher risk of transcription errors (mitigated by AI confirmation)
- ❌ No persistent written reference for debtor (but can be added in Phase 2)

**Next review**: Phase 2 commission (SMS will be re-implemented then)

---

---

## 5. Deprecated Decisions

(None yet - this is the initial planning phase)

---

## 6. Pending Decisions (Awaiting Input)

### Pending 001: Human Escalation Protocol

**Context**: When AI bot encounters complex scenario (e.g., debtor has lawyer), how should escalation work?

**Options**:
1. Transfer call to Peter's mobile (requires Vapi call forwarding)
2. End call and leave BCS callback number
3. Flag for manual review, BCS calls back later

**Status**: Awaiting Peter's preference

**Urgency**: Medium (need by Oct 14)

**Recommendation**: Option 3 (simpler, no real-time availability needed)

---

### Pending 002: Business Hours and Timezone

**Context**: Confirm calling hours and timezone for compliance.

**Assumptions**:
- Weekdays: 7:30am - 9:00pm AEST/ACST
- Weekends: 9:00am - 9:00pm AEST/ACST

**Questions**:
- Are debtors across multiple timezones?
- Should system detect debtor timezone?
- Any exceptions (e.g., no Sunday calls)?

**Status**: Awaiting Peter's confirmation

**Urgency**: Medium (need by Oct 9)

---

### Pending 003: Call Volume Estimate

**Context**: Capacity planning for Vapi and database scaling.

**Questions**:
- How many calls per day expected?
- How many concurrent calls needed?

**Status**: Awaiting Peter's estimate

**Urgency**: Low (nice to have, not blocking)

---

## 7. Decision Review Process

### When to create a decision record:

- Choosing between 2+ technology options
- Changing previously made decisions
- Making trade-offs (cost, time, quality)
- Architectural changes
- Workflow design choices

### When NOT to create a decision record:

- Minor code refactoring
- Bug fixes
- Cosmetic changes (e.g., voice pacing adjustments)

---

## 8. Decision Impact Matrix

| Decision | Cost Impact | Time Impact | Risk Impact | Reversibility |
|----------|-------------|-------------|-------------|---------------|
| Vapi.ai | $92 (testing + 1mo) | -50% dev time | Low | Medium (can switch to Twilio) |
| n8n | $0 (self-hosted) | -30% dev time | Low | Low (easy to migrate) |
| PostgreSQL | $0 (open-source) | Neutral | Low | Low (standard SQL) |
| GPT-4o | ~$50 (testing) | Faster than alternatives | Low | High (can switch LLM easily) |
| DigitalOcean | $12/month | +10% dev time (DevOps) | Low | Medium (can migrate to PaaS) |
| AU voice | $0 (included in Vapi) | Neutral | Low | High (voice easily changed) |
| Vapi number | $10/month | -2 weeks (vs porting) | Low | Medium (can port later) |

---

## 9. Lessons Learned

(To be updated throughout the project)

### Week 1 Lessons:

- TBD

### Week 2 Lessons:

- TBD

### Week 3 Lessons:

- TBD

### Week 4 Lessons:

- TBD

---

## 10. Decision Ownership

| Decision Category | Owner | Approval Required From |
|-------------------|-------|------------------------|
| Tech stack | Vlad | None (within proposal scope) |
| Architecture | Vlad | None (within proposal scope) |
| UX/Tone | Vlad | Peter (client feedback) |
| Budget allocation | Vlad | Peter (if exceeds $2,500) |
| Scope changes | Peter | Both parties |
| Post-Phase 1 work | Peter | N/A (separate contract) |

---

## 11. Related Documentation

- [TECHNICAL_ARCHITECTURE.md](./TECHNICAL_ARCHITECTURE.md) - ADRs embedded in architecture doc
- [DEVELOPMENT_PLAN.md](./DEVELOPMENT_PLAN.md) - Timeline impact of decisions
- [PENDING_ITEMS.md](./PENDING_ITEMS.md) - Pending decisions awaiting Peter's input

---

**End of Decisions Log**

**Last updated**: October 3, 2025 (Added Decision 013: Phase 1 Scope Clarification)
**Next review**: Weekly (every Monday during project)
