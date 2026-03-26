---
name: odoo-18-webservices-reports
description: Odoo 18 guide for external web services (XML-RPC, JSON-RPC, API keys), custom QWeb reports (PDF/HTML), paper formats, barcodes, and multi-company patterns. Use when building integrations, creating reports, or implementing multi-company logic.
---

# Odoo 18 Web Services, Custom Reports & Multi-Company

## 1. Web Services (External API)

Odoo exposes its business logic via XML-RPC, JSON-RPC, and HTTP endpoints. External clients (Python, Node.js, PHP, etc.) can authenticate, read, create, update, and delete records remotely.

### 1.1 XML-RPC Endpoints

Two XML-RPC endpoints:

| Endpoint | Fault Codes | Notes |
|----------|-------------|-------|
| `/xmlrpc/<service>` | String (legacy) | Backwards-compatible |
| `/xmlrpc/2/<service>` | Integer (compliant) | Recommended |

**Services:** `common` (auth/version), `object` (ORM operations), `db` (database management)

### 1.2 Authentication

#### Password Authentication (XML-RPC - Python)

```python
import xmlrpc.client

url = "https://myodoo.com"
db = "mydb"
username = "admin"
password = "admin"

# 1. Get server version (no auth needed)
common = xmlrpc.client.ServerProxy(f"{url}/xmlrpc/2/common")
version = common.version()
# {'server_version': '18.0', 'server_version_info': [18, 0, 0, 'final', 0, ''], ...}

# 2. Authenticate → returns uid
uid = common.authenticate(db, username, password, {})

# 3. Use uid for all subsequent calls
models = xmlrpc.client.ServerProxy(f"{url}/xmlrpc/2/object")
```

#### Password Authentication (JSON-RPC - Python)

```python
import json
import requests

url = "https://myodoo.com"
db = "mydb"

def jsonrpc(url, service, method, args):
    payload = {
        "jsonrpc": "2.0",
        "id": None,
        "method": "call",
        "params": {
            "service": service,
            "method": method,
            "args": args,
        },
    }
    resp = requests.post(f"{url}/jsonrpc", json=payload)
    result = resp.json()
    if "error" in result:
        raise Exception(result["error"]["data"]["message"])
    return result["result"]

# Authenticate
uid = jsonrpc(url, "common", "authenticate", [db, "admin", "admin", {}])

# Call ORM methods
partner_ids = jsonrpc(url, "object", "execute_kw",
    [db, uid, "admin", "res.partner", "search", [[["is_company", "=", True]]]])
```

#### JSON-RPC (Node.js)

```javascript
const fetch = require("node-fetch");

const url = "https://myodoo.com";
const db = "mydb";

async function jsonrpc(service, method, args) {
    const response = await fetch(`${url}/jsonrpc`, {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({
            jsonrpc: "2.0",
            id: Date.now(),
            method: "call",
            params: { service, method, args },
        }),
    });
    const result = await response.json();
    if (result.error) throw new Error(result.error.data.message);
    return result.result;
}

// Authenticate
const uid = await jsonrpc("common", "authenticate", [db, "admin", "admin", {}]);

// Search partners
const ids = await jsonrpc("object", "execute_kw",
    [db, uid, "admin", "res.partner", "search", [[["is_company", "=", true]]]]);
```

#### API Key Authentication

API keys can be used in place of passwords for all RPC calls:

```python
# Generate API key: Settings → Users → API Keys → New API Key
api_key = "your-api-key-here"

# Use API key exactly like a password
uid = common.authenticate(db, username, api_key, {})
result = models.execute_kw(db, uid, api_key, "res.partner", "search", [[]])
```

**Bearer Token (HTTP):**

```python
# For HTTP/JSON controllers that support bearer auth
headers = {"Authorization": f"Bearer {api_key}"}
response = requests.get(f"{url}/api/endpoint", headers=headers)
```

**API Key internals:**
- Stored as PBKDF2-SHA512 hashed values (6000 rounds)
- Scope-based: `scope='rpc'` for general RPC access
- Support expiration dates
- Auto garbage-collected via `_gc_user_apikeys()` autovacuum

### 1.3 CRUD Operations via RPC

