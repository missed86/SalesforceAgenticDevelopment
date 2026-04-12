# Jest Testing

## Setup

LWC tests use `@salesforce/sfdx-lwc-jest`. Tests live in a `__tests__`
folder inside the component directory:

```
accountCard/
  accountCard.html
  accountCard.js
  accountCard.js-meta.xml
  __tests__/
    accountCard.test.js
```

### Test File Naming

| Convention | Example |
|-----------|---------|
| Match component name | `accountCard.test.js` |
| Suffix `.test.js` | Not `.spec.js` (SF convention) |
| One test file per component | Split into `describe` blocks for sections |

---

## Test Structure

Follow Arrange-Act-Assert:

```javascript
import { createElement } from 'lwc';
import AccountCard from 'c/accountCard';

describe('c-account-card', () => {
    afterEach(() => {
        while (document.body.firstChild) {
            document.body.removeChild(document.body.firstChild);
        }
    });

    it('displays account name when provided', () => {
        // Arrange
        const element = createElement('c-account-card', { is: AccountCard });
        element.accountName = 'Acme Corp';

        // Act
        document.body.appendChild(element);

        // Assert
        const nameEl = element.shadowRoot.querySelector('.account-name');
        expect(nameEl.textContent).toBe('Acme Corp');
    });
});
```

### Critical: Always Clean Up

The `afterEach` block must remove all created elements to prevent test
pollution. Every test file needs this cleanup.

---

## DOM Assertions

### Querying Shadow DOM

```javascript
// Single element
const button = element.shadowRoot.querySelector('lightning-button');

// Multiple elements
const items = element.shadowRoot.querySelectorAll('c-list-item');
expect(items.length).toBe(5);

// By data attribute
const row = element.shadowRoot.querySelector('[data-id="001"]');
```

### Waiting for Async Rendering

After changing a reactive property, the DOM updates asynchronously.
Use `Promise.resolve()` to flush the microtask queue:

```javascript
it('shows spinner when loading', async () => {
    const element = createElement('c-account-card', { is: AccountCard });
    document.body.appendChild(element);

    // Change reactive property
    element.isLoading = true;

    // Wait for re-render
    await Promise.resolve();

    // Assert
    const spinner = element.shadowRoot.querySelector('lightning-spinner');
    expect(spinner).not.toBeNull();
});
```

For multiple sequential state changes, chain `await Promise.resolve()` calls
or use a helper:

```javascript
const flushPromises = () => new Promise(process.nextTick);

it('loads data then renders list', async () => {
    const element = createElement('c-account-card', { is: AccountCard });
    document.body.appendChild(element);

    await flushPromises();

    const items = element.shadowRoot.querySelectorAll('c-list-item');
    expect(items.length).toBeGreaterThan(0);
});
```

---

## Mocking Wire Adapters

### Mock Apex Wire

```javascript
import { createElement } from 'lwc';
import AccountList from 'c/accountList';
import getAccounts from '@salesforce/apex/AccountController.getAccountsByIndustry';

// Register the Apex wire adapter mock
jest.mock(
    '@salesforce/apex/AccountController.getAccountsByIndustry',
    () => {
        const { createApexTestWireAdapter } = require('@salesforce/sfdx-lwc-jest');
        return { default: createApexTestWireAdapter(jest.fn()) };
    },
    { virtual: true }
);

const MOCK_ACCOUNTS = [
    { Id: '001xx000003DGXAA', Name: 'Acme Corp', Industry: 'Technology' },
    { Id: '001xx000003DGXAB', Name: 'Global Inc', Industry: 'Technology' }
];

describe('c-account-list', () => {
    afterEach(() => {
        while (document.body.firstChild) {
            document.body.removeChild(document.body.firstChild);
        }
    });

    it('renders accounts when wire returns data', async () => {
        const element = createElement('c-account-list', { is: AccountList });
        document.body.appendChild(element);

        // Emit mock data
        getAccounts.emit(MOCK_ACCOUNTS);

        await Promise.resolve();

        const rows = element.shadowRoot.querySelectorAll('c-account-row');
        expect(rows.length).toBe(2);
    });

    it('shows error when wire fails', async () => {
        const element = createElement('c-account-list', { is: AccountList });
        document.body.appendChild(element);

        // Emit error
        getAccounts.error({ body: { message: 'Server error' } });

        await Promise.resolve();

        const errorPanel = element.shadowRoot.querySelector('c-error-panel');
        expect(errorPanel).not.toBeNull();
    });
});
```

### Mock LDS Wire (getRecord)

```javascript
import { createElement } from 'lwc';
import AccountDetail from 'c/accountDetail';
import { getRecord } from 'lightning/uiRecordApi';

const MOCK_RECORD = {
    fields: {
        Name: { value: 'Acme Corp' },
        Industry: { value: 'Technology' }
    }
};

describe('c-account-detail', () => {
    afterEach(() => {
        while (document.body.firstChild) {
            document.body.removeChild(document.body.firstChild);
        }
    });

    it('displays account name from getRecord', async () => {
        const element = createElement('c-account-detail', {
            is: AccountDetail
        });
        document.body.appendChild(element);

        getRecord.emit(MOCK_RECORD);

        await Promise.resolve();

        const name = element.shadowRoot.querySelector('.account-name');
        expect(name.textContent).toBe('Acme Corp');
    });
});
```

