# STD-002: Configuration Management Standard

**Standard ID:** STD-002
**Version:** 1.0
**Status:** Active
**Effective Date:** 2026-02-06

---

## 1. Purpose

This standard establishes guidelines for managing configurable values in Salesforce implementations. It defines when to use Custom Metadata Types versus Custom Settings, and provides implementation patterns for accessing configuration values across Apex, Validation Rules, Flows, and Formula Fields.

## 2. Scope

This standard applies to:
- Application configuration values (feature flags, thresholds, limits)
- Business rules that change infrequently
- Integration settings (endpoints, timeouts, retry counts)
- User/Profile-specific preferences
- Environment-specific values (sandbox vs production)

## 3. Decision Framework

### 3.1 When to Use Custom Metadata Types

Use **Custom Metadata Types** when:

| Criteria | Reasoning |
|----------|-----------|
| Values need to be deployed across environments | CMT records are metadata and deploy with changesets/packages |
| Values are referenced in Validation Rules | CMT is accessible in validation rule formulas |
| Values are referenced in Flows/Process Builder | Native support for CMT lookups |
| Values should be source-controlled | CMT records can be stored in version control |
| Values are application-wide (not user-specific) | CMT doesn't support hierarchy-based values |
| You need relationships between config records | CMT supports metadata relationships |
| Values rarely change in production | CMT changes require deployment or Setup access |

**Examples:**
- Feature toggle flags
- Business rule thresholds (e.g., approval limits)
- Integration endpoint configurations
- Picklist value mappings
- Field mapping configurations

> **Note:** Validation rule bypass settings are an exception - see Section 3.5.

### 3.2 When to Use Custom Settings

Use **Custom Settings** when:

| Criteria | Reasoning |
|----------|-----------|
| Values need to vary by User or Profile | Hierarchy settings cascade Org → Profile → User |
| Values change frequently in production | Can be modified via UI without deployment |
| Values need runtime modification by admins | Editable in Setup without metadata deployment |
| You need org-wide defaults with user overrides | Hierarchy provides this pattern natively |
| Performance is critical | Custom Settings are cached in application cache |

**Hierarchy Custom Settings Examples:**
- User-specific display preferences
- Profile-based feature access
- Debug/logging level per user
- Page size preferences

**List Custom Settings Examples:**
- Simple key-value lookups
- Country/currency code mappings
- Legacy configurations (prefer CMT for new development)

### 3.3 Decision Matrix

| Requirement | Custom Metadata Type | Hierarchy Custom Setting | List Custom Setting |
|-------------|---------------------|-------------------------|---------------------|
| Deployable via changeset | Yes | Data only (not values) | Data only (not values) |
| Accessible in Validation Rules | Yes | No | No |
| Accessible in Formula Fields | Yes | No | No |
| Accessible in Flows | Yes | Limited | Limited |
| Accessible in Apex | Yes | Yes | Yes |
| User/Profile specific values | No | Yes | No |
| SOQL governor limits | Not counted | Not counted | Not counted |
| Packageable with values | Yes | No (data only) | No (data only) |
| Runtime editable | Yes (Setup) | Yes (Setup/Apex) | Yes (Setup/Apex) |
| Source controllable | Yes | No | No |

### 3.4 Migration Recommendation

For existing List Custom Settings, evaluate migration to Custom Metadata Types when:
- The setting is used in validation rules or flows
- The setting needs cross-environment deployment
- The setting should be source-controlled

### 3.5 Exception: Automation Bypass Configuration

**Automation bypass settings MUST use Hierarchy Custom Settings, not Custom Metadata Types.**

| Reason | Explanation |
|--------|-------------|
| Environment-specific | Production bypass should be OFF; sandboxes may need ON for testing |
| User/Profile granularity | Different users may need different bypass levels |
| Non-deployable by design | Prevents accidentally deploying with bypass enabled |
| Runtime modification | Admins can toggle bypass without deployment |

See [STD-004: Automation Bypass Standard](./automation-bypass.md) for implementation details.

**Related exceptions that should use Custom Settings:**
- Debug/logging settings (per-user)
- Test data factory configuration (per-environment)
- Integration credentials flags (per-environment)

## 4. Custom Metadata Type Implementation

### 4.1 Naming Convention

| Component | Convention | Example |
|-----------|------------|---------|
| Object API Name | `Feature_Config__mdt` | `Integration_Config__mdt` |
| Record DeveloperName | `PascalCase_Descriptive` | `Account_Sync_Settings` |
| Field API Names | `Snake_Case__c` | `Max_Retry_Count__c` |

### 4.2 Standard Fields for Configuration CMTs