All ORM operations use `execute_kw(db, uid, password, model, method, args, kwargs)`.

#### Search

```python
# search(domain) → list of IDs
ids = models.execute_kw(db, uid, password,
    "res.partner", "search",
    [[["is_company", "=", True], ["country_id.code", "=", "US"]]],
    {"limit": 10, "offset": 0, "order": "name asc"})
```

#### Search Count

```python
# search_count(domain) → integer
count = models.execute_kw(db, uid, password,
    "res.partner", "search_count",
    [[["is_company", "=", True]]])
```

#### Read

```python
# read([ids], [fields]) → list of dicts
records = models.execute_kw(db, uid, password,
    "res.partner", "read",
    [ids],
    {"fields": ["name", "email", "phone", "country_id"]})
# Returns: [{"id": 1, "name": "...", "email": "...", "country_id": [10, "US"]}, ...]
```

#### Search + Read (Combined)

```python
# search_read(domain, fields, limit, offset, order) → list of dicts
records = models.execute_kw(db, uid, password,
    "res.partner", "search_read",
    [[["is_company", "=", True]]],
    {"fields": ["name", "email"], "limit": 5, "offset": 0, "order": "name"})
```

#### Create

```python
# create([vals_list]) → list of IDs (or single ID for single dict)
new_id = models.execute_kw(db, uid, password,
    "res.partner", "create",
    [{"name": "New Partner", "email": "new@example.com", "is_company": True}])

# Batch create
new_ids = models.execute_kw(db, uid, password,
    "res.partner", "create",
    [[{"name": "Partner A"}, {"name": "Partner B"}]])
```

#### Write (Update)

```python
# write([ids], vals) → True
models.execute_kw(db, uid, password,
    "res.partner", "write",
    [[new_id], {"name": "Updated Name", "phone": "+1234567890"}])
```

#### Unlink (Delete)

```python
# unlink([ids]) → True
models.execute_kw(db, uid, password,
    "res.partner", "unlink",
    [[new_id]])
```

#### Fields Get (Introspection)

```python
# fields_get([field_names], attributes) → dict
fields = models.execute_kw(db, uid, password,
    "res.partner", "fields_get",
    [],
    {"attributes": ["string", "type", "help", "required", "readonly"]})
# Returns: {"name": {"string": "Name", "type": "char", ...}, ...}
```

#### Name Search

```python
# name_search(name, domain, operator, limit) → [(id, display_name), ...]
results = models.execute_kw(db, uid, password,
    "res.partner", "name_search",
    ["admin"],
    {"operator": "ilike", "limit": 10})
```

#### Read Group (Aggregation)

```python
# read_group(domain, fields, groupby) → list of group dicts
groups = models.execute_kw(db, uid, password,
    "res.partner", "read_group",
    [[["is_company", "=", True]]],
    {"fields": ["country_id"], "groupby": ["country_id"]})
# Returns: [{"country_id": [10, "US"], "country_id_count": 5, ...}, ...]
```

#### Calling Custom Methods

```python
# Any public method (not starting with _) can be called
result = models.execute_kw(db, uid, password,
    "sale.order", "action_confirm",
    [[order_id]])

# With keyword arguments
result = models.execute_kw(db, uid, password,
    "sale.order", "action_confirm",
    [[order_id]],
    {"context": {"send_email": True}})
```

### 1.4 Domain Filter Syntax

```python
# Basic comparison
[("field", "=", value)]
[("field", "!=", value)]
[("field", ">", value)]
[("field", ">=", value)]
[("field", "<", value)]
[("field", "<=", value)]

# String operators
[("name", "like", "pattern")]       # SQL LIKE (case-sensitive)
[("name", "ilike", "pattern")]      # Case-insensitive LIKE
[("name", "not like", "pattern")]
[("name", "not ilike", "pattern")]
[("name", "=like", "pat%ern")]      # Exact LIKE pattern
[("name", "=ilike", "pat%ern")]     # Case-insensitive exact LIKE

# Collection operators
[("id", "in", [1, 2, 3])]
[("id", "not in", [1, 2, 3])]

# Relational operators
[("partner_id.country_id.code", "=", "US")]   # Dot notation for related fields
[("partner_id", "child_of", parent_id)]        # Hierarchical (includes children)
[("partner_id", "parent_of", child_id)]        # Hierarchical (includes parents)

# Boolean logic (prefix notation)
["&", ("state", "=", "sale"), ("amount", ">", 1000)]           # AND (default)
["|", ("state", "=", "draft"), ("state", "=", "sent")]          # OR
["!", ("active", "=", False)]                                     # NOT

# Complex: (state = sale AND amount > 1000) OR state = draft
["|", "&", ("state", "=", "sale"), ("amount", ">", 1000), ("state", "=", "draft")]
```

