---
name: odoo-18-dev
description: Comprehensive Odoo 18 development guide covering project structure, ORM patterns, OWL frontend, legacy JS, controllers, webhooks, cron jobs, Bootstrap 5, testing, and commit conventions. Use when developing, reviewing, or debugging Odoo 18 modules.
---

# Odoo 18 Development Guide

## 1. Project Structure

### Top-Level Layout

```
odoo/
├── odoo/                    # Core framework (ORM, fields, API, base module)
│   ├── models.py            # ORM base classes
│   ├── fields.py            # Field definitions
│   ├── api.py               # Decorators (@depends, @constrains, @onchange)
│   └── addons/base/         # Foundational models (ir.model, res.partner, res.company)
├── addons/                  # Community & enterprise modules
└── setup/                   # Installation scripts
```

### Module Structure

```
my_module/
├── __init__.py
├── __manifest__.py
├── models/
│   ├── __init__.py
│   └── my_model.py
├── views/
│   ├── my_model_views.xml
│   └── menus.xml
├── controllers/
│   ├── __init__.py
│   └── main.py
├── security/
│   ├── ir.model.access.csv
│   ├── ir_rules.xml
│   └── res_groups.xml
├── data/
│   ├── ir_sequence_data.xml
│   └── ir_cron_data.xml
├── demo/
│   └── demo_data.xml
├── wizard/
│   ├── __init__.py
│   ├── my_wizard.py
│   └── my_wizard_views.xml
├── report/
│   ├── report_templates.xml
│   └── ir_actions_report.xml
├── static/
│   ├── src/
│   │   ├── js/
│   │   ├── xml/
│   │   └── scss/
│   └── tests/
├── i18n/
├── tests/
│   ├── __init__.py
│   └── test_my_model.py
└── migrations/
```

### `__manifest__.py`

```python
{
    'name': 'My Module',
    'version': '18.0.1.0.0',
    'category': 'Sales/Sales',
    'summary': 'Short description',
    'description': """Long description""",
    'depends': ['sale', 'account'],
    'data': [
        'security/ir.model.access.csv',
        'security/ir_rules.xml',
        'data/ir_sequence_data.xml',
        'wizard/wizard_views.xml',
        'views/my_model_views.xml',
        'views/menus.xml',           # menus last
    ],
    'demo': ['demo/demo_data.xml'],
    'assets': {
        'web.assets_backend': [
            'my_module/static/src/js/**/*',
            'my_module/static/src/xml/**/*',
            'my_module/static/src/scss/**/*',
        ],
        'web.assets_frontend': [
            'my_module/static/src/scss/portal.scss',
        ],
        'web.assets_tests': ['my_module/static/tests/tours/**/*'],
        'web.qunit_suite_tests': ['my_module/static/tests/**/*.test.js'],
    },
    'installable': True,
    'application': True,
    'auto_install': False,
    'license': 'LGPL-3',
    'pre_init_hook': 'pre_init',
    'post_init_hook': '_post_init_hook',
    'uninstall_hook': 'uninstall_hook',
}
```

### Asset Bundle Directives

```python
'assets': {
    'web.assets_backend': [
        ('include', 'web._assets_helpers'),          # include another bundle
        'my_module/static/src/**/*',                  # glob pattern
        ('remove', 'my_module/static/src/excluded.js'), # exclude file
        ('after', 'web/static/src/file.scss', 'my_module/static/src/override.scss'), # insert after
    ],
}
```

### Security

**ir.model.access.csv:**
```csv
id,name,model_id:id,group_id:id,perm_read,perm_write,perm_create,perm_unlink
access_my_model_user,my.model.user,model_my_model,base.group_user,1,1,1,0
access_my_model_manager,my.model.manager,model_my_model,my_module.group_manager,1,1,1,1
```

**ir_rules.xml (domain-based access):**
```xml
<odoo noupdate="1">
    <record id="my_model_comp_rule" model="ir.rule">
        <field name="name">Multi-company</field>
        <field name="model_id" ref="model_my_model"/>
        <field name="domain_force">[('company_id', 'in', company_ids)]</field>
    </record>
    <record id="my_model_personal_rule" model="ir.rule">
        <field name="name">Personal records</field>
        <field name="model_id" ref="model_my_model"/>
        <field name="domain_force">['|',('user_id','=',user.id),('user_id','=',False)]</field>
        <field name="groups" eval="[(4, ref('base.group_user'))]"/>
    </record>
</odoo>
```

---

## 2. ORM Patterns

### Model Types

