---
name: sf-cartographer
description: >-
  Scans every force-app/ directory in a Salesforce DX project and generates
  (or updates) comprehensive living documentation: architecture overview,
  data model, Apex inventory, LWC catalog, automation map, security model,
  integration points, UI configuration, and naming-pattern detection. Use when
  someone says "document the project", "cartographer", "generate docs",
  "update docs", "documentar proyecto", "mapear proyecto", or when the team
  needs a fresh snapshot of what the codebase contains.
metadata:
  domain: salesforce-documentation
  salesforce-era: spring-26
  version: "1.0"
---

# SF Cartographer

Automated, living documentation for any Salesforce DX project.
One invocation scans every package directory, classifies every metadata type,
and produces a complete doc suite. Re-invoke to update — nothing is lost,
everything is refreshed.

## Design Philosophy

SF Cartographer is **fully standalone** — it does not depend on any other skill,
naming standard, or architecture pattern. It documents **what the project actually
does**, not what it should do. Every project has its own conventions, structure,
and architecture. The Cartographer observes and maps reality, never prescribes.

---

## Output Structure

All generated docs land in `docs/` at the project root (sibling of `force-app/`).

```
docs/
├── INDEX.md                 # Master index — entry point to all documentation
├── architecture.md          # Detected architecture patterns and layer diagram
├── data-model.md            # Custom objects, fields, relationships, events, CMDT
├── apex-inventory.md        # All Apex classes and triggers, categorized by layer
├── lwc-catalog.md           # All LWC components — props, events, targets, deps
├── automation-map.md        # Flows, validation rules, approval processes
├── security-model.md        # Profiles, permission sets, PSGs, sharing, queues
├── integrations.md          # Named/external credentials, remote sites, connected apps
├── ui-config.md             # Layouts, Lightning pages, tabs, apps, quick actions
└── project-structure.md     # Full directory tree with annotations
```

> If a metadata category has **zero items** (e.g. no Flows exist), **skip that section entirely** — do not create an empty file.

---

## Instructions for the Agent

### Phase 0 — Pre-flight

1. **Verify SFDX project**: confirm `sfdx-project.json` exists at the repo root. If not, stop and ask the user.
2. **Read `sfdx-project.json`**: parse `packageDirectories` to identify ALL source paths (not just `force-app`). Some projects have `core-app/`, `accounts-app/`, etc.
3. **Check existing docs**: if `docs/INDEX.md` already exists, you are in **update mode** — regenerate every file with fresh data. Preserve any `<!-- CUSTOM -->` … `<!-- /CUSTOM -->` blocks the user may have added manually.
4. **Create `docs/` folder** if it does not exist.

### Phase 1 — Discovery (Scan Everything)

For every package directory found in step 2, walk the tree and build an in-memory inventory. Use `Glob` and `Read` tools — never guess.

