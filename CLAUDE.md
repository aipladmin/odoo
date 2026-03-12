# CLAUDE.md — Odoo Development Guide

## Project Overview

Odoo 19.0 (Community Edition) — a comprehensive open-source ERP and business application suite. Python 3.10–3.13, PostgreSQL 13+, licensed under LGPL-3.

## Repository Structure

```
odoo-bin              # Entry point: ./odoo-bin [command] [options]
odoo/                 # Core framework
  orm/                # ORM: BaseModel, fields, environments, registry, domains
  api/                # Public ORM API exports (decorators, Environment, SUPERUSER_ID)
  fields/             # Field type definitions
  models/             # Model re-exports
  modules/            # Module loading, registry, migration
  cli/                # CLI commands (server, shell, scaffold, db, cloc, i18n, etc.)
  service/            # RPC services, server lifecycle
  tools/              # Utilities (config, safe_eval, SQL, cache, image, mail, etc.)
  http.py             # HTTP/WSGI layer, request/response, routing
  tests/              # Core test framework (TransactionCase, HttpCase, tagged, Form)
  upgrade/            # Database upgrade scripts
  upgrade_code/       # Code upgrade automation
addons/               # ~612 addon modules (the business logic)
setup/                # Packaging and installation scripts
doc/                  # Documentation (CLA)
```

## Addon Module Structure

Each addon in `addons/` follows this convention:

```
addons/<module_name>/
  __manifest__.py     # Module metadata: name, version, depends, data, assets, demo
  __init__.py         # Python imports
  models/             # ORM model definitions (one file per model or logical group)
  views/              # XML view definitions (form, list, kanban, search views)
  controllers/        # HTTP route controllers (inheriting Controller)
  security/           # Access control:
    ir.model.access.csv   # Model-level CRUD permissions per group
    ir_rules.xml          # Record-level domain rules
    res_groups.xml        # Security group definitions
  data/               # Default data loaded on install (XML/CSV)
  demo/               # Demo data (only loaded in demo mode)
  wizard/             # Transient model wizards (views + models)
  report/             # QWeb report templates and report actions
  static/             # Web assets:
    src/              # JS, SCSS, XML (OWL components/templates)
    tests/            # JS test files
  tests/              # Python test files (test_*.py)
  i18n/               # Translation files (.po/.pot)
```

### Manifest File (`__manifest__.py`)

Key fields: `name`, `version`, `category`, `summary`, `depends` (list of module dependencies), `data` (XML/CSV files loaded in order), `demo`, `assets` (JS/CSS bundle declarations), `installable`, `auto_install`, `license` (always `'LGPL-3'`), `post_init_hook`.

## ORM Patterns

### Model Types
- `models.Model` — persistent database models (regular tables)
- `models.TransientModel` — temporary records (auto-vacuumed, for wizards)
- `models.AbstractModel` — no database table, used as mixins

### Model Definition
```python
from odoo import api, fields, models

class MyModel(models.Model):
    _name = 'my.model'              # Dot-notation technical name
    _description = 'My Model'
    _inherit = ['mail.thread']       # Mixin inheritance
    _order = 'date desc, id desc'

    name = fields.Char(string='Name', required=True)
    partner_id = fields.Many2one('res.partner', string='Partner')
    line_ids = fields.One2many('my.model.line', 'order_id')
    state = fields.Selection([('draft', 'Draft'), ('done', 'Done')])
```

### Key Decorators (`odoo.api`)
- `@api.depends('field')` — computed field dependencies
- `@api.constrains('field')` — validation constraints
- `@api.onchange('field')` — UI-triggered logic
- `@api.model` — classmethod-like (no recordset)
- `@api.model_create_multi` — batch create
- `@api.ondelete(at_uninstall=False)` — delete hook

### Record Operations
- `fields.Command` for relational writes: `Command.create({})`, `Command.update(id, {})`, `Command.delete(id)`, `Command.link(id)`, `Command.unlink(id)`, `Command.set([ids])`, `Command.clear()`
- `self.env['model.name']` to access other models
- `self.env.user`, `self.env.company`, `self.env.context`

## HTTP Controllers

```python
from odoo.http import Controller, request, route

class MyController(Controller):
    @route('/my/path', type='http', auth='public', website=True)
    def my_page(self, **kwargs):
        return request.render('module.template_id', values)

    @route('/my/api', type='jsonrpc', auth='user')
    def my_api(self, param):
        return {'result': ...}
```

Route types: `type='http'` (HTML/redirect) or `type='jsonrpc'` (JSON-RPC).
Auth: `'public'`, `'user'`, `'none'`.

## Frontend (OWL Framework)

Odoo uses the OWL component framework for its web client. Frontend code lives in `addons/<module>/static/src/`.

- Components: JS classes extending `Component` with XML templates
- Assets declared in `__manifest__.py` under `assets` key
- Key bundles: `web.assets_backend`, `web.assets_frontend`, `web.assets_tests`, `web.assets_unit_tests`
- Views: form, list, kanban, graph, pivot, calendar (defined in XML, rendered by OWL)

## Running Odoo