`sfdx-lwc-jest` automatically registers LDS wire adapters. No manual
`jest.mock` is needed for `getRecord`, `getFieldValue`, etc.

---

## Mocking Imperative Apex

```javascript
import saveAccount from '@salesforce/apex/AccountController.saveAccount';

jest.mock(
    '@salesforce/apex/AccountController.saveAccount',
    () => ({ default: jest.fn() }),
    { virtual: true }
);

it('calls save and shows success toast', async () => {
    saveAccount.mockResolvedValue({
        isSuccess: true,
        message: 'Account saved'
    });

    const element = createElement('c-account-form', { is: AccountForm });
    document.body.appendChild(element);

    // Simulate form fill and submit
    const nameInput = element.shadowRoot.querySelector('[data-field="name"]');
    nameInput.value = 'New Account';
    nameInput.dispatchEvent(new CustomEvent('change'));

    const saveBtn = element.shadowRoot.querySelector('[data-action="save"]');
    saveBtn.click();

    await Promise.resolve();

    expect(saveAccount).toHaveBeenCalledWith({
        accountDto: expect.objectContaining({ name: 'New Account' })
    });
});
```

---

## Testing Events

### Testing Event Dispatch

```javascript
it('fires select event with record ID', async () => {
    const element = createElement('c-account-row', { is: AccountRow });
    element.account = { Id: '001xx000003DGX', Name: 'Acme' };
    document.body.appendChild(element);

    const handler = jest.fn();
    element.addEventListener('select', handler);

    const row = element.shadowRoot.querySelector('.account-row');
    row.click();

    expect(handler).toHaveBeenCalledTimes(1);
    expect(handler.mock.calls[0][0].detail).toEqual({
        recordId: '001xx000003DGX'
    });
});
```

### Testing Event Handling

```javascript
it('updates selected ID when child fires select', async () => {
    const element = createElement('c-account-dashboard', {
        is: AccountDashboard
    });
    document.body.appendChild(element);

    // Simulate child event
    const list = element.shadowRoot.querySelector('c-account-list');
    list.dispatchEvent(new CustomEvent('select', {
        detail: { recordId: '001xx000003DGX' }
    }));

    await Promise.resolve();

    const detail = element.shadowRoot.querySelector('c-account-detail');
    expect(detail.recordId).toBe('001xx000003DGX');
});
```

---

## Mocking Modules

### Mock Navigation

```javascript
import { NavigationMixin } from 'lightning/navigation';

const NAVIGATE_SPY = jest.fn();

jest.mock('lightning/navigation', () => ({
    NavigationMixin: (Base) => {
        return class extends Base {
            [Symbol.for('NavigationMixin.Navigate')] = NAVIGATE_SPY;
        };
    }
}), { virtual: true });

it('navigates to record on click', async () => {
    // ... click action ...
    expect(NAVIGATE_SPY).toHaveBeenCalledWith(
        expect.objectContaining({
            type: 'standard__recordPage',
            attributes: expect.objectContaining({
                recordId: '001xx000003DGX',
                actionName: 'view'
            })
        })
    );
});
```

### Mock Toast

```javascript
it('shows success toast on save', async () => {
    const handler = jest.fn();
    element.addEventListener('lightning__showtoast', handler);

    // ... trigger save ...

    expect(handler).toHaveBeenCalled();
    const toast = handler.mock.calls[0][0];
    expect(toast.detail.variant).toBe('success');
    expect(toast.detail.title).toBe('Success');
});
```

Use the string `'lightning__showtoast'` directly. Do not rely on
`ShowToastEvent.type` as it may not exist in the Jest environment.

### Mock Custom Labels and Permissions

```javascript
jest.mock('@salesforce/label/c.ErrorOrderNotFound',
    () => ({ default: 'Order not found' }),
    { virtual: true }
);

jest.mock('@salesforce/userPermission/CanApproveOrders',
    () => ({ default: true }),
    { virtual: true }
);
```

---

## Test Categories

| Category | What to Test | Example |
|----------|-------------|---------|
| Rendering | Correct DOM output for given props | Name displays in header |
| Events | Component dispatches correct events | Select fires with ID |
| Wire data | Component handles wire success/error | List renders from mock data |
| Imperative | Apex called with correct params | Save called with DTO |
| User interaction | Click, input, keyboard events | Button click triggers save |
| Edge cases | Empty data, null props, error states | No accounts shows empty message |
| Accessibility | ARIA attributes present | `aria-label` set on custom control |

---

## Testing Checklist

- [ ] `afterEach` cleans up all created elements
- [ ] Wire success AND error paths tested
- [ ] Imperative calls mock both resolve and reject
- [ ] Events tested for dispatch and handling
- [ ] Loading, error, and content states all tested
- [ ] Edge cases: null, undefined, empty arrays
- [ ] `await Promise.resolve()` after every state change
- [ ] No real Apex calls (all mocked with `jest.mock`)
