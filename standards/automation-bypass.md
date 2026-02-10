# STD-004: Automation Bypass Standard

**Standard ID:** STD-004
**Version:** 1.0
**Status:** Active
**Effective Date:** 2026-02-06

---

## 1. Purpose

This standard establishes a consistent, secure mechanism for bypassing Salesforce automations when necessary. It ensures that bypass capabilities are implemented uniformly across triggers, flows, validation rules, and other automation types while maintaining proper security controls and audit trails.

## 2. Scope

This standard applies to:
- Apex Triggers
- Validation Rules
- Flows (all types)
- Process Builders (legacy)
- Workflow Rules (legacy)
- Assignment Rules
- Escalation Rules
- Auto-Response Rules

## 3. When to Use Automation Bypass

### 3.1 Approved Use Cases

| Use Case | Description | Duration |
|----------|-------------|----------|
| Data Migration | Bulk loading historical data without triggering notifications or validations | Temporary |
| Data Correction | Admin fixing data issues that would otherwise fail validation | Temporary |
| Integration Sync | External system syncing data that shouldn't trigger downstream automations | Permanent (integration user) |
| Testing | Unit tests or QA scenarios requiring isolated behavior | Test context only |
| Batch Processing | Large batch jobs where automation would cause governor limit issues | Per-execution |
| Circular Reference Prevention | Preventing infinite loops between related automations | Permanent (code pattern) |

### 3.2 Prohibited Use Cases

| Prohibited Use | Why |
|----------------|-----|
| Bypassing security validations | Security rules must always execute |
| Permanent bypass for regular users | Creates data integrity risks |
| Bypassing audit trail creation | Compliance requirement |
| Avoiding error handling | Masks real issues |

## 4. Architecture Overview

### 4.1 Recommended Pattern: Hierarchy Custom Setting

Use a **Hierarchy Custom Setting** as the primary bypass mechanism:

```
Automation_Bypass__c (Hierarchy Custom Setting)
├── SetupOwnerId (Organization / Profile / User)
├── Bypass_Triggers__c (Checkbox)
├── Bypass_Validation_Rules__c (Checkbox)
├── Bypass_Flows__c (Checkbox)
├── Bypass_Workflow_Rules__c (Checkbox)
└── Bypass_All__c (Checkbox) - Master switch
```

**Why Hierarchy Custom Setting:**
- Per-user/profile granularity
- No SOQL query required (cached)
- Easily toggled in Setup UI
- Works in validation rule formulas (via `$Setup`)
- Can be set programmatically for integrations

### 4.2 Custom Setting Definition

**Object:** `Automation_Bypass__c`
**Type:** Hierarchy

| Field API Name | Type | Default | Description |
|----------------|------|---------|-------------|
| `Bypass_All__c` | Checkbox | FALSE | Master switch - bypasses everything |
| `Bypass_Triggers__c` | Checkbox | FALSE | Bypass all Apex triggers |
| `Bypass_Validation_Rules__c` | Checkbox | FALSE | Bypass validation rules |
| `Bypass_Flows__c` | Checkbox | FALSE | Bypass Flows |
| `Bypass_Workflow_Rules__c` | Checkbox | FALSE | Bypass Workflow Rules |
| `Bypass_Process_Builders__c` | Checkbox | FALSE | Bypass Process Builders |
| `Bypass_Assignment_Rules__c` | Checkbox | FALSE | Bypass Assignment Rules |

### 4.3 Supplementary: Object-Specific Bypasses

For granular control, add object-specific bypass Custom Metadata:

**Object:** `Trigger_Bypass__mdt`

| Field API Name | Type | Description |
|----------------|------|-------------|
| `Object_API_Name__c` | Text | Object to bypass (e.g., "Account") |
| `Bypass_Before_Insert__c` | Checkbox | Bypass before insert trigger |
| `Bypass_After_Insert__c` | Checkbox | Bypass after insert trigger |
| `Bypass_Before_Update__c` | Checkbox | Bypass before update trigger |
| `Bypass_After_Update__c` | Checkbox | Bypass after update trigger |
| `Bypass_Before_Delete__c` | Checkbox | Bypass before delete trigger |
| `Bypass_After_Delete__c` | Checkbox | Bypass after delete trigger |
| `Is_Active__c` | Checkbox | Enable/disable this bypass |