Every configuration Custom Metadata Type SHOULD include:

| Field API Name | Type | Purpose |
|----------------|------|---------|
| `MasterLabel` | Text (40) | Human-readable name (standard field) |
| `DeveloperName` | Text (40) | Unique identifier (standard field) |
| `Is_Active__c` | Checkbox | Enable/disable without deleting |
| `Description__c` | Text Area | Document the purpose |
| `Effective_Date__c` | Date | When config becomes active |
| `Expiration_Date__c` | Date | When config expires (optional) |

### 4.3 Apex Access Pattern

```apex
/**
 * Configuration service for accessing Custom Metadata Type values
 * Provides cached, null-safe access to configuration records
 */
public class ConfigurationService {

    /**
     * Get a single configuration record by DeveloperName
     * @param developerName The DeveloperName of the CMT record
     * @return The configuration record or null if not found
     */
    public static Feature_Config__mdt getFeatureConfig(String developerName) {
        return Feature_Config__mdt.getInstance(developerName);
    }

    /**
     * Get all active configuration records
     * @return Map of DeveloperName to configuration record
     */
    public static Map<String, Feature_Config__mdt> getAllActiveFeatureConfigs() {
        Map<String, Feature_Config__mdt> activeConfigs = new Map<String, Feature_Config__mdt>();

        for (Feature_Config__mdt config : Feature_Config__mdt.getAll().values()) {
            if (config.Is_Active__c && isEffective(config)) {
                activeConfigs.put(config.DeveloperName, config);
            }
        }

        return activeConfigs;
    }

    /**
     * Check if a feature is enabled
     * @param featureName The DeveloperName of the feature
     * @return true if feature exists and is active
     */
    public static Boolean isFeatureEnabled(String featureName) {
        Feature_Config__mdt config = getFeatureConfig(featureName);
        return config != null && config.Is_Active__c && isEffective(config);
    }

    /**
     * Get a configuration value with a default fallback
     * @param developerName The DeveloperName of the CMT record
     * @param fieldName The API name of the field to retrieve
     * @param defaultValue The default value if not found
     * @return The field value or default
     */
    public static Object getConfigValue(String developerName, String fieldName, Object defaultValue) {
        Feature_Config__mdt config = getFeatureConfig(developerName);

        if (config == null) {
            return defaultValue;
        }

        Object value = config.get(fieldName);
        return value != null ? value : defaultValue;
    }

    /**
     * Check if configuration is within its effective date range
     */
    private static Boolean isEffective(Feature_Config__mdt config) {
        Date today = Date.today();

        Boolean afterStart = config.Effective_Date__c == null ||
                            config.Effective_Date__c <= today;
        Boolean beforeEnd = config.Expiration_Date__c == null ||
                           config.Expiration_Date__c >= today;

        return afterStart && beforeEnd;
    }
}
```

**Usage Examples:**

```apex
// Check if a feature is enabled
if (ConfigurationService.isFeatureEnabled('Enhanced_Validation')) {
    // Execute enhanced validation logic
}

// Get a specific configuration value with default
Integer maxRetries = (Integer) ConfigurationService.getConfigValue(
    'API_Settings',
    'Max_Retry_Count__c',
    3
);

// Get full configuration record
Feature_Config__mdt config = ConfigurationService.getFeatureConfig('Batch_Settings');
if (config != null) {
    Integer batchSize = (Integer) config.Batch_Size__c;
}
```

### 4.4 Validation Rule Access

Custom Metadata Types can be referenced directly in validation rules using the `$CustomMetadata` global variable:

> **Security Note:** CMT values referenced in validation rules and formula fields are NOT subject to Field-Level Security. All users can indirectly access CMT field values through formulas, regardless of their profile permissions. Do not store sensitive data in CMT fields that are referenced in user-visible formulas. See [STD-007: Security & Access Control](./security-access-control.md).

```
// Syntax
$CustomMetadata.CustomMetadataTypeName__mdt.RecordName.FieldName__c

// Example: Prevent amounts over configured limit
Amount__c > $CustomMetadata.Approval_Limits__mdt.Standard_User.Max_Amount__c
```

**Complete Validation Rule Example:**

```
AND(
    $CustomMetadata.Feature_Config__mdt.Enforce_Amount_Limits.Is_Active__c,
    Amount__c > $CustomMetadata.Approval_Limits__mdt.Standard_Approval.Max_Amount__c,
    NOT(CONTAINS($Profile.Name, 'Admin'))
)
```

Error Message: "Amount exceeds the configured approval limit. Please submit for manager approval."

### 4.5 Flow Access