### 1.5 Error Handling

**RPC Fault Codes:**

| Code | Meaning |
|------|---------|
| 1 | Client/Application Error |
| 2 | Warning |
| 3 | Access Error |
| 4 | Access Denied (authentication failure) |

```python
import xmlrpc.client

try:
    result = models.execute_kw(db, uid, password, "res.partner", "read", [[99999]])
except xmlrpc.client.Fault as e:
    print(f"Fault code: {e.faultCode}")
    print(f"Fault string: {e.faultString}")
```

### 1.6 Web Session Authentication (HTTP/JSON Controllers)

```python
# Authenticate via web session (for HTTP controllers)
session = requests.Session()

# Login
login_resp = session.post(f"{url}/web/session/authenticate", json={
    "jsonrpc": "2.0",
    "params": {
        "db": db,
        "login": username,
        "password": password,
    }
})

# Now use session for web controller calls
result = session.post(f"{url}/web/dataset/call_kw/res.partner/search_read", json={
    "jsonrpc": "2.0",
    "params": {
        "model": "res.partner",
        "method": "search_read",
        "args": [[["is_company", "=", True]]],
        "kwargs": {"fields": ["name"], "limit": 5},
    }
})
```

---

## 2. Custom Reports

### 2.1 Report Action (XML)

Define a report action to make it available from a model's print menu:

```xml
<record id="action_report_my_document" model="ir.actions.report">
    <field name="name">My Document</field>
    <field name="model">my.model</field>
    <field name="report_type">qweb-pdf</field>          <!-- qweb-pdf | qweb-html | qweb-text -->
    <field name="report_name">my_module.report_my_document</field>
    <field name="report_file">my_module.report_my_document</field>
    <field name="print_report_name">'Document - %s' % (object.name)</field>
    <field name="binding_model_id" ref="model_my_model"/>
    <field name="binding_type">report</field>            <!-- adds to Print menu -->
    <field name="paperformat_id" ref="my_module.my_paperformat"/>  <!-- optional -->
</record>
```

**Real example (sale order):**

```xml
<record id="action_report_saleorder" model="ir.actions.report">
    <field name="name">Quotation / Order</field>
    <field name="model">sale.order</field>
    <field name="report_type">qweb-pdf</field>
    <field name="report_name">sale.report_saleorder</field>
    <field name="report_file">sale.report_saleorder</field>
    <field name="print_report_name">
        (object.state in ('draft', 'sent') and 'Quotation - %s' % (object.name))
        or 'Order - %s' % (object.name)
    </field>
    <field name="binding_model_id" ref="model_sale_order"/>
    <field name="binding_type">report</field>
</record>
```

### 2.2 Report Template (QWeb)

Reports use QWeb templates with special wrapper templates:

