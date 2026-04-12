# Data and State Management

## Decorators

### @api (Public Properties)

Exposes a property to parent components. Treat as read-only inside the child.

```javascript
// child: accountSummaryCard.js — simple read-only case
@api accountName;
@api recordId;
```

If the child needs to modify the value locally, use a getter/setter
to copy into a private field:

```javascript
// child: editableField.js — local mutation case
_localName;

@api
get accountName() {
    return this._localName;
}
set accountName(value) {
    this._localName = value;
}
```

Parent passes data via attributes:

```html
<c-account-summary-card
    account-name={account.Name}
    record-id={account.Id}>
</c-account-summary-card>
```

### @api Methods

Expose a public method the parent can call imperatively:

```javascript
// child
@api
reset() {
    this.selectedItems = [];
    this.searchTerm = '';
}

// parent JS
this.template.querySelector('c-filter-panel').reset();
```

### @track (Deep Reactivity)

Since LWC 1.17, primitive fields are reactive without `@track`. Use `@track`
only for deep observation of nested object or array mutations:

```javascript
// Primitives: auto-reactive, no @track needed
isLoading = false;
selectedId = null;
count = 0;

// Objects: use spread to trigger reactivity (preferred)
user = { name: 'John', role: 'Admin' };
updateName(newName) {
    this.user = { ...this.user, name: newName };
}

// Arrays: use spread to trigger reactivity (preferred)
items = [];
addItem(item) {
    this.items = [...this.items, item];
}

// @track only when you MUST mutate in place (rare)
@track complexConfig = { nested: { deep: { value: 1 } } };
updateDeep() {
    this.complexConfig.nested.deep.value = 2; // tracked
}
```

**Rule of thumb:** Prefer immutable patterns (spread) over `@track`.

---

## Wire Service

### Wire to Property (Simplest)

Returns `{ data, error }` object. Best for straightforward display.

```javascript
import getAccounts from '@salesforce/apex/AccountController.getAccountsByIndustry';

@wire(getAccounts, { industry: 'Technology' })
accounts;

// In template:
// lwc:if={accounts.data}  ->  for:each={accounts.data}
// lwc:if={accounts.error} ->  show error
```

### Wire to Function (Full Control)

Allows transformation, logging, and error handling before assignment.

```javascript
import { formatCurrency } from 'c/utils';

wiredAccountsResult;

@wire(getAccounts, { industry: '$selectedIndustry' })
wiredAccounts(result) {
    this.wiredAccountsResult = result;
    const { data, error } = result;
    if (data) {
        this.accounts = data.map(acc => ({
            ...acc,
            formattedRevenue: formatCurrency(acc.revenue)
        }));
        this.error = undefined;
    } else if (error) {
        this.error = error;
        this.accounts = [];
    }
}
```

Store the full `result` reference so `refreshApex()` can re-provision:

```javascript
async handleRefresh() {
    this.isLoading = true;
    try {
        await refreshApex(this.wiredAccountsResult);
    } finally {
        this.isLoading = false;
    }
}
```

### Reactive Wire Parameters

Prefix parameter values with `$` to make them reactive. The wire re-fires
whenever the referenced property changes:

```javascript
selectedIndustry = 'Technology';

@wire(getAccounts, { industry: '$selectedIndustry' })
accounts;

handleIndustryChange(event) {
    this.selectedIndustry = event.detail.value;
    // wire automatically re-fires
}
```

Wire only fires if the parameter is not `undefined`. To prevent the initial
call until the user selects a value, initialize as `undefined`:

```javascript
selectedIndustry; // undefined = wire won't fire until set
```

---

## Imperative Apex Calls

Use when DML is needed or when you need full timing control.

```javascript
import saveAccount from '@salesforce/apex/AccountController.saveAccount';

async handleSave() {
    this.isLoading = true;
    try {
        const result = await saveAccount({ accountDto: this.formData });
        this.showToast('Success', result.message, 'success');
        this.dispatchEvent(new CustomEvent('save', { detail: result }));
    } catch (error) {
        this.showToast('Error', reduceErrors(error).join(', '), 'error');
    } finally {
        this.isLoading = false;
    }
}
```

### Wire vs Imperative Decision

| Criteria | @wire | Imperative |
|----------|-------|-----------|
| Read-only display | Yes | Overkill |
| Reactive to params | Automatic | Manual |
| DML operations | No | Yes |
| Precise timing control | No | Yes |
| Client-side caching | Automatic | None (call `refreshApex` manually) |
| `cacheable=true` required | Yes | No (but recommended for reads) |
| Error handling | Wire result | try/catch |

---

## Lightning Data Service (LDS)

### When to Use LDS vs Apex

| Use LDS when | Use Apex when |
|-------------|---------------|
| Single record CRUD | Multi-record operations |
| Standard field display | Complex business logic |
| Need auto FLS/CRUD enforcement | Cross-object queries |
| No custom validation needed | External system callouts |
| Simple UI form | Aggregations or SOSL |

### Record Forms (Zero Apex)

```html
<!-- Auto-layout form (simplest) -->
<lightning-record-form
    record-id={recordId}
    object-api-name="Account"
    fields={fields}
    mode="view"
    onsuccess={handleSuccess}>
</lightning-record-form>

<!-- Edit form (more control) -->
<lightning-record-edit-form
    record-id={recordId}
    object-api-name="Account"
    onsuccess={handleSuccess}
    onerror={handleError}>

    <lightning-messages></lightning-messages>
    <lightning-input-field field-name="Name"></lightning-input-field>
    <lightning-input-field field-name="Industry"></lightning-input-field>

    <lightning-button
        type="submit"
        variant="brand"
        label="Save">
    </lightning-button>
</lightning-record-edit-form>
```

