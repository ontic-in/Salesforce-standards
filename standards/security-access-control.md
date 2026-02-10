# STD-007: Security & Access Control Standard

**Standard ID:** STD-007
**Version:** 1.0
**Status:** Active
**Effective Date:** 2026-02-06

---

## 1. Purpose

This standard establishes security requirements for all Salesforce development to protect data integrity, ensure proper access control, and maintain compliance with security best practices. All code and configuration must enforce appropriate security boundaries.

## 2. Scope

This standard applies to:
- Apex classes and triggers
- Lightning Web Components and Aura components
- Visualforce pages and controllers
- Flow and Process Builder automations
- Integration endpoints
- Custom permissions and permission sets
- Sharing rules and record access

## 3. Object-Level Security (CRUD)

### 3.1 When to Check CRUD

| Scenario | Required? | Notes |
|----------|-----------|-------|
| User-initiated operations | Yes | Always check before DML |
| Trigger context | No | System already validated access |
| Batch/Scheduled jobs | Depends | Check if job acts as specific user |
| Integration user context | Depends | Document security model |
| Internal service methods | Optional | If called by checked methods, may skip |

### 3.2 CRUD Check Implementation

```apex
public class SecureDMLService {

    /**
     * Check if current user can create records of this type
     */
    public static Boolean canCreate(SObjectType objectType) {
        return objectType.getDescribe().isCreateable();
    }

    /**
     * Check if current user can read records of this type
     */
    public static Boolean canRead(SObjectType objectType) {
        return objectType.getDescribe().isAccessible();
    }

    /**
     * Check if current user can update records of this type
     */
    public static Boolean canUpdate(SObjectType objectType) {
        return objectType.getDescribe().isUpdateable();
    }

    /**
     * Check if current user can delete records of this type
     */
    public static Boolean canDelete(SObjectType objectType) {
        return objectType.getDescribe().isDeletable();
    }

    /**
     * Secure insert with CRUD check
     */
    public static void secureInsert(List<SObject> records) {
        if (records == null || records.isEmpty()) {
            return;
        }

        SObjectType objectType = records[0].getSObjectType();

        if (!canCreate(objectType)) {
            throw new SecurityException(
                'Insufficient privileges to create ' + objectType.getDescribe().getName()
            );
        }

        insert records;
    }

    /**
     * Secure update with CRUD check
     */
    public static void secureUpdate(List<SObject> records) {
        if (records == null || records.isEmpty()) {
            return;
        }

        SObjectType objectType = records[0].getSObjectType();

        if (!canUpdate(objectType)) {
            throw new SecurityException(
                'Insufficient privileges to update ' + objectType.getDescribe().getName()
            );
        }

        update records;
    }

    /**
     * Secure delete with CRUD check
     */
    public static void secureDelete(List<SObject> records) {
        if (records == null || records.isEmpty()) {
            return;
        }

        SObjectType objectType = records[0].getSObjectType();

        if (!canDelete(objectType)) {
            throw new SecurityException(
                'Insufficient privileges to delete ' + objectType.getDescribe().getName()
            );
        }

        delete records;
    }
}
```

### 3.3 Usage Pattern

```apex
public class AccountController {

    @AuraEnabled
    public static Account createAccount(String name, String industry) {
        // Validate CRUD access
        if (!SecureDMLService.canCreate(Account.SObjectType)) {
            throw new AuraHandledException('You do not have permission to create accounts');
        }

        Account acc = new Account(Name = name, Industry = industry);
        SecureDMLService.secureInsert(new List<Account>{ acc });
        return acc;
    }
}
```

## 4. Field-Level Security (FLS)

### 4.1 Reading Fields Securely

```apex
public class SecureQueryService {

    /**
     * Query accounts with FLS enforcement
     * Strips fields the user cannot access
     */
    public static List<Account> getAccountsSecure(Set<Id> accountIds) {
        List<Account> accounts = [
            SELECT Id, Name, Industry, AnnualRevenue, Phone, Website
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

    /**
     * Check if specific fields are accessible
     */
    public static Boolean isFieldAccessible(SObjectType objectType, String fieldName) {
        Map<String, Schema.SObjectField> fieldMap = objectType.getDescribe().fields.getMap();

        if (!fieldMap.containsKey(fieldName.toLowerCase())) {
            return false;
        }

        return fieldMap.get(fieldName.toLowerCase()).getDescribe().isAccessible();
    }
}
```

