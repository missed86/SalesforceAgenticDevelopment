# Communication Patterns

## Parent to Child: @api Properties

The simplest communication. Parent sets attributes, child receives via `@api`.

```html
<!-- parent template -->
<c-account-card
    account-name={selectedAccount.Name}
    is-active={selectedAccount.IsActive}>
</c-account-card>
```

```javascript
// child JS
@api accountName;
@api isActive;
```

Attribute names in HTML use kebab-case; they map to camelCase `@api` properties.

---

## Child to Parent: CustomEvent

Child dispatches events. Parent handles them declaratively in the template.

### Basic Event (No Data)

```javascript
// child JS
handleClick() {
    this.dispatchEvent(new CustomEvent('close'));
}
```

```html
<!-- parent template -->
<c-modal onclose={handleModalClose}></c-modal>
```

### Event with Payload

```javascript
// child JS
handleSelect(event) {
    const selectedId = event.currentTarget.dataset.id;
    this.dispatchEvent(new CustomEvent('select', {
        detail: { recordId: selectedId }
    }));
}
```

```html
<!-- parent template -->
<c-account-list onselect={handleAccountSelect}></c-account-list>
```

```javascript
// parent JS
handleAccountSelect(event) {
    this.selectedRecordId = event.detail.recordId;
}
```

### Event Naming Rules

| Rule | Example | Anti-Pattern |
|------|---------|-------------|
| Lowercase, no hyphens in JS | `new CustomEvent('itemselected')` | `new CustomEvent('item-selected')` |
| Descriptive action name | `'save'`, `'select'`, `'delete'` | `'click'`, `'action'`, `'event'` |
| No `on` prefix in event name | `'close'` | `'onclose'` |
| Handler: `on` + eventname | `onclose={handleClose}` | `close={handleClose}` |

### Event Propagation

By default, events do not bubble beyond the parent. To propagate:

```javascript
this.dispatchEvent(new CustomEvent('notify', {
    detail: { message: 'Updated' },
    bubbles: true,    // propagates up the DOM
    composed: true    // crosses shadow DOM boundaries
}));
```

Use `bubbles: true, composed: true` sparingly. Prefer explicit parent-to-parent
relay for clarity and maintainability.

---

## Sibling Communication via Parent Relay

When two siblings need to communicate, the parent acts as relay:

```html
<!-- parent template -->
<c-filter-panel onfilterchange={handleFilterChange}></c-filter-panel>
<c-data-table filters={activeFilters}></c-data-table>
```

```javascript
// parent JS
activeFilters = {};

handleFilterChange(event) {
    this.activeFilters = event.detail;
}
```

The filter panel fires `filterchange`, the parent stores the filters,
and the data table reactively receives them via `@api filters`.

---

## Lightning Message Service (LMS)

For unrelated components (different regions of a page, different pages, or
even Aura/Visualforce interop), use LMS.

### Step 1: Create a Message Channel

File: `force-app/main/default/messageChannels/RecordSelected.messageChannel-meta.xml`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<LightningMessageChannel xmlns="http://soap.sforce.com/2006/04/metadata">
    <masterLabel>Record Selected</masterLabel>
    <description>Published when a user selects a record in any list component.</description>
    <isExposed>true</isExposed>
    <lightningMessageFields>
        <fieldName>recordId</fieldName>
        <description>The ID of the selected record.</description>
    </lightningMessageFields>
    <lightningMessageFields>
        <fieldName>objectApiName</fieldName>
        <description>The API name of the selected object.</description>
    </lightningMessageFields>
</LightningMessageChannel>
```

### Step 2: Publish

```javascript
import { LightningElement, wire } from 'lwc';
import { publish, MessageContext } from 'lightning/messageService';
import RECORD_SELECTED from '@salesforce/messageChannel/RecordSelected__c';

export default class AccountList extends LightningElement {
    @wire(MessageContext)
    messageContext;

    handleSelect(event) {
        publish(this.messageContext, RECORD_SELECTED, {
            recordId: event.detail.recordId,
            objectApiName: 'Account'
        });
    }
}
```

### Step 3: Subscribe

```javascript
import { LightningElement, wire } from 'lwc';
import {
    subscribe, unsubscribe, MessageContext
} from 'lightning/messageService';
import RECORD_SELECTED from '@salesforce/messageChannel/RecordSelected__c';

