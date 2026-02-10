# STD-008: Integration Patterns Standard

**Standard ID:** STD-008
**Version:** 1.0
**Status:** Active
**Effective Date:** 2026-02-06

---

## 1. Purpose

This standard establishes patterns and best practices for integrating Salesforce with external systems. It covers REST/SOAP callouts, Platform Events, inbound APIs, and middleware patterns to ensure reliable, secure, and maintainable integrations.

## 2. Scope

This standard applies to:
- Outbound REST and SOAP callouts
- Inbound REST and SOAP APIs
- Platform Events (publish and subscribe)
- External Services
- Middleware integrations (MuleSoft, Dell Boomi, etc.)
- Batch integrations and data synchronization

## 3. Integration Pattern Selection

### 3.1 Pattern Decision Matrix

| Requirement | Recommended Pattern |
|-------------|---------------------|
| Real-time, synchronous data | REST Callout / Inbound API |
| High-volume, near real-time | Platform Events |
| Large data volumes (>10K records) | Batch / Bulk API |
| Complex transformations | Middleware |
| Legacy systems (XML/WSDL) | SOAP |
| Fire-and-forget notifications | Platform Events |
| Request-response with retry | Queueable + Callout |

### 3.2 Volume Guidelines

| Records per Hour | Recommended Approach |
|------------------|---------------------|
| < 1,000 | Synchronous REST |
| 1,000 - 10,000 | Queueable / Platform Events |
| 10,000 - 100,000 | Batch Apex |
| > 100,000 | Bulk API / Middleware |

## 4. REST Callout Pattern

### 4.1 Named Credentials (Required)

```
Setup → Security → Named Credentials

Name: External_API_Credential
URL: https://api.example.com
Identity Type: Named Principal / Per User
Authentication Protocol: OAuth 2.0 / Password Authentication
```

### 4.2 REST Service Class

```apex
/**
 * @description HTTP callout service for External API
 * All callouts use Named Credentials for security
 */
public class ExternalApiService {

    private static final String NAMED_CREDENTIAL = 'callout:External_API_Credential';
    private static final Integer TIMEOUT_MS = 30000;

    /**
     * GET request
     */
    public static HttpResponse doGet(String endpoint) {
        return doCallout('GET', endpoint, null);
    }

    /**
     * POST request with JSON body
     */
    public static HttpResponse doPost(String endpoint, Object body) {
        return doCallout('POST', endpoint, JSON.serialize(body));
    }

    /**
     * PUT request with JSON body
     */
    public static HttpResponse doPut(String endpoint, Object body) {
        return doCallout('PUT', endpoint, JSON.serialize(body));
    }

    /**
     * DELETE request
     */
    public static HttpResponse doDelete(String endpoint) {
        return doCallout('DELETE', endpoint, null);
    }

    /**
     * Core callout method with error handling
     */
    private static HttpResponse doCallout(String method, String endpoint, String body) {
        HttpRequest req = new HttpRequest();
        req.setEndpoint(NAMED_CREDENTIAL + endpoint);
        req.setMethod(method);
        req.setTimeout(TIMEOUT_MS);
        req.setHeader('Content-Type', 'application/json');
        req.setHeader('Accept', 'application/json');

        if (String.isNotBlank(body)) {
            req.setBody(body);
        }

        Http http = new Http();
        HttpResponse res;

        try {
            res = http.send(req);

            // Log non-success responses
            if (res.getStatusCode() >= 400) {
                logError(method, endpoint, req.getBody(), res);
            }

        } catch (CalloutException e) {
            logCalloutException(method, endpoint, body, e);
            throw new IntegrationException('Callout failed: ' + e.getMessage(), e);
        }

        return res;
    }

    private static void logError(String method, String endpoint, String reqBody, HttpResponse res) {
        ErrorLogger.logIntegrationError(
            'ExternalApiService',
            endpoint,
            res.getStatusCode(),
            reqBody,
            res.getBody(),
            'HTTP ' + res.getStatusCode() + ': ' + res.getStatus(),
            null
        );
    }

    private static void logCalloutException(String method, String endpoint, String body, Exception e) {
        ErrorLogger.logIntegrationError(
            'ExternalApiService',
            endpoint,
            0,
            body,
            null,
            'CalloutException: ' + e.getMessage(),
            null
        );
    }

    public class IntegrationException extends Exception {}
}
```

