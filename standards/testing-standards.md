# STD-009: Testing Standards

**Standard ID:** STD-009
**Version:** 1.0
**Status:** Active
**Effective Date:** 2026-02-06

---

## 1. Purpose

This standard establishes testing requirements and best practices for all Salesforce development. Proper testing ensures code quality, prevents regressions, and facilitates safe deployments to production.

## 2. Scope

This standard applies to:
- Apex classes and triggers
- Lightning Web Components
- Aura Components
- Flows (via Apex tests)
- Integration endpoints
- All deployable Apex code

## 3. Coverage Requirements

### 3.1 Minimum Coverage Thresholds

| Context | Minimum Coverage | Target Coverage |
|---------|------------------|-----------------|
| Production Deployment | 75% (Salesforce requirement) | 85%+ |
| Individual Class | 75% | 90%+ |
| Trigger | 100% | 100% |
| Critical Business Logic | 95% | 100% |
| Integration Classes | 85% | 95% |

### 3.2 Coverage Quality

Coverage percentage alone is insufficient. Tests must:
- Include meaningful assertions
- Test positive and negative scenarios
- Test boundary conditions
- Test bulk operations (200+ records)
- Test with different user contexts (when applicable)

## 4. Test Class Structure

### 4.1 Standard Template

```apex
/**
 * @description Test class for AccountService
 * @author Author Name
 * @date YYYY-MM-DD
 */
@IsTest
private class AccountServiceTest {

    // ============================================================
    // TEST SETUP
    // ============================================================

    @TestSetup
    static void setup() {
        // Create test data used by multiple tests
        List<Account> accounts = TestDataFactory.createAccounts(5);
        insert accounts;

        List<Contact> contacts = TestDataFactory.createContacts(accounts, 2);
        insert contacts;
    }

    // ============================================================
    // POSITIVE TESTS
    // ============================================================

    @IsTest
    static void testCreateAccount_Success() {
        // Arrange
        String accountName = 'Test Account';
        String industry = 'Technology';

        // Act
        Test.startTest();
        Account result = AccountService.createAccount(accountName, industry);
        Test.stopTest();

        // Assert
        System.assertNotEquals(null, result.Id, 'Account should be created');
        System.assertEquals(accountName, result.Name, 'Account name should match');
        System.assertEquals(industry, result.Industry, 'Industry should match');

        // Verify in database
        Account dbAccount = [SELECT Id, Name, Industry FROM Account WHERE Id = :result.Id];
        System.assertEquals(accountName, dbAccount.Name, 'Database record should match');
    }

    @IsTest
    static void testGetAccountById_Found() {
        // Arrange
        Account testAccount = [SELECT Id, Name FROM Account LIMIT 1];

        // Act
        Test.startTest();
        Account result = AccountService.getAccountById(testAccount.Id);
        Test.stopTest();

        // Assert
        System.assertNotEquals(null, result, 'Account should be found');
        System.assertEquals(testAccount.Id, result.Id, 'IDs should match');
    }

    // ============================================================
    // NEGATIVE TESTS
    // ============================================================

    @IsTest
    static void testCreateAccount_NullName_ThrowsException() {
        // Arrange
        String accountName = null;
        String industry = 'Technology';

        // Act & Assert
        Test.startTest();
        try {
            AccountService.createAccount(accountName, industry);
            System.assert(false, 'Expected exception was not thrown');
        } catch (AccountService.AccountServiceException e) {
            System.assert(e.getMessage().contains('name'), 'Error should mention name');
        }
        Test.stopTest();
    }

    @IsTest
    static void testGetAccountById_NotFound_ReturnsNull() {
        // Arrange
        Id fakeId = TestDataFactory.generateFakeId(Account.SObjectType);

        // Act
        Test.startTest();
        Account result = AccountService.getAccountById(fakeId);
        Test.stopTest();

        // Assert
        System.assertEquals(null, result, 'Should return null for non-existent ID');
    }

    // ============================================================
    // BULK TESTS
    // ============================================================

    @IsTest
    static void testBulkAccountCreation_200Records() {
        // Arrange
        List<Account> accounts = new List<Account>();
        for (Integer i = 0; i < 200; i++) {
            accounts.add(new Account(Name = 'Bulk Test ' + i));
        }

        // Act
        Test.startTest();
        insert accounts;
        Test.stopTest();

        // Assert
        Integer insertedCount = [SELECT COUNT() FROM Account WHERE Name LIKE 'Bulk Test%'];
        System.assertEquals(200, insertedCount, 'All 200 accounts should be inserted');
    }

    // ============================================================
    // USER CONTEXT TESTS
    // ============================================================

    @IsTest
    static void testAccountAccess_StandardUser() {
        // Arrange
        User standardUser = TestDataFactory.createStandardUser();

        // Act & Assert
        System.runAs(standardUser) {
            Test.startTest();
            List<Account> accounts = AccountService.getMyAccounts();
            Test.stopTest();

            // Standard user should only see their own accounts
            for (Account acc : accounts) {
                System.assertEquals(standardUser.Id, acc.OwnerId, 'Should only see owned accounts');
            }
        }
    }
}
```