### 4.4 Salesforce Order of Execution and Bypass Points

Understanding where bypass checks apply in Salesforce's Order of Execution:

| Step | Automation Type | Bypass Check Location |
|------|----------------|----------------------|
| 1 | System validation rules | Cannot bypass (system-enforced) |
| 2 | **Before triggers** | Check `Bypass_Triggers__c` at trigger entry |
| 3 | **Custom validation rules** | Check `$Setup.Automation_Bypass__c.Bypass_Validation_Rules__c` in formula |
| 4 | Duplicate rules | Cannot bypass via Custom Setting |
| 5 | **After triggers** | Check `Bypass_Triggers__c` at trigger entry |
| 6 | Assignment rules | Use Data Loader checkbox or `Bypass_Assignment_Rules__c` |
| 7 | Auto-response rules | Deactivate or use bypass |
| 8 | **Workflow rules** | Check `$Setup` in workflow criteria (limited) |
| 9 | **Processes (Process Builder)** | Check `$Setup` in entry criteria |
| 10 | **Flows** | Check `$Setup` in entry conditions or Decision element |
| 11 | Escalation rules | Deactivate for bypass |
| 12 | Roll-up summary calculation | Cannot bypass |
| 13 | Criteria-based sharing | Cannot bypass |

**Key Insights:**
- Bypass checks in **before triggers** prevent record changes and downstream automations
- Bypassing **validation rules** still allows before/after triggers to run
- `Bypass_All__c = TRUE` is the safest for data migrations (stops everything bypassable)
- Some steps (duplicate rules, roll-up summaries) cannot be bypassed via Custom Settings

**Recommended Bypass Order for Data Migration:**
1. Set `Bypass_All__c = TRUE` on user/profile
2. Disable email deliverability (see STD-003)
3. Deactivate automations that cannot read Custom Settings
4. Perform migration
5. Re-enable in reverse order

## 5. Implementation Patterns

### 5.1 Bypass Utility Class

Create a centralized utility class for all bypass checks:

```apex
/**
 * Centralized bypass check utility
 * All automation should use this class to check bypass status
 */
public class BypassUtility {

    // Cache the setting to avoid repeated fetches within a transaction
    private static Automation_Bypass__c bypassSetting;

    /**
     * Get the current user's bypass settings
     * Uses hierarchy: User > Profile > Org Default
     */
    public static Automation_Bypass__c getBypassSettings() {
        if (bypassSetting == null) {
            bypassSetting = Automation_Bypass__c.getInstance();

            // If no setting exists, return empty (no bypass)
            if (bypassSetting == null) {
                bypassSetting = new Automation_Bypass__c();
            }
        }
        return bypassSetting;
    }

    /**
     * Check if all automations should be bypassed
     */
    public static Boolean bypassAll() {
        return getBypassSettings().Bypass_All__c == true;
    }

    /**
     * Check if triggers should be bypassed
     */
    public static Boolean bypassTriggers() {
        Automation_Bypass__c settings = getBypassSettings();
        return settings.Bypass_All__c == true ||
               settings.Bypass_Triggers__c == true;
    }

    /**
     * Check if validation rules should be bypassed
     */
    public static Boolean bypassValidationRules() {
        Automation_Bypass__c settings = getBypassSettings();
        return settings.Bypass_All__c == true ||
               settings.Bypass_Validation_Rules__c == true;
    }

    /**
     * Check if Flows should be bypassed
     */
    public static Boolean bypassFlows() {
        Automation_Bypass__c settings = getBypassSettings();
        return settings.Bypass_All__c == true ||
               settings.Bypass_Flows__c == true;
    }

    /**
     * Check if Workflow Rules should be bypassed
     */
    public static Boolean bypassWorkflowRules() {
        Automation_Bypass__c settings = getBypassSettings();
        return settings.Bypass_All__c == true ||
               settings.Bypass_Workflow_Rules__c == true;
    }

    /**
     * Check if Process Builders should be bypassed
     */
    public static Boolean bypassProcessBuilders() {
        Automation_Bypass__c settings = getBypassSettings();
        return settings.Bypass_All__c == true ||
               settings.Bypass_Process_Builders__c == true;
    }

    /**
     * Check if Assignment Rules should be bypassed
     */
    public static Boolean bypassAssignmentRules() {
        Automation_Bypass__c settings = getBypassSettings();
        return settings.Bypass_All__c == true ||
               settings.Bypass_Assignment_Rules__c == true;
    }

    /**
     * Clear the cached settings (use when settings change mid-transaction)
     */
    public static void clearCache() {
        bypassSetting = null;
    }

    /**
     * Programmatically enable bypass for current transaction
     * Useful for data migration or batch processing
     * Note: Does NOT persist - only affects current transaction
     */
    public static void enableBypassForTransaction() {
        bypassSetting = new Automation_Bypass__c(
            Bypass_All__c = true
        );
    }

    /**
     * Programmatically enable specific bypass for current transaction
     */
    public static void enableTriggerBypassForTransaction() {
        if (bypassSetting == null) {
            bypassSetting = new Automation_Bypass__c();
        }
        bypassSetting.Bypass_Triggers__c = true;
    }
}
```

