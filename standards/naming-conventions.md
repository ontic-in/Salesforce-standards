# STD-005: Naming Conventions Standard

**Standard ID:** STD-005
**Version:** 1.0
**Status:** Active
**Effective Date:** 2026-02-06

---

## 1. Purpose

This standard establishes consistent naming conventions for all Salesforce metadata, code, and configuration elements. Consistent naming improves readability, maintainability, searchability, and reduces confusion across development teams.

## 2. Scope

This standard applies to:
- Custom Objects and Fields
- Apex Classes, Triggers, and Interfaces
- Flows and Process Builders
- Lightning Web Components and Aura Components
- Validation Rules and Formula Fields
- Permission Sets and Profiles
- Custom Labels and Custom Metadata Types
- Reports, Dashboards, and List Views

## 3. General Principles

| Principle | Description |
|-----------|-------------|
| Descriptive | Names should clearly describe purpose/function |
| Consistent | Follow the same pattern across similar elements |
| Searchable | Use prefixes/suffixes that enable filtering |
| No Abbreviations | Avoid abbreviations unless universally understood |
| English Only | Use English for all API names |
| No Special Characters | Avoid underscores in middle of words (except API name suffixes) |

## 4. Custom Objects

### 4.1 API Name Convention

```
Format: [BusinessEntity]__c
Pattern: PascalCase + __c suffix

Examples:
✓ Customer_Order__c
✓ Invoice_Line_Item__c
✓ Service_Request__c

✗ custOrder__c (abbreviation)
✗ customer_order__c (lowercase)
✗ CUSTOMER_ORDER__c (all caps)
```

### 4.2 Label Convention

```
Format: [Business Entity Name]
Pattern: Title Case with spaces

Examples:
✓ "Customer Order"
✓ "Invoice Line Item"
✓ "Service Request"
```

### 4.3 Plural Label

```
Format: [Business Entity Name]s or irregular plural
Pattern: Standard English plural

Examples:
✓ "Customer Orders"
✓ "Invoice Line Items"
✓ "Service Requests"
```

### 4.4 Junction Objects

```
Format: [Parent1]_[Parent2]__c
Pattern: Both parent names, alphabetical order preferred

Examples:
✓ Account_Product__c
✓ Contact_Campaign__c
✓ Order_Product__c
```

### 4.5 Custom Settings and Custom Metadata Types

**Custom Settings (Hierarchy):**
```
Format: [Purpose]_Settings__c or [Purpose]_Config__c
Pattern: Descriptive purpose + Settings/Config suffix

Examples:
✓ Automation_Bypass__c (see STD-004)
✓ Integration_Settings__c
✓ Feature_Flags__c

✗ Settings__c (too generic)
✗ MyConfig__c (unclear purpose)
```

**Custom Metadata Types:**
```
Format: [Purpose]__mdt
Pattern: Descriptive purpose + __mdt suffix

Examples:
✓ Trigger_Setting__mdt
✓ Country_Mapping__mdt
✓ Integration_Endpoint__mdt
✓ Validation_Rule_Config__mdt

✗ Config__mdt (too generic)
✗ CMT__mdt (abbreviation)
```

**Field Naming in Settings/CMTs:**
| Pattern | Usage | Example |
|---------|-------|---------|
| `Is_[Feature]__c` | Boolean toggles | `Is_Enabled__c`, `Is_Active__c` |
| `Bypass_[Automation]__c` | Bypass flags | `Bypass_Triggers__c`, `Bypass_Flows__c` |
| `[Setting]_Value__c` | Configuration values | `Timeout_Value__c`, `Retry_Count__c` |

> See [STD-002: Configuration Management](./configuration-management.md) for CMT vs Custom Settings decisions.
> See [STD-004: Automation Bypass](./automation-bypass.md) for bypass settings structure.

## 5. Custom Fields

### 5.1 Standard Field Types

| Field Type | Suffix | Example |
|------------|--------|---------|
| Text | (none) | `First_Name__c` |
| Number | (none) or `_Count`, `_Amount` | `Item_Count__c` |
| Currency | `_Amount` | `Total_Amount__c` |
| Date | `_Date` | `Start_Date__c` |
| DateTime | `_DateTime` or `_Timestamp` | `Created_DateTime__c` |
| Checkbox | `Is_` prefix | `Is_Active__c` |
| Picklist | (none) | `Status__c` |
| Lookup | (none) or relationship name | `Account__c` |
| Master-Detail | (none) | `Parent_Order__c` |
| Formula | (none) or `_Formula` if clarity needed | `Full_Name__c` |
| Roll-Up Summary | (none) or `_Rollup` | `Total_Line_Items__c` |
| Percent | `_Percent` or `_Pct` | `Discount_Percent__c` |