### 4.2 Arrange-Act-Assert Pattern

Every test should follow the AAA pattern:

```apex
@IsTest
static void testMethodName_Scenario_ExpectedResult() {
    // ARRANGE - Set up test data and conditions
    Account testAccount = new Account(Name = 'Test');
    insert testAccount;

    // ACT - Execute the code being tested
    Test.startTest();
    Account result = MyService.processAccount(testAccount.Id);
    Test.stopTest();

    // ASSERT - Verify the results
    System.assertNotEquals(null, result, 'Result should not be null');
    System.assertEquals('Processed', result.Status__c, 'Status should be Processed');
}
```

## 5. Test Data Factory

### 5.1 Factory Pattern

```apex
/**
 * @description Centralized test data creation for all test classes
 */
@IsTest
public class TestDataFactory {

    // ============================================================
    // ACCOUNTS
    // ============================================================

    public static Account createAccount() {
        return createAccount('Test Account');
    }

    public static Account createAccount(String name) {
        return new Account(
            Name = name,
            Industry = 'Technology',
            Phone = '(555) 123-4567',
            BillingStreet = '123 Test St',
            BillingCity = 'San Francisco',
            BillingState = 'CA',
            BillingPostalCode = '94105',
            BillingCountry = 'USA'
        );
    }

    public static List<Account> createAccounts(Integer count) {
        List<Account> accounts = new List<Account>();
        for (Integer i = 0; i < count; i++) {
            accounts.add(createAccount('Test Account ' + i));
        }
        return accounts;
    }

    // ============================================================
    // CONTACTS
    // ============================================================

    public static Contact createContact(Id accountId) {
        return new Contact(
            FirstName = 'Test',
            LastName = 'Contact',
            Email = 'test.contact@example.com',
            AccountId = accountId
        );
    }

    public static List<Contact> createContacts(List<Account> accounts, Integer contactsPerAccount) {
        List<Contact> contacts = new List<Contact>();
        for (Account acc : accounts) {
            for (Integer i = 0; i < contactsPerAccount; i++) {
                contacts.add(new Contact(
                    FirstName = 'Test',
                    LastName = 'Contact ' + i,
                    Email = 'test' + i + '@' + acc.Name.deleteWhitespace() + '.com',
                    AccountId = acc.Id
                ));
            }
        }
        return contacts;
    }

    // ============================================================
    // OPPORTUNITIES
    // ============================================================

    public static Opportunity createOpportunity(Id accountId) {
        return new Opportunity(
            Name = 'Test Opportunity',
            AccountId = accountId,
            StageName = 'Prospecting',
            CloseDate = Date.today().addDays(30),
            Amount = 10000
        );
    }

    // ============================================================
    // USERS
    // ============================================================

    public static User createStandardUser() {
        Profile standardProfile = [SELECT Id FROM Profile WHERE Name = 'Standard User' LIMIT 1];
        String uniqueString = String.valueOf(DateTime.now().getTime());

        return new User(
            FirstName = 'Test',
            LastName = 'User',
            Email = 'testuser' + uniqueString + '@test.com',
            Username = 'testuser' + uniqueString + '@test.com.sandbox',
            Alias = 'tuser',
            ProfileId = standardProfile.Id,
            TimeZoneSidKey = 'America/Los_Angeles',
            LocaleSidKey = 'en_US',
            EmailEncodingKey = 'UTF-8',
            LanguageLocaleKey = 'en_US'
        );
    }

    public static User createAdminUser() {
        Profile adminProfile = [SELECT Id FROM Profile WHERE Name = 'System Administrator' LIMIT 1];
        String uniqueString = String.valueOf(DateTime.now().getTime());

        return new User(
            FirstName = 'Admin',
            LastName = 'User',
            Email = 'adminuser' + uniqueString + '@test.com',
            Username = 'adminuser' + uniqueString + '@test.com.sandbox',
            Alias = 'auser',
            ProfileId = adminProfile.Id,
            TimeZoneSidKey = 'America/Los_Angeles',
            LocaleSidKey = 'en_US',
            EmailEncodingKey = 'UTF-8',
            LanguageLocaleKey = 'en_US'
        );
    }

    // ============================================================
    // UTILITY METHODS
    // ============================================================

    /**
     * Generate a fake ID for testing without DML
     */
    public static Id generateFakeId(Schema.SObjectType objectType) {
        String prefix = objectType.getDescribe().getKeyPrefix();
        String fakeId = prefix + '0'.repeat(12 - prefix.length());
        return Id.valueOf(fakeId + String.valueOf(Math.random()).right(3));
    }

    /**
     * Create custom setting for bypass
     */
    public static void enableBypass() {
        Automation_Bypass__c bypass = new Automation_Bypass__c(
            SetupOwnerId = UserInfo.getUserId(),
            Bypass_All__c = true
        );
        insert bypass;
    }

    /**
     * Create permission set assignment for testing
     */
    public static void assignPermissionSet(Id userId, String permSetName) {
        PermissionSet ps = [SELECT Id FROM PermissionSet WHERE Name = :permSetName LIMIT 1];
        insert new PermissionSetAssignment(AssigneeId = userId, PermissionSetId = ps.Id);
    }
}
```

