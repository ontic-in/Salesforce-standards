# STD-012: Lightning Component Standards

**Standard ID:** STD-012
**Version:** 1.0
**Status:** Active
**Effective Date:** 2026-02-06

---

## 1. Purpose

This standard establishes development guidelines for Lightning Web Components (LWC) and Aura Components. It ensures consistent, performant, accessible, and maintainable UI components across Salesforce implementations.

## 2. Scope

This standard applies to:
- Lightning Web Components
- Aura Components (legacy)
- Lightning App Builder components
- Quick Actions
- Custom Lightning Pages
- Utility Bar items

## 3. Framework Selection

### 3.1 LWC vs Aura Decision

| Use LWC When | Use Aura When |
|--------------|---------------|
| New development (always prefer) | Extending existing Aura components |
| Better performance needed | Using Aura-only features (events) |
| Modern web standards preferred | Complex container requirements |
| Reusable components | Legacy system compatibility |

**Default Choice:** Always use LWC for new development.

### 3.2 Component Types

| Type | Use Case |
|------|----------|
| Record Page Component | Display/edit record-specific data |
| App Page Component | Standalone functionality |
| Home Page Component | Dashboard/summary views |
| Utility Bar | Quick access tools |
| Quick Action | Contextual record actions |
| Flow Screen | Flow integration |

## 4. LWC Project Structure

### 4.1 Component Folder Structure

```
force-app/main/default/lwc/
├── accountList/
│   ├── accountList.html
│   ├── accountList.js
│   ├── accountList.css
│   ├── accountList.js-meta.xml
│   └── __tests__/
│       └── accountList.test.js
├── contactCard/
│   ├── contactCard.html
│   ├── contactCard.js
│   ├── contactCard.css
│   └── contactCard.js-meta.xml
└── shared/
    ├── utils/
    │   └── formatters.js
    └── constants/
        └── labels.js
```

### 4.2 Naming Conventions

```javascript
// Component folders: camelCase
accountList/
contactDetailCard/
opportunityLineItems/

// JavaScript files: camelCase (matches folder)
accountList.js

// CSS files: camelCase (matches folder)
accountList.css

// Test files: componentName.test.js
accountList.test.js

// Class names: PascalCase
export default class AccountList extends LightningElement { }

// Methods: camelCase, verb prefix
handleClick() { }
loadAccountData() { }
validateInput() { }

// Properties: camelCase
@api recordId;
@track isLoading;
accountName;

// Constants: UPPER_SNAKE_CASE
const MAX_RECORDS = 50;
const DEFAULT_PAGE_SIZE = 10;

// Private properties: underscore prefix (optional convention)
_internalState = {};
```

## 5. LWC JavaScript Patterns

### 5.1 Component Template