### 5.2 Boolean/Checkbox Fields

```
Format: Is_[State]__c or Has_[Entity]__c
Pattern: Question that answers Yes/No

Examples:
✓ Is_Active__c
✓ Is_Primary__c
✓ Has_Attachments__c
✓ Is_Approved__c

✗ Active__c (unclear if boolean)
✗ Primary_Flag__c (redundant)
```

### 5.3 Relationship Fields

```
Lookup Format: [Related_Object]__c or [Relationship_Description]__c
Master-Detail Format: [Parent_Object]__c

Examples:
✓ Account__c (lookup to Account)
✓ Primary_Contact__c (specific relationship)
✓ Billing_Address__c (lookup to Address__c)
✓ Parent_Case__c (self-lookup)
```

### 5.4 Formula Fields

```
Format: [Descriptive_Name]__c
Include comments in formula explaining logic

Examples:
✓ Full_Name__c (concatenates First + Last)
✓ Days_Open__c (calculates days since created)
✓ Is_Overdue__c (boolean formula)
```

### 5.5 External ID Fields

```
Format: [Source_System]_Id__c or External_Id__c
Pattern: Identifies the source system + "Id" suffix

Examples:
✓ External_Id__c (generic, for single integration)
✓ SAP_Id__c (SAP system identifier)
✓ Legacy_CRM_Id__c (from legacy system migration)
✓ Stripe_Customer_Id__c (Stripe integration)

✗ ExternalID__c (missing underscore)
✗ ExtId__c (unclear abbreviation)
✗ Id__c (too generic)
```

**External ID Field Requirements:**
- Mark as "External ID" in field settings
- Recommend enabling "Unique" checkbox
- Use Text data type (not Number) for flexibility
- Document source system in field description

> See [STD-003: Data Migration](./data-migration.md) Section 4.2 for External ID strategy.
> See [STD-008: Integration Patterns](./integration-patterns.md) for integration-specific guidance.

## 6. Apex Classes

### 6.1 Class Types and Suffixes

| Class Type | Suffix | Example |
|------------|--------|---------|
| Trigger Handler | `TriggerHandler` | `AccountTriggerHandler` |
| Service Class | `Service` | `AccountService` |
| Selector/Query | `Selector` | `AccountSelector` |
| Domain Class | (none) or `Domain` | `Accounts` or `AccountDomain` |
| Controller | `Controller` | `AccountListController` |
| REST Service | `RestService` or `API` | `AccountRestService` |
| Batch Class | `Batch` | `AccountCleanupBatch` |
| Queueable | `Queueable` or `Job` | `AccountSyncQueueable` |
| Schedulable | `Scheduler` or `Schedule` | `DailyReportScheduler` |
| Test Class | `Test` | `AccountServiceTest` |
| Utility | `Utility` or `Utils` or `Helper` | `StringUtility` |
| Exception | `Exception` | `AccountServiceException` |
| Wrapper/DTO | `Wrapper` or `DTO` | `AccountWrapper` |
| Builder | `Builder` | `AccountBuilder` |
| Factory | `Factory` | `TestDataFactory` |
| Interface | `I` prefix or `able` suffix | `IAccountService` or `Loggable` |
| Abstract | `Abstract` prefix or `Base` suffix | `AbstractTriggerHandler` |

### 6.2 Class Naming Pattern

```
Format: [Entity/Feature][Type]
Pattern: PascalCase

Examples:
✓ AccountTriggerHandler
✓ OpportunityService
✓ ContactSelector
✓ OrderLineItemBatch
✓ InvoiceRestService

✗ accountTriggerHandler (camelCase)
✗ Account_Trigger_Handler (underscores)
✗ AcctTrigHandler (abbreviations)
```

### 6.3 Method Naming

```
Format: verbNoun or verb
Pattern: camelCase, starts with verb

Common Verbs:
- get, set, is, has (accessors)
- create, insert, save (DML)
- update, upsert, merge
- delete, remove
- find, search, query
- calculate, compute
- validate, verify, check
- send, notify
- process, handle, execute

Examples:
✓ getAccountById(Id accountId)
✓ calculateTotalAmount()
✓ isValidEmail(String email)
✓ sendNotification(Id userId)
✓ processOrders(List<Order__c> orders)

✗ AccountById() (missing verb)
✗ Calculate_Total() (underscore)
✗ GETACCOUNT() (all caps)
```

### 6.4 Variable Naming