### 5.2 Builder Pattern for Complex Objects

```apex
/**
 * @description Builder for creating test Account records with fluent API
 */
@IsTest
public class AccountBuilder {

    private Account account;

    public AccountBuilder() {
        // Initialize with defaults
        this.account = new Account(
            Name = 'Default Account',
            Industry = 'Other'
        );
    }

    public AccountBuilder withName(String name) {
        account.Name = name;
        return this;
    }

    public AccountBuilder withIndustry(String industry) {
        account.Industry = industry;
        return this;
    }

    public AccountBuilder withRevenue(Decimal revenue) {
        account.AnnualRevenue = revenue;
        return this;
    }

    public AccountBuilder withBillingAddress(String street, String city, String state, String zip) {
        account.BillingStreet = street;
        account.BillingCity = city;
        account.BillingState = state;
        account.BillingPostalCode = zip;
        return this;
    }

    public AccountBuilder withOwner(Id ownerId) {
        account.OwnerId = ownerId;
        return this;
    }

    public Account build() {
        return account;
    }

    public Account buildAndInsert() {
        insert account;
        return account;
    }
}

// Usage:
// Account acc = new AccountBuilder()
//     .withName('Acme Corp')
//     .withIndustry('Technology')
//     .withRevenue(1000000)
//     .buildAndInsert();
```

## 6. Testing Triggers

### 6.1 Trigger Test Template

