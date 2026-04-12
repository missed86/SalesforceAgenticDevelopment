---
name: salesforce-foundation-init
description: >-
  Scaffolds the foundational Apex classes and metadata that every Salesforce DX
  project needs: Logger (Platform Event + custom object + subscriber trigger),
  TriggerControl (bypass with CMDT), AppException hierarchy, ServiceResponseDto,
  RecordTypes utility, and TestDataFactory skeleton. Use when starting a new
  Salesforce project, bootstrapping a scratch org, or when someone says
  "init foundation", "scaffold base classes", "crear clases base", or
  "inicializar proyecto Salesforce".
metadata:
  domain: salesforce-apex
  salesforce-era: spring-26
---

# Salesforce Foundation Init

Generate all the foundational plumbing that the **apex-architecture** skill assumes exists.
Run this once at the start of a new project — then build features on top.

## What gets created

| # | Component | Type | Purpose |
|---|-----------|------|---------|
| 1 | `TriggerControl` | Apex class | Enable/disable triggers at runtime + via CMDT |
| 2 | `TriggerSetting__mdt` | Custom Metadata Type | Admin-managed trigger on/off per object |
| 3 | `Logger` | Apex class | Publish structured log events (ERROR/WARN/INFO) |
| 4 | `LogEvent__e` | Platform Event | Async, non-blocking log transport |
| 5 | `ApplicationLog__c` | Custom Object | Persistent, queryable log storage |
| 6 | `LogEventSubscriber` | Apex trigger on `LogEvent__e` | Persists events to `ApplicationLog__c` |
| 7 | `AppException` | Apex class | Base exception for all custom exceptions |
| 8 | `ServiceResponseDto` | Apex class | Standard success/error wrapper for controllers |
| 9 | `RecordTypes` | Apex class | Resolve Record Type IDs safely via Schema describe |
| 10 | `TestDataFactory` | Apex class | Skeleton test data builder |

## Instructions for the agent

### Pre-flight checks

1. Confirm a **SFDX project** exists (`sfdx-project.json` in the repo root). If not, ask the user before proceeding.
2. Identify the **default package directory** from `sfdx-project.json` (usually `force-app/main/default`).
3. Check which of the 10 components **already exist** — skip any that do; never overwrite.

### File generation

Create every file below inside the default package directory. Each `.cls` needs its companion `.cls-meta.xml`, each `.trigger` needs `.trigger-meta.xml`, and metadata types need their XML definitions.

