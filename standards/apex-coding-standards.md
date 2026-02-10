# STD-006: Apex Coding Standards

**Standard ID:** STD-006
**Version:** 1.0
**Status:** Active
**Effective Date:** 2026-02-06

---

## 1. Purpose

This standard establishes coding practices for Apex development that ensure code quality, maintainability, performance, and adherence to Salesforce governor limits. Following these standards reduces bugs, improves collaboration, and creates a consistent codebase.

## 2. Scope

This standard applies to:
- All Apex classes (controllers, services, utilities, etc.)
- Apex triggers
- Batch, Queueable, and Schedulable classes
- Test classes
- Anonymous Apex scripts (for production use)

## 3. Code Structure

### 3.1 Class Structure Template

```apex
/**
 * @description Brief description of class purpose
 * @author Author Name
 * @date YYYY-MM-DD
 * @group Functional Area (e.g., Sales, Service, Integration)
 */
public with sharing class ClassName {

    // ============================================================
    // CONSTANTS
    // ============================================================
    private static final String DEFAULT_STATUS = 'New';
    private static final Integer MAX_RECORDS = 200;

    // ============================================================
    // STATIC VARIABLES
    // ============================================================
    private static Map<Id, Account> accountCache;

    // ============================================================
    // INSTANCE VARIABLES
    // ============================================================
    private Id recordId;
    private Boolean isProcessed = false;

    // ============================================================
    // CONSTRUCTORS
    // ============================================================
    public ClassName() {
        // Default constructor
    }

    public ClassName(Id recordId) {
        this.recordId = recordId;
    }

    // ============================================================
    // PUBLIC METHODS
    // ============================================================
    public void publicMethod() {
        // Implementation
    }

    // ============================================================
    // PRIVATE METHODS
    // ============================================================
    private void privateMethod() {
        // Implementation
    }

    // ============================================================
    // INNER CLASSES
    // ============================================================
    public class CustomException extends Exception {}
}
```

### 3.2 Method Structure

```apex
/**
 * @description Brief description of what method does
 * @param paramName Description of parameter
 * @return Description of return value
 * @throws CustomException When something fails
 */
public ReturnType methodName(ParamType paramName) {
    // Input validation first
    if (paramName == null) {
        throw new IllegalArgumentException('paramName cannot be null');
    }

    // Main logic
    ReturnType result = processLogic(paramName);

    // Return
    return result;
}
```

### 3.3 Access Modifiers

| Modifier | Usage |
|----------|-------|
| `public` | APIs meant for external consumption |
| `private` | Internal implementation details |
| `protected` | Base class methods for extension |
| `global` | Only for managed packages or webservices |

**Rule:** Use the most restrictive modifier possible. Prefer `private` unless there's a specific need for broader access.

## 4. Trigger Pattern

### 4.1 One Trigger Per Object

```apex
// AccountTrigger.trigger
trigger AccountTrigger on Account (
    before insert, after insert,
    before update, after update,
    before delete, after delete,
    after undelete
) {
    new AccountTriggerHandler().execute();
}
```

### 4.2 Trigger Handler Pattern

```apex
/**
 * @description Handler for Account trigger logic
 */
public with sharing class AccountTriggerHandler extends TriggerHandler {

    private List<Account> newAccounts;
    private List<Account> oldAccounts;
    private Map<Id, Account> newMap;
    private Map<Id, Account> oldMap;

    public AccountTriggerHandler() {
        this.newAccounts = (List<Account>) Trigger.new;
        this.oldAccounts = (List<Account>) Trigger.old;
        this.newMap = (Map<Id, Account>) Trigger.newMap;
        this.oldMap = (Map<Id, Account>) Trigger.oldMap;
    }

    public override void beforeInsert() {
        setDefaultValues();
        validateAccounts();
    }

    public override void afterInsert() {
        createRelatedRecords();
    }

    public override void beforeUpdate() {
        validateAccounts();
    }

    public override void afterUpdate() {
        processFieldChanges();
    }

    // Private methods for each operation
    private void setDefaultValues() {
        for (Account acc : newAccounts) {
            if (String.isBlank(acc.Industry)) {
                acc.Industry = 'Other';
            }
        }
    }

    private void validateAccounts() {
        for (Account acc : newAccounts) {
            if (acc.AnnualRevenue != null && acc.AnnualRevenue < 0) {
                acc.addError('Annual Revenue cannot be negative');
            }
        }
    }
}
```

### 4.3 Base Trigger Handler

