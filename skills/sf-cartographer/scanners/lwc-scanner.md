# LWC & Aura Scanner

Detailed instructions for scanning Lightning Web Components and Aura components.

## Discovery — LWC

1. Glob for `**/lwc/*/` across all package directories. Each subfolder is one component.
2. For each component folder, identify available files:
   - `{name}.js` — JavaScript class (required)
   - `{name}.html` — Template (optional for service components)
   - `{name}.css` — Styles (optional)
   - `{name}.js-meta.xml` — Metadata config (required)
   - `__tests__/` — Jest tests (optional)

### Metadata Extraction (.js-meta.xml)

Read the full `.js-meta.xml` and extract:

| Field | XML Path | Example |
|-------|----------|---------|
| API version | `<apiVersion>` | 62.0 |
| Exposed | `<isExposed>` | true / false |
| Master label | `<masterLabel>` | Account Card |
| Description | `<description>` | Displays account summary |
| Targets | `<targets><target>` | lightning__RecordPage, lightning__AppPage |
| Target objects | `<targetConfig><objects><object>` | Account |
| Design attributes | `<targetConfig><property>` | name, type, default, label |

### JavaScript Analysis ({name}.js)

Read the **full JS file** and extract:

#### Public Properties (@api)

Look for lines matching `@api propertyName` or `@api get propertyName`:

```
@api recordId;              → { name: "recordId", type: "property" }
@api get isVisible() {...}  → { name: "isVisible", type: "getter" }
@api refresh() {...}        → { name: "refresh", type: "method" }
```

#### Wire Declarations

Look for `@wire(AdapterName, { params })`:

```
@wire(getRecord, { recordId: '$recordId', fields }) → wire adapter: getRecord (LDS)
@wire(getAccounts, { industry: '$filter' })          → wire adapter: getAccounts (Apex)
```

Classify wire adapters:
- LDS adapters: `getRecord`, `getFieldValue`, `getObjectInfo`, `getPicklistValues`, `getRelatedListRecords`, etc.
- Apex adapters: imported from `@salesforce/apex/{ClassName}.{methodName}`

#### Apex Method Imports

Look for imports from `@salesforce/apex/`:

```
import getAccounts from '@salesforce/apex/AccountController.getAccountsByIndustry';
→ { class: "AccountController", method: "getAccountsByIndustry", usage: "wire" or "imperative" }
```

To determine if it's wire vs imperative: if the import is used in a `@wire` decorator, it's wire. Otherwise it's imperative (called in a method body).

#### Custom Events Dispatched

Search for `new CustomEvent('eventname'`:

```
this.dispatchEvent(new CustomEvent('itemselected', { detail: ... }));
→ { event: "itemselected", hasDetail: true }
```

#### Child Components Used

Search the HTML template for `<c-*` tags:

```
<c-account-card ...>    → uses component: accountCard
<c-error-panel ...>     → uses component: errorPanel
```

Map kebab-case tag names back to camelCase folder names.

#### LMS Usage

Search for imports from `lightning/messageService`:
- `publish` → publishes to LMS
- `subscribe` → subscribes to LMS

Search for `@wire(MessageContext)` and channel imports from `@salesforce/messageChannel/`.

#### Navigation

Search for `NavigationMixin` import and `this[NavigationMixin.Navigate]` calls.

#### Schema Imports

Search for imports from `@salesforce/schema/` — these reference specific SObject fields.

### Component Classification

| Type | Detection Rule |
|------|---------------|
| **Container** | Has Apex imports or wire adapters that fetch data, renders child `<c-*>` components |
| **Presentational** | Has `@api` props, dispatches events, no direct Apex calls |
| **Service (headless)** | Has `.js` file but NO `.html` file; exports functions |
| **Form** | Contains `<lightning-record-form>`, `<lightning-record-edit-form>`, or multiple `<lightning-input>` |

If ambiguous, default to Container if it fetches data, Presentational otherwise.

---

## Discovery — Aura (Legacy)

1. Glob for `**/aura/*/` across all package directories.
2. For each Aura folder, note:
   - `{name}.cmp` — Component markup
   - `{name}Controller.js` — Client-side controller
   - `{name}Helper.js` — Helper (if exists)
   - `{name}.design` — Design resource (if exists)
   - `{name}.cmp-meta.xml` — Metadata

3. For each Aura component, extract:
   - Name
   - Whether it has a design file (exposed in App Builder)
   - Any `<aura:attribute>` declarations (similar to `@api`)
   - Mark all Aura components as **Legacy** in documentation

---

## Output Format for lwc-catalog.md

### Summary

```markdown
## Summary

| Type | Count |
|------|-------|
| LWC — Container | 5 |
| LWC — Presentational | 12 |
| LWC — Service | 2 |
| LWC — Form | 3 |
| Aura (legacy) | 1 |
| **Total** | **23** |
```

### Component Catalog

For each LWC component, generate a card:

```markdown
### `accountDashboard`

| Property | Value |
|----------|-------|
| Type | Container |
| Exposed | Yes |
| Targets | Record Page (Account), App Page |
| Description | Main dashboard for account overview |
| Has Tests | Yes |
| Package | force-app |

**Public API:**

| Member | Kind | Description |
|--------|------|-------------|
| `recordId` | @api property | Record context |
| `refresh()` | @api method | Refreshes data |

**Apex Dependencies:**

| Class | Method | Usage |
|-------|--------|-------|
| `AccountController` | `getAccountSummary` | wire |
| `AccountController` | `updateAccount` | imperative |

**Child Components:** `accountSummaryCard`, `accountActivityList`, `errorPanel`

**Events Dispatched:** `accountupdated` (detail: { recordId })
```

### LMS Channels

```markdown
## Lightning Message Channels

| Channel | Published By | Subscribed By |
|---------|-------------|---------------|
| `RecordSelected__c` | `accountList` | `accountDetail`, `accountMap` |
```

### Aura Components (Legacy)

```markdown
## Aura Components (Legacy)

| Component | Has Design | Attributes | Notes |
|-----------|-----------|------------|-------|
| `legacyDashboard` | Yes | recordId, mode | Consider migrating to LWC |
```

---

## Composition Tree

If the project has enough components, generate a visual composition tree:

```
accountDashboard (container)
├── accountSummaryCard (presentational)
├── accountActivityList (presentational)
│   └── activityItem (presentational)
├── accountEditForm (form)
└── errorPanel (presentational)
```

Build this by tracing `<c-*>` tags recursively from container components.