```xml
<?xml version="1.0" encoding="utf-8"?>
<odoo>
    <!-- Main report template (iterates over docs) -->
    <template id="report_my_document">
        <t t-call="web.html_container">
            <t t-foreach="docs" t-as="doc">
                <t t-call="web.external_layout">
                    <div class="page">
                        <h2><span t-field="doc.name"/></h2>

                        <div class="row mt-4">
                            <div class="col-6">
                                <strong>Customer:</strong>
                                <span t-field="doc.partner_id.name"/>
                            </div>
                            <div class="col-6 text-end">
                                <strong>Date:</strong>
                                <span t-field="doc.date_order"
                                      t-options="{'widget': 'date'}"/>
                            </div>
                        </div>

                        <!-- Table of lines -->
                        <table class="table table-sm mt-4">
                            <thead>
                                <tr>
                                    <th>Product</th>
                                    <th class="text-end">Quantity</th>
                                    <th class="text-end">Unit Price</th>
                                    <th class="text-end">Subtotal</th>
                                </tr>
                            </thead>
                            <tbody>
                                <t t-foreach="doc.order_line" t-as="line">
                                    <tr>
                                        <td><span t-field="line.product_id.name"/></td>
                                        <td class="text-end">
                                            <span t-field="line.product_uom_qty"/>
                                        </td>
                                        <td class="text-end">
                                            <span t-field="line.price_unit"
                                                  t-options="{'widget': 'monetary',
                                                              'display_currency': doc.currency_id}"/>
                                        </td>
                                        <td class="text-end">
                                            <span t-field="line.price_subtotal"
                                                  t-options="{'widget': 'monetary',
                                                              'display_currency': doc.currency_id}"/>
                                        </td>
                                    </tr>
                                </t>
                            </tbody>
                        </table>

                        <!-- Totals -->
                        <div class="row justify-content-end">
                            <div class="col-4">
                                <table class="table table-sm">
                                    <tr>
                                        <td><strong>Total</strong></td>
                                        <td class="text-end">
                                            <span t-field="doc.amount_total"
                                                  t-options="{'widget': 'monetary',
                                                              'display_currency': doc.currency_id}"/>
                                        </td>
                                    </tr>
                                </table>
                            </div>
                        </div>
                    </div>
                </t>
            </t>
        </t>
    </template>
</odoo>
```

### 2.3 Key Layout Templates

| Template | Purpose |
|----------|---------|
| `web.html_container` | Basic HTML wrapper (doctype, head, body) |
| `web.external_layout` | Company header + footer (logo, address, page numbers) |
| `web.internal_layout` | Internal document layout (simpler header) |
| `web.basic_layout` | Minimal layout without header/footer |

**Using external_layout with company override:**

```xml
<t t-call="web.external_layout">
    <!-- Automatically includes company header, footer, and page numbers -->
    <t t-set="company" t-value="doc.company_id"/>  <!-- override company for header -->
    <div class="page">
        <!-- report content -->
    </div>
</t>
```

### 2.4 Field Rendering Options (t-field / t-options)

```xml
<!-- Date formatting -->
<span t-field="doc.date_order" t-options="{'widget': 'date'}"/>
<span t-field="doc.create_date" t-options="{'widget': 'datetime'}"/>

<!-- Monetary with currency -->
<span t-field="doc.amount_total"
      t-options="{'widget': 'monetary', 'display_currency': doc.currency_id}"/>

<!-- Image with dimensions -->
<span t-field="doc.image_128" t-options="{'widget': 'image', 'style': 'max-width:150px;'}"/>

<!-- Barcode -->
<span t-field="doc.name"
      t-options="{'widget': 'barcode', 'width': 600, 'height': 100,
                  'img_style': 'width:300px;height:50px;'}"/>

<!-- QR Code -->
<span t-field="doc.name"
      t-options="{'widget': 'barcode', 'symbology': 'QR',
                  'width': 200, 'height': 200,
                  'barLevel': 'M'}"/>

<!-- Auto-detect barcode type -->
<span t-field="doc.product_id.barcode"
      t-options="{'widget': 'barcode', 'symbology': 'auto',
                  'width': 600, 'height': 100}"/>

<!-- Text with line breaks preserved -->
<span t-field="doc.note"/>

<!-- HTML content (rendered as-is) -->
<div t-field="doc.description" class="oe_no_empty"/>

<!-- Duration -->
<span t-field="doc.duration" t-options="{'widget': 'duration', 'unit': 'hour'}"/>

<!-- Contact widget (full address block) -->
<div t-field="doc.partner_id"
     t-options='{"widget": "contact", "fields": ["address", "name", "phone", "email"],
                 "no_marker": true}'/>
```

### 2.5 Barcode Types

| Symbology | Usage |
|-----------|-------|
| `Code128` | General-purpose (default) |
| `EAN8` | 8-digit product codes |
| `EAN13` | 13-digit product codes |
| `UPCA` | 12-digit US product codes |
| `QR` | QR codes (supports barLevel: L/M/Q/H) |
| `auto` | Auto-detect from value |

