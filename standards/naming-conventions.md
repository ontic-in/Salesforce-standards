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

## 4. Description and Help Text Requirements

Every custom object, field, automation, and metadata element **must** include a Description and (where applicable) Help Text. These are not optional — they are a required part of every metadata deployment.

### 4.1 Description (Required on All Custom Metadata and Automations)

The Description should explain **what the value contributes to a larger process** — not just restate the element name. This applies equally to data elements (objects, fields) and automations (flows, validation rules, Apex triggers, scheduled jobs).

| Principle | Guidance |
|-----------|----------|
| Purpose-driven | Explain the business process or workflow this element supports |
| Context | Describe how this element fits into the broader data model or automation |
| Not a label restatement | Never just reword the API name — add genuine context |

**Examples:**

| Element | Bad Description | Good Description |
|---------|----------------|-----------------|
| `Is_Active__c` | "Whether the record is active" | "Controls whether this account is included in the nightly sync to the billing system and appears in active account reports" |
| `Total_Amount__c` | "The total amount" | "Aggregated line item total used by the invoice generation process to calculate final billing amounts" |
| `Status__c` | "Status of the record" | "Drives the approval workflow and determines which automation rules (validation, assignment, escalation) apply to this case" |
| `Customer_Order__c` | "Custom object for orders" | "Represents a customer purchase order that flows through the fulfillment pipeline from creation through shipping confirmation" |
| `SAP_Id__c` | "SAP ID" | "External identifier from SAP ERP used by the nightly integration to match and reconcile account records across systems" |

**Automation Examples:**

| Automation | Bad Description | Good Description |
|------------|----------------|-----------------|
| `Account_Update_Billing_RTF` | "Updates billing on account" | "Triggered on Account billing address change. Propagates updated address to all open Orders and active Subscriptions, then notifies the finance team via email alert." |
| `Case_Escalation_AF` | "Escalates cases" | "Invoked by the Case Escalation scheduled job when a case exceeds its SLA response time. Reassigns to the manager queue, sets priority to Urgent, and creates a Chatter notification for the service director." |
| `Daily_Report_Generation_SCH` | "Scheduled report flow" | "Runs daily at 6 AM UTC. Aggregates prior-day case metrics (volume, resolution time, CSAT) and posts a summary to the #service-ops Slack channel via the Slack integration." |
| `AccountTrigger` | "Trigger on Account" | "Single trigger entry point for all Account DML events. Delegates to AccountTriggerHandler which enforces territory assignment, billing sync validation, and partner scorecard recalculation." |
| `Account_Phone_Required_When_Active` | "Phone required" | "Ensures active accounts always have a contact phone number, which is required by the outbound dialer integration and the nightly lead routing process." |
| `AccountCleanupBatch` | "Cleans up accounts" | "Weekly batch job that identifies accounts with no activity in 18+ months, flags them as Inactive, removes them from territory assignment, and generates a review list for the sales ops team." |

### 4.2 Help Text (Required on All Custom Fields)

Help Text should explain **how the value should be used or populated** — it is the field-level guidance that appears to end users when they hover over the info icon.

| Principle | Guidance |
|-----------|----------|
| User-facing | Write for the person filling in the field, not the developer |
| Actionable | Tell the user what to enter, where to find the value, or what format to use |
| Boundary-setting | Clarify valid values, expected formats, or selection criteria |

**Examples:**

| Field | Bad Help Text | Good Help Text |
|-------|--------------|----------------|
| `Is_Active__c` | "Check if active" | "Check this box if the account has an active contract. Unchecking will remove the account from billing sync and active reports." |
| `Total_Amount__c` | "Enter total" | "Auto-calculated from line items. If you need to override, enter the agreed contract total in USD. This value drives invoice generation." |
| `Status__c` | "Select a status" | "Select the current case status. Moving to 'Escalated' triggers manager notification. Moving to 'Closed' requires a resolution summary." |
| `SAP_Id__c` | "Enter SAP ID" | "Enter the 10-digit SAP customer number from the ERP system. Found in SAP transaction BP under 'Business Partner Number'. Leave blank for non-SAP accounts." |
| `Start_Date__c` | "Enter start date" | "Enter the contract effective date. Must be a future date for new contracts. For renewals, use the day after the previous contract end date." |

