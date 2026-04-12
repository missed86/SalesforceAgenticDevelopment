# Security Patterns

Apex runs in system context by default. You must explicitly enforce security.

## Sharing Keywords

```apex
// DEFAULT: Enforce record-level security
public with sharing class AccountService { }

// Utility/helper classes: inherit caller's context
public inherited sharing class QueryHelper { }

// Admin operations ONLY: bypass sharing (isolate and minimize)
public without sharing class DataMigrationService { }
```

### Sharing Decision

| Class Type | Keyword | Reason |
|-----------|---------|--------|
| Service classes | `with sharing` | User-facing operations respect sharing rules |
| Utility/helper | `inherited sharing` | Flexible — inherits caller's context |
| Trigger handler | `with sharing` | Default safe, unless admin-level operations |
| Selector | `inherited sharing` | Adapts to caller's security context |
| Admin tool | `without sharing` | Isolated, documented, minimal scope |
| Batch Apex | `with sharing` or `without sharing` | Depends on use case — document the choice |

## CRUD/FLS Enforcement in SOQL

### WITH SECURITY_ENFORCED (Recommended for most queries)

Throws `System.QueryException` if user lacks access to any queried field/object:

```apex
List<Account> accounts = [
    SELECT Id, Name, AnnualRevenue
    FROM Account
    WHERE Industry = :industry
    WITH SECURITY_ENFORCED
];
```

### WITH USER_MODE (Spring '23+, more flexible)

Silently omits inaccessible fields instead of throwing:

```apex
List<Account> accounts = [
    SELECT Id, Name, AnnualRevenue, SecretField__c
    FROM Account
    WHERE Industry = :industry
    WITH USER_MODE
];
// SecretField__c will be null if user lacks FLS, no exception thrown
```

### Decision: SECURITY_ENFORCED vs USER_MODE

| Scenario | Use |
|----------|-----|
| Strict enforcement, fail fast | `WITH SECURITY_ENFORCED` |
| Graceful degradation, optional fields | `WITH USER_MODE` |
| Internal/system queries (batch, async) | Neither — system context is fine |

## Security.stripInaccessible()

Removes fields the user cannot access from SObject records:

```apex
public with sharing class AccountService {

    public static List<Account> getAccountsSanitized() {
        List<Account> accounts = [
            SELECT Id, Name, AnnualRevenue, Phone, Industry
            FROM Account
            LIMIT 50
        ];

        SObjectAccessDecision decision = Security.stripInaccessible(
            AccessType.READABLE, accounts
        );

        return (List<Account>) decision.getRecords();
    }
}

// In controllers: convert sanitized SObjects to DTOs before returning to LWC
// List<Account> safe = AccountService.getAccountsSanitized();
// Loop over safe, build List<AccountDto>, return (see controller-layer.md)
```

Use cases for `stripInaccessible`:

| AccessType | Use When |
|-----------|----------|
| `READABLE` | Returning data to UI or API |
| `CREATABLE` | Before insert — strips non-creatable fields |
| `UPDATABLE` | Before update — strips non-updatable fields |
| `UPSERTABLE` | Before upsert |

## SOQL Injection Prevention

### ALWAYS use bind variables:

```apex
// SAFE — bind variable
String industry = userInput;
List<Account> accounts = [
    SELECT Id, Name FROM Account WHERE Industry = :industry
];
```

### NEVER concatenate user input:

```apex
// VULNERABLE — string concatenation
String query = 'SELECT Id, Name FROM Account WHERE Industry = \'' + userInput + '\'';
List<Account> accounts = Database.query(query);
```

### If dynamic SOQL is unavoidable:

```apex
String safeName = String.escapeSingleQuotes(userInput);
String query = 'SELECT Id, Name FROM Account WHERE Name = \'' + safeName + '\'';
```

Prefer `String.escapeSingleQuotes()` AND bind variables where possible.

## System.runAs() in Tests

Test security by running as restricted users:

```apex
@isTest
static void testRestrictedAccess() {
    User limitedUser = TestDataFactory.createStandardUser();

    System.runAs(limitedUser) {
        try {
            List<Account> accounts = AccountService.getAccountsSanitized();
            Assert.isTrue(accounts.size() >= 0, 'Query should succeed');
        } catch (System.QueryException e) {
            Assert.isTrue(
                e.getMessage().contains('SECURITY_ENFORCED'),
                'Should fail on FLS'
            );
        }
    }
}
```

## Key Rules

- Default to `with sharing` — opt out only when justified and documented
- Use `inherited sharing` for reusable utility classes
- Enforce FLS with `WITH SECURITY_ENFORCED` or `Security.stripInaccessible()`
- Never concatenate user input into SOQL — always use bind variables
- Test with restricted users using `System.runAs()`
- Document every `without sharing` with a comment explaining why
