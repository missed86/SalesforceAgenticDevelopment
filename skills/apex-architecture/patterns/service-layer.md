# Service Layer Pattern

Caller-agnostic business logic. Called by triggers, controllers, batch jobs, APIs, and queueables identically.

## Core Principles

- **Static methods by default** — simple, stateless, easy to call from anywhere
- **Instance methods when using DI** — for testable code with injected dependencies (see [configuration.md](configuration.md#lightweight-dependency-injection-no-framework))
- **Caller-agnostic** — same method works from trigger handler, LWC controller, batch, REST API
- **Accepts IDs or collections** — never single SObjects from trigger context
- **Owns DML** — the service decides when to commit; use Savepoints for multi-DML rollback
- **Uses Selectors** for queries, **Domains** for validations

## Basic Service

```apex
public with sharing class OpportunityService {

    public static void closeWonOpportunities(Set<Id> opportunityIds) {
        List<Opportunity> opps = OpportunitySelector.selectByIds(opportunityIds);
        List<Opportunity> toUpdate = new List<Opportunity>();

        for (Opportunity opp : opps) {
            if (opp.StageName != 'Closed Won') {
                toUpdate.add(new Opportunity(
                    Id = opp.Id,
                    StageName = 'Closed Won',
                    CloseDate = Date.today()
                ));
            }
        }

        if (!toUpdate.isEmpty()) {
            update toUpdate;
        }
    }

    public static void reassignToOwner(Set<Id> opportunityIds, Id newOwnerId) {
        List<Opportunity> opps = OpportunitySelector.selectByIds(opportunityIds);
        List<Opportunity> toUpdate = new List<Opportunity>();
        List<Task> tasksToUpdate = new List<Task>();

        for (Opportunity opp : opps) {
            toUpdate.add(new Opportunity(Id = opp.Id, OwnerId = newOwnerId));
        }

        List<Task> relatedTasks = TaskSelector.selectOpenByWhatIds(opportunityIds);
        for (Task t : relatedTasks) {
            tasksToUpdate.add(new Task(Id = t.Id, OwnerId = newOwnerId));
        }

        update toUpdate;
        if (!tasksToUpdate.isEmpty()) {
            update tasksToUpdate;
        }
    }
}
```

## Service With Error Aggregation

For operations that can partially fail, collect errors instead of throwing immediately:

```apex
public with sharing class OrderService {

    public static OrderResultDto processOrders(Set<Id> orderIds) {
        List<Order> orders = OrderSelector.selectWithLineItems(orderIds);
        List<Order> toUpdate = new List<Order>();
        List<String> errors = new List<String>();
        Integer successCount = 0;

        for (Order ord : orders) {
            try {
                OrderDomain.validateForProcessing(ord);
                toUpdate.add(new Order(
                    Id = ord.Id,
                    Status = 'Processing'
                ));
            } catch (OrderDomain.ValidationException e) {
                errors.add(ord.OrderNumber + ': ' + e.getMessage());
            }
        }

        if (!toUpdate.isEmpty()) {
            List<Database.SaveResult> results = Database.update(toUpdate, false);
            for (Integer i = 0; i < results.size(); i++) {
                if (results[i].isSuccess()) {
                    successCount++;
                } else {
                    errors.add(toUpdate[i].Id + ': ' + results[i].getErrors()[0].getMessage());
                }
            }
        }

        return new OrderResultDto(successCount, errors);
    }
}
```

## Service Called From Multiple Entry Points

```apex
// From Trigger Handler (owner already changed on the record)
public void afterUpdate(List<Account> newRecords, Map<Id, Account> oldMap) {
    Set<Id> ownerChanged = new Set<Id>();
    for (Account acc : newRecords) {
        if (acc.OwnerId != oldMap.get(acc.Id).OwnerId) {
            ownerChanged.add(acc.Id);
        }
    }
    if (!ownerChanged.isEmpty()) {
        AccountService.cascadeOwnerToChildren(ownerChanged);
    }
}

// From LWC Controller (explicit new owner from UI)
@AuraEnabled
public static ServiceResponseDto reassignAccounts(List<Id> accountIds, Id newOwnerId) {
    try {
        AccountService.reassignToOwner(new Set<Id>(accountIds), newOwnerId);
        return ServiceResponseDto.success('Accounts reassigned');
    } catch (AppException e) {
        throw new AuraHandledException(e.getMessage());
    }
}

// From Batch Apex
public void execute(Database.BatchableContext bc, List<Account> scope) {
    Set<Id> ids = new Map<Id, Account>(scope).keySet();
    AccountService.cascadeOwnerToChildren(ids);
}
```

## Transaction Control With Savepoints

When a service does multiple DML operations that must succeed or fail together:

```apex
public with sharing class TransferService {

    public static void transferOwnership(Set<Id> accountIds, Id newOwnerId) {
        Savepoint sp = Database.setSavepoint();

        try {
            List<Account> accounts = AccountSelector.selectByIds(accountIds);
            List<Contact> contacts = ContactSelector.selectByAccountIds(accountIds);
            List<Opportunity> opps = OpportunitySelector.selectOpenByAccountIds(accountIds);

            List<SObject> allUpdates = new List<SObject>();

            for (Account acc : accounts) {
                allUpdates.add(new Account(Id = acc.Id, OwnerId = newOwnerId));
            }
            for (Contact c : contacts) {
                allUpdates.add(new Contact(Id = c.Id, OwnerId = newOwnerId));
            }
            for (Opportunity opp : opps) {
                allUpdates.add(new Opportunity(Id = opp.Id, OwnerId = newOwnerId));
            }

            update allUpdates;

        } catch (Exception e) {
            Database.rollback(sp);
            throw new AppException('Transfer failed: ' + e.getMessage(), e);
        }
    }
}
```

Key points:
- `Database.setSavepoint()` before multi-DML operations
- `Database.rollback(sp)` on failure — undoes all DML since the savepoint
- Combine updates into a single `List<SObject>` when possible to reduce DML count
- Rethrow after rollback so the caller knows the operation failed

## Method Signature Guidelines

| Parameter Type | When |
|---------------|------|
| `Set<Id>` | Most common — lets service query fresh data |
| `List<SObject>` | When caller already has the data and you want to avoid re-query |
| `Dto` | Complex input with multiple fields from LWC/API |

Return types:

| Return Type | When |
|-------------|------|
| `void` | Simple operations, errors thrown as exceptions |
| `List<Database.SaveResult>` | Caller needs granular DML results |
| `CustomDto` | Structured response for LWC/API consumers |

## Key Rules

- One service per business domain (not per SObject)
- Services can call other services for cross-domain operations
- Services call Selectors — never inline SOQL
- Services call Domain methods for validation — never inline validation
- Keep services thin: orchestrate, don't implement low-level details
