---
name: lwc-best-practices
description: >-
  Lightning Web Components: composition, @api, lifecycle, @wire and imperative
  Apex vs LDS, caching and refresh patterns, LMS, navigation, SLDS 2 and
  styling hooks, accessibility, error handling, and Jest testing. Covers
  client-side security expectations (LWS, CSP, static resources, no secrets in
  the browser). Use when building, refactoring, reviewing, or debugging LWC,
  Experience Cloud UI, or LWC Jest tests.
metadata:
  domain: salesforce-lwc
  salesforce-era: spring-26
---

# Lightning Web Components Best Practices

Modern, performant, accessible LWC patterns. No Aura. No third-party frameworks.
Aligned with SLDS 2, Lightning Web Security, and Spring '26 standards.

## Related skills

- **Apex layers, DTOs, and server security:** [apex-architecture](../apex-architecture/SKILL.md) — thin controllers, DTOs, sharing/FLS, bulk-safe services for LWC.
- **Metadata naming and governance:** [salesforce-metadata-standards](../salesforce-metadata-standards/SKILL.md) — labels, objects, fields, and automation naming for UI-facing schema.

**Governor limits:** LWC runs in the browser; SOQL/DML limits apply in Apex invoked from LWC. Prefer fewer round-trips, smaller payloads, and cacheable `@wire` where appropriate; see [performance.md](patterns/performance.md) and Apex skill for bulk patterns.

## Universal Rules

1. **Composition over inheritance** — build complex UIs from small, focused components
2. **Props down, events up** — pass data via `@api`, communicate back via `CustomEvent`
3. **Use `lwc:if` / `lwc:elseif` / `lwc:else`** — `if:true` and `if:false` are deprecated
4. **No complex template expressions** — use getters in JavaScript instead
5. **Prefer LDS over Apex** for simple CRUD — use Apex only when logic demands it
6. **SLDS 2 with global styling hooks** — design tokens are deprecated
7. **Always clean up** — remove listeners and timers in `disconnectedCallback`

## Component Architecture Decision

```
Is it a simple CRUD form?
  YES -> lightning-record-form / record-edit-form / record-view-form
  NO  -> Does it need reusable logic across components?
           YES -> Create a shared JS module (service component)
           NO  -> Build a standard LWC with Apex if needed
```

## Quick Naming Reference

| Element | Convention | Example |
|---------|-----------|---------|
| Component folder | camelCase | `accountCard` |
| HTML tag in markup | kebab-case (auto) | `<c-account-card>` |
| JS class name | PascalCase | `AccountCard` |
| Public property | camelCase with `@api` | `@api recordId` |
| Private field | camelCase, no prefix | `isLoading` |
| Event name | lowercase, no hyphens, no `on` prefix | `new CustomEvent('itemselected')` |
| Handler in parent | `on` + event name | `onitemselected={handleItemSelected}` |
| Handler method | `handle` + action | `handleItemSelected(event)` |
| Getter | camelCase, descriptive | `get formattedTotal()` |
| Constant | UPPER_SNAKE_CASE | `const MAX_RECORDS = 50` |
| CSS class | SLDS utility or BEM | `slds-m-bottom_small` |
| Test file | `__tests__/componentName.test.js` | `__tests__/accountCard.test.js` |

## Data Access Decision Table

| Need | Approach | Why |
|------|----------|-----|
| Display record fields | `lightning-record-view-form` | Zero Apex, auto FLS |
| Edit single record | `lightning-record-edit-form` | Built-in validation |
| Full CRUD form | `lightning-record-form` | Auto layout |
| Read-only list (cacheable) | `@wire` + Apex `(cacheable=true)` | Client cache, reactive |
| DML operation | Imperative Apex call | Cannot use wire for DML |
| Complex multi-object query | Imperative Apex call | Wire limitations |
| Refresh after mutation | `refreshApex()` or `notifyRecordUpdateAvailable` | Cache invalidation |

## Lifecycle Hooks Quick Reference

| Hook | Fires | Use For | Avoid |
|------|-------|---------|-------|
| `constructor()` | Instance created | Declare fields | DOM access, `@api` reads |
| `connectedCallback()` | Inserted in DOM | Fetch data, subscribe LMS, init | Heavy DOM manipulation |
| `renderedCallback()` | After every render | Third-party libs, DOM measurements | State changes (infinite loop) |
| `disconnectedCallback()` | Removed from DOM | Cleanup listeners, timers, LMS | Async calls |
| `errorCallback(error, stack)` | Child error | Error boundary UI | Business logic |

## Communication Decision Table

| Relationship | Pattern | Mechanism |
|-------------|---------|-----------|
| Parent to Child | Props down | `@api` properties |
| Parent calls Child method | Public method | `@api` method on child |
| Child to Parent | Events up | `new CustomEvent('name', { detail })` |
| Sibling to Sibling | Shared parent | Parent relays events via handler |
| Unrelated components | LMS | `@wire(MessageContext)` + `publish` / `subscribe` |
| Component to Flow | Flow events | `FlowNavigationNextEvent`, `FlowAttributeChangeEvent` |

## Pattern Reference (Progressive Disclosure)

| Pattern | File | When to Read |
|---------|------|-------------|
| Component Structure | [component-structure.md](patterns/component-structure.md) | Creating components, naming, file layout |
| Data and State | [data-and-state.md](patterns/data-and-state.md) | Wire, imperative Apex, LDS, reactivity |
| Communication | [communication.md](patterns/communication.md) | Events, LMS, parent-child, navigation |
| Performance | [performance.md](patterns/performance.md) | Lazy loading, rendering, caching |
| Security & trust | [security-and-trust.md](patterns/security-and-trust.md) | LWS, CSP, static resources, secrets, DOM boundaries |
| Styling and Accessibility | [styling-and-a11y.md](patterns/styling-and-a11y.md) | CSS, SLDS 2, ARIA, keyboard navigation |
| Error Handling | [error-handling.md](patterns/error-handling.md) | Try-catch, toasts, error boundaries |
| Testing | [testing.md](patterns/testing.md) | Jest setup, mocking wire, DOM assertions |

## Anti-Patterns to Flag

| Anti-Pattern | Problem | Fix |
|-------------|---------|-----|
| Using `if:true` / `if:false` | Deprecated, may be removed | Use `lwc:if` / `lwc:elseif` / `lwc:else` |
| Complex template expressions | Hard to debug, poor performance | Extract to JS getter |
| `@track` on primitives | Unnecessary since LWC 1.17 | Remove `@track`; primitives auto-reactive |
| Mutating `@api` properties | Breaks one-way data flow | Copy to private field, fire event to parent |
| DOM manipulation in `renderedCallback` | Infinite render loops | Use boolean guard or move to `connectedCallback` |
| Hardcoded record IDs | Breaks across orgs and sandboxes | Use Custom Labels or Custom Metadata |
| No `disconnectedCallback` cleanup | Memory leaks | Unsubscribe LMS, clear intervals |
| Inline styles instead of SLDS | Inconsistent UI, no theme support | Use SLDS classes or styling hooks |
| `console.log` in production | Performance hit, exposed internals | Remove or use conditional logging |
| Loading external JS from CDN | CSP violations, security risk | Upload as Static Resource |
| Pub/Sub module (legacy) | Unmaintained, no official support | Use Lightning Message Service |
| Returning raw SObjects from Apex | Exposes schema, no field control | Use DTOs (see apex-architecture skill) |
