# Data Model Scanner

Detailed instructions for scanning custom objects, fields, relationships, platform events, custom metadata types, and custom labels.

## Discovery — Custom Objects

1. Glob for `**/objects/*/` across all package directories.
2. For each object folder, determine the object type:
   - Ends with `__c` → Custom Object
   - Ends with `__mdt` → Custom Metadata Type
   - Ends with `__e` → Platform Event
   - No suffix → Standard Object (with customizations)
3. Look for `{ObjectName}.object-meta.xml` in the folder for object-level metadata.

### Object-Level Metadata

Read the `*.object-meta.xml` file and extract:

| Field | XML Element | Notes |
|-------|------------|-------|
| Label | `<label>` | Display name |
| Plural label | `<pluralLabel>` | Plural display name |
| Description | `<description>` | Business purpose — may be empty |
| Name field | `<nameField>` | Auto Number format or Text |
| Sharing model | `<sharingModel>` | ReadWrite, Read, Private, etc. |
| Deployment status | `<deploymentStatus>` | Deployed / InDevelopment |
| Gender | `<gender>` | For gendered languages |
| Record types | Subfolder `recordTypes/` | Separate XML per record type |

### Record Types

Glob `**/objects/{Object}/recordTypes/*.recordType-meta.xml`:

| Field | XML Element |
|-------|------------|
| Name | `<fullName>` |
| Label | `<label>` |
| Description | `<description>` |
| Active | `<active>` |

---

## Discovery — Custom Fields

1. Glob for `**/objects/*/fields/*.field-meta.xml`.
2. For each field XML, extract:

| Field | XML Element | Example |
|-------|------------|---------|
| API name | `<fullName>` | `PaymentStatus__c` |
| Label | `<label>` | Payment Status |
| Type | `<type>` | Text, Checkbox, Lookup, MasterDetail, Picklist, Formula, Currency, etc. |
| Length | `<length>` | 255 |
| Required | `<required>` | true / false |
| Description | `<description>` | Business context |
| Help text | `<inlineHelpText>` | User-facing guidance |
| Default value | `<defaultValue>` | false, 'Pending', etc. |
| Formula | `<formula>` | For formula fields |
| Reference to | `<referenceTo>` | For Lookup / MasterDetail — the related object |
| Relationship name | `<relationshipName>` | For Lookup / MasterDetail |
| Relationship label | `<relationshipLabel>` | For Lookup / MasterDetail |
| Picklist values | `<valueSet><valueSetDefinition><value>` | For Picklist/MultiselectPicklist |
| Global value set | `<valueSet><valueSetName>` | References a global value set |
| External ID | `<externalId>` | true / false |
| Unique | `<unique>` | true / false |

### Relationship Extraction

For every field where `<type>` is `Lookup` or `MasterDetail`:

```
Source Object: the object folder this field lives in
Target Object: value of <referenceTo>
Relationship: Lookup or MasterDetail
Field: the field API name
```

Collect all relationships to build the relationship map.

---

## Discovery — Platform Events (__e)

1. From the object folders ending in `__e`, read the `.object-meta.xml`.
2. Fields are in `fields/` subfolder, same format as custom fields.
3. Note: Platform Events have no sharing model, no record types.

Extract:
- Event name
- All field names, types, and descriptions
- Description of the event itself

Cross-reference with Apex scanner: find classes that `EventBus.publish` this event and classes with triggers on the event.

---

## Discovery — Custom Metadata Types (__mdt)

1. From object folders ending in `__mdt`, read the `.object-meta.xml`.
2. Fields in `fields/` subfolder.
3. Also scan `**/customMetadata/*.md-meta.xml` for CMDT records.

For CMDT records:
- `<label>` — Record label
- `<values>` — Field values stored in the record

---

## Discovery — Custom Labels

1. Glob for `**/labels/CustomLabels.labels-meta.xml`.
2. This is a single XML file containing ALL labels. Parse each `<labels>` element:

| Field | XML Element |
|-------|------------|
| Full name | `<fullName>` |
| Value | `<value>` |
| Language | `<language>` |
| Category | `<categories>` |
| Short description | `<shortDescription>` |
| Protected | `<protected>` |

### Label Grouping

Group labels by prefix to identify categories:
- `Error*` → Error messages
- `Success*` → Success messages
- `UI*` → UI labels
- `Validation*` → Validation messages
- Other prefixes → custom categories