### 4.3 Where Each Applies

| Metadata Type | Description Required | Help Text Required |
|---------------|---------------------|--------------------|
| **Data Elements** | | |
| Custom Object | Yes | N/A |
| Custom Field | Yes | Yes |
| Custom Metadata Type | Yes | N/A |
| Custom Setting | Yes | N/A |
| Custom Setting Field | Yes | Yes |
| Custom Metadata Field | Yes | Yes |
| Picklist Value | Yes (via description) | N/A |
| **Automations** | | |
| Flow (all types) | Yes (in Flow Description field) | N/A |
| Validation Rule | Yes (error message serves as user guidance) | N/A |
| Apex Trigger | Yes (in class-level comment block) | N/A |
| Apex Batch/Schedulable | Yes (in class-level comment block) | N/A |
| Approval Process | Yes (in process description) | N/A |
| Email Alert / Template | Yes (in description field) | N/A |
| Platform Event | Yes | N/A |

### 4.4 Description and Help Text Anti-Patterns

| Anti-Pattern | Why It Fails | Fix |
|--------------|-------------|-----|
| Restating the label | "Account Name field" adds zero value | Explain the business role |
| Generic instructions | "Enter a value" | Specify what value, format, and source |
| Developer jargon | "FK to Account object" | Write for business users |
| Empty/blank | Provides no guidance | Always fill in — no exceptions |
| Copy-pasted | Same text on every field | Tailor to each field's specific role |

---

## 5. Custom Objects

### 5.1 API Name Convention

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

### 5.2 Label Convention

```
Format: [Business Entity Name]
Pattern: Title Case with spaces

Examples:
✓ "Customer Order"
✓ "Invoice Line Item"
✓ "Service Request"
```

### 5.3 Plural Label

```
Format: [Business Entity Name]s or irregular plural
Pattern: Standard English plural

Examples:
✓ "Customer Orders"
✓ "Invoice Line Items"
✓ "Service Requests"
```

### 5.4 Junction Objects

```
Format: [Parent1]_[Parent2]__c
Pattern: Both parent names, alphabetical order preferred

Examples:
✓ Account_Product__c
✓ Contact_Campaign__c
✓ Order_Product__c
```

### 5.5 Custom Settings and Custom Metadata Types

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

## 6. Custom Fields

### 6.1 Standard Field Types

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

### 6.2 Boolean/Checkbox Fields

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

### 6.3 Relationship Fields

```
Lookup Format: [Related_Object]__c or [Relationship_Description]__c
Master-Detail Format: [Parent_Object]__c

Examples:
✓ Account__c (lookup to Account)
✓ Primary_Contact__c (specific relationship)
✓ Billing_Address__c (lookup to Address__c)
✓ Parent_Case__c (self-lookup)
```

### 6.4 Formula Fields

```
Format: [Descriptive_Name]__c
Include comments in formula explaining logic

Examples:
✓ Full_Name__c (concatenates First + Last)
✓ Days_Open__c (calculates days since created)
✓ Is_Overdue__c (boolean formula)
```

### 6.5 External ID Fields

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
- Use Text data type (not Number) for flexibility
- Document source system in field description

**Foreign Key Standard:** When a field is created to store a foreign key from an external system (e.g., an ERP customer ID, a billing platform subscription ID), it **MUST** be created as an External ID in Salesforce. This enables upsert operations and ensures the field is indexed for efficient cross-system lookups.

**Uniqueness Decision:**

