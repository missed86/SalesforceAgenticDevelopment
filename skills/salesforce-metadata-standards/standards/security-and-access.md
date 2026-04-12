# Security and Access Standards

## Permission Sets

### Naming Convention

Pattern: `PS_{Capability}`

Permission Sets describe a single capability or access level:

| Example | Grants |
|---------|--------|
| `PS_EditAccounts` | Edit access to Account records |
| `PS_ManageInvoices` | Full CRUD on Invoice object |
| `PS_ViewSensitiveFields` | Read access to restricted fields |
| `PS_RunReports` | Access to run reports in specific folders |
| `PS_ApproveDiscounts` | Custom permission for discount approval |
| `PS_APIIntegration` | Object and field access for integrations |

### Naming Rules

| Rule | Example | Anti-Pattern |
|------|---------|-------------|
| `PS_` prefix always | `PS_EditAccounts` | `EditAccounts` (no prefix) |
| Verb + Noun | `PS_ManageInvoices` | `PS_Invoice` (ambiguous) |
| PascalCase after prefix | `PS_ViewSensitiveFields` | `PS_view_sensitive_fields` |
| No user names | `PS_ApproveDiscounts` | `PS_JohnSmithAccess` |
| No team names | `PS_ManageInvoices` | `PS_FinanceTeam` (use PSG) |

### Permission Set Description

Must include:
- What access it grants (objects, fields, permissions)
- Why it exists (business justification)
- Which Permission Set Group(s) include it

```
Grants Edit access to Account, Contact, and Opportunity objects.
Includes FLS for all standard and custom fields except sensitive financial fields.
Included in PSG_SalesRepresentative and PSG_AccountManager groups.
Business justification: Required for day-to-day sales operations.
```

---

## Permission Set Groups

### Naming Convention

Pattern: `PSG_{Persona}`

Permission Set Groups represent a job function or persona:

| Example | Persona |
|---------|---------|
| `PSG_SalesRepresentative` | Field sales team member |
| `PSG_AccountManager` | Key account manager |
| `PSG_SupportAgent` | Customer support rep |
| `PSG_FinanceAnalyst` | Finance team member |
| `PSG_SystemIntegration` | Integration user persona |
| `PSG_ReadOnlyAuditor` | External auditor access |

### Design Principles

1. **One PSG per job function**: do not create PSGs per individual
2. **Compose from Permission Sets**: each PSG bundles multiple PS_* sets
3. **Use Muting Permission Sets** to remove specific permissions within a group
4. **Assignment expiration** for temporary access (contractor, audit)

### PSG Architecture Example

```
PSG_SalesRepresentative
  |-- PS_EditAccounts
  |-- PS_EditContacts
  |-- PS_EditOpportunities
  |-- PS_ViewReports
  |-- PS_UseQuoteBuilder
  |-- [Muting] MPS_BlockSensitiveFields
```

### PSG Description

Must include:
- Target persona and department
- List of included Permission Sets
- Any muting sets applied
- Expiration policy if applicable

```
Persona: Sales Representative (Field Sales team).
Includes: PS_EditAccounts, PS_EditContacts, PS_EditOpportunities,
PS_ViewReports, PS_UseQuoteBuilder.
Muting: MPS_BlockSensitiveFields (blocks Revenue__c, Margin__c).
Expiration: None (permanent assignment).
```

---

## Profiles

### Best Practices

- Use minimum custom profiles; prefer generic base profiles
- Layer all specific access through Permission Sets and PSGs
- Standard profiles to use as base: `Minimum Access - Salesforce`, `Standard User`
- Never grant `Modify All Data` or `View All Data` on profiles

### Profile Naming

| Pattern | Example |
|---------|---------|
| `{BaseLevel} User` | `Standard User` |
| `{App} Platform User` | `Integration Platform User` |
| `Minimum Access` | `Minimum Access - Salesforce` |

---

## Public Groups

### Naming Convention

Pattern: `GRP_{Application}_{Purpose}`

| Example | Usage |
|---------|-------|
| `GRP_Finance_Approvers` | Sharing and approval routing |
| `GRP_Sales_WestRegion` | Criteria-based sharing |
| `GRP_Support_EscalationTeam` | Email alerts and case routing |
| `GRP_Compliance_Auditors` | Report folder access |

### Naming Rules

| Rule | Example | Anti-Pattern |
|------|---------|-------------|
| `GRP_` prefix always | `GRP_Finance_Approvers` | `Finance Approvers` |
| Application then purpose | `GRP_Sales_WestRegion` | `GRP_WestSales` |
| Label matches business language | Label: `Finance Approvers` | Label: `GRP_FIN_APR` |

### Public Group Description

Must include:
- Business purpose of the group
- What sharing rules or features use it
- Who manages membership (admin, manager, automated)

```
Groups all Finance department members who can approve purchase orders.
Used in: SR_PurchaseOrder_FinanceAccess sharing rule,
PurchaseOrderApproval approval process Step 2.
Membership managed by Finance department manager.
```

---

## Queues

### Naming Convention

Pattern: `Q_{Team}_{Object}`

| Example | Usage |
|---------|-------|
| `Q_Support_Cases` | Case ownership for support team |
| `Q_Sales_Leads` | Lead ownership for sales team |
| `Q_Billing_Orders` | Order ownership for billing |
| `Q_Compliance_Tasks` | Task ownership for compliance |

