---
name: lwc-preview
description: >-
  Layout and visualize Lightning Web Components before production wiring: SLDS 2–first
  static previews, iterative design in Cursor's browser, clear handoff artifacts (@api,
  events, labels, states, accessibility). Use when mocking up LWC UI, iterating on
  spacing and hierarchy, or producing specs detailed enough to implement business logic
  without rework.
metadata:
  domain: salesforce-lwc
  salesforce-era: spring-26
---

# LWC Preview — SLDS 2 Layout & Cursor Browser Iteration

Goal: produce **layout-first** artifacts that mirror production LWC structure so that,
once approved in preview, you only add **data, Apex/LDS, and behavior** — not rework
structure or visual language.

## Related skills

- **Implementation, wire, security, tests:** [lwc-best-practices](../lwc-best-practices/SKILL.md)
- **Metadata naming for UI-facing schema:** [salesforce-metadata-standards](../salesforce-metadata-standards/SKILL.md)

---

## Principles

1. **SLDS 2 almost everywhere** — Prefer [Lightning Base Components](https://developer.salesforce.com/docs/component-library/overview/components) in the real LWC; in static preview, mirror their **visual and structural patterns** with SLDS blueprint classes (`slds-*`) and documented placeholders for base components (see below).
2. **Global styling hooks over deprecated tokens** — Use `--slds-g-*` hooks with fallbacks in CSS snippets; align with [styling guidance](../lwc-best-practices/patterns/styling-and-a11y.md).
3. **Preview is dev-only** — Loading SLDS from a CDN or local bundle for a browser tab is **only** for Cursor/design iteration. Production LWCs rely on the platform stylesheet and base components; do not ship preview CDN links in customer-facing apps.
4. **Design outputs are contracts** — Every preview session should tighten: layout regions, responsive breakpoints, empty/loading/error states, keyboard order, and the `@api` / event surface.

---

## Default workflow (Cursor browser)

### 1. Scaffold a preview folder

For component `featureWidget`, create a sibling folder used only for layout proofing:

```text
featureWidget/
  preview/
    index.html       # entry for Cursor Simple Browser / local open
    preview.css      # optional :host-like wrapper using --slds-g-* hooks
    README-PREVIEW.md # optional: URL notes; keep short
```

Commit `preview/` only if the team wants versioned mocks; otherwise keep it gitignored locally — match repo policy.

### 2. Load SLDS 2 for static HTML

Static HTML cannot render real `<lightning-*>` custom elements without a Salesforce or LWC toolchain. For **visual** iteration:

- Include SLDS 2 CSS appropriate for **static markup** (Salesforce distributes design system assets; use the same major version your org targets). Typical approaches:
  - **Local:** Copy or build CSS from `@salesforce-ux/design-system` into `preview/vendor/` and link relatively.
  - **Temporary CDN** (iteration only): link the official SLDS stylesheet your team approves for dev — **never** paste that link into packaged LWC or Experience Cloud static resources as the primary styling strategy.

Wrap content in a root that mimics Lightning container spacing (e.g. `slds-scope` patterns per SLDS docs for static sites).

### 3. Map regions to future base components

In `index.html`, use **comments + parallel naming** so implementation is mechanical:

```html
<!-- PRODUCTION: lightning-card -->
<section class="slds-card" aria-labelledby="feature-widget-heading">
    <!-- PRODUCTION: lightning-card title slot -->
    <div class="slds-card__header slds-grid">
        <header class="slds-media slds-media_center">
            <!-- PRODUCTION: lightning-icon alternativeText -->
            ...
        </header>
    </div>
    <!-- PRODUCTION: lightning-card body -->
    <div class="slds-card__body slds-card__body_inner">
        <!-- PRODUCTION: lightning-datatable | lightning-record-picker | custom -->
        ...
    </div>
</section>
```

Rule: **every** interactive control in preview should name the target base primitive (`lightning-button`, `lightning-input`, `lightning-combobox`, etc.).

### 4. Iterate in Cursor's browser

- Open `preview/index.html` via **Cursor Simple Browser** or live preview.
- Resize to validate `slds-*` responsive modifiers (`slds-medium-size_*`, `slds-large-size_*`, etc.).
- Capture issues as checklist updates (spacing tokens, breakpoint changes), not vague notes.

### 5. Handoff package (production-ready specification)

Deliver alongside or inside the preview folder a concise spec (Markdown or structured comments) containing:

| Area | What to specify |
|------|-----------------|
| **Public API** | `@api` properties: name, type, required, default, description |
| **Events** | Event name, payload (`detail`), bubbling/composed, parent handler name |
| **Labels & i18n** | Custom Label API names or placeholder keys for every string |
| **Data** | LDS (`getRecord`, `getRelatedListRecords`) vs Apex: method names, DTO shapes |
| **States** | Loading, empty, error, success; which SLDS patterns (slds-illustration, spinner inside region, etc.) |
| **a11y** | Landmark roles, headings (`h1`–`h3`), `aria-*`, focus order for modals/wizards |
| **Responsive** | Mobile vs tablet vs desktop behavior; what collapses or stacks |

This package should be sufficient for another developer to implement **logic only** without redesigning markup.

---

## SLDS 2 composition priority (preview + production)

Same order as [lwc-best-practices styling](../lwc-best-practices/patterns/styling-and-a11y.md):

1. **Lightning Base Components** — Use in real `.html`; in preview, annotate and mimic structure.
2. **SLDS blueprint utilities** — `slds-grid`, `slds-col`, `slds-gutters`, spacing, typography.
3. **Global styling hooks** — `--slds-g-color-*`, `--slds-g-spacing-*`, `--slds-g-font-size-*` in scoped CSS.

Avoid bespoke pixel chasing unless a hook cannot express the need.

---

## Preview HTML patterns (copy-paste starters)

### Page shell

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="utf-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1" />
    <title>LWC Preview — FeatureWidget</title>
    <!-- DEV ONLY: link SLDS 2 stylesheet (local path or approved dev CDN) -->
    <link rel="stylesheet" href="./vendor/slds.min.css" />
    <link rel="stylesheet" href="./preview.css" />
</head>
<body class="slds-scope">
    <div class="slds-p-around_medium">
        <!-- component preview root -->
    </div>
</body>
</html>
```

### Responsive grid

```html
<div class="slds-grid slds-wrap slds-gutters">
    <div class="slds-col slds-size_1-of-1 slds-medium-size_1-of-2">
        <!-- column -->
    </div>
</div>
```

### States (document all three in preview)

- **Loading:** `lightning-spinner` in prod → preview with `slds-spinner` markup from SLDS or a labeled placeholder region.
- **Empty:** Illustration + short copy; specify label keys.
- **Error:** `inline` vs `toast` — state which; errors from Apex should map to `lightning-form` or toast pattern.

---

## From preview to production LWC (checklist)

- [ ] Replace static regions with actual `<lightning-*>` tags; remove dev-only CSS links.
- [ ] Move strings to Custom Labels; wire `@salesforce/label/...`.
- [ ] Implement `@api` and `CustomEvent` contracts exactly as in the handoff table.
- [ ] Add `lwc:if` / `lwc:elseif` / `lwc:else` for state branches (no deprecated `if:true`).
- [ ] Add Jest tests for conditional rendering and events ([testing patterns](../lwc-best-practices/patterns/testing.md)).
- [ ] Verify in Lightning Experience / Experience Cloud with real data and FLS.

---

## Anti-patterns

| Avoid | Prefer |
|-------|--------|
| Pixel-perfect custom CSS that ignores SLDS hooks | Hooks + utilities; justify exceptions |
| Anonymous buttons/links in preview | Explicit target base component + label key |
| Skipping empty/error in preview | Same layout shell for all states |
| Preview CDN SLDS in packaged static resources for prod | Platform-delivered SLDS + base components |

---

## When this skill applies

- New LWC screens or refactors where **layout risk** is high (dashboards, wizards, dense tables).
- Need to **demo** structure to stakeholders before backend is ready.
- Pairing with Cursor browser for **fast feedback** on spacing, density, and responsive behavior.

Stop using preview-only assets once the component is merged to the package; keep the **handoff spec** as living documentation if your team maintains ADRs or Confluence — optional, not required by this skill.