### 4.2 Writing Fields Securely

```apex
public class SecureUpdateService {

    /**
     * Update accounts with FLS enforcement
     * Strips fields the user cannot update
     */
    public static void updateAccountsSecure(List<Account> accounts) {
        // Check object-level access first
        if (!Schema.sObjectType.Account.isUpdateable()) {
            throw new SecurityException('No update access to Account');
        }

        // Strip fields user cannot update
        SObjectAccessDecision decision = Security.stripInaccessible(
            AccessType.UPDATABLE,
            accounts
        );

        // Get list of removed fields for logging
        Map<String, Set<String>> removedFields = decision.getRemovedFields();
        if (!removedFields.isEmpty()) {
            System.debug('Fields stripped due to FLS: ' + removedFields);
        }

        update decision.getRecords();
    }

    /**
     * Check if specific field is updateable
     */
    public static Boolean isFieldUpdateable(SObjectType objectType, String fieldName) {
        Map<String, Schema.SObjectField> fieldMap = objectType.getDescribe().fields.getMap();

        if (!fieldMap.containsKey(fieldName.toLowerCase())) {
            return false;
        }

        return fieldMap.get(fieldName.toLowerCase()).getDescribe().isUpdateable();
    }
}
```

### 4.3 Dynamic Field Access Check

```apex
public class FieldSecurityChecker {

    /**
     * Validate that user can access all specified fields
     * @throws SecurityException if any field is not accessible
     */
    public static void validateFieldAccess(
        SObjectType objectType,
        List<String> fieldNames,
        AccessType accessType
    ) {
        Map<String, Schema.SObjectField> fieldMap = objectType.getDescribe().fields.getMap();
        List<String> inaccessibleFields = new List<String>();

        for (String fieldName : fieldNames) {
            String lowerFieldName = fieldName.toLowerCase();

            if (!fieldMap.containsKey(lowerFieldName)) {
                inaccessibleFields.add(fieldName + ' (not found)');
                continue;
            }

            Schema.DescribeFieldResult fieldDescribe = fieldMap.get(lowerFieldName).getDescribe();

            Boolean hasAccess = false;
            switch on accessType {
                when READABLE {
                    hasAccess = fieldDescribe.isAccessible();
                }
                when CREATABLE {
                    hasAccess = fieldDescribe.isCreateable();
                }
                when UPDATABLE {
                    hasAccess = fieldDescribe.isUpdateable();
                }
            }

            if (!hasAccess) {
                inaccessibleFields.add(fieldName);
            }
        }

        if (!inaccessibleFields.isEmpty()) {
            throw new SecurityException(
                'Insufficient field-level access: ' + String.join(inaccessibleFields, ', ')
            );
        }
    }
}
```

### 4.4 FLS Limitations and Exceptions

Certain Salesforce features do NOT enforce FLS:

| Feature | FLS Enforced? | Security Consideration |
|---------|---------------|------------------------|
| Custom Metadata Types in formulas | No | All users see CMT values via `$CustomMetadata` |
| Custom Settings in formulas | No | All users see values via `$Setup` |
| Validation rule formulas | No | Referenced field values visible in error messages |
| Roll-up summary fields | No | Aggregates may expose restricted field data |
| Formula fields | No | Formula output visible even if source fields are restricted |

**Security Guidelines:**

1. **Do not store sensitive data** in CMT fields referenced in user-visible formulas
2. **Avoid exposing** restricted field values in validation rule error messages
3. **Review formula fields** that reference FLS-restricted source fields
4. **Use Apex with FLS checks** for sensitive data access instead of formulas

See [STD-002: Configuration Management](./configuration-management.md) for CMT security guidance.

## 5. Sharing and Record Access

### 5.1 Sharing Keywords

| Keyword | Behavior | Use When |
|---------|----------|----------|
| `with sharing` | Enforces sharing rules | Default for user-facing code |
| `without sharing` | Bypasses sharing rules | System operations, integrations |
| `inherited sharing` | Uses caller's context | Utility classes |

### 5.2 Default Pattern