### 5.2 Apex Trigger Implementation

**Standard Trigger Pattern:**

```apex
trigger AccountTrigger on Account (
    before insert, after insert,
    before update, after update,
    before delete, after delete,
    after undelete
) {
    // Check bypass at the start of every trigger
    if (BypassUtility.bypassTriggers()) {
        return;
    }

    // Delegate to handler
    AccountTriggerHandler handler = new AccountTriggerHandler();

    if (Trigger.isBefore) {
        if (Trigger.isInsert) handler.beforeInsert(Trigger.new);
        if (Trigger.isUpdate) handler.beforeUpdate(Trigger.old, Trigger.new, Trigger.oldMap, Trigger.newMap);
        if (Trigger.isDelete) handler.beforeDelete(Trigger.old, Trigger.oldMap);
    }

    if (Trigger.isAfter) {
        if (Trigger.isInsert) handler.afterInsert(Trigger.new, Trigger.newMap);
        if (Trigger.isUpdate) handler.afterUpdate(Trigger.old, Trigger.new, Trigger.oldMap, Trigger.newMap);
        if (Trigger.isDelete) handler.afterDelete(Trigger.old, Trigger.oldMap);
        if (Trigger.isUndelete) handler.afterUndelete(Trigger.new, Trigger.newMap);
    }
}
```

**Trigger Handler with Granular Bypass:**

```apex
public class AccountTriggerHandler {

    public void beforeInsert(List<Account> newAccounts) {
        // Each method can have additional bypass checks if needed
        if (shouldBypassMethod('beforeInsert')) {
            return;
        }

        validateAccounts(newAccounts);
        setDefaults(newAccounts);
    }

    public void afterInsert(List<Account> newAccounts, Map<Id, Account> newMap) {
        if (shouldBypassMethod('afterInsert')) {
            return;
        }

        createRelatedRecords(newAccounts);
        sendNotifications(newAccounts);
    }

    // ... other handlers

    /**
     * Check object-specific bypass from Custom Metadata
     */
    private Boolean shouldBypassMethod(String methodName) {
        Trigger_Bypass__mdt bypass = Trigger_Bypass__mdt.getInstance('Account_Bypass');

        if (bypass == null || !bypass.Is_Active__c) {
            return false;
        }

        switch on methodName {
            when 'beforeInsert' { return bypass.Bypass_Before_Insert__c; }
            when 'afterInsert' { return bypass.Bypass_After_Insert__c; }
            when 'beforeUpdate' { return bypass.Bypass_Before_Update__c; }
            when 'afterUpdate' { return bypass.Bypass_After_Update__c; }
            when 'beforeDelete' { return bypass.Bypass_Before_Delete__c; }
            when 'afterDelete' { return bypass.Bypass_After_Delete__c; }
            when else { return false; }
        }
    }
}
```

