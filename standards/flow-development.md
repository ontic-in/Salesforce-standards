# STD-010: Flow Development Standard

**Standard ID:** STD-010
**Version:** 1.0
**Status:** Active
**Effective Date:** 2026-02-06

---

## 1. Purpose

This standard establishes guidelines for developing, documenting, and maintaining Salesforce Flows. Following these standards ensures consistent, maintainable, and performant Flow automations that integrate well with the overall Salesforce architecture.

## 2. Scope

This standard applies to:
- Record-Triggered Flows
- Screen Flows
- Schedule-Triggered Flows
- Platform Event-Triggered Flows
- Autolaunched Flows (Subflows)
- Flow Orchestrations

## 3. Flow Type Selection

### 3.1 Decision Matrix

| Requirement | Recommended Flow Type |
|-------------|----------------------|
| React to record changes | Record-Triggered Flow |
| User-guided process with screens | Screen Flow |
| Scheduled batch processing | Schedule-Triggered Flow |
| React to platform events | Platform Event-Triggered Flow |
| Reusable logic called by other flows | Autolaunched Flow (Subflow) |
| Multi-step approval/process | Flow Orchestration |

### 3.2 Record-Triggered Flow Timing

| Timing | Use When |
|--------|----------|
| Before Save | Updating fields on the triggering record (no DML needed) |
| After Save | Creating/updating other records, sending notifications |
| After Save - Run Asynchronously | Long-running operations, callouts, complex processing |

### 3.3 Flow vs Apex Decision

| Use Flow | Use Apex |
|----------|----------|
| Simple field updates | Complex calculations |
| Basic record creation | Cross-object transactions requiring rollback |
| Guided user processes | Heavy data processing (>10K records) |
| No complex error handling | Complex error handling with partial commits |
| Admins need to maintain | Developers maintain |
| < 5 DML operations | Governor limit concerns |

## 4. Naming Conventions

### 4.1 Flow Names

```
Format: [Object/Process]_[Action]_[Type]

Types:
- RTF: Record-Triggered Flow
- SF: Screen Flow
- AF: Autolaunched Flow
- SCH: Schedule-Triggered Flow
- PEF: Platform Event-Triggered Flow
- SUB: Subflow
- ORCH: Orchestration

Examples:
✓ Account_Update_Billing_Address_RTF
✓ Case_Escalation_Process_AF
✓ Lead_Qualification_Form_SF
✓ Daily_Inactive_Account_Cleanup_SCH
✓ Order_Event_Handler_PEF
✓ Validate_Address_SUB
✓ New_Employee_Onboarding_ORCH

✗ My Flow (not descriptive)
✗ Test (too vague)
✗ Account Update (missing type)
```

### 4.2 Flow Element Naming

| Element Type | Prefix | Example |
|--------------|--------|---------|
| Decision | Decision_ | Decision_Is_VIP_Customer |
| Assignment | Assign_ | Assign_Default_Values |
| Get Records | Get_ | Get_Related_Contacts |
| Create Records | Create_ | Create_Follow_Up_Task |
| Update Records | Update_ | Update_Account_Status |
| Delete Records | Delete_ | Delete_Expired_Records |
| Loop | Loop_ | Loop_Through_Line_Items |
| Screen | Screen_ | Screen_Enter_Contact_Info |
| Action | Action_ | Action_Send_Email_Alert |
| Subflow | Subflow_ | Subflow_Calculate_Pricing |

### 4.3 Variable Naming

```
Format: var[Type][Description]

Types:
- Record: Single record variable
- Collection: Collection of records
- Text: String variable
- Number: Numeric variable
- Boolean: True/false variable
- Date: Date variable
- DateTime: DateTime variable

Examples:
✓ varRecordAccount
✓ varCollectionContacts
✓ varTextAccountName
✓ varNumberTotalAmount
✓ varBoolIsApproved
✓ varDateStartDate
✓ varRecordInputOpportunity (for input variables)
✓ varRecordOutputCase (for output variables)

✗ myVar (not descriptive)
✗ x, y, z (too short)
✗ account (ambiguous type)
```

### 4.4 Formula Naming

```
Format: formula[Description]

Examples:
✓ formulaCurrentDateTime
✓ formulaIsHighPriority
✓ formulaFullAddress
✓ formulaNextBusinessDay
```

## 5. Flow Structure

### 5.1 Record-Triggered Flow Structure

