# UI Configuration Scanner

Detailed instructions for scanning layouts, Lightning pages (flexipages), tabs, applications, quick actions, static resources, and message channels.

## Discovery — Page Layouts

1. Glob for `**/layouts/*.layout-meta.xml`.
2. Layout filenames follow the pattern `{Object}-{LayoutName}.layout-meta.xml`.
3. For each layout, extract:

| Field | Source | Notes |
|-------|--------|-------|
| Object | Parsed from filename (before first `-`) | Account, Case, CustomObj__c |
| Layout name | Parsed from filename (after first `-`) | `Support Agent`, `Sales Layout` |
| Sections | `<layoutSections>` | Count of sections |
| Related lists | `<relatedLists>` | List of related lists included |
| Quick actions | `<quickActionList>` | Actions available on the layout |
| Mini layout | `<miniLayout>` | Highlights panel fields |
| Feed layout | `<feedLayout>` | Chatter feed configuration |
| Platform action list | `<platformActionList>` | Lightning action bar |

### Layout Summary

Count sections and related lists. Do NOT list every field on the layout (too verbose). Focus on:
- Which objects have layouts in source
- Layout names (which audience/persona)
- Number of sections and related lists
- Quick actions included

---

## Discovery — Lightning Pages (Flexipages)

1. Glob for `**/flexipages/*.flexipage-meta.xml`.
2. For each flexipage, extract:

| Field | XML Element | Notes |
|-------|------------|-------|
| Name | Filename | API name |
| Label | `<masterLabel>` | Display name |
| Type | `<type>` | RecordPage, AppPage, HomePage, UtilityBar, FlowPage |
| Object | `<sobjectType>` | For RecordPage types |
| Template | `<template>` | Layout template used |
| Components | `<flexipageRegions><componentInstances>` | LWC/Aura components placed |

### Component Extraction

For each `<componentInstances>`:
- `<componentName>` — the component placed (e.g., `flexipage:reportChart`, `c__accountDashboard`)
- Custom LWC components are prefixed with namespace (usually `c__`)

This cross-references with the LWC catalog: which components are actually placed on Lightning Pages.

---

## Discovery — Tabs

1. Glob for `**/tabs/*.tab-meta.xml`.
2. For each tab, extract:

| Field | XML Element |
|-------|------------|
| Name | Filename |
| Label | `<label>` |
| Object | `<customObject>` (true/false) |
| Motif / Icon | `<motif>` |
| URL | `<url>` (for web tabs) |
| Flexipage | `<flexiPage>` (for Lightning page tabs) |
| LWC component | `<lwcComponent>` (for component tabs) |
| Description | `<description>` |

### Tab Types

| Has Element | Tab Type |
|-------------|----------|
| `<customObject>` = true | Object Tab (linked to custom object) |
| `<url>` present | Web Tab |
| `<flexiPage>` present | Lightning Page Tab |
| `<lwcComponent>` present | Lightning Component Tab |

---

## Discovery — Applications

1. Glob for `**/applications/*.app-meta.xml`.
2. For each application, extract:

| Field | XML Element |
|-------|------------|
| Name | Filename |
| Label | `<label>` |
| Description | `<description>` |
| Type | Infer from `<uiType>` or `<navType>` | Lightning (Console/Standard) or Classic |
| Tabs | `<tabs>` | Ordered list of tab names |
| Form factors | `<formFactors>` | Large, Small, Medium |
| Nav type | `<navType>` | Standard, Console |
| Utility bar | `<utilityBar>` | Associated utility bar |
| Brand | `<brand>` | Brand image/color |

---

## Discovery — Quick Actions

1. Glob for `**/quickActions/*.quickAction-meta.xml`.
2. For each quick action, extract:

| Field | XML Element |
|-------|------------|
| Name | Filename (format: `{Object}.{ActionName}` or just `{ActionName}`) |
| Label | `<label>` |
| Type | `<type>` (Create, Update, VisualforcePage, LightningComponent, Flow, SendEmail, etc.) |
| Target object | `<targetObject>` |
| Target record type | `<targetRecordType>` |
| LWC component | `<lightningWebComponent>` |
| Flow | `<flowDefinition>` |
| Description | `<description>` |
| Icon | `<icon>` |

---

## Discovery — Static Resources

1. Glob for `**/staticresources/*.resource-meta.xml`.
2. For each resource metadata file, extract:

