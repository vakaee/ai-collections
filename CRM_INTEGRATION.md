# CRM Integration (Interim: Google Sheets)

## Document Information

**Project**: AI Debt Collection Bot - Phase 1
**Version**: 3.0 (Updated to 23-column schema with call tracking and compliance fields)
**Last Updated**: October 3, 2025
**Status**: ⚠️ **STOPGAP SOLUTION** - Will migrate to real CRM later

---

## 1. Overview

**Decision**: Use Google Sheets as interim CRM for Phase 1 to avoid waiting for [CLIENT]'s database access.

**Rationale**:
- Zero setup time (no waiting for credentials)
- Easy for [CLIENT] to view/edit/export data
- Native n8n integration (Google Sheets node)
- Simple migration path to real CRM later

**Migration plan**: Once [CLIENT] provides CRM access, swap Google Sheets node with PostgreSQL/MySQL node in n8n workflows. Data can be exported to CSV and imported to real CRM.

---

## 2. Google Sheets Setup

### 2.1 Create Spreadsheet

**Sheet Name**: "BCS_Debtors"

**Google Drive Location**: Share with [CLIENT] (edit access)

**Access**:
- Peter: Owner (can edit debtor data manually)
- n8n service account: Editor (read/write via API)

---

### 2.2 Column Schema

**Total: 23 columns** (updated from 18 to include call tracking and compliance fields)

| Column | Data Type | Example | Purpose |
|--------|-----------|---------|---------|
| **debtor_id** | Number | 1, 2, 3... | Unique identifier (auto-increment) |
| **name** | Text | "John Smith" or "XYZ Pty Ltd" | Debtor name |
| **phone** | Text | "+61412345678" | Phone number (E.164 format) |
| **email** | Text | "john@example.com" | Email address |
| **debtor_type** | Text | "consumer" or "commercial" | Routes to appropriate script |
| **amount** | Number | 1500.00 | Debt amount (dollars) |
| **creditor** | Text | "ABC Company" | Creditor name |
| **invoice_number** | Text | "INV-2024-001" | Invoice/account reference |
| **debt_date** | Date | 2024-06-01 | Debt origin date |
| **dob** | Date | 1980-05-15 | Date of birth (consumer verification) |
| **abn** | Text | "12345678901" | ABN (commercial debtors only) |
| **timezone** | Text | "AEST" | Debtor timezone (AEST/ACST/AWST) for business hours compliance |
| **call_status** | Text | "calling" | null/"calling"/"completed" - prevents duplicate calls |
| **call_id** | Text | "call_xxx" | Vapi call ID for tracking |
| **last_call_date** | Timestamp | 2025-10-05 10:30 | Last call attempt |
| **last_outcome** | Text | "PROMISE_TO_PAY" | Most recent outcome |
| **next_action** | Text | "SCHEDULE_CALL" | What to do next |
| **next_call_date** | Timestamp | 2025-10-11 14:00 | Scheduled next call |
| **attempt_number** | Number | 1, 2, or 3 | Call progression (1st/2nd/3rd) |
| **notes** | Text | "Promised to pay by Oct 10" | Free-form notes |
| **assigned_to_ai** | Boolean | TRUE / FALSE | Include in AI call queue? |
| **payment_status** | Text | "unpaid", "paid", "disputed" | Current status |
| **do_not_call** | Boolean | FALSE | Compliance flag - debtor requested no contact |
| **recording_url** | Text | "https://vapi..." | Vapi call recording link |

---

### 2.3 Sample Data (for testing)

Sample data with 23 columns (scroll horizontally to see all fields):

| debtor_id | name | phone | email | debtor_type | amount | creditor | invoice_number | debt_date | dob | abn | timezone | call_status | call_id | last_call_date | last_outcome | next_action | next_call_date | attempt_number | notes | assigned_to_ai | payment_status | do_not_call | recording_url |
|-----------|------|-------|-------|-------------|--------|----------|----------------|-----------|-----|-----|----------|-------------|---------|----------------|--------------|-------------|----------------|----------------|-------|----------------|----------------|-------------|---------------|
| 1 | John Smith | +61412345678 | john@example.com | consumer | 1500.00 | ABC Company | INV-001 | 2024-06-01 | 1980-05-15 | | AEST | | | | | SCHEDULE_CALL | 2025-10-05 10:00 | 1 | | TRUE | unpaid | FALSE | |
| 2 | XYZ Pty Ltd | +61498765432 | accounts@xyz.com.au | commercial | 5000.00 | DEF Corp | INV-002 | 2024-07-15 | | 12345678901 | ACST | | | | | SCHEDULE_CALL | 2025-10-05 11:00 | 1 | | TRUE | unpaid | FALSE | |
| 3 | Jane Doe | +61423456789 | jane@example.com | consumer | 800.00 | GHI Services | INV-003 | 2024-08-01 | 1975-03-22 | | AEST | completed | call_abc123 | 2025-10-03 15:30 | NO_ANSWER | SCHEDULE_CALL | 2025-10-03 19:30 | 1 | [03/10/2025 15:30] NO_ANSWER - Left voicemail | TRUE | unpaid | FALSE | https://vapi.ai/recording/abc123 |