```apex
/**
 * @description Abstract base class for trigger handlers
 */
public abstract class TriggerHandler {

    public void execute() {
        // Check bypass
        if (BypassUtility.bypassTriggers()) {
            return;
        }

        if (Trigger.isBefore) {
            if (Trigger.isInsert) beforeInsert();
            if (Trigger.isUpdate) beforeUpdate();
            if (Trigger.isDelete) beforeDelete();
        }

        if (Trigger.isAfter) {
            if (Trigger.isInsert) afterInsert();
            if (Trigger.isUpdate) afterUpdate();
            if (Trigger.isDelete) afterDelete();
            if (Trigger.isUndelete) afterUndelete();
        }
    }

    protected virtual void beforeInsert() {}
    protected virtual void afterInsert() {}
    protected virtual void beforeUpdate() {}
    protected virtual void afterUpdate() {}
    protected virtual void beforeDelete() {}
    protected virtual void afterDelete() {}
    protected virtual void afterUndelete() {}
}
```

## 5. Bulkification

### 5.1 Core Principle

**All Apex code must handle up to 200 records in a single transaction.**

### 5.2 Anti-Patterns (Never Do This)

```apex
// ❌ WRONG: SOQL in loop
for (Account acc : accounts) {
    List<Contact> contacts = [SELECT Id FROM Contact WHERE AccountId = :acc.Id];
}

// ❌ WRONG: DML in loop
for (Account acc : accounts) {
    acc.Description = 'Updated';
    update acc;
}

// ❌ WRONG: Callout in loop
for (Account acc : accounts) {
    HttpResponse response = makeCallout(acc.Id);
}
```

### 5.3 Correct Patterns

```apex
// ✓ CORRECT: Collect IDs, single query
Set<Id> accountIds = new Set<Id>();
for (Account acc : accounts) {
    accountIds.add(acc.Id);
}
Map<Id, List<Contact>> contactsByAccountId = getContactsByAccountId(accountIds);

// ✓ CORRECT: Collect records, single DML
List<Account> accountsToUpdate = new List<Account>();
for (Account acc : accounts) {
    accountsToUpdate.add(new Account(
        Id = acc.Id,
        Description = 'Updated'
    ));
}
update accountsToUpdate;

// ✓ CORRECT: Use Queueable for callouts
System.enqueueJob(new AccountCalloutQueueable(accountIds));
```

### 5.4 Collection Patterns

```apex
// Map by Id
Map<Id, Account> accountById = new Map<Id, Account>([
    SELECT Id, Name FROM Account WHERE Id IN :accountIds
]);

// Map of Lists (one-to-many)
Map<Id, List<Contact>> contactsByAccountId = new Map<Id, List<Contact>>();
for (Contact con : [SELECT Id, AccountId FROM Contact WHERE AccountId IN :accountIds]) {
    if (!contactsByAccountId.containsKey(con.AccountId)) {
        contactsByAccountId.put(con.AccountId, new List<Contact>());
    }
    contactsByAccountId.get(con.AccountId).add(con);
}

// Set for uniqueness
Set<String> uniqueEmails = new Set<String>();
for (Contact con : contacts) {
    if (String.isNotBlank(con.Email)) {
        uniqueEmails.add(con.Email.toLowerCase());
    }
}
```

## 6. Governor Limits

### 6.1 Key Limits to Monitor

| Limit | Synchronous | Asynchronous |
|-------|-------------|--------------|
| SOQL Queries | 100 | 200 |
| SOQL Rows | 50,000 | 50,000 |
| DML Statements | 150 | 150 |
| DML Rows | 10,000 | 10,000 |
| Callouts | 100 | 100 |
| CPU Time | 10,000 ms | 60,000 ms |
| Heap Size | 6 MB | 12 MB |

**Per-Record Budget (for triggers processing 200 records):**

| Resource | Total Limit | Per Record Max | Recommended |
|----------|-------------|----------------|-------------|
| SOQL Queries | 100 | 0.5 | Avoid per-record queries |
| DML Statements | 150 | 0.75 | Batch all DML |
| DML Rows | 10,000 | 50 | Max 40 per record |

> **Key Rule:** When designing triggers, assume 200 records and divide limits accordingly. Example: With 10,000 DML rows and 200 records, each trigger execution can create/update at most 50 related records per triggering record.

### 6.2 Checking Limits