### Queue vs Public Group

| Use Queue when | Use Public Group when |
|----------------|----------------------|
| Need record ownership | Only need sharing/visibility |
| Queue appears in list views as owner | Used in sharing rules or approvals |
| Round-robin or manual assignment from queue | No ownership transfer needed |
| Users work records from a shared pool | Users need read access only |

### Queue Description

Must include:
- Which object(s) the queue supports
- Business process that routes records to this queue
- Assignment rules or lead assignment that feed it

```
Owns unassigned Case records for the Tier 1 Support team.
Fed by Case Assignment Rules (Region = Americas, Priority != Critical).
Agents pull cases from the queue list view for self-assignment.
Supported objects: Case.
```

---

## Sharing Rules

### Naming Convention

Pattern: `SR_{Object}_{Criteria}`

| Type | Example | Shares With |
|------|---------|------------|
| Owner-based | `SR_Account_SalesManagerAccess` | Role: Sales Managers |
| Criteria-based | `SR_Account_RegionalAccess` | Group: GRP_Sales_WestRegion |
| Guest user | `SR_Case_PortalCaseAccess` | Guest User Sharing Rule |

### Sharing Rule Description

Must include:
- What records are shared (criteria or owner filter)
- Who gains access (group, role, territory)
- Access level granted (Read Only or Read/Write)
- Business justification

```
Shares Account records where BillingCountry = 'US' with
GRP_Sales_WestRegion group. Access: Read/Write.
Business reason: West Region sales team needs to collaborate
on all US-based accounts regardless of ownership.
```

### Sharing Best Practices

- Prefer criteria-based sharing rules over owner-based when possible
- Use Public Groups as targets (not individual roles) for maintainability
- Document all sharing rules in a central matrix (spreadsheet or Confluence)
- Test sharing with `System.runAs()` in Apex tests
- Avoid `without sharing` in Apex unless documented and justified
- Review sharing rules quarterly for relevance

---

## Page Layouts

### Naming Convention

Pattern: `{Object} - {Audience}`

| Example | Audience |
|---------|----------|
| `Case - Support Agent` | Standard support team |
| `Case - Support Manager` | Support management |
| `Account - Sales` | Sales team |
| `Account - Finance` | Finance team view |
| `Opportunity - Partner` | Partner community users |

### Page Layout Best Practices

- Assign layouts via Record Type + Profile combination
- Use Related Lists strategically (show only relevant child objects)
- Group fields into logical sections with clear section headers
- Hide fields that the audience never needs (reduce clutter)
- Use Dynamic Forms on Lightning pages instead of multiple layouts when possible

---

## Lightning Record Pages

### Naming Convention

Pattern: `{Object}RecordPage_{Context}`

| Example | Context |
|---------|---------|
| `AccountRecordPage_Sales` | Sales team view |
| `CaseRecordPage_Support` | Support agent view |
| `AccountRecordPage_Default` | Default for all profiles |
| `OpportunityRecordPage_Partner` | Partner community |

### Lightning Page Description

Must include which Record Type or App it is assigned to and what
components are visible:

```
Default Account record page for Sales app.
Assigned to: Sales profiles via App Default.
Key components: Account details, Activity Timeline, Related Contacts,
Open Opportunities, Account Health Score (custom LWC).
```

---

## Reports and Dashboards

### Folder Naming

Pattern: `{Team} - {Topic} {Type}`

| Example | Contains |
|---------|----------|
| `Sales - Pipeline Reports` | Pipeline and forecast reports |
| `Sales - Pipeline Dashboards` | Pipeline visualizations |
| `Finance - Monthly Close Reports` | Month-end reconciliation |
| `Support - SLA Dashboards` | Service level tracking |
| `Admin - Data Quality Reports` | Duplicate detection, completeness |

### Folder Best Practices

- Separate folders for Reports and Dashboards (even same topic)
- Use Enhanced Folder Sharing for access control
- Grant folder edit access via Permission Sets, not broad profile permissions
- Purge unused reports quarterly (check Last Run Date)
- Name reports descriptively: `Pipeline by Stage and Region Q1 2026`
- Avoid personal report folders for team-shared content

---

## Email Templates

### Naming Convention

Pattern: `{Process}_{Action}`

| Example | Usage |
|---------|-------|
| `CaseEscalation_NotifyManager` | Escalation alert to manager |
| `OrderConfirmation_CustomerReceipt` | Order confirmation to customer |
| `ApprovalRequest_DiscountReview` | Approval notification |
| `WelcomeEmail_NewContact` | Onboarding email |

### Email Template Best Practices

- Store in shared folders, never personal folders
- Use merge fields for personalization (never hardcode names or values)
- Include the template name in the subject line prefix for traceability (optional)
- Test rendering in both Classic and Lightning
- Use Enhanced Letterhead for consistent branding
- Document the template purpose in the Description field

---

## Governance Summary

### Quarterly Audit Checklist

- [ ] Remove or deactivate unused Permission Sets
- [ ] Review PSG composition (do all PS still belong?)
- [ ] Audit Public Group membership (remove departed employees)
- [ ] Verify Queue assignment rules still match business process
- [ ] Review sharing rules against current org chart
- [ ] Purge unused reports (no runs in 90+ days)
- [ ] Check for orphaned page layouts not assigned to any Record Type
- [ ] Validate all descriptions are current and accurate
