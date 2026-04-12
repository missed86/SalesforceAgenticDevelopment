# Automation Standards

## Validation Rules

### Naming Convention

Pattern: `VR{NN}_{ObjectShorthand}_{Purpose}`

| Component | Rule | Example |
|-----------|------|---------|
| Prefix | Always `VR` + 2-digit number | `VR01`, `VR12` |
| Object | Short recognizable name | `Opp`, `Acc`, `Con`, `Case` |
| Purpose | What the rule prevents | `CancelReasonRequired` |

Full examples:

```
VR01_Opp_CancelReasonRequired
VR02_Opp_NoApprovalCantAdvance
VR03_Con_PhoneOrEmailRequired
VR04_Acc_BillingAddressComplete
```

### Error Message Standard

Every VR error message must include the VR code in brackets at the end:

```
If you select 'Other' as a cancellation reason, you must provide details. [VR01]
```

```
The status cannot advance without approval. [VR02]
```

This lets users and admins trace errors back to the exact validation rule.

### VR Description

Must explain the business use case, not the technical formula:

Good:
```
Prevents opportunities from being closed as Lost without providing a
cancellation reason. Required by Sales Operations for quarterly loss analysis.
```

Bad:
```
Checks if Cancellation_Reason is null when StageName equals Closed Lost.
```

### Bypass Mechanism

Every validation rule formula must include a bypass check:

```
AND(
  NOT($User.IsBypassVR__c),
  /* actual validation logic here */
)
```

Implementation options (from simple to granular):

| Mechanism | Setup | Use Case |
|-----------|-------|----------|
| `IsBypassVR__c` checkbox on User | Single checkbox | Data loads, migrations |
| Hierarchical Custom Setting | Custom Setting with checkbox per rule group | Per-team bypass |
| Custom Permission | Permission Set assignable | Temporary bypass via PSG |

### VR Best Practices

- One rule per business concern (do not combine unrelated validations)
- Never cascade VRs (VR2 should not depend on VR1 passing first)
- Show errors at field level when possible, page level only for cross-field rules
- Test every VR in sandbox before deploying to production
- Deactivate (do not delete) deprecated rules for audit trail

---

## Flows

### Naming Convention by Type

| Flow Type | Pattern | Example |
|-----------|---------|---------|
| Record-Triggered | `{Object}_RT_{Timing}_{Action}` | `Account_RT_BeforeSave_SetDefaults` |
| Screen Flow | `{Object}_SCR_{Purpose}` | `Case_SCR_EscalationWizard` |
| Scheduled | `{Object}_SCH_{Purpose}` | `Contact_SCH_BirthdayEmails` |
| Autolaunched | `{Object}_AL_{Purpose}` | `Order_AL_CalculateTotals` |
| Subflow | `{Object}_SFL_{Purpose}` | `Account_SFL_ValidateAddress` |
| Platform Event-Triggered | `{Event}_EVT_{Action}` | `OrderEvent_EVT_ProcessOrder` |

### Timing Indicators for Record-Triggered Flows

| Timing | Indicator | When to Use |
|--------|-----------|-------------|
| Before Save | `BeforeSave` | Set defaults, validate, transform (no DML) |
| After Save | `AfterSave` | Create related records, call subflows |
| Before Delete | `BeforeDelete` | Prevent deletion, archive data |
| After Delete | `AfterDelete` | Clean up related data |

### Flow Element Naming

Every element inside a Flow must have a descriptive name:

| Element Type | Prefix | Example |
|-------------|--------|---------|
| Get Records | `Get_` | `Get_PrimaryContact` |
| Create Records | `Create_` | `Create_FollowUpTask` |
| Update Records | `Update_` | `Update_AccountStatus` |
| Delete Records | `Delete_` | `Delete_OldAttachments` |
| Decision | `Decision_` | `Decision_IsHighValue` |
| Assignment | `Assign_` | `Assign_DefaultOwner` |
| Loop | `Loop_` | `Loop_EachLineItem` |
| Subflow | `Sub_` | `Sub_ValidateAddress` |
| Action | `Action_` | `Action_SendEmail` |
| Screen | `Screen_` | `Screen_EnterDetails` |

