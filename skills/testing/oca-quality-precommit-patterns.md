# OCA Code Quality, Pre-Commit & CI/CD Patterns

```
╔══════════════════════════════════════════════════════════════════════════════╗
║  OCA CODE QUALITY, PRE-COMMIT & CI/CD PIPELINE PATTERNS                      ║
║  Standardized .pre-commit-config.yaml, pylint-odoo rules, formatting,        ║
║  Automated Test Execution CLI, and GitHub Actions CI Workflow                ║
║  Covers Odoo 16.0, 17.0, 18.0, 19.0, and 20.0                                ║
╚══════════════════════════════════════════════════════════════════════════════╝
```

## Overview

The Odoo Community Association (OCA) enforces strict coding and architectural standards across all official modules. To ensure maximum reliability, security, and maintainability, modules should adhere to:
1. **Automated Static Analysis**: Linting via `pylint-odoo` and `flake8` to detect security vulnerabilities, anti-patterns, and deprecated APIs.
2. **Deterministic Code Formatting**: Formatting via `black` (Python) and `prettier` (XML, JS, SCSS).
3. **Automated Testing**: Running test suites on clean databases via CLI with `--test-enable`.
4. **CI/CD Integration**: Automated quality gates running on pull requests.

---

## 1. Production `.pre-commit-config.yaml`

Place this file at the root of your Odoo addons repository. Run `pre-commit install` to enable git commit hooks:

```yaml
exclude: |
  (?x)
  # Exclude binary data or third-party vendored assets
  static/lib/|
  static/src/lib/

repos:
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.6.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-yaml
      - id: check-xml
      - id: check-json
      - id: debug-statements

  - repo: https://github.com/psf/black
    rev: 24.8.0
    hooks:
      - id: black
        args: [--line-length=88]

  - repo: https://github.com/PyCQA/flake8
    rev: 7.1.1
    hooks:
      - id: flake8
        additional_dependencies:
          - flake8-bugbear
        args:
          - --max-line-length=88
          - --extend-ignore=E203,E501,W503

  - repo: https://github.com/OCA/pylint-odoo
    rev: v10.0.12
    hooks:
      - id: pylint
        name: pylint-odoo
        args:
          - --rcfile=.pylintrc
          - --load-plugins=pylint_odoo
        verbose: true

  - repo: https://github.com/pre-commit/mirrors-prettier
    rev: v3.1.0
    hooks:
      - id: prettier
        types_or: [javascript, scss, xml]
        additional_dependencies:
          - prettier@3.3.3
          - "@prettier/plugin-xml@0.12.0"
```

---

## 2. Standardized `.pylintrc` Configuration

Create `.pylintrc` at the root of the repository to configure specific rules and severities:

```ini
[MASTER]
load-plugins=pylint_odoo
score=no

[MESSAGES CONTROL]
disable=all
enable=
    # Security checks
    sql-injection,
    unhandled-error-mode,
    
    # Odoo ORM best practices
    api-one-deprecated,
    api-one-multi-together,
    attribute-deprecated,
    class-camelcase,
    copy-attribute-deprecated,
    create-user-wo-reset-password,
    dangerous-default-value,
    duplicate-id-csv,
    field-name-warning,
    incoherent-interpreter-exec-perm,
    invalid-commit,
    manifest-author-empty,
    manifest-deprecated-key,
    manifest-required-author,
    manifest-required-key,
    manifest-version-format,
    method-compute-clean,
    method-inverse-clean,
    method-required-super,
    missing-newline-ext-files,
    missing-readme,
    no-wizard-in-models,
    odoo-addons-relative-import,
    old-api7-method-defined,
    print-used,
    redundant-modulename-xml,
    sql-injection,
    translation-contains-variable,
    translation-field,
    translation-required,
    use-vim-comment,
    xml-deprecated-qweb-directive,
    xml-deprecated-tree-attribute

[REPORTS]
output-format=colorized
reports=no
```

---

## 3. OCA Module Structure & Metadata Standards

To satisfy OCA linter rules and packaging standards, structure every module as follows:

```text
my_addon_module/
├── __init__.py
├── __manifest__.py
├── models/
│   ├── __init__.py
│   └── my_model.py
├── views/
│   └── my_model_views.xml
├── security/
│   ├── ir.model.access.csv      # (or security/ir.access.csv in Odoo 20)
│   └── my_model_security.xml
├── readme/
│   ├── DESCRIPTION.rst          # Detailed technical description
│   ├── USAGE.rst                # User how-to and configuration guide
│   └── CONTRIBUTORS.rst         # Author and maintainer list
├── static/
│   └── description/
│       └── icon.png             # 100x100 PNG or SVG module icon
└── tests/
    ├── __init__.py
    └── test_my_model.py
```

### Standard OCA Manifest Keys

```python
{
    'name': 'My Clean Addon',
    'version': '18.0.1.0.0',
    'category': 'Sales',
    'summary': 'Short description of feature',
    'author': 'Your Name, Odoo Community Association (OCA)',
    'website': 'https://github.com/OCA/your-repo',
    'license': 'AGPL-3',  # Standard OCA license (or LGPL-3)
    'depends': ['sale_management'],
    'data': [
        'security/ir.model.access.csv',
        'views/my_model_views.xml',
    ],
    'installable': True,
    'application': False,
    'auto_install': False,
}
```

---

## 4. Automated Testing via Odoo CLI

To run tests without starting a web server or leaving dirty test records in production databases:

### Syntax:
```bash
odoo-bin \
  -c /path/to/odoo.conf \
  -d test_database_name \
  --test-enable \
  --stop-after-init \
  -i my_addon_module \
  --test-tags /my_addon_module
```

### Flags Breakdown:
- `--test-enable`: Activates unit and integration test runner (`TransactionCase`, `HttpCase`).
- `--stop-after-init`: Terminates the Odoo process immediately after running tests and returns exit code `0` (pass) or `1` (fail).
- `-i <module>`: Installs/upgrades the module in the test database.
- `--test-tags /<module>`: Filters test discovery to only execute tests belonging to this specific module directory.

---

## 5. GitHub Actions CI Workflow (`.github/workflows/test.yml`)

Set up automated testing and linting on every push and pull request:

```yaml
name: CI Quality & Test Suite

on:
  push:
    branches: [main, 16.0, 17.0, 18.0]
  pull_request:
    branches: [main, 16.0, 17.0, 18.0]

jobs:
  pre-commit:
    name: Lint & Code Style
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.11"
      - name: Run Pre-Commit Hooks
        uses: pre-commit/action@v3.0.1

  odoo-unit-tests:
    name: Odoo Unit Tests
    needs: pre-commit
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_USER: odoo
          POSTGRES_PASSWORD: odoo_password
          POSTGRES_DB: postgres
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    steps:
      - uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.11"
          cache: "pip"

      - name: Install System Dependencies
        run: |
          sudo apt-get update
          sudo apt-get install -y libpq-dev libldap2-dev libsasl2-dev libxml2-dev libxslt1-dev

      - name: Run Test Suite
        env:
          HOST: 127.0.0.1
          USER: odoo
          PASSWORD: odoo_password
        run: |
          # Run tests with stop-after-init
          python3 odoo-bin \
            --db_host=127.0.0.1 \
            --db_port=5432 \
            --db_user=odoo \
            --db_password=odoo_password \
            -d ci_test_db \
            --test-enable \
            --stop-after-init \
            -i my_addon_module \
            --test-tags /my_addon_module
```