```python
class MyModel(models.Model):          # Persistent database table
    _name = 'my.model'

class MyWizard(models.TransientModel): # Temporary (auto-cleaned)
    _name = 'my.wizard'

class MyMixin(models.AbstractModel):   # No table, used for inheritance
    _name = 'my.mixin'
```

### Model Attributes

```python
class SaleOrder(models.Model):
    _name = 'sale.order'
    _inherit = ['portal.mixin', 'mail.thread', 'mail.activity.mixin']
    _description = "Sales Order"
    _order = 'date_order desc, id desc'
    _rec_name = 'name'
    _rec_names_search = ['name', 'partner_id.name']
    _check_company_auto = True
```

### Inheritance

```python
# Extend existing model (add fields/methods)
class SaleOrder(models.Model):
    _inherit = 'sale.order'
    custom_field = fields.Char()

# Multiple mixin inheritance
class SaleOrder(models.Model):
    _inherit = ['sale.order', 'utm.mixin']

# Delegation inheritance (composition)
class Child(models.Model):
    _inherits = {'parent.model': 'parent_id'}
    parent_id = fields.Many2one('parent.model', required=True, ondelete='cascade')
```

### Field Types

```python
# Basic
name = fields.Char(string="Name", required=True, index='trigram', translate=True)
active = fields.Boolean(default=True)
sequence = fields.Integer(default=10)
amount = fields.Float(digits=(16, 2))
amount_total = fields.Monetary(currency_field='currency_id')
description = fields.Text()
notes = fields.Html(sanitize=True)
date = fields.Date()
create_date = fields.Datetime(readonly=True)
state = fields.Selection([('draft', 'Draft'), ('done', 'Done')], default='draft')
image = fields.Image(max_width=1024, max_height=1024)
file = fields.Binary(attachment=True)
properties = fields.PropertiesDefinition("Properties")

# Relational
partner_id = fields.Many2one('res.partner', string="Customer", required=True,
                              ondelete='restrict', domain="[('is_company', '=', True)]")
line_ids = fields.One2many('sale.order.line', 'order_id', string="Lines", copy=True)
tag_ids = fields.Many2many('crm.tag', relation='sale_order_tag_rel',
                            column1='order_id', column2='tag_id')

# Computed
total = fields.Monetary(compute='_compute_total', store=True, readonly=False, precompute=True)

# Related (shortcut)
partner_name = fields.Char(related='partner_id.name', store=True)
```

### Decorators

```python
@api.depends('line_ids.price_total')
def _compute_total(self):
    for record in self:
        record.total = sum(record.line_ids.mapped('price_total'))

@api.constrains('date_start', 'date_end')
def _check_dates(self):
    for record in self:
        if record.date_start > record.date_end:
            raise ValidationError("End date must be after start date.")

@api.onchange('partner_id')
def _onchange_partner_id(self):
    self.payment_term_id = self.partner_id.property_payment_term_id

@api.ondelete(at_uninstall=False)
def _unlink_except_confirmed(self):
    if any(rec.state == 'done' for rec in self):
        raise UserError("Cannot delete confirmed records.")

@api.model
def default_get(self, fields_list):
    defaults = super().default_get(fields_list)
    defaults['company_id'] = self.env.company.id
    return defaults

@api.model_create_multi
def create(self, vals_list):
    for vals in vals_list:
        if vals.get('name', _("New")) == _("New"):
            vals['name'] = self.env['ir.sequence'].next_by_code('my.model')
    return super().create(vals_list)
```

### SQL Constraints

```python
_sql_constraints = [
    ('name_unique', 'UNIQUE(name, company_id)', "Name must be unique per company."),
    ('positive_amount', 'CHECK(amount >= 0)', "Amount must be positive."),
]
```

### CRUD Overrides

```python
@api.model_create_multi
def create(self, vals_list):
    records = super().create(vals_list)
    records._post_create_hook()
    return records

def write(self, vals):
    if 'state' in vals:
        self._check_state_transition(vals['state'])
    return super().write(vals)

def unlink(self):
    self._cleanup_related()
    return super().unlink()

def copy(self, default=None):
    default = dict(default or {}, name=_("%s (copy)", self.name))
    return super().copy(default)
```

### Recordset Operations

```python
# Filtering
draft_orders = orders.filtered(lambda o: o.state == 'draft')
draft_orders = orders.filtered_domain([('state', '=', 'draft')])

# Mapping
partner_ids = orders.mapped('partner_id.id')
names = orders.mapped('name')

# Sorting
sorted_orders = orders.sorted(key=lambda o: o.date_order, reverse=True)

# Set operations
all_records = recordset1 | recordset2   # union
common = recordset1 & recordset2        # intersection
diff = recordset1 - recordset2          # difference

# Search
results = self.env['sale.order'].search([
    ('state', '=', 'sale'),
    ('partner_id', 'child_of', parent_id),
    '|', ('user_id', '=', uid), ('team_id', '=', team_id),
], limit=10, order='date_order desc')

count = self.env['sale.order'].search_count(domain)
```