```bash
# Start the server
./odoo-bin --addons-path=addons,odoo/addons -d <database>

# Common options
  -d <database>           # Database name
  --addons-path=<paths>   # Comma-separated addon directories
  -i <module>             # Install module(s)
  -u <module>             # Update module(s)
  --dev=xml,reload        # Dev mode (auto-reload, XML hot-reload)
  -p <port>               # HTTP port (default: 8069)
  --db_host, --db_port, --db_user, --db_password  # PostgreSQL connection

# Interactive shell
./odoo-bin shell -d <database>

# Scaffold a new module
./odoo-bin scaffold <module_name> <destination_path>

# Other CLI commands
./odoo-bin cloc           # Count lines of code
./odoo-bin db             # Database management
./odoo-bin i18n           # Translation import/export
./odoo-bin neutralize     # Neutralize a production database
./odoo-bin populate       # Generate test data
```

## Testing

### Running Tests

```bash
# Run tests for a specific module during install
./odoo-bin -d test_db -i <module> --test-enable --stop-after-init

# Run tests for a specific module during update
./odoo-bin -d test_db -u <module> --test-enable --stop-after-init

# Run specific test tags
./odoo-bin -d test_db --test-tags=/<module>:<class>.<method>

# Run post_install tests only
./odoo-bin -d test_db --test-tags=post_install --stop-after-init
```

### Test Classes

```python
from odoo.tests import TransactionCase, HttpCase, tagged, Form

@tagged('post_install', '-at_install')
class TestMyFeature(TransactionCase):
    """Each test method runs in its own rolled-back transaction."""

    @classmethod
    def setUpClass(cls):
        super().setUpClass()
        cls.partner = cls.env['res.partner'].create({'name': 'Test'})

    def test_something(self):
        self.assertEqual(self.partner.name, 'Test')
```

- `TransactionCase` — each test in a separate transaction (rolled back)
- `SingleTransactionCase` — all tests share one transaction
- `HttpCase` — includes a browser for UI/tour testing
- `@tagged('post_install', '-at_install')` — controls when tests run
- Default tags: `standard`, `at_install`
- `Form` helper for testing form views and onchanges

### Test File Naming

Test files must be named `test_*.py` and imported in `tests/__init__.py`.

## Linting (Ruff)

Configured in `ruff.toml`. Target: Python 3.10.

```bash
ruff check .                    # Run linter
ruff check --fix .              # Auto-fix issues
```

### Key Rules
- Line length: **not enforced** (E501 ignored)
- Import order (isort): `future` → `standard-library` → `third-party` → `first-party` (`odoo`) → `local-folder` (`odoo.addons`)
- No `print()` statements (T rules enabled)
- Unused imports allowed only in `__init__.py`
- See `ruff.toml` for full rule configuration

### Import Conventions
```python
# Standard library
import logging
from datetime import datetime

# Third-party
import requests
from lxml import etree

# Odoo core (first-party)
from odoo import api, fields, models
from odoo.exceptions import UserError, ValidationError
from odoo.tools import SQL, float_compare

# Odoo addons (local-folder)
from odoo.addons.sale.models.sale_order import SALE_ORDER_STATE
```

## Key Conventions

### Python
- All files start with `# Part of Odoo. See LICENSE file for full copyright and licensing details.`
- Model names use dot notation: `sale.order`, `account.move`
- One model per file (or logical grouping), filename matches model: `sale_order.py` for `sale.order`
- Use `_()` for translatable strings
- Security: always consider access rights; use `sudo()` sparingly and document why
- Use `self.env.ref('module.xml_id')` to reference data records

### XML
- XML IDs follow: `<module>.<type>_<model_underscored>_<description>` (e.g., `sale.view_order_form`)
- View inheritance uses `<xpath>` expressions or field `name` attribute with `position`

### JavaScript
- OWL components in `static/src/`
- Test files in `static/tests/` (unit tests: `*.test.js`)
- Tours for integration tests in `static/tests/tours/`

### Security
- `ir.model.access.csv`: model-level CRUD per group (`id,name,model_id:id,group_id:id,perm_read,perm_write,perm_create,perm_unlink`)
- `ir_rules.xml`: record-level domain-based rules
- `res_groups.xml`: security group definitions with implied groups

### Database
- PostgreSQL required (13+); no raw SQL unless necessary — prefer the ORM
- When raw SQL is needed, use `odoo.tools.SQL` for safe query building
- Migrations go in `migrations/<version>/` directories within the module

## Dependencies

Key Python packages: `lxml`, `psycopg2`, `werkzeug`, `Pillow`, `reportlab`, `requests`, `gevent`, `babel`, `cryptography`, `passlib`, `freezegun` (tests).

Full list in `requirements.txt`.

## Common Gotchas

- `self` in model methods is always a **recordset**, not a single record — iterate or use `ensure_one()`
- Computed fields need `store=True` to be searchable/sortable
- `@api.depends` is required for stored computed fields
- XML data files in `__manifest__.py` `data` list are loaded in order — references must come after definitions
- `__init__.py` files must import all submodules/subpackages explicitly
- Test databases need modules installed before tests can run against them