```xml
<!-- Barcode from a field -->
<div t-if="doc.barcode"
     t-field="doc.barcode"
     t-options="{'widget': 'barcode', 'width': 600, 'height': 120,
                 'img_style': 'max-width:100%;', 'img_align': 'center'}"/>

<!-- Barcode from raw value -->
<img t-att-src="'/report/barcode/?barcode_type=QR&amp;value=%s&amp;width=200&amp;height=200' % doc.name"/>
```

### 2.6 Paper Format

```xml
<record id="my_paperformat" model="report.paperformat">
    <field name="name">My Custom Format</field>
    <field name="default">False</field>
    <field name="format">A4</field>           <!-- A4, Letter, Legal, etc. -->
    <field name="orientation">Portrait</field> <!-- Portrait | Landscape -->
    <field name="margin_top">40</field>        <!-- mm -->
    <field name="margin_bottom">20</field>
    <field name="margin_left">7</field>
    <field name="margin_right">7</field>
    <field name="header_line">False</field>
    <field name="header_spacing">35</field>    <!-- mm between header and content -->
    <field name="dpi">90</field>
</record>
```

**Common formats:** A0-A9, B0-B10, Letter, Legal, Tabloid, custom

### 2.7 Custom Report with Python Logic

When you need custom data processing beyond what the template can do, create an AbstractModel:

```python
# report/my_report.py
from odoo import api, models

class MyDocumentReport(models.AbstractModel):
    _name = 'report.my_module.report_my_document'
    _description = 'My Document Report'

    @api.model
    def _get_report_values(self, docids, data=None):
        docs = self.env['my.model'].browse(docids)

        # Custom computations
        totals_by_category = {}
        for doc in docs:
            for line in doc.line_ids:
                cat = line.product_id.categ_id.name
                totals_by_category.setdefault(cat, 0)
                totals_by_category[cat] += line.price_subtotal

        return {
            'doc_ids': docids,
            'doc_model': 'my.model',
            'docs': docs,
            'totals_by_category': totals_by_category,
            'company': self.env.company,
            'today': fields.Date.today(),
        }
```

**Important:** The AbstractModel `_name` must follow the pattern `report.<report_name>` where `<report_name>` matches the `report_name` field in the `ir.actions.report` XML record.

**Using custom values in template:**

```xml
<template id="report_my_document">
    <t t-call="web.html_container">
        <t t-foreach="docs" t-as="doc">
            <t t-call="web.external_layout">
                <div class="page">
                    <h2 t-field="doc.name"/>
                    <p>Report generated: <span t-esc="today"/></p>

                    <!-- Use custom computed data -->
                    <h4>Totals by Category</h4>
                    <table class="table">
                        <t t-foreach="totals_by_category.items()" t-as="cat">
                            <tr>
                                <td t-esc="cat[0]"/>
                                <td class="text-end" t-esc="cat[1]"
                                    t-options="{'widget': 'float', 'precision': 2}"/>
                            </tr>
                        </t>
                    </table>
                </div>
            </t>
        </t>
    </t>
</template>
```

### 2.8 Report Inheritance / Extension

Extend existing reports using XPath:

```xml
<template id="report_saleorder_inherit" inherit_id="sale.report_saleorder_document">
    <!-- Add a field after an existing element -->
    <xpath expr="//div[@name='payment_term']" position="after">
        <div class="col-6">
            <strong>Custom Field:</strong>
            <span t-field="doc.x_custom_field"/>
        </div>
    </xpath>

    <!-- Replace content -->
    <xpath expr="//h2" position="replace">
        <h2>
            <span t-field="doc.name"/> -
            <span class="text-muted" t-field="doc.partner_id.name"/>
        </h2>
    </xpath>

    <!-- Add inside an element -->
    <xpath expr="//table[@name='sale_order_line_table']/thead/tr" position="inside">
        <th>Custom Column</th>
    </xpath>
</template>
```

### 2.9 Triggering Reports from Python

```python
# Get report action
report = self.env.ref('my_module.action_report_my_document')

# Generate PDF content
pdf_content, content_type = self.env['ir.actions.report']._render_qweb_pdf(
    report, res_ids=[record.id])

# Generate HTML content
html_content, content_type = self.env['ir.actions.report']._render_qweb_html(
    report, res_ids=[record.id])

# Return report action from a button
def action_print_report(self):
    return self.env.ref('my_module.action_report_my_document').report_action(self)
```