### Environment & Context

```python
self.env['res.partner']              # access model
self.env.user                        # current user
self.env.company                     # current company
self.env.companies                   # allowed companies
self.env.ref('module.xml_id')        # get record by XML ID
self.env.cr                          # database cursor
self.env.context                     # context dict

# Context manipulation
record.with_context(lang='fr_FR', active_test=False)
record.with_company(company_id)
record.with_user(user_id)
record.sudo()                        # bypass access rights

# Direct SQL (use sparingly)
self.env.cr.execute("SELECT id FROM my_table WHERE x = %s", (value,))
rows = self.env.cr.fetchall()
self.env.invalidate_all()            # clear ORM cache after raw SQL
```

### Command Objects (for One2many/Many2many writes)

```python
from odoo.fields import Command

record.line_ids = [
    Command.create({'product_id': 1, 'qty': 5}),     # (0, 0, vals)
    Command.update(line_id, {'qty': 10}),             # (1, id, vals)
    Command.delete(line_id),                          # (2, id, 0)
    Command.unlink(line_id),                          # (3, id, 0)
    Command.link(existing_id),                        # (4, id, 0)
    Command.clear(),                                  # (5, 0, 0)
    Command.set([id1, id2, id3]),                     # (6, 0, [ids])
]
```

---

## 3. OWL Frontend (In-Depth)

### Component Definition

```javascript
import { Component, useState, useRef, onMounted, onWillStart, useEffect, onWillUnmount } from "@odoo/owl";
import { useService } from "@web/core/utils/hooks";
import { registry } from "@web/core/registry";

export class MyComponent extends Component {
    static template = "my_module.MyComponent";
    static components = { ChildComponent };
    static props = {
        title: { type: String, optional: true },
        items: Array,
        onSave: Function,
        config: {
            type: Object,
            shape: { id: Number, name: { type: String, optional: true } },
            optional: true,
        },
    };
    static defaultProps = { title: "Default" };

    setup() {
        this.state = useState({ count: 0, isOpen: false });
        this.rootRef = useRef("root");
        this.orm = useService("orm");
        this.notification = useService("notification");
        this.dialog = useService("dialog");
        this.action = useService("action");

        onWillStart(async () => {
            this.data = await this.orm.searchRead("res.partner", [], ["name"]);
        });

        onMounted(() => {
            this.rootRef.el?.focus();
        });

        useEffect(
            (el) => {
                if (el) { /* side effect */ }
                return () => { /* cleanup */ };
            },
            () => [this.rootRef.el]
        );

        onWillUnmount(() => {
            clearTimeout(this.timeout);
        });
    }

    async onClick() {
        this.state.count++;
        await this.orm.call("res.partner", "my_method", [1], { key: "value" });
        this.notification.add("Done!", { type: "success" });
    }
}
```

### Component Inheritance

```javascript
import { CogMenu } from "@web/search/cog_menu/cog_menu";

export class FormCogMenu extends CogMenu {
    static template = "web.FormCogMenu";
    static components = { ...CogMenu.components, ExtraItem };
    static props = { ...CogMenu.props, slots: { type: Object, optional: true } };
}
```

### Patching (Monkey-Patching Existing Components)

```javascript
import { patch } from "@web/core/utils/patch";
import { SomeComponent } from "@module/path/to/component";

patch(SomeComponent.prototype, {
    setup() {
        super.setup();
        // additional setup
    },
    myMethod() {
        const result = super.myMethod();
        // extend behavior
        return result;
    },
});
```

### Hooks Reference

```javascript
// State management
this.state = useState({ key: "value" });

// DOM references (use with t-ref="name" in template)
this.inputRef = useRef("myInput");  // access via this.inputRef.el

// Lifecycle
onWillStart(async () => { /* async init before first render */ });
onMounted(() => { /* DOM is ready */ });
onWillUnmount(() => { /* cleanup */ });

// Reactive side effects
useEffect(
    (dep1, dep2) => { /* runs when deps change */ return () => { /* cleanup */ }; },
    () => [this.state.value, this.props.id]  // dependency array
);

// Service injection
const orm = useService("orm");
const dialog = useService("dialog");
const notification = useService("notification");
const action = useService("action");
const ui = useService("ui");

// Event bus subscription
import { useBus } from "@web/core/utils/hooks";
useBus(this.env.bus, "event-name", (ev) => { /* handler */ });

// Autofocus (focuses element with t-ref="autofocus")
import { useAutofocus } from "@web/core/utils/hooks";
useAutofocus({ refName: "autofocus", selectAll: true });
```