In Flows, use the **Get Records** element with Custom Metadata Types:

1. **Object**: Select the Custom Metadata Type (e.g., `Feature_Config__mdt`)
2. **Filter**: `DeveloperName Equals [ConfigName]`
3. **Store**: In a single record variable

**Example Flow Pattern:**

```
[Get Records: Feature_Config__mdt]
    Filter: DeveloperName = "Order_Processing"
    Store: {!varFeatureConfig}

[Decision: Is Feature Active?]
    Condition: {!varFeatureConfig.Is_Active__c} = true

    [True Path] → Execute Feature Logic
    [False Path] → Skip Feature
```

### 4.6 Formula Field Access

```
// Reference in formula fields
$CustomMetadata.Feature_Config__mdt.Tax_Settings.Tax_Rate__c

// Example: Calculate tax using configured rate
Amount__c * $CustomMetadata.Tax_Config__mdt.Default_Rate.Tax_Percentage__c / 100
```

## 5. Custom Settings Implementation

### 5.1 Hierarchy Custom Settings Pattern

```apex
/**
 * Service class for accessing Hierarchy Custom Settings
 * Values cascade: Organization Default → Profile → User
 */
public class UserPreferenceService {

    /**
     * Get the effective settings for the current user
     * Automatically handles the hierarchy cascade
     */
    public static User_Preferences__c getCurrentUserPreferences() {
        // getInstance() automatically returns the most specific level
        // (User → Profile → Org Default)
        return User_Preferences__c.getInstance();
    }

    /**
     * Get settings for a specific user
     */
    public static User_Preferences__c getPreferencesForUser(Id userId) {
        return User_Preferences__c.getInstance(userId);
    }

    /**
     * Get settings for a specific profile
     */
    public static User_Preferences__c getPreferencesForProfile(Id profileId) {
        return User_Preferences__c.getInstance(profileId);
    }

    /**
     * Get organization defaults
     */
    public static User_Preferences__c getOrgDefaults() {
        return User_Preferences__c.getOrgDefaults();
    }

    /**
     * Update user-specific preference
     */
    public static void updateUserPreference(String fieldName, Object value) {
        User_Preferences__c prefs = User_Preferences__c.getInstance();

        // If no user-level setting exists, create one
        if (prefs.SetupOwnerId != UserInfo.getUserId()) {
            prefs = new User_Preferences__c(SetupOwnerId = UserInfo.getUserId());
        }

        prefs.put(fieldName, value);
        upsert prefs;
    }
}
```

**Usage:**

```apex
// Get current user's preferences (cascaded)
User_Preferences__c prefs = UserPreferenceService.getCurrentUserPreferences();
Integer pageSize = prefs.Page_Size__c != null ? (Integer) prefs.Page_Size__c : 25;
Boolean showAdvanced = prefs.Show_Advanced_Features__c;

// Check if debug mode is enabled for current user
if (User_Preferences__c.getInstance().Debug_Mode__c) {
    System.debug('Debug logging enabled for user');
}
```

### 5.2 List Custom Settings Pattern

```apex
/**
 * Service for List Custom Settings (key-value lookup)
 */
public class CountryCodeService {

    private static Map<String, Country_Codes__c> codeCache;

    /**
     * Get all country codes (cached)
     */
    public static Map<String, Country_Codes__c> getAllCodes() {
        if (codeCache == null) {
            codeCache = Country_Codes__c.getAll();
        }
        return codeCache;
    }

    /**
     * Get country name by ISO code
     */
    public static String getCountryName(String isoCode) {
        Country_Codes__c code = Country_Codes__c.getInstance(isoCode);
        return code != null ? code.Country_Name__c : null;
    }

    /**
     * Check if country code is valid
     */
    public static Boolean isValidCountryCode(String isoCode) {
        return Country_Codes__c.getInstance(isoCode) != null;
    }
}
```

## 6. Example: Feature Flag Configuration Pattern

This example demonstrates a complete configuration pattern for managing feature flags.

### 6.1 Custom Metadata Type Definition

**Object:** `Feature_Flag__mdt`

| Field API Name | Type | Description |
|----------------|------|-------------|
| `MasterLabel` | Text (40) | Feature display name |
| `DeveloperName` | Text (40) | Unique feature identifier |
| `Is_Enabled__c` | Checkbox | Master on/off switch |
| `Description__c` | Long Text Area | Feature description |
| `Enabled_For_Profiles__c` | Long Text Area | Comma-separated Profile names (optional) |
| `Enabled_For_Permission_Sets__c` | Long Text Area | Comma-separated Permission Set names |
| `Start_Date__c` | Date | When feature becomes available |
| `End_Date__c` | Date | When feature is disabled (sunset) |
| `Rollout_Percentage__c` | Percent | Gradual rollout percentage (0-100) |