```apex
public class GovernorLimitChecker {

    public static void logLimits(String context) {
        System.debug('=== Governor Limits: ' + context + ' ===');
        System.debug('SOQL Queries: ' + Limits.getQueries() + '/' + Limits.getLimitQueries());
        System.debug('SOQL Rows: ' + Limits.getQueryRows() + '/' + Limits.getLimitQueryRows());
        System.debug('DML Statements: ' + Limits.getDmlStatements() + '/' + Limits.getLimitDmlStatements());
        System.debug('DML Rows: ' + Limits.getDmlRows() + '/' + Limits.getLimitDmlRows());
        System.debug('CPU Time: ' + Limits.getCpuTime() + '/' + Limits.getLimitCpuTime());
        System.debug('Heap Size: ' + Limits.getHeapSize() + '/' + Limits.getLimitHeapSize());
    }

    public static Boolean isApproachingQueryLimit() {
        return Limits.getQueries() > (Limits.getLimitQueries() * 0.8);
    }
}
```

### 6.3 Limit-Safe Patterns

```apex
// Check before executing
if (Limits.getQueries() < Limits.getLimitQueries()) {
    // Safe to query
}

// Use Database methods for partial success
Database.SaveResult[] results = Database.insert(records, false);
for (Database.SaveResult result : results) {
    if (!result.isSuccess()) {
        for (Database.Error error : result.getErrors()) {
            System.debug('Error: ' + error.getMessage());
        }
    }
}
```

## 7. SOQL Best Practices

### 7.1 Query Optimization

```apex
// ✓ Query only needed fields
List<Account> accounts = [
    SELECT Id, Name, Industry
    FROM Account
    WHERE Industry = 'Technology'
    LIMIT 100
];

// ✓ Use selective filters (indexed fields)
// Id, Name, OwnerId, CreatedDate, SystemModstamp, RecordTypeId
// External ID fields, Master-Detail relationship fields

// ✓ Use bind variables
String industry = 'Technology';
List<Account> accounts = [
    SELECT Id, Name
    FROM Account
    WHERE Industry = :industry
];

// ❌ Avoid non-selective queries on large objects
// WHERE Name LIKE '%test%'  -- Not selective
// WHERE Description != null -- Not selective
```

### 7.2 Relationship Queries

```apex
// Parent-to-child (subquery)
List<Account> accounts = [
    SELECT Id, Name,
           (SELECT Id, Name, Email FROM Contacts LIMIT 5)
    FROM Account
    WHERE Id IN :accountIds
];

// Child-to-parent (dot notation)
List<Contact> contacts = [
    SELECT Id, Name, Account.Name, Account.Industry
    FROM Contact
    WHERE AccountId IN :accountIds
];
```

### 7.3 Selector Pattern

```apex
/**
 * @description Centralized queries for Account object
 */
public with sharing class AccountSelector {

    public static List<Account> getById(Set<Id> accountIds) {
        return [
            SELECT Id, Name, Industry, AnnualRevenue, OwnerId
            FROM Account
            WHERE Id IN :accountIds
        ];
    }

    public static List<Account> getByIdWithContacts(Set<Id> accountIds) {
        return [
            SELECT Id, Name,
                   (SELECT Id, Name, Email FROM Contacts)
            FROM Account
            WHERE Id IN :accountIds
        ];
    }

    public static List<Account> getActiveByIndustry(String industry) {
        return [
            SELECT Id, Name, Industry
            FROM Account
            WHERE Industry = :industry
            AND Is_Active__c = true
            ORDER BY Name
            LIMIT 1000
        ];
    }
}
```

## 8. DML Best Practices

### 8.1 Partial Success Pattern

```apex
public class DMLService {

    public static List<Database.SaveResult> insertRecords(List<SObject> records) {
        List<Database.SaveResult> results = Database.insert(records, false);
        handleErrors(results, 'Insert');
        return results;
    }

    public static List<Database.SaveResult> updateRecords(List<SObject> records) {
        List<Database.SaveResult> results = Database.update(records, false);
        handleErrors(results, 'Update');
        return results;
    }

    private static void handleErrors(List<Database.SaveResult> results, String operation) {
        List<Error_Log__c> errorLogs = new List<Error_Log__c>();

        for (Integer i = 0; i < results.size(); i++) {
            if (!results[i].isSuccess()) {
                for (Database.Error error : results[i].getErrors()) {
                    errorLogs.add(new Error_Log__c(
                        Error_Type__c = 'Apex',
                        Error_Source__c = 'DMLService.' + operation,
                        Error_Message__c = error.getMessage(),
                        Severity__c = 'High'
                    ));
                }
            }
        }

        if (!errorLogs.isEmpty()) {
            insert errorLogs;
        }
    }
}
```

### 8.2 Avoid Mixed DML