```
┌─────────────────────────────────────────┐
│ START (Entry Conditions)                │
│ - Object: Account                       │
│ - Trigger: After Save                   │
│ - Entry: Industry = 'Technology'        │
├─────────────────────────────────────────┤
│ Decision_Check_Bypass                   │
│ ├── Bypass Active → END                 │
│ └── Continue → Main Logic               │
├─────────────────────────────────────────┤
│ Decision_Check_Criteria                 │
│ ├── Criteria Met → Processing Path      │
│ └── Not Met → END                       │
├─────────────────────────────────────────┤
│ Main Processing Logic                   │
│ - Get_Related_Records                   │
│ - Loop_Process_Records                  │
│ - Create/Update_Records                 │
├─────────────────────────────────────────┤
│ Error Handling (Fault Path)             │
│ - Create_Error_Log                      │
└─────────────────────────────────────────┘
```

### 5.2 Screen Flow Structure

```
┌─────────────────────────────────────────┐
│ START                                   │
├─────────────────────────────────────────┤
│ Screen_Welcome                          │
│ (Introduction and instructions)         │
├─────────────────────────────────────────┤
│ Screen_Input_Details                    │
│ (Collect user input)                    │
├─────────────────────────────────────────┤
│ Decision_Validate_Input                 │
│ ├── Invalid → Screen_Error_Message      │
│ └── Valid → Continue                    │
├─────────────────────────────────────────┤
│ Processing Logic                        │
│ - Create/Update Records                 │
│ - Call External Services                │
├─────────────────────────────────────────┤
│ Screen_Confirmation                     │
│ (Show results to user)                  │
├─────────────────────────────────────────┤
│ END                                     │
└─────────────────────────────────────────┘
```

## 6. Entry Conditions and Filters

### 6.1 Entry Conditions (Record-Triggered)

```
Best Practices:
1. Always include bypass check
2. Filter as early as possible
3. Use formula for complex conditions

Entry Condition Example:
AND(
    {!$Setup.Automation_Bypass__c.Bypass_Flows__c} = FALSE,
    {!$Setup.Automation_Bypass__c.Bypass_All__c} = FALSE,
    {!$Record.Status__c} = 'Active'
)
```

### 6.2 Decision Conditions

```
Decision: Decision_Is_Major_Account

Outcome 1: Is_Major_Account
Conditions:
- {!$Record.AnnualRevenue} >= 1000000
- OR {!$Record.Type} = 'Strategic Partner'

Outcome 2: Default Outcome
(No conditions - catches everything else)
```

## 7. Error Handling

### 7.1 Fault Path Pattern

**Every DML element and callout MUST have a fault path.**

```
┌───────────────────────┐
│ Update_Account_Status │
├───────────────────────┤
│         │             │
│    ┌────┴────┐        │
│    ▼         ▼        │
│ Success   Fault       │
│    │         │        │
│    ▼         ▼        │
│ Continue  Create_Error_Log
│              │        │
│              ▼        │
│           END         │
└───────────────────────┘
```

### 7.2 Error Log Creation

Create an Error_Log__c record on fault per [STD-001: Error Handling](./error-handling.md):

| Field | Value |
|-------|-------|
| Error_Type__c | "Flow" |
| Error_Source__c | {!$Flow.CurrentLabel} |
| Error_Message__c | {!$Flow.FaultMessage} |
| Related_Record_Id__c | {!$Record.Id} or triggering record ID |
| Related_Object__c | Object API name (e.g., "Account") |
| User__c | {!$User.Id} |
| Error_DateTime__c | {!$Flow.CurrentDateTime} |
| Severity__c | Based on business impact (see table below) |
| Status__c | "New" |

**Severity Selection Guide:**

| Severity | When to Use |
|----------|-------------|
| Critical | Payment/billing failures, data corruption risk, security issues |
| High | Core business process failures, integration errors |
| Medium | Non-critical automation failures, partial failures |
| Low | Minor issues, cosmetic failures, logging-only scenarios |

### 7.3 User-Friendly Error Messages

For Screen Flows, catch faults and display friendly messages:

```
Screen_Error_Message:
┌─────────────────────────────────────────┐
│ Error                                   │
│                                         │
│ We encountered an issue processing      │
│ your request. Please try again or       │
│ contact support with reference:         │
│ {!formulaErrorReference}                │
│                                         │
│ [Try Again]  [Contact Support]          │
└─────────────────────────────────────────┘
```

## 8. Performance Guidelines

### 8.1 Query Optimization

