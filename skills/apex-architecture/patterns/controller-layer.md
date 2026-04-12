# Controller Layer Pattern (LWC Apex Controllers)

Thin translation layer between LWC and the Service Layer. No business logic here.

## Core Principles

- **Thin** — controllers only translate, validate input, and delegate to Services
- **`with sharing`** — always enforce record-level security
- **`@AuraEnabled`** — every method exposed to LWC must have this annotation
- **Static methods only** — LWC can only call `public static` methods
- **Return DTOs** — never expose raw SObjects; use wrappers with `@AuraEnabled` properties
- **Catch and wrap** — translate service exceptions to `AuraHandledException`

## Basic Controller

```apex
public with sharing class AccountController {

    @AuraEnabled(cacheable=true)
    public static List<AccountDto> getAccountsByIndustry(String industry) {
        List<Account> accounts = AccountSelector.selectByIndustry(industry, 50);
        List<AccountDto> result = new List<AccountDto>();

        for (Account acc : accounts) {
            result.add(new AccountDto(acc));
        }

        return result;
    }

    @AuraEnabled
    public static ServiceResponseDto reassignAccounts(List<Id> accountIds, Id newOwnerId) {
        try {
            AccountService.reassignToOwner(new Set<Id>(accountIds), newOwnerId);
            return ServiceResponseDto.success('Accounts reassigned successfully');
        } catch (AppException e) {
            throw new AuraHandledException(e.getMessage());
        } catch (Exception e) {
            throw new AuraHandledException(
                'An unexpected error occurred. Please contact your administrator.'
            );
        }
    }

    @AuraEnabled
    public static ServiceResponseDto updateAccountDetails(String accountJson) {
        try {
            AccountDto dto = (AccountDto) JSON.deserialize(accountJson, AccountDto.class);
            AccountService.updateDetails(dto);
            return ServiceResponseDto.success('Account updated');
        } catch (JSONException e) {
            throw new AuraHandledException('Invalid data format.');
        } catch (AppException e) {
            throw new AuraHandledException(e.getMessage());
        }
    }
}
```

## @AuraEnabled: cacheable vs non-cacheable

| Annotation | Wire-Compatible | DML Allowed | Use When |
|-----------|----------------|-------------|----------|
| `@AuraEnabled(cacheable=true)` | Yes | No | Read-only queries, data doesn't change often |
| `@AuraEnabled` | No (imperative only) | Yes | Write operations, data that must be fresh |

**Rule**: If the method does any DML, create, update, or delete, it **cannot** be `cacheable=true`.

## Wire vs Imperative — Decision Table

| Scenario | Approach | Why |
|----------|----------|-----|
| Load data on component init, rarely changes | `@wire` + `cacheable=true` | Auto-refresh, framework-managed cache |
| Load data on component init, always fresh | Imperative in `connectedCallback` | No caching, fresh every time |
| User clicks a button to fetch data | Imperative (`async/await`) | Triggered by user action |
| Any write operation (create/update/delete) | Imperative (`async/await`) | DML not allowed in wire |
| Data depends on user input (search, filter) | `@wire` with reactive params | Re-fires when params change |

## Wire Pattern (Read-Only, Cached)

Controller:
```apex
@AuraEnabled(cacheable=true)
public static List<ContactDto> getContactsByAccount(Id accountId) {
    List<Contact> contacts = ContactSelector.selectByAccountId(
        new Set<Id>{ accountId }
    );
    List<ContactDto> result = new List<ContactDto>();
    for (Contact c : contacts) {
        result.add(new ContactDto(c));
    }
    return result;
}
```

LWC JavaScript:
```javascript
import getContactsByAccount from '@salesforce/apex/ContactController.getContactsByAccount';

export default class ContactList extends LightningElement {
    @api recordId;
    contacts;
    error;

    @wire(getContactsByAccount, { accountId: '$recordId' })
    wiredContacts({ data, error }) {
        if (data) {
            this.contacts = data;
            this.error = undefined;
        } else if (error) {
            this.error = error.body.message;
            this.contacts = undefined;
        }
    }
}
```

## Imperative Pattern (Write Operations)

Controller:
```apex
@AuraEnabled
public static ServiceResponseDto createContact(String contactJson) {
    try {
        ContactDto dto = (ContactDto) JSON.deserialize(contactJson, ContactDto.class);
        ContactService.createFromDto(dto);
        return ServiceResponseDto.success('Contact created');
    } catch (AppException e) {
        throw new AuraHandledException(e.getMessage());
    } catch (Exception e) {
        throw new AuraHandledException('Failed to create contact.');
    }
}
```

