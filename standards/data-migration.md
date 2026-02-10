# STD-003: Data Migration Standard

**Standard ID:** STD-003
**Version:** 1.0
**Status:** Active
**Effective Date:** 2026-02-06

---

## 1. Purpose

This standard establishes requirements and procedures for migrating data into Salesforce orgs. It ensures data integrity, prevents unintended side effects (such as email notifications), and provides a consistent, repeatable process for all data migration activities.

## 2. Scope

This standard applies to:
- Initial data loads into new orgs
- Data migrations during org mergers/consolidations
- Bulk data imports from external systems
- Historical data archival imports
- Sandbox data refreshes with production data
- Any bulk record creation/update operation exceeding 1,000 records

## 3. Pre-Migration Checklist

**Execution Order:** Complete these steps in the order listed below. Email deliverability MUST be disabled first (3.1) before configuring automation bypass (3.3) to prevent any emails from being sent during the bypass configuration process itself.

| Priority | Section | Rationale |
|----------|---------|-----------|
| 1st | 3.1 Email Deliverability | Prevents emails immediately; protects against early trigger fires |
| 2nd | 3.2 Field-Level Security | Ensures data loader user can write to all fields |
| 3rd | 3.3 Automation Bypass | Safe to configure after emails disabled |

### 3.1 Email Deliverability

**CRITICAL: Disable email deliverability before any data migration.**

| Step | Action | Location |
|------|--------|----------|
| 1 | Navigate to Setup → Email → Deliverability | Setup Menu |
| 2 | Set Access Level to **"No Access"** | Deliverability Settings |
| 3 | Document the original setting | Migration Log |
| 4 | Confirm change is saved | Verify in Setup |

**Why:** Migrated records can trigger:
- Workflow email alerts
- Process Builder email actions
- Flow email actions
- Apex-triggered emails
- Assignment rule notifications

**Restore:** Set back to original level (typically "All Email") after migration and validation.

### 3.2 Field-Level Security (FLS)

Ensure the data migration user/profile has **Edit access** to all target fields.

**Verification Steps:**

| Step | Action |
|------|--------|
| 1 | Identify all fields being populated in the migration |
| 2 | Check FLS for the migration user's profile |
| 3 | Grant Edit access to any restricted fields |
| 4 | For managed package fields, verify accessibility |
| 5 | Document any temporary FLS changes for rollback |

**Common Issues:**
- Master-Detail relationship fields (require specific handling)
- Formula fields (cannot be directly populated)
- Roll-up summary fields (calculated, not loadable)
- Encrypted fields (require special permissions)
- Fields restricted by validation rules

**Profile/Permission Set Checklist:**

```
[ ] All standard fields accessible
[ ] All custom fields accessible
[ ] Object-level Create permission granted
[ ] Object-level Edit permission granted (for upserts)
[ ] "Modify All Data" or object-specific permissions as needed
```

### 3.3 Automation Bypass

Temporarily disable automations that should not fire during migration.

| Automation Type | Bypass Method |
|-----------------|---------------|
| **Validation Rules** | Deactivate or add bypass condition |
| **Workflow Rules** | Deactivate |
| **Process Builder** | Deactivate |
| **Flows** | Deactivate or add entry condition |
| **Apex Triggers** | Add bypass flag check (see Section 4.2) |
| **Assignment Rules** | Uncheck "Assign using active assignment rules" in Data Loader |
| **Auto-Response Rules** | Deactivate |
| **Escalation Rules** | Deactivate |

**Bypass Flag Pattern for Triggers:**

```apex
// In Custom Settings or Custom Metadata
// Hierarchy Custom Setting: Automation_Settings__c
// Field: Bypass_Triggers__c (Checkbox)

// In Trigger Handler
public class AccountTriggerHandler {

    public static void handleBeforeInsert(List<Account> newAccounts) {
        // Check bypass flag
        if (Automation_Settings__c.getInstance().Bypass_Triggers__c) {
            return;
        }

        // Normal trigger logic
        // ...
    }
}
```

### 3.4 Sharing Rules and Record Access

| Consideration | Action |
|---------------|--------|
| OWD Settings | Document current settings; may need temporary adjustment |
| Sharing Rules | Note if records need specific sharing post-migration |
| Teams/Territories | Plan for assignment after migration |
| Manual Shares | Cannot be migrated; must be recreated |

### 3.5 Data Volume Assessment

Before migration, assess the data volume impact:

| Metric | Check |
|--------|-------|
| Current record count | Query: `SELECT COUNT() FROM Object__c` |
| Records to migrate | Count source records |
| Storage impact | Estimate: Records × ~2KB per record |
| Current storage usage | Setup → Company Information → Data Storage |
| Available storage | Ensure sufficient headroom (recommend 20% buffer) |

