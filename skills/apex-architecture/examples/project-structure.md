# Project Structure — SFDX Organization

Recommended folder organization for Salesforce DX projects at every scale.

## Critical: How SFDX Handles `classes/`

Salesforce DX treats `classes/` as a **flat metadata directory**. Key facts:

- `sf project retrieve start` deposits ALL classes flat in `classes/` — no subfolders
- `sf project deploy start` DOES deploy classes from subfolders within `classes/`
- If you use subfolders locally and another dev retrieves from the org, their classes land flat
- Subfolders inside `classes/` are a **local Git convention only** — the org has no concept of them

### Implications

| Approach | Pros | Cons |
|----------|------|------|
| **Flat `classes/`** | Works with all SF CLI commands, no surprises | Hard to navigate 50+ classes |
| **Subfolders in `classes/`** | Visual organization, clear architecture | Breaks on retrieve; team must enforce manually |
| **Multiple package directories** | SF CLI respects them, clean separation | More `sfdx-project.json` config |

**Recommendation by project size:**

| Size | Strategy |
|------|----------|
| Small (< 20 classes) | Flat `classes/` — rely on naming conventions |
| Medium (20-60 classes) | Flat `classes/` + strict naming prefixes, or subfolders with team agreement |
| Large (60+ classes) | Multiple package directories to separate domains |

## Small Project (< 20 Classes)

Flat structure. Naming conventions do all the organization work.

```
force-app/
└── main/
    └── default/
        ├── classes/
        │   ├── AccountTriggerHandler.cls
        │   ├── AccountTriggerHandler.cls-meta.xml
        │   ├── AccountService.cls
        │   ├── AccountService.cls-meta.xml
        │   ├── AccountServiceTest.cls
        │   ├── AccountServiceTest.cls-meta.xml
        │   ├── TestDataFactory.cls
        │   └── TestDataFactory.cls-meta.xml
        ├── triggers/
        │   ├── AccountTrigger.trigger
        │   └── AccountTrigger.trigger-meta.xml
        ├── lwc/
        │   └── accountManager/
        ├── objects/
        │   └── Account/
        └── customMetadata/
```

With good naming (see [naming-conventions.md](naming-conventions.md)), classes self-organize
alphabetically: all `Account*` classes group together, all `*Test` classes are visible, etc.

## Medium Project — Flat With Naming Prefixes

For 20-60 classes, stay flat but rely on consistent naming for grouping:

```
force-app/
└── main/
    └── default/
        ├── classes/
        │   ├── AccountController.cls
        │   ├── AccountControllerTest.cls
        │   ├── AccountDomain.cls
        │   ├── AccountDomainTest.cls
        │   ├── AccountDto.cls
        │   ├── AccountEnrichmentJob.cls
        │   ├── AccountEnrichmentJobTest.cls
        │   ├── AccountSelector.cls
        │   ├── AccountSelectorTest.cls
        │   ├── AccountService.cls
        │   ├── AccountServiceTest.cls
        │   ├── AccountTriggerHandler.cls
        │   ├── AccountTriggerHandlerTest.cls
        │   ├── AppConstants.cls
        │   ├── AppException.cls
        │   ├── CacheService.cls
        │   ├── FeatureToggle.cls
        │   ├── HttpCalloutService.cls
        │   ├── Logger.cls
        │   ├── OrderController.cls
        │   ├── OrderControllerTest.cls
        │   ├── OrderDomain.cls
        │   ├── OrderException.cls
        │   ├── OrderResultDto.cls
        │   ├── OrderSelector.cls
        │   ├── OrderService.cls
        │   ├── OrderServiceTest.cls
        │   ├── OrderTriggerHandler.cls
        │   ├── PageResultDto.cls
        │   ├── PaymentException.cls
        │   ├── PaymentGatewayService.cls
        │   ├── PaymentResultDto.cls
        │   ├── PaymentService.cls
        │   ├── PaymentServiceTest.cls
        │   ├── ServiceResponseDto.cls
        │   ├── StaleAccountCleanupBatch.cls
        │   ├── StaleAccountCleanupBatchTest.cls
        │   ├── TestDataFactory.cls
        │   ├── TriggerControl.cls
        │   └── WeeklyCleanupScheduler.cls
        ├── triggers/
        │   ├── AccountTrigger.trigger
        │   ├── OrderTrigger.trigger
        │   └── OpportunityTrigger.trigger
        ├── lwc/
        ├── objects/
        ├── customMetadata/
        └── labels/
```

Alphabetical sorting naturally groups by domain (`Account*`, `Order*`, `Payment*`).
Tests sit next to their production class for easy discovery.

