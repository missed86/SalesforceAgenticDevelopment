# Security & Access Scanner

Detailed instructions for scanning profiles, permission sets, permission set groups, sharing rules, queues, public groups, and custom permissions.

## Discovery — Profiles

1. Glob for `**/profiles/*.profile-meta.xml` across all package directories.
2. Profile XMLs can be very large. Read selectively — extract these elements:

| Field | XML Element | Notes |
|-------|------------|-------|
| Name | Filename (without extension) | `Admin`, `Standard`, `Custom Sales Profile` |
| User license | `<userLicense>` | Salesforce, Salesforce Platform, etc. |
| Custom | `<custom>` | true = custom profile, false = standard |
| Description | `<description>` | May not exist in older profiles |
| Tab visibilities | `<tabVisibilities>` | Which tabs are visible/hidden |
| Object permissions | `<objectPermissions>` | CRUD per object |
| Field permissions | `<fieldPermissions>` | Read/Edit per field |
| Page layout assignments | `<layoutAssignments>` | Which layout per record type |
| Record type visibilities | `<recordTypeVisibilities>` | Available and default RTs |
| Application visibilities | `<applicationVisibilities>` | App access |
| Class accesses | `<classAccesses>` | Apex class access |
| Custom permissions | `<customPermissions>` | Custom permission assignments |
| Login IP ranges | `<loginIpRanges>` | IP restrictions |
| Login hours | `<loginHours>` | Time-based restrictions |

### Profile Summary Strategy

Do NOT document every field permission (profiles can have thousands). Instead:

1. Count total object permissions granted (CRUD)
2. Count total field permissions
3. List custom objects with access
4. Note any login restrictions
5. List Apex class and page accesses

---

## Discovery — Permission Sets

1. Glob for `**/permissionsets/*.permissionset-meta.xml`.
2. For each permission set XML, extract:

| Field | XML Element |
|-------|------------|
| Name | Filename |
| Label | `<label>` |
| Description | `<description>` |
| License | `<license>` |
| Has activate session | `<hasActivationRequired>` |
| Object permissions | `<objectPermissions>` |
| Field permissions | `<fieldPermissions>` |
| Apex class accesses | `<classAccesses>` |
| Tab settings | `<tabSettings>` |
| Custom permissions | `<customPermissions>` |
| Record type assignments | `<recordTypeVisibilities>` |
| Page accesses | `<pageAccesses>` |
| User permissions | `<userPermissions>` |

### Permission Set Analysis

For each permission set, summarize:
- Objects it grants access to (list object names + CRUD levels)
- Number of field permissions
- Apex classes enabled
- Custom permissions granted
- Whether it requires a specific license

---

## Discovery — Permission Set Groups

1. Glob for `**/permissionsetgroups/*.permissionsetgroup-meta.xml`.
2. Extract:

| Field | XML Element |
|-------|------------|
| Name | Filename |
| Label | `<label>` |
| Description | `<description>` |
| Status | `<status>` |
| Has activation required | `<hasActivationRequired>` |
| Permission sets | `<permissionSets>` | List of included PS names |
| Muting PS | `<mutingPermissionSets>` | Permission set that removes access |

---

## Discovery — Custom Permissions

1. Glob for `**/customPermissions/*.customPermission-meta.xml`.
2. Extract:

| Field | XML Element |
|-------|------------|
| Name | Filename |
| Label | `<label>` |
| Description | `<description>` |
| Connected app | `<connectedApp>` |
| Required permission | `<requiredPermission>` |

Cross-reference: search Apex code for `FeatureManagement.checkPermission('PermName')` to find where custom permissions are enforced.

---

## Discovery — Sharing Rules

1. Glob for `**/sharingRules/*.sharingRules-meta.xml`.
2. Sharing rules XML is per-object. Parse:

### Criteria-Based Sharing Rules

Look for `<sharingCriteriaRules>`:

| Field | XML Element |
|-------|------------|
| Name | `<fullName>` |
| Label | `<label>` |
| Access level | `<accessLevel>` |
| Shared to | `<sharedTo>` (group, role, roleAndSubordinates, etc.) |
| Criteria items | `<criteriaItems>` |
| Description | `<description>` |

### Owner-Based Sharing Rules

Look for `<sharingOwnerRules>`:

| Field | XML Element |
|-------|------------|
| Name | `<fullName>` |
| Label | `<label>` |
| Access level | `<accessLevel>` |
| Shared from | `<sharedFrom>` |
| Shared to | `<sharedTo>` |

