# Selector Layer Pattern

Centralized SOQL. One Selector per SObject. Every query lives here.

## Core Principles

- **One class per SObject** — `AccountSelector`, `ContactSelector`
- **Static methods** — no instance state needed
- **Returns typed Lists/Maps** — never raw query results
- **Single source of truth** for field lists — no scattered queries
- **Security enforced** via `WITH SECURITY_ENFORCED` or `WITH USER_MODE`

## Basic Selector

```apex
public inherited sharing class AccountSelector {

    public static List<Account> selectByIds(Set<Id> accountIds) {
        return [
            SELECT Id, Name, Industry, OwnerId, AnnualRevenue,
                   BillingCity, BillingCountry, CreatedDate
            FROM Account
            WHERE Id IN :accountIds
            WITH SECURITY_ENFORCED
        ];
    }

    public static List<Account> selectByOwnerId(Set<Id> ownerIds) {
        return [
            SELECT Id, Name, Industry, OwnerId, AnnualRevenue
            FROM Account
            WHERE OwnerId IN :ownerIds
            WITH SECURITY_ENFORCED
        ];
    }

    public static Map<Id, Account> selectMapByIds(Set<Id> accountIds) {
        return new Map<Id, Account>(selectByIds(accountIds));
    }

    public static List<Account> selectWithContacts(Set<Id> accountIds) {
        return [
            SELECT Id, Name, Industry,
                (SELECT Id, FirstName, LastName, Email
                 FROM Contacts
                 WHERE IsDeleted = false
                 ORDER BY LastName)
            FROM Account
            WHERE Id IN :accountIds
            WITH SECURITY_ENFORCED
        ];
    }

    public static List<Account> selectByIndustry(String industry, Integer recordLimit) {
        return [
            SELECT Id, Name, Industry, AnnualRevenue
            FROM Account
            WHERE Industry = :industry
            WITH SECURITY_ENFORCED
            ORDER BY AnnualRevenue DESC
            LIMIT :recordLimit
        ];
    }
}
```

## Selector With Dynamic SOQL (When Needed)

Use only when field sets or conditions are truly dynamic.

**Warning**: `String.escapeSingleQuotes()` protects string VALUES from injection,
but does NOT protect object/field NAMES. Validate identifiers against Schema:

```apex
public inherited sharing class GenericSelector {

    public static List<SObject> selectByFieldValues(
        String sObjectType,
        List<String> fields,
        String filterField,
        Set<String> filterValues
    ) {
        validateObjectName(sObjectType);
        validateFieldNames(sObjectType, fields);
        validateFieldNames(sObjectType, new List<String>{ filterField });

        String fieldList = String.join(fields, ', ');
        String query = 'SELECT ' + fieldList
            + ' FROM ' + sObjectType
            + ' WHERE ' + filterField
            + ' IN :filterValues'
            + ' WITH SECURITY_ENFORCED';

        return Database.query(query);
    }

    private static void validateObjectName(String sObjectType) {
        if (Schema.getGlobalDescribe().get(sObjectType) == null) {
            throw new AppException('Invalid SObject type: ' + sObjectType);
        }
    }

    private static void validateFieldNames(String sObjectType, List<String> fields) {
        Map<String, Schema.SObjectField> fieldMap =
            Schema.getGlobalDescribe().get(sObjectType).getDescribe().fields.getMap();
        for (String field : fields) {
            if (!fieldMap.containsKey(field.toLowerCase())) {
                throw new AppException('Invalid field: ' + field + ' on ' + sObjectType);
            }
        }
    }
}
```

Always use bind variables (`:paramName`) for VALUES, and Schema validation for identifiers.

## Selector Returning Maps for Bulk Lookups

```apex
public inherited sharing class ContactSelector {

    public static Map<Id, List<Contact>> selectGroupedByAccountId(Set<Id> accountIds) {
        Map<Id, List<Contact>> contactsByAccountId = new Map<Id, List<Contact>>();

        for (Contact c : [
            SELECT Id, FirstName, LastName, Email, AccountId
            FROM Contact
            WHERE AccountId IN :accountIds
            WITH SECURITY_ENFORCED
        ]) {
            if (!contactsByAccountId.containsKey(c.AccountId)) {
                contactsByAccountId.put(c.AccountId, new List<Contact>());
            }
            contactsByAccountId.get(c.AccountId).add(c);
        }

        return contactsByAccountId;
    }
}
```

## Apex Cursors (Spring '26) for Large Datasets

```apex
public inherited sharing class LargeDataSelector {

    public static Database.Cursor getCursorForStaleAccounts(Date cutoffDate) {
        return Database.getCursor(
            'SELECT Id, Name, LastActivityDate FROM Account WHERE LastActivityDate < :cutoffDate'
        );
    }

    public static List<Account> fetchChunk(Database.Cursor cursor, Integer chunkSize) {
        return (List<Account>) cursor.fetch(chunkSize);
    }
}
```

## Key Rules

- **Never inline SOQL** in handlers, services, or controllers — always go through a Selector
- **Use `WITH SECURITY_ENFORCED`** for user-facing queries
- **Use `inherited sharing`** so the Selector respects the caller's sharing context
- **Return typed collections** — `List<Account>`, not `List<SObject>`
- **Name methods descriptively** — `selectByOwnerId`, `selectWithContacts`, `selectActiveByStatus`
- **Keep field lists consistent** — avoids "field not queried" runtime errors
