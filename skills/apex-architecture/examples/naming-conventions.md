# Naming Conventions — Complete Reference

Consistent naming across the entire Salesforce project.

## Apex Classes

| Type | Convention | Pattern | Example |
|------|-----------|---------|---------|
| Trigger | PascalCase | `{SObject}Trigger` | `AccountTrigger` |
| Trigger Handler | PascalCase | `{SObject}TriggerHandler` | `AccountTriggerHandler` |
| Service | PascalCase | `{Domain}Service` | `OrderService` |
| Selector | PascalCase | `{SObject}Selector` | `OpportunitySelector` |
| Domain | PascalCase | `{SObject}Domain` | `CaseDomain` |
| Controller (LWC) | PascalCase | `{Feature}Controller` | `InvoiceController` |
| Batch | PascalCase | `{Purpose}Batch` | `StaleAccountCleanupBatch` |
| Queueable | PascalCase | `{Purpose}Job` | `AccountEnrichmentJob` |
| Schedulable | PascalCase | `{Purpose}Scheduler` | `WeeklyCleanupScheduler` |
| Test Class | PascalCase | `{ClassUnderTest}Test` | `OrderServiceTest` |
| Test Factory | PascalCase | `TestDataFactory` | `TestDataFactory` |
| Response DTO | PascalCase | `{Entity}Dto` | `AccountDto` |
| Request DTO | PascalCase | `{Action}Request` | `CreateOrderRequest` |
| Service Response | PascalCase | `ServiceResponseDto` | `ServiceResponseDto` |
| Operation Result | PascalCase | `{Action}ResultDto` | `PaymentResultDto` |
| Page Result | PascalCase | `PageResultDto` | `PageResultDto` |
| Lookup Option | PascalCase | `OptionDto` | `OptionDto` |
| Custom Exception | PascalCase | `{Domain}Exception` | `PaymentException` |
| Interface | PascalCase, I-prefix | `I{Capability}` | `IPaymentGateway` |
| REST Resource | PascalCase | `{Resource}RestService` | `OrderRestService` |
| Integration Service | PascalCase | `{System}Service` | `PaymentGatewayService` |
| Logger | PascalCase | `Logger` | `Logger` |
| Trigger Control | PascalCase | `TriggerControl` | `TriggerControl` |
| Utility | PascalCase | `{Domain}Utils` | `DateUtils` |
| Constants | PascalCase | `{Domain}Constants` | `AppConstants` |

## Methods

| Convention | Pattern | Example |
|-----------|---------|---------|
| camelCase, verb-first | `{verb}{Noun}` | `calculateTotalAmount()` |
| Boolean getters | `is{Condition}` / `has{Thing}` | `isActive()`, `hasPermission()` |
| Selectors | `select{Criteria}` | `selectByOwnerId()` |
| Bulk selectors | `select{Criteria}` (returns List) | `selectWithContacts()` |
| Map selectors | `selectMap{Criteria}` | `selectMapByIds()` |
| Grouped selectors | `selectGroupedBy{Field}` | `selectGroupedByAccountId()` |
| Domain validators | `validate{Rule}` | `validateRequiredFields()` |
| Domain defaults | `setDefaults` / `apply{Rule}` | `setDefaults()`, `applyDiscountRules()` |
| Service operations | `{verb}{BusinessAction}` | `processOrders()`, `cascadeOwnerToChildren()` |
| Factory methods | `create{Thing}` | `createAccount()`, `createContacts()` |
| Test methods | `method_scenario_expected` | `calculateDiscount_over10k_shouldApply10Percent()` |

## Variables and Parameters

| Type | Convention | Example |
|------|-----------|---------|
| Local variable | camelCase, descriptive | `accountsByOwnerId` |
| Loop variable | camelCase, meaningful | `for (Account acc : accounts)` |
| Loop index (only) | Single letter | `for (Integer i = 0; ...)` |
| Parameter | camelCase | `processPayment(Id orderId, Decimal amount)` |
| Set of IDs | camelCase | `Set<Id> accountIds` |
| Map by ID | camelCase + By{Key} | `Map<Id, Account> accountsById` |
| Map grouped | camelCase + By{Key} | `Map<Id, List<Contact>> contactsByAccountId` |
| Boolean | `is`/`has`/`should` prefix | `isProcessed`, `hasErrors`, `shouldRetry` |
| Constant | UPPER_SNAKE_CASE | `MAX_RETRY_COUNT`, `DEFAULT_BATCH_SIZE` |