---

## 3. n8n Integration

### 3.1 Google Sheets Node Configuration

**Authentication**:
1. Create Google Cloud project
2. Enable Google Sheets API
3. Create service account
4. Download service account JSON key
5. Add to n8n credentials

**OR**: Use OAuth (simpler for single user)

---

### 3.2 Read Operations (Workflow 1: Call Scheduler)

**n8n Google Sheets node** - "Read Rows"

**Configuration**:
- **Document**: BCS_Debtors (spreadsheet ID)
- **Sheet**: Sheet1
- **Range**: A2:W1000 (23 columns, up to 1000 rows)
- **Options**: Return all matching rows

**Filter logic** (Code node after read):
```javascript
// Filter for rows where assigned_to_ai=TRUE and payment_status=unpaid
// and next_call_date is now or past

const now = new Date();
const rows = $input.all();

const filteredRows = rows.filter(row => {
  const data = row.json;

  // Check if assigned to AI
  if (data.assigned_to_ai !== 'TRUE') return false;

  // Check if unpaid
  if (data.payment_status !== 'unpaid') return false;

  // Check if scheduled for now or past
  const nextCallDate = new Date(data.next_call_date);
  if (nextCallDate > now) return false;

  return true;
});

return filteredRows;
```

---

### 3.3 Write Operations (Workflow 2: Webhook Handler)

**n8n Google Sheets node** - "Update Row"

**Configuration**:
- **Document**: BCS_Debtors
- **Sheet**: Sheet1
- **Row Number**: `{{$node['Find Row'].json.rowNumber}}`
- **Columns**: Map n8n variables to Google Sheets columns

**Example update after call**:
```javascript
// Code node to prepare update data
const outcome = $input.item.json.outcome;
const notes = $input.item.json.notes;
const debtorId = $input.item.json.debtor_id;

// Calculate next call date based on outcome
let nextCallDate = null;
let nextAction = null;

switch(outcome) {
  case 'PROMISE_TO_PAY':
    // Schedule follow-up day after promised date (extract from notes)
    nextAction = 'SCHEDULE_CALL';
    nextCallDate = new Date();
    nextCallDate.setDate(nextCallDate.getDate() + 7);
    break;
  case 'NO_ANSWER':
    // Retry in 2 hours
    nextAction = 'SCHEDULE_CALL';
    nextCallDate = new Date();
    nextCallDate.setHours(nextCallDate.getHours() + 2);
    break;
  case 'DISPUTE_RAISED':
  case 'HARDSHIP_CLAIMED':
    nextAction = 'MANUAL_REVIEW';
    break;
  default:
    nextAction = 'MANUAL_REVIEW';
}

return {
  last_call_date: new Date().toISOString(),
  last_outcome: outcome,
  next_action: nextAction,
  next_call_date: nextCallDate ? nextCallDate.toISOString() : '',
  attempt_number: $input.item.json.attempt_number + 1,
  notes: `${$input.item.json.existing_notes}\n[${new Date().toLocaleString()}] ${outcome}: ${notes}`
};
```

**Google Sheets node** (updated column positions for 23-column schema):
- **Column M** (call_status): `{{$node['Prepare Update'].json.call_status}}`
- **Column N** (call_id): `{{$node['Prepare Update'].json.call_id}}`
- **Column O** (last_call_date): `{{$node['Prepare Update'].json.last_call_date}}`
- **Column P** (last_outcome): `{{$node['Prepare Update'].json.last_outcome}}`
- **Column Q** (next_action): `{{$node['Prepare Update'].json.next_action}}`
- **Column R** (next_call_date): `{{$node['Prepare Update'].json.next_call_date}}`
- **Column S** (attempt_number): `{{$node['Prepare Update'].json.attempt_number}}`
- **Column T** (notes): `{{$node['Prepare Update'].json.notes}}`
- **Column U** (assigned_to_ai): `{{$node['Prepare Update'].json.assigned_to_ai}}`
- **Column V** (payment_status): `{{$node['Prepare Update'].json.payment_status}}`
- **Column W** (do_not_call): `{{$node['Prepare Update'].json.do_not_call}}`
- **Column X** (recording_url): `{{$node['Prepare Update'].json.recording_url}}`

---

