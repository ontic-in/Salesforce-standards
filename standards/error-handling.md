# STD-001: Error Handling Standard

**Standard ID:** STD-001
**Version:** 1.0
**Status:** Active
**Effective Date:** 2026-02-04

---

## 1. Purpose

This standard establishes a centralized error handling pattern for all Salesforce implementations. It ensures that errors from Flows, Integrations, and Apex are captured, logged, and made visible to users and administrators for troubleshooting and resolution.

## 2. Scope

This standard applies to:
- All Flow automations (Screen Flows, Record-Triggered Flows, Scheduled Flows, Platform Event Flows)
- All Integration processes (REST/SOAP callouts, Platform Events, External Services)
- All Apex code (Triggers, Classes, Batch Jobs, Queueable, Schedulable)

## 3. Custom Object Specification

### 3.1 Object Definition

**Object Name:** `Error_Log__c`
**Label:** Error Log
**Plural Label:** Error Logs
**Description:** Centralized repository for all system errors across automations and integrations

### 3.2 Required Fields

| Field API Name | Label | Data Type | Length/Values | Required | Description |
|----------------|-------|-----------|---------------|----------|-------------|
| `Name` | Error Log Number | Auto Number | ERR-{00000} | Yes | Auto-generated unique identifier |
| `Error_Type__c` | Error Type | Picklist | Flow, Apex, Integration, Batch, Trigger, Scheduled Job | Yes | Category of the error source |
| `Error_Source__c` | Error Source | Text | 255 | Yes | Name of the Flow, Class, or Integration that generated the error |
| `Error_Message__c` | Error Message | Long Text Area | 32,000 | Yes | Full error message returned by the system |
| `Stack_Trace__c` | Stack Trace | Long Text Area | 32,000 | No | Technical stack trace for Apex errors |
| `Related_Record_Id__c` | Related Record ID | Text | 18 | No | Salesforce ID of the record that caused the error |
| `Related_Record_Link__c` | Related Record | Formula (Text) | - | No | Hyperlink formula to the related record |
| `Related_Object__c` | Related Object | Text | 255 | No | API name of the related object |
| `User__c` | User | Lookup (User) | - | No | User context when the error occurred |
| `Error_DateTime__c` | Error Date/Time | DateTime | - | Yes | Timestamp when the error occurred |
| `Severity__c` | Severity | Picklist | Critical, High, Medium, Low, Info | Yes | Impact level of the error |
| `Status__c` | Status | Picklist | New, In Progress, Resolved, Ignored | Yes | Current resolution status |
| `Resolution_Notes__c` | Resolution Notes | Long Text Area | 10,000 | No | Notes on how the error was resolved |
| `Resolved_By__c` | Resolved By | Lookup (User) | - | No | User who resolved the error |
| `Resolved_DateTime__c` | Resolved Date/Time | DateTime | - | No | When the error was resolved |
| `Request_Payload__c` | Request Payload | Long Text Area | 131,072 | No | For integrations: the request body/parameters |
| `Response_Payload__c` | Response Payload | Long Text Area | 131,072 | No | For integrations: the response body |
| `HTTP_Status_Code__c` | HTTP Status Code | Number | 3,0 | No | HTTP response code for integration errors |
| `Endpoint__c` | Endpoint | URL | - | No | Integration endpoint URL |
| `Mitigation_Steps__c` | Recommended Mitigation | Long Text Area | 10,000 | No | Suggested steps to resolve the error |
| `Error_Hash__c` | Error Hash | Text | 255 | No | Hash of error signature for duplicate detection |
| `Occurrence_Count__c` | Occurrence Count | Number | 10,0 | No | Number of times this error has occurred |
| `First_Occurrence__c` | First Occurrence | DateTime | - | No | When this error pattern first appeared |
| `Last_Occurrence__c` | Last Occurrence | DateTime | - | No | Most recent occurrence of this error pattern |