## SObject Fields (Custom)

| Type | Convention | Example |
|------|-----------|---------|
| Custom field | PascalCase + `__c` | `PaymentStatus__c` |
| Lookup relationship | PascalCase + `__c` | `PrimaryContact__c` |
| Master-Detail | PascalCase + `__c` | `ParentAccount__c` |
| Formula field | PascalCase + `__c` | `TotalWithTax__c` |
| Rollup summary | PascalCase + `__c` | `ContactCount__c` |
| Custom object | PascalCase + `__c` | `Invoice__c` |
| Custom metadata | PascalCase + `__mdt` | `DiscountRule__mdt` |
| Platform event | PascalCase + `__e` | `OrderEvent__e` |

## Custom Labels and Custom Settings

| Type | Convention | Example |
|------|-----------|---------|
| Custom Label | PascalCase, descriptive | `ErrorOrderNotFound` |
| Custom Setting (hierarchy) | PascalCase + `__c` | `AppSettings__c` |
| Custom Metadata Type | PascalCase + `__mdt` | `FeatureToggle__mdt` |

## LWC Component Names

| Type | Convention | Example |
|------|-----------|---------|
| Component folder | camelCase | `orderSummary` |
| HTML file | camelCase | `orderSummary.html` |
| JS file | camelCase | `orderSummary.js` |
| CSS file | camelCase | `orderSummary.css` |

## Code Comments

### When to Comment

| Situation | Comment? | Example |
|-----------|----------|---------|
| Non-obvious business rule | Yes | `// Closed Won opps can't reopen per sales policy` |
| Platform gotcha / limitation | Yes | `// LastActivityDate is read-only, can't set via DML` |
| Why a workaround exists | Yes | `// Savepoint here because partial DML leaves orphans` |
| `without sharing` justification | Always | `// System mode: batch must see all records regardless of user` |
| Public API / interface method | Yes (brief) | `// Processes pending orders and returns success count` |
| Section dividers in long classes | Yes | `// ── Before ──` or `// ── Private Helpers ──` |
| TODO / known debt | Yes, with ticket | `// TODO(JIRA-123): replace with Queueable when limit increases` |
| Constants context | Only if non-obvious | `// Salesforce max SOQL rows per transaction` |

### When NOT to Comment

| Anti-Pattern | Why It's Bad |
|-------------|-------------|
| `// Query accounts` before `AccountSelector.selectByIds()` | Code already says it |
| `// Loop through records` before a for loop | Obvious from syntax |
| `// Set the name` before `acc.Name = value` | Adds noise, zero value |
| `// Check if null` before a null check | Reader can see this |
| `// Return the result` before a return | Pure narration |
| Commented-out code blocks | Use version control, don't leave dead code |
| Change log in comments (`// Modified by X on date`) | That's what git history is for |

### Comment Style

```apex
// Single-line for brief context
if (ord.TotalAmount > acc.CreditLimit__c) {

// Multi-line for complex reasoning:
// Orders over credit limit require VP approval per finance policy.
// The approval process is triggered async to avoid blocking the user.
ApprovalService.submitForReview(ord.Id);

// ── Section Dividers ── (use in classes with 5+ methods)

// ── Before ──
public void beforeInsert(List<Account> records) { }

// ── After ──
public void afterInsert(List<Account> records) { }
```

### ApexDoc for Public APIs

Use ApexDoc (`/** */`) only on public/global methods meant as APIs for other teams or packages:

```apex
/**
 * Calculates discount based on CMDT rules.
 * @param orderAmount Gross order amount before discount
 * @return Discount amount (0 if no rule matches)
 */
public static Decimal calculateDiscount(Decimal orderAmount) {
    // ...
}
```

Skip ApexDoc on internal methods where the name + signature are self-documenting.

## General Rules

- **No abbreviations** unless universally understood (`Id`, `URL`, `API`)
- **No single/double character** names except loop indices
- **Pluralize collections**: `accounts` (List), `accountIds` (Set), `accountsById` (Map)
- **Be specific**: `accountName` not `name`, `startDate` not `date`
- **Constants in a dedicated class**: `AppConstants.MAX_RETRY_COUNT`
- **Avoid Hungarian notation**: no `strName`, `lstAccounts`, `mapAccById`