## 4. Migration Plan (Future)

### 4.1 When [CLIENT] Provides Real CRM Access

**Step 1**: Export Google Sheets to CSV
- Download BCS_Debtors sheet
- Save as `bcs_debtors_export.csv`

**Step 2**: Import to real CRM
- Use CRM's import function (or write SQL INSERT script)
- Map columns to CRM schema

**Step 3**: Update n8n workflows
- Replace "Google Sheets" nodes with "PostgreSQL" or "MySQL" nodes
- Update queries to match CRM schema
- Test thoroughly

**Step 4**: Deactivate Google Sheets
- Archive spreadsheet
- Remove n8n access

**Estimated migration time**: 2-4 hours

---

## 5. Advantages of Google Sheets Approach

✅ **Immediate start**: No waiting for credentials
✅ **Visibility**: [CLIENT] can see all data in real-time
✅ **Manual editing**: [CLIENT] can add/remove debtors easily
✅ **Export**: Easy to export to CSV for backup or migration
✅ **Collaboration**: Can share with BCS staff for review
✅ **Audit trail**: Google Sheets version history

---

## 6. Limitations

❌ **Scalability**: Performance degrades above ~1000 rows (acceptable for Phase 1)
❌ **No relational integrity**: No foreign keys or constraints
❌ **No transactions**: Risk of race conditions (low risk at this scale)
❌ **Manual schema**: No automatic validation

---

## 7. [CLIENT]'s Workflow

### 7.1 Adding New Debtor Accounts

**Manual entry**:
1. Open BCS_Debtors spreadsheet
2. Add new row with debtor details
3. Set `assigned_to_ai = TRUE`
4. Set `payment_status = unpaid`
5. Set `next_call_date` to desired time
6. AI bot will pick up on next cron run (every 30 minutes)

**Bulk import**:
1. Prepare CSV with debtor data
2. Copy/paste into Google Sheets
3. Ensure columns match schema

---

### 7.2 Reviewing Call Outcomes

**Check outcomes**:
1. Open BCS_Debtors spreadsheet
2. Sort by `last_call_date` (newest first)
3. Review `last_outcome` and `notes` columns

**Flag for manual action**:
- If `next_action = MANUAL_REVIEW`, [CLIENT] reviews and decides next steps

---

### 7.3 Removing Debtors from AI Queue

**Pause AI calls**:
- Set `assigned_to_ai = FALSE`

**Mark as paid**:
- Set `payment_status = paid`

**Delete row**:
- Right-click row → Delete row (or move to archive sheet)

---

## 8. Security Considerations

**Access control**:
- [CLIENT] owns spreadsheet (full control)
- n8n service account has Editor role (read/write only)
- No public sharing (private to [CLIENT] + n8n)

**Data privacy**:
- Spreadsheet contains PII (names, DOB, phone numbers)
- Ensure Google account has 2FA enabled
- Do not share link publicly

**Backup**:
- Google Sheets auto-saves
- Version history available (File → Version history)
- [CLIENT] should download CSV backup weekly

---

## 9. Testing

### 9.1 Create Test Debtors

Add 3-5 test rows:
- Use fake names like "Test Debtor 1"
- Use test phone numbers (your own or colleagues)
- Set `assigned_to_ai = TRUE`
- Set `next_call_date` to current time + 5 minutes

**Test scenarios**:
1. AI calls test number
2. You answer and interact with bot
3. Check if outcome logged correctly in Google Sheets
4. Verify next_call_date updated

---

### 9.2 Validation

**After first test call**:
- [ ] `last_call_date` populated
- [ ] `last_outcome` matches actual outcome
- [ ] `next_action` appropriate
- [ ] `attempt_number` incremented
- [ ] `notes` contain summary

---

## 10. Related Documentation

- [TECHNICAL_ARCHITECTURE.md](./TECHNICAL_ARCHITECTURE.md) - Google Sheets schema and n8n integration
- [DEVELOPMENT_PLAN.md](./DEVELOPMENT_PLAN.md) - Week 1 setup includes creating Google Sheet
- [DECISIONS_LOG.md](./DECISIONS_LOG.md) - ADR-006: Google Sheets decision

---

## 11. Future: Real CRM Integration

**Once [CLIENT] provides CRM access**, refer to original CRM_INTEGRATION.md draft for:
- PostgreSQL/MySQL connection setup
- Direct database queries
- Schema mapping
- Migration instructions

**File location**: `CRM_INTEGRATION_ORIGINAL.md` (to be created when needed)

---

**End of CRM Integration Document (Interim: Google Sheets)**

**Status**: STOPGAP SOLUTION - Production-ready for Phase 1, migrate to real CRM post-launch
**Last Updated**: October 3, 2025