### 6.2 Feature Flag Service

```apex
/**
 * Centralized service for feature flag evaluation
 * Supports profile-based, permission-based, and percentage-based rollouts
 */
public class FeatureFlagService {

    // Cache for performance (cleared on CMT updates via Platform Cache if needed)
    private static Map<String, Feature_Flag__mdt> flagCache;
    private static Map<String, Boolean> evaluationCache = new Map<String, Boolean>();

    /**
     * Primary method: Check if a feature is enabled for the current user
     * @param featureName DeveloperName of the feature flag
     * @return true if feature is enabled for current user context
     */
    public static Boolean isEnabled(String featureName) {
        // Check evaluation cache first (per-transaction)
        String cacheKey = featureName + '_' + UserInfo.getUserId();
        if (evaluationCache.containsKey(cacheKey)) {
            return evaluationCache.get(cacheKey);
        }

        Feature_Flag__mdt flag = getFlag(featureName);
        Boolean result = evaluateFlag(flag);

        evaluationCache.put(cacheKey, result);
        return result;
    }

    /**
     * Check if feature is enabled (simple boolean, no user context)
     */
    public static Boolean isGloballyEnabled(String featureName) {
        Feature_Flag__mdt flag = getFlag(featureName);
        return flag != null && flag.Is_Enabled__c && isWithinDateRange(flag);
    }

    /**
     * Get all enabled features for current user
     */
    public static Set<String> getEnabledFeatures() {
        Set<String> enabled = new Set<String>();

        for (Feature_Flag__mdt flag : getAllFlags().values()) {
            if (evaluateFlag(flag)) {
                enabled.add(flag.DeveloperName);
            }
        }

        return enabled;
    }

    // Private helper methods

    private static Feature_Flag__mdt getFlag(String featureName) {
        return getAllFlags().get(featureName);
    }

    private static Map<String, Feature_Flag__mdt> getAllFlags() {
        if (flagCache == null) {
            flagCache = new Map<String, Feature_Flag__mdt>();
            for (Feature_Flag__mdt flag : Feature_Flag__mdt.getAll().values()) {
                flagCache.put(flag.DeveloperName, flag);
            }
        }
        return flagCache;
    }

    private static Boolean evaluateFlag(Feature_Flag__mdt flag) {
        // Null or disabled = false
        if (flag == null || !flag.Is_Enabled__c) {
            return false;
        }

        // Check date range
        if (!isWithinDateRange(flag)) {
            return false;
        }

        // Check profile restrictions
        if (String.isNotBlank(flag.Enabled_For_Profiles__c)) {
            if (!isUserInProfiles(flag.Enabled_For_Profiles__c)) {
                return false;
            }
        }

        // Check permission set restrictions
        if (String.isNotBlank(flag.Enabled_For_Permission_Sets__c)) {
            if (!userHasPermissionSets(flag.Enabled_For_Permission_Sets__c)) {
                return false;
            }
        }

        // Check rollout percentage
        if (flag.Rollout_Percentage__c != null && flag.Rollout_Percentage__c < 100) {
            if (!isUserInRolloutPercentage(flag.Rollout_Percentage__c)) {
                return false;
            }
        }

        return true;
    }

    private static Boolean isWithinDateRange(Feature_Flag__mdt flag) {
        Date today = Date.today();

        if (flag.Start_Date__c != null && today < flag.Start_Date__c) {
            return false;
        }

        if (flag.End_Date__c != null && today > flag.End_Date__c) {
            return false;
        }

        return true;
    }

    private static Boolean isUserInProfiles(String profileList) {
        Set<String> allowedProfiles = new Set<String>();
        for (String p : profileList.split(',')) {
            allowedProfiles.add(p.trim().toLowerCase());
        }

        String currentProfile = [SELECT Name FROM Profile WHERE Id = :UserInfo.getProfileId()].Name;
        return allowedProfiles.contains(currentProfile.toLowerCase());
    }

    private static Boolean userHasPermissionSets(String permSetList) {
        Set<String> requiredPermSets = new Set<String>();
        for (String ps : permSetList.split(',')) {
            requiredPermSets.add(ps.trim());
        }

        Set<String> userPermSets = new Set<String>();
        for (PermissionSetAssignment psa : [
            SELECT PermissionSet.Name
            FROM PermissionSetAssignment
            WHERE AssigneeId = :UserInfo.getUserId()
        ]) {
            userPermSets.add(psa.PermissionSet.Name);
        }

        // User must have at least one of the required permission sets
        userPermSets.retainAll(requiredPermSets);
        return !userPermSets.isEmpty();
    }

    private static Boolean isUserInRolloutPercentage(Decimal percentage) {
        // Use consistent hashing based on User ID for stable rollout
        String userIdHash = EncodingUtil.convertToHex(
            Crypto.generateDigest('MD5', Blob.valueOf(UserInfo.getUserId()))
        );

        // Take first 8 hex chars and convert to number (0 to 4294967295)
        Long hashValue = Long.valueOf(userIdHash.substring(0, 8), 16);

        // Convert to percentage (0-100)
        Decimal userPercentile = Math.mod(hashValue, 100);

        return userPercentile < percentage;
    }
}
```