## Medium Project — Subfolders (Optional, Git-Only)

If the team agrees to maintain subfolders manually, this works for local development.
**Be aware**: `sf project retrieve start` will NOT place classes into these subfolders.
After retrieving, you must manually move new classes to the correct subfolder.

```
force-app/
└── main/
    └── default/
        ├── classes/
        │   ├── handlers/
        │   │   └── AccountTriggerHandler.cls
        │   ├── services/
        │   │   └── AccountService.cls
        │   ├── selectors/
        │   │   └── AccountSelector.cls
        │   └── (etc.)
        └── triggers/
```

**Team requirement**: document the convention in the project README and use a
pre-commit hook or CI check to enforce folder placement.

## Large Project — Multiple Package Directories

For 60+ classes, use separate `sfdx-project.json` package directories.
This IS fully supported by SF CLI — each directory is an independent deployment unit.

```
core-app/                              # Shared infrastructure
└── main/
    └── default/
        ├── classes/
        │   ├── AppException.cls
        │   ├── IntegrationException.cls
        │   ├── CacheService.cls
        │   ├── FeatureToggle.cls
        │   ├── HttpCalloutService.cls
        │   ├── Logger.cls
        │   ├── OptionDto.cls
        │   ├── PageResultDto.cls
        │   ├── ServiceFactory.cls
        │   ├── ServiceResponseDto.cls
        │   └── TriggerControl.cls
        ├── customMetadata/
        └── objects/
            ├── ApplicationLog__c/
            └── TriggerSetting__mdt/

accounts-app/                          # Account domain
└── main/
    └── default/
        ├── classes/
        │   ├── AccountController.cls
        │   ├── AccountControllerTest.cls
        │   ├── AccountDomain.cls
        │   ├── AccountDomainTest.cls
        │   ├── AccountDto.cls
        │   ├── AccountSelector.cls
        │   ├── AccountSelectorTest.cls
        │   ├── AccountService.cls
        │   ├── AccountServiceTest.cls
        │   ├── AccountTriggerHandler.cls
        │   └── AccountTriggerHandlerTest.cls
        └── triggers/
            └── AccountTrigger.trigger

orders-app/                            # Order domain
└── main/
    └── default/
        ├── classes/
        │   ├── OrderController.cls
        │   ├── OrderControllerTest.cls
        │   ├── OrderDomain.cls
        │   ├── OrderDomainTest.cls
        │   ├── OrderException.cls
        │   ├── OrderResultDto.cls
        │   ├── OrderSelector.cls
        │   ├── OrderService.cls
        │   ├── OrderServiceTest.cls
        │   ├── OrderTriggerHandler.cls
        │   └── OrderTriggerHandlerTest.cls
        └── triggers/
            └── OrderTrigger.trigger

test-utils/                            # Shared test utilities
└── main/
    └── default/
        └── classes/
            ├── TestDataFactory.cls
            ├── PaymentGatewayMock.cls
            └── OrderRepoStub.cls
```

### sfdx-project.json

```json
{
  "packageDirectories": [
    {
      "path": "core-app",
      "default": true
    },
    {
      "path": "accounts-app"
    },
    {
      "path": "orders-app"
    },
    {
      "path": "test-utils"
    }
  ],
  "namespace": "",
  "sfdcLoginUrl": "https://login.salesforce.com",
  "sourceApiVersion": "63.0"
}
```

Benefits:
- SF CLI fully respects package directories — deploy/retrieve works correctly
- Clear domain boundaries — easier to reason about dependencies
- Can deploy domains independently
- Tests stay close to production code (same package) or in a shared `test-utils`

### Dependency Order

When deploying, `core-app` must deploy before domain apps (they depend on `AppException`, etc.).
Use the `dependencies` key in `sfdx-project.json` if using unlocked packages.

## Scaling Decision

| Size | Structure | Organization Strategy |
|------|-----------|----------------------|
| **Small** (< 20) | Single `force-app`, flat `classes/` | Naming conventions only |
| **Medium** (20-60) | Single `force-app`, flat `classes/` | Naming prefixes group by domain |
| **Large** (60+) | Multiple package directories | Domain-driven separation |

## Key Rules

- **Triggers always in `triggers/`** — never mix with classes
- **One trigger file per SObject** — handler logic in `classes/`
- **`classes/` is flat by default** — don't assume subfolders survive retrieve
- **Naming conventions are your primary organizer** — see [naming-conventions.md](naming-conventions.md)
- **Multiple package directories for true separation** — supported by SF CLI
- **Custom Metadata in `customMetadata/`** — deployable configuration
- **Upgrade structure as you grow** — start flat, split into packages when complexity demands it