---

## Discovery — Global Value Sets

Glob for `**/globalValueSets/*.globalValueSet-meta.xml`. Extract:
- Name
- Values (sorted)
- Description

---

## Output Format for data-model.md

### Custom Objects

```markdown
## Custom Objects ({count})

| Object | Label | Fields | Record Types | Sharing | Description |
|--------|-------|--------|-------------|---------|-------------|
| `CustomerAsset__c` | Customer Asset | 12 | 2 | Private | Tracks assets assigned to customers |
| `Payment__c` | Payment | 8 | 0 | ReadWrite | Payment transactions |
```

### Standard Objects with Customizations

```markdown
## Standard Object Customizations

| Object | Custom Fields | Record Types | Notes |
|--------|--------------|-------------|-------|
| Account | 5 | 2 | Extended with industry-specific fields |
| Contact | 3 | 0 | Added loyalty tracking fields |
```

### Object Detail Sections

For each custom object, create a detailed subsection:

```markdown
### CustomerAsset__c

**Label:** Customer Asset
**Description:** Tracks assets assigned to customers for warranty and service purposes.
**Sharing:** Private
**Name Field:** Auto Number (CA-{00000})

#### Fields

| Field | Label | Type | Required | Description |
|-------|-------|------|----------|-------------|
| `SerialNumber__c` | Serial Number | Text(50) | Yes | Manufacturer serial number |
| `PurchaseDate__c` | Purchase Date | Date | No | Date customer acquired the asset |
| `Account__c` | Account | MasterDetail(Account) | Yes | Owning customer account |
| `WarrantyExpiry__c` | Warranty Expiry | Formula(Date) | — | PurchaseDate + 365 days |
| `IsUnderWarranty__c` | Under Warranty | Formula(Checkbox) | — | True if warranty not expired |

#### Record Types

| Record Type | Label | Active | Description |
|-------------|-------|--------|-------------|
| `Hardware` | Hardware | Yes | Physical hardware assets |
| `Software` | Software License | Yes | Software license assets |
```

### Relationship Map

Build an ASCII relationship diagram. Group objects that are related:

```markdown
## Relationship Map

Account (standard)
├── [MD] CustomerAsset__c.Account__c
│   └── [LK] ServiceRequest__c.Asset__c
├── [LK] Payment__c.Account__c
│   └── [LK] Payment__c.Invoice__c → Invoice__c
└── [LK] Contact (standard)

Legend: [MD] = Master-Detail, [LK] = Lookup
```

For complex models, organize as a table:

```markdown
| Source Object | Field | Relationship | Target Object |
|---------------|-------|-------------|---------------|
| `CustomerAsset__c` | `Account__c` | Master-Detail | Account |
| `ServiceRequest__c` | `Asset__c` | Lookup | `CustomerAsset__c` |
| `Payment__c` | `Account__c` | Lookup | Account |
| `Payment__c` | `Invoice__c` | Lookup | `Invoice__c` |
```

### Platform Events

```markdown
## Platform Events

### LogEvent__e

**Description:** Async transport for structured application logs.
**Published by:** Logger.cls
**Subscribed by:** LogEventSubscriber.trigger

| Field | Type | Description |
|-------|------|-------------|
| `Level__c` | Text(10) | ERROR, WARN, INFO |
| `SourceClass__c` | Text(100) | Originating class name |
| `Message__c` | LongTextArea | Log message content |
```

### Custom Metadata Types

```markdown
## Custom Metadata Types

### TriggerSetting__mdt

**Description:** Admin-managed trigger on/off per object.
**Records:** {count} records found

| Field | Type | Description |
|-------|------|-------------|
| `IsActive__c` | Checkbox | Whether the trigger is active |
| `Description__c` | Text(255) | Purpose of the trigger setting |
```

### Custom Labels

```markdown
## Custom Labels ({total count})

| Category | Count | Examples |
|----------|-------|---------|
| Error | 12 | `ErrorOrderNotFound`, `ErrorPaymentFailed` |
| UI | 8 | `UIButtonSubmit`, `UIHeaderDashboard` |
| Validation | 5 | `ValidationPhoneRequired` |
| Uncategorized | 3 | `DefaultCurrency`, `CompanyName` |
```