```
Format: descriptiveName
Pattern: camelCase

Examples:
✓ accountList
✓ totalAmount
✓ isValid
✓ accountIdToContactsMap
✓ processedRecordIds

Map/Collection Naming:
✓ accountById (Map<Id, Account>)
✓ contactsByAccountId (Map<Id, List<Contact>>)
✓ activeAccountIds (Set<Id>)
✓ pendingOrders (List<Order__c>)
```

### 6.5 Constant Naming

```
Format: UPPER_SNAKE_CASE
Pattern: All caps with underscores

Examples:
✓ private static final String DEFAULT_STATUS = 'New';
✓ private static final Integer MAX_RETRY_COUNT = 3;
✓ public static final String API_VERSION = 'v1';
```

## 7. Apex Triggers

### 7.1 Trigger Naming

```
Format: [ObjectName]Trigger
Pattern: One trigger per object

Examples:
✓ AccountTrigger
✓ OpportunityTrigger
✓ Custom_Object__cTrigger (for custom objects)

✗ AccountBeforeInsertTrigger (event in name)
✗ Acc_Trigger (abbreviation)
✗ triggerAccount (wrong order)
```

### 7.2 Trigger Structure

```apex
trigger AccountTrigger on Account (
    before insert, after insert,
    before update, after update,
    before delete, after delete,
    after undelete
) {
    AccountTriggerHandler handler = new AccountTriggerHandler();
    handler.execute();
}
```

## 8. Flows

### 8.1 Flow Naming Convention

```
Format: [Object/Process]_[Action]_[Type]
Pattern: Underscores between major parts

Types:
- RTF (Record-Triggered Flow)
- SF (Screen Flow)
- AF (Autolaunched Flow)
- SCH (Scheduled Flow)
- PEF (Platform Event Flow)
- SUB (Subflow)

Examples:
✓ Account_Update_Billing_RTF
✓ Case_Escalation_AF
✓ Lead_Qualification_SF
✓ Daily_Report_Generation_SCH
✓ Order_Event_Handler_PEF
✓ Address_Validation_SUB

✗ My Flow (not descriptive)
✗ AccountFlow (missing action/type)
```

### 8.2 Flow Element Naming

| Element Type | Prefix | Example |
|--------------|--------|---------|
| Screen | Screen_ | Screen_Contact_Info |
| Decision | Decision_ | Decision_Is_VIP |
| Assignment | Assign_ | Assign_Default_Values |
| Get Records | Get_ | Get_Related_Contacts |
| Create Records | Create_ | Create_Follow_Up_Task |
| Update Records | Update_ | Update_Account_Status |
| Delete Records | Delete_ | Delete_Old_Records |
| Loop | Loop_ | Loop_Through_Contacts |
| Action | Action_ | Action_Send_Email |
| Subflow | Subflow_ | Subflow_Validate_Address |

### 8.3 Flow Variable Naming

```
Format: var[Type][Description]
Pattern: Prefix indicates data type

Examples:
✓ varRecordAccount (single record)
✓ varCollectionContacts (collection)
✓ varTextAccountName (text)
✓ varNumberTotalAmount (number)
✓ varBoolIsApproved (boolean)
✓ varDateStartDate (date)
```

## 9. Lightning Web Components

### 9.1 Component Naming

```
Format: camelCase (folder and files)
Pattern: Descriptive, action-oriented

Examples:
✓ accountList
✓ contactCard
✓ orderLineItem
✓ searchFilter
✓ customDataTable

✗ AccountList (PascalCase)
✗ account-list (kebab-case in folder)
✗ acctLst (abbreviations)
```

### 9.2 LWC File Structure

```
componentName/
├── componentName.html
├── componentName.js
├── componentName.css (optional)
├── componentName.js-meta.xml
└── __tests__/
    └── componentName.test.js
```

### 9.3 JavaScript Naming

```javascript
// Properties (camelCase)
@api recordId;
@track isLoading = false;
accountName = '';

// Private properties (underscore prefix optional)
_internalState = {};

// Methods (camelCase, verb prefix)
handleClick() { }
loadAccountData() { }
validateInput() { }

// Constants (UPPER_SNAKE_CASE)
const MAX_RECORDS = 50;
const DEFAULT_PAGE_SIZE = 10;

// Event names (kebab-case)
this.dispatchEvent(new CustomEvent('record-selected'));
```

## 10. Aura Components (Legacy)

### 10.1 Component Naming

```
Format: PascalCase
Pattern: Descriptive bundle name

Examples:
✓ AccountList
✓ ContactCard
✓ CustomLookup
```

### 10.2 Aura Bundle Structure

```
ComponentName/
├── ComponentName.cmp
├── ComponentNameController.js
├── ComponentNameHelper.js
├── ComponentName.css
├── ComponentName.design
├── ComponentName.auradoc
└── ComponentName.svg
```