```
✓ GOOD: Filter in Get Records element
Get Records: Get_Active_Contacts
- Object: Contact
- Filter: AccountId = {!$Record.Id} AND Is_Active__c = TRUE
- How Many: Only the first record (if you only need one)
- Store Fields: Only the fields you need

✗ BAD: Get all, then filter in loop
Get Records → Loop → Decision to filter
(This wastes resources)
```

### 8.2 Loop Best Practices

```
✓ GOOD: Build collection in loop, DML after
Loop_Through_Contacts
├── Assign_Add_To_Collection
│   (Add to varCollectionContactsToUpdate)
└── After Loop
    Update_Contacts (varCollectionContactsToUpdate)

✗ BAD: DML inside loop
Loop_Through_Contacts
├── Update_Single_Contact
│   (Causes multiple DML operations!)
└── ...
```

### 8.3 Bulkification

Record-Triggered Flows process one record at a time but are bulkified automatically. However:

```
Avoid:
- Multiple Get Records for the same object in one flow
- Unnecessary DML operations
- Complex loops with many elements

Optimize:
- Consolidate queries
- Use assignment to build collections
- Single DML operation after loop
```

### 8.4 Governor Limit Considerations

| Limit | Flow Consideration |
|-------|-------------------|
| SOQL Queries (100) | Each Get Records = 1 query |
| SOQL Rows (50,000) | Limit Get Records results |
| DML Operations (150) | Consolidate creates/updates |
| DML Rows (10,000) | Watch collection sizes |

## 9. Subflows

### 9.1 Subflow vs Invocable Apex Decision

**Use this guide to choose between Subflows and Invocable Apex:**

| Criteria | Use Subflow | Use Invocable Apex |
|----------|-------------|-------------------|
| Logic complexity | Simple to moderate | Complex algorithms, heavy processing |
| Reusability | Flow-to-Flow only | Flow, Process Builder, REST API |
| Governor limits | Standard Flow limits | Need finer control, @future, Queueable |
| External callouts | No callouts | HTTP callouts required |
| Error handling | Basic fault paths | Custom exception handling, retry logic |
| Bulk processing | Small record sets | Large volume, batch-style processing |
| Maintainer skill | Flow builders (admins) | Apex developers |
| Testing | Flow debug mode | Unit tests with assertions |

**When to Use Subflows:**

| Use Subflow When |
|------------------|
| Logic is reused across multiple flows |
| Complex validation that appears in multiple places |
| Standard calculations (tax, shipping, etc.) |
| Common data operations (address validation, etc.) |
| Logic maintained by different teams |
| Admin-maintainable without code deployment |

**When to Use Invocable Apex Instead:**

| Use Invocable Apex When |
|-------------------------|
| Need HTTP callouts (Flows cannot make callouts in before-save) |
| Complex business logic requiring unit testing |
| Performance-critical operations with governor limit concerns |
| Need to call @future or Queueable for async processing |
| Logic requires advanced error handling or retry mechanisms |
| Operation needs to be exposed as REST API |

> See [STD-006: Apex Coding Standards](./apex-coding-standards.md) for Invocable Apex patterns.

### 9.2 Subflow Design Pattern

```apex
Subflow: Calculate_Tax_SUB

Input Variables:
- varInputAmount (Number, Required)
- varInputState (Text, Required)

Processing:
- Get tax rate from Custom Metadata
- Calculate tax amount
- Apply any exemptions

Output Variables:
- varOutputTaxAmount (Number)
- varOutputTaxRate (Number)
- varOutputIsExempt (Boolean)
```

### 9.3 Subflow Invocation

```
In Parent Flow:
Action: Subflow_Calculate_Tax
├── Input: varInputAmount = {!varRecordOrder.Total_Amount__c}
├── Input: varInputState = {!varRecordOrder.Shipping_State__c}
└── Outputs:
    ├── Store varOutputTaxAmount → varTaxAmount
    └── Store varOutputIsExempt → varIsExempt
```

## 10. Testing Flows

### 10.1 Debug Mode

Before activating any flow:

1. Run in Debug mode
2. Test with various input scenarios
3. Verify all paths execute correctly
4. Check for null handling
5. Verify error paths work

### 10.2 Test Scenarios

| Scenario Type | What to Test |
|---------------|--------------|
| Happy Path | Normal successful execution |
| Null Values | Fields are null/empty |
| Boundary Values | Maximum/minimum values |
| Invalid Data | Data that should fail validation |
| Bulk | 200 records trigger flow |
| Permission | Different user profiles |
| Bypass | With bypass enabled |