### QWeb Template Syntax

```xml
<?xml version="1.0" encoding="UTF-8"?>
<templates xml:space="preserve">
    <t t-name="my_module.MyComponent">
        <div class="o_my_component" t-ref="root">
            <!-- Conditional -->
            <div t-if="state.isOpen">Open</div>
            <div t-elif="state.count > 0">Has count</div>
            <div t-else="">Empty</div>

            <!-- Loops -->
            <div t-foreach="props.items" t-as="item" t-key="item.id">
                <span t-esc="item.name"/>           <!-- escaped text -->
                <span t-out="item.htmlContent"/>     <!-- raw HTML -->
            </div>

            <!-- Dynamic attributes -->
            <div t-att-class="{ 'active': state.isOpen, 'disabled': !props.enabled }"/>
            <div t-attf-class="btn btn-{{ props.variant }}"/>
            <div t-attf-style="color: {{ state.color }}"/>

            <!-- Events -->
            <button t-on-click="onClick">Click</button>
            <button t-on-click="() => this.doSomething(item.id)">With args</button>

            <!-- Child components -->
            <ChildComponent title="'Hello'" items="state.items" onSave.bind="save"/>

            <!-- Slots -->
            <t t-slot="default"/>
            <t t-set-slot="header">
                <h1>Custom Header</h1>
            </t>

            <!-- Call sub-template -->
            <t t-call="my_module.SubTemplate">
                <t t-set="variable" t-value="someValue"/>
            </t>

            <!-- Component reference -->
            <input t-ref="myInput"/>
        </div>
    </t>
</templates>
```

### Registry Patterns

```javascript
import { registry } from "@web/core/registry";

// Register views
registry.category("views").add("my_view", myViewDefinition);

// Register fields
registry.category("fields").add("my_widget", { component: MyFieldComponent, ... });

// Register actions
registry.category("actions").add("my_action", MyActionComponent);

// Register systray items
registry.category("systray").add("my_item", { Component: MySystrayItem }, { sequence: 50 });

// Register main components (auto-mounted in webclient)
registry.category("main_components").add("MyOverlay", { Component: MyOverlay, props: {} });
```

### Service Definition

```javascript
export const myService = {
    dependencies: ["orm", "notification"],

    start(env, { orm, notification }) {
        let cache = {};

        function doSomething(id) {
            // implementation
        }

        async function fetchData(model, domain) {
            return orm.searchRead(model, domain, ["name"]);
        }

        return { doSomething, fetchData };
    },
};

registry.category("services").add("myService", myService);

// Usage in component:
// const myService = useService("myService");
// myService.doSomething(42);
```

### ORM Service (Frontend to Backend)

```javascript
const orm = useService("orm");

// CRUD
await orm.create("res.partner", [{ name: "John" }]);
await orm.read("res.partner", [1, 2], ["name", "email"]);
await orm.write("res.partner", [1], { name: "Jane" });
await orm.unlink("res.partner", [1]);

// Search
await orm.search("res.partner", [["is_company", "=", true]]);
await orm.searchRead("res.partner", [["state", "=", "active"]], ["name"], { limit: 10 });
await orm.webSearchRead("res.partner", { domain: [], specification: { name: {} } });

// Aggregation
await orm.readGroup("sale.order", [["state", "=", "sale"]], ["amount_total"], ["partner_id"]);

// Custom method call
await orm.call("res.partner", "action_archive", [[1, 2, 3]]);
await orm.call("sale.order", "action_confirm", [[orderId]], { context: { send_email: true } });

// Silent mode (no error notifications)
await orm.silent.read("res.partner", [1]);
```

### Low-Level RPC

```javascript
import { rpc } from "@web/core/network/rpc";

const result = await rpc("/my/controller/route", { param1: "value" });
```

### x2Many Commands (Frontend)

```javascript
import { x2ManyCommands } from "@web/core/orm_service";

x2ManyCommands.create(virtualId, { name: "New" });  // [0, id, vals]
x2ManyCommands.update(id, { name: "Updated" });     // [1, id, vals]
x2ManyCommands.delete(id);                           // [2, id, 0]
x2ManyCommands.unlink(id);                           // [3, id, 0]
x2ManyCommands.link(id);                             // [4, id, 0]
x2ManyCommands.clear();                              // [5, 0, 0]
x2ManyCommands.set([1, 2, 3]);                       // [6, 0, [ids]]
```