> **Security Note - Sensitive Fields:** The `Request_Payload__c` and `Response_Payload__c` fields may contain sensitive data (API credentials, tokens, PII). These fields MUST:
> 1. Be restricted via FLS to System Administrator and Integration team profiles only
> 2. Have data masking applied before logging (see Section 4.5)
> 3. Never contain unmasked passwords, tokens, or credentials
>
> See [STD-007: Security & Access Control](./security-access-control.md) for data masking patterns.

### 3.3 Formula Field: Related Record Link

```
IF(
    AND(
        NOT(ISBLANK(Related_Record_Id__c)),
        NOT(ISBLANK(Related_Object__c))
    ),
    HYPERLINK('/' & Related_Record_Id__c, Related_Object__c & ': ' & Related_Record_Id__c, '_blank'),
    ''
)
```

### 3.4 Validation Rules

**Error Type Required for Non-Manual Entries:**
```
AND(
    ISBLANK(TEXT(Error_Type__c)),
    NOT($User.Alias = 'admin')
)
```

**Resolved Fields Required When Status is Resolved:**
```
AND(
    ISPICKVAL(Status__c, 'Resolved'),
    OR(
        ISBLANK(Resolution_Notes__c),
        ISBLANK(Resolved_By__c),
        ISBLANK(Resolved_DateTime__c)
    )
)
```

### 3.5 Page Layouts

Create the following page layouts:
1. **Error Log - Admin Layout**: All fields visible for system administrators
2. **Error Log - User Layout**: Key fields only (hide Stack Trace, Payloads)

### 3.6 List Views

| View Name | Filter Criteria | Columns |
|-----------|-----------------|---------|
| All Open Errors | Status != Resolved, Status != Ignored | Name, Error Type, Error Source, Severity, Error DateTime, Related Record Link |
| Critical Errors | Severity = Critical, Status = New | Name, Error Source, Error Message, Related Record Link, Error DateTime |
| My Errors | User = $CurrentUser | Name, Error Type, Error Source, Error Message, Status |
| Today's Errors | Error DateTime = TODAY | Name, Error Type, Severity, Error Source, Error Message |
| Integration Errors | Error Type = Integration | Name, Endpoint, HTTP Status Code, Error Message, Status |

## 4. Implementation Requirements

### 4.1 Flow Error Handling

All Flows MUST implement fault paths that log errors to Error_Log__c:

**Required Fault Path Elements:**
1. Fault connector from each element that can fail
2. Create Records element to create Error_Log__c record
3. Populate all applicable fields

**Flow Error Log Field Mapping:**

| Error_Log__c Field | Flow Value |
|--------------------|------------|
| Error_Type__c | "Flow" |
| Error_Source__c | {!$Flow.CurrentLabel} or hardcoded Flow name |
| Error_Message__c | {!$Flow.FaultMessage} |
| Related_Record_Id__c | {!recordId} or triggering record ID |
| Related_Object__c | Object API name |
| User__c | {!$User.Id} |
| Error_DateTime__c | {!$Flow.CurrentDateTime} |
| Severity__c | Based on business impact |
| Status__c | "New" |

### 4.2 Apex Error Handling

All Apex code MUST use try-catch blocks and log errors:

**Required Pattern:**
```apex
public class ErrorLogger {

    public static void logError(
        String errorType,
        String errorSource,
        String errorMessage,
        String stackTrace,
        Id relatedRecordId,
        String relatedObject,
        String severity,
        String mitigationSteps
    ) {
        Error_Log__c errorLog = new Error_Log__c(
            Error_Type__c = errorType,
            Error_Source__c = errorSource,
            Error_Message__c = errorMessage,
            Stack_Trace__c = stackTrace,
            Related_Record_Id__c = relatedRecordId,
            Related_Object__c = relatedObject,
            User__c = UserInfo.getUserId(),
            Error_DateTime__c = DateTime.now(),
            Severity__c = severity,
            Status__c = 'New',
            Mitigation_Steps__c = mitigationSteps
        );

        // Use future method or platform event to avoid mixed DML
        insertErrorLog(errorLog);
    }

    public static void logException(Exception e, String source, Id relatedRecordId, String relatedObject) {
        logError(
            'Apex',
            source,
            e.getMessage(),
            e.getStackTraceString(),
            relatedRecordId,
            relatedObject,
            'High',
            null
        );
    }

    public static void logIntegrationError(
        String source,
        String endpoint,
        Integer httpStatusCode,
        String requestPayload,
        String responsePayload,
        String errorMessage,
        Id relatedRecordId
    ) {
        Error_Log__c errorLog = new Error_Log__c(
            Error_Type__c = 'Integration',
            Error_Source__c = source,
            Error_Message__c = errorMessage,
            Endpoint__c = endpoint,
            HTTP_Status_Code__c = httpStatusCode,
            Request_Payload__c = requestPayload,
            Response_Payload__c = responsePayload,
            Related_Record_Id__c = relatedRecordId,
            User__c = UserInfo.getUserId(),
            Error_DateTime__c = DateTime.now(),
            Severity__c = httpStatusCode >= 500 ? 'Critical' : 'High',
            Status__c = 'New'
        );

        insertErrorLog(errorLog);
    }

    @future
    private static void insertErrorLog(Error_Log__c errorLog) {
        try {
            insert errorLog;
        } catch (Exception e) {
            System.debug(LoggingLevel.ERROR, 'Failed to insert error log: ' + e.getMessage());
        }
    }
}
```

**Usage in Apex Classes:**
```apex
public class AccountService {

    public void processAccount(Id accountId) {
        try {
            Account acc = [SELECT Id, Name FROM Account WHERE Id = :accountId];
            // Business logic here

        } catch (QueryException e) {
            ErrorLogger.logException(e, 'AccountService.processAccount', accountId, 'Account');
            throw new AccountServiceException('Failed to process account: ' + e.getMessage());

        } catch (DmlException e) {
            ErrorLogger.logException(e, 'AccountService.processAccount', accountId, 'Account');
            throw new AccountServiceException('Failed to save account changes: ' + e.getMessage());
        }
    }
}
```

### 4.3 Integration Error Handling

All integration callouts MUST log failures:

```apex
public class ExternalAPIService {

    private static final String ENDPOINT = 'https://api.example.com/v1/';

    public HttpResponse makeCallout(String endpoint, String method, String body, Id relatedRecordId) {
        HttpRequest req = new HttpRequest();
        req.setEndpoint(ENDPOINT + endpoint);
        req.setMethod(method);
        req.setBody(body);
        req.setHeader('Content-Type', 'application/json');

        Http http = new Http();
        HttpResponse res;

        try {
            res = http.send(req);

            if (res.getStatusCode() >= 400) {
                ErrorLogger.logIntegrationError(
                    'ExternalAPIService',
                    ENDPOINT + endpoint,
                    res.getStatusCode(),
                    body,
                    res.getBody(),
                    'HTTP Error: ' + res.getStatus(),
                    relatedRecordId
                );
            }

        } catch (CalloutException e) {
            ErrorLogger.logIntegrationError(
                'ExternalAPIService',
                ENDPOINT + endpoint,
                0,
                body,
                null,
                'Callout Exception: ' + e.getMessage(),
                relatedRecordId
            );
            throw e;
        }

        return res;
    }
}
```

### 4.4 Batch Job Error Handling

```apex
public class AccountBatchProcessor implements Database.Batchable<sObject>, Database.Stateful {

    private List<Error_Log__c> errorLogs = new List<Error_Log__c>();

    public Database.QueryLocator start(Database.BatchableContext bc) {
        return Database.getQueryLocator('SELECT Id, Name FROM Account');
    }

    public void execute(Database.BatchableContext bc, List<Account> scope) {
        for (Account acc : scope) {
            try {
                // Process account
            } catch (Exception e) {
                errorLogs.add(new Error_Log__c(
                    Error_Type__c = 'Batch',
                    Error_Source__c = 'AccountBatchProcessor',
                    Error_Message__c = e.getMessage(),
                    Stack_Trace__c = e.getStackTraceString(),
                    Related_Record_Id__c = acc.Id,
                    Related_Object__c = 'Account',
                    Error_DateTime__c = DateTime.now(),
                    Severity__c = 'Medium',
                    Status__c = 'New'
                ));
            }
        }
    }

    public void finish(Database.BatchableContext bc) {
        if (!errorLogs.isEmpty()) {
            insert errorLogs;
        }
    }
}
```