### 4.3 Response Handling

```apex
public class ApiResponseHandler {

    /**
     * Parse JSON response to typed object
     */
    public static T parseResponse<T>(HttpResponse res, Type targetType) {
        if (res.getStatusCode() >= 200 && res.getStatusCode() < 300) {
            return (T) JSON.deserialize(res.getBody(), targetType);
        }

        throw new ApiException(
            'API Error: ' + res.getStatusCode(),
            res.getStatusCode(),
            res.getBody()
        );
    }

    /**
     * Check if response is successful
     */
    public static Boolean isSuccess(HttpResponse res) {
        return res.getStatusCode() >= 200 && res.getStatusCode() < 300;
    }

    public class ApiException extends Exception {
        public Integer statusCode { get; private set; }
        public String responseBody { get; private set; }

        public ApiException(String message, Integer statusCode, String responseBody) {
            this(message);
            this.statusCode = statusCode;
            this.responseBody = responseBody;
        }
    }
}
```

### 4.4 Retry Pattern with Queueable

```apex
public class ApiCalloutQueueable implements Queueable, Database.AllowsCallouts {

    private String endpoint;
    private String method;
    private String body;
    private Id relatedRecordId;
    private Integer retryCount;

    private static final Integer MAX_RETRIES = 3;
    private static final Set<Integer> RETRYABLE_CODES = new Set<Integer>{
        408, 429, 500, 502, 503, 504
    };

    public ApiCalloutQueueable(String endpoint, String method, String body, Id relatedRecordId) {
        this(endpoint, method, body, relatedRecordId, 0);
    }

    private ApiCalloutQueueable(String endpoint, String method, String body, Id relatedRecordId, Integer retryCount) {
        this.endpoint = endpoint;
        this.method = method;
        this.body = body;
        this.relatedRecordId = relatedRecordId;
        this.retryCount = retryCount;
    }

    public void execute(QueueableContext context) {
        HttpResponse res;

        try {
            res = ExternalApiService.doCallout(method, endpoint, body);

            if (ApiResponseHandler.isSuccess(res)) {
                handleSuccess(res);
            } else if (shouldRetry(res.getStatusCode())) {
                scheduleRetry();
            } else {
                handlePermanentFailure(res);
            }

        } catch (CalloutException e) {
            if (retryCount < MAX_RETRIES) {
                scheduleRetry();
            } else {
                handlePermanentFailure(e);
            }
        }
    }

    private Boolean shouldRetry(Integer statusCode) {
        return retryCount < MAX_RETRIES && RETRYABLE_CODES.contains(statusCode);
    }

    private void scheduleRetry() {
        // Exponential backoff would require Schedulable
        System.enqueueJob(new ApiCalloutQueueable(
            endpoint, method, body, relatedRecordId, retryCount + 1
        ));
    }

    private void handleSuccess(HttpResponse res) {
        // Process successful response
        System.debug('Callout successful: ' + res.getBody());
    }

    private void handlePermanentFailure(HttpResponse res) {
        ErrorLogger.logIntegrationError(
            'ApiCalloutQueueable',
            endpoint,
            res.getStatusCode(),
            body,
            res.getBody(),
            'Permanent failure after ' + retryCount + ' retries',
            relatedRecordId
        );
    }

    private void handlePermanentFailure(Exception e) {
        ErrorLogger.logIntegrationError(
            'ApiCalloutQueueable',
            endpoint,
            0,
            body,
            null,
            'Exception after ' + retryCount + ' retries: ' + e.getMessage(),
            relatedRecordId
        );
    }
}
```

## 5. Inbound REST API

### 5.1 REST Resource Class

