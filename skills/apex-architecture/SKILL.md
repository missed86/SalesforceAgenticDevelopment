---
name: apex-architecture
description: >-
  Apex code architecture patterns, naming conventions, and best practices for
  Salesforce projects of any size. Pure Apex without frameworks (no fflib, no
  Flow, no AI). Covers trigger handlers, service/selector/domain/controller layers,
  LWC Apex controllers, DTOs, async patterns, security, testing, error handling,
  logging, callouts, REST APIs, bulkification, and governor limit optimization.
  Use when writing Apex classes, designing Salesforce architecture, creating
  triggers, services, selectors, controllers, tests, callouts, REST endpoints,
  LWC Apex integration, or reviewing Apex code quality.
---

# Apex Architecture Patterns (Framework-Free)

Pure Apex patterns optimized for projects from small to enterprise scale.
No external frameworks. No Flow. No AI. Just clean, scalable Apex.

## Quick Decision Tables

### Which Layer Do I Need?

| Situation | Layer | Suffix |
|-----------|-------|--------|
| Responding to DML events | Trigger + Handler | `Trigger` / `TriggerHandler` |
| Exposing Apex to LWC components | Controller | `Controller` |
| Reusable business logic callable from anywhere | Service | `Service` |
| Centralizing SOQL queries | Selector | `Selector` |
| Validations/defaults on a single SObject | Domain | `Domain` |
| Structuring data for LWC/API responses | DTO / Wrapper | `Dto` / `Response` |
| Background/heavy processing | Async (see table below) | varies |
| External HTTP communication | Callout / Integration | `Service` / `Job` |
| Exposing Apex as REST API | REST Resource | `RestService` |
| Persistent error tracking | Logging | `Logger` |

### Which Async Pattern?

| Need | Pattern | Why |
|------|---------|-----|
| Simple fire-and-forget, callout from trigger | `@future` | Lightest async, primitive params only |
| Chaining jobs, passing complex objects | `Queueable` | Modern replacement for @future |
| Processing thousands/millions of records | `Batch` | Chunked execution, configurable size |
| Running jobs on a schedule | `Schedulable` | CRON-based orchestrator |
| Decoupled event-driven communication | Platform Events | Pub/sub, cross-boundary |

### Security Keywords

| Keyword | Effect | When to Use |
|---------|--------|-------------|
| `with sharing` | Enforces record-level security | Default for most classes |
| `without sharing` | System mode, ignores sharing | Admin operations, isolated & minimal |
| `inherited sharing` | Inherits caller's context | Utility/service classes |
| `WITH SECURITY_ENFORCED` | FLS + object security in SOQL | Queries exposed to users |
| `WITH USER_MODE` | Full user-mode SOQL | Flexible FLS enforcement |
| `Security.stripInaccessible()` | Removes inaccessible fields | Sanitize before DML or return |

## Core Naming Conventions

| Element | Convention | Example |
|---------|-----------|---------|
| Class | PascalCase + descriptive suffix | `AccountTriggerHandler` |
| Method | camelCase, verb-first | `calculateTotalAmount()` |
| Variable | camelCase, descriptive | `accountsByOwnerId` |
| Constant | UPPER_SNAKE_CASE | `MAX_RETRY_COUNT` |
| Test class | Mirror + `Test` suffix | `AccountServiceTest` |
| Test factory | `TestDataFactory` | `TestDataFactory` |
| Custom exception | PascalCase + `Exception` | `PaymentProcessingException` |
| Trigger | `{SObject}Trigger` | `OpportunityTrigger` |
| Handler | `{SObject}TriggerHandler` | `OpportunityTriggerHandler` |
| Service | `{Domain}Service` | `OrderService` |
| Selector | `{SObject}Selector` | `ContactSelector` |
| Domain | `{SObject}Domain` | `CaseDomain` |
| Controller | `{Feature}Controller` | `InvoiceController` |
| Response DTO | `{Entity}Dto` | `AccountDto` |
| Request DTO | `{Action}Request` | `CreateOrderRequest` |
| Service Response | `ServiceResponseDto` | `ServiceResponseDto` |
| REST Resource | `{Resource}RestService` | `OrderRestService` |
| Integration | `{System}Service` | `PaymentGatewayService` |
| Logger | `Logger` | `Logger` |
| Trigger Control | `TriggerControl` | `TriggerControl` |

For the full naming guide, see [naming-conventions.md](examples/naming-conventions.md).

## Architecture At a Glance

```
┌──────────────────────────────────────────────────────┐
│  Entry Points                                        │
│  Trigger │ LWC Controller │ Batch │ REST API │ Queue │
└────┬─────┴───────┬────────┴───┬───┴────┬─────┴──┬───┘
     │             │            │        │        │
     │      ┌──────┴──────┐    │        │        │
     │      │ Controller  │    │        │        │
     │      │ (thin, @AE) │    │        │        │
     │      └──────┬──────┘    │        │        │
     │             │           │        │        │
     ▼             ▼           ▼        ▼        ▼
┌──────────────────────────────────────────────────────┐
│  Service Layer  (caller-agnostic business logic)     │
├──────────────────────────────────────────────────────┤
│  Domain Layer   (SObject validations/defaults)       │
├──────────────────────────────────────────────────────┤
│  Selector Layer (centralized SOQL)                   │
└──────────────────────────────────────────────────────┘
```