```javascript
/**
 * @description Display a list of accounts with filtering
 * @author Author Name
 * @date YYYY-MM-DD
 */
import { LightningElement, api, wire, track } from 'lwc';
import { ShowToastEvent } from 'lightning/platformShowToastEvent';
import { NavigationMixin } from 'lightning/navigation';
import getAccounts from '@salesforce/apex/AccountController.getAccounts';

// Constants
const COLUMNS = [
    { label: 'Name', fieldName: 'Name', type: 'text' },
    { label: 'Industry', fieldName: 'Industry', type: 'text' },
    { label: 'Annual Revenue', fieldName: 'AnnualRevenue', type: 'currency' }
];
const PAGE_SIZE = 10;

export default class AccountList extends NavigationMixin(LightningElement) {

    // ============================================================
    // PUBLIC PROPERTIES (@api)
    // ============================================================
    @api recordId;
    @api objectApiName;

    // ============================================================
    // TRACKED PROPERTIES (@track)
    // ============================================================
    @track accounts = [];

    // ============================================================
    // PRIVATE PROPERTIES
    // ============================================================
    isLoading = false;
    error;
    columns = COLUMNS;

    // ============================================================
    // GETTERS
    // ============================================================
    get hasAccounts() {
        return this.accounts && this.accounts.length > 0;
    }

    get accountCount() {
        return this.accounts ? this.accounts.length : 0;
    }

    // ============================================================
    // LIFECYCLE HOOKS
    // ============================================================
    connectedCallback() {
        this.loadAccounts();
    }

    disconnectedCallback() {
        // Cleanup if needed
    }

    // ============================================================
    // WIRE ADAPTERS
    // ============================================================
    @wire(getAccounts, { recordId: '$recordId' })
    wiredAccounts({ error, data }) {
        if (data) {
            this.accounts = data;
            this.error = undefined;
        } else if (error) {
            this.error = this.reduceErrors(error);
            this.accounts = [];
        }
        this.isLoading = false;
    }

    // ============================================================
    // EVENT HANDLERS
    // ============================================================
    handleRowAction(event) {
        const actionName = event.detail.action.name;
        const row = event.detail.row;

        switch (actionName) {
            case 'view':
                this.navigateToRecord(row.Id);
                break;
            case 'edit':
                this.editRecord(row.Id);
                break;
            default:
                break;
        }
    }

    handleRefresh() {
        this.loadAccounts();
    }

    // ============================================================
    // PRIVATE METHODS
    // ============================================================
    loadAccounts() {
        this.isLoading = true;
        // Wire adapter handles loading
    }

    navigateToRecord(recordId) {
        this[NavigationMixin.Navigate]({
            type: 'standard__recordPage',
            attributes: {
                recordId: recordId,
                objectApiName: 'Account',
                actionName: 'view'
            }
        });
    }

    showToast(title, message, variant) {
        this.dispatchEvent(
            new ShowToastEvent({
                title: title,
                message: message,
                variant: variant
            })
        );
    }

    reduceErrors(errors) {
        if (!Array.isArray(errors)) {
            errors = [errors];
        }

        return errors
            .filter(error => !!error)
            .map(error => {
                if (Array.isArray(error.body)) {
                    return error.body.map(e => e.message);
                } else if (error.body && error.body.message) {
                    return error.body.message;
                } else if (typeof error.message === 'string') {
                    return error.message;
                }
                return 'Unknown error';
            })
            .flat()
            .join(', ');
    }
}
```

### 5.2 Wire vs Imperative Apex

```javascript
// Wire: Reactive, automatic refresh
// Use when: Data should refresh when parameters change
@wire(getAccounts, { searchTerm: '$searchTerm' })
wiredAccounts;

// Imperative: Manual control
// Use when: Need to control when data loads
async loadAccounts() {
    this.isLoading = true;
    try {
        this.accounts = await getAccounts({ searchTerm: this.searchTerm });
        this.error = undefined;
    } catch (error) {
        this.error = this.reduceErrors(error);
        this.accounts = [];
    } finally {
        this.isLoading = false;
    }
}
```

### 5.3 Event Communication

```javascript
// Child Component: Dispatch event
handleSelection() {
    const selectedEvent = new CustomEvent('accountselected', {
        detail: { accountId: this.selectedAccountId }
    });
    this.dispatchEvent(selectedEvent);
}

// Parent Component: Handle event
// In HTML: <c-child-component onaccountselected={handleAccountSelected}></c-child-component>
handleAccountSelected(event) {
    const accountId = event.detail.accountId;
    this.processSelectedAccount(accountId);
}
```

### 5.4 Public Methods

```javascript
// Child component exposes method
@api
refresh() {
    return this.loadData();
}

@api
validate() {
    const isValid = this.checkValidity();
    if (!isValid) {
        this.reportValidity();
    }
    return isValid;
}

// Parent calls child method
handleSubmit() {
    const childComponent = this.template.querySelector('c-child-component');
    if (childComponent.validate()) {
        // Proceed
    }
}
```

## 6. LWC HTML Patterns

### 6.1 Template Structure

