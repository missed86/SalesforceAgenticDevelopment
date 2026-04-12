# Salesforce Agent Skills

Reusable [Agent Skills](https://agentskills.io) for Salesforce development. Each skill is a `SKILL.md` with YAML frontmatter (`name`, `description`) so coding agents can load focused guidance for Apex, LWC, and declarative metadata.

## Skills

| Skill | Summary |
| ----- | ------- |
| [apex-architecture](skills/apex-architecture/SKILL.md) | Apex architecture, layers, triggers, services, selectors, security, testing, bulkification, and governor limits (framework-free). |
| [lwc-best-practices](skills/lwc-best-practices/SKILL.md) | Lightning Web Components: architecture, data, lifecycle, performance, accessibility, security, and Jest testing. |
| [salesforce-metadata-standards](skills/salesforce-metadata-standards/SKILL.md) | Declarative metadata naming, descriptions, and governance (objects, fields, Flows, permission sets, and more). |

## Install with the Skills CLI

Use the open [skills](https://github.com/vercel-labs/skills) CLI (`npx skills add`). Replace `owner/repo` with your GitHub namespace after you publish this repository.

```bash
# List skills in the repo without installing
npx skills add owner/repo --list

# Install all discovered skills (interactive; pick agents such as Cursor)
npx skills add owner/repo

# Install for Cursor only, non-interactive
npx skills add owner/repo -a cursor -y

# Install specific skills
npx skills add owner/repo --skill apex-architecture --skill lwc-best-practices
```

Other useful forms:

```bash
npx skills add https://github.com/owner/repo
npx skills add https://github.com/owner/repo/tree/main/skills/apex-architecture
```

For a **local** checkout before publishing:

```bash
npx skills add /path/to/this/repo --list
```

## Repository layout

```text
skills/
├── apex-architecture/
│   └── SKILL.md
├── lwc-best-practices/
│   └── SKILL.md
└── salesforce-metadata-standards/
    └── SKILL.md
```

## License

Add a `LICENSE` file in the repository root if you intend to share this collection publicly.