```apex
@IsTest
private class AccountTriggerTest {

    @TestSetup
    static void setup() {
        // Setup without trigger logic if needed
        TestDataFactory.enableBypass();
        List<Account> accounts = TestDataFactory.createAccounts(5);
        insert accounts;
    }

    @IsTest
    static void testBeforeInsert_SetsDefaults() {
        // Arrange - remove bypass for this test
        delete [SELECT Id FROM Automation_Bypass__c];

        Account acc = new Account(Name = 'New Account');
        // Industry is blank - should be set by trigger

        // Act
        Test.startTest();
        insert acc;
        Test.stopTest();

        // Assert
        Account result = [SELECT Id, Industry FROM Account WHERE Id = :acc.Id];
        System.assertEquals('Other', result.Industry, 'Default industry should be set');
    }

    @IsTest
    static void testAfterInsert_CreatesRelatedRecords() {
        // Arrange
        delete [SELECT Id FROM Automation_Bypass__c];
        Account acc = new Account(Name = 'New Account', Create_Default_Contact__c = true);

        // Act
        Test.startTest();
        insert acc;
        Test.stopTest();

        // Assert
        List<Contact> contacts = [SELECT Id FROM Contact WHERE AccountId = :acc.Id];
        System.assertEquals(1, contacts.size(), 'Default contact should be created');
    }

    @IsTest
    static void testBeforeUpdate_ValidatesChanges() {
        // Arrange
        delete [SELECT Id FROM Automation_Bypass__c];
        Account acc = [SELECT Id, AnnualRevenue FROM Account LIMIT 1];

        // Act & Assert
        Test.startTest();
        try {
            acc.AnnualRevenue = -1000; // Invalid
            update acc;
            System.assert(false, 'Expected validation error');
        } catch (DmlException e) {
            System.assert(e.getMessage().contains('Revenue'), 'Should mention revenue in error');
        }
        Test.stopTest();
    }

    @IsTest
    static void testBulkInsert_200Records() {
        // Arrange
        delete [SELECT Id FROM Automation_Bypass__c];
        List<Account> accounts = TestDataFactory.createAccounts(200);

        // Act
        Test.startTest();
        insert accounts;
        Test.stopTest();

        // Assert - verify trigger processed all records
        List<Account> inserted = [SELECT Id, Industry FROM Account WHERE Id IN :accounts];
        System.assertEquals(200, inserted.size(), 'All accounts should be inserted');

        for (Account acc : inserted) {
            System.assertNotEquals(null, acc.Industry, 'Industry should be set for all');
        }
    }

    @IsTest
    static void testBypass_NoTriggerExecution() {
        // Arrange - bypass is already enabled from @TestSetup

        Account acc = new Account(Name = 'Bypassed Account');
        // Industry is blank and should stay blank with bypass

        // Act
        Test.startTest();
        insert acc;
        Test.stopTest();

        // Assert
        Account result = [SELECT Id, Industry FROM Account WHERE Id = :acc.Id];
        System.assertEquals(null, result.Industry, 'Trigger should not execute with bypass');
    }
}
```

## 7. Testing Asynchronous Code

### 7.1 Testing Future Methods

```apex
@IsTest
private class AsyncServiceTest {

    @IsTest
    static void testFutureMethod() {
        // Arrange
        Account acc = TestDataFactory.createAccount();
        insert acc;

        // Act
        Test.startTest();
        AsyncService.processAccountAsync(acc.Id);
        Test.stopTest(); // Forces future method to complete

        // Assert
        Account result = [SELECT Id, Processed__c FROM Account WHERE Id = :acc.Id];
        System.assertEquals(true, result.Processed__c, 'Account should be marked as processed');
    }
}
```

### 7.2 Testing Queueable

```apex
@IsTest
private class AccountQueueableTest {

    @IsTest
    static void testQueueableExecution() {
        // Arrange
        List<Account> accounts = TestDataFactory.createAccounts(10);
        insert accounts;

        Set<Id> accountIds = new Map<Id, Account>(accounts).keySet();

        // Act
        Test.startTest();
        System.enqueueJob(new AccountProcessingQueueable(accountIds));
        Test.stopTest(); // Forces queueable to complete

        // Assert
        List<Account> results = [SELECT Id, Processed__c FROM Account WHERE Id IN :accountIds];
        for (Account acc : results) {
            System.assertEquals(true, acc.Processed__c, 'All accounts should be processed');
        }
    }
}
```

### 7.3 Testing Batch Apex

