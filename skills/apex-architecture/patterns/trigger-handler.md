# Trigger + Handler Pattern

One trigger per object. Zero business logic in the trigger. All logic in the handler.

## Trigger (Delegator Only)

```apex
trigger AccountTrigger on Account (
    before insert, before update, before delete,
    after insert, after update, after delete, after undelete
) {
    if (TriggerControl.isDisabled('AccountTrigger')) return;

    AccountTriggerHandler handler = new AccountTriggerHandler();

    switch on Trigger.operationType {
        when BEFORE_INSERT  { handler.beforeInsert(Trigger.new); }
        when BEFORE_UPDATE  { handler.beforeUpdate(Trigger.new, Trigger.oldMap); }
        when BEFORE_DELETE  { handler.beforeDelete(Trigger.old, Trigger.oldMap); }
        when AFTER_INSERT   { handler.afterInsert(Trigger.new, Trigger.newMap); }
        when AFTER_UPDATE   { handler.afterUpdate(Trigger.new, Trigger.oldMap); }
        when AFTER_DELETE   { handler.afterDelete(Trigger.old, Trigger.oldMap); }
        when AFTER_UNDELETE { handler.afterUndelete(Trigger.new); }
    }
}
```

Uses `Trigger.operationType` enum (cleaner than nested `isBefore`/`isInsert` checks).

## Trigger Bypass / Disable Pattern

Disable triggers during data loads, migrations, or test setup:

```apex
public with sharing class TriggerControl {

    private static Set<String> disabledTriggers = new Set<String>();

    public static Boolean isDisabled(String triggerName) {
        if (disabledTriggers.contains('ALL') || disabledTriggers.contains(triggerName)) {
            return true;
        }

        TriggerSetting__mdt setting = TriggerSetting__mdt.getInstance(triggerName);
        return setting != null && !setting.IsActive__c;
    }

    public static void disable(String triggerName) {
        disabledTriggers.add(triggerName);
    }

    public static void enable(String triggerName) {
        disabledTriggers.remove(triggerName);
    }

    public static void disableAll() {
        disabledTriggers.add('ALL');
    }

    public static Boolean isDisabledAll() {
        return disabledTriggers.contains('ALL');
    }
}
```

Two bypass mechanisms:
- **CMDT-based** (`TriggerSetting__mdt`): admin-controlled, persistent, deployable
- **Static runtime** (`disable()`/`enable()`): code-controlled, transaction-scoped

Usage in data loads:

```apex
TriggerControl.disable('AccountTrigger');
insert largeAccountList;
TriggerControl.enable('AccountTrigger');
```

## Handler Class

```apex
public with sharing class AccountTriggerHandler {

    // ── Recursion Guard ──
    private static Set<Id> processedIds = new Set<Id>();

    // ── Before ──

    public void beforeInsert(List<Account> newRecords) {
        AccountDomain.setDefaults(newRecords);
        AccountDomain.validateRequiredFields(newRecords);
    }

    public void beforeUpdate(List<Account> newRecords, Map<Id, Account> oldMap) {
        List<Account> changed = filterChanged(newRecords, oldMap);
        if (changed.isEmpty()) return;

        AccountDomain.validateStateTransitions(changed, oldMap);
    }

    public void beforeDelete(List<Account> oldRecords, Map<Id, Account> oldMap) {
        AccountDomain.preventDeletionWithActiveContracts(oldRecords);
    }

    // ── After ──

    public void afterInsert(List<Account> newRecords, Map<Id, Account> newMap) {
        Set<Id> unprocessed = excludeProcessed(newMap.keySet());
        if (unprocessed.isEmpty()) return;

        markProcessed(unprocessed);
        AccountService.createDefaultContacts(unprocessed);
    }

    public void afterUpdate(List<Account> newRecords, Map<Id, Account> oldMap) {
        Set<Id> unprocessed = excludeProcessed(new Map<Id, Account>(newRecords).keySet());
        if (unprocessed.isEmpty()) return;

        markProcessed(unprocessed);
        List<Account> ownerChanged = filterOwnerChanged(newRecords, oldMap);
        if (!ownerChanged.isEmpty()) {
            AccountService.cascadeOwnerToChildren(
                new Map<Id, Account>(ownerChanged).keySet()
            );
        }
    }

    public void afterDelete(List<Account> oldRecords, Map<Id, Account> oldMap) {
        AccountService.cleanupOrphanedData(oldMap.keySet());
    }

    public void afterUndelete(List<Account> newRecords) {
        AccountService.restoreRelatedRecords(
            new Map<Id, Account>(newRecords).keySet()
        );
    }

    // ── Private Helpers ──

    private List<Account> filterChanged(
        List<Account> newRecords, Map<Id, Account> oldMap
    ) {
        List<Account> changed = new List<Account>();
        for (Account acc : newRecords) {
            Account old = oldMap.get(acc.Id);
            if (acc.Name != old.Name || acc.Industry != old.Industry) {
                changed.add(acc);
            }
        }
        return changed;
    }

    private List<Account> filterOwnerChanged(
        List<Account> newRecords, Map<Id, Account> oldMap
    ) {
        List<Account> result = new List<Account>();
        for (Account acc : newRecords) {
            if (acc.OwnerId != oldMap.get(acc.Id).OwnerId) {
                result.add(acc);
            }
        }
        return result;
    }

    private Set<Id> excludeProcessed(Set<Id> ids) {
        Set<Id> unprocessed = new Set<Id>(ids);
        unprocessed.removeAll(processedIds);
        return unprocessed;
    }

    private void markProcessed(Set<Id> ids) {
        processedIds.addAll(ids);
    }
}
```

## Recursion Guard Strategies

### Strategy 1: Static Set of IDs (Recommended)

```apex
private static Set<Id> processedIds = new Set<Id>();

public void afterUpdate(List<Account> newRecords, Map<Id, Account> oldMap) {
    Set<Id> toProcess = new Set<Id>();
    for (Account acc : newRecords) {
        if (!processedIds.contains(acc.Id)) {
            toProcess.add(acc.Id);
        }
    }
    if (toProcess.isEmpty()) return;
    processedIds.addAll(toProcess);
    // proceed with logic
}
```

### Strategy 2: Static Boolean (Simple cases only)

```apex
private static Boolean isRunning = false;

public void afterUpdate(List<Account> newRecords, Map<Id, Account> oldMap) {
    if (isRunning) return;
    isRunning = true;
    try {
        // logic here
    } finally {
        isRunning = false;
    }
}
```

Use Set<Id> for granular control. Use Boolean only for trivial single-pass scenarios.

## Before vs After: When to Use

| Context | Use When |
|---------|----------|
| **Before Insert** | Set defaults, validate fields, transform data |
| **Before Update** | Validate state transitions, normalize fields |
| **Before Delete** | Block deletion based on business rules |
| **After Insert** | Create related records (you need IDs), callouts |
| **After Update** | Cascade changes to children, async operations |
| **After Delete** | Cleanup orphaned data, audit logging |
| **After Undelete** | Restore relationships |

## Key Rules

- Trigger body: **only** routing to handler methods
- Handler: delegates to Domain (validations) and Service (business logic)
- Bulkify everything — the handler receives `List<SObject>`, never single records
- Filter early — check for actual changes before executing expensive logic
- Guard recursion at the handler level, not in the service/domain