### 5.3 Validation Rule Implementation

Reference the bypass setting directly in validation rules using `$Setup`:

```
// Validation Rule: Account_Name_Required
// Error Condition Formula:

AND(
    NOT($Setup.Automation_Bypass__c.Bypass_All__c),
    NOT($Setup.Automation_Bypass__c.Bypass_Validation_Rules__c),
    ISBLANK(Name)
)
```

**Complex Validation with Bypass:**

```
// Validation Rule: Opportunity_Close_Date_Required
// Error Condition Formula:

AND(
    NOT($Setup.Automation_Bypass__c.Bypass_All__c),
    NOT($Setup.Automation_Bypass__c.Bypass_Validation_Rules__c),
    ISPICKVAL(StageName, 'Closed Won'),
    ISBLANK(CloseDate)
)
```

**Profile-Based Bypass Alternative:**

```
// Allow System Administrators to bypass
AND(
    $Profile.Name <> 'System Administrator',
    NOT($Setup.Automation_Bypass__c.Bypass_Validation_Rules__c),
    // Actual validation logic
    ISBLANK(Required_Field__c)
)
```

### 5.4 Flow Implementation

**Option 1: Entry Condition (Recommended for Record-Triggered Flows)**

In Flow Builder, add an entry condition:
```
{!$Setup.Automation_Bypass__c.Bypass_Flows__c} = FALSE
AND
{!$Setup.Automation_Bypass__c.Bypass_All__c} = FALSE
```

**Option 2: Decision Element (For complex flows)**

Add a Decision element at the start:

```
Decision: Check Bypass
├── Outcome: Bypass Active
│   Condition: {!$Setup.Automation_Bypass__c.Bypass_Flows__c} = TRUE
│            OR {!$Setup.Automation_Bypass__c.Bypass_All__c} = TRUE
│   → End Flow
│
└── Default Outcome: Continue
    → Normal Flow Logic
```

**Option 3: Invocable Apex for Complex Bypass Logic**

```apex
public class FlowBypassChecker {

    @InvocableMethod(
        label='Check Flow Bypass'
        description='Returns true if flows should be bypassed'
    )
    public static List<Boolean> checkBypass(List<FlowBypassRequest> requests) {
        List<Boolean> results = new List<Boolean>();

        for (FlowBypassRequest request : requests) {
            Boolean shouldBypass = BypassUtility.bypassFlows();

            // Additional custom logic if needed
            if (request.flowName != null) {
                shouldBypass = shouldBypass || isFlowSpecificallyBypassed(request.flowName);
            }

            results.add(shouldBypass);
        }

        return results;
    }

    private static Boolean isFlowSpecificallyBypassed(String flowName) {
        // Check Custom Metadata for flow-specific bypass
        Flow_Bypass__mdt bypass = Flow_Bypass__mdt.getInstance(flowName);
        return bypass != null && bypass.Is_Bypassed__c;
    }

    public class FlowBypassRequest {
        @InvocableVariable(label='Flow Name' description='Optional: specific flow to check')
        public String flowName;
    }
}
```

### 5.5 Process Builder Implementation (Legacy)

For existing Process Builders, add bypass criteria to the entry condition:

```
[Account].Id != null
AND
NOT($Setup.Automation_Bypass__c.Bypass_Process_Builders__c)
AND
NOT($Setup.Automation_Bypass__c.Bypass_All__c)
```

### 5.6 Workflow Rule Implementation (Legacy)

Add bypass check to the rule criteria formula:

```
AND(
    NOT($Setup.Automation_Bypass__c.Bypass_Workflow_Rules__c),
    NOT($Setup.Automation_Bypass__c.Bypass_All__c),
    -- Original workflow criteria --
    ISCHANGED(Status__c)
)
```

## 6. Integration User Configuration

### 6.1 Dedicated Integration User

Create a dedicated integration user with permanent bypass enabled:

| Setting | Value |
|---------|-------|
| Username | `integration@company.com.sandbox` |
| Profile | Integration User (custom profile) |
| License | Salesforce Integration |
| API Enabled | Yes |