### getRecord Wire Adapter

For read-only display outside of forms:

```javascript
import { LightningElement, api, wire } from 'lwc';
import { getRecord, getFieldValue } from 'lightning/uiRecordApi';
import NAME_FIELD from '@salesforce/schema/Account.Name';
import INDUSTRY_FIELD from '@salesforce/schema/Account.Industry';

const FIELDS = [NAME_FIELD, INDUSTRY_FIELD];

export default class AccountDetail extends LightningElement {
    @api recordId;

    @wire(getRecord, { recordId: '$recordId', fields: FIELDS })
    account;

    get accountName() {
        return getFieldValue(this.account.data, NAME_FIELD);
    }

    get accountIndustry() {
        return getFieldValue(this.account.data, INDUSTRY_FIELD);
    }
}
```

**Always import field references** from `@salesforce/schema` instead of
using string literals. This ensures compile-time validation and proper
dependency tracking.

### createRecord and updateRecord

```javascript
import { createRecord, updateRecord } from 'lightning/uiRecordApi';
import ACCOUNT_OBJECT from '@salesforce/schema/Account';
import NAME_FIELD from '@salesforce/schema/Account.Name';

// Create
async handleCreate() {
    const fields = {};
    fields[NAME_FIELD.fieldApiName] = this.accountName;

    try {
        const record = await createRecord({
            apiName: ACCOUNT_OBJECT.objectApiName,
            fields
        });
        this.showToast('Success', 'Account created', 'success');
    } catch (error) {
        this.showToast('Error', reduceErrors(error).join(', '), 'error');
    }
}

// Update
async handleUpdate() {
    const fields = {};
    fields.Id = this.recordId;
    fields[NAME_FIELD.fieldApiName] = this.updatedName;

    try {
        await updateRecord({ fields });
        this.showToast('Success', 'Account updated', 'success');
    } catch (error) {
        this.showToast('Error', reduceErrors(error).join(', '), 'error');
    }
}
```

### Cache Management

```javascript
import { refreshApex } from '@salesforce/apex';
import { notifyRecordUpdateAvailable } from 'lightning/uiRecordApi';

// After imperative DML, refresh wired data:
await refreshApex(this.wiredResult);

// After LDS mutation, notify other components:
await notifyRecordUpdateAvailable([{ recordId: this.recordId }]);
```

---

## Record Types in LWC

### Getting Available Record Types

Use `getObjectInfo` to retrieve record type metadata without Apex:

```javascript
import { LightningElement, wire } from 'lwc';
import { getObjectInfo } from 'lightning/uiObjectInfoApi';
import CASE_OBJECT from '@salesforce/schema/Case';

export default class RecordTypeSelector extends LightningElement {
    recordTypeOptions = [];
    selectedRecordTypeId;

    @wire(getObjectInfo, { objectApiName: CASE_OBJECT })
    handleObjectInfo({ data, error }) {
        if (data) {
            const rtInfos = data.recordTypeInfos;
            this.recordTypeOptions = Object.values(rtInfos)
                .filter(rt => rt.available && !rt.master)
                .map(rt => ({ label: rt.name, value: rt.recordTypeId }));

            if (this.recordTypeOptions.length === 1) {
                this.selectedRecordTypeId = this.recordTypeOptions[0].value;
            }
        } else if (error) {
            this.recordTypeOptions = [];
        }
    }

    handleRecordTypeChange(event) {
        this.selectedRecordTypeId = event.detail.value;
    }
}
```

### Passing Record Type to Forms

```html
<lightning-combobox
    label="Record Type"
    options={recordTypeOptions}
    value={selectedRecordTypeId}
    onchange={handleRecordTypeChange}>
</lightning-combobox>

<template lwc:if={selectedRecordTypeId}>
    <lightning-record-form
        object-api-name="Case"
        record-type-id={selectedRecordTypeId}
        fields={fields}
        onsuccess={handleSuccess}>
    </lightning-record-form>
</template>
```

### Getting the Default Record Type

```javascript
get defaultRecordTypeId() {
    if (!this.objectInfo) return undefined;
    return this.objectInfo.defaultRecordTypeId;
}
```

### Record Type Decision Table

| Need | Approach |
|------|----------|
| List available RTs for a picklist | `getObjectInfo` → filter `recordTypeInfos` |
| Pre-select default RT | `objectInfo.defaultRecordTypeId` |
| Show different form layouts per RT | Pass `record-type-id` to `lightning-record-form` |
| Conditional UI based on RT of existing record | `getRecord` with `RecordTypeId` field, compare against `getObjectInfo` |
| Create record with specific RT from Apex | Pass RT Id from LWC → Controller → Service (resolve via `Schema` in Apex, see [configuration.md](../../apex-architecture/patterns/configuration.md)) |

---

## State Patterns

### Loading / Error / Content Pattern

Every data-fetching component should track 3 states. See the full HTML
template example in [component-structure.md](component-structure.md).

```javascript
isLoading = false;
error;
data;

get hasData() {
    return this.data && this.data.length > 0;
}

get hasError() {
    return !!this.error;
}
```

### Pagination Pattern

For large datasets, use server-side pagination:

```javascript
pageSize = 10;
pageNumber = 1;
totalRecords = 0;

get totalPages() {
    return Math.ceil(this.totalRecords / this.pageSize);
}

get isFirstPage() {
    return this.pageNumber <= 1;
}

get isLastPage() {
    return this.pageNumber >= this.totalPages;
}

handlePrevious() {
    if (!this.isFirstPage) {
        this.pageNumber--;
        this.loadPage();
    }
}

handleNext() {
    if (!this.isLastPage) {
        this.pageNumber++;
        this.loadPage();
    }
}
```