LWC JavaScript:
```javascript
import createContact from '@salesforce/apex/ContactController.createContact';
import { ShowToastEvent } from 'lightning/platformShowToastEvent';

async handleSave() {
    try {
        const result = await createContact({
            contactJson: JSON.stringify(this.contactData)
        });
        this.dispatchEvent(new ShowToastEvent({
            title: 'Success',
            message: result.message,
            variant: 'success'
        }));
    } catch (error) {
        this.dispatchEvent(new ShowToastEvent({
            title: 'Error',
            message: error.body.message,
            variant: 'error'
        }));
    }
}
```

## Refreshing Wire Cache After DML

After a write operation, refresh wired data to reflect changes:

```javascript
import { refreshApex } from '@salesforce/apex';

wiredContactsResult; // store the full wire result

@wire(getContactsByAccount, { accountId: '$recordId' })
wiredContacts(result) {
    this.wiredContactsResult = result;
    if (result.data) {
        this.contacts = result.data;
    } else if (result.error) {
        this.error = result.error.body.message;
    }
}

async handleSave() {
    try {
        await createContact({ contactJson: JSON.stringify(this.contactData) });
        await refreshApex(this.wiredContactsResult);
    } catch (error) {
        // handle error
    }
}
```

## DTO Pattern for Controller Responses

```apex
public class ContactDto {
    @AuraEnabled public Id contactId;
    @AuraEnabled public String fullName;
    @AuraEnabled public String email;
    @AuraEnabled public String accountName;

    public ContactDto(Contact c) {
        this.contactId = c.Id;
        this.fullName = c.FirstName + ' ' + c.LastName;
        this.email = c.Email;
        this.accountName = c.Account?.Name;
    }

    public ContactDto() {}
}
```

Why DTOs instead of raw SObjects:
- Expose **only** the fields LWC needs
- **Shape data** for the UI (computed fields, flattened relationships)
- **Decouple** UI from data model — schema changes don't break the component
- **Security** — no accidental field exposure

## Controller for Datatable With Pagination

```apex
public with sharing class AccountListController {

    @AuraEnabled(cacheable=true)
    public static PageResultDto getAccounts(Integer pageSize, Integer pageNumber) {
        Integer totalRecords = AccountSelector.countAll();
        List<Account> accounts = AccountSelector.selectPaginated(pageSize, pageNumber);

        List<AccountDto> dtos = new List<AccountDto>();
        for (Account acc : accounts) {
            dtos.add(new AccountDto(acc));
        }

        return new PageResultDto(dtos, totalRecords, pageNumber, pageSize);
    }
}
```

Selector methods for pagination (in `AccountSelector`):

```apex
public static Integer countAll() {
    return [SELECT COUNT() FROM Account WITH SECURITY_ENFORCED];
}

public static List<Account> selectPaginated(Integer pageSize, Integer pageNumber) {
    Integer offset = (pageNumber - 1) * pageSize;
    return [
        SELECT Id, Name, Industry, AnnualRevenue
        FROM Account
        WITH SECURITY_ENFORCED
        ORDER BY Name
        LIMIT :pageSize
        OFFSET :offset
    ];
}
```

`PageResultDto` — see [dto-wrapper.md](dto-wrapper.md#pagination-dto) for full implementation.

## Error Handling Flow

```
LWC calls @AuraEnabled method
  └─► Controller receives params
       ├─► Validates/transforms input
       ├─► Delegates to Service
       │    ├─► Service succeeds → return DTO
       │    └─► Service throws AppException
       │         └─► Controller catches → throw AuraHandledException
       └─► Unexpected exception
            └─► Controller catches → throw AuraHandledException (generic message)

LWC receives:
  ├─► Success → process result
  └─► Error → error.body.message contains the user-friendly string
```

## Key Rules

- Controllers are **entry points**, not logic holders — always delegate to Services
- **One controller per feature/page**, not per SObject
- Always return **DTOs**, never raw SObjects
- `cacheable=true` only for **read-only** methods
- Wrap **all** exceptions in `AuraHandledException` before they reach LWC
- Never expose **internal error details** (stack traces, field API names) to the UI
- Use `ServiceResponseDto` for consistent response structure across controllers
- Parameter deserialization: accept `String` + `JSON.deserialize` for complex objects