```apex
@RestResource(urlMapping='/api/v1/accounts/*')
global with sharing class AccountRestResource {

    /**
     * GET /services/apexrest/api/v1/accounts/{id}
     */
    @HttpGet
    global static AccountResponse getAccount() {
        RestRequest req = RestContext.request;
        RestResponse res = RestContext.response;

        try {
            String accountId = extractIdFromUri(req.requestURI);

            if (String.isBlank(accountId)) {
                return errorResponse(res, 400, 'Account ID is required');
            }

            Account acc = AccountSelector.getById(new Set<Id>{ accountId })[0];

            res.statusCode = 200;
            return new AccountResponse(acc);

        } catch (QueryException e) {
            return errorResponse(res, 404, 'Account not found');
        } catch (Exception e) {
            ErrorLogger.logException(e, 'AccountRestResource.getAccount', null, null);
            return errorResponse(res, 500, 'Internal server error');
        }
    }

    /**
     * POST /services/apexrest/api/v1/accounts
     */
    @HttpPost
    global static AccountResponse createAccount() {
        RestRequest req = RestContext.request;
        RestResponse res = RestContext.response;

        try {
            AccountRequest request = (AccountRequest) JSON.deserialize(
                req.requestBody.toString(),
                AccountRequest.class
            );

            // Validate request
            ValidationResult validation = request.validate();
            if (!validation.isValid) {
                return errorResponse(res, 400, String.join(validation.errors, '; '));
            }

            // Create account
            Account acc = request.toAccount();
            insert acc;

            res.statusCode = 201;
            return new AccountResponse(acc);

        } catch (DmlException e) {
            return errorResponse(res, 400, e.getDmlMessage(0));
        } catch (JSONException e) {
            return errorResponse(res, 400, 'Invalid JSON: ' + e.getMessage());
        } catch (Exception e) {
            ErrorLogger.logException(e, 'AccountRestResource.createAccount', null, null);
            return errorResponse(res, 500, 'Internal server error');
        }
    }

    /**
     * PUT /services/apexrest/api/v1/accounts/{id}
     */
    @HttpPut
    global static AccountResponse updateAccount() {
        RestRequest req = RestContext.request;
        RestResponse res = RestContext.response;

        try {
            String accountId = extractIdFromUri(req.requestURI);
            AccountRequest request = (AccountRequest) JSON.deserialize(
                req.requestBody.toString(),
                AccountRequest.class
            );

            Account acc = request.toAccount();
            acc.Id = accountId;
            update acc;

            res.statusCode = 200;
            return new AccountResponse(acc);

        } catch (Exception e) {
            ErrorLogger.logException(e, 'AccountRestResource.updateAccount', null, null);
            return errorResponse(res, 500, 'Internal server error');
        }
    }

    private static String extractIdFromUri(String uri) {
        List<String> parts = uri.split('/');
        return parts[parts.size() - 1];
    }

    private static AccountResponse errorResponse(RestResponse res, Integer statusCode, String message) {
        res.statusCode = statusCode;
        AccountResponse response = new AccountResponse();
        response.success = false;
        response.errorMessage = message;
        return response;
    }

    // Request/Response DTOs
    global class AccountRequest {
        public String name;
        public String industry;
        public String phone;

        public ValidationResult validate() {
            ValidationResult result = new ValidationResult();
            if (String.isBlank(name)) {
                result.addError('Name is required');
            }
            return result;
        }

        public Account toAccount() {
            return new Account(
                Name = this.name,
                Industry = this.industry,
                Phone = this.phone
            );
        }
    }

    global class AccountResponse {
        public Boolean success;
        public String errorMessage;
        public String id;
        public String name;
        public String industry;

        public AccountResponse() {
            this.success = true;
        }

        public AccountResponse(Account acc) {
            this.success = true;
            this.id = acc.Id;
            this.name = acc.Name;
            this.industry = acc.Industry;
        }
    }
}
```

### 5.2 API Versioning

```apex
// Version in URL
@RestResource(urlMapping='/api/v1/accounts/*')  // Version 1
@RestResource(urlMapping='/api/v2/accounts/*')  // Version 2

// Or version in header
@HttpGet
global static Response getResource() {
    String apiVersion = RestContext.request.headers.get('X-API-Version');
    if (apiVersion == '2') {
        return getResourceV2();
    }
    return getResourceV1();
}
```

## 6. Platform Events

### 6.1 Event Design

```
Platform Event: Order_Event__e

Fields:
- Order_Id__c (Text) - External order ID
- Order_Status__c (Text) - Current status
- Customer_Id__c (Text) - Customer reference
- Order_Total__c (Number) - Order amount
- Event_Type__c (Text) - Created, Updated, Cancelled
- Correlation_Id__c (Text) - For tracking across systems
- Timestamp__c (DateTime) - When event occurred
```

