# Salesforce Agent Skills

Reusable [Agent Skills](https://agentskills.io) for Salesforce development. Each skill is a `SKILL.md` with YAML frontmatter (`name`, `description`, optional `metadata`) per the [Agent Skills specification](https://agentskills.io/specification.md). Supporting detail lives in `patterns/` and `standards/` for progressive disclosure.

## Skills

| Skill | Summary |
| ----- | ------- |
| [apex-architecture](skills/apex-architecture/SKILL.md) | Apex architecture, layers, triggers, services, selectors, security, testing, bulkification, and governor limits (framework-free). |
| [lwc-best-practices](skills/lwc-best-practices/SKILL.md) | Lightning Web Components: architecture, wire/LDS, lifecycle, performance, accessibility, client security (LWS/CSP), and Jest testing. |
| [salesforce-metadata-standards](skills/salesforce-metadata-standards/SKILL.md) | Declarative metadata naming, descriptions, and governance (objects, fields, Flows, permission sets, and more). |
| [salesforce-foundation-init](skills/salesforce-foundation-init/SKILL.md) | Scaffolds base Apex classes and metadata for a new project: Logger, TriggerControl, AppException, ServiceResponseDto, RecordTypes, TestDataFactory. |

## Install with the Skills CLI

Use the open [skills](https://github.com/vercel-labs/skills) CLI (`npx skills add`) against this repository:

**Repository:** [missed86/SalesforceAgenticDevelopment](https://github.com/missed86/SalesforceAgenticDevelopment)

```bash
# List skills in the repo without installing
npx skills add missed86/SalesforceAgenticDevelopment --list

# Install all discovered skills (interactive; pick agents such as Cursor)
npx skills add missed86/SalesforceAgenticDevelopment

# Install for Cursor only, non-interactive
npx skills add missed86/SalesforceAgenticDevelopment -a cursor -y

# Install specific skills
npx skills add missed86/SalesforceAgenticDevelopment --skill apex-architecture --skill lwc-best-practices
```

Other useful forms:

```bash
npx skills add https://github.com/missed86/SalesforceAgenticDevelopment
npx skills add https://github.com/missed86/SalesforceAgenticDevelopment/tree/main/skills/apex-architecture
```

From a **local** clone of this repo:

```bash
npx skills add . --list
```

## Repository layout

```text
skills/
├── apex-architecture/
│   ├── SKILL.md
│   ├── patterns/
│   └── examples/
├── lwc-best-practices/
│   ├── SKILL.md
│   └── patterns/
├── salesforce-metadata-standards/
│   ├── SKILL.md
│   └── standards/
└── salesforce-foundation-init/
    └── SKILL.md
```

## License

Add a `LICENSE` file in the repository root if you intend to share this collection publicly.