```apex
// ❌ WRONG: Setup and non-setup objects in same transaction
User u = new User(...);
insert u;
Account a = new Account(...);
insert a; // Error!

// ✓ CORRECT: Use future method for setup objects
@future
public static void assignPermissionSet(Id userId, String permSetName) {
    PermissionSet ps = [SELECT Id FROM PermissionSet WHERE Name = :permSetName];
    insert new PermissionSetAssignment(AssigneeId = userId, PermissionSetId = ps.Id);
}
```

## 9. Error Handling

### 9.1 Try-Catch Pattern

```apex
public class AccountService {

    public Account createAccount(String name, String industry) {
        try {
            Account acc = new Account(Name = name, Industry = industry);
            insert acc;
            return acc;

        } catch (DmlException e) {
            ErrorLogger.logException(e, 'AccountService.createAccount', null, 'Account');
            throw new AccountServiceException('Failed to create account: ' + e.getMessage(), e);

        } catch (Exception e) {
            ErrorLogger.logException(e, 'AccountService.createAccount', null, 'Account');
            throw new AccountServiceException('Unexpected error: ' + e.getMessage(), e);
        }
    }
}
```

### 9.2 Custom Exceptions

```apex
public class AccountServiceException extends Exception {
    public String errorCode { get; private set; }

    public AccountServiceException(String message, String errorCode) {
        this(message);
        this.errorCode = errorCode;
    }
}
```

### 9.3 Validation Pattern

```apex
public class ValidationResult {
    public Boolean isValid { get; private set; }
    public List<String> errors { get; private set; }

    public ValidationResult() {
        this.isValid = true;
        this.errors = new List<String>();
    }

    public void addError(String error) {
        this.isValid = false;
        this.errors.add(error);
    }
}

public class AccountValidator {

    public static ValidationResult validate(Account acc) {
        ValidationResult result = new ValidationResult();

        if (String.isBlank(acc.Name)) {
            result.addError('Account Name is required');
        }

        if (acc.AnnualRevenue != null && acc.AnnualRevenue < 0) {
            result.addError('Annual Revenue cannot be negative');
        }

        return result;
    }
}
```

## 10. Asynchronous Apex

### 10.1 When to Use Each Type

| Type | Use Case | Limits |
|------|----------|--------|
| Future | Simple async, callouts | 50 per transaction |
| Queueable | Chaining, complex logic | 50 per transaction |
| Batch | Large data volumes | 5 concurrent jobs |
| Schedulable | Timed execution | N/A |

### 10.2 Queueable Pattern (Preferred)

```apex
public class AccountProcessingQueueable implements Queueable, Database.AllowsCallouts {

    private Set<Id> accountIds;
    private Integer retryCount;
    private static final Integer MAX_RETRIES = 3;

    public AccountProcessingQueueable(Set<Id> accountIds) {
        this(accountIds, 0);
    }

    public AccountProcessingQueueable(Set<Id> accountIds, Integer retryCount) {
        this.accountIds = accountIds;
        this.retryCount = retryCount;
    }

    public void execute(QueueableContext context) {
        try {
            List<Account> accounts = [SELECT Id, Name FROM Account WHERE Id IN :accountIds];

            for (Account acc : accounts) {
                processAccount(acc);
            }

        } catch (Exception e) {
            if (retryCount < MAX_RETRIES) {
                System.enqueueJob(new AccountProcessingQueueable(accountIds, retryCount + 1));
            } else {
                ErrorLogger.logException(e, 'AccountProcessingQueueable', null, 'Account');
            }
        }
    }

    private void processAccount(Account acc) {
        // Processing logic
    }
}
```

### 10.3 Batch Pattern

```apex
public class AccountCleanupBatch implements Database.Batchable<SObject>, Database.Stateful {

    private Integer totalProcessed = 0;
    private Integer totalErrors = 0;

    public Database.QueryLocator start(Database.BatchableContext bc) {
        return Database.getQueryLocator([
            SELECT Id, Name, LastActivityDate
            FROM Account
            WHERE LastActivityDate < LAST_N_YEARS:2
        ]);
    }

    public void execute(Database.BatchableContext bc, List<Account> scope) {
        List<Account> toUpdate = new List<Account>();

        for (Account acc : scope) {
            acc.Status__c = 'Inactive';
            toUpdate.add(acc);
        }

        Database.SaveResult[] results = Database.update(toUpdate, false);

        for (Database.SaveResult result : results) {
            if (result.isSuccess()) {
                totalProcessed++;
            } else {
                totalErrors++;
            }
        }
    }

    public void finish(Database.BatchableContext bc) {
        // Send summary email or log completion
        System.debug('Batch completed. Processed: ' + totalProcessed + ', Errors: ' + totalErrors);
    }
}
```

## 11. Security

### 11.1 With Sharing Keywords

