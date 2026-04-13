# Automation Scanner

Detailed instructions for scanning Flows, validation rules, and approval processes.

## Discovery — Flows

1. Glob for `**/flows/*.flow-meta.xml` across all package directories.
2. Read each Flow XML and extract:

| Field | XML Element | Notes |
|-------|------------|-------|
| API name | Filename (without `.flow-meta.xml`) | `Account_RT_BeforeSave_SetDefaults` |
| Label | `<label>` | Human-readable name |
| Description | `<description>` | Purpose and context |
| Process type | `<processType>` | See classification table below |
| Status | `<status>` | Active, Draft, Obsolete |
| API version | `<apiVersion>` | 62.0 |
| Start object | `<start><object>` | For record-triggered Flows |
| Trigger type | `<start><triggerType>` | RecordBeforeSave, RecordAfterSave, Scheduled, etc. |
| Record trigger | `<start><recordTriggerType>` | Create, CreateAndUpdate, Update, Delete |
| Scheduled paths | `<start><scheduledPaths>` | For Flows with scheduled actions |
| Environments | `<environments>` | Default (blank) or specific environments |
| Run mode | `<runInMode>` | DefaultMode, SystemModeWithoutSharing, etc. |

### Flow Classification

| `<processType>` Value | Category | Icon |
|-----------------------|----------|------|
| `AutoLaunchedFlow` with `<start><object>` + `<start><triggerType>` | Record-Triggered Flow | RT |
| `AutoLaunchedFlow` without trigger | Autolaunched Flow | AL |
| `Flow` | Screen Flow | SCR |
| `CustomEvent` | Platform Event-Triggered Flow | PE |
| `Workflow` | Scheduled Flow (legacy) | SCH |
| `InvocableProcess` | Subflow / Invocable | SFL |

### Flow Complexity Assessment

Count these elements to assess complexity:

| Element | XML Tag | Concern if > |
|---------|---------|-------------|
| Decisions | `<decisions>` | 10 |
| Loops | `<loops>` | 3 |
| Record creates | `<recordCreates>` | 5 |
| Record updates | `<recordUpdates>` | 5 |
| Record lookups | `<recordLookups>` | 5 |
| Subflow calls | `<subflows>` | 3 |
| Assignments | `<assignments>` | 15 |
| Screens | `<screens>` | 8 |
| Total elements | All of the above | 30 |

If total elements > 30, flag as "High complexity — consider refactoring into subflows."

### Scheduled Paths

For Flows with `<scheduledPaths>`, extract:
- Path name
- Offset (number and unit)
- Time source field

---

## Discovery — Validation Rules

1. Glob for `**/objects/*/validationRules/*.validationRule-meta.xml`.
2. For each validation rule XML, extract:

| Field | XML Element |
|-------|------------|
| API name | `<fullName>` |
| Active | `<active>` |
| Description | `<description>` |
| Error condition formula | `<errorConditionFormula>` |
| Error display field | `<errorDisplayField>` |
| Error message | `<errorMessage>` |

### Per-Object Grouping

Group validation rules by their parent object (the object folder they live in).

---

## Discovery — Approval Processes

1. Glob for `**/approvalProcesses/*.approvalProcess-meta.xml`.
2. For each approval process XML, extract:

| Field | XML Element |
|-------|------------|
| Name | `<fullName>` |
| Label | `<label>` |
| Description | `<description>` |
| Active | `<active>` |
| Entry criteria | `<entryCriteria>` |
| Record editability | `<recordEditability>` |
| Object | Parsed from fullName (format: `Object.ProcessName`) |
| Approval steps | `<approvalStep>` |
| Final approval actions | `<finalApprovalActions>` |
| Final rejection actions | `<finalRejectionActions>` |

### Approval Steps

For each `<approvalStep>`:
- Step name and label
- Assigned approver type (user, queue, manager, related user)
- Step criteria

---

## Output Format for automation-map.md

### Summary

```markdown
## Summary

| Type | Active | Draft/Inactive | Total |
|------|--------|---------------|-------|
| Record-Triggered Flows | 8 | 2 | 10 |
| Screen Flows | 3 | 0 | 3 |
| Autolaunched Flows | 2 | 1 | 3 |
| Scheduled Flows | 1 | 0 | 1 |
| Subflows | 2 | 0 | 2 |
| Validation Rules | 15 | 3 | 18 |
| Approval Processes | 1 | 0 | 1 |
```

### Flows by Object

Group Flows by their trigger object:

```markdown
## Flows by Object

### Account

| Flow | Type | Timing | Trigger | Status | API Ver | Complexity |
|------|------|--------|---------|--------|---------|-----------|
| `Account_RT_BeforeSave_SetDefaults` | RT | Before Save | Create, Update | Active | 62.0 | Low (8 elements) |
| `Account_RT_AfterSave_NotifyOwner` | RT | After Save | Create | Active | 62.0 | Medium (18 elements) |

### (No Object — Standalone)

| Flow | Type | Status | Description |
|------|------|--------|-------------|
| `Case_SCR_EscalationWizard` | Screen | Active | Guided wizard for case escalation |
```

### Validation Rules by Object

```markdown
## Validation Rules

### Opportunity ({count} rules)

| Rule | Active | Error Message | Field |
|------|--------|---------------|-------|
| `VR01_Opp_CloseReasonRequired` | Yes | "Close reason is required when stage is Closed Lost" | `CloseReason__c` |
| `VR02_Opp_AmountPositive` | Yes | "Amount must be greater than zero" | `Amount` |

### Account ({count} rules)

| Rule | Active | Error Message | Field |
|------|--------|---------------|-------|
| `VR01_Account_IndustryRequired` | No | "Industry is required for Enterprise accounts" | `Industry` |
```

### Approval Processes

```markdown
## Approval Processes

### OpportunityDiscountApproval

**Object:** Opportunity
**Status:** Active
**Entry Criteria:** Discount > 20%
**Record Editability:** Locked during approval

| Step | Approver | Criteria |
|------|----------|----------|
| Manager Approval | Hierarchy: Manager | All records |
| Director Approval | User: Finance Director | Discount > 40% |

**On Approval:** Sets `IsApproved__c` = true, sends email to owner
**On Rejection:** Sets `IsApproved__c` = false, sends notification
```

### Observed Naming Patterns

Detect the actual naming patterns the project uses for Flows. Look for common structures:
- Do Flow names start with the object name?
- Do they include a type indicator (RT, SCR, AL, SCH, SFL)?
- Do they include timing (BeforeSave, AfterSave)?
- Is there a consistent separator (underscore, camelCase)?

Document the **detected pattern** and any outliers that break the project's own convention:

```markdown
## Flow Naming Patterns

**Detected pattern:** `{Object}_{Type}_{Timing}_{Action}` (used by 8 of 10 Flows)

| Flow | Follows Project Pattern |
|------|:-----------------------:|
| `Account_RT_BeforeSave_SetDefaults` | Yes |
| `SetAccountDefaults` | No — no object prefix or type indicator |
```
