---
name: salesforce-metadata-standards
description: >-
  Salesforce declarative metadata naming conventions, description standards,
  and governance rules. Covers custom objects, fields, validation rules, Flows,
  permission sets, sharing rules, custom labels, page layouts, record types,
  reports, dashboards, approval processes, queues, and public groups.
  Use when creating or reviewing Salesforce metadata, custom objects, fields,
  Flows, permission sets, validation rules, or any declarative configuration.
---

# Salesforce Metadata Standards

Naming conventions, description requirements, and governance rules for all
declarative Salesforce metadata. No abbreviations. No ambiguity. Consistent
across every sandbox and production org.

## Universal Rules

1. **PascalCase** for all API names — no underscores except the Salesforce suffix (`__c`, `__mdt`, `__e`)
2. **English API names always** — labels can be localized, API names cannot
3. **Singular form** for objects — `Order` not `Orders`, `Invoice` not `Invoices`
4. **Descriptions are mandatory** — every custom object, field, validation rule, and Flow must have one
5. **No abbreviations** unless universally understood (`Id`, `URL`, `API`, `HTTP`)

## Quick Naming Reference

### Objects & Fields

| Element | Convention | Example |
|---------|-----------|---------|
| Custom Object | PascalCase, singular | `CustomerAsset__c` |
| Custom Field | PascalCase | `PaymentStatus__c` |
| Lookup / MD | PascalCase | `PrimaryContact__c` |
| Formula field | PascalCase | `TotalWithTax__c` |
| Boolean field | `Is` / `Has` prefix | `IsActive__c`, `HasContract__c` |
| Custom Metadata Type | PascalCase | `DiscountRule__mdt` |
| Platform Event | PascalCase | `OrderEvent__e` |

### Automation

| Element | Convention | Example |
|---------|-----------|---------|
| Validation Rule | `VR{NN}_{Object}_{Purpose}` | `VR01_Opp_CancelReasonRequired` |
| Record-Triggered Flow | `{Object}_RT_{Timing}_{Action}` | `Account_RT_BeforeSave_SetDefaults` |
| Screen Flow | `{Object}_SCR_{Purpose}` | `Case_SCR_EscalationWizard` |
| Scheduled Flow | `{Object}_SCH_{Purpose}` | `Contact_SCH_BirthdayEmails` |
| Autolaunched Flow | `{Object}_AL_{Purpose}` | `Order_AL_CalculateTotals` |
| Subflow | `{Object}_SFL_{Purpose}` | `Account_SFL_ValidateAddress` |
| Approval Process | Describes entry criteria | `OpportunityDiscountApproval` |
| Approval Step | Describes outcome | `AutoApproved_WithinLimit` |

### Security & Access

| Element | Convention | Example |
|---------|-----------|---------|
| Permission Set | `PS_{Capability}` | `PS_EditAccounts` |
| Permission Set Group | `PSG_{Persona}` | `PSG_SalesRepresentative` |
| Public Group | `GRP_{Team}_{Purpose}` | `GRP_Finance_Approvers` |
| Queue | `Q_{Team}_{Object}` | `Q_Support_Cases` |
| Sharing Rule | `SR_{Object}_{Criteria}` | `SR_Account_RegionalAccess` |

### UI & Reporting

| Element | Convention | Example |
|---------|-----------|---------|
| Page Layout | `{Object} - {Audience}` | `Case - Support Agent` |
| Lightning Page | `{Object}RecordPage_{Context}` | `AccountRecordPage_Sales` |
| Record Type | PascalCase, descriptive | `InternalTransfer` |
| Custom Label | PascalCase, prefixed by category | `ErrorOrderNotFound` |
| Report Folder | `{Team} - {Topic}` | `Sales - Pipeline Reports` |
| Dashboard Folder | `{Team} - {Topic}` | `Sales - Pipeline Dashboards` |
| Email Template | `{Process}_{Action}` | `CaseEscalation_NotifyManager` |

## Description & Help Text Rules

| Element | Description | Help Text |
|---------|------------|-----------|
| **Who sees it** | Admins only (Setup) | End users (field hover) |
| **Character limit** | 1,000 chars | 255 chars |
| **Required?** | Always on custom elements | When field purpose isn't obvious |
| **Content** | Why it exists, business context, dependencies | How to fill it in, expected format |

### Description Must Include

- **Why** the element exists (business reason)
- **Who** uses it (team/department)
- **Dependencies**: automations, integrations, or processes that rely on it
- **Data source**: if populated by automation or integration, say which one

### Help Text Must Include

- What value to enter and in what format
- Where to find the information
- Keep it action-oriented: "Enter the 10-digit order number from the ERP system"

## Pattern Reference (Progressive Disclosure)

| Standard | File | When to Read |
|----------|------|-------------|
| Objects & Fields | [objects-and-fields.md](standards/objects-and-fields.md) | Creating objects, fields, descriptions, help text |
| Automation | [automation.md](standards/automation.md) | Flows, validation rules, approval processes |
| Security & Access | [security-and-access.md](standards/security-and-access.md) | Permission sets, groups, queues, sharing rules |

## Anti-Patterns to Flag

| Anti-Pattern | Problem | Fix |
|-------------|---------|-----|
| No description on custom field | No one knows why it exists in 6 months | Always write description with business context |
| Abbreviated object/field names | `CustAsst`, `OrdItm` — unreadable | Use full words: `CustomerAsset`, `OrderItem` |
| Plural object names | `Orders__c` — breaks Salesforce conventions | Singular: `Order__c` |
| Underscores in API names | `Payment_Status__c` — not PascalCase | `PaymentStatus__c` |
| Generic validation rule names | `Validate Address` — which validation? | `VR01_Contact_AddressRequired` with error code |
| Flow named `My Flow` or `New Flow 2` | Impossible to find or understand | `Account_RT_AfterSave_NotifyOwner` |
| Permission Set named `Custom PS 1` | No indication of what it grants | `PS_ManageInvoices` |
| No VR bypass mechanism | Can't load data without disabling rules | Add `IsBypassVR__c` checkbox on User |
| Dead metadata (unused fields/flows) | Clutter, confusion, limit consumption | Audit quarterly, deactivate or delete |
| Help text that repeats the label | `Account Name: Enter the account name` | Explain format, source, or business context |
