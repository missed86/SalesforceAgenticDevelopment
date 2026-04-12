# Async Patterns

Background processing in Apex. Choose the right tool for the job.

## Decision Table

| Need | Pattern | Limits |
|------|---------|--------|
| Simple callout from trigger | `@future(callout=true)` | 50 calls/txn, primitives only |
| Chain jobs, pass complex data | `Queueable` | 50 queued/txn, 250K async/day |
| Process millions of records | `Batch` | 5 concurrent, configurable chunk |
| Run on schedule | `Schedulable` | 100 scheduled jobs |
| Decouple systems | Platform Events | 150 publishes/txn |

## @future Method

```apex
public with sharing class ExternalNotificationService {

    @future(callout=true)
    public static void notifyExternalSystem(Set<Id> accountIds) {
        List<Account> accounts = AccountSelector.selectByIds(accountIds);

        HttpRequest req = new HttpRequest();
        req.setEndpoint('callout:ExternalSystem/api/accounts');
        req.setMethod('POST');
        req.setHeader('Content-Type', 'application/json');
        req.setBody(JSON.serialize(accounts));

        Http http = new Http();
        HttpResponse res = http.send(req);

        if (res.getStatusCode() != 200) {
            Logger.error('ExternalNotificationService', 'notifyExternalSystem',
                'Notification failed (HTTP ' + res.getStatusCode() + '): ' + res.getBody(), null);
        }
    }
}
```

## Queueable (Preferred Over @future)

```apex
public class AccountEnrichmentJob implements Queueable, Database.AllowsCallouts {

    private final Set<Id> accountIds;

    public AccountEnrichmentJob(Set<Id> accountIds) {
        this.accountIds = accountIds;
    }

    public void execute(QueueableContext ctx) {
        List<Account> accounts = AccountSelector.selectByIds(accountIds);
        List<Account> toUpdate = new List<Account>();

        for (Account acc : accounts) {
            String enrichedData = callEnrichmentApi(acc.Name);
            if (enrichedData != null) {
                toUpdate.add(new Account(
                    Id = acc.Id,
                    Description = enrichedData
                ));
            }
        }

        if (!toUpdate.isEmpty()) {
            update toUpdate;
        }
    }

    private String callEnrichmentApi(String companyName) {
        HttpRequest req = new HttpRequest();
        req.setEndpoint('callout:EnrichmentAPI/lookup');
        req.setMethod('GET');
        req.setHeader('Content-Type', 'application/json');

        Http http = new Http();
        HttpResponse res = http.send(req);

        return res.getStatusCode() == 200 ? res.getBody() : null;
    }
}

// Enqueue from handler or service:
// System.enqueueJob(new AccountEnrichmentJob(accountIds));
```

## Queueable With Chaining

```apex
public class DataSyncJob implements Queueable {

    private final List<String> objectsToSync;
    private final Integer currentIndex;

    public DataSyncJob(List<String> objectsToSync) {
        this(objectsToSync, 0);
    }

    private DataSyncJob(List<String> objectsToSync, Integer currentIndex) {
        this.objectsToSync = objectsToSync;
        this.currentIndex = currentIndex;
    }

    public void execute(QueueableContext ctx) {
        if (currentIndex >= objectsToSync.size()) return;

        String objectName = objectsToSync[currentIndex];
        syncObject(objectName);

        Integer nextIndex = currentIndex + 1;
        if (nextIndex < objectsToSync.size()) {
            System.enqueueJob(new DataSyncJob(objectsToSync, nextIndex));
        }
    }

    private void syncObject(String objectName) {
        // sync logic per object type
    }
}
```

## Batch Apex

```apex
public class StaleAccountCleanupBatch implements
    Database.Batchable<SObject>, Database.Stateful
{
    private Integer totalProcessed = 0;
    private Integer totalErrors = 0;

    public Database.QueryLocator start(Database.BatchableContext bc) {
        Date cutoff = Date.today().addMonths(-12);
        return Database.getQueryLocator([
            SELECT Id, Name, LastActivityDate
            FROM Account
            WHERE LastActivityDate < :cutoff
        ]);
    }

    public void execute(Database.BatchableContext bc, List<Account> scope) {
        List<Account> toUpdate = new List<Account>();

        for (Account acc : scope) {
            toUpdate.add(new Account(
                Id = acc.Id,
                Status__c = 'Stale'
            ));
        }

        List<Database.SaveResult> results = Database.update(toUpdate, false);
        for (Database.SaveResult sr : results) {
            if (sr.isSuccess()) {
                totalProcessed++;
            } else {
                totalErrors++;
            }
        }
    }

    public void finish(Database.BatchableContext bc) {
        Logger.info('StaleAccountCleanupBatch', 'finish',
            'Batch complete. Processed: ' + totalProcessed + ' Errors: ' + totalErrors);
    }
}
```

## Schedulable (Orchestrator)

```apex
public class WeeklyCleanupScheduler implements Schedulable {

    public void execute(SchedulableContext ctx) {
        Database.executeBatch(new StaleAccountCleanupBatch(), 200);
    }
}

// Schedule via Anonymous Apex:
// String cronExp = '0 0 2 ? * SAT';
// System.schedule('Weekly Stale Account Cleanup', cronExp, new WeeklyCleanupScheduler());
```

## Platform Events for Decoupling

```apex
// Publisher (in service or trigger handler)
public static void publishOrderEvent(Set<Id> orderIds) {
    List<OrderEvent__e> events = new List<OrderEvent__e>();
    for (Id orderId : orderIds) {
        events.add(new OrderEvent__e(
            OrderId__c = orderId,
            Action__c = 'PROCESSED'
        ));
    }
    List<Database.SaveResult> results = EventBus.publish(events);
}

// Subscriber (Apex trigger on platform event)
trigger OrderEventTrigger on OrderEvent__e (after insert) {
    OrderEventHandler handler = new OrderEventHandler();
    handler.afterInsert(Trigger.new);
}
```

## Queueable Finalizer (Post-Execution Handling)

Finalizers run after a Queueable job completes, even on unhandled exceptions:

```apex
public class AccountEnrichmentFinalizer implements Finalizer {

    public void execute(FinalizerContext ctx) {
        if (ctx.getResult() == ParentJobResult.SUCCESS) {
            Logger.info('AccountEnrichmentFinalizer', 'execute', 'Enrichment completed');
        } else {
            Logger.error('AccountEnrichmentFinalizer', 'execute',
                'Enrichment failed: ' + ctx.getException().getMessage(), null);
        }
    }
}
```

Attach the finalizer inside the Queueable:

```apex
public void execute(QueueableContext ctx) {
    System.attachFinalizer(new AccountEnrichmentFinalizer());
    // job logic here
}
```

Use finalizers for: logging, alerting, cleanup, or chaining recovery jobs on failure.

## Batch Size Guidelines

| Logic Complexity | Batch Size | Reason |
|-----------------|------------|--------|
| Light (field updates) | 200 | Max throughput |
| Medium (calculations, related queries) | 50-100 | Balance speed/limits |
| Heavy (callouts, complex logic) | 10-25 | Stay under CPU/callout limits |

## Key Rules

- **Never** enqueue/future inside a loop — collect, then enqueue once
- Queueable > @future for new code (more flexible, monitorable)
- Use `Database.Stateful` in Batch when you need cross-chunk state
- Use `Database.AllowsCallouts` interface when making HTTP calls from async
- Use `allOrNone=false` (`Database.update(records, false)`) in Batch for partial success
- Schedule Batch jobs via Schedulable — never call `Database.executeBatch` from triggers