```html
<template>
    <!-- Loading Spinner -->
    <template if:true={isLoading}>
        <lightning-spinner alternative-text="Loading" size="medium"></lightning-spinner>
    </template>

    <!-- Error Display -->
    <template if:true={error}>
        <div class="slds-text-color_error slds-p-around_medium">
            <lightning-icon icon-name="utility:error" alternative-text="Error" size="small"></lightning-icon>
            <span class="slds-p-left_x-small">{error}</span>
        </div>
    </template>

    <!-- Main Content -->
    <template if:false={isLoading}>
        <template if:true={hasAccounts}>
            <lightning-card title="Accounts" icon-name="standard:account">
                <lightning-button
                    slot="actions"
                    label="Refresh"
                    icon-name="utility:refresh"
                    onclick={handleRefresh}>
                </lightning-button>

                <lightning-datatable
                    key-field="Id"
                    data={accounts}
                    columns={columns}
                    onrowaction={handleRowAction}>
                </lightning-datatable>
            </lightning-card>
        </template>

        <template if:false={hasAccounts}>
            <div class="slds-illustration slds-illustration_small">
                <div class="slds-text-longform">
                    <h3 class="slds-text-heading_medium">No accounts found</h3>
                </div>
            </div>
        </template>
    </template>
</template>
```

### 6.2 Iteration

```html
<!-- Simple iteration -->
<template for:each={accounts} for:item="account">
    <div key={account.Id} class="slds-p-around_small">
        {account.Name}
    </div>
</template>

<!-- Iteration with index -->
<template for:each={accounts} for:item="account" for:index="index">
    <div key={account.Id}>
        {index}: {account.Name}
    </div>
</template>

<!-- Iterator for first/last -->
<template iterator:it={accounts}>
    <div key={it.value.Id} class={it.first ? 'first-item' : ''}>
        {it.value.Name}
    </div>
</template>
```

### 6.3 Conditional Rendering

```html
<!-- if:true/if:false -->
<template if:true={isVisible}>
    <div>Visible content</div>
</template>

<!-- lwc:if/lwc:elseif/lwc:else (preferred in newer API versions) -->
<template lwc:if={isAdmin}>
    <c-admin-panel></c-admin-panel>
</template>
<template lwc:elseif={isManager}>
    <c-manager-panel></c-manager-panel>
</template>
<template lwc:else>
    <c-user-panel></c-user-panel>
</template>
```

## 7. LWC CSS Patterns

### 7.1 Component Styling

```css
/* Component-scoped styles */
:host {
    display: block;
}

/* Use SLDS classes when possible */
.container {
    padding: var(--lwc-spacingMedium);
}

/* Custom properties for theming */
.custom-header {
    color: var(--lwc-colorTextDefault);
    font-size: var(--lwc-fontSizeLarge);
}

/* Responsive design */
@media (max-width: 768px) {
    .desktop-only {
        display: none;
    }
}

/* Avoid !important */
/* Avoid global selectors */
/* Avoid element selectors - use classes */
```

### 7.2 SLDS Integration

```html
<!-- Use SLDS utility classes -->
<div class="slds-p-around_medium slds-m-bottom_small">
    <span class="slds-text-heading_small slds-text-color_weak">
        Subtitle
    </span>
</div>

<!-- Use SLDS components -->
<lightning-button variant="brand" label="Save"></lightning-button>
<lightning-icon icon-name="utility:check" size="small"></lightning-icon>
```

## 8. Meta Configuration

### 8.1 Component Meta XML

```xml
<?xml version="1.0" encoding="UTF-8"?>
<LightningComponentBundle xmlns="http://soap.sforce.com/2006/04/metadata">
    <apiVersion>59.0</apiVersion>
    <isExposed>true</isExposed>
    <masterLabel>Account List</masterLabel>
    <description>Displays a filterable list of accounts</description>

    <targets>
        <target>lightning__RecordPage</target>
        <target>lightning__AppPage</target>
        <target>lightning__HomePage</target>
        <target>lightning__FlowScreen</target>
    </targets>

    <targetConfigs>
        <targetConfig targets="lightning__RecordPage">
            <objects>
                <object>Account</object>
            </objects>
            <property name="title" type="String" label="Card Title" default="Related Accounts"/>
            <property name="maxRecords" type="Integer" label="Max Records" default="10"/>
        </targetConfig>

        <targetConfig targets="lightning__FlowScreen">
            <property name="selectedAccountId" type="String" label="Selected Account ID" role="outputOnly"/>
        </targetConfig>
    </targetConfigs>
</LightningComponentBundle>
```

### 8.2 Design Attributes