## 11. Validation Rules

### 11.1 Naming Convention

```
Format: [Object]_[Field/Logic]_[Validation]
Pattern: Descriptive of what is being validated

Examples:
✓ Account_Phone_Required_When_Active
✓ Opportunity_CloseDate_Future_Required
✓ Contact_Email_Format_Valid
✓ Order_Amount_Positive_Required

✗ VR_001 (not descriptive)
✗ Check_Email (too vague)
```

## 12. Permission Sets

### 12.1 Naming Convention

```
Format: [App/Module]_[Access Level]
Pattern: Indicates scope and level

Examples:
✓ Sales_Cloud_Admin
✓ Sales_Cloud_User
✓ Service_Console_Read_Only
✓ Custom_App_Manager
✓ Integration_API_Access

✗ PS_Admin (abbreviation)
✗ John_Permission_Set (user-specific)
```

## 13. Custom Labels

### 13.1 Naming Convention

```
Format: [Category]_[Context]_[Description]
Pattern: Hierarchical organization

Examples:
✓ Error_Validation_Required_Field
✓ Button_Save_And_New
✓ Message_Success_Record_Created
✓ Label_Account_Name
✓ Help_Text_Opportunity_Stage
```

## 14. Reports and Dashboards

### 14.1 Report Naming

```
Format: [Object/Subject] - [Metric/View] [(Owner/Team)]
Pattern: Descriptive with optional ownership

Examples:
✓ Accounts - New This Month
✓ Opportunities - Pipeline by Stage
✓ Cases - Open by Priority (Support Team)
✓ Leads - Conversion Rate by Source
```

### 14.2 Dashboard Naming

```
Format: [Audience] - [Subject] Dashboard
Pattern: Indicates who uses it

Examples:
✓ Sales Team - Pipeline Dashboard
✓ Executive - Company KPIs Dashboard
✓ Service Manager - Case Metrics Dashboard
```

### 14.3 Report Folders

```
Format: [Department/Function] Reports
Pattern: Organized by team or function

Examples:
✓ Sales Reports
✓ Marketing Reports
✓ Executive Reports
✓ Operations Reports
```

## 15. Quick Reference Table

| Element | Convention | Example |
|---------|------------|---------|
| Custom Object | PascalCase__c | `Customer_Order__c` |
| Custom Field | PascalCase__c | `Total_Amount__c` |
| Apex Class | PascalCase + Suffix | `AccountService` |
| Apex Trigger | ObjectTrigger | `AccountTrigger` |
| Apex Method | camelCase verb | `getAccountById()` |
| Apex Variable | camelCase | `accountList` |
| Apex Constant | UPPER_SNAKE | `MAX_RECORDS` |
| Flow | Object_Action_Type | `Account_Update_RTF` |
| Flow Variable | varTypeDesc | `varRecordAccount` |
| LWC | camelCase | `accountList` |
| Aura | PascalCase | `AccountList` |
| Validation Rule | Object_Field_Rule | `Account_Phone_Required` |
| Permission Set | App_Level | `Sales_Cloud_Admin` |
| Custom Label | Category_Context_Desc | `Error_Required_Field` |

## 16. Anti-Patterns to Avoid

| Anti-Pattern | Issue | Correct Approach |
|--------------|-------|------------------|
| `obj1`, `obj2` | Non-descriptive | Use meaningful names |
| `temp`, `test`, `foo` | Temporary names left in | Remove or rename |
| `myClass`, `myMethod` | Generic prefix | Remove "my" prefix |
| `data`, `info`, `details` | Vague suffixes | Be specific |
| `accnt`, `oppty`, `cntct` | Abbreviations | Spell out fully |
| `AccountAccountService` | Redundant | `AccountService` |
| `doStuff()`, `handleIt()` | Vague verbs | Specific actions |
| `flag`, `status`, `type` | Ambiguous alone | Add context |

## 17. Compliance Checklist

Before code review or deployment:

- [ ] All custom objects follow PascalCase__c pattern
- [ ] All custom fields have appropriate type suffixes
- [ ] Boolean fields use Is_ or Has_ prefix
- [ ] Apex classes have appropriate type suffixes
- [ ] One trigger per object with Handler pattern
- [ ] Flows include type suffix (RTF, SF, AF, etc.)
- [ ] LWC folders use camelCase
- [ ] No abbreviations in API names
- [ ] No numbered suffixes (field1, field2)
- [ ] All names are in English
- [ ] Validation rules are descriptive

---

**Document Owner:** Platform Team
**Next Review Date:** 2026-08-06
**Related Documents:** [Apex Coding Standards](./apex-coding-standards.md)