### 6.2 Integration User Bypass Settings

In Setup → Custom Settings → Automation Bypass → Manage:

1. Click "New" next to the integration user
2. Set `Bypass_All__c = TRUE` or specific bypasses as needed
3. Save

**Alternative: Profile-Level Bypass**

If multiple integration users share a profile:
1. Click "New" at the Profile level
2. Select the Integration User profile
3. Configure bypass settings

### 6.3 Integration Apex Pattern

For API integrations using named credentials:

```apex
public class ExternalSystemIntegration {

    public void syncRecords(List<SObject> records) {
        // Enable bypass for this integration operation
        BypassUtility.enableBypassForTransaction();

        try {
            // Process records without triggering automations
            Database.insert(records, false);

        } finally {
            // Clear cache to restore normal behavior
            BypassUtility.clearCache();
        }
    }
}
```

## 7. Data Migration Bypass Pattern

### 7.1 Pre-Migration Setup (UI Method)

1. Navigate to Setup → Custom Settings → Automation Bypass
2. Click "Manage"
3. Create or edit the setting for the migration user
4. Enable all required bypasses
5. **Document the change with timestamp**

### 7.2 Apex-Based Migration Bypass

For Data Loader or Apex batch jobs:

```apex
public class DataMigrationBatch implements Database.Batchable<sObject> {

    public Database.QueryLocator start(Database.BatchableContext bc) {
        return Database.getQueryLocator('SELECT Id, Name FROM Migration_Source__c');
    }

    public void execute(Database.BatchableContext bc, List<Migration_Source__c> scope) {
        // Enable bypass for this batch chunk
        BypassUtility.enableBypassForTransaction();

        List<Account> accountsToInsert = new List<Account>();

        for (Migration_Source__c source : scope) {
            accountsToInsert.add(new Account(
                Name = source.Name
                // ... other mappings
            ));
        }

        // Insert without trigger/automation execution
        Database.insert(accountsToInsert, false);
    }

    public void finish(Database.BatchableContext bc) {
        // Log completion
        System.debug('Migration complete. Bypass was used for data loading.');
    }
}
```

## 8. Testing with Bypass

### 8.1 Test Class Pattern

```apex
@IsTest
private class AccountTriggerTest {

    @TestSetup
    static void setup() {
        // Create test data with bypass to avoid trigger logic during setup
        Automation_Bypass__c bypassSettings = new Automation_Bypass__c(
            SetupOwnerId = UserInfo.getUserId(),
            Bypass_Triggers__c = true
        );
        insert bypassSettings;

        // Insert test data without triggers firing
        insert new Account(Name = 'Test Account');

        // Remove bypass after setup
        delete bypassSettings;
    }

    @IsTest
    static void testTriggerLogic() {
        // Test WITH trigger logic enabled (normal operation)
        Account acc = [SELECT Id, Name FROM Account LIMIT 1];

        Test.startTest();
        acc.Name = 'Updated Name';
        update acc;
        Test.stopTest();

        // Assert trigger logic executed
        acc = [SELECT Id, Name, Trigger_Processed__c FROM Account WHERE Id = :acc.Id];
        System.assertEquals(true, acc.Trigger_Processed__c, 'Trigger should have processed');
    }

    @IsTest
    static void testBypassedBehavior() {
        // Test WITH bypass enabled
        Automation_Bypass__c bypassSettings = new Automation_Bypass__c(
            SetupOwnerId = UserInfo.getUserId(),
            Bypass_Triggers__c = true
        );
        insert bypassSettings;

        Test.startTest();
        Account acc = new Account(Name = 'Bypassed Account');
        insert acc;
        Test.stopTest();

        // Assert trigger logic did NOT execute
        acc = [SELECT Id, Trigger_Processed__c FROM Account WHERE Id = :acc.Id];
        System.assertEquals(false, acc.Trigger_Processed__c, 'Trigger should NOT have processed');
    }
}
```

### 8.2 Mocking Bypass in Tests