```apex
// ✓ CORRECT: Default to with sharing
public with sharing class AccountService {
    // User-facing operations
}

// ✓ CORRECT: Document why without sharing is needed
/**
 * System-level operations that require full data access
 * Used only for batch processing and integrations
 */
public without sharing class AccountSystemService {
    // Internal system operations
}

// ✓ CORRECT: Utility classes inherit context
public inherited sharing class StringUtility {
    // Helper methods that don't query data
}
```

### 5.3 Mixing Sharing Contexts

```apex
public with sharing class AccountController {

    @AuraEnabled
    public static Account getAccount(Id accountId) {
        // This respects sharing rules
        return [SELECT Id, Name FROM Account WHERE Id = :accountId];
    }

    @AuraEnabled
    public static void processAccount(Id accountId) {
        // Need to access system data
        AccountSystemService.updateSystemFields(accountId);
    }
}

// Separate class for operations requiring elevated access
public without sharing class AccountSystemService {

    /**
     * Updates system-managed fields
     * Called from with sharing context when system updates are needed
     */
    public static void updateSystemFields(Id accountId) {
        Account acc = [SELECT Id, System_Timestamp__c FROM Account WHERE Id = :accountId];
        acc.System_Timestamp__c = DateTime.now();
        update acc;
    }
}
```

### 5.4 Sharing Rules Configuration

| Rule Type | Use Case | Performance |
|-----------|----------|-------------|
| Owner-based | Share with owner's manager/team | Fast |
| Criteria-based | Share based on field values | Fast |
| Apex Managed | Complex logic-based sharing | Slower |
| Manual | User-initiated sharing | N/A |

## 6. Permission Sets & Custom Permissions

### 6.1 Permission Set Naming

```
Format: [App/Feature]_[Access Level]

Examples:
✓ Sales_Console_Admin
✓ Sales_Console_User
✓ Integration_API_Full_Access
✓ Reports_Read_Only
```

### 6.2 Custom Permission Pattern

```apex
public class FeatureAccessService {

    private static final String PERM_ADVANCED_SEARCH = 'Advanced_Search_Access';
    private static final String PERM_BULK_OPERATIONS = 'Bulk_Operations_Access';
    private static final String PERM_ADMIN_FEATURES = 'Admin_Features_Access';

    /**
     * Check if user has custom permission
     */
    public static Boolean hasPermission(String permissionName) {
        return FeatureManagement.checkPermission(permissionName);
    }

    /**
     * Check advanced search access
     */
    public static Boolean canUseAdvancedSearch() {
        return hasPermission(PERM_ADVANCED_SEARCH);
    }

    /**
     * Check bulk operations access
     */
    public static Boolean canUseBulkOperations() {
        return hasPermission(PERM_BULK_OPERATIONS);
    }

    /**
     * Validate permission and throw if not granted
     */
    public static void requirePermission(String permissionName) {
        if (!hasPermission(permissionName)) {
            throw new SecurityException(
                'Missing required permission: ' + permissionName
            );
        }
    }
}
```

### 6.3 Usage in Controllers

```apex
public with sharing class AdvancedSearchController {

    @AuraEnabled
    public static List<Account> advancedSearch(String criteria) {
        // Check custom permission
        if (!FeatureAccessService.canUseAdvancedSearch()) {
            throw new AuraHandledException(
                'You do not have permission to use advanced search'
            );
        }

        // Proceed with search
        return AccountSelector.advancedSearch(criteria);
    }
}
```

## 7. Lightning Component Security

### 7.1 Aura Controller Security

```apex
public with sharing class MyComponentController {

    /**
     * Always use with sharing for Aura/LWC controllers
     * Always validate input parameters
     * Always check CRUD/FLS
     */
    @AuraEnabled(cacheable=true)
    public static List<Account> getAccounts(String searchTerm) {
        // Validate input
        if (String.isBlank(searchTerm) || searchTerm.length() < 2) {
            throw new AuraHandledException('Search term must be at least 2 characters');
        }

        // Sanitize input (prevent SOQL injection)
        String sanitizedTerm = String.escapeSingleQuotes(searchTerm);

        // Check CRUD
        if (!Schema.sObjectType.Account.isAccessible()) {
            throw new AuraHandledException('Insufficient access to Accounts');
        }

        // Query with FLS
        List<Account> accounts = [
            SELECT Id, Name, Industry
            FROM Account
            WHERE Name LIKE :('%' + sanitizedTerm + '%')
            LIMIT 50
        ];

        // Strip inaccessible fields
        return (List<Account>) Security.stripInaccessible(
            AccessType.READABLE,
            accounts
        ).getRecords();
    }
}
```