---

## 4. JS Beyond OWL

### POS (Point of Sale) Patterns

POS runs as a standalone OWL app with its own asset bundle.

**POS Component:**
```javascript
import { Component, useState, useEffect } from "@odoo/owl";
import { usePos } from "@point_of_sale/app/store/pos_hook";
import { useTrackedAsync } from "@point_of_sale/app/utils/hooks";
import { useService } from "@web/core/utils/hooks";

export class ProductInfoBanner extends Component {
    static template = "point_of_sale.ProductInfoBanner";
    static components = { AccordionItem };
    static props = {
        product: Object,
        info: { type: Object, optional: true },
    };

    setup() {
        this.pos = usePos();                      // POS-specific hook for store access
        this.ui = useState(useService("ui"));
        this.state = useState({ available_quantity: 0 });
        this.fetchStock = useTrackedAsync(
            (p) => this.pos.getProductInfo(p, 1),
            { keepLast: true }
        );

        useEffect(
            () => { this.fetchStock.call(this.props.product); },
            () => [this.props.product]
        );
    }
}
```

**POS Asset Bundle (`__manifest__.py`):**
```python
'point_of_sale._assets_pos': [
    'web/static/src/module_loader.js',
    'web/static/lib/owl/owl.js',
    'point_of_sale/static/lib/**/*',
    'point_of_sale/static/src/**/*',
    ('remove', 'point_of_sale/static/src/backend/**/*'),
    ('remove', 'point_of_sale/static/src/app/main.js'),
],
'point_of_sale.assets_prod': [
    ('include', 'point_of_sale._assets_pos'),
    'point_of_sale/static/src/app/main.js',
],
```

**Patching POS Services:**
```javascript
import { patch } from "@web/core/utils/patch";
import { AccountMoveService } from "@account/services/account_move_service";

patch(AccountMoveService.prototype, {
    async downloadPdf(accountMoveId) {
        if (isIosApp()) {
            return this.action.doAction({
                type: "ir.actions.act_url",
                url: `/account/download_invoice_documents/${accountMoveId}/pdf`,
            });
        }
        return super.downloadPdf(accountMoveId);
    },
});
```

### Public/Website JS

**Public Widget (Legacy Pattern - still used on website frontend):**
```javascript
import { publicWidget } from "@web/legacy/js/public/public_widget";

const MyPublicWidget = publicWidget.Widget.extend({
    selector: ".my-element",       // CSS selector to auto-bind
    events: {
        'click .btn': '_onClickBtn',
    },

    start() {
        this._super(...arguments);
        // DOM is ready
    },

    _onClickBtn(ev) {
        ev.preventDefault();
        this._rpc('/my/route', { param: 'value' }).then((result) => {
            this.$el.html(result);
        });
    },

    destroy() {
        this._super(...arguments);
    },
});

publicWidget.registry.MyWidget = MyPublicWidget;
```

**Public Components (OWL on website frontend):**
```javascript
import { registry } from "@web/core/registry";

// Register OWL component for public pages
registry.category("public_components").add("my_widget", {
    Component: MyPublicOWLComponent,
    selector: ".my-owl-widget",
});
```

**Website Systray Items:**
```javascript
import { registry } from "@web/core/registry";

export const systrayItem = {
    Component: EditWebsiteSystray,
    isDisplayed: (env) => env.services.website.isRestrictedEditor,
};

registry.category("website_systray").add("EditWebsite", systrayItem, { sequence: 7 });
```

### `patch()` Utility (Monkey-Patching)

```javascript
import { patch } from "@web/core/utils/patch";
import { FormController } from "@web/views/form/form_controller";

patch(FormController.prototype, {
    setup() {
        super.setup();
        // extend setup
        this.customState = useState({ extra: true });
    },

    async onRecordSaved(record) {
        await super.onRecordSaved(record);
        // additional logic after save
    },
});
```

---

## 5. Controllers & Webhooks

### HTTP Controller Definition