| Field | XML Element |
|-------|------------|
| Name | Filename (without `.resource-meta.xml`) |
| Content type | `<contentType>` | application/zip, text/css, image/png, etc. |
| Cache control | `<cacheControl>` | Public or Private |
| Description | `<description>` |

3. Also check for the actual resource file (same name without `-meta.xml`) to get file size.

---

## Discovery — Lightning Message Channels (LMS)

1. Glob for `**/messageChannels/*.messageChannel-meta.xml`.
2. For each channel, extract:

| Field | XML Element |
|-------|------------|
| Name | Filename |
| Master label | `<masterLabel>` |
| Description | `<description>` |
| Exposed | `<isExposed>` |
| Fields | `<lightningMessageFields>` → `<fieldName>`, `<description>` |

Cross-reference with LWC scanner: which components import and publish/subscribe to each channel.

---

## Discovery — Email Templates

1. Glob for `**/email/**/*.email-meta.xml`.
2. For each template, extract:

| Field | XML Element |
|-------|------------|
| Name | `<fullName>` |
| Subject | `<subject>` |
| Description | `<description>` |
| Type | Classic, Visualforce, Lightning |
| Available for use | `<available>` |

---

## Output Format for ui-config.md

### Summary

```markdown
## Summary

| Category | Count |
|----------|-------|
| Applications | 2 |
| Lightning Pages | 8 |
| Page Layouts | 15 |
| Tabs | 6 |
| Quick Actions | 10 |
| Static Resources | 4 |
| Message Channels | 2 |
| Email Templates | 3 |
```

### Applications

```markdown
## Applications

| App | Type | Nav | Tabs | Description |
|-----|------|-----|------|-------------|
| `SalesApp` | Lightning | Standard | Home, Accounts, Contacts, Opportunities | Main sales application |
| `ServiceConsole` | Lightning | Console | Cases, Knowledge, Accounts | Support agent workspace |
```

### Lightning Pages

```markdown
## Lightning Pages

| Page | Type | Object | Custom Components |
|------|------|--------|-------------------|
| `Account_Record_Page` | RecordPage | Account | `c__accountDashboard`, `c__activityTimeline` |
| `Sales_Home` | HomePage | — | `c__salesMetrics`, `c__pipelineChart` |
| `Case_Console_Page` | RecordPage | Case | `c__caseEscalationPanel` |
```

### Page Layouts

```markdown
## Page Layouts

| Object | Layout | Sections | Related Lists |
|--------|--------|:--------:|:-------------:|
| Account | Account - Sales | 4 | 5 |
| Account | Account - Support | 3 | 3 |
| Case | Case - Support Agent | 5 | 4 |
| `Invoice__c` | Invoice - Finance | 3 | 2 |
```

### Tabs

```markdown
## Tabs

| Tab | Type | Target | Description |
|-----|------|--------|-------------|
| `CustomerAsset__c` | Object | CustomerAsset__c | — |
| `SalesDashboard` | Lightning Page | SalesDashboard flexipage | Sales team home dashboard |
| `PartnerPortal` | Web | https://partner.example.com | External partner site |
```

### Quick Actions

```markdown
## Quick Actions

| Action | Object | Type | Target | Description |
|--------|--------|------|--------|-------------|
| `Account.SendWelcomeEmail` | Account | SendEmail | — | Welcome email to new accounts |
| `Case.EscalateToManager` | Case | Flow | `Case_SCR_Escalation` | Launches escalation wizard |
| `NewTask` | Global | Create | Task | Quick task creation |
```

### Static Resources

```markdown
## Static Resources

| Resource | Content Type | Cache | Description |
|----------|-------------|:-----:|-------------|
| `chartjs` | application/zip | Public | Chart.js library for dashboards |
| `companyLogo` | image/png | Public | Company brand logo |
| `appStyles` | text/css | Public | Global custom styles |
```

### Message Channels

```markdown
## Lightning Message Channels

| Channel | Exposed | Fields | Publishers | Subscribers |
|---------|:-------:|:------:|-----------|------------|
| `RecordSelected__c` | Yes | recordId, objectApiName | `accountList` | `accountDetail`, `mapView` |
| `FilterChanged__c` | No | filterType, filterValue | `filterPanel` | `dataTable`, `chartPanel` |
```
