# Performance Optimization

## Rendering Performance

### Conditional Rendering vs CSS Visibility

```html
<!-- Removes/recreates DOM (use for heavy sections rarely shown) -->
<template lwc:if={showPanel}>
    <c-heavy-panel></c-heavy-panel>
</template>

<!-- Hides via CSS (use for frequent toggle, preserves state) -->
<div class={panelClass}>
    <c-light-panel></c-light-panel>
</div>
```

```javascript
get panelClass() {
    return this.showPanel ? 'slds-show' : 'slds-hide';
}
```

**Decision:** Use `lwc:if` when the hidden content is expensive to render
and shown infrequently. Use CSS `slds-show/slds-hide` when toggling is
frequent and the component must retain internal state.

### Avoid Expensive Getters

Getters execute on every render cycle. Never put heavy computation inside:

```javascript
// BAD: runs on every render
get sortedItems() {
    return [...this.items].sort((a, b) => a.name.localeCompare(b.name));
}

// GOOD: sort once when data changes
_items = [];
_sortedItems = [];

set items(value) {
    this._items = value;
    this._sortedItems = [...value].sort((a, b) =>
        a.name.localeCompare(b.name)
    );
}

@api
get items() {
    return this._items;
}

get sortedItems() {
    return this._sortedItems;
}
```

### Key Attribute in for:each

Always use a stable unique identifier, never the array index:

```html
<!-- GOOD: stable unique ID -->
<template for:each={accounts} for:item="account">
    <c-account-row key={account.Id} account={account}></c-account-row>
</template>

<!-- BAD: index causes full re-render on reorder -->
<template for:each={accounts} for:item="account" for:index="idx">
    <c-account-row key={idx} account={account}></c-account-row>
</template>
```

---

## Lazy Loading

### Deferred Component Loading

Load heavy sections only when the user needs them:

```html
<lightning-tabset>
    <lightning-tab label="Details" onactive={handleDetailsActive}>
        <template lwc:if={detailsLoaded}>
            <c-account-details record-id={recordId}></c-account-details>
        </template>
    </lightning-tab>
    <lightning-tab label="Analytics" onactive={handleAnalyticsActive}>
        <template lwc:if={analyticsLoaded}>
            <c-account-analytics record-id={recordId}></c-account-analytics>
        </template>
    </lightning-tab>
</lightning-tabset>
```

```javascript
detailsLoaded = true;
analyticsLoaded = false;

handleAnalyticsActive() {
    this.analyticsLoaded = true;
}
```

### Lazy Data Loading

Do not fetch all data on `connectedCallback`. Fetch only what the user sees:

```javascript
connectedCallback() {
    this.loadPage(1);
}

async loadPage(page) {
    this.isLoading = true;
    try {
        const result = await getAccountsPage({
            pageNumber: page,
            pageSize: this.pageSize
        });
        this.accounts = result.records;
        this.totalRecords = result.totalCount;
        this.pageNumber = page;
    } catch (error) {
        this.error = reduceErrors(error);
    } finally {
        this.isLoading = false;
    }
}
```

---

## Caching Strategies

### Wire Service Caching

Methods annotated with `@AuraEnabled(cacheable=true)` are cached client-side.
The cache is shared across components on the same page.

```java
// Apex
@AuraEnabled(cacheable=true)
public static List<AccountDto> getAccountsByIndustry(String industry) {
    return AccountSelector.selectByIndustry(industry, 50);
}
```

Cache invalidation:

```javascript
// After a mutation, refresh wired data:
await refreshApex(this.wiredResult);

// Or notify LDS cache for specific records:
import { notifyRecordUpdateAvailable } from 'lightning/uiRecordApi';
await notifyRecordUpdateAvailable([{ recordId: this.recordId }]);
```

### Local Computation Caching

Cache expensive computations to avoid rerunning on every render:

```javascript
_cachedMap;
_lastAccountsRef;

get accountMap() {
    if (this._lastAccountsRef !== this.accounts) {
        this._cachedMap = new Map(
            this.accounts.map(acc => [acc.Id, acc])
        );
        this._lastAccountsRef = this.accounts;
    }
    return this._cachedMap;
}
```

The cache invalidates when `this.accounts` is reassigned (new reference).
Since immutable patterns produce a new array on every change, reference
comparison is a reliable and cheap invalidation strategy.

---

## Debounce User Input

For search inputs or filters that trigger Apex calls:

```javascript
import { debounce } from 'c/utils';

searchTerm = '';
_debouncedSearch;

connectedCallback() {
    this._debouncedSearch = debounce((term) => {
        this.searchTerm = term;
    }, 300);
}

handleSearchInput(event) {
    this._debouncedSearch(event.target.value);
}
```

This prevents firing an Apex call on every keystroke.

---

## Reducing Wire Calls

### Prevent Unnecessary Fires

Wire adapters fire when any reactive parameter changes. Avoid triggering
spurious calls by guarding against undefined:

```javascript
// Wire won't fire until selectedIndustry is defined
selectedIndustry;

@wire(getAccounts, { industry: '$selectedIndustry' })
accounts;
```

### Batch Related Data

Instead of multiple Apex calls from one component, create a single Apex
method that returns a composite DTO:

```java
// Apex
@AuraEnabled(cacheable=true)
public static AccountDashboardDto getDashboardData(Id accountId) {
    return new AccountDashboardDto(
        AccountSelector.selectById(accountId),
        ContactSelector.selectByAccountId(accountId),
        OpportunitySelector.selectOpenByAccountId(accountId)
    );
}
```

```javascript
// LWC
@wire(getDashboardData, { accountId: '$recordId' })
dashboard;

get contacts() { return this.dashboard.data?.contacts; }
get opportunities() { return this.dashboard.data?.opportunities; }
```

One wire call instead of three.

---

## Large List Performance

### Virtual Scrolling with lightning-datatable

For lists over 50 items, use `lightning-datatable` with `enable-infinite-loading`:

```html
<lightning-datatable
    key-field="Id"
    data={accounts}
    columns={columns}
    enable-infinite-loading
    onloadmore={handleLoadMore}>
</lightning-datatable>
```

```javascript
async handleLoadMore(event) {
    event.target.isLoading = true;
    try {
        const moreData = await getAccountsPage({
            pageNumber: ++this.pageNumber,
            pageSize: this.pageSize
        });
        this.accounts = [...this.accounts, ...moreData.records];
        if (this.accounts.length >= this.totalRecords) {
            event.target.enableInfiniteLoading = false;
        }
    } catch (error) {
        this.error = reduceErrors(error);
    } finally {
        event.target.isLoading = false;
    }
}
```

### Limit Displayed Children

If rendering a list of custom components, cap the visible count:

```javascript
get visibleAccounts() {
    return this.accounts.slice(0, this.visibleCount);
}

get hasMore() {
    return this.visibleCount < this.accounts.length;
}

handleShowMore() {
    this.visibleCount += 20;
}
```

---

## Performance Checklist

- [ ] No complex expressions in templates (use getters)
- [ ] `key` uses unique IDs, not array indices
- [ ] Debounce on search/filter inputs
- [ ] Lazy load tabs and sections not immediately visible
- [ ] Use `cacheable=true` on all read-only Apex methods
- [ ] Batch related queries into composite DTOs where possible
- [ ] `@track` only on objects that need deep mutation tracking
- [ ] No state mutations inside `renderedCallback`
- [ ] Use `lightning-datatable` infinite loading for large lists
- [ ] Remove `console.log` before production deployment