| Metadata Type | Where to Look | Scanner Reference |
|---------------|--------------|-------------------|
| Apex classes | `**/classes/**/*.cls` | [apex-scanner.md](scanners/apex-scanner.md) |
| Apex triggers | `**/triggers/**/*.trigger` | [apex-scanner.md](scanners/apex-scanner.md) |
| LWC components | `**/lwc/*/` | [lwc-scanner.md](scanners/lwc-scanner.md) |
| Aura components | `**/aura/*/` | [lwc-scanner.md](scanners/lwc-scanner.md) |
| Custom objects | `**/objects/*/` | [data-model-scanner.md](scanners/data-model-scanner.md) |
| Custom fields | `**/objects/*/fields/*.field-meta.xml` | [data-model-scanner.md](scanners/data-model-scanner.md) |
| Platform events | `**/objects/*__e/` | [data-model-scanner.md](scanners/data-model-scanner.md) |
| Custom metadata types | `**/objects/*__mdt/` | [data-model-scanner.md](scanners/data-model-scanner.md) |
| Custom labels | `**/labels/CustomLabels.labels-meta.xml` | [data-model-scanner.md](scanners/data-model-scanner.md) |
| Flows | `**/flows/*.flow-meta.xml` | [automation-scanner.md](scanners/automation-scanner.md) |
| Validation rules | `**/objects/*/validationRules/*` | [automation-scanner.md](scanners/automation-scanner.md) |
| Approval processes | `**/approvalProcesses/*` | [automation-scanner.md](scanners/automation-scanner.md) |
| Permission sets | `**/permissionsets/*.permissionset-meta.xml` | [security-scanner.md](scanners/security-scanner.md) |
| Permission set groups | `**/permissionsetgroups/*` | [security-scanner.md](scanners/security-scanner.md) |
| Profiles | `**/profiles/*.profile-meta.xml` | [security-scanner.md](scanners/security-scanner.md) |
| Sharing rules | `**/sharingRules/*` | [security-scanner.md](scanners/security-scanner.md) |
| Queues | `**/queues/*` | [security-scanner.md](scanners/security-scanner.md) |
| Public groups | `**/groups/*` | [security-scanner.md](scanners/security-scanner.md) |
| Named credentials | `**/namedCredentials/*` | [integration-scanner.md](scanners/integration-scanner.md) |
| External credentials | `**/externalCredentials/*` | [integration-scanner.md](scanners/integration-scanner.md) |
| Remote site settings | `**/remoteSiteSettings/*` | [integration-scanner.md](scanners/integration-scanner.md) |
| Connected apps | `**/connectedApps/*` | [integration-scanner.md](scanners/integration-scanner.md) |
| Page layouts | `**/layouts/*` | [ui-scanner.md](scanners/ui-scanner.md) |
| Lightning pages | `**/flexipages/*` | [ui-scanner.md](scanners/ui-scanner.md) |
| Tabs | `**/tabs/*` | [ui-scanner.md](scanners/ui-scanner.md) |
| Applications | `**/applications/*` | [ui-scanner.md](scanners/ui-scanner.md) |
| Quick actions | `**/quickActions/*` | [ui-scanner.md](scanners/ui-scanner.md) |
| Static resources | `**/staticresources/*` | [ui-scanner.md](scanners/ui-scanner.md) |
| Message channels | `**/messageChannels/*` | [ui-scanner.md](scanners/ui-scanner.md) |
| Custom permissions | `**/customPermissions/*` | [security-scanner.md](scanners/security-scanner.md) |

**Parallel scanning**: when possible, scan independent metadata types in parallel using multiple tool calls to speed up discovery.

### Phase 2 — Analysis

After discovery, perform cross-cutting analysis:

1. **Architecture detection**: based on Apex class suffixes (`*Service`, `*Selector`, `*Controller`, `*TriggerHandler`, `*Domain`, `*Dto`, `*Batch`, `*Scheduler`, `*RestService`), determine which architecture layers are used and draw an ASCII layer diagram.

2. **Relationship mapping**: from object XML, identify Master-Detail and Lookup fields to build a relationship map between objects.

3. **Naming pattern detection**: observe the actual naming conventions the project uses (e.g. do classes use suffixes like `*Service`? do Flows follow `Object_Type_Purpose`? do permission sets use a prefix?). Document the **observed patterns** — do not judge against any external standard.

4. **LWC dependency graph**: from LWC imports, identify which components use which Apex controllers, which components compose other components (`<c-*>` tags), and which LMS channels are used.

5. **Coverage analysis**: identify Apex classes that have no corresponding `*Test` class (by naming convention detected in the project).

6. **Package directory mapping**: if the project uses multiple packages, document which metadata lives in which package and any implied dependency order.

### Phase 3 — Generation

Generate each doc file following the templates below. Every file must:

- Start with a YAML-like header comment: `<!-- Generated by SF Cartographer | {ISO date} -->`
- Use tables for inventories (not bullet lists)
- Include counts (e.g. "**23 Apex classes** across 3 layers")
- Use relative links between docs (e.g. `[see data model](data-model.md)`)
- Never truncate — document everything found

#### 3.1 INDEX.md