### 6.2 Publishing Events

```apex
public class OrderEventPublisher {

    /**
     * Publish single event
     */
    public static Database.SaveResult publishEvent(Order__c order, String eventType) {
        Order_Event__e event = createEvent(order, eventType);
        return EventBus.publish(event);
    }

    /**
     * Publish multiple events
     */
    public static List<Database.SaveResult> publishEvents(List<Order__c> orders, String eventType) {
        List<Order_Event__e> events = new List<Order_Event__e>();

        for (Order__c order : orders) {
            events.add(createEvent(order, eventType));
        }

        return EventBus.publish(events);
    }

    private static Order_Event__e createEvent(Order__c order, String eventType) {
        return new Order_Event__e(
            Order_Id__c = order.External_Id__c,
            Order_Status__c = order.Status__c,
            Customer_Id__c = order.Customer__r.External_Id__c,
            Order_Total__c = order.Total_Amount__c,
            Event_Type__c = eventType,
            Correlation_Id__c = generateCorrelationId(),
            Timestamp__c = DateTime.now()
        );
    }

    private static String generateCorrelationId() {
        return EncodingUtil.convertToHex(Crypto.generateAESKey(128)).substring(0, 32);
    }
}
```

### 6.3 Subscribing to Events (Trigger)

```apex
trigger OrderEventTrigger on Order_Event__e (after insert) {
    OrderEventHandler.handleEvents(Trigger.new);
}

public class OrderEventHandler {

    public static void handleEvents(List<Order_Event__e> events) {
        // Group events by type for efficient processing
        Map<String, List<Order_Event__e>> eventsByType = new Map<String, List<Order_Event__e>>();

        for (Order_Event__e event : events) {
            if (!eventsByType.containsKey(event.Event_Type__c)) {
                eventsByType.put(event.Event_Type__c, new List<Order_Event__e>());
            }
            eventsByType.get(event.Event_Type__c).add(event);
        }

        // Process each type
        if (eventsByType.containsKey('Created')) {
            processCreatedEvents(eventsByType.get('Created'));
        }
        if (eventsByType.containsKey('Updated')) {
            processUpdatedEvents(eventsByType.get('Updated'));
        }
        if (eventsByType.containsKey('Cancelled')) {
            processCancelledEvents(eventsByType.get('Cancelled'));
        }
    }

    private static void processCreatedEvents(List<Order_Event__e> events) {
        // Create local order records
        List<Order__c> ordersToCreate = new List<Order__c>();

        for (Order_Event__e event : events) {
            ordersToCreate.add(new Order__c(
                External_Id__c = event.Order_Id__c,
                Status__c = event.Order_Status__c,
                Total_Amount__c = event.Order_Total__c
            ));
        }

        Database.insert(ordersToCreate, false);
    }

    private static void processUpdatedEvents(List<Order_Event__e> events) {
        // Update existing orders
    }

    private static void processCancelledEvents(List<Order_Event__e> events) {
        // Cancel orders
    }
}
```

### 6.4 Event Replay and Error Handling

```apex
public class EventSubscriptionManager {

    /**
     * Set replay ID for subscription recovery
     */
    public static void setReplayId(String subscriptionName, String replayId) {
        EventBusSubscriber subscriber = [
            SELECT Id, ExternalId
            FROM EventBusSubscriber
            WHERE Topic = :subscriptionName
            LIMIT 1
        ];

        // Store replay ID for recovery
        Integration_State__c state = new Integration_State__c(
            Name = subscriptionName,
            Replay_Id__c = replayId,
            Last_Updated__c = DateTime.now()
        );
        upsert state Name;
    }

    /**
     * Get last processed replay ID
     */
    public static String getLastReplayId(String subscriptionName) {
        List<Integration_State__c> states = [
            SELECT Replay_Id__c
            FROM Integration_State__c
            WHERE Name = :subscriptionName
            LIMIT 1
        ];

        return states.isEmpty() ? null : states[0].Replay_Id__c;
    }
}
```

## 7. Middleware Integration Pattern

### 7.1 Canonical Data Model