## Scaling Strategy

| Project Size | Recommended Layers | Notes |
|-------------|-------------------|-------|
| **Small** (< 10 classes) | Trigger + Handler + Controller | Keep it simple, inline queries OK |
| **Medium** (10-50 classes) | + Service + Selector + DTO | Extract reusable logic and queries |
| **Large** (50+ classes) | + Domain + Config + DI | Full separation, CMDT-driven config |

## Governor Limit Survival Rules

1. **Never** SOQL/DML inside loops — collect, then operate
2. **Always** bulkify — assume 200 records, code for collections
3. **Use Maps** for O(1) lookups instead of nested loops
4. **Leverage** `Test.startTest()`/`Test.stopTest()` for fresh limits in tests
5. **Offload** heavy work to Queueable/Batch
6. **Cache** config data with Platform Cache + CacheBuilder
7. **Use Apex Cursors** (Spring '26) for datasets over SOQL limits

## Spring '26 Features to Leverage

| Feature | Use Case |
|---------|----------|
| **Apex Cursors** | Stream millions of records without Batch |
| **RunRelevantTests** | Faster deploys with `@isTest(testFor=...)` |
| **System.Blob.toPDF()** | On-platform PDF generation |
| **Programmatic Picklist Values** | Dynamic picklist extraction by record type |

## Pattern Reference (Progressive Disclosure)

Read these only when you need the full pattern with code examples:

| Pattern | File | When to Read |
|---------|------|-------------|
| Trigger + Handler | [trigger-handler.md](patterns/trigger-handler.md) | Writing triggers or handlers |
| Controller (LWC) | [controller-layer.md](patterns/controller-layer.md) | Exposing Apex to LWC, wire vs imperative |
| DTO & Wrappers | [dto-wrapper.md](patterns/dto-wrapper.md) | Request/Response DTOs, pagination, serialization |
| Service Layer | [service-layer.md](patterns/service-layer.md) | Creating reusable business logic |
| Selector Layer | [selector-layer.md](patterns/selector-layer.md) | Centralizing SOQL queries |
| Domain Layer | [domain-layer.md](patterns/domain-layer.md) | SObject validations and defaults |
| Async Patterns | [async-patterns.md](patterns/async-patterns.md) | Background/bulk processing |
| Error Handling | [error-handling.md](patterns/error-handling.md) | Exceptions, error responses |
| Security | [security.md](patterns/security.md) | Sharing, CRUD/FLS, injection |
| Testing | [testing.md](patterns/testing.md) | Test classes, factories, mocking |
| Configuration | [configuration.md](patterns/configuration.md) | CMDT, Platform Cache, DI |
| Logging | [logging.md](patterns/logging.md) | Platform Event logger, structured logging |
| Callouts & REST API | [callout-integration.md](patterns/callout-integration.md) | HTTP callouts, Named Credentials, @RestResource |
| Naming & Comments | [naming-conventions.md](examples/naming-conventions.md) | Full naming reference, comment rules, ApexDoc |
| Project Structure | [project-structure.md](examples/project-structure.md) | SFDX folder organization, package directories |

## Anti-Patterns to Flag

| Anti-Pattern | Problem | Fix |
|-------------|---------|-----|
| SOQL/DML inside loop | Governor limit breach | Collect IDs, query/DML once outside |
| Multiple triggers per object | Non-deterministic execution | One trigger → one handler |
| Hardcoded IDs | Breaks across environments | Use Custom Metadata or Custom Labels |
| `seeAllData=true` in tests | Fragile, environment-dependent | Use TestDataFactory |
| Business logic in trigger body | Untestable, not reusable | Move to handler/service |
| Business logic in controller | Tight coupling to LWC, not reusable | Controller delegates to service |
| Returning raw SObjects to LWC | Exposes all fields, couples UI to schema | Use DTO with `@AuraEnabled` |
| Generic `catch (Exception e)` | Swallows errors silently | Catch specific types, log/rethrow |
| `without sharing` everywhere | Security holes | Default to `with sharing` or `inherited sharing` |
| No trigger bypass mechanism | Can't disable for data loads | Use `TriggerControl` with CMDT |
| `System.debug` only logging | Transient, not queryable | Use Platform Event → custom object Logger |
| Hardcoded callout URLs | Breaks across environments | Use Named Credentials |
| Concatenating SOQL identifiers | Injection risk | Validate against Schema.getGlobalDescribe |
| Multiple DML without Savepoint | Partial commits on failure | Use `Database.setSavepoint()` / rollback |
| Narrating comments (`// Query accounts`) | Noise, no value | Comment only **why**, never **what** |
| Commented-out code | Dead code, confuses readers | Delete it — use git history |
| No `without sharing` justification | Unclear security intent | Always comment **why** system mode is needed |
