# Error Handling

## Error Types in LWC

| Source | Error Shape | Access |
|--------|------------|--------|
| Apex imperative | `{ body: { message } }` or `{ body: [{ message }] }` | `catch (error)` |
| Wire adapter | `{ body: { message } }` | `wired({ error })` |
| LDS (getRecord, etc.) | `{ body: { message, statusCode } }` | Wire error property |
| JavaScript runtime | `Error` object | `try/catch` |
| Network failure | `{ body: { message: 'network error' } }` | `catch (error)` |

## reduceErrors Utility

A centralized function to normalize all error shapes into user-friendly strings:

```javascript
// c/utils
const reduceErrors = (errors) => {
    if (!Array.isArray(errors)) {
        errors = [errors];
    }
    return errors
        .filter((error) => !!error)
        .map((error) => {
            if (Array.isArray(error.body)) {
                return error.body.map((e) => e.message);
            }
            if (error.body && typeof error.body.message === 'string') {
                return error.body.message;
            }
            if (typeof error.message === 'string') {
                return error.message;
            }
            return String(error);
        })
        .flat();
};

export { reduceErrors };
```

Use in every component that handles errors:

```javascript
import { reduceErrors } from 'c/utils';

catch (error) {
    this.errorMessages = reduceErrors(error);
}
```

---

## Toast Notifications

```javascript
import { ShowToastEvent } from 'lightning/platformShowToastEvent';

showToast(title, message, variant) {
    this.dispatchEvent(new ShowToastEvent({
        title,
        message,
        variant  // 'success', 'error', 'warning', 'info'
    }));
}

// Usage
async handleSave() {
    try {
        await saveRecord({ data: this.formData });
        this.showToast('Success', 'Record saved.', 'success');
    } catch (error) {
        this.showToast('Error', reduceErrors(error).join(', '), 'error');
    }
}
```

### Toast Best Practices

| Do | Do Not |
|----|--------|
| Use `error` variant for failures | Show stack traces to users |
| Use `success` variant for confirmations | Show toasts for trivial actions |
| Keep messages under 2 sentences | Use technical jargon |
| Include actionable next steps when relevant | Show raw Apex exception messages |

---

## Error Boundary Component

Use `errorCallback` to catch errors from descendant components and
display a fallback UI instead of a broken component.

### Error Boundary Wrapper

```javascript
// c/errorBoundary
import { LightningElement } from 'lwc';

export default class ErrorBoundary extends LightningElement {
    hasError = false;
    errorMessage;
    errorStack;

    errorCallback(error, stack) {
        this.hasError = true;
        this.errorMessage = error.message;
        this.errorStack = stack;
    }

    handleRetry() {
        this.hasError = false;
        this.errorMessage = undefined;
        this.errorStack = undefined;
    }
}
```

```html
<!-- c/errorBoundary -->
<template>
    <template lwc:if={hasError}>
        <div class="slds-box slds-theme_error slds-m-around_medium">
            <p class="slds-text-heading_small">Something went wrong</p>
            <p class="slds-m-top_small">{errorMessage}</p>
            <lightning-button
                label="Try Again"
                variant="neutral"
                onclick={handleRetry}
                class="slds-m-top_small">
            </lightning-button>
        </div>
    </template>
    <template lwc:else>
        <slot></slot>
    </template>
</template>
```

### Usage

Wrap any component that might fail:

```html
<c-error-boundary>
    <c-risky-component record-id={recordId}></c-risky-component>
</c-error-boundary>
```

### errorCallback Limitations

`errorCallback` captures errors from:
- Lifecycle hooks (`connectedCallback`, `renderedCallback`, etc.)
- Event handlers declared in HTML templates

It does NOT capture errors from:
- Programmatic `addEventListener` handlers
- `setTimeout` / `setInterval` callbacks
- Promise rejections (use try/catch)

For those cases, wrap in try/catch and handle manually.

---

## Wire Error Handling

```javascript
@wire(getAccounts, { industry: '$selectedIndustry' })
wiredAccounts({ data, error }) {
    if (data) {
        this.accounts = data;
        this.error = undefined;
    } else if (error) {
        this.accounts = [];
        this.error = reduceErrors(error).join(', ');
    }
}
```

Always handle BOTH data and error branches. Reset the opposite property
to avoid showing stale data alongside a new error (or vice versa).

---

## Imperative Call Error Handling

```javascript
async handleSubmit() {
    this.isLoading = true;
    this.error = undefined;

    try {
        const result = await processOrder({ orderDto: this.formData });

        if (result.isSuccess) {
            this.showToast('Success', result.message, 'success');
            this.dispatchEvent(new CustomEvent('ordercreated', {
                detail: { orderId: result.data }
            }));
        } else {
            this.showToast('Warning', result.message, 'warning');
        }
    } catch (error) {
        this.error = reduceErrors(error).join(', ');
        this.showToast('Error', this.error, 'error');
    } finally {
        this.isLoading = false;
    }
}
```

### Handling ServiceResponseDto from Apex

When the Apex controller returns a `ServiceResponseDto` (see apex-architecture
skill), the LWC must check `isSuccess` before treating the result as a success:

```javascript
const result = await processPayment({ paymentDto: this.payment });

if (result.isSuccess) {
    this.showToast('Success', result.message, 'success');
} else {
    this.showToast('Error', result.message, 'error');
    this.validationErrors = result.errors || [];
}
```

---

## Inline Error Display

For form-level errors that should persist on screen (not just toast):

```html
<template lwc:if={hasErrors}>
    <div class="slds-notify slds-notify_alert slds-alert_error" role="alert">
        <h2>
            <lightning-icon icon-name="utility:error" size="x-small"
                alternative-text="Error" class="slds-m-right_x-small">
            </lightning-icon>
            Please fix the following errors:
        </h2>
        <ul class="slds-list_dotted slds-m-top_small">
            <template for:each={errorMessages} for:item="msg">
                <li key={msg}>{msg}</li>
            </template>
        </ul>
    </div>
</template>
```

---

## Error Handling Checklist

- [ ] Every imperative Apex call is wrapped in try/catch/finally
- [ ] Every wire handler checks both `data` and `error` branches
- [ ] `reduceErrors` utility used for consistent error extraction
- [ ] Toast messages use appropriate variant and user-friendly language
- [ ] Error boundary wraps components that may fail
- [ ] Form errors displayed inline when relevant (not just toast)
- [ ] `isLoading` set to `false` in `finally` block (never skipped on error)
- [ ] `ServiceResponseDto.isSuccess` checked before treating as success