```apex
@IsTest
private class AccountBatchTest {

    @IsTest
    static void testBatchExecution() {
        // Arrange
        List<Account> accounts = TestDataFactory.createAccounts(200);
        for (Account acc : accounts) {
            acc.Status__c = 'Active';
        }
        insert accounts;

        // Act
        Test.startTest();
        AccountCleanupBatch batch = new AccountCleanupBatch();
        Database.executeBatch(batch, 50);
        Test.stopTest(); // Forces batch to complete

        // Assert
        List<Account> results = [SELECT Id, Status__c FROM Account];
        for (Account acc : results) {
            System.assertEquals('Inactive', acc.Status__c, 'Status should be updated');
        }
    }
}
```

### 7.4 Testing Scheduled Apex

```apex
@IsTest
private class ScheduledJobTest {

    @IsTest
    static void testScheduledJobExecution() {
        // Arrange
        String cronExpression = '0 0 0 1 1 ? 2030'; // Far future date

        // Act
        Test.startTest();
        String jobId = System.schedule('Test Job', cronExpression, new DailyReportScheduler());
        Test.stopTest();

        // Assert - verify job was scheduled
        CronTrigger ct = [SELECT Id, State FROM CronTrigger WHERE Id = :jobId];
        System.assertEquals('WAITING', ct.State, 'Job should be scheduled');
    }
}
```

## 8. Testing Integrations

### 8.1 HTTP Callout Mocks

```apex
@IsTest
public class ApiMockFactory {

    /**
     * Generic mock for any HTTP callout
     */
    public class GenericMock implements HttpCalloutMock {
        private Integer statusCode;
        private String body;
        private Map<String, String> headers;

        public GenericMock(Integer statusCode, String body) {
            this.statusCode = statusCode;
            this.body = body;
            this.headers = new Map<String, String>();
        }

        public GenericMock withHeader(String key, String value) {
            this.headers.put(key, value);
            return this;
        }

        public HttpResponse respond(HttpRequest req) {
            HttpResponse res = new HttpResponse();
            res.setStatusCode(statusCode);
            res.setBody(body);
            for (String key : headers.keySet()) {
                res.setHeader(key, headers.get(key));
            }
            return res;
        }
    }

    /**
     * Multi-request mock for testing multiple callouts
     */
    public class MultiRequestMock implements HttpCalloutMock {
        private Map<String, HttpCalloutMock> mocksByEndpoint;

        public MultiRequestMock() {
            this.mocksByEndpoint = new Map<String, HttpCalloutMock>();
        }

        public MultiRequestMock addMock(String endpointPattern, HttpCalloutMock mock) {
            mocksByEndpoint.put(endpointPattern, mock);
            return this;
        }

        public HttpResponse respond(HttpRequest req) {
            for (String pattern : mocksByEndpoint.keySet()) {
                if (req.getEndpoint().contains(pattern)) {
                    return mocksByEndpoint.get(pattern).respond(req);
                }
            }
            // Default response
            HttpResponse res = new HttpResponse();
            res.setStatusCode(404);
            res.setBody('No mock configured for: ' + req.getEndpoint());
            return res;
        }
    }

    // Pre-built mocks
    public static HttpCalloutMock success(Object body) {
        return new GenericMock(200, JSON.serialize(body));
    }

    public static HttpCalloutMock created(Object body) {
        return new GenericMock(201, JSON.serialize(body));
    }

    public static HttpCalloutMock badRequest(String message) {
        return new GenericMock(400, '{"error": "' + message + '"}');
    }

    public static HttpCalloutMock unauthorized() {
        return new GenericMock(401, '{"error": "Unauthorized"}');
    }

    public static HttpCalloutMock serverError() {
        return new GenericMock(500, '{"error": "Internal Server Error"}');
    }
}
```

### 8.2 Integration Test Example