```apex
/**
 * Canonical representation for cross-system data exchange
 * Independent of Salesforce field names
 */
public class CanonicalAccount {
    public String externalId;
    public String name;
    public String industry;
    public String phone;
    public String email;
    public CanonicalAddress billingAddress;
    public List<CanonicalContact> contacts;

    public CanonicalAccount() {
        this.contacts = new List<CanonicalContact>();
    }

    /**
     * Convert from Salesforce Account
     */
    public static CanonicalAccount fromSalesforce(Account acc) {
        CanonicalAccount canonical = new CanonicalAccount();
        canonical.externalId = acc.External_Id__c;
        canonical.name = acc.Name;
        canonical.industry = acc.Industry;
        canonical.phone = acc.Phone;
        canonical.email = acc.Email__c;

        if (acc.BillingStreet != null) {
            canonical.billingAddress = new CanonicalAddress(
                acc.BillingStreet,
                acc.BillingCity,
                acc.BillingState,
                acc.BillingPostalCode,
                acc.BillingCountry
            );
        }

        return canonical;
    }

    /**
     * Convert to Salesforce Account
     */
    public Account toSalesforce() {
        Account acc = new Account(
            External_Id__c = this.externalId,
            Name = this.name,
            Industry = this.industry,
            Phone = this.phone,
            Email__c = this.email
        );

        if (this.billingAddress != null) {
            acc.BillingStreet = this.billingAddress.street;
            acc.BillingCity = this.billingAddress.city;
            acc.BillingState = this.billingAddress.state;
            acc.BillingPostalCode = this.billingAddress.postalCode;
            acc.BillingCountry = this.billingAddress.country;
        }

        return acc;
    }
}

public class CanonicalAddress {
    public String street;
    public String city;
    public String state;
    public String postalCode;
    public String country;

    public CanonicalAddress(String street, String city, String state, String postalCode, String country) {
        this.street = street;
        this.city = city;
        this.state = state;
        this.postalCode = postalCode;
        this.country = country;
    }
}

public class CanonicalContact {
    public String externalId;
    public String firstName;
    public String lastName;
    public String email;
    public String phone;
}
```

### 7.2 Transformation Service

```apex
public class DataTransformationService {

    /**
     * Transform external system data to Salesforce format
     */
    public static Account transformToSalesforce(Map<String, Object> externalData) {
        Account acc = new Account();

        // Map external fields to Salesforce fields
        acc.External_Id__c = (String) externalData.get('customer_id');
        acc.Name = (String) externalData.get('company_name');
        acc.Industry = mapIndustry((String) externalData.get('industry_code'));
        acc.Phone = formatPhone((String) externalData.get('phone'));
        acc.AnnualRevenue = parseDecimal(externalData.get('annual_revenue'));

        return acc;
    }

    /**
     * Map external industry codes to Salesforce picklist values
     */
    private static String mapIndustry(String externalCode) {
        Map<String, String> industryMapping = new Map<String, String>{
            'TECH' => 'Technology',
            'FIN' => 'Finance',
            'HEALTH' => 'Healthcare',
            'MFG' => 'Manufacturing'
        };

        return industryMapping.containsKey(externalCode)
            ? industryMapping.get(externalCode)
            : 'Other';
    }

    /**
     * Format phone to standard format
     */
    private static String formatPhone(String phone) {
        if (String.isBlank(phone)) return null;

        // Remove non-numeric characters
        String digits = phone.replaceAll('[^0-9]', '');

        if (digits.length() == 10) {
            return '(' + digits.substring(0, 3) + ') ' +
                   digits.substring(3, 6) + '-' +
                   digits.substring(6);
        }

        return phone;
    }

    private static Decimal parseDecimal(Object value) {
        if (value == null) return null;
        if (value instanceof Decimal) return (Decimal) value;
        if (value instanceof String) {
            try {
                return Decimal.valueOf((String) value);
            } catch (Exception e) {
                return null;
            }
        }
        return null;
    }
}
```

## 8. Batch Integration Pattern

### 8.1 Batch Sync Job