```apex
// Default: Enforce sharing rules
public with sharing class AccountService { }

// Bypass sharing (use sparingly, document why)
public without sharing class SystemService { }

// Inherit from caller
public inherited sharing class UtilityClass { }
```

**Sharing Keyword Decision Guide:**

| Keyword | When to Use | Examples |
|---------|-------------|----------|
| `with sharing` | **Default for all user-facing code.** Use when users should only see/modify records they have access to. | Controllers, Services, Selectors called from UI, Invocable methods from Flows |
| `without sharing` | **System operations requiring full data access.** Requires documented justification in class header. | Integration handlers, Batch processing, Admin utilities, Escalation/rollup calculations, Platform Event handlers |
| `inherited sharing` | **Utility classes that should respect caller's context.** Prevents accidental elevation when called from `with sharing`. | Helper classes, Utility methods, Shared libraries, Generic data factories |

> **Important:** Classes without any sharing keyword default to `without sharing` behavior. Always explicitly declare sharing intent.

> **Documentation Requirement:** When using `without sharing`, add a `@description` comment explaining why sharing bypass is necessary:
> ```apex
> /**
>  * @description Runs without sharing to calculate org-wide rollup summaries
>  *              across all accounts regardless of user visibility.
>  *              Called only from scheduled batch context.
>  */
> public without sharing class AccountRollupCalculator { }
> ```

### 11.2 CRUD/FLS Checks

```apex
public class SecureAccountService {

    public List<Account> getAccounts(Set<Id> accountIds) {
        // Check object-level access
        if (!Schema.sObjectType.Account.isAccessible()) {
            throw new SecurityException('No access to Account object');
        }

        // Query will respect FLS automatically with Security.stripInaccessible
        List<Account> accounts = [
            SELECT Id, Name, Industry, AnnualRevenue
            FROM Account
            WHERE Id IN :accountIds
        ];

        // Strip inaccessible fields
        SObjectAccessDecision decision = Security.stripInaccessible(
            AccessType.READABLE,
            accounts
        );

        return (List<Account>) decision.getRecords();
    }

    public void updateAccounts(List<Account> accounts) {
        // Check update access
        if (!Schema.sObjectType.Account.isUpdateable()) {
            throw new SecurityException('No update access to Account');
        }

        // Strip inaccessible fields before DML
        SObjectAccessDecision decision = Security.stripInaccessible(
            AccessType.UPDATABLE,
            accounts
        );

        update decision.getRecords();
    }
}
```

## 12. Code Quality

### 12.1 Null Safety

```apex
// ✓ Always check for null
if (account != null && String.isNotBlank(account.Name)) {
    // Safe to use
}

// ✓ Use safe navigation operator (API 45.0+)
String accountName = account?.Name;

// ✓ Provide defaults
Integer count = record.Count__c != null ? (Integer) record.Count__c : 0;
```

### 12.2 String Handling

```apex
// ✓ Use String class methods
String.isBlank(value)      // null, empty, or whitespace
String.isNotBlank(value)   // opposite
String.isEmpty(value)      // null or empty
String.isNotEmpty(value)   // opposite

// ✓ Case-insensitive comparison
if (status.equalsIgnoreCase('active')) { }

// ✓ Safe string operations
String result = String.valueOf(someObject);  // Never throws NPE
```

### 12.3 Collection Safety

```apex
// ✓ Initialize collections
List<Account> accounts = new List<Account>();
Map<Id, Account> accountMap = new Map<Id, Account>();
Set<Id> accountIds = new Set<Id>();

// ✓ Check before accessing
if (!accounts.isEmpty()) {
    Account first = accounts[0];
}

if (accountMap.containsKey(accountId)) {
    Account acc = accountMap.get(accountId);
}
```

## 13. Compliance Checklist

Before code review:

- [ ] No SOQL/DML in loops
- [ ] Handles 200 records per trigger execution
- [ ] Uses `with sharing` unless documented reason not to
- [ ] CRUD/FLS checks for user-facing operations
- [ ] Try-catch blocks with proper error logging
- [ ] No hardcoded IDs or credentials
- [ ] Methods are focused (single responsibility)
- [ ] Variables and methods have meaningful names
- [ ] Complex logic has inline comments
- [ ] Class header documentation present
- [ ] Code coverage meets requirements per [STD-009](./testing-standards.md) (75% minimum, 85%+ target)

---

**Document Owner:** Platform Team
**Next Review Date:** 2026-08-06
**Related Documents:**
- [Naming Conventions Standard](./naming-conventions.md)
- [Testing Standards](./testing-standards.md)
- [Security & Access Control Standard](./security-access-control.md)