```apex
@IsTest
private class ExternalApiServiceTest {

    @IsTest
    static void testGetAccount_Success() {
        // Arrange
        Map<String, Object> mockResponse = new Map<String, Object>{
            'id' => '12345',
            'name' => 'External Account',
            'status' => 'active'
        };
        Test.setMock(HttpCalloutMock.class, ApiMockFactory.success(mockResponse));

        // Act
        Test.startTest();
        HttpResponse res = ExternalApiService.doGet('/accounts/12345');
        Test.stopTest();

        // Assert
        System.assertEquals(200, res.getStatusCode());

        Map<String, Object> responseBody = (Map<String, Object>) JSON.deserializeUntyped(res.getBody());
        System.assertEquals('12345', responseBody.get('id'));
        System.assertEquals('External Account', responseBody.get('name'));
    }

    @IsTest
    static void testCreateAccount_Success() {
        // Arrange
        Map<String, Object> requestBody = new Map<String, Object>{
            'name' => 'New Account'
        };
        Map<String, Object> mockResponse = new Map<String, Object>{
            'id' => 'new-123',
            'name' => 'New Account'
        };
        Test.setMock(HttpCalloutMock.class, ApiMockFactory.created(mockResponse));

        // Act
        Test.startTest();
        HttpResponse res = ExternalApiService.doPost('/accounts', requestBody);
        Test.stopTest();

        // Assert
        System.assertEquals(201, res.getStatusCode());
    }

    @IsTest
    static void testGetAccount_Unauthorized() {
        // Arrange
        Test.setMock(HttpCalloutMock.class, ApiMockFactory.unauthorized());

        // Act
        Test.startTest();
        HttpResponse res = ExternalApiService.doGet('/accounts/12345');
        Test.stopTest();

        // Assert
        System.assertEquals(401, res.getStatusCode());
    }
}
```

## 9. Assertions Best Practices

### 9.1 Assert Methods

```apex
// Basic assertions
System.assertEquals(expected, actual, 'Descriptive message');
System.assertNotEquals(unexpected, actual, 'Descriptive message');
System.assert(condition, 'Descriptive message');

// Null checks
System.assertNotEquals(null, result, 'Result should not be null');
System.assertEquals(null, result, 'Result should be null');

// Collection assertions
System.assertEquals(5, results.size(), 'Should return 5 records');
System.assert(!results.isEmpty(), 'Results should not be empty');

// Exception assertions
try {
    dangerousMethod();
    System.assert(false, 'Expected exception was not thrown');
} catch (SpecificException e) {
    System.assert(e.getMessage().contains('expected text'), 'Error message should contain expected text');
}
```

### 9.2 Meaningful Assertion Messages

```apex
// ❌ BAD: No message or unclear message
System.assertEquals(5, accounts.size());
System.assert(result != null);

// ✓ GOOD: Clear, descriptive messages
System.assertEquals(5, accounts.size(), 'Should return exactly 5 accounts matching criteria');
System.assertNotEquals(null, result, 'AccountService.getById should return account for valid ID');
```

## 10. Test Isolation

### 10.1 SeeAllData

```apex
// ❌ NEVER use seeAllData=true (except for specific edge cases)
@IsTest(seeAllData=true) // AVOID
private class BadTest { }

// ✓ ALWAYS create test data
@IsTest
private class GoodTest {
    @TestSetup
    static void setup() {
        // Create all needed test data
    }
}
```

### 10.2 When SeeAllData is Required

```apex
// Only acceptable uses:
// 1. Testing reports
// 2. Testing with specific org data that cannot be created in tests
// 3. ConnectApi tests

@IsTest(seeAllData=true)
private class ReportServiceTest {
    // Document why seeAllData is necessary
}
```

## 11. Compliance Checklist

Before code review:

- [ ] All classes have 75%+ coverage (85%+ preferred)
- [ ] All triggers have 100% coverage
- [ ] Tests follow Arrange-Act-Assert pattern
- [ ] All assertions have meaningful messages
- [ ] Bulk tests (200+ records) included
- [ ] Negative test cases included
- [ ] No seeAllData=true (unless documented exception)
- [ ] Test data created via TestDataFactory
- [ ] Async tests use Test.startTest/stopTest
- [ ] HTTP callouts use mocks
- [ ] Tests are independent (no order dependency)
- [ ] Test class naming follows convention (*Test suffix)

---

**Document Owner:** Platform Team
**Next Review Date:** 2026-08-06
**Related Documents:**
- [Apex Coding Standards](./apex-coding-standards.md)
- [Integration Patterns Standard](./integration-patterns.md)