```xml
<!-- For App Builder configuration -->
<targetConfig targets="lightning__AppPage,lightning__RecordPage">
    <!-- Text input -->
    <property name="title"
              type="String"
              label="Title"
              default="My Component"
              description="The title displayed in the header"/>

    <!-- Number input -->
    <property name="pageSize"
              type="Integer"
              label="Page Size"
              default="10"
              min="5"
              max="100"/>

    <!-- Boolean toggle -->
    <property name="showHeader"
              type="Boolean"
              label="Show Header"
              default="true"/>

    <!-- Picklist -->
    <property name="displayMode"
              type="String"
              label="Display Mode"
              datasource="Compact,Standard,Expanded"
              default="Standard"/>

    <!-- Apex data source -->
    <property name="selectedRecordId"
              type="String"
              label="Record"
              datasource="apex://MyDataSourceProvider"/>
</targetConfig>
```

## 9. Aura Component Guidelines

### 9.1 Aura Structure

```
accountListAura/
├── accountListAura.cmp        (markup)
├── accountListAura.css        (styles)
├── accountListAuraController.js    (client controller)
├── accountListAuraHelper.js   (helper functions)
├── accountListAura.design     (App Builder config)
├── accountListAura.auradoc    (documentation)
└── accountListAura.svg        (icon)
```

### 9.2 Aura Component Pattern

```html
<!-- accountListAura.cmp -->
<aura:component controller="AccountController"
                implements="flexipage:availableForAllPageTypes,force:hasRecordId"
                access="global">

    <!-- Attributes -->
    <aura:attribute name="recordId" type="Id"/>
    <aura:attribute name="accounts" type="Account[]"/>
    <aura:attribute name="isLoading" type="Boolean" default="false"/>

    <!-- Handlers -->
    <aura:handler name="init" value="{!this}" action="{!c.doInit}"/>

    <!-- Body -->
    <aura:if isTrue="{!v.isLoading}">
        <lightning:spinner alternativeText="Loading"/>
    </aura:if>

    <lightning:card title="Accounts" iconName="standard:account">
        <aura:iteration items="{!v.accounts}" var="account">
            <p>{!account.Name}</p>
        </aura:iteration>
    </lightning:card>
</aura:component>
```

```javascript
// accountListAuraController.js
({
    doInit: function(component, event, helper) {
        helper.loadAccounts(component);
    },

    handleRefresh: function(component, event, helper) {
        helper.loadAccounts(component);
    }
})
```

```javascript
// accountListAuraHelper.js
({
    loadAccounts: function(component) {
        component.set('v.isLoading', true);

        var action = component.get('c.getAccounts');
        action.setParams({
            recordId: component.get('v.recordId')
        });

        action.setCallback(this, function(response) {
            var state = response.getState();
            if (state === 'SUCCESS') {
                component.set('v.accounts', response.getReturnValue());
            } else {
                this.handleError(component, response);
            }
            component.set('v.isLoading', false);
        });

        $A.enqueueAction(action);
    },

    handleError: function(component, response) {
        var errors = response.getError();
        var message = 'Unknown error';
        if (errors && errors[0] && errors[0].message) {
            message = errors[0].message;
        }
        this.showToast('Error', message, 'error');
    },

    showToast: function(title, message, type) {
        var toastEvent = $A.get('e.force:showToast');
        toastEvent.setParams({
            title: title,
            message: message,
            type: type
        });
        toastEvent.fire();
    }
})
```

## 10. Testing

### 10.1 Jest Test Structure

