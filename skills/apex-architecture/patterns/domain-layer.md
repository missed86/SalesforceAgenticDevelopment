# Domain Layer Pattern

SObject-specific validations, defaults, and business rules. One Domain class per SObject.

## Core Principles

- **Static methods** operating on `List<SObject>` — always bulk-safe
- **Pure logic** — no SOQL, no DML, no callouts
- **Called by** Handlers (before triggers) and Services
- **Throws custom exceptions** when validation fails

## Basic Domain Class

```apex
public with sharing class OpportunityDomain {

    public static void setDefaults(List<Opportunity> records) {
        for (Opportunity opp : records) {
            if (opp.CloseDate == null) {
                opp.CloseDate = Date.today().addDays(30);
            }
            if (String.isBlank(opp.StageName)) {
                opp.StageName = 'Prospecting';
            }
            if (opp.Probability == null) {
                opp.Probability = 10;
            }
        }
    }

    public static void validateRequiredFields(List<Opportunity> records) {
        for (Opportunity opp : records) {
            if (opp.Amount == null || opp.Amount <= 0) {
                opp.Amount.addError('Amount must be greater than zero.');
            }
            if (opp.CloseDate != null && opp.CloseDate < Date.today()) {
                opp.CloseDate.addError('Close date cannot be in the past.');
            }
        }
    }

    public static void validateStateTransitions(
        List<Opportunity> newRecords, Map<Id, Opportunity> oldMap
    ) {
        for (Opportunity opp : newRecords) {
            Opportunity old = oldMap.get(opp.Id);
            if (old.StageName == 'Closed Won' && opp.StageName != 'Closed Won') {
                opp.StageName.addError('Cannot reopen a Closed Won opportunity.');
            }
            if (old.StageName == 'Closed Lost' && opp.StageName == 'Closed Won') {
                opp.StageName.addError('Cannot move from Closed Lost to Closed Won.');
            }
        }
    }
}
```

## Domain With Complex Validation

```apex
public with sharing class OrderDomain {

    public class ValidationException extends Exception {}

    public static void validateForProcessing(Order ord) {
        List<String> errors = new List<String>();

        if (ord.Status == 'Cancelled') {
            errors.add('Cannot process a cancelled order.');
        }
        if (ord.TotalAmount == null || ord.TotalAmount <= 0) {
            errors.add('Order total must be positive.');
        }
        if (ord.OrderItems == null || ord.OrderItems.isEmpty()) {
            errors.add('Order must have at least one line item.');
        }

        if (!errors.isEmpty()) {
            throw new ValidationException(String.join(errors, ' | '));
        }
    }

    public static void applyDiscountRules(List<Order> orders) {
        for (Order ord : orders) {
            if (ord.TotalAmount > 10000) {
                ord.Discount__c = 0.10;
            } else if (ord.TotalAmount > 5000) {
                ord.Discount__c = 0.05;
            } else {
                ord.Discount__c = 0;
            }
        }
    }
}
```

## Domain Called From Handler (Before Trigger)

```apex
// In OpportunityTriggerHandler
public void beforeInsert(List<Opportunity> newRecords) {
    OpportunityDomain.setDefaults(newRecords);
    OpportunityDomain.validateRequiredFields(newRecords);
}

public void beforeUpdate(List<Opportunity> newRecords, Map<Id, Opportunity> oldMap) {
    OpportunityDomain.validateStateTransitions(newRecords, oldMap);
}
```

## Domain Called From Service (Validation Before DML)

```apex
// In OrderService
public static OrderResultDto processOrders(Set<Id> orderIds) {
    List<Order> orders = OrderSelector.selectWithLineItems(orderIds);

    for (Order ord : orders) {
        OrderDomain.validateForProcessing(ord);  // throws on failure
    }

    // proceed with processing...
}
```

## addError() vs Custom Exceptions

| Technique | Context | Behavior |
|-----------|---------|----------|
| `field.addError('msg')` | Before trigger | Prevents DML, shows error on field in UI |
| `record.addError('msg')` | Before trigger | Prevents DML, shows error at record level |
| `throw new CustomException()` | Service/Domain | Bubbles up to caller, caught in try/catch |

Use `addError()` in before-trigger domain methods. Use exceptions in service-layer validations.

## Domain That Needs External Context

When validation depends on data from other objects, the **caller** (Service or Handler)
pre-queries the context and passes it in — the Domain never queries:

```apex
// Domain method receives pre-queried context
public static void validateCreditLimit(
    List<Order> orders,
    Map<Id, Account> accountsById
) {
    for (Order ord : orders) {
        Account acc = accountsById.get(ord.AccountId);
        if (acc != null && ord.TotalAmount > acc.CreditLimit__c) {
            ord.addError('Order exceeds account credit limit of ' + acc.CreditLimit__c);
        }
    }
}
```

```apex
// Service or Handler pre-queries and passes context
public void beforeInsert(List<Order> newOrders) {
    Set<Id> accountIds = new Set<Id>();
    for (Order ord : newOrders) {
        accountIds.add(ord.AccountId);
    }

    Map<Id, Account> accountsById = AccountSelector.selectMapByIds(accountIds);
    OrderDomain.validateCreditLimit(newOrders, accountsById);
}
```

## Record-Type-Aware Domain Logic

When validation or defaults vary by record type, resolve the `RecordTypeId` via
`Schema` (never hardcode IDs) and branch inside the same bulk-safe method:

```apex
public with sharing class CaseDomain {

    private static final Map<String, Id> RT_IDS = new Map<String, Id>();

    private static Id rtId(String developerName) {
        if (!RT_IDS.containsKey(developerName)) {
            RT_IDS.put(developerName,
                Schema.SObjectType.Case
                    .getRecordTypeInfosByDeveloperName()
                    .get(developerName)
                    .getRecordTypeId()
            );
        }
        return RT_IDS.get(developerName);
    }

    public static void setDefaultsByRecordType(List<Case> records) {
        Id internalId = rtId('Internal');
        Id externalId = rtId('External');

        for (Case c : records) {
            if (c.RecordTypeId == internalId) {
                if (c.Priority == null) c.Priority = 'Low';
                if (c.Origin == null)   c.Origin = 'Internal';
            } else if (c.RecordTypeId == externalId) {
                if (c.Priority == null) c.Priority = 'Medium';
                if (c.Origin == null)   c.Origin = 'Web';
            }
        }
    }

    public static void validateByRecordType(List<Case> records) {
        Id externalId = rtId('External');

        for (Case c : records) {
            if (c.RecordTypeId == externalId && String.isBlank(c.ContactId)) {
                c.addError('External cases require a Contact.');
            }
        }
    }
}
```

### When to use record-type branching in Domain

| Scenario | Approach |
|----------|----------|
| Different defaults per record type | Branch in `setDefaults*` |
| Different validations per record type | Branch in `validate*` |
| Completely different business domains sharing an SObject | Consider separate Domain methods per RT, called by the Handler |
| Only 1 record type or no RT differences | Skip branching — keep it simple |

## Key Rules

- Domain classes have **no SOQL, no DML** — pure transformation and validation
- Accept `List<SObject>` — bulk-safe by design
- **Before trigger** context: use `addError()` to block individual records
- **Service** context: throw custom exceptions for validation failures
- Keep domain methods **small and focused** — one rule per method
- Domain is the **only place** where field-level business rules live
- When the Domain needs related data, the **caller pre-queries and passes it** as a parameter
- **Never hardcode Record Type IDs** — use `Schema.SObjectType.*.getRecordTypeInfosByDeveloperName()`