```apex
@IsTest
private class BypassUtilityTest {

    @IsTest
    static void testBypassEnabled() {
        // Setup: Enable bypass via custom setting
        insert new Automation_Bypass__c(
            SetupOwnerId = UserInfo.getUserId(),
            Bypass_All__c = true
        );

        // Clear any cached values
        BypassUtility.clearCache();

        Test.startTest();
        Boolean result = BypassUtility.bypassAll();
        Test.stopTest();

        System.assertEquals(true, result, 'Bypass should be enabled');
    }

    @IsTest
    static void testBypassDisabled() {
        // No custom setting = no bypass
        BypassUtility.clearCache();

        Test.startTest();
        Boolean result = BypassUtility.bypassAll();
        Test.stopTest();

        System.assertEquals(false, result, 'Bypass should be disabled by default');
    }
}
```

## 9. Security Considerations

### 9.1 Access Control

| Who | Can Enable Bypass | Method |
|-----|-------------------|--------|
| System Administrator | Yes | Setup UI |
| Integration User | Pre-configured | Profile/User setting |
| Developer | Test context only | Test class code |
| Regular User | No | N/A |

### 9.2 Required Permission Set

Create a permission set for bypass management:

**Permission Set:** `Automation_Bypass_Manager`

| Permission | Setting |
|------------|---------|
| Custom Setting: Automation_Bypass__c | Read, Edit |
| "Customize Application" | Required for Setup access |

### 9.3 Audit Trail

Log all bypass usage for compliance:

```apex
public class BypassAuditLogger {

    public static void logBypassUsage(String operation, String reason) {
        Bypass_Audit_Log__c log = new Bypass_Audit_Log__c(
            User__c = UserInfo.getUserId(),
            Operation__c = operation,
            Reason__c = reason,
            Timestamp__c = DateTime.now(),
            Bypass_Settings__c = JSON.serialize(BypassUtility.getBypassSettings())
        );

        // Insert without triggering more automations
        Database.insert(log, false);
    }
}
```

### 9.4 Never Bypass These

| Automation Type | Reason |
|-----------------|--------|
| Security validation rules | Data security requirement |
| Audit field updates | Compliance requirement |
| Encryption triggers | Data protection |
| Sharing rule calculations | Security model |

## 10. Troubleshooting

### 10.1 Bypass Not Working

| Symptom | Check |
|---------|-------|
| Trigger still firing | Is `Bypass_Triggers__c` TRUE for the running user? |
| Validation still failing | Is bypass referenced in the validation rule formula? |
| Flow still running | Is entry condition correctly checking `$Setup`? |
| Wrong user context | Check `SetupOwnerId` on the Custom Setting record |

### 10.2 Debug Query

```apex
// Check current user's bypass settings
Automation_Bypass__c settings = Automation_Bypass__c.getInstance();
System.debug('Bypass All: ' + settings.Bypass_All__c);
System.debug('Bypass Triggers: ' + settings.Bypass_Triggers__c);
System.debug('SetupOwnerId: ' + settings.SetupOwnerId);

// Check if user vs profile vs org level
System.debug('User ID: ' + UserInfo.getUserId());
System.debug('Profile ID: ' + UserInfo.getProfileId());
```

## 11. Compliance Checklist

Before using automation bypass:

- [ ] Documented business justification
- [ ] Approved by appropriate stakeholder
- [ ] Bypass setting created at correct level (User/Profile/Org)
- [ ] All affected automations include bypass check
- [ ] Validation rules reference `$Setup.Automation_Bypass__c`
- [ ] Flows have entry conditions or decision elements
- [ ] Test coverage for bypassed scenarios
- [ ] Audit logging implemented
- [ ] Restoration plan documented
- [ ] Time-bound (temporary bypasses have removal date)

After bypass usage:

- [ ] Bypass setting removed/disabled
- [ ] Data integrity verified
- [ ] Audit log reviewed
- [ ] Stakeholder notified of completion

---

**Document Owner:** Platform Team
**Next Review Date:** 2026-08-06
**Related Documents:**
- [Data Migration Standard](./data-migration.md)
- [Error Handling Standard](./error-handling.md)
- [Configuration Management Standard](./configuration-management.md)