```markdown
<!-- Generated by SF Cartographer | YYYY-MM-DD -->

# {Project Name} — Project Documentation

> Auto-generated by [SF Cartographer](https://github.com/missed86/SalesforceAgenticDevelopment).
> Last updated: YYYY-MM-DD

## Project Summary

| Metric | Count |
|--------|-------|
| Package directories | N |
| Apex classes | N |
| Apex triggers | N |
| LWC components | N |
| Aura components | N |
| Custom objects | N |
| Custom fields (total) | N |
| Flows | N |
| Permission sets | N |
| Profiles | N |
| Named credentials | N |

## Documentation Map

| Document | Description |
|----------|-------------|
| [Architecture](architecture.md) | Architecture patterns, layer diagram, scaling assessment |
| [Data Model](data-model.md) | Custom objects, fields, relationships, events, CMDT |
| [Apex Inventory](apex-inventory.md) | All Apex classes categorized by layer |
| [LWC Catalog](lwc-catalog.md) | Lightning Web Components catalog |
| [Automation Map](automation-map.md) | Flows, validation rules, approval processes |
| [Security Model](security-model.md) | Profiles, permission sets, sharing rules |
| [Integrations](integrations.md) | Named credentials, remote sites, connected apps |
| [UI Configuration](ui-config.md) | Layouts, Lightning pages, tabs, apps |
| [Project Structure](project-structure.md) | Full directory tree |

## Observed Naming Patterns

{Summary table of naming patterns detected in the project: class suffixes, Flow prefixes, permission set naming, object/field style, etc.}
```

#### 3.2 architecture.md

Must contain:
- **Layers detected** with ASCII diagram showing the actual architecture tiers found
- **Classes per layer** counts
- **Scaling assessment** (small / medium / large based on class count)
- **Foundation classes present** (Logger, TriggerControl, AppException, etc.)
- **Async patterns in use** (Batch, Queueable, Schedulable, Platform Events)
- **Test coverage by convention** (classes without matching `*Test`)

#### 3.3 data-model.md

Must contain:
- **Custom objects table**: API name, label, description, record types, number of custom fields
- **Relationship map**: ASCII or table showing Master-Detail and Lookup relationships
- **Platform events**: name, fields, description
- **Custom metadata types**: name, fields, purpose
- **Custom labels**: grouped by category/prefix with count
- **Standard objects with customizations**: any standard object that has custom fields

#### 3.4 apex-inventory.md

Must contain:
- **Summary table**: layer | class count
- **Full class inventory** organized by layer:
  - Triggers + Handlers
  - Controllers (LWC-facing)
  - Services
  - Selectors
  - Domains
  - DTOs / Wrappers
  - Async (Batch, Queueable, Schedulable)
  - REST Resources
  - Utilities / Helpers
  - Test classes
  - Uncategorized
- For each class: API name, detected layer, sharing mode, lines of code, whether a test exists
- **Custom exceptions** found

#### 3.5 lwc-catalog.md

Must contain:
- **Component table**: name, type (container/presentational/service/form), `isExposed`, targets
- **Detailed component cards**: for each component list `@api` properties, custom events dispatched, Apex methods called, child components used
- **Aura components** (if any): name, type, noted as legacy
- **Service components** (headless JS modules)
- **LMS channels** used

#### 3.6 automation-map.md

Must contain:
- **Flows table**: API name, type (Record-Triggered/Screen/Autolaunched/Scheduled/Subflow), object, trigger timing, status, API version
- **Validation rules**: per object, API name, description, error message, active status
- **Approval processes**: name, entry criteria, approval steps

#### 3.7 security-model.md

Must contain:
- **Profiles**: name, description (if available), license type
- **Permission sets**: name, description, license, key object/field permissions granted
- **Permission set groups**: name, included permission sets
- **Custom permissions**: name, description
- **Sharing rules**: object, criteria, access level
- **Queues**: name, supported objects
- **Public groups**: name, members (if discoverable)

#### 3.8 integrations.md

