# Styling and Accessibility

## SLDS 2 and Styling

### Three-Layer Approach

1. **Lightning Base Components** (preferred) — `lightning-button`, `lightning-card`, etc.
2. **SLDS Blueprint Classes** — `slds-grid`, `slds-col`, `slds-m-bottom_small`
3. **Global Styling Hooks** — CSS custom properties for advanced theming

Use the highest layer that meets the need. Drop to lower layers only when
the higher one lacks the required customization.

### Global Styling Hooks (Replacing Design Tokens)

Design tokens (e.g., `--lwc-colorBrand`) are deprecated in SLDS 2.
Use global styling hooks instead:

```css
/* accountCard.css */
.card-header {
    color: var(--slds-g-color-brand-base-50, #0176d3);
    font-size: var(--slds-g-font-size-5, 1rem);
    padding: var(--slds-g-spacing-4, 1rem);
    border-bottom: 1px solid var(--slds-g-color-border-base-1, #e5e5e5);
}
```

Always provide fallback values for hooks that may not be available in all
contexts (e.g., Experience Builder sites).

### CSS Scoping Rules

LWC uses Shadow DOM by default. CSS in a component only affects that
component and does not leak to parents, children, or siblings.

```css
/* This :host selector styles the component's outer element */
:host {
    display: block;
    padding: var(--slds-g-spacing-4, 1rem);
}

/* This only affects elements INSIDE this component */
.container {
    display: flex;
    gap: var(--slds-g-spacing-3, 0.75rem);
}
```

### CSS Best Practices

| Do | Do Not |
|----|--------|
| Use SLDS utility classes | Write inline styles |
| Use global styling hooks | Use deprecated design tokens |
| Use `:host` for component-level styles | Use `!important` |
| Use relative units (`rem`, `em`) | Use pixel values for spacing |
| Keep CSS files small and focused | Duplicate SLDS styles manually |
| Use SLDS Linter in VS Code | Ignore SLDS 2 migration warnings |

### Responsive Layout with SLDS Grid

```html
<div class="slds-grid slds-wrap slds-gutters">
    <div class="slds-col slds-size_1-of-1 slds-medium-size_1-of-2 slds-large-size_1-of-3">
        <c-account-card account={account}></c-account-card>
    </div>
</div>
```

### Loading Third-Party CSS

Upload CSS libraries as Static Resources and load via `platformResourceLoader`:

```javascript
import { loadStyle } from 'lightning/platformResourceLoader';
import customStyles from '@salesforce/resourceUrl/customStyles';

renderedCallback() {
    if (this._stylesLoaded) return;
    this._stylesLoaded = true;
    loadStyle(this, customStyles + '/styles.css');
}
```

Never load CSS from external CDNs. It violates CSP and creates security risks.

---

## Accessibility (a11y)

### Core Principles

1. All interactive elements must be keyboard accessible
2. All images and icons must have alternative text
3. All form fields must have associated labels
4. Color must not be the only means of conveying information
5. Focus management must be intentional

### Lightning Base Components Handle a11y

Prefer base components because they implement ARIA automatically:

```html
<!-- lightning-input includes label, aria, and error states -->
<lightning-input
    label="Account Name"
    value={accountName}
    required
    onchange={handleNameChange}>
</lightning-input>

<!-- lightning-button includes role and keyboard handling -->
<lightning-button
    label="Save"
    variant="brand"
    onclick={handleSave}>
</lightning-button>

<!-- lightning-icon includes assistive text -->
<lightning-icon
    icon-name="utility:warning"
    alternative-text="Warning"
    size="small">
</lightning-icon>
```

### ARIA Attributes for Custom Components

When building custom interactive elements, add ARIA attributes:

```html
<!-- Custom dropdown -->
<div role="listbox" aria-label="Select an account">
    <template for:each={options} for:item="option">
        <div
            key={option.id}
            role="option"
            aria-selected={option.isSelected}
            tabindex="0"
            onclick={handleOptionClick}
            onkeydown={handleOptionKeydown}
            data-id={option.id}>
            {option.label}
        </div>
    </template>
</div>
```

### Common ARIA Attributes

| Attribute | Use For | Example |
|-----------|---------|---------|
| `aria-label` | Non-visible label | `aria-label="Close dialog"` |
| `aria-describedby` | Link to description | `aria-describedby="help-text-1"` |
| `aria-expanded` | Collapsible sections | `aria-expanded={isOpen}` |
| `aria-hidden` | Decorative elements | `aria-hidden="true"` |
| `aria-live` | Dynamic updates | `aria-live="polite"` for status messages |
| `aria-busy` | Loading states | `aria-busy={isLoading}` |
| `role` | Semantic role | `role="alert"`, `role="dialog"` |

### Keyboard Navigation

```javascript
handleKeydown(event) {
    switch (event.key) {
        case 'Enter':
        case ' ':
            this.handleSelect(event);
            break;
        case 'ArrowDown':
            event.preventDefault();
            this.focusNextItem();
            break;
        case 'ArrowUp':
            event.preventDefault();
            this.focusPreviousItem();
            break;
        case 'Escape':
            this.handleClose();
            break;
        default:
            break;
    }
}
```

### Focus Management

```javascript
// Set focus programmatically after render
renderedCallback() {
    if (this._shouldFocusInput) {
        this._shouldFocusInput = false;
        const input = this.template.querySelector('lightning-input');
        if (input) {
            input.focus();
        }
    }
}

// delegatesFocus for composite components
static delegatesFocus = true;
```

Use `tabindex="0"` to make non-interactive elements focusable.
Use `tabindex="-1"` to make elements programmatically focusable but
not in the tab order. Never use values greater than 0.

### Status Messages for Screen Readers

```html
<div aria-live="polite" class="slds-assistive-text">
    {statusMessage}
</div>
```

Update `statusMessage` after async operations so screen readers announce
the result without requiring the user to navigate.

---

## Accessibility Checklist

- [ ] All form inputs have visible labels (or `aria-label`)
- [ ] All images and icons have `alternative-text`
- [ ] Custom interactive elements have `role`, `tabindex`, `aria-*`
- [ ] Keyboard navigation works for all interactive elements
- [ ] Focus is managed after modal open/close and navigation
- [ ] Color is not the sole indicator of state
- [ ] Loading states use `aria-busy` and/or `lightning-spinner` with alt text
- [ ] Dynamic content updates use `aria-live` regions
- [ ] Error messages are associated with fields via `aria-describedby`