### 2.10 SQL-Based Reporting Models

For dashboard/analysis reports (pivot, graph views):

```python
from odoo import fields, models, tools

class SaleReport(models.Model):
    _name = "sale.report"
    _description = "Sales Analysis Report"
    _auto = False                    # No auto-created table
    _rec_name = 'date'
    _order = 'date desc'

    name = fields.Char('Order Reference', readonly=True)
    date = fields.Datetime('Order Date', readonly=True)
    partner_id = fields.Many2one('res.partner', 'Customer', readonly=True)
    product_id = fields.Many2one('product.product', 'Product', readonly=True)
    state = fields.Selection([('draft', 'Draft'), ('sale', 'Sales Order')], readonly=True)
    price_total = fields.Float('Total', readonly=True)
    product_uom_qty = fields.Float('Qty Ordered', readonly=True)
    company_id = fields.Many2one('res.company', 'Company', readonly=True)

    def init(self):
        tools.drop_view_if_exists(self.env.cr, self._table)
        self.env.cr.execute("""
            CREATE OR REPLACE VIEW %s AS (
                SELECT
                    min(l.id) AS id,
                    l.product_id AS product_id,
                    s.name AS name,
                    s.date_order AS date,
                    s.partner_id AS partner_id,
                    s.state AS state,
                    sum(l.price_total) AS price_total,
                    sum(l.product_uom_qty) AS product_uom_qty,
                    s.company_id AS company_id
                FROM sale_order_line l
                JOIN sale_order s ON s.id = l.order_id
                GROUP BY l.product_id, s.name, s.date_order, s.partner_id,
                         s.state, s.company_id
            )
        """ % self._table)
```

### 2.11 Report CSS/Styling

```xml
<template id="report_my_document">
    <t t-call="web.html_container">
        <t t-foreach="docs" t-as="doc">
            <t t-call="web.external_layout">
                <div class="page">
                    <!-- Inline styles for PDF (wkhtmltopdf doesn't load external CSS well) -->
                    <style>
                        .report-header { border-bottom: 2px solid #333; padding-bottom: 10px; }
                        .line-even { background-color: #f8f9fa; }
                        .total-row { font-weight: bold; border-top: 2px solid #333; }
                        @media print {
                            .page-break { page-break-before: always; }
                        }
                    </style>

                    <div class="report-header">
                        <h2 t-field="doc.name"/>
                    </div>

                    <!-- Use Bootstrap classes (available in reports) -->
                    <table class="table table-sm table-bordered mt-3">
                        <thead class="table-light">
                            <tr><th>Item</th><th class="text-end">Amount</th></tr>
                        </thead>
                        <tbody>
                            <tr t-foreach="doc.line_ids" t-as="line"
                                t-attf-class="{{ 'line-even' if line_even else '' }}">
                                <td t-field="line.name"/>
                                <td class="text-end" t-field="line.price_subtotal"
                                    t-options="{'widget': 'monetary',
                                                'display_currency': doc.currency_id}"/>
                            </tr>
                        </tbody>
                    </table>

                    <!-- Force page break between docs -->
                    <div t-if="not doc_last" class="page-break"/>
                </div>
            </t>
        </t>
    </t>
</template>
```

---

## 3. Multi-Company

### 3.1 Making a Model Multi-Company Aware

```python
class MyModel(models.Model):
    _name = 'my.model'
    _check_company_auto = True     # Auto-validate company consistency

    company_id = fields.Many2one(
        'res.company', string='Company',
        required=True, default=lambda self: self.env.company)

    # Relational fields with check_company=True enforce same-company constraint
    partner_id = fields.Many2one('res.partner', check_company=True)
    warehouse_id = fields.Many2one('stock.warehouse', check_company=True)
    product_id = fields.Many2one('product.product')  # No check (products are often shared)
```

**How `_check_company_auto` works:** When `True`, the ORM automatically validates that all relational fields marked with `check_company=True` belong to the same company (or have no company set).

### 3.2 Multi-Company Record Rules