Must contain:
- **Named credentials**: name, endpoint URL, auth type
- **External credentials**: name, authentication protocol
- **Remote site settings**: name, URL, active status
- **Connected apps**: name, description
- **Apex callout classes**: classes that implement `HttpCalloutMock` or use `Http`, `HttpRequest`

#### 3.9 ui-config.md

Must contain:
- **Applications**: name, type (Classic/Lightning), tabs included
- **Lightning pages**: name, type (Record/App/Home), objects
- **Page layouts**: object, name, audience
- **Tabs**: name, object or URL
- **Quick actions**: name, object, type
- **Static resources**: name, content type, size
- **Message channels**: name, fields

#### 3.10 project-structure.md

Must contain:
- **sfdx-project.json** package directory listing
- **Full directory tree** (2 levels deep from each package directory, 3 levels for `classes/` and `lwc/`)
- **File counts** per directory
- **API version** from project config

### Phase 4 — Update Mode

When `docs/INDEX.md` already exists:

1. Read the existing `INDEX.md` header to confirm it was generated by SF Cartographer.
2. **Regenerate every file entirely** with fresh scan data — do not attempt to diff.
3. **Preserve custom blocks**: any content between `<!-- CUSTOM -->` and `<!-- /CUSTOM -->` markers must be copied verbatim into the regenerated file at the same position.
4. Update the `Last updated` timestamp in `INDEX.md`.
5. At the end, summarize what changed: new items found, items removed, new patterns detected.

---

## Scanner Reference (Progressive Disclosure)

Read these only when you need detailed parsing instructions for a specific metadata type:

| Scanner | File | When to Read |
|---------|------|-------------|
| Apex Classes & Triggers | [apex-scanner.md](scanners/apex-scanner.md) | Scanning `.cls` and `.trigger` files |
| LWC & Aura Components | [lwc-scanner.md](scanners/lwc-scanner.md) | Scanning `lwc/` and `aura/` folders |
| Data Model | [data-model-scanner.md](scanners/data-model-scanner.md) | Scanning objects, fields, events, CMDT, labels |
| Automation | [automation-scanner.md](scanners/automation-scanner.md) | Scanning Flows, validation rules, approvals |
| Security & Access | [security-scanner.md](scanners/security-scanner.md) | Scanning profiles, perm sets, sharing rules |
| Integrations | [integration-scanner.md](scanners/integration-scanner.md) | Scanning named creds, remote sites, connected apps |
| UI Configuration | [ui-scanner.md](scanners/ui-scanner.md) | Scanning layouts, flexipages, tabs, apps |

---

## Execution Strategy

For large projects, parallelize aggressively:

```
                    ┌─ Apex scanner ──────────┐
                    ├─ LWC scanner ───────────┤
  Phase 1:          ├─ Data model scanner ────┤ (parallel)
  Discovery         ├─ Automation scanner ────┤
                    ├─ Security scanner ──────┤
                    ├─ Integration scanner ───┤
                    └─ UI scanner ────────────┘
                              │
                              ▼
  Phase 2:          Cross-cutting analysis
  Analysis          (architecture, relationships, patterns)
                              │
                              ▼
                    ┌─ INDEX.md ──────────────┐
  Phase 3:          ├─ architecture.md ───────┤
  Generation        ├─ data-model.md ─────────┤ (parallel)
                    ├─ apex-inventory.md ─────┤
                    ├─ lwc-catalog.md ────────┤
                    ├─ ... remaining docs ────┤
                    └─ project-structure.md ──┘
```

Use subagents (`Task` tool) when scanning large codebases to parallelize Phase 1.

---

## Quality Rules

1. **Never invent information** — only document what exists in the source files.
2. **Never truncate** — if the project has 200 classes, list all 200.
3. **Tables over prose** — inventories must be tables, not paragraphs.
4. **Counts must be accurate** — triple-check totals.
5. **Links must be relative** — all inter-doc links use relative paths.
6. **Date format** — ISO 8601 (`YYYY-MM-DD`) for all timestamps.
7. **English only** — all generated documentation in English regardless of project language.
8. **No hallucinated metadata** — if an XML field is missing or empty, say "Not specified" rather than guessing.