### 10.3 Apex Test Coverage

Test Flows through Apex:

```apex
@IsTest
private class AccountFlowTest {

    @IsTest
    static void testAccountUpdateFlow_SetsStatus() {
        // Arrange
        Account acc = new Account(
            Name = 'Test Account',
            Industry = 'Technology'
        );
        insert acc;

        // Act - Update triggers flow
        Test.startTest();
        acc.AnnualRevenue = 2000000;
        update acc;
        Test.stopTest();

        // Assert - Verify flow results
        Account result = [SELECT Id, Customer_Tier__c FROM Account WHERE Id = :acc.Id];
        System.assertEquals('Enterprise', result.Customer_Tier__c,
            'Flow should set tier to Enterprise for revenue > 1M');
    }

    @IsTest
    static void testAccountFlow_WithBypass() {
        // Arrange
        TestDataFactory.enableBypass();

        Account acc = new Account(
            Name = 'Test Account',
            AnnualRevenue = 2000000
        );
        insert acc;

        // Assert - Flow should not execute
        Account result = [SELECT Id, Customer_Tier__c FROM Account WHERE Id = :acc.Id];
        System.assertEquals(null, result.Customer_Tier__c,
            'Flow should not execute when bypass is enabled');
    }
}
```

## 11. Documentation

### 11.1 Flow Description

Every flow must include a description:

```
Flow: Account_Update_Customer_Tier_RTF

Description:
Purpose: Automatically sets the Customer Tier field based on Annual Revenue.

Trigger: After Save on Account when AnnualRevenue is changed.

Logic:
- Revenue >= $5M → Tier = 'Strategic'
- Revenue >= $1M → Tier = 'Enterprise'
- Revenue >= $100K → Tier = 'Business'
- Revenue < $100K → Tier = 'Starter'

Dependencies: None

Author: [Name]
Created: [Date]
Last Modified: [Date]
```

### 11.2 Element Descriptions

Add descriptions to key elements:

```
Decision: Decision_Check_Revenue_Tier
Description: Routes account to appropriate tier based on revenue threshold

Get Records: Get_Related_Opportunities
Description: Retrieves all open opportunities for the account to calculate pipeline value
```

### 11.3 Change Log

Maintain version history in the description:

```
Change Log:
v1.0 (2026-02-06): Initial version - John Doe
v1.1 (2026-03-15): Added Strategic tier - Jane Smith
v1.2 (2026-04-01): Added bypass check - John Doe
```

## 12. Deployment Considerations

### 12.1 Flow Versioning

```
Active Flow Versions:
- Only ONE version can be active at a time
- Deactivating stops all running interviews
- New version doesn't affect in-progress interviews

Best Practice:
1. Test new version thoroughly
2. Activate during low-usage period
3. Monitor for errors after activation
```

### 12.2 Deployment Order

```
Deploy in this order:
1. Custom Objects and Fields (dependencies)
2. Custom Metadata Types (configuration)
3. Apex Classes (invocable actions)
4. Subflows (dependencies for parent flows)
5. Main Flows (depend on subflows)
6. Activate flows after deployment
```

### 12.3 Migration Checklist

- [ ] Flow description is complete
- [ ] All elements have meaningful names
- [ ] All variables follow naming convention
- [ ] Entry conditions include bypass check
- [ ] All DML elements have fault paths
- [ ] Error logging is configured
- [ ] Tested in sandbox
- [ ] Apex test coverage exists
- [ ] No hardcoded IDs
- [ ] Documented in change log

## 13. Compliance Checklist

Before activating a flow:

- [ ] Follows naming conventions
- [ ] Description documents purpose and logic
- [ ] Entry conditions include bypass check
- [ ] All fault paths create Error_Log__c records
- [ ] Screen flows have user-friendly error handling
- [ ] No SOQL/DML inside loops
- [ ] Tested with Debug mode
- [ ] Apex tests provide coverage
- [ ] Tested with different user profiles
- [ ] Tested bulk scenarios (if applicable)
- [ ] Peer reviewed by another team member
- [ ] Added to release notes/documentation

---

**Document Owner:** Platform Team
**Next Review Date:** 2026-08-06
**Related Documents:**
- [Error Handling Standard](./error-handling.md)
- [Automation Bypass Standard](./automation-bypass.md)
- [Naming Conventions Standard](./naming-conventions.md)