export default class AccountDetail extends LightningElement {
    @wire(MessageContext)
    messageContext;

    subscription;
    selectedRecordId;

    connectedCallback() {
        this.subscription = subscribe(
            this.messageContext,
            RECORD_SELECTED,
            (message) => this.handleMessage(message)
        );
    }

    disconnectedCallback() {
        unsubscribe(this.subscription);
        this.subscription = null;
    }

    handleMessage(message) {
        this.selectedRecordId = message.recordId;
    }
}
```

### LMS Best Practices

- Always unsubscribe in `disconnectedCallback` to prevent memory leaks
- Keep message payloads small (IDs and short strings, not full records)
- Name message channels after the action, not the component
- Set `isExposed="true"` only if other namespaces need access
- Use LMS instead of the deprecated pub/sub module

---

## Navigation

### NavigationMixin Setup

```javascript
import { LightningElement } from 'lwc';
import { NavigationMixin } from 'lightning/navigation';

export default class MyComponent extends NavigationMixin(LightningElement) {
    // NavigationMixin.Navigate and NavigationMixin.GenerateUrl available
}
```

### Common Navigation Patterns

```javascript
// Navigate to record
navigateToRecord(recordId) {
    this[NavigationMixin.Navigate]({
        type: 'standard__recordPage',
        attributes: {
            recordId,
            objectApiName: 'Account',
            actionName: 'view'
        }
    });
}

// Navigate to list view
navigateToList() {
    this[NavigationMixin.Navigate]({
        type: 'standard__objectPage',
        attributes: {
            objectApiName: 'Account',
            actionName: 'list'
        },
        state: {
            filterName: 'Recent'
        }
    });
}

// Navigate to custom tab / app page
navigateToTab() {
    this[NavigationMixin.Navigate]({
        type: 'standard__navItemPage',
        attributes: {
            apiName: 'CustomTab'
        }
    });
}

// Open external URL
navigateToExternal(url) {
    this[NavigationMixin.Navigate]({
        type: 'standard__webPage',
        attributes: { url }
    });
}

// Generate URL for href
connectedCallback() {
    this[NavigationMixin.GenerateUrl]({
        type: 'standard__recordPage',
        attributes: {
            recordId: this.recordId,
            actionName: 'view'
        }
    }).then((url) => {
        this.recordUrl = url;
    });
}
```

### URL State Parameters

Prefix custom state properties with your namespace (`c__` for unmanaged):

```javascript
this[NavigationMixin.Navigate]({
    type: 'standard__component',
    attributes: {
        componentName: 'c__orderWizard'
    },
    state: {
        c__step: '2',
        c__orderId: this.orderId
    }
});
```

Read state with `@wire(CurrentPageReference)`:

```javascript
import { CurrentPageReference } from 'lightning/navigation';

@wire(CurrentPageReference)
pageRef;

get currentStep() {
    return this.pageRef?.state?.c__step || '1';
}
```

### Navigation Rules

- Never hardcode URLs; always use `PageReference` objects
- Use `GenerateUrl` for `href` attributes (enables middle-click, right-click)
- All `state` values must be strings
- Never include sensitive data in URL parameters
- Use `replace: true` for redirects to avoid polluting browser history

---

## Flow Communication

When a LWC is embedded in a Screen Flow:

```javascript
import { LightningElement, api } from 'lwc';
import { FlowNavigationNextEvent, FlowAttributeChangeEvent } from 'lightning/flowSupport';

export default class FlowInput extends LightningElement {
    @api selectedValue;

    handleChange(event) {
        const newValue = event.detail.value;
        this.dispatchEvent(
            new FlowAttributeChangeEvent('selectedValue', newValue)
        );
    }

    handleNext() {
        const validation = this.validate();
        if (validation.isValid) {
            this.dispatchEvent(new FlowNavigationNextEvent());
        }
    }

    @api
    validate() {
        if (!this.selectedValue) {
            return {
                isValid: false,
                errorMessage: 'Please select a value before proceeding.'
            };
        }
        return { isValid: true };
    }
}
```