**Storage Estimation Formula:**

```
Estimated Storage (MB) = (Number of Records × 2KB) / 1024

Example:
- 500,000 records × 2KB = 1,000,000 KB
- 1,000,000 KB / 1024 = ~977 MB
```

## 4. Data Preparation

### 4.1 Data Quality Requirements

| Requirement | Validation |
|-------------|------------|
| Required fields populated | No blanks in required columns |
| Picklist values valid | Values exist in picklist definition |
| Lookup references valid | Referenced records exist |
| Date formats correct | ISO 8601 or Salesforce format |
| Number formats correct | No currency symbols or commas |
| Text length within limits | Check field length limits |
| No duplicate external IDs | Unique values for upsert keys |
| Character encoding | UTF-8 recommended |

### 4.2 External ID Strategy

For upserts and relationship mapping, establish external IDs:

| Scenario | Approach |
|----------|----------|
| New org, no existing data | Use source system IDs as External ID |
| Existing data, need to merge | Create composite key or use existing External ID |
| Parent-child relationships | Load parents first, map External IDs to children |

**External ID Field Requirements:**
- Must be marked as External ID in field definition
- Should be unique (recommend "Unique" checkbox)
- Case-sensitive option available

### 4.3 Data Transformation Rules

Document all transformation rules applied:

```
| Source Field | Target Field | Transformation |
|--------------|--------------|----------------|
| Status | Status__c | Map: "A" → "Active", "I" → "Inactive" |
| Created | CreatedDate | Format: MM/DD/YYYY → YYYY-MM-DD |
| Amount | Amount__c | Remove "$" and "," |
| Phone | Phone | Format to (XXX) XXX-XXXX |
```

## 5. Migration Execution

### 5.1 Load Order

Load objects in dependency order:

```
1. Reference/Lookup Objects (no dependencies)
   └── Account (if independent)
   └── Product2
   └── Pricebook2

2. Primary Objects
   └── Account (with parent accounts)
   └── Contact
   └── Opportunity

3. Child/Junction Objects
   └── OpportunityLineItem
   └── OpportunityContactRole
   └── AccountTeamMember

4. Activity Objects
   └── Task
   └── Event

5. Attachment/File Objects
   └── ContentVersion
   └── Attachment (legacy)
```

### 5.2 Batch Size Recommendations

| Object Complexity | Recommended Batch Size |
|-------------------|------------------------|
| Simple (few fields, no triggers) | 10,000 |
| Moderate (triggers, some automation) | 2,000 - 5,000 |
| Complex (heavy automation, integrations) | 200 - 500 |
| With Attachments/Files | 1 - 10 |

### 5.3 Data Loader Settings

**Recommended Settings:**

```
Insert/Update Settings:
- Batch Size: Based on complexity (see above)
- Insert Null Values: TRUE (if intentionally clearing fields)
- Assignment Rule: Unchecked (unless specifically needed)
- Serial Mode: TRUE for complex loads (avoids lock contention)

CSV Settings:
- Write UTF-8 with BOM
- Read UTF-8 (or match source encoding)
```

### 5.4 Execution Log Template

Maintain a migration log:

```markdown
## Migration Log: [Object Name]

**Date:** YYYY-MM-DD
**Executed By:** [Name]
**Environment:** [Sandbox/Production]

### Pre-Migration
- [ ] Email deliverability: OFF (was: [original setting])
- [ ] FLS verified for all fields
- [ ] Triggers bypassed
- [ ] Validation rules: [Deactivated/Bypassed]
- [ ] Workflow rules: [Deactivated/N/A]
- [ ] Record count before: [X]

### Execution
- **Start Time:** HH:MM
- **End Time:** HH:MM
- **Total Records Attempted:** X
- **Success Count:** X
- **Error Count:** X
- **Error File:** [path/filename]

### Post-Migration
- [ ] Record count after: [X]
- [ ] Spot check completed
- [ ] Email deliverability: RESTORED
- [ ] Automations: REACTIVATED
- [ ] FLS changes: REVERTED
```

## 6. Post-Migration Validation

### 6.1 Record Count Verification

```sql
-- Before vs After comparison
SELECT COUNT() FROM [Object] WHERE CreatedDate = TODAY
SELECT COUNT() FROM [Object] WHERE LastModifiedDate = TODAY
```

### 6.2 Data Integrity Checks