```apex
public class AccountSyncBatch implements Database.Batchable<SObject>, Database.AllowsCallouts, Database.Stateful {

    private Integer successCount = 0;
    private Integer errorCount = 0;
    private List<String> errors = new List<String>();

    public Database.QueryLocator start(Database.BatchableContext bc) {
        return Database.getQueryLocator([
            SELECT Id, Name, External_Id__c, Industry, Phone,
                   BillingStreet, BillingCity, BillingState,
                   BillingPostalCode, BillingCountry,
                   Sync_Status__c, Last_Sync_Date__c
            FROM Account
            WHERE Sync_Status__c = 'Pending'
            OR (Sync_Status__c = 'Error' AND Retry_Count__c < 3)
        ]);
    }

    public void execute(Database.BatchableContext bc, List<Account> scope) {
        List<Account> accountsToUpdate = new List<Account>();

        for (Account acc : scope) {
            try {
                // Transform to canonical format
                CanonicalAccount canonical = CanonicalAccount.fromSalesforce(acc);

                // Send to external system
                HttpResponse res = ExternalApiService.doPost('/accounts', canonical);

                if (ApiResponseHandler.isSuccess(res)) {
                    acc.Sync_Status__c = 'Synced';
                    acc.Last_Sync_Date__c = DateTime.now();
                    acc.Retry_Count__c = 0;
                    successCount++;
                } else {
                    acc.Sync_Status__c = 'Error';
                    acc.Sync_Error__c = res.getBody().left(255);
                    acc.Retry_Count__c = (acc.Retry_Count__c == null ? 0 : acc.Retry_Count__c) + 1;
                    errorCount++;
                }

            } catch (Exception e) {
                acc.Sync_Status__c = 'Error';
                acc.Sync_Error__c = e.getMessage().left(255);
                acc.Retry_Count__c = (acc.Retry_Count__c == null ? 0 : acc.Retry_Count__c) + 1;
                errorCount++;
                errors.add(acc.Id + ': ' + e.getMessage());
            }

            accountsToUpdate.add(acc);
        }

        // Bypass triggers for sync status updates
        BypassUtility.enableTriggerBypassForTransaction();
        update accountsToUpdate;
    }

    public void finish(Database.BatchableContext bc) {
        // Send summary notification
        String subject = 'Account Sync Complete: ' + successCount + ' success, ' + errorCount + ' errors';
        String body = 'Sync completed.\n\nSuccess: ' + successCount + '\nErrors: ' + errorCount;

        if (!errors.isEmpty()) {
            body += '\n\nError Details:\n' + String.join(errors, '\n');
        }

        // Send email or create task for review
        Messaging.SingleEmailMessage email = new Messaging.SingleEmailMessage();
        email.setToAddresses(new List<String>{ 'admin@company.com' });
        email.setSubject(subject);
        email.setPlainTextBody(body);
        Messaging.sendEmail(new List<Messaging.SingleEmailMessage>{ email });
    }
}
```

## 9. Error Handling and Monitoring

### 9.1 Integration Error Object

```
Object: Integration_Error__c

Fields:
- Name (Auto Number): ERR-{00000}
- Integration_Name__c (Text): Name of integration
- Direction__c (Picklist): Inbound, Outbound
- Endpoint__c (URL): API endpoint
- Method__c (Text): GET, POST, PUT, DELETE
- Request_Body__c (Long Text): Request payload
- Response_Body__c (Long Text): Response payload
- Status_Code__c (Number): HTTP status code
- Error_Message__c (Long Text): Error description
- Related_Record_Id__c (Text): Salesforce record ID
- Related_Object__c (Text): Object API name
- Correlation_Id__c (Text): For tracing
- Retry_Count__c (Number): Number of retries
- Status__c (Picklist): New, Retrying, Resolved, Failed
- Created_Date__c (DateTime): When error occurred
```

### 9.2 Circuit Breaker Pattern