### 7.2 LWC Security Best Practices

```javascript
// accountList.js
import { LightningElement, wire, api } from 'lwc';
import getAccounts from '@salesforce/apex/AccountController.getAccounts';

export default class AccountList extends LightningElement {
    @api recordId;
    accounts;
    error;

    // Never expose sensitive data in public properties
    // Never store credentials client-side
    // Always validate in Apex controller

    @wire(getAccounts, { searchTerm: '$searchTerm' })
    wiredAccounts({ error, data }) {
        if (data) {
            // Data is already secured by Apex controller
            this.accounts = data;
            this.error = undefined;
        } else if (error) {
            this.error = this.reduceErrors(error);
            this.accounts = undefined;
        }
    }

    // Don't expose full error details to users
    reduceErrors(error) {
        if (Array.isArray(error.body)) {
            return error.body.map(e => e.message).join(', ');
        } else if (error.body && error.body.message) {
            return error.body.message;
        }
        return 'An error occurred';
    }
}
```

## 8. SOQL Injection Prevention

### 8.1 Always Use Bind Variables

```apex
// ✓ CORRECT: Bind variables (safe)
String searchTerm = userInput;
List<Account> accounts = [
    SELECT Id, Name
    FROM Account
    WHERE Name = :searchTerm
];

// ❌ WRONG: String concatenation (vulnerable)
String query = 'SELECT Id, Name FROM Account WHERE Name = \'' + userInput + '\'';
List<Account> accounts = Database.query(query);
```

### 8.2 Dynamic SOQL Safety

```apex
public class SafeQueryBuilder {

    /**
     * Build dynamic query safely
     */
    public static List<SObject> executeQuery(
        String objectName,
        List<String> fields,
        String whereClause,
        Map<String, Object> bindVars
    ) {
        // Validate object exists and is accessible
        Schema.SObjectType objectType = Schema.getGlobalDescribe().get(objectName);
        if (objectType == null || !objectType.getDescribe().isAccessible()) {
            throw new SecurityException('Invalid or inaccessible object: ' + objectName);
        }

        // Validate fields
        Map<String, Schema.SObjectField> fieldMap = objectType.getDescribe().fields.getMap();
        List<String> validFields = new List<String>();

        for (String field : fields) {
            if (fieldMap.containsKey(field.toLowerCase())) {
                Schema.DescribeFieldResult fieldDesc = fieldMap.get(field.toLowerCase()).getDescribe();
                if (fieldDesc.isAccessible()) {
                    validFields.add(field);
                }
            }
        }

        if (validFields.isEmpty()) {
            validFields.add('Id'); // Always include Id
        }

        // Build query
        String query = 'SELECT ' + String.join(validFields, ', ') +
                      ' FROM ' + String.escapeSingleQuotes(objectName);

        if (String.isNotBlank(whereClause)) {
            query += ' WHERE ' + whereClause;
        }

        // Execute with bind variables
        return Database.queryWithBinds(query, bindVars, AccessLevel.USER_MODE);
    }
}
```

### 8.3 Escape User Input

```apex
public class InputSanitizer {

    /**
     * Escape string for use in SOQL
     */
    public static String escapeSoql(String input) {
        if (String.isBlank(input)) {
            return input;
        }
        return String.escapeSingleQuotes(input);
    }

    /**
     * Escape string for use in SOSL
     */
    public static String escapeSosl(String input) {
        if (String.isBlank(input)) {
            return input;
        }
        // SOSL reserved characters: ? & | ! { } [ ] ( ) ^ ~ * : \ " ' + -
        String escaped = input;
        escaped = escaped.replace('\\', '\\\\');
        escaped = escaped.replace('\'', '\\\'');
        escaped = escaped.replace('"', '\\"');
        // ... escape other reserved characters as needed
        return escaped;
    }
}
```

## 9. Secure Integration Patterns

### 9.1 Named Credentials

```apex
// ✓ CORRECT: Use Named Credentials
HttpRequest req = new HttpRequest();
req.setEndpoint('callout:My_Named_Credential/api/endpoint');
req.setMethod('GET');

// ❌ WRONG: Hardcoded credentials
req.setHeader('Authorization', 'Bearer abc123'); // Never do this!
```