Use **API version 62.0** (Spring '26) in all `-meta.xml` files.

---

### 1. TriggerControl.cls

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

### 2. TriggerSetting__mdt

Custom Metadata Type with fields:
- `IsActive__c` — Checkbox, default `true`
- `Description__c` — Text(255)

Records are created per trigger (e.g. DeveloperName = `AccountTrigger`, `IsActive__c` = true).

### 3. Logger.cls

```apex
public without sharing class Logger {

    public static void error(String className, String methodName, Exception ex) {
        publish(new LogEvent__e(
            Level__c = 'ERROR',
            SourceClass__c = className,
            SourceMethod__c = methodName,
            Message__c = ex.getMessage(),
            StackTrace__c = ex.getStackTraceString(),
            UserId__c = UserInfo.getUserId()
        ));
    }

    public static void error(String className, String methodName, String message, Id recordId) {
        publish(new LogEvent__e(
            Level__c = 'ERROR',
            SourceClass__c = className,
            SourceMethod__c = methodName,
            Message__c = message,
            RecordId__c = recordId,
            UserId__c = UserInfo.getUserId()
        ));
    }

    public static void warn(String className, String methodName, String message) {
        publish(new LogEvent__e(
            Level__c = 'WARN',
            SourceClass__c = className,
            SourceMethod__c = methodName,
            Message__c = message,
            UserId__c = UserInfo.getUserId()
        ));
    }

    public static void info(String className, String methodName, String message) {
        publish(new LogEvent__e(
            Level__c = 'INFO',
            SourceClass__c = className,
            SourceMethod__c = methodName,
            Message__c = message,
            UserId__c = UserInfo.getUserId()
        ));
    }

    private static void publish(LogEvent__e event) {
        EventBus.publish(event);
    }
}
```

### 4. LogEvent__e (Platform Event)

Fields:
- `Level__c` — Text(10)
- `SourceClass__c` — Text(100)
- `SourceMethod__c` — Text(100)
- `Message__c` — Long Text Area(32000)
- `StackTrace__c` — Long Text Area(32000)
- `RecordId__c` — Text(18)
- `UserId__c` — Text(18)

### 5. ApplicationLog__c (Custom Object)

Fields (mirror `LogEvent__e`):
- `Level__c` — Text(10)
- `SourceClass__c` — Text(100)
- `SourceMethod__c` — Text(100)
- `Message__c` — Long Text Area(32000)
- `StackTrace__c` — Long Text Area(32000)
- `RecordId__c` — Text(18)
- `User__c` — Lookup(User)
- `LoggedAt__c` — DateTime

Set the object Name field to auto-number: `LOG-{00000}`.

### 6. LogEventSubscriber.trigger

```apex
trigger LogEventSubscriber on LogEvent__e (after insert) {
    List<ApplicationLog__c> logs = new List<ApplicationLog__c>();

    for (LogEvent__e event : Trigger.new) {
        logs.add(new ApplicationLog__c(
            Level__c = event.Level__c,
            SourceClass__c = event.SourceClass__c,
            SourceMethod__c = event.SourceMethod__c,
            Message__c = event.Message__c,
            StackTrace__c = event.StackTrace__c,
            RecordId__c = event.RecordId__c,
            User__c = event.UserId__c,
            LoggedAt__c = Datetime.now()
        ));
    }

    insert logs;
}
```

### 7. AppException.cls

```apex
public class AppException extends Exception {}
```

### 8. ServiceResponseDto.cls

```apex
public class ServiceResponseDto {

    @AuraEnabled public Boolean isSuccess;
    @AuraEnabled public String message;
    @AuraEnabled public Object data;

    private ServiceResponseDto(Boolean isSuccess, String message, Object data) {
        this.isSuccess = isSuccess;
        this.message = message;
        this.data = data;
    }

    public static ServiceResponseDto success(String message) {
        return new ServiceResponseDto(true, message, null);
    }

    public static ServiceResponseDto success(String message, Object data) {
        return new ServiceResponseDto(true, message, data);
    }

    public static ServiceResponseDto failure(String message) {
        return new ServiceResponseDto(false, message, null);
    }
}
```

### 9. RecordTypes.cls

```apex
public with sharing class RecordTypes {

    public static Id get(SObjectType sot, String developerName) {
        Schema.RecordTypeInfo info = sot.getDescribe()
            .getRecordTypeInfosByDeveloperName()
            .get(developerName);
        if (info == null) {
            throw new AppException(
                'Record Type not found: ' + sot + '.' + developerName
            );
        }
        return info.getRecordTypeId();
    }

    public static Map<String, Id> getAll(SObjectType sot) {
        Map<String, Id> result = new Map<String, Id>();
        for (Schema.RecordTypeInfo info : sot.getDescribe().getRecordTypeInfosByDeveloperName().values()) {
            if (info.isAvailable() && !info.isMaster()) {
                result.put(info.getDeveloperName(), info.getRecordTypeId());
            }
        }
        return result;
    }
}
```

### 10. TestDataFactory.cls

```apex
@isTest
public class TestDataFactory {

    public static Account createAccount(String name, Boolean doInsert) {
        Account acc = new Account(Name = name);
        if (doInsert) insert acc;
        return acc;
    }

    public static Contact createContact(Id accountId, String lastName, Boolean doInsert) {
        Contact con = new Contact(
            AccountId = accountId,
            LastName = lastName,
            Email = lastName.toLowerCase().replaceAll('\\s+', '') + '@test.example.com'
        );
        if (doInsert) insert con;
        return con;
    }

    public static User createStandardUser(String profileName, Boolean doInsert) {
        Profile p = [SELECT Id FROM Profile WHERE Name = :profileName LIMIT 1];
        String uniqueness = String.valueOf(Datetime.now().getTime());
        User u = new User(
            Alias = 'tst' + uniqueness.right(5),
            Email = 'testuser' + uniqueness + '@test.example.com',
            EmailEncodingKey = 'UTF-8',
            LastName = 'TestUser',
            LanguageLocaleKey = 'en_US',
            LocaleSidKey = 'en_US',
            ProfileId = p.Id,
            TimeZoneSidKey = 'America/New_York',
            UserName = 'testuser' + uniqueness + '@test.example.com'
        );
        if (doInsert) insert u;
        return u;
    }
}
```

---

## Post-generation checklist

After creating all files, the agent should:

1. **Verify** every `.cls` has its `-meta.xml` companion.
2. **Verify** metadata XML is well-formed (objects, events, CMDT).
3. **List** created files so the user can review before deploying.
4. **Suggest next step**: `sf project deploy start -d <path> -o <org>` to push to a scratch org or sandbox.

## What this skill does NOT create

- Object-specific triggers, handlers, services, selectors, or domains (those are per-feature).
- LWC components.
- Permission sets or profiles (the user should create `PS_ViewApplicationLogs` manually for the log object).

## Related skills

- [apex-architecture](../apex-architecture/SKILL.md) — full patterns for every layer built on top of this foundation.
- [lwc-best-practices](../lwc-best-practices/SKILL.md) — how controllers use `ServiceResponseDto` with LWC.
- [salesforce-metadata-standards](../salesforce-metadata-standards/SKILL.md) — naming and description rules for the metadata created here.