---

## Discovery — Queues

1. Glob for `**/queues/*.queue-meta.xml`.
2. Extract:

| Field | XML Element |
|-------|------------|
| Name | `<fullName>` |
| Queue email | `<queueSobject>` (lists supported objects) |
| Members | `<queueMembers>` |

---

## Discovery — Public Groups

1. Glob for `**/groups/*.group-meta.xml`.
2. Extract:

| Field | XML Element |
|-------|------------|
| Name | `<fullName>` |
| Label | `<label>` |
| Does include bosses | `<doesIncludeBosses>` |

---

## Output Format for security-model.md

### Summary

```markdown
## Summary

| Category | Count |
|----------|-------|
| Profiles (in source) | 3 |
| Permission Sets | 8 |
| Permission Set Groups | 2 |
| Custom Permissions | 4 |
| Sharing Rules | 6 |
| Queues | 3 |
| Public Groups | 2 |
```

### Profiles

```markdown
## Profiles

| Profile | Type | License | Objects with Access | Field Perms | Apex Classes | IP Restricted |
|---------|------|---------|--------------------:|------------:|-------------:|:------------:|
| `Admin` | Standard | Salesforce | 45 | 312 | 28 | No |
| `SalesUser` | Custom | Salesforce | 12 | 87 | 5 | Yes |
```

### Permission Sets

```markdown
## Permission Sets

| Permission Set | Label | License | Description |
|----------------|-------|---------|-------------|
| `PS_EditAccounts` | Edit Accounts | — | Grants edit access to Account and related fields |
| `PS_ManageInvoices` | Manage Invoices | — | Full CRUD on Invoice__c, read on Payment__c |

### PS_EditAccounts

**Description:** Grants edit access to Account and related fields for sales operations.

**Object Permissions:**

| Object | Create | Read | Update | Delete |
|--------|:------:|:----:|:------:|:------:|
| Account | ✅ | ✅ | ✅ | ❌ |
| Contact | ❌ | ✅ | ✅ | ❌ |

**Apex Classes:** `AccountController`, `AccountService`
**Custom Permissions:** `CanEditAccountOwner`
```

### Permission Set Groups

```markdown
## Permission Set Groups

| Group | Label | Status | Included Permission Sets |
|-------|-------|--------|------------------------|
| `PSG_SalesRepresentative` | Sales Representative | Active | `PS_EditAccounts`, `PS_ViewReports`, `PS_CreateOpportunities` |
| `PSG_FinanceAnalyst` | Finance Analyst | Active | `PS_ViewInvoices`, `PS_RunFinanceReports` |
```

### Security Architecture Diagram

If permission set groups exist, show the hierarchy:

```markdown
## Security Architecture

PSG_SalesRepresentative
├── PS_EditAccounts
├── PS_ViewReports
└── PS_CreateOpportunities

PSG_FinanceAnalyst
├── PS_ViewInvoices
├── PS_RunFinanceReports
└── [Muting] MPS_RestrictDeleteInvoice
```

### Sharing Rules

```markdown
## Sharing Rules

### Account Sharing Rules

| Rule | Type | Access | Shared To | Criteria |
|------|------|--------|-----------|----------|
| `SR_Account_RegionalAccess` | Criteria | Read/Write | Role: Regional Manager | Region__c = 'West' |
| `SR_Account_PartnerView` | Owner | Read Only | Group: Partner Users | Owner in Partner Role |
```

### Queues

```markdown
## Queues

| Queue | Supported Objects | Members |
|-------|------------------|---------|
| `Q_Support_Cases` | Case | Support Team role |
| `Q_Finance_Invoices` | Invoice__c | Finance Group |
```

### Observed Naming Patterns

Detect the actual naming patterns the project uses for security metadata. Look for:
- Do permission sets use a prefix (e.g. `PS_`, `Perm_`, or none)?
- Do PSGs use a prefix (e.g. `PSG_`, or none)?
- Do queues include an object or team reference?
- Do sharing rules follow a consistent structure?

Document the **detected patterns** and note any items that deviate from the project's own convention:

```markdown
## Security Naming Patterns

| Category | Detected Pattern | Consistency |
|----------|-----------------|:-----------:|
| Permission Sets | `PS_{Capability}` prefix | 7/8 follow pattern |
| Permission Set Groups | `PSG_{Persona}` prefix | 2/2 |
| Queues | `Q_{Team}_{Object}` | 3/3 |
| Sharing Rules | No consistent pattern detected | — |
```
