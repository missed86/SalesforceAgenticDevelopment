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
| 11 | `TriggerControlTest` | Apex test | Tests for TriggerControl |
| 12 | `LoggerTest` | Apex test | Tests for Logger + LogEventSubscriber |
| 13 | `ServiceResponseDtoTest` | Apex test | Tests for ServiceResponseDto |
| 14 | `RecordTypesTest` | Apex test | Tests for RecordTypes utility |

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

## Test Classes

Every foundation class needs tests. The agent must generate these alongside the main classes.

### 11. TriggerControlTest.cls

```apex
@isTest
private class TriggerControlTest {

    @isTest
    static void isDisabled_shouldReturnFalseByDefault() {
        Assert.isFalse(TriggerControl.isDisabled('AccountTrigger'));
    }

    @isTest
    static void disable_shouldDisableSpecificTrigger() {
        TriggerControl.disable('AccountTrigger');

        Assert.isTrue(TriggerControl.isDisabled('AccountTrigger'));
        Assert.isFalse(TriggerControl.isDisabled('ContactTrigger'));
    }

    @isTest
    static void enable_shouldReEnableTrigger() {
        TriggerControl.disable('AccountTrigger');
        TriggerControl.enable('AccountTrigger');

        Assert.isFalse(TriggerControl.isDisabled('AccountTrigger'));
    }

    @isTest
    static void disableAll_shouldDisableEveryTrigger() {
        TriggerControl.disableAll();

        Assert.isTrue(TriggerControl.isDisabled('AccountTrigger'));
        Assert.isTrue(TriggerControl.isDisabled('AnythingElse'));
        Assert.isTrue(TriggerControl.isDisabledAll());
    }

    @isTest
    static void isDisabled_shouldRespectCmdt() {
        // TriggerSetting__mdt records are read-only in tests.
        // If no CMDT record exists for the name, isDisabled returns false.
        Assert.isFalse(TriggerControl.isDisabled('NonExistentTrigger'));
    }
}
```

### 12. LoggerTest.cls

```apex
@isTest
private class LoggerTest {

    @isTest
    static void error_withException_shouldPublishEvent() {
        Test.startTest();
        try {
            Integer result = 1 / 0;
        } catch (Exception ex) {
            Logger.error('LoggerTest', 'error_withException', ex);
        }
        Test.stopTest();

        List<ApplicationLog__c> logs = [
            SELECT Level__c, SourceClass__c, SourceMethod__c, Message__c, StackTrace__c
            FROM ApplicationLog__c
        ];
        Assert.areEqual(1, logs.size(), 'Should persist one log record');
        Assert.areEqual('ERROR', logs[0].Level__c);
        Assert.areEqual('LoggerTest', logs[0].SourceClass__c);
        Assert.isTrue(String.isNotBlank(logs[0].StackTrace__c));
    }

    @isTest
    static void error_withMessage_shouldIncludeRecordId() {
        Account acc = new Account(Name = 'Test');
        insert acc;

        Test.startTest();
        Logger.error('LoggerTest', 'error_withMessage', 'Something broke', acc.Id);
        Test.stopTest();

        List<ApplicationLog__c> logs = [
            SELECT RecordId__c, Message__c FROM ApplicationLog__c
        ];
        Assert.areEqual(1, logs.size());
        Assert.areEqual(acc.Id, logs[0].RecordId__c);
    }

    @isTest
    static void warn_shouldPersistWarnLevel() {
        Test.startTest();
        Logger.warn('LoggerTest', 'warn_test', 'Something suspicious');
        Test.stopTest();

        List<ApplicationLog__c> logs = [SELECT Level__c FROM ApplicationLog__c];
        Assert.areEqual(1, logs.size());
        Assert.areEqual('WARN', logs[0].Level__c);
    }

    @isTest
    static void info_shouldPersistInfoLevel() {
        Test.startTest();
        Logger.info('LoggerTest', 'info_test', 'Operation completed');
        Test.stopTest();

        List<ApplicationLog__c> logs = [SELECT Level__c FROM ApplicationLog__c];
        Assert.areEqual(1, logs.size());
        Assert.areEqual('INFO', logs[0].Level__c);
    }
}
```

**Note:** `Test.stopTest()` forces Platform Event delivery synchronously, so
the `LogEventSubscriber` trigger fires and inserts into `ApplicationLog__c`
before the assertions run.

### 13. ServiceResponseDtoTest.cls

```apex
@isTest
private class ServiceResponseDtoTest {

    @isTest
    static void success_withMessage_shouldSetFields() {
        ServiceResponseDto dto = ServiceResponseDto.success('All good');

        Assert.isTrue(dto.isSuccess);
        Assert.areEqual('All good', dto.message);
        Assert.isNull(dto.data);
    }

    @isTest
    static void success_withData_shouldIncludePayload() {
        Map<String, String> payload = new Map<String, String>{ 'key' => 'value' };
        ServiceResponseDto dto = ServiceResponseDto.success('Done', payload);

        Assert.isTrue(dto.isSuccess);
        Assert.areEqual('Done', dto.message);
        Assert.isNotNull(dto.data);
    }

    @isTest
    static void failure_shouldSetIsSuccessFalse() {
        ServiceResponseDto dto = ServiceResponseDto.failure('Something went wrong');

        Assert.isFalse(dto.isSuccess);
        Assert.areEqual('Something went wrong', dto.message);
        Assert.isNull(dto.data);
    }
}
```

### 14. RecordTypesTest.cls

```apex
@isTest
private class RecordTypesTest {

    @isTest
    static void get_withValidRecordType_shouldReturnId() {
        // Account usually has a Master RT available in every org.
        // If the org has no custom RTs, use getAll to verify empty map behavior.
        Map<String, Schema.RecordTypeInfo> rtInfos =
            Account.SObjectType.getDescribe().getRecordTypeInfosByDeveloperName();

        if (rtInfos.containsKey('Master')) {
            Id rtId = RecordTypes.get(Account.SObjectType, 'Master');
            Assert.isNotNull(rtId);
        }
    }

    @isTest
    static void get_withInvalidName_shouldThrowAppException() {
        Boolean threw = false;
        try {
            RecordTypes.get(Account.SObjectType, 'CompletelyFakeRT_999');
        } catch (AppException e) {
            threw = true;
            Assert.isTrue(e.getMessage().contains('Record Type not found'));
        }
        Assert.isTrue(threw, 'Should throw AppException for non-existent RT');
    }

    @isTest
    static void getAll_shouldReturnMap() {
        Map<String, Id> result = RecordTypes.getAll(Account.SObjectType);
        Assert.isNotNull(result);
        // Map may be empty if no custom RTs exist — that's OK, no exception should occur.
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
