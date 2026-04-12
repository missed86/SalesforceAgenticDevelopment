# Objects and Fields Standards

## Custom Objects

### Naming Rules

| Rule | Example | Anti-Pattern |
|------|---------|-------------|
| PascalCase, no underscores | `CustomerAsset__c` | `Customer_Asset__c` |
| Singular form always | `Order__c` | `Orders__c` |
| Full words, no abbreviations | `ReverseMortgage__c` | `RevMtg__c` |
| Label matches API name | Label: `Customer Asset` | Label: `Client Thing` |

### Object Description Template

Every custom object description must answer:

```
Purpose: [One-sentence business reason for this object]
Owner: [Team or department that owns the data]
Relationships: [Parent objects or key lookups]
Automations: [Flows, triggers, or processes that act on this object]
```

Example:

```
Purpose: Tracks payment transactions between the company and vendors.
Owner: Finance department.
Relationships: Child of Account (Master-Detail), lookup to Invoice.
Automations: RT Flow sets PaymentStatus after save; Batch recalculates monthly totals.
```

### Custom Object vs Custom Metadata Type

| Use Custom Object when | Use CMDT when |
|------------------------|---------------|
| Data is transactional or user-entered | Data is config or admin-managed |
| Records have owners and sharing rules | Records are org-wide, no sharing needed |
| Needs reporting and list views | Needs deployment across environments |
| Volume grows over time | Small stable dataset under 1000 records |

---

## Custom Fields

### Naming Rules

| Rule | Example | Anti-Pattern |
|------|---------|-------------|
| PascalCase, no underscores | `PaymentStatus__c` | `Payment_Status__c` |
| Full words, no abbreviations | `ZipCode__c` | `ZC__c` |
| Boolean: `Is` or `Has` prefix | `IsActive__c`, `HasContract__c` | `Active__c` |
| English API name always | `CompanyName__c` | `NombreEmpresa__c` |

### Field Type Prefixes (Optional)

When an org has 50+ fields per object, use type-hinting prefixes to improve
readability in Setup:

| Field Type | Prefix | Example |
|-----------|--------|---------|
| Lookup or Master-Detail | `Ref` suffix | `PrimaryContactRef__c` |
| Formula | `Auto` prefix | `AutoTotalWithTax__c` |
| Rollup Summary | `Auto` prefix | `AutoTotalLineItems__c` |
| Filled by automation | `Trig` prefix | `TrigLastProcessedDate__c` |
| Boolean | `Is` `Has` `Can` | `IsApproved__c` |

Prefixes are optional. For orgs with fewer fields, plain PascalCase is
cleaner. Choose one convention and apply it consistently.

### Multi-Service Orgs

When multiple apps share the same object, prefix fields with the app name
separated by an underscore to avoid collisions:

```
Billing_InvoiceNumber__c
Shipping_TrackingCode__c
```

This is the only case where an underscore between words is acceptable.
The underscore separates the app namespace from the field name.

### Grouping Fields (50+ per Object)

Group related fields with consistent prefixes so they sort together in Setup:

```
Address_Street__c
Address_City__c
Address_PostalCode__c
Address_Country__c
```

---

## Description Standards

### Field Description (1000 chars, admins only)

Every custom field description must include:

| Section | Content | Example |
|---------|---------|---------|
| What | What the field stores | Stores the payment status of the transaction |
| Who | Who or what populates it | Set by Finance team; updated by RT Flow |
| Why | Business reason | Required for month-end reconciliation |
| Deps | What relies on it | Used in VR02; referenced in MonthlyRecon report |

Full example:

```
Stores the current payment status (Pending, Approved, Rejected).
Populated by Finance team during review; set to Pending automatically
by Account_RT_AfterSave_SetDefaults Flow on creation.
Required for month-end reconciliation reporting. Drives the
PaymentApproval approval process. Referenced in VR03_Payment_StatusRequired.
```

### Field Help Text (255 chars, end users)

Good:

```
Enter the 10-digit purchase order number from the vendor invoice.
```

```
Select the region where customer headquarters is located.
```

Bad:

```
Enter the account name.    (repeats the label)
This is the status field.  (says nothing useful)
```

---

## Record Types

### Naming Rules

| Rule | Example | Anti-Pattern |
|------|---------|-------------|
| PascalCase API name | `InternalTransfer` | `Internal_Transfer` |
| Descriptive label | `Internal Transfer` | `Type A` |
| Keep it concise | `Warranty` | `WarrantyRecordType` |

### Record Type Description

Must explain:
- What distinguishes this record type from others
- Which page layout and business process it drives
- Which picklist values are available

```
Records created for internal stock transfers between warehouses.
Uses the InternalTransfer page layout (hides shipping cost fields).
Available status values: Draft, Submitted, Approved, Completed.
```

---

## Platform Events and Custom Metadata Types

### Platform Events

| Rule | Example |
|------|---------|
| PascalCase, `__e` suffix | `OrderEvent__e` |
| Fields: PascalCase | `OrderId__c`, `Status__c` |
| Description includes publisher and subscriber | Published by OrderService; consumed by LogEventSubscriber |

### Custom Metadata Types

| Rule | Example |
|------|---------|
| PascalCase, `__mdt` suffix | `DiscountRule__mdt` |
| Fields: PascalCase | `MinAmount__c`, `IsActive__c` |
| Description includes deployment context | Deployed via change sets; controls discount tiers |

---

## Custom Labels

### Naming Rules

| Rule | Example | Anti-Pattern |
|------|---------|-------------|
| PascalCase | `ErrorOrderNotFound` | `Error_Order_Not_Found` |
| Category prefix for grouping | `ErrorPaymentFailed` | `Label1` |
| Use the Categories field | Category: `Error Messages` | No category |

### Category Strategy

Use the built-in Categories field (comma-separated) to create List Views:

| Category | Usage | Examples |
|----------|-------|---------|
| Error | Error messages | `ErrorOrderNotFound` |
| Success | Confirmation messages | `SuccessOrderCreated` |
| UI | Button labels, headers | `UIButtonSubmit` |
| Notification | Email and alert text | `NotificationCaseEscalated` |
| Validation | VR error messages | `ValidationPhoneRequired` |

### Label Description

Must explain where the label is used (LWC, VR, email template)
and provide context for translators:

```
Error shown in OrderEntry LWC when order ID from URL has no match.
Translators: keep the tone professional and helpful.
```

---

## Limits Awareness

| Metadata Type | Per-Org Limit | Action |
|---------------|--------------|--------|
| Custom Objects | 2000 (EE+) | Audit quarterly for unused |
| Custom Fields per Object | 800 (EE+) | Archive fields with 0% population |
| Validation Rules per Object | 500 | Consolidate overlapping rules |
| Flows (active) | 2000 | Deactivate obsolete versions |
| Custom Labels | 5000 | Use categories; merge duplicates |
| Custom Metadata Types | 200 types | Combine related configs |
| CMDT Records | 10 MB total | Monitor in Setup |
| Permission Sets | No hard limit | Audit for orphaned sets |
