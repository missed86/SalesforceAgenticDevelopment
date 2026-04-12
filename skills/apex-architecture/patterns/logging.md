# Logging Pattern

Structured logging for debugging, auditing, and production monitoring.

## Core Principles

- **Never rely on `System.debug` alone** — debug logs are transient and size-limited
- **Log to a custom object** for persistent, queryable records
- **Use Platform Events** for async, non-blocking log capture
- **Include context**: class name, method, record IDs, user, timestamp
- **Log levels**: ERROR (always), WARN (anomalies), INFO (key operations), DEBUG (dev only)

## Lightweight Logger (Platform Event Based)

Platform Events decouple log creation from the main transaction — a log failure
never causes your business logic to fail.

### Log Event Definition

Create a Platform Event: `LogEvent__e`
- `Level__c` (Text) — ERROR, WARN, INFO, DEBUG
- `SourceClass__c` (Text) — class name
- `SourceMethod__c` (Text) — method name
- `Message__c` (Long Text) — error message or description
- `StackTrace__c` (Long Text) — exception stack trace
- `RecordId__c` (Text) — related record ID
- `UserId__c` (Text) — running user

### Logger Utility

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

### Log Subscriber (Persists to Custom Object)

Create a custom object: `ApplicationLog__c` with matching fields.

```apex
trigger LogEventTrigger on LogEvent__e (after insert) {
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

### Usage In Services

```apex
public with sharing class PaymentService {

    private static final String CLASS_NAME = 'PaymentService';

    public static PaymentResultDto processPayment(Id orderId, Decimal amount) {
        try {
            Order ord = OrderSelector.selectById(orderId);
            OrderDomain.validateForPayment(ord);

            PaymentGatewayResponse response = callPaymentGateway(ord, amount);

            if (!response.isSuccess) {
                Logger.warn(CLASS_NAME, 'processPayment',
                    'Payment declined for Order ' + orderId + ': ' + response.declineReason);
                return new PaymentResultDto(false, response.declineReason, null);
            }

            update new Order(Id = orderId, PaymentStatus__c = 'Paid');
            Logger.info(CLASS_NAME, 'processPayment',
                'Payment processed for Order ' + orderId);

            return new PaymentResultDto(true, 'Payment processed', response.transactionId);

        } catch (Exception e) {
            Logger.error(CLASS_NAME, 'processPayment', e);
            throw new PaymentException('Payment failed: ' + e.getMessage(), e);
        }
    }
}
```

### Usage In Batch

```apex
public void execute(Database.BatchableContext bc, List<Account> scope) {
    List<Database.SaveResult> results = Database.update(scope, false);

    for (Integer i = 0; i < results.size(); i++) {
        if (!results[i].isSuccess()) {
            Logger.error('StaleAccountBatch', 'execute',
                results[i].getErrors()[0].getMessage(), scope[i].Id);
        }
    }
}
```

## When to Use Each Approach

| Approach | When | Retention |
|----------|------|-----------|
| `System.debug()` | Local development, temporary debugging | Transient (debug logs) |
| Platform Event → Custom Object | Production errors, auditing | Persistent, queryable |
| Custom Setting flag + Logger | Conditional verbose logging | Toggle per user/profile |

## Key Rules

- **Never** let logging cause a business transaction to fail — use Platform Events
- **Always** log: unhandled exceptions, callout failures, batch errors, security violations
- **Include context**: class, method, record ID, user — never just the message
- **Rotate logs**: schedule a batch to delete `ApplicationLog__c` records older than 30-90 days
- **`without sharing`** on Logger — logs must persist regardless of user permissions