```python
from odoo import http
from odoo.http import request, Controller

class MyController(Controller):

    @http.route('/my/page', type='http', auth='user', website=True, methods=['GET'])
    def my_page(self, page=1, **kw):
        records = request.env['my.model'].search([], limit=10)
        return request.render('my_module.my_template', {
            'records': records,
            'page': page,
        })

    @http.route('/my/data', type='json', auth='user', methods=['POST'])
    def get_data(self, model, domain=None, **kw):
        records = request.env[model].search(domain or [])
        return records.read(['name', 'state'])

    @http.route('/my/download/<int:record_id>', type='http', auth='user')
    def download(self, record_id, **kw):
        record = request.env['my.model'].browse(record_id)
        content = record.generate_report()
        return request.make_response(content, headers=[
            ('Content-Type', 'application/pdf'),
            ('Content-Disposition', f'attachment; filename="report.pdf"'),
        ])

    @http.route('/my/json-response', type='http', auth='public')
    def json_response(self, **kw):
        data = {'status': 'ok', 'count': 42}
        return request.make_json_response(data, status=200)
```

### Route Decorator Parameters

```python
@http.route(
    '/path/<string:param>/<int:id>',   # URL with converters
    type='http',          # 'http' (HTML/files) or 'json' (JSON-RPC)
    auth='public',        # 'public' (no login), 'user' (login required), 'none' (internal)
    methods=['GET','POST'],
    csrf=True,            # CSRF protection (disable with False for webhooks)
    website=True,         # Enable website context (lang, theme)
    sitemap=True,         # Include in sitemap
    readonly=True,        # Mark as read-only (optimization)
    multilang=True,       # Language prefix in URL
)
```

### Request Object

```python
request.env['model.name']                  # ORM access (with user's rights)
request.env['model.name'].sudo()           # bypass access rights
request.env.user                           # current user
request.env.company                        # current company
request.httprequest.method                 # GET, POST, etc.
request.httprequest.headers.get('X-Key')   # HTTP headers
request.httprequest.data                   # raw body bytes
request.get_json_data()                    # parse JSON body
request.params                             # query string + form params
request.redirect('/target/url')            # HTTP redirect
request.render('template.name', values)    # render QWeb template
request.make_response(content, headers)    # raw response
request.make_json_response(data, status)   # JSON response
request.session                            # session data
```

### Webhook Pattern

```python
class PaymentWebhook(Controller):

    @http.route('/payment/webhook', type='http', auth='public',
                methods=['POST'], csrf=False)
    def handle_webhook(self, **kw):
        # 1. Get raw data
        raw_data = request.httprequest.data
        notification = request.get_json_data()

        # 2. Verify signature
        received_sig = request.httprequest.headers.get('X-Signature')
        expected_sig = hmac.new(secret, raw_data, hashlib.sha256).hexdigest()
        if not hmac.compare_digest(received_sig, expected_sig):
            raise ValidationError("Invalid signature")

        # 3. Process & acknowledge
        try:
            tx = request.env['payment.transaction'].sudo()._get_tx_from_notification(notification)
            tx._handle_notification_data(notification)
        except Exception:
            _logger.exception("Error processing webhook")

        # 4. Always return 200 to avoid retries
        return request.make_json_response({'status': 'ok'})
```

### Portal Controller Pattern

```python
from odoo.addons.portal.controllers.portal import CustomerPortal

class MyPortal(CustomerPortal):

    @http.route('/my/records', type='http', auth='user', website=True)
    def portal_my_records(self, page=1, sortby=None, **kw):
        partner = request.env.user.partner_id
        domain = [('partner_id', '=', partner.id)]
        records = request.env['my.model'].search(domain, limit=20)
        return request.render('my_module.portal_records', {
            'records': records,
            'page_name': 'my_records',
        })
```

---

## 6. Cron Jobs

### XML Definition

```xml
<odoo noupdate="1">
    <record id="ir_cron_my_task" model="ir.cron">
        <field name="name">My Module: Process Queue</field>
        <field name="model_id" ref="model_my_model"/>
        <field name="state">code</field>
        <field name="code">model._cron_process_queue(batch_size=500)</field>
        <field name="interval_number">1</field>
        <field name="interval_type">hours</field>  <!-- minutes|hours|days|weeks|months -->
        <field name="user_id" ref="base.user_root"/>
        <field name="active" eval="True"/>
        <field name="priority">10</field>
    </record>

    <!-- Daily cleanup -->
    <record id="ir_cron_cleanup" model="ir.cron">
        <field name="name">My Module: Daily Cleanup</field>
        <field name="model_id" ref="model_my_model"/>
        <field name="state">code</field>
        <field name="code">model._gc_old_records(max_age_days=180)</field>
        <field name="interval_number">1</field>
        <field name="interval_type">days</field>
        <field name="active" eval="True"/>
    </record>

    <!-- 5-minute polling -->
    <record id="ir_cron_fetch" model="ir.cron">
        <field name="name">My Module: Fetch External Data</field>
        <field name="model_id" ref="model_my_model"/>
        <field name="state">code</field>
        <field name="code">model._fetch_external_data()</field>
        <field name="interval_number">5</field>
        <field name="interval_type">minutes</field>
        <field name="active" eval="False"/>
    </record>
</odoo>
```