### 4.5 Payload Data Masking (Required for Integration Errors)

Before logging integration payloads, sensitive data MUST be masked:

```apex
public class PayloadMasker {

    private static final Set<String> SENSITIVE_KEYS = new Set<String>{
        'password', 'token', 'apikey', 'api_key', 'secret',
        'authorization', 'bearer', 'credential', 'ssn', 'credit_card'
    };

    /**
     * Mask sensitive values in JSON payload
     */
    public static String maskPayload(String payload) {
        if (String.isBlank(payload)) {
            return payload;
        }

        String masked = payload;
        for (String key : SENSITIVE_KEYS) {
            // Mask JSON values: "key": "value" -> "key": "***MASKED***"
            masked = masked.replaceAll(
                '(?i)"' + key + '"\\s*:\\s*"[^"]*"',
                '"' + key + '": "***MASKED***"'
            );
        }
        return masked;
    }
}

// Usage in ErrorLogger.logIntegrationError:
// Request_Payload__c = PayloadMasker.maskPayload(requestPayload),
// Response_Payload__c = PayloadMasker.maskPayload(responsePayload),
```

**Never log unmasked:**
- API keys, tokens, or credentials
- Passwords or secrets
- PII (SSN, credit card numbers, etc.)
- Authorization headers

## 5. Monitoring and Alerts

### 5.1 Required Reports

Create the following reports:
1. **Daily Error Summary**: Count of errors by Type and Severity for the current day
2. **Error Trend Analysis**: Error counts over time by Error Source
3. **Unresolved Critical Errors**: All Critical severity errors with Status = New
4. **Integration Health Dashboard**: Integration errors by Endpoint with success/failure rates

### 5.2 Recommended Alerts

Configure Flow-based alerts or custom notifications for:
- Any Critical severity error
- More than 10 errors from the same source within 1 hour
- Integration errors with HTTP 5xx status codes

## 6. Data Retention

| Severity | Retention Period | Action |
|----------|-----------------|--------|
| Critical | 1 year | Archive after resolution |
| High | 6 months | Delete after resolution + 30 days |
| Medium | 3 months | Delete after resolution + 14 days |
| Low/Info | 1 month | Delete after resolution + 7 days |

Implement a scheduled Apex job to enforce retention policies.

## 7. Security Considerations

### 7.1 Field-Level Security

| Profile Type | Access Level |
|--------------|--------------|
| System Administrator | Full Read/Write |
| Integration User | Create only (no Read) |
| Support User | Read/Edit Status and Resolution fields |
| Standard User | Read only (filtered by User__c) |

### 7.2 Sharing Settings

- **OWD**: Private
- **Sharing Rules**: Share with Support team for their assigned users' errors
- **Apex Sharing**: Consider programmatic sharing based on business requirements

## 8. Compliance Checklist

Before deployment, verify:

- [ ] Error_Log__c object created with all required fields
- [ ] All page layouts configured
- [ ] All list views created
- [ ] Field-level security configured
- [ ] Sharing settings configured
- [ ] ErrorLogger Apex class deployed
- [ ] All existing Flows updated with fault paths
- [ ] All Apex classes updated with try-catch and logging
- [ ] All integration classes updated with error logging
- [ ] Reports and dashboards created
- [ ] Alert notifications configured
- [ ] Data retention job scheduled

---

**Document Owner:** Platform Team
**Next Review Date:** 2026-08-04
**Related Documents:** [Error Messages Best Practices](../best-practices/error-messages.md)