```apex
public class CircuitBreaker {

    private static Map<String, CircuitState> circuitStates = new Map<String, CircuitState>();

    private static final Integer FAILURE_THRESHOLD = 5;
    private static final Integer TIMEOUT_SECONDS = 60;

    public static Boolean isOpen(String serviceName) {
        CircuitState state = getState(serviceName);

        if (state.status == 'OPEN') {
            // Check if timeout has passed
            if (DateTime.now().getTime() - state.lastFailure.getTime() > TIMEOUT_SECONDS * 1000) {
                state.status = 'HALF_OPEN';
                return false;
            }
            return true;
        }

        return false;
    }

    public static void recordSuccess(String serviceName) {
        CircuitState state = getState(serviceName);
        state.failureCount = 0;
        state.status = 'CLOSED';
    }

    public static void recordFailure(String serviceName) {
        CircuitState state = getState(serviceName);
        state.failureCount++;
        state.lastFailure = DateTime.now();

        if (state.failureCount >= FAILURE_THRESHOLD) {
            state.status = 'OPEN';
        }
    }

    private static CircuitState getState(String serviceName) {
        if (!circuitStates.containsKey(serviceName)) {
            circuitStates.put(serviceName, new CircuitState());
        }
        return circuitStates.get(serviceName);
    }

    private class CircuitState {
        public String status = 'CLOSED';
        public Integer failureCount = 0;
        public DateTime lastFailure;
    }
}

// Usage
public class ExternalServiceCaller {

    public static HttpResponse callService(String endpoint) {
        String serviceName = 'ExternalService';

        if (CircuitBreaker.isOpen(serviceName)) {
            throw new CircuitOpenException('Service temporarily unavailable');
        }

        try {
            HttpResponse res = ExternalApiService.doGet(endpoint);

            if (ApiResponseHandler.isSuccess(res)) {
                CircuitBreaker.recordSuccess(serviceName);
            } else {
                CircuitBreaker.recordFailure(serviceName);
            }

            return res;

        } catch (Exception e) {
            CircuitBreaker.recordFailure(serviceName);
            throw e;
        }
    }

    public class CircuitOpenException extends Exception {}
}
```

## 10. Testing Integrations

### 10.1 HTTP Mock

```apex
@IsTest
public class ExternalApiMock implements HttpCalloutMock {

    private Integer statusCode;
    private String body;

    public ExternalApiMock(Integer statusCode, String body) {
        this.statusCode = statusCode;
        this.body = body;
    }

    public HttpResponse respond(HttpRequest req) {
        HttpResponse res = new HttpResponse();
        res.setStatusCode(statusCode);
        res.setBody(body);
        res.setHeader('Content-Type', 'application/json');
        return res;
    }

    // Factory methods for common responses
    public static ExternalApiMock success(Object responseBody) {
        return new ExternalApiMock(200, JSON.serialize(responseBody));
    }

    public static ExternalApiMock notFound() {
        return new ExternalApiMock(404, '{"error": "Not found"}');
    }

    public static ExternalApiMock serverError() {
        return new ExternalApiMock(500, '{"error": "Internal server error"}');
    }
}
```

### 10.2 Integration Test

```apex
@IsTest
private class ExternalApiServiceTest {

    @IsTest
    static void testSuccessfulCallout() {
        // Setup mock
        Map<String, Object> mockResponse = new Map<String, Object>{
            'id' => '12345',
            'status' => 'success'
        };
        Test.setMock(HttpCalloutMock.class, ExternalApiMock.success(mockResponse));

        Test.startTest();
        HttpResponse res = ExternalApiService.doGet('/accounts/12345');
        Test.stopTest();

        System.assertEquals(200, res.getStatusCode());
        System.assert(res.getBody().contains('success'));
    }

    @IsTest
    static void testFailedCallout() {
        Test.setMock(HttpCalloutMock.class, ExternalApiMock.serverError());

        Test.startTest();
        HttpResponse res = ExternalApiService.doGet('/accounts/12345');
        Test.stopTest();

        System.assertEquals(500, res.getStatusCode());
    }
}
```

## 11. Compliance Checklist

Before deploying integrations:

- [ ] Named Credentials used (no hardcoded credentials)
- [ ] Error handling with logging implemented
- [ ] Retry logic for transient failures
- [ ] Circuit breaker for external dependencies
- [ ] Timeout configured (not default)
- [ ] Request/response size within limits
- [ ] Governor limits considered (callout limits)
- [ ] Idempotency for retryable operations
- [ ] Correlation IDs for tracing
- [ ] API versioning strategy defined
- [ ] Mock classes for testing
- [ ] Integration monitoring/alerting configured
- [ ] Documentation for external systems

---

**Document Owner:** Integration Team
**Next Review Date:** 2026-08-06
**Related Documents:**
- [Error Handling Standard](./error-handling.md)
- [Security & Access Control Standard](./security-access-control.md)