Anti-patterns for element names:

```
Decision_3          (auto-generated, meaningless)
myDecision          (no prefix, vague)
Loop 1              (auto-generated)
```

### Flow Description

Every Flow must have a description that includes:

```
Purpose: [What the Flow does]
Entry Criteria: [When it fires or who launches it]
Owner: [Team responsible]
Related: [Other Flows, VRs, or Apex it interacts with]
Version: [If versioned, current version number and date]
```

Example:

```
Purpose: Sets default field values on new Account records.
Entry Criteria: Record-Triggered, Before Save, on Create.
Owner: Sales Operations.
Related: Works with VR04_Acc_BillingAddressComplete.
```

### Flow Best Practices

- One Flow per object per timing context (combine logic, do not create duplicates)
- Use subflows for reusable logic shared across multiple Flows
- Never hardcode IDs or record type names; use Custom Metadata or Custom Labels
- Use fault paths on every DML element to handle errors gracefully
- Keep Screen Flows under 10 screens; break complex wizards into subflows
- Version management: use the Flow description field for change notes

### Flow vs Apex Decision

| Use Flow when | Use Apex when |
|---------------|---------------|
| Simple field updates or record creation | Complex logic with multiple conditionals |
| Admin-maintainable logic | Callouts to external systems |
| Under 10 DML operations | Bulk processing over 200 records |
| Screen-based user interaction | High-performance batch operations |
| No complex error handling needed | Custom error handling with try-catch |

---

## Approval Processes

### Naming Convention

| Element | Pattern | Example |
|---------|---------|---------|
| Process Name | `{Object}{BusinessAction}Approval` | `OpportunityDiscountApproval` |
| Step Name | `{Outcome}_{Condition}` | `AutoApproved_WithinLimit` |
| Field Update | `Set_{Field}_{Value}` | `Set_Status_Approved` |
| Email Alert | `Email_{Recipient}_{Context}` | `Email_Manager_PendingApproval` |

### Approval Step Naming

Steps must be self-documenting:

Good:
```
AutoApproved_ValueWithinPersonalLimit
AutoRejected_ExceedsCompanyPolicy
SentToManager_StandardApproval
SentToLegal_HighValueDeal
```

Bad:
```
Step 1
Step 2
ComplexApproval_Step1
```

### Approval Process Description

Must include:
- Entry criteria (when does submission happen)
- Approver chain (who approves at each level)
- Final actions on approve and reject
- Related Field Updates and Email Alerts

---

## Email Alerts and Field Updates

### Email Alert Naming

Pattern: `Email_{Recipient}_{Context}`

| Example | Meaning |
|---------|---------|
| `Email_DeceasedCustomerTeam_NewDeceased` | Notifies the deceased customer group |
| `Email_Manager_ApprovalRequired` | Sends approval request to manager |
| `Email_Owner_CaseEscalated` | Notifies case owner of escalation |

### Field Update Naming

Pattern: `Set_{FieldName}_{Value}`

| Example | Meaning |
|---------|---------|
| `Set_InactiveFlag_True` | Sets the inactive checkbox to true |
| `Set_Status_Approved` | Changes status picklist to Approved |
| `Set_ReviewDate_Today` | Sets review date to current date |

### Description Requirements

Both Email Alerts and Field Updates must have descriptions that explain:
- What triggers this action
- Business reason for the action
- Dependencies (what other processes rely on the outcome)

Good:
```
Updates the inactive flag on the customer record. This flag is used by
StaleAccountCleanupBatch for monthly processing. Triggered by the
DateOfDeathChanged workflow rule.
```

Bad:
```
Updates the inactive flag.
```

---

## Governance Checklist

Before deploying any automation:

- [ ] Naming follows the convention for its type
- [ ] Description is complete with purpose, owner, and dependencies
- [ ] No hardcoded IDs or record type developer names in formulas
- [ ] VR bypass mechanism is in place
- [ ] Flow has fault paths on all DML elements
- [ ] No duplicate automation (check for existing Flows on same object and timing)
- [ ] Tested in sandbox with bulk data (at least 200 records for Flows)
- [ ] Error messages include VR code and are user-friendly
- [ ] Approval steps have self-documenting names
