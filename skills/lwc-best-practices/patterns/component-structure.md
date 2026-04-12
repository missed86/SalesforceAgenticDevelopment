# Component Structure

## File Layout

Every LWC component lives in its own folder under `force-app/main/default/lwc/`:

```
lwc/
  accountCard/
    accountCard.html
    accountCard.js
    accountCard.js-meta.xml
    accountCard.css                  (optional)
    __tests__/
      accountCard.test.js            (optional but recommended)
```

Components cannot be nested inside other component folders.

## Naming Rules

| Element | Convention | Platform Rule |
|---------|-----------|---------------|
| Folder name | camelCase | Must start lowercase, alphanumeric + underscore only |
| JS file | Same as folder | `accountCard.js` |
| HTML file | Same as folder | `accountCard.html` |
| CSS file | Same as folder | `accountCard.css` |
| Meta file | Same as folder + `.js-meta.xml` | `accountCard.js-meta.xml` |
| JS class | PascalCase | `export default class AccountCard extends LightningElement` |

camelCase folder names automatically map to kebab-case in markup:

```
Folder: accountCard  ->  Markup: <c-account-card>
Folder: orderLineItem  ->  Markup: <c-order-line-item>
```

Underscores in folder names do NOT map to hyphens. Avoid them.

## Component Classification

Organize components by responsibility:

| Type | Purpose | Example |
|------|---------|---------|
| Container | Fetches data, coordinates children | `accountDashboard` |
| Presentational | Displays data, fires events, no Apex | `accountSummaryCard` |
| Service (headless) | Exports shared JS logic, no HTML | `utils`, `dateFormatter` |
| Form | Captures user input, validates | `accountEditForm` |

### Container vs Presentational

```
accountDashboard (container)
  |-- accountSummaryCard (presentational)
  |-- accountActivityList (presentational)
  |-- accountEditForm (form)
```

The container calls Apex, holds state, and passes data down. Presentational
components receive data via `@api` and fire events up. This separation makes
presentational components reusable and testable.

## Meta Configuration (.js-meta.xml)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<LightningComponentBundle xmlns="http://soap.sforce.com/2006/04/metadata">
    <apiVersion>62.0</apiVersion>
    <isExposed>true</isExposed>
    <masterLabel>Account Card</masterLabel>
    <description>Displays account summary with key metrics.</description>
    <targets>
        <target>lightning__RecordPage</target>
        <target>lightning__AppPage</target>
        <target>lightning__HomePage</target>
    </targets>
    <targetConfigs>
        <targetConfig targets="lightning__RecordPage">
            <objects>
                <object>Account</object>
            </objects>
            <property name="showActions" type="Boolean" default="true"
                      label="Show Action Buttons"
                      description="Toggle visibility of edit/delete buttons." />
        </targetConfig>
    </targetConfigs>
</LightningComponentBundle>
```

### Meta Best Practices

- Always set `isExposed="true"` if the component is used in App Builder
- Always include `masterLabel` and `description` for admin discoverability
- Lock `apiVersion` to the version you tested against
- Restrict `targets` to only where the component should appear
- Use `targetConfigs` to expose admin-configurable properties with labels and descriptions
- Specify `objects` to limit record page placement to relevant objects

## JavaScript Class Structure

Recommended order of members in the JS class:

```javascript
import { LightningElement, api, wire } from 'lwc';
import { NavigationMixin } from 'lightning/navigation';
import getAccounts from '@salesforce/apex/AccountController.getAccountsByIndustry';
import ACCOUNT_OBJECT from '@salesforce/schema/Account';

const MAX_RECORDS = 50;

export default class AccountDashboard extends NavigationMixin(LightningElement) {

    // 1. Public reactive properties (@api)
    @api recordId;
    @api objectApiName;

    // 2. Private reactive fields
    accounts = [];
    isLoading = false;
    errorMessage;

    // 3. Wire declarations
    _wiredAccountsResult;

    @wire(getAccounts, { industry: '$selectedIndustry' })
    wiredAccounts(result) {
        this._wiredAccountsResult = result;
        const { data, error } = result;
        if (data) {
            this.accounts = data;
            this.errorMessage = undefined;
        } else if (error) {
            this.errorMessage = error.body?.message;
            this.accounts = [];
        }
    }

    // 4. Lifecycle hooks (in execution order)
    connectedCallback() {
        this.loadInitialData();
    }

    disconnectedCallback() {
        this.cleanupResources();
    }

    // 5. Getters
    get hasAccounts() {
        return this.accounts.length > 0;
    }

    get formattedCount() {
        return `${this.accounts.length} accounts`;
    }

    // 6. Public methods (@api)
    @api
    refresh() {
        return refreshApex(this._wiredAccountsResult);
    }

    // 7. Event handlers
    handleSelect(event) {
        const selectedId = event.detail.recordId;
        this.navigateToRecord(selectedId);
    }

    // 8. Private methods
    loadInitialData() { }

    navigateToRecord(recordId) {
        this[NavigationMixin.Navigate]({
            type: 'standard__recordPage',
            attributes: {
                recordId,
                objectApiName: ACCOUNT_OBJECT.objectApiName,
                actionName: 'view'
            }
        });
    }

    cleanupResources() { }
}
```

## HTML Template Structure

```html
<template>
    <!-- Loading state -->
    <template lwc:if={isLoading}>
        <lightning-spinner alternative-text="Loading" size="medium">
        </lightning-spinner>
    </template>

    <!-- Error state -->
    <template lwc:elseif={errorMessage}>
        <c-error-panel message={errorMessage} onretry={handleRetry}>
        </c-error-panel>
    </template>

    <!-- Content -->
    <template lwc:else>
        <lightning-card title="Accounts" icon-name="standard:account">
            <template lwc:if={hasAccounts}>
                <template for:each={accounts} for:item="account">
                    <c-account-summary-card
                        key={account.Id}
                        account={account}
                        onselect={handleSelect}>
                    </c-account-summary-card>
                </template>
            </template>

            <template lwc:else>
                <div class="slds-align_absolute-center slds-m-around_large">
                    <p>No accounts found.</p>
                </div>
            </template>
        </lightning-card>
    </template>
</template>
```

### Template Best Practices

- Use `lwc:if` / `lwc:elseif` / `lwc:else` (never `if:true`)
- Always provide `key` in `for:each` loops (use unique ID, not array index)
- Handle 3 states: loading, error, content
- Keep templates flat; extract deeply nested sections into child components
- Never use complex expressions in templates; use getters
- Use `lightning-spinner` with `alternative-text` for accessible loading

## Service Components (Shared Logic)

For reusable logic without UI, create a headless component (JS only, no HTML):

```javascript
// lwc/utils/utils.js
const formatCurrency = (value, currency = 'USD') => {
    return new Intl.NumberFormat('en-US', {
        style: 'currency',
        currency
    }).format(value);
};

const debounce = (fn, delay = 300) => {
    let timeoutId;
    return (...args) => {
        clearTimeout(timeoutId);
        timeoutId = setTimeout(() => fn(...args), delay);
    };
};

export { formatCurrency, debounce };
```

The `reduceErrors` utility is defined in [error-handling.md](error-handling.md)
and should also live in this module.

Import in any component:

```javascript
import { formatCurrency, debounce, reduceErrors } from 'c/utils';
```