### Python Methods for Cron

```python
class MyModel(models.Model):
    _name = 'my.model'

    def _cron_process_queue(self, batch_size=500):
        """Called by ir.cron. Processes pending records in batches."""
        records = self.search([('state', '=', 'pending')], limit=batch_size)
        for record in records:
            try:
                record._process()
                record.state = 'done'
            except Exception:
                record.state = 'error'
                _logger.exception("Failed to process record %s", record.id)
        # Commit periodically for long-running jobs
        if len(records) == batch_size:
            self.env.cr.commit()

    def _gc_old_records(self, max_age_days=180):
        """Garbage-collect old records."""
        cutoff = fields.Datetime.now() - timedelta(days=max_age_days)
        self.search([('create_date', '<', cutoff)]).unlink()
```

---

## 7. Bootstrap 5 Integration

### Overview

Odoo 18 uses Bootstrap 5 with significant customizations. The SCSS import chain:

1. `web/static/src/scss/bootstrap_overridden.scss` - Odoo variable overrides
2. `web/static/lib/bootstrap/scss/_variables.scss` - Bootstrap defaults
3. `web/static/src/scss/import_bootstrap.scss` - Selective Bootstrap imports

### Key Customizations

```scss
// Odoo maps its brand colors to Bootstrap
$primary: $o-brand-primary !default;
$success: $o-success !default;
$info: $o-info !default;
$warning: $o-warning !default;
$danger: $o-danger !default;

// BS4 color palette restored for backwards compatibility
$blue: #007bff !default;
$green: #28a745 !default;

// Disabled features
$enable-dark-mode: false !default;      // Odoo has its own dark mode
$enable-smooth-scroll: false !default;

// Enabled features
$enable-negative-margins: true !default;
$enable-cssgrid: true !default;
```

### Frontend vs Backend Overrides

- **Backend** (`bootstrap_overridden.scss`): Odoo brand colors, custom contrast ratios
- **Frontend** (`bootstrap_overridden_frontend.scss`): White backgrounds, website-specific table/form styling

### Using Bootstrap in Odoo Templates

```xml
<!-- Grid -->
<div class="container">
    <div class="row g-3">
        <div class="col-md-6 col-lg-4">Content</div>
    </div>
</div>

<!-- Cards -->
<div class="card shadow-sm">
    <div class="card-body">
        <h5 class="card-title" t-esc="record.name"/>
    </div>
</div>

<!-- Buttons (use Odoo utility classes too) -->
<button class="btn btn-primary">Action</button>
<button class="btn btn-secondary btn-sm">Small</button>

<!-- Alerts -->
<div class="alert alert-warning" role="alert">Warning message</div>

<!-- Utilities -->
<div class="d-flex justify-content-between align-items-center gap-2 mt-3 px-2">
    <span class="text-muted small text-truncate">Label</span>
    <span class="badge bg-success rounded-pill">Active</span>
</div>
```

### SCSS in Custom Modules

```scss
// my_module/static/src/scss/my_styles.scss

// Use Odoo variables (available via _assets_helpers bundle)
.o_my_component {
    color: $o-brand-primary;
    background: $o-view-background-color;
    border: 1px solid $border-color;     // Bootstrap variable
    border-radius: $border-radius;
    padding: map-get($spacers, 3);       // Bootstrap spacer

    &.active {
        background-color: rgba($primary, 0.1);
    }

    @include media-breakpoint-down(md) {
        padding: map-get($spacers, 2);
    }
}
```

---

## 8. Testing

### Python Tests

**Test Base Classes:**

```python
from odoo.tests import TransactionCase, HttpCase, tagged, Form

@tagged('post_install', '-at_install')
class TestMyModel(TransactionCase):

    @classmethod
    def setUpClass(cls):
        super().setUpClass()
        cls.partner = cls.env['res.partner'].create({'name': 'Test Partner'})
        cls.product = cls.env['product.product'].create({
            'name': 'Test Product', 'list_price': 100,
        })

    def test_create_order(self):
        order = self.env['sale.order'].create({
            'partner_id': self.partner.id,
        })
        self.assertEqual(order.state, 'draft')
        self.assertRecordValues(order, [{'state': 'draft', 'partner_id': self.partner.id}])

    def test_compute_total(self):
        order = self.env['sale.order'].create({
            'partner_id': self.partner.id,
            'order_line': [Command.create({
                'product_id': self.product.id,
                'product_uom_qty': 2,
            })],
        })
        self.assertAlmostEqual(order.amount_total, 200.0, places=2)

    def test_constraint(self):
        with self.assertRaises(ValidationError):
            self.env['my.model'].create({'amount': -1})

    def test_form_view(self):
        """Test using Form helper for onchange simulation."""
        with Form(self.env['sale.order']) as form:
            form.partner_id = self.partner
            # onchange fires automatically
            self.assertTrue(form.pricelist_id)
```

