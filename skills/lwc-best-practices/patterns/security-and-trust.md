# LWC Security & Trust

Client-side guidance under **Lightning Web Security (LWS)** and platform Content Security Policy (CSP). Server-side enforcement (CRUD/FLS, sharing) lives in Apex — see [apex-architecture security.md](../../apex-architecture/patterns/security.md).

## Principles

1. **Treat the browser as untrusted** — Never embed API keys, OAuth secrets, or privileged tokens in LWC JavaScript, HTML, or static resources shipped to the client.
2. **Prefer platform primitives** — `lightning-input`, `lightning-form`, and LDS-backed components reduce foot-guns compared to raw HTML and manual DOM.
3. **Minimize third-party script** — External CDNs often violate CSP; load vetted code via **Static Resources** and keep supply chain small.

## Lightning Web Security (LWS)

- LWS isolates components and restricts dangerous patterns; do not rely on workarounds that weaken isolation.
- Avoid patterns that assume global `window` mutation or cross-namespace DOM access unless explicitly supported for your use case.

## Static resources and libraries

- Wrap third-party scripts in Static Resources; load with `loadScript` / `loadStyle` from `lightning/platformResourceLoader`.
- Audit minified bundles for `eval`, dynamic code injection, and unexpected network calls.

## DOM and markup

- Use **`lwc:dom="manual"`** only when required for third-party DOM integration; sanitize untrusted HTML and never assign unsanitized user input to `innerHTML`.
- Prefer Lightning base components for inputs and formatted output to reduce XSS surface.

## Data in the UI

- Do not log sensitive fields to `console` in production builds.
- For lists and forms, prefer **field-level aware** patterns (`lightning-record-*`, `@wire(getRecord)`) when business rules allow — Apex DTOs still need server-side FLS checks for imperative calls.

## Callouts from LWC

- The browser does not call external systems directly with org credentials; use **Named Credentials** and Apex (or supported integration patterns). Never expose Named Credential names or session identifiers in client-visible code beyond what the platform requires.

## Experience Cloud

- Respect guest-user restrictions; assume stricter object/field exposure. Combine declarative **profiles/permission sets** with Apex `with sharing` and explicit DTO shaping.