### 6.3 Usage in Apex

```apex
public class OrderController {

    public void processOrder(Order__c order) {
        // Check feature flag before executing new logic
        if (FeatureFlagService.isEnabled('Enhanced_Order_Validation')) {
            validateOrderEnhanced(order);
        } else {
            validateOrderLegacy(order);
        }

        // Feature flag for beta features
        if (FeatureFlagService.isEnabled('AI_Recommendations_Beta')) {
            order.Recommended_Products__c = getAIRecommendations(order);
        }
    }
}
```

### 6.4 Usage in Validation Rule

```
// Only enforce new validation if feature is enabled
AND(
    $CustomMetadata.Feature_Flag__mdt.Strict_Address_Validation.Is_Enabled__c,
    ISBLANK(BillingStreet),
    NOT(ISBLANK(BillingCity))
)
```

### 6.5 Usage in Flow

```
[Get Records: Feature_Flag__mdt]
    Filter: DeveloperName = "Auto_Case_Assignment"
    Store: {!varFeatureFlag}

[Decision: Feature Enabled?]
    Condition: {!varFeatureFlag.Is_Enabled__c} = true

    [True] → [Assignment Logic]
    [False] → [Skip Assignment]
```

## 7. Testing Configuration

### 7.1 Testing Custom Metadata Types

Custom Metadata Types are visible in tests without `@IsTest(SeeAllData=true)`:

```apex
@IsTest
private class FeatureFlagServiceTest {

    @IsTest
    static void testFeatureEnabled() {
        // CMT records are accessible in tests
        // To test different scenarios, use dependency injection or test-specific logic

        Test.startTest();
        Boolean result = FeatureFlagService.isGloballyEnabled('Known_Feature');
        Test.stopTest();

        // Assert based on actual CMT data in your org/package
        System.assertNotEquals(null, result);
    }

    @IsTest
    static void testFeatureNotFound() {
        Test.startTest();
        Boolean result = FeatureFlagService.isEnabled('NonExistent_Feature');
        Test.stopTest();

        System.assertEquals(false, result, 'Non-existent feature should return false');
    }
}
```

### 7.2 Testing Custom Settings

```apex
@IsTest
private class UserPreferenceServiceTest {

    @TestSetup
    static void setup() {
        // Create org defaults
        User_Preferences__c orgDefaults = new User_Preferences__c(
            SetupOwnerId = UserInfo.getOrganizationId(),
            Page_Size__c = 25,
            Debug_Mode__c = false
        );
        insert orgDefaults;
    }

    @IsTest
    static void testGetCurrentUserPreferences() {
        Test.startTest();
        User_Preferences__c prefs = UserPreferenceService.getCurrentUserPreferences();
        Test.stopTest();

        System.assertEquals(25, prefs.Page_Size__c);
    }

    @IsTest
    static void testUserOverride() {
        // Create user-level override
        User_Preferences__c userPrefs = new User_Preferences__c(
            SetupOwnerId = UserInfo.getUserId(),
            Page_Size__c = 50
        );
        insert userPrefs;

        Test.startTest();
        User_Preferences__c prefs = UserPreferenceService.getCurrentUserPreferences();
        Test.stopTest();

        System.assertEquals(50, prefs.Page_Size__c, 'User override should take precedence');
    }
}
```

## 8. Compliance Checklist

Before implementing configuration:

- [ ] Determined appropriate storage mechanism (CMT vs Custom Setting)
- [ ] Followed naming conventions
- [ ] Included standard fields (Is_Active, Description, Effective Dates)
- [ ] Created service class for access
- [ ] Documented all configuration records
- [ ] Created test coverage
- [ ] Validated formula/validation rule syntax
- [ ] Verified deployment to target environments

---

**Document Owner:** Platform Team
**Next Review Date:** 2026-08-06
**Related Documents:** [Error Handling Standard](./error-handling.md)