**HTTP Tests:**

```python
@tagged('post_install', '-at_install')
class TestMyController(HttpCase):

    def test_portal_page(self):
        self.authenticate('demo', 'demo')
        response = self.url_open('/my/records')
        self.assertEqual(response.status_code, 200)

    def test_json_endpoint(self):
        self.authenticate('admin', 'admin')
        response = self.url_open_json('/my/data', data={'model': 'res.partner'})
        self.assertIn('result', response)
```

**Test Tags:**

```python
@tagged('post_install', '-at_install')   # Run after install only
@tagged('-at_install', 'post_install')   # Same
@tagged('post_install', '-at_install', 'my_feature')  # Custom tag for selective running
```

### JavaScript Tests

**Hoot Framework (Odoo 18 test framework):**

```javascript
import { expect, test, describe, beforeEach } from "@odoo/hoot";
import { click, press, waitFor, queryAll } from "@odoo/hoot-dom";
import { animationFrame } from "@odoo/hoot-mock";
import { mountView, getService, patchWithCleanup } from "@web/../tests/web_test_helpers";

describe("MyComponent", () => {
    beforeEach(async () => {
        // setup
    });

    test("renders correctly", async () => {
        await mountView({
            type: "form",
            resModel: "res.partner",
            arch: `<form><field name="name"/></form>`,
        });
        expect(".o_field_widget[name='name']").toHaveCount(1);
    });

    test("button click triggers action", async () => {
        await mountView({ ... });
        await click(".o_my_button");
        await animationFrame();
        expect(".o_notification").toHaveCount(1);
    });
});
```

**Asset Bundle for Tests:**
```python
'assets': {
    'web.assets_tests': ['my_module/static/tests/tours/**/*'],
    'web.qunit_suite_tests': ['my_module/static/tests/**/*.test.js'],
}
```

**Tour Tests (UI Automation):**

```javascript
import { registry } from "@web/core/registry";

registry.category("web_tour.tours").add("my_module_tour", {
    url: "/odoo/sales",
    steps: () => [
        {
            content: "Click create button",
            trigger: ".o_list_button_add",
            run: "click",
        },
        {
            content: "Set partner",
            trigger: ".o_field_widget[name='partner_id'] input",
            run: "edit Test Partner",
        },
        {
            content: "Select partner from dropdown",
            trigger: ".o_field_widget[name='partner_id'] .dropdown-item:contains('Test Partner')",
            run: "click",
        },
        {
            content: "Save the record",
            trigger: ".o_form_button_save",
            run: "click",
        },
    ],
});
```

---

## 9. Commit Conventions

### Commit Message Format

```
[PREFIX] module_name: short description

Optional longer description explaining the why.

closes #1234   (optional: link to GitHub issue)
```

### Prefixes

| Prefix  | Usage                          | Example                                              |
|---------|--------------------------------|------------------------------------------------------|
| `[FIX]` | Bug fixes                      | `[FIX] sale: prevent duplicate SO confirmation`      |
| `[IMP]` | Improvements/enhancements      | `[IMP] stock: optimize inventory valuation query`    |
| `[ADD]` | New features                   | `[ADD] sale_loyalty: add gift card support`           |
| `[REM]` | Removals                       | `[REM] sale: remove deprecated price method`         |
| `[REV]` | Reversions/rollbacks           | `[REV] stock: revert picking group changes`          |
| `[MOV]` | Code moves between modules     | `[MOV] sale: move margin computation to sale_margin` |
| `[REF]` | Refactoring (no behavior change)| `[REF] account: simplify tax computation logic`     |
| `[PERF]`| Performance optimizations      | `[PERF] purchase_mrp: cache BOM explode results`    |
| `[I18N]`| Translations                   | `[I18N] *: fetch latest Weblate translations`       |
| `[CLA]` | CLA-related changes            | `[CLA] partner: contributor license agreement`       |

### Rules

- Module name in lowercase with underscores
- Short description in lowercase, action-oriented
- Multiple modules: `[FIX] sale,purchase: fix shared domain filter`
- All modules: `[I18N] *: export source terms`
- PR body should reference the issue and describe current vs desired behavior