| Check | Query/Method |
|-------|--------------|
| Orphan records | `SELECT Id FROM Child__c WHERE Parent__c = null` |
| Duplicate check | `SELECT Name, COUNT(Id) FROM Object GROUP BY Name HAVING COUNT(Id) > 1` |
| Required field blanks | `SELECT Id FROM Object WHERE Required_Field__c = null` |
| Invalid picklist values | Review validation rule errors |
| Lookup integrity | Verify parent records exist |

### 6.3 Spot Check Sample

Manually verify a random sample:

```
Sample Size Guidelines:
- < 1,000 records: 5% sample
- 1,000 - 10,000 records: 2% sample (min 50)
- > 10,000 records: 1% sample (min 100)
```

### 6.4 Functional Testing

| Test | Description |
|------|-------------|
| Record display | Open records in UI, verify all fields |
| Related lists | Check child records appear correctly |
| Reports | Run key reports, verify data appears |
| Lookups | Test lookup filters work correctly |
| Search | Verify records are searchable (may take time to index) |

## 7. Post-Migration Restoration

### 7.1 Restoration Checklist

Execute in this order after successful validation:

```
[ ] 1. Reactivate Validation Rules
[ ] 2. Reactivate Workflow Rules
[ ] 3. Reactivate Process Builders
[ ] 4. Reactivate Flows
[ ] 5. Remove Trigger bypass flags
[ ] 6. Revert FLS changes (if temporary)
[ ] 7. Restore Email Deliverability
[ ] 8. Re-enable Assignment Rules (if applicable)
[ ] 9. Re-enable Escalation Rules (if applicable)
```

### 7.2 Email Deliverability Restoration

**IMPORTANT: Only restore after confirming:**
- All data loaded successfully
- No pending retry batches
- Validation complete

| Step | Action |
|------|--------|
| 1 | Navigate to Setup → Email → Deliverability |
| 2 | Set Access Level back to **"All Email"** (or original setting) |
| 3 | Confirm no automated emails were queued during migration |

## 8. Rollback Procedures

### 8.1 When to Rollback

Consider rollback if:
- Error rate exceeds 5% of total records
- Critical data integrity issues discovered
- Duplicate records created
- Production system performance impacted

### 8.2 Rollback Methods

| Scenario | Method |
|----------|--------|
| Insert only (new records) | Mass delete using Data Loader with IDs from success file |
| Upsert (mixed) | Restore from backup, delete new records |
| Update only | Re-load original values from backup |

### 8.3 Backup Requirements

**Before any migration:**

```
[ ] Export current state of target object(s)
[ ] Include all fields being modified
[ ] Store backup securely with timestamp
[ ] Verify backup file integrity (record count, spot check)
```

## 9. Special Considerations

### 9.1 Large Data Volumes (LDV)

For migrations exceeding 1 million records:

| Consideration | Recommendation |
|---------------|----------------|
| Skinny tables | Request from Salesforce Support if needed |
| Indexes | Ensure External ID fields are indexed |
| Query optimization | Use selective queries, avoid full table scans |
| Off-hours execution | Schedule during low-usage periods |
| Bulk API | Use Bulk API 2.0 for best performance |

### 9.2 Attachments and Files

| Old Model (Attachments) | New Model (Files) |
|------------------------|-------------------|
| Attachment object | ContentVersion + ContentDocumentLink |
| ParentId relationship | ContentDocumentLink.LinkedEntityId |
| Body field (base64) | VersionData field |
| 25MB limit | 2GB limit |

**Recommendation:** Migrate to Files (ContentVersion) for new implementations.

### 9.3 Record Types

```
[ ] Verify Record Types exist in target org
[ ] Map source Record Type names/IDs to target
[ ] Ensure profile has access to required Record Types
[ ] Consider default Record Type for the migration user
```

### 9.4 Currency and Multi-Currency

```
[ ] Verify currency ISO codes match
[ ] Check dated exchange rates if using advanced currency management
[ ] Map currency fields correctly (CurrencyIsoCode)
```

## 10. Compliance Checklist

Before starting migration:

- [ ] Migration plan documented and approved
- [ ] Backup of existing data completed
- [ ] Email deliverability set to "No Access"
- [ ] All FLS permissions verified
- [ ] Automation bypass configured
- [ ] Test migration completed in sandbox
- [ ] Rollback plan documented
- [ ] Stakeholder communication sent

After migration:

- [ ] Record counts verified
- [ ] Data integrity checks passed
- [ ] Spot check completed
- [ ] All automations restored
- [ ] Email deliverability restored
- [ ] Migration log completed
- [ ] Backup files archived
- [ ] Stakeholder notification sent

---

**Document Owner:** Platform Team
**Next Review Date:** 2026-08-06
**Related Documents:**
- [Error Handling Standard](./error-handling.md)
- [Configuration Management Standard](./configuration-management.md)