```javascript
// accountList.test.js
import { createElement } from 'lwc';
import AccountList from 'c/accountList';
import getAccounts from '@salesforce/apex/AccountController.getAccounts';

// Mock Apex
jest.mock(
    '@salesforce/apex/AccountController.getAccounts',
    () => ({ default: jest.fn() }),
    { virtual: true }
);

// Mock data
const MOCK_ACCOUNTS = [
    { Id: '001xx000003DGb1', Name: 'Acme Corp', Industry: 'Technology' },
    { Id: '001xx000003DGb2', Name: 'Global Inc', Industry: 'Finance' }
];

describe('c-account-list', () => {
    afterEach(() => {
        while (document.body.firstChild) {
            document.body.removeChild(document.body.firstChild);
        }
        jest.clearAllMocks();
    });

    it('displays accounts when data is returned', async () => {
        // Arrange
        getAccounts.mockResolvedValue(MOCK_ACCOUNTS);

        // Act
        const element = createElement('c-account-list', { is: AccountList });
        document.body.appendChild(element);

        // Wait for async operations
        await Promise.resolve();

        // Assert
        const accountElements = element.shadowRoot.querySelectorAll('.account-item');
        expect(accountElements.length).toBe(2);
    });

    it('displays error when fetch fails', async () => {
        // Arrange
        getAccounts.mockRejectedValue(new Error('Failed to fetch'));

        // Act
        const element = createElement('c-account-list', { is: AccountList });
        document.body.appendChild(element);

        await Promise.resolve();

        // Assert
        const errorElement = element.shadowRoot.querySelector('.error-message');
        expect(errorElement).not.toBeNull();
    });

    it('shows loading spinner while fetching', () => {
        // Arrange
        getAccounts.mockResolvedValue(MOCK_ACCOUNTS);

        // Act
        const element = createElement('c-account-list', { is: AccountList });
        document.body.appendChild(element);

        // Assert - before promise resolves
        const spinner = element.shadowRoot.querySelector('lightning-spinner');
        expect(spinner).not.toBeNull();
    });
});
```

### 10.2 Running Tests

```bash
# Run all LWC tests
npm run test:unit

# Run tests with coverage
npm run test:unit:coverage

# Run specific test file
npm run test:unit -- accountList.test.js

# Watch mode
npm run test:unit:watch
```

## 11. Accessibility

### 11.1 Accessibility Requirements

```html
<!-- Labels for inputs -->
<lightning-input
    label="Account Name"
    name="accountName"
    required>
</lightning-input>

<!-- Alternative text for icons/images -->
<lightning-icon
    icon-name="utility:check"
    alternative-text="Success"
    title="Record saved successfully">
</lightning-icon>

<!-- ARIA attributes when needed -->
<div role="alert" aria-live="polite">
    {errorMessage}
</div>

<!-- Keyboard navigation -->
<button onclick={handleClick} onkeypress={handleKeyPress}>
    Activate
</button>
```

### 11.2 Accessibility Checklist

- [ ] All interactive elements are keyboard accessible
- [ ] All images/icons have alternative text
- [ ] Form inputs have associated labels
- [ ] Color is not the only means of conveying information
- [ ] Focus states are visible
- [ ] Content structure uses proper headings
- [ ] Dynamic content updates are announced to screen readers
- [ ] Touch targets are at least 44x44 pixels

## 12. Performance

### 12.1 Performance Best Practices

```javascript
// Use wire for reactive data (cached)
@wire(getRecord, { recordId: '$recordId', fields: FIELDS })
record;

// Avoid unnecessary re-renders
// Use @track only when needed (complex objects)

// Debounce user input
handleSearch(event) {
    window.clearTimeout(this.searchTimeout);
    this.searchTimeout = window.setTimeout(() => {
        this.searchTerm = event.target.value;
    }, 300);
}

// Lazy load components
// Use lightning-dynamic-icon for conditional icons
// Use lightning-formatted-* components for data display
```

### 12.2 Bundle Size Optimization

```javascript
// Import only what you need
import { LightningElement, api } from 'lwc'; // Not: import * from 'lwc'

// Use platformResourceLoader for large libraries
import { loadScript, loadStyle } from 'lightning/platformResourceLoader';

// Avoid large inline data
// Move constants to separate files if large
```

## 13. Compliance Checklist

Before code review:

- [ ] Follows naming conventions
- [ ] Component has clear description in meta XML
- [ ] All public APIs documented
- [ ] Error handling implemented
- [ ] Loading states shown to users
- [ ] Accessibility requirements met
- [ ] Jest tests written with good coverage
- [ ] No console.log statements in production code
- [ ] No hardcoded text (use Custom Labels)
- [ ] SLDS classes used where applicable
- [ ] Mobile-responsive design
- [ ] Security review for Apex calls

---

**Document Owner:** UI/UX Team
**Next Review Date:** 2026-08-06
**Related Documents:**
- [Naming Conventions Standard](./naming-conventions.md)
- [Apex Coding Standards](./apex-coding-standards.md)
- [Testing Standards](./testing-standards.md)
