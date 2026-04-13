# Apex Scanner

Detailed instructions for scanning Apex classes and triggers.

## Discovery

### Classes

1. Glob for `**/classes/**/*.cls` across all package directories (skip `*-meta.xml`).
2. For each `.cls` file, read the **first 30 lines** to capture:
   - Class declaration line (`public class`, `public with sharing class`, `global class`, etc.)
   - Interfaces implemented (`implements Queueable`, `implements Database.Batchable`, etc.)
   - Annotations (`@isTest`, `@RestResource`, `@AuraEnabled` on methods, `@InvocableMethod`)
   - Sharing mode (`with sharing`, `without sharing`, `inherited sharing`, or omitted)
3. Count total lines of code per file (use line count, not character count).

### Triggers

1. Glob for `**/triggers/**/*.trigger` (skip `*-meta.xml`).
2. Read the **first 5 lines** to capture:
   - Object the trigger fires on
   - Events handled (`before insert, after update`, etc.)
   - Whether it delegates to a handler class or contains inline logic

---

## Classification

Assign each class to exactly one layer based on these rules (first match wins):

| Priority | Condition | Layer |
|----------|-----------|-------|
| 1 | Has `@isTest` annotation | **Test** |
| 2 | Name ends with `Test` AND is in test context | **Test** |
| 3 | Name is `TestDataFactory` or ends with `DataFactory` or `Mock` or `Stub` | **Test Support** |
| 4 | Name ends with `TriggerHandler` or `TriggerHelper` | **Trigger Handler** |
| 5 | Name ends with `Controller` AND has `@AuraEnabled` methods | **Controller** |
| 6 | Name ends with `RestService` or has `@RestResource` annotation | **REST Resource** |
| 7 | Name ends with `Service` (not `RestService`) | **Service** |
| 8 | Name ends with `Selector` | **Selector** |
| 9 | Name ends with `Domain` | **Domain** |
| 10 | Name ends with `Dto` or `Request` or `Response` or `Wrapper` | **DTO / Wrapper** |
| 11 | Implements `Database.Batchable` | **Batch** |
| 12 | Implements `Queueable` | **Queueable** |
| 13 | Implements `Schedulable` | **Schedulable** |
| 14 | Name ends with `Job` or `Worker` | **Async** |
| 15 | Extends `Exception` or name ends with `Exception` | **Exception** |
| 16 | Name is `Logger` or contains `Log` and relates to logging | **Infrastructure** |
| 17 | Name is `TriggerControl` or `RecordTypes` or `AppConstants` or `FeatureToggle` | **Infrastructure** |
| 18 | Has `@InvocableMethod` or `@InvocableVariable` | **Invocable** |
| 19 | Everything else | **Utility / Uncategorized** |

### Sharing Mode Extraction

Parse the class declaration for sharing keywords:

```
public with sharing class Foo      → "with sharing"
public without sharing class Foo   → "without sharing"
public inherited sharing class Foo → "inherited sharing"
public class Foo                   → "not specified"
```

### Test Coverage Detection

For each non-test class `FooService`, check if a class named `FooServiceTest` exists anywhere in the scanned classes. Record:
- `Has Test`: Yes / No
- If No, note it in the coverage analysis section

---

## Output Format for apex-inventory.md

### Summary Section

```markdown
## Summary

| Layer | Count |
|-------|-------|
| Trigger Handler | 5 |
| Controller | 3 |
| Service | 8 |
| Selector | 4 |
| ... | ... |
| **Total** | **N** |
```

### Triggers Section

```markdown
## Triggers

| Trigger | Object | Events | Handler Class | Package |
|---------|--------|--------|---------------|---------|
| `AccountTrigger` | Account | before insert, after update | `AccountTriggerHandler` | force-app |
```

### Per-Layer Sections

For each non-empty layer, create a section:

```markdown
## Service Layer

| Class | Sharing | Lines | Has Test | Package |
|-------|---------|-------|----------|---------|
| `AccountService` | with sharing | 145 | Yes | force-app |
| `OrderService` | with sharing | 230 | Yes | orders-app |
| `PaymentService` | inherited sharing | 89 | No ⚠️ | force-app |
```

### Exceptions Section

```markdown
## Custom Exceptions

| Exception | Extends | Used By |
|-----------|---------|---------|
| `AppException` | Exception | Base exception for all custom errors |
| `PaymentException` | AppException | PaymentService |
```

### Architecture Detection Notes

At the top of `architecture.md`, include:

```markdown
## Detected Layers

Based on {N} Apex classes scanned:

- ✅ Trigger Handlers ({n} classes)
- ✅ Service Layer ({n} classes)
- ✅ Selector Layer ({n} classes)
- ❌ Domain Layer (not detected)
- ✅ Controllers ({n} classes)
- ✅ DTOs ({n} classes)

## Architecture Diagram

{ASCII diagram showing only the layers that exist}
```

The ASCII diagram should follow this template but only include rows for detected layers:

```
┌──────────────────────────────────────────────────────┐
│  Entry Points                                        │
│  Trigger │ LWC Controller │ Batch │ REST API │ Queue │
└────┬─────┴───────┬────────┴───┬───┴────┬─────┴──┬────┘
     │             │            │        │        │
     ▼             ▼            ▼        ▼        ▼
┌──────────────────────────────────────────────────────┐
│  Service Layer  (N classes)                          │
├──────────────────────────────────────────────────────┤
│  Selector Layer (N classes)                          │
└──────────────────────────────────────────────────────┘
```

---

## Parsing Tips

### Detect @AuraEnabled Methods

Search within class body for `@AuraEnabled` — if found on at least one method, the class exposes Apex to LWC. Useful for:
- Classifying as Controller if name ends with `Controller`
- Documenting in LWC catalog which Apex methods each component calls

### Detect Callout Classes

Search for any of: `Http `, `HttpRequest`, `HttpResponse`, `HttpCalloutMock`, `WebServiceCallout`, `ExternalService`. These classes should appear in `integrations.md` as well.

### Detect Platform Event Publishing

Search for `EventBus.publish` — these classes publish platform events. Cross-reference with data model scanner.

### Detect SOQL Patterns

For selector classes, optionally note if they use:
- `WITH SECURITY_ENFORCED`
- `WITH USER_MODE`
- `Database.query` (dynamic SOQL — flag for review)
- `Apex Cursors` (Database.getCursor / Database.getQueryLocator patterns)