| Scenario | Set Unique? | Rationale |
|----------|-------------|-----------|
| 1:1 key link between systems (one external record maps to one Salesforce record) | **Yes** | Prevents accidental duplicate creation and enforces data integrity across systems |
| Logical duplicates exist (multiple Salesforce records can share the same external key) | **No** | Examples: a shared parent account ID used across child records, or a campaign code applied to multiple leads |

**Best practice:** Always confirm whether the field should be marked as unique during the design phase. Ask: *"Will there ever be more than one Salesforce record with the same value in this field?"* If the answer is no, set it to unique. If yes (or uncertain), leave it non-unique and document the reasoning in the field description.

> See [STD-003: Data Migration](./data-migration.md) Section 4.2 for External ID strategy.
> See [STD-008: Integration Patterns](./integration-patterns.md) for integration-specific guidance.

## 7. Apex Classes

### 7.1 Class Types and Suffixes

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

### 7.2 Class Naming Pattern

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

### 7.3 Method Naming

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

### 7.4 Variable Naming

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

### 7.5 Constant Naming

```
Format: UPPER_SNAKE_CASE
Pattern: All caps with underscores

Examples:
✓ private static final String DEFAULT_STATUS = 'New';
✓ private static final Integer MAX_RETRY_COUNT = 3;
✓ public static final String API_VERSION = 'v1';
```

## 8. Apex Triggers

### 8.1 Trigger Naming

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

### 8.2 Trigger Structure

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

## 9. Flows

### 9.1 Flow Naming Convention

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

### 9.2 Flow Element Naming

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

### 9.3 Flow Variable Naming

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

## 10. Lightning Web Components

### 10.1 Component Naming

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

### 10.2 LWC File Structure

```
componentName/
├── componentName.html
├── componentName.js
├── componentName.css (optional)
├── componentName.js-meta.xml
└── __tests__/
    └── componentName.test.js
```

### 10.3 JavaScript Naming

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

## 11. Aura Components (Legacy)

### 11.1 Component Naming

```
Format: PascalCase
Pattern: Descriptive bundle name

Examples:
✓ AccountList
✓ ContactCard
✓ CustomLookup
```

### 11.2 Aura Bundle Structure

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

## 12. Validation Rules

### 12.1 Naming Convention

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

## 13. Permission Sets

### 13.1 Naming Convention

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

## 14. Custom Labels

### 14.1 Naming Convention

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

## 15. Reports and Dashboards

### 15.1 Report Naming

```
Format: [Object/Subject] - [Metric/View] [(Owner/Team)]
Pattern: Descriptive with optional ownership

Examples:
✓ Accounts - New This Month
✓ Opportunities - Pipeline by Stage
✓ Cases - Open by Priority (Support Team)
✓ Leads - Conversion Rate by Source
```

### 15.2 Dashboard Naming

```
Format: [Audience] - [Subject] Dashboard
Pattern: Indicates who uses it

Examples:
✓ Sales Team - Pipeline Dashboard
✓ Executive - Company KPIs Dashboard
✓ Service Manager - Case Metrics Dashboard
```

### 15.3 Report Folders

```
Format: [Department/Function] Reports
Pattern: Organized by team or function

Examples:
✓ Sales Reports
✓ Marketing Reports
✓ Executive Reports
✓ Operations Reports
```

## 16. Quick Reference Table

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

## 17. Anti-Patterns to Avoid

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

## 18. Compliance Checklist

Before code review or deployment:

- [ ] All custom objects follow PascalCase__c pattern
- [ ] All custom fields have appropriate type suffixes
- [ ] Boolean fields use Is_ or Has_ prefix
- [ ] **All custom objects and fields have a Description that explains their role in the larger process**
- [ ] **All automations (Flows, triggers, batch jobs, validation rules, approval processes) have a Description**
- [ ] **All custom fields have Help Text that explains how to use or populate the value**
- [ ] **Descriptions are not label restatements — they provide genuine business context**
- [ ] **Help Text is user-facing, actionable, and specifies format/source where applicable**
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
