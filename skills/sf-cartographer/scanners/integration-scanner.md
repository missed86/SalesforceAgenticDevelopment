# Integration Scanner

Detailed instructions for scanning named credentials, external credentials, remote site settings, connected apps, and Apex-based integrations.

## Discovery — Named Credentials

1. Glob for `**/namedCredentials/*.namedCredential-meta.xml`.
2. For each file, extract:

| Field | XML Element | Notes |
|-------|------------|-------|
| Name | `<fullName>` or filename | API name |
| Label | `<label>` | Display name |
| Endpoint | `<endpoint>` | Base URL (may be omitted in newer format) |
| Protocol | `<principalType>` | NamedUser, PerUser, Anonymous |
| Generate auth header | `<generateAuthorizationHeader>` | true/false |
| Allow merge fields | `<allowMergeFieldsInHeader>` / `<allowMergeFieldsInBody>` | true/false |
| External credential | `<externalCredential>` | For new-style named credentials (Spring '23+) |

### Named Credential Formats

Salesforce has two formats:

**Legacy format** (pre-Spring '23): endpoint, protocol, and auth config in the named credential itself.

**New format** (Spring '23+): Named Credential references an External Credential for auth. The named credential only holds the endpoint URL.

Document which format each credential uses.

---

## Discovery — External Credentials

1. Glob for `**/externalCredentials/*.externalCredential-meta.xml`.
2. Extract:

| Field | XML Element |
|-------|------------|
| Name | `<fullName>` or filename |
| Label | `<label>` |
| Authentication protocol | `<authenticationProtocol>` (OAuth2, Custom, NoAuth, etc.) |
| External credential parameters | `<externalCredentialParameters>` |

Cross-reference: which named credentials reference this external credential.

---

## Discovery — Remote Site Settings

1. Glob for `**/remoteSiteSettings/*.remoteSite-meta.xml`.
2. Extract:

| Field | XML Element |
|-------|------------|
| Name | `<fullName>` or filename |
| URL | `<url>` |
| Description | `<description>` |
| Active | `<isActive>` |
| Disable protocol security | `<disableProtocolSecurity>` |

Flag any remote sites with `disableProtocolSecurity = true` as a security concern.

---

## Discovery — Connected Apps

1. Glob for `**/connectedApps/*.connectedApp-meta.xml`.
2. Extract:

| Field | XML Element |
|-------|------------|
| Name | `<fullName>` or filename |
| Label | `<label>` |
| Description | `<description>` |
| Contact email | `<contactEmail>` |
| OAuth enabled | `<oauthConfig>` (presence indicates OAuth) |
| Callback URL | `<oauthConfig><callbackUrl>` |
| Scopes | `<oauthConfig><scopes>` |
| SAML enabled | `<samlConfig>` (presence indicates SAML) |
| Certificate | `<certificate>` |
| Plugin | `<plugin>` |

---

## Discovery — CSP Trusted Sites

1. Glob for `**/cspTrustedSites/*.cspTrustedSite-meta.xml`.
2. Extract:

| Field | XML Element |
|-------|------------|
| Name | `<fullName>` |
| Endpoint URL | `<endpointUrl>` |
| Description | `<description>` |
| Active | `<isActive>` |
| Context | `<context>` (All, LEX, Communities, etc.) |

---

## Discovery — Apex Callout Classes

This is a cross-cutting scan using data from the Apex scanner. Identify classes that:

1. **Use `Http`, `HttpRequest`, `HttpResponse`** — direct HTTP callouts
2. **Implement `HttpCalloutMock`** — test mocks for callouts
3. **Use `WebServiceCallout`** — SOAP callouts
4. **Import `ExternalService`** — declarative external service classes
5. **Reference named credentials** — search for `callout:` prefix in strings
6. **Use `Continuation`** — async callouts from Visualforce or LWC

For each callout class found, document:
- Class name
- Callout type (REST / SOAP / External Service)
- Named credential used (if detectable from `callout:CredentialName`)
- Whether a mock class exists for testing

---

## Discovery — Platform Event Integrations

Cross-reference with the data model scanner:
- Platform Events (`__e`) defined in the project
- Apex classes that call `EventBus.publish()`
- Apex triggers on `__e` objects (subscribers)
- Flows triggered by platform events

This creates an integration map of event-driven patterns.

---

## Output Format for integrations.md

### Summary

```markdown
## Summary

| Category | Count |
|----------|-------|
| Named Credentials | 3 |
| External Credentials | 2 |
| Remote Site Settings | 4 |
| Connected Apps | 1 |
| CSP Trusted Sites | 2 |
| Apex Callout Classes | 5 |
| Platform Event Channels | 2 |
```

### Named Credentials

```markdown
## Named Credentials

| Name | Endpoint | Auth Type | External Credential | Format |
|------|----------|-----------|--------------------:|--------|
| `PaymentGateway` | https://api.payments.example.com | OAuth 2.0 | `PaymentGateway_EC` | New |
| `ERPSystem` | https://erp.internal.example.com/api | Named User | — | Legacy |
| `SlackWebhook` | https://hooks.slack.com/services/... | No Auth | — | Legacy |
```

### External Credentials

```markdown
## External Credentials

| Name | Protocol | Used By |
|------|----------|---------|
| `PaymentGateway_EC` | OAuth 2.0 (Client Credentials) | `PaymentGateway` named credential |
| `MapsProvider_EC` | Custom (API Key) | `MapsProvider` named credential |
```

### Remote Site Settings

```markdown
## Remote Site Settings

| Name | URL | Active | Security ⚠️ |
|------|-----|:------:|:-----------:|
| `ERPLegacy` | https://old-erp.example.com | Yes | OK |
| `TestSandbox` | http://sandbox.test.com | Yes | ⚠️ HTTP only |
| `PartnerAPI` | https://partner.example.com | No | — |
```

Flag `disableProtocolSecurity = true` or `http://` URLs with ⚠️.

### Connected Apps

```markdown
## Connected Apps

| App | OAuth | SAML | Contact | Description |
|-----|:-----:|:----:|---------|-------------|
| `InternalMobileApp` | Yes | No | dev@company.com | Mobile app for field sales |
```

### Apex Integration Classes

```markdown
## Apex Callout Classes

| Class | Type | Named Credential | Mock Class | Package |
|-------|------|-----------------|------------|---------|
| `PaymentGatewayService` | REST | `PaymentGateway` | `PaymentGatewayMock` | force-app |
| `ERPSyncService` | REST | `ERPSystem` | `ERPSyncMock` | force-app |
| `AddressValidationService` | REST | `MapsProvider` | `AddressValidationMock` | force-app |
```

### Event-Driven Integration Map

```markdown
## Platform Event Integration

### LogEvent__e

```text
Publisher(s) → LogEvent__e → Subscriber(s)
Logger.cls   ─────────────→ LogEventSubscriber.trigger
                           → ApplicationLog__c (persistence)
```

### OrderEvent__e

```text
Publisher(s) → OrderEvent__e → Subscriber(s)
OrderService ───────────────→ Order_PE_ProcessEvent (Flow)
                             → InventoryUpdateJob.cls (Queueable)
```
```

### Integration Architecture Diagram

If multiple integration points exist, provide an overview:

```markdown
## Integration Architecture

┌─────────────┐     callout:PaymentGateway     ┌──────────────────┐
│ PaymentSvc   │ ──────────────────────────────→ │ Payment Gateway   │
└─────────────┘                                 └──────────────────┘

┌─────────────┐     callout:ERPSystem           ┌──────────────────┐
│ ERPSyncSvc   │ ──────────────────────────────→ │ ERP System        │
└─────────────┘                                 └──────────────────┘

┌─────────────┐     Platform Event              ┌──────────────────┐
│ OrderService  │ ── OrderEvent__e ────────────→ │ InventoryJob      │
└─────────────┘                                 └──────────────────┘
```