```xml
<odoo noupdate="1">
    <!-- Standard pattern: records visible if company matches or is empty -->
    <record id="my_model_company_rule" model="ir.rule">
        <field name="name">My Model: multi-company</field>
        <field name="model_id" ref="model_my_model"/>
        <field name="domain_force">[('company_id', 'in', company_ids)]</field>
    </record>

    <!-- Allow shared records (company_id = False) -->
    <record id="my_model_company_rule" model="ir.rule">
        <field name="name">My Model: multi-company</field>
        <field name="model_id" ref="model_my_model"/>
        <field name="domain_force">[('company_id', 'in', company_ids + [False])]</field>
    </record>

    <!-- Hierarchical: parent company can see child company records -->
    <record id="my_model_company_rule" model="ir.rule">
        <field name="name">My Model: multi-company (parent_of)</field>
        <field name="model_id" ref="model_my_model"/>
        <field name="domain_force">['|', ('company_id', '=', False),
                                        ('company_id', 'parent_of', company_ids)]</field>
    </record>

    <!-- Group-specific rules -->
    <record id="my_model_rule_portal" model="ir.rule">
        <field name="name">Portal: own records only</field>
        <field name="model_id" ref="model_my_model"/>
        <field name="domain_force">[('partner_id', '=', user.partner_id.id)]</field>
        <field name="groups" eval="[(4, ref('base.group_portal'))]"/>
    </record>

    <!-- Manager sees everything -->
    <record id="my_model_rule_manager" model="ir.rule">
        <field name="name">Manager: all records</field>
        <field name="model_id" ref="model_my_model"/>
        <field name="domain_force">[(1, '=', 1)]</field>
        <field name="groups" eval="[(4, ref('my_module.group_manager'))]"/>
    </record>
</odoo>
```

### 3.3 Company-Dependent Fields

Store different values per company in a single field (backed by JSONB):

```python
class ProductTemplate(models.Model):
    _inherit = 'product.template'

    # Each company sees its own value for these fields
    property_account_income_id = fields.Many2one(
        'account.account', company_dependent=True, check_company=True)
    property_stock_production = fields.Many2one(
        'stock.location', company_dependent=True, check_company=True)
    standard_price = fields.Float(company_dependent=True)
    responsible_id = fields.Many2one('res.users', company_dependent=True)
```

**Allowed types:** `char`, `float`, `boolean`, `integer`, `text`, `many2one`, `date`, `datetime`, `selection`, `html`

**Restrictions:** Cannot be `required=True` or `translate=True`.

### 3.4 `with_company()` Usage

```python
# Switch company context for a computation
order = order.with_company(order.company_id)

# Get company-specific property values
partner_account = partner.with_company(company_id).property_account_receivable_id

# Generate company-specific sequences
seq = self.env['ir.sequence'].with_company(company_id).next_by_code('sale.order')

# Create records in a specific company context
invoice = self.env['account.move'].with_company(vals['company_id']).create(vals)
```

### 3.5 `_check_company_domain` Override

Customize how company filtering works for a model:

```python
class ResPartner(models.Model):
    _inherit = 'res.partner'
    # Use parent_of: a parent company can access child company partners
    _check_company_domain = models.check_company_domain_parent_of

class ResUsers(models.Model):
    _inherit = 'res.users'
    # Custom domain: user must have the company in their allowed companies
    def _check_company_domain(self, companies):
        if not companies:
            return []
        return [('company_ids', 'in', models.to_company_ids(companies))]
```

### 3.6 Multi-Company Best Practices

1. **Always add `company_id`** to models that should be company-scoped
2. **Use `check_company=True`** on relational fields pointing to company-scoped models
3. **Set `_check_company_auto = True`** to enable automatic validation
4. **Create `ir.rule`** records with `company_ids` domain for access control
5. **Use `with_company()`** when creating records or accessing company-dependent fields in a different company context
6. **Default company:** `default=lambda self: self.env.company`
7. **Allow shared records** by making `company_id` optional and including `False` in domain rules
8. **Use `parent_of`** in rules when parent companies should see child company data
9. **Multi-currency:** Use `self.env['res.currency']._get_simple_currency_table(self.env.companies)` for SQL reports