### 9.2 Secure Credential Storage

| Storage Method | Use Case | Security Level |
|----------------|----------|----------------|
| Named Credentials | Callouts | High (encrypted) |
| Custom Metadata | Non-sensitive config | Medium |
| Protected Custom Settings | Org-level secrets | Medium |
| Custom Labels | Never for credentials | N/A |
| Apex Code | Never for credentials | N/A |

### 9.3 Integration User Pattern

```apex
public class IntegrationSecurityService {

    private static final String INTEGRATION_PERM_SET = 'Integration_API_Access';

    /**
     * Verify current user is valid integration user
     */
    public static Boolean isIntegrationUser() {
        // Check for integration permission set
        List<PermissionSetAssignment> assignments = [
            SELECT Id
            FROM PermissionSetAssignment
            WHERE AssigneeId = :UserInfo.getUserId()
            AND PermissionSet.Name = :INTEGRATION_PERM_SET
            LIMIT 1
        ];

        return !assignments.isEmpty();
    }

    /**
     * Validate integration request
     */
    public static void validateIntegrationAccess() {
        if (!isIntegrationUser()) {
            throw new SecurityException('Invalid integration credentials');
        }
    }
}
```

## 10. Data Masking and Privacy

### 10.1 Sensitive Data Handling

```apex
public class DataMaskingService {

    /**
     * Mask email address for display
     * john.doe@example.com → j***@example.com
     */
    public static String maskEmail(String email) {
        if (String.isBlank(email) || !email.contains('@')) {
            return email;
        }

        List<String> parts = email.split('@');
        String localPart = parts[0];
        String domain = parts[1];

        if (localPart.length() <= 1) {
            return localPart + '***@' + domain;
        }

        return localPart.substring(0, 1) + '***@' + domain;
    }

    /**
     * Mask phone number
     * (555) 123-4567 → (555) ***-4567
     */
    public static String maskPhone(String phone) {
        if (String.isBlank(phone) || phone.length() < 4) {
            return phone;
        }

        // Keep last 4 digits
        String lastFour = phone.substring(phone.length() - 4);
        String masked = phone.substring(0, phone.length() - 7).replaceAll('[0-9]', '*');
        return masked + '***-' + lastFour;
    }

    /**
     * Mask SSN
     * 123-45-6789 → ***-**-6789
     */
    public static String maskSSN(String ssn) {
        if (String.isBlank(ssn) || ssn.length() < 4) {
            return '***-**-****';
        }

        return '***-**-' + ssn.substring(ssn.length() - 4);
    }
}
```

### 10.2 Audit Logging

```apex
public class SecurityAuditLogger {

    /**
     * Log security-relevant events
     */
    public static void logSecurityEvent(
        String eventType,
        String details,
        String severity
    ) {
        Security_Audit_Log__c log = new Security_Audit_Log__c(
            Event_Type__c = eventType,
            Details__c = details,
            Severity__c = severity,
            User__c = UserInfo.getUserId(),
            Timestamp__c = DateTime.now(),
            IP_Address__c = Auth.SessionManagement.getCurrentSession()?.SourceIp
        );

        // Insert without triggers to avoid loops
        Database.insert(log, false);
    }

    /**
     * Log failed access attempt
     */
    public static void logAccessDenied(String resource, String reason) {
        logSecurityEvent(
            'ACCESS_DENIED',
            'Resource: ' + resource + ', Reason: ' + reason,
            'Warning'
        );
    }
}
```

## 11. Compliance Checklist

Before code review:

- [ ] All user-facing classes use `with sharing`
- [ ] CRUD checks before all user-initiated DML
- [ ] FLS enforced using `Security.stripInaccessible()`
- [ ] No SOQL injection vulnerabilities (bind variables used)
- [ ] No hardcoded credentials or IDs
- [ ] Input validation on all public methods
- [ ] Custom permissions used for feature gating
- [ ] Named Credentials used for callouts
- [ ] Sensitive data masked in UI and logs
- [ ] Security audit logging for sensitive operations
- [ ] Error messages don't expose system details

---

**Document Owner:** Security Team
**Next Review Date:** 2026-08-06
**Related Documents:**
- [Apex Coding Standards](./apex-coding-standards.md)
- [Integration Patterns Standard](./integration-patterns.md)
