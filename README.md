# Odoo Development Universal Skill

A universal Odoo development skill for AI agents compatible with `skills.sh`. It provides fast code indexing, intelligent patterns, and strict adherence to Odoo Community Association (OCA) standards for versions 14 to 20.

This repository is a fork and adaptation of the Odoo plugin from [letzdoo/claude-marketplace](https://github.com/letzdoo/claude-marketplace), restructured to be 100% agnostic and compatible with any IDE that supports `skills.sh` (like Windsurf, Cursor, Cline, etc.).

## 🌟 Features

- **Version Awareness**: Covers Odoo 14 to 20, including breaking changes and deprecated methods.
- **OCA Standards Strict Adherence**: Enforces PEP8, DRY, KISS, and SOLID principles.
- **OWL Compatibility**: Knowledge of OWL 1.x (v15), 2.x (v16-19), and 3.x (v20).
- **Odoo 20 Ready**: `ir.access` security, OWL 3, `BinaryValue`, `zoneinfo`, Material Symbols icons and the 19→20 migration path (`odoo-bin upgrade_code`).
- **Specialized Agents**: Includes 4 workflows (agents) for Context Gathering, Code Review, Upgrade Analysis, and Skill Discovery.
- **142 Skill Patterns**: Detailed copy-paste ready code snippets for every functional domain in Odoo.

## 📦 Installation

To install this skill globally in your environment using `skills.sh`:

```bash
npx skills add fhidalgodev/odoo-development-skill --global
```

Or install it for a specific project:

```bash
npx skills add fhidalgodev/odoo-development-skill
```

## 🧠 Architecture

```text
odoo-development-skill/
├── SKILL.md                 # Main entrypoint for skills.sh
├── README.md                # This documentation
├── agents/                  # 4 specialized workflows
│   ├── odoo-context-gatherer.md
│   ├── odoo-code-reviewer.md
│   ├── odoo-upgrade-analyzer.md
│   └── odoo-skill-finder.md
└── skills/                  # 142 modular pattern files across 8 domains
    ├── backend/             # Python models, ORM, hooks, queue_job, API, controllers (24 files)
    ├── frontend/            # XML views, OWL components (v15-v20), advanced patterns (23 files)
    ├── database/            # Performance, raw SQL cache sync, locking, SQL views (7 files)
    ├── security/            # Security guides (v14-v20), safe sudo & IDOR audit, portal access (17 files)
    ├── functional/          # Standard ERP modules: stock, account, mrp, pos, payment, sale, hr (14 files)
    ├── testing/             # Test patterns, form view simulation, tour tests, mocking, OCA CI (6 files)
    ├── deploy/              # Dedicated configs: odoo.conf, nginx, postgres, docker, backup (5 files)
    └── versions/            # Version-specific core patterns (v14-v20) & migration guides (46 files)
```

## 🛠️ Usage

Once installed, your AI assistant will automatically read `SKILL.md` when Odoo-related tasks are requested. The assistant will:
1. Detect your Odoo version automatically.
2. Check existing Enterprise or Community modules before reinventing the wheel.
3. Fetch the exact pattern needed from the `skills/` folder.
4. Implement the solution adhering to OCA guidelines.

## 🤝 Contributing

Contributions, issues, and feature requests are welcome!

## 📝 License

This project is licensed under the MIT License - see the LICENSE file for details.
