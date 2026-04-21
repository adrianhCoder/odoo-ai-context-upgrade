Odoo Migration Decoded: 16 → 17 → 18 → 19 — Backend Overhaul: ORM, Views & Controllers (Part 1)
Osama Alhalabi
Osama Alhalabi

Follow
15 min read
·
Mar 4, 2026
2




Everything that changed on the backend, everything that broke in your XML, and exactly how to fix it — with code you can copy into your module today.

It’s Monday morning. Your client sends you a Slack message: “Hey, we’re upgrading to Odoo 19. How long will it take?” Your coffee goes cold. Your soul briefly exits your body. You’ve heard the stories — the attrs that stopped working, the name_get that got gone, the XML views that looked right but weren't.

This is Part 1 of a two-part guide covering everything except JavaScript and OWL (that’s Part 2). Here we tackle the Python, XML, SCSS, and server-side changes that will break your module — and exactly how to fix them.

🗺️ The Version Map: What Changed When
Before we dive in, here’s the 30,000-foot view. Think of each Odoo version like a movie sequel — same universe, different rules.

Press enter or click to view image in full size

Odoo 16–19 at a Glance: Releases, Breaking Changes & Vibes
Pro tip: The 16 to 17 jump is the most painful by far. If you survive that, 17 to 18 to 19 is mostly incremental polish. Think of 16 to 17 as moving from an apartment to a house — same city, completely different furniture arrangement.

💥 The Big Breaking Changes (16 to 17)
These two changes (plus OWL, covered in Part 2) are responsible for roughly 80% of migration headaches. Address them first.

Goodbye attrs, Hello Inline Expressions

attrs was the XML mechanism for dynamically showing, hiding, or making fields required based on conditions. It looked like a Python dict embedded in your XML -- which, honestly, was always a bit odd. Odoo 17 ripped it out entirely and replaced it with direct inline attribute expressions. Cleaner, more readable, and absolutely guaranteed to break your module if you don't update it.

Think of attrs like an old universal remote with one overloaded button. The new system gives each function its own dedicated button.

Before (Odoo 16)

<field name="partner_id"
       attrs="{'invisible': [('state', '=', 'draft')],
               'readonly':  [('state', 'not in', ['draft', 'sent'])],
               'required':  [('state', '=', 'confirmed')]}" />

<button name="action_confirm"
        states="draft"
        type="object"
        string="Confirm" />
After (Odoo 17+)

<field name="partner_id"
       invisible="state == 'draft'"
       readonly="state not in ('draft', 'sent')"
       required="state == 'confirmed'" />

<!-- states attribute is GONE -- use invisible instead -->
<button name="action_confirm"
        invisible="state != 'draft'"
        type="object"
        string="Confirm" />
Complete Expression Syntax Reference
Here’s every pattern you’ll encounter when translating attrs domains to inline expressions:

<!-- Simple equality -->
<!-- Old: attrs="{'invisible': [('state', '=', 'draft')]}" -->
<field name="x" invisible="state == 'draft'"/>

<!-- NOT equal -->
<!-- Old: attrs="{'invisible': [('state', '!=', 'sale')]}" -->
<field name="x" invisible="state != 'sale'"/>

<!-- IN operator -->
<!-- Old: attrs="{'invisible': [('state', 'in', ['draft', 'cancel'])]}" -->
<field name="x" invisible="state in ('draft', 'cancel')"/>

<!-- NOT IN operator -->
<!-- Old: attrs="{'invisible': [('state', 'not in', ['sale', 'done'])]}" -->
<field name="x" invisible="state not in ('sale', 'done')"/>

<!-- Boolean field check -->
<!-- Old: attrs="{'invisible': [('active', '=', False)]}" -->
<field name="x" invisible="not active"/>

<!-- AND condition (list of conditions in old attrs was AND by default) -->
<!-- Old: attrs="{'invisible': [('state', '=', 'draft'), ('type', '=', 'service')]}" -->
<field name="x" invisible="state == 'draft' and type == 'service'"/>

<!-- OR condition (old attrs used '|' prefix) -->
<!-- Old: attrs="{'invisible': ['|', ('type', '=', 'service'), ('active', '=', False)]}" -->
<field name="x" invisible="type == 'service' or not active"/>

<!-- Numeric comparison -->
<!-- Old: attrs="{'readonly': [('qty_invoiced', '>', 0)]}" -->
<field name="x" readonly="qty_invoiced > 0"/>

<!-- Nested conditions -->
<field name="x" invisible="(state == 'draft' and not partner_id) or state == 'cancel'"/>

<!-- column_invisible with parent.field syntax (inside O2M sub-views) -->
<field name="qty_delivered"
       column_invisible="parent.state == 'draft'"/>
The states attribute was shorthand for visibility based on a field named state. It's fully gone in 17+:

<!-- Odoo 16 -->
<field name="partner_id" states="draft,sent"/>
<!-- meant: visible only when state in ('draft', 'sent') -->

<!-- Odoo 17+ -->
<field name="partner_id" invisible="state not in ('draft', 'sent')"/>
Note: These inline expressions evaluate in a JavaScript context. No imports, no function calls — just simple comparisons and logical operators. If your attrs had really complex nested logic, you may need a helper Boolean computed field on the model side.

name_get() to _compute_display_name: The Identity Crisis

name_get() was the OG method for controlling how a record's name appears in dropdowns, breadcrumbs, and anywhere Odoo needs a human-readable label. In 17, it was deprecated in favor of a proper computed field. In 18+, the old method is essentially dead. Your module will silently return wrong names or throw warnings — neither is fun to debug in production.

Before (Odoo 16)

class ProductVariant(models.Model):
    _name = 'product.variant'

    name = fields.Char(required=True)
    color = fields.Char()
    size = fields.Char()

    def name_get(self):
        result = []
        for record in self:
            name = record.name
            if record.color:
                name = f"{name} [{record.color}]"
            if record.size:
                name = f"{name} - {record.size}"
            result.append((record.id, name))
        return result
After (Odoo 17+)

class ProductVariant(models.Model):
    _name = 'product.variant'

    name = fields.Char(required=True)
    color = fields.Char()
    size = fields.Char()

    def _compute_display_name(self):
        for record in self:
            name = record.name
            if record.color:
                name = f"{name} [{record.color}]"
            if record.size:
                name = f"{name} - {record.size}"
            record.display_name = name
The Inheritance Case
If you were extending an existing model’s name_get via super(), the pattern changes too:

# Odoo 16
class SaleOrder(models.Model):
    _inherit = 'sale.order'

    def name_get(self):
        result = super().name_get()
        return [(rec_id, f"{name} (Custom)") for rec_id, name in result]

# Odoo 17+
class SaleOrder(models.Model):
    _inherit = 'sale.order'

    def _compute_display_name(self):
        super()._compute_display_name()
        for record in self:
            record.display_name = f"{record.display_name} (Custom)"
Why this is actually an upgrade: Because display_name is now a proper computed field, you can set store=True and use it in search filters and ORDER BY clauses. That's a genuine improvement, not just a rename.

❤️ ORM Upgrades You’ll Actually Love
Not everything about migration is suffering. Some of these ORM changes are genuinely great once you get used to them.

Command Objects: Goodbye Magic Numbers 👋
If you’ve ever written (0, 0, {...}) to create a One2many record and had to Google what those zeros mean, you're in good company. Odoo 17+ makes Command objects the official pattern. Here's the full reference:

from odoo import Command

# CREATE a new related record
# Old: (0, 0, {'name': 'New Line', 'qty': 1})
Command.create({'name': 'New Line', 'qty': 1})

# UPDATE an existing related record
# Old: (1, record_id, {'qty': 2})
Command.update(record_id, {'qty': 2})

# DELETE a record (removes from DB entirely)
# Old: (2, record_id, 0)
Command.delete(record_id)

# UNLINK (disconnect without deleting from DB)
# Old: (3, record_id, 0)
Command.unlink(record_id)

# LINK an existing record
# Old: (4, record_id, 0)
Command.link(record_id)

# CLEAR all records (unlink all)
# Old: (5, 0, 0)
Command.clear()

# SET (replace entire list with these IDs)
# Old: (6, 0, [id1, id2, id3])
Command.set([id1, id2, id3])
Real-world before and after:

# Odoo 16
order = self.env['sale.order'].create({
    'partner_id': partner.id,
    'order_line': [
        (0, 0, {'product_id': prod1.id, 'product_uom_qty': 2}),
        (1, line.id, {'product_uom_qty': 5}),
        (2, old_line.id, 0),
    ],
    'tag_ids': [(6, 0, [tag1.id, tag2.id])],
})

# Odoo 17+
from odoo import Command

order = self.env['sale.order'].create({
    'partner_id': partner.id,
    'order_line': [
        Command.create({'product_id': prod1.id, 'product_uom_qty': 2}),
        Command.update(line.id, {'product_uom_qty': 5}),
        Command.delete(old_line.id),
    ],
    'tag_ids': [Command.set([tag1.id, tag2.id])],
})
Self-documenting, IDE-autocomplete-friendly, and far less likely to cause “wait, was it (0, 0, {}) or (0, False, {})?" debates in code review.

Import note: It’s from odoo import Command, not from odoo.fields import Command. This trips people up.

The New _read_group() API

The old read_group() returned a list of dictionaries. The new _read_group() returns tuples with actual record objects for relational fields. That means no more group["account_id"][1] to get the account name.

Press enter or click to view image in full size

read_group() vs _read_group(): What Changed in Odoo 17+
# Odoo 16
results = self.env['account.move.line'].read_group(
    domain=[('move_id.state', '=', 'posted')],
    fields=['account_id', 'debit:sum', 'credit:sum'],
    groupby=['account_id'],
)
for group in results:
    print(group['account_id'][1], group['debit'], group['credit'])

# Odoo 17+
results = self.env['account.move.line']._read_group(
    domain=[('move_id.state', '=', 'posted')],
    groupby=['account_id'],
    aggregates=['debit:sum', 'credit:sum'],
)
for account, debit_sum, credit_sum in results:
    print(account.name, debit_sum, credit_sum)  # Actual records!
Markup() for HTML Safety

Raw HTML string concatenation is officially a security problem in Odoo 17+. All HTML content in Python must use markupsafe.Markup to prevent XSS vulnerabilities.

# Odoo 16 -- common but unsafe
def _get_description(self):
    return "<p>Hello <b>" + self.name + "</b></p>"  # XSS risk

# Odoo 17+ -- Markup required
from markupsafe import Markup, escape

def _get_description(self):
    return Markup("<p>Hello <b>%s</b></p>") % escape(self.name)# Odoo 17+ -- Markup required
from markupsafe import Markup, escape
This extends to message_post in the chatter:

# Odoo 16
self.message_post(body="<p>Invoice " + inv.name + " confirmed</p>")

# Odoo 17+
from markupsafe import Markup, escape

self.message_post(
    body=Markup("<p>Invoice <b>%s</b> confirmed</p>") % escape(inv.name)
)
flush() and invalidate_cache() Replacements

The old blanket methods are replaced with more granular versions:

Press enter or click to view image in full size

Odoo 17 ORM Renames: Cache Invalidation & Flush Methods
# Odoo 16
self.env['res.partner'].invalidate_cache(['name', 'email'])
self.env['res.partner'].flush()

# Odoo 17+
records.invalidate_recordset(['name', 'email'])
records.flush_recordset(['name', 'email'])
# Or at model level:
self.env['res.partner'].invalidate_model(['name'])
self.env['res.partner'].flush_model(['name'])
🎨 Views: The Great XML Makeover
If the ORM changes are the plumbing renovation, the view changes are the interior redesign. Same house, new fixtures everywhere.

<tree> to <list>

The most cosmetic but also most-searched-on-Google rename: tree views are now list views.

<!-- Odoo 16 -->
<tree string="Sale Orders" decoration-danger="state == 'cancel'">
    <field name="name"/>
    <field name="state"/>
</tree>

<!-- Odoo 17+ -->
<list string="Sale Orders" decoration-danger="state == 'cancel'">
    <field name="name"/>
    <field name="state"/>
</list>
Don’t forget inherited views — your XPath targets need updating too:

<!-- Odoo 16 -->
<xpath expr="//field[@name='order_line']/tree/field[@name='price_unit']" position="after">

<!-- Odoo 17+ -->
<xpath expr="//field[@name='order_line']/list/field[@name='price_unit']" position="after">
Kanban Card Template Rename
The kanban template name changed from kanban-card to card, and the internal structure was simplified with dedicated <header>, <main>, and <footer> elements:

<!-- Odoo 16 -->
<kanban>
    <templates>
        <t t-name="kanban-card">
            <div class="oe_kanban_card oe_kanban_global_click">
                <div class="o_kanban_record_top">
                    <strong><field name="name"/></strong>
                </div>
                <div class="o_kanban_record_bottom">
                    <div class="oe_kanban_bottom_left">
                        <field name="priority" widget="priority"/>
                    </div>
                    <div class="oe_kanban_bottom_right">
                        <field name="user_id" widget="many2one_avatar_user"/>
                    </div>
                </div>
            </div>
        </t>
    </templates>
</kanban>

<!-- Odoo 17+ -->
<kanban>
    <templates>
        <t t-name="card">
            <field name="name" class="fw-bold"/>
            <footer>
                <field name="priority" widget="priority"/>
                <field name="user_id" widget="many2one_avatar_user" class="ms-auto"/>
            </footer>
        </t>
    </templates>
</kanban>
t-raw to t-out

t-raw rendered unescaped HTML -- a walking XSS vulnerability. It's removed in 17+. Use t-out instead, which auto-escapes strings but renders Markup() objects as HTML.

<!-- Odoo 16 -- XSS risk -->
<div t-raw="record.description"/>

<!-- Odoo 17+ -- safe by default -->
<div t-out="record.description"/>
Also, t-esc (which was the safe option in 16) is deprecated in favor of t-out. One directive to rule them all.

Complete Form View Migration
Here’s a full before/after for a form view, showing every change in context:

<!-- Odoo 16 -->
<form string="Sale Order">
    <header>
        <button name="action_confirm" type="object" string="Confirm"
            attrs="{'invisible': [('state', '!=', 'draft')]}"/>
        <button name="action_cancel" type="object" string="Cancel"
            attrs="{'invisible': [('state', 'in', ['cancel', 'done'])]}"/>
        <field name="state" widget="statusbar" statusbar_visible="draft,sale,done"/>
    </header>
    <sheet>
        <div class="oe_title">
            <h1><field name="name" attrs="{'readonly': [('state', '!=', 'draft')]}"/></h1>
        </div>
        <group>
            <group>
                <field name="partner_id"
                    attrs="{'readonly': [('state', 'not in', ['draft', 'sent'])]}"/>
                <field name="date_order"
                    attrs="{'required': [('state', '=', 'sale')]}"/>
            </group>
            <group>
                <field name="warehouse_id"
                    attrs="{'invisible': [('picking_policy', '=', 'direct')]}"/>
            </group>
        </group>
        <notebook>
            <page string="Order Lines">
                <field name="order_line"
                    attrs="{'readonly': [('state', 'in', ['done', 'cancel'])]}">
                    <tree editable="bottom">
                        <field name="product_id"/>
                        <field name="qty_delivered"
                            attrs="{'column_invisible': [('parent.state', '=', 'draft')]}"/>
                    </tree>
                </field>
            </page>
        </notebook>
    </sheet>
    <div class="oe_chatter">
        <field name="message_follower_ids" widget="mail_followers"/>
        <field name="activity_ids" widget="mail_activity"/>
        <field name="message_ids" widget="mail_thread"/>
    </div>
</form>
<!-- Odoo 17+ (migrated) -->
<form string="Sale Order">
    <header>
        <button name="action_confirm" type="object" string="Confirm"
            invisible="state != 'draft'"/>
        <button name="action_cancel" type="object" string="Cancel"
            invisible="state in ('cancel', 'done')"/>
        <field name="state" widget="statusbar" statusbar_visible="draft,sale,done"/>
    </header>
    <sheet>
        <div class="oe_title">
            <h1><field name="name" readonly="state != 'draft'"/></h1>
        </div>
        <group>
            <group>
                <field name="partner_id"
                    readonly="state not in ('draft', 'sent')"/>
                <field name="date_order"
                    required="state == 'sale'"/>
            </group>
            <group>
                <field name="warehouse_id"
                    invisible="picking_policy == 'direct'"/>
            </group>
        </group>
        <notebook>
            <page string="Order Lines">
                <field name="order_line"
                    readonly="state in ('done', 'cancel')">
                    <list editable="bottom">
                        <field name="product_id"/>
                        <field name="qty_delivered"
                            column_invisible="parent.state == 'draft'"/>
                    </list>
                </field>
            </page>
        </notebook>
    </sheet>
    <chatter/>
</form>
Notice every change: attrs dicts become inline expressions, <tree> becomes <list>, column_invisible uses parent. syntax directly, and the entire chatter boilerplate collapses into <chatter/>.

⚠️ Controllers: Small Change, Serious Consequences
type="json" vs type="jsonrpc"

Odoo 17 introduced a distinction between plain JSON endpoints and JSON-RPC protocol endpoints. The web client’s internal calls now use type="jsonrpc" for the standard JSON-RPC 2.0 protocol.

Press enter or click to view image in full size

Odoo Controller Types: http vs json vs jsonrpc — When to Use Each
# Odoo 16
@http.route("/my_module/get_data", type="json", auth="user")
def get_data(self, **kwargs):
    return {"data": request.env["my.model"].search([]).read(["name"])}

# Odoo 17+ -- for endpoints called by the Odoo web client
@http.route("/my_module/get_data", type="jsonrpc", auth="user")
def get_data(self, **kwargs):
    return {"data": request.env["my.model"].search([]).read(["name"])}
Bearer Token Authentication (17+)
Odoo 17 introduced auth="bearer" for API token authentication, which is a much cleaner pattern for external integrations:

# Odoo 17+: Bearer token auth for external API consumers
@http.route("/api/v1/resource", auth="bearer", type="json")
def api_resource(self):
    return {"user": request.env.user.name}
💅 Assets and SCSS: The Sass Saga
LibSass to Dart Sass
Odoo 17+ switched from libsass (deprecated upstream) to dart-sass. This breaks two things: the division operator and @import statements.

Division operator:

// Odoo 16 (libsass) -- worked fine
.element {
    width: 100px / 2;       // = 50px
    font-size: 24px / 1.5;  // = 16px
}

// Odoo 17+ (dart-sass) -- division with / is treated as CSS slash
@use "sass:math";

.element {
    width: math.div(100px, 2);       // = 50px
    font-size: math.div(24px, 1.5);  // = 16px
}

// Alternative: use CSS calc() -- works everywhere
.element {
    width: calc(100px / 2);
}
@import to @use/@forward:

// Odoo 16 (libsass)
@import "variables";
@import "mixins";

.element {
    color: $primary-color;
    @include my-mixin();
}

// Odoo 17+ (dart-sass)
@use "variables" as vars;
@use "mixins" as mix;

.element {
    color: vars.$primary-color;
    @include mix.my-mixin();
}

// Or with wildcard namespace (closest to old behavior)
@use "variables" as *;
@use "mixins" as *;

.element {
    color: $primary-color;
    @include my-mixin();
}
🆕 What’s New in Odoo 18 and 19
After the 16-to-17 gauntlet, the 17-to-18-to-19 journey is more of a scenic hike than a death march.

models.Constraint (Odoo 18+)

Get Osama Alhalabi’s stories in your inbox
Join Medium for free to get updates from this writer.

Enter your email
Subscribe

Remember me for faster sign in

A new declarative approach for SQL constraints that provides better ORM integration:

# Odoo 16/17 -- still valid in 18, but the old way
_sql_constraints = [
    ('unique_ref', 'UNIQUE(default_code)', 'Product reference must be unique.'),
    ('positive_price', 'CHECK(list_price >= 0)', 'Price must be non-negative.'),
]

# Odoo 18+ -- the new way
from odoo.models import Constraint

_constraints = [
    Constraint('unique_ref', 'UNIQUE(default_code)', 'Product reference must be unique.'),
    Constraint('positive_price', 'CHECK(list_price >= 0)', 'Price must be non-negative.'),
]
Note that _sql_constraints still works across all versions. The Constraint class is the direction Odoo is heading.

Python 3.12 (Odoo 18+) and 3.11 Minimum (Odoo 19)
Key deprecations to watch:

datetime.datetime.utcnow() is deprecated. Use datetime.datetime.now(datetime.timezone.utc).
The distutils module is removed entirely in 3.12.
Performance improvements of 10–60% over 3.10.
Odoo 19 raises the floor to Python 3.11 minimum.
Content Security Policy Tightening (18 and 19)
Odoo 18+ enforces stricter CSP headers. If your module dynamically injects scripts or loads from CDNs, it will break:

# This pattern BREAKS in Odoo 18+ (CSP blocks inline script creation)
# No more document.createElement("script") with external URLs

# Instead, vendor libraries into your module and declare them in assets:
"assets": {
    "web.assets_backend": [
        "my_module/static/lib/some-library/some-library.min.js",
        "my_module/static/src/js/my_component.js",
    ],
},
No inline <script> tags in QWeb templates either. Everything must go through the asset bundle system.

Dart Sass Enforcement (Odoo 18)
While Odoo 17 introduced dart-sass, Odoo 18 fully enforces it. Any SCSS files still using / for division or @import will fail compilation.

Manifest Changes
license field is mandatory starting in Odoo 17. Omitting it will raise a warning in 16 and an error in 17+.
post_init_hook signature changed in 17+: it now receives env directly instead of (cr, registry).
# Odoo 16
def post_init_hook(cr, registry):
    from odoo import api, SUPERUSER_ID
    env = api.Environment(cr, SUPERUSER_ID, {})
    # ...

# Odoo 17+
def post_init_hook(env):
    # env is ready to use
    env["my.model"].search([])._compute_something()
Testing: SavepointCase Removed (17+)

SavepointCase is gone in 17+. Use TransactionCase, which now handles savepoints automatically when setUpClass is defined:

# Odoo 16
from odoo.tests.common import SavepointCase  # Works

# Odoo 17+ -- ImportError! Use TransactionCase instead.
from odoo.tests import common
class TestMyModel(common.TransactionCase):
    @classmethod
    def setUpClass(cls):
        super().setUpClass()
        cls.shared_record = cls.env["my.model"].create({"name": "Shared"})
Reports: _render_qweb_pdf Deprecation

The _render_qweb_pdf() method is deprecated in 17 and removed in 18+. Use _render() instead:

# Odoo 16
pdf_content, content_type = self.env["ir.actions.report"]._render_qweb_pdf(
    report_action, res_ids=self.ids
)

# Odoo 17+
report = self.env.ref("my_module.action_report_my_document")
pdf_content, mime = report._render(self.ids)
Mail and Chatter
Two notable changes:

<chatter/> shorthand (17+): Replaces the verbose three-field boilerplate with a single tag.

Markup for message_post: The body parameter in message_post must be a Markup object for HTML content in 17+, as shown in the Markup section above.

✅ Part 1 Migration Checklist
Print this. Tape it to your monitor.

16 to 17

Replace all attrs= with inline invisible=, readonly=, required=
Remove all states= attributes; replace with invisible="state not in (...)"
Migrate name_get() overrides to _compute_display_name()
Adopt Command objects for O2M/M2M operations (replace tuple syntax)
Wrap HTML strings in Markup(), always escape() user input
Replace read_group(lazy=...) calls with _read_group() using aggregates
Replace invalidate_cache() with invalidate_model() / invalidate_recordset()
Replace flush() with flush_model() / flush_recordset()
Rename <tree> to <list> in views and XPath targets
Update kanban templates: kanban-card to card
Replace t-raw with t-out, t-esc with t-out
Switch controllers from type="json" to type="jsonrpc" where appropriate
Add "license": "LGPL-3" to __manifest__.py (mandatory)
Update post_init_hook signature from (cr, registry) to (env)
Replace SavepointCase with TransactionCase in tests
Add <chatter/> shorthand or verify chatter field order in form views
Update SCSS: start migration from @import to @use/@forward
17 to 18

Ensure Python 3.12 compatibility (fix utcnow(), distutils, etc.)
Migrate _sql_constraints to models.Constraint (or leave as-is -- both work)
Complete SCSS dart-sass migration (division operator, @import removal)
Remove inline script injection or CDN loads that violate CSP
Vendor third-party JS libraries into static folder, register in asset bundles
Replace _render_qweb_pdf() with _render()
Test all frontend components for CSP errors in browser console
18 to 19

Update Python minimum to 3.11
Update version to 19.0.x.x.x in manifest
Audit all HTTP routes for stricter security checks
Review Constraint definitions for latest patterns
Run odoo-bin --update=all in staging before touching production
· · ·

🕳️ Common Pitfalls (Part 1 Scope)
These are the mistakes that will cost you hours. Learn from our suffering.

Pitfall 1: The Inverted Invisible
When migrating attrs, it's surprisingly easy to invert the logic. The old states="draft" meant "show when draft", but the new syntax uses invisible which means "hide when...":

<!-- WRONG (logic is inverted) -->
<button invisible="state != 'draft'" .../>
<!-- This HIDES when state is NOT draft. Is that what you meant? -->

<!-- Careful: states="draft" meant VISIBLE when draft -->
<!-- So the correct invisible equivalent is: -->
<button invisible="state != 'draft'" .../>
<!-- Actually this IS correct. But double-check every single one. -->
The trap: states="draft,sent" meant "visible when state is draft OR sent". The equivalent invisible is invisible="state not in ('draft', 'sent')". Miss one not and your button vanishes.

Pitfall 2: The Phantom JSON Endpoint
Forgetting to change type="json" to type="jsonrpc" for endpoints called by Odoo's web client. Your AJAX calls will silently fail with cryptic 404s or JSON parsing errors.

Pitfall 3: The Markup XSS Trap
# WRONG -- f-string inside Markup does NOT escape!
from markupsafe import Markup
name = "<script>alert('hacked')</script>"
body = Markup(f"<p>Hello {name}</p>")  # XSS vulnerability!

# RIGHT -- use % formatting or escape() explicitly
from markupsafe import Markup, escape
body = Markup("<p>Hello %s</p>") % escape(name)
# Result: "<p>Hello &lt;script&gt;alert(&#39;hacked&#39;)&lt;/script&gt;</p>"
The key insight: Markup() marks the whole string as safe, including whatever's inside the f-string. Use % formatting with Markup -- it auto-escapes string arguments.

Pitfall 4: SCSS Division That Isn’t
// Dart-sass treats this as CSS "slash" notation, NOT division
.element { font-size: 24px / 1.5; }  // Outputs literally "24px / 1.5"

// You need:
@use "sass:math";
.element { font-size: math.div(24px, 1.5); }  // Outputs 16pxsss
Pitfall 5: The Command Import
# WRONG
from odoo.fields import Command  # ImportError!

# RIGHT
from odoo import Command
# Or:
from odoo import models, fields, api, Command
⚡ TL;DR — The Quick Reference
Migrating Odoo 16 to 17 to 18 to 19? Here’s the one-minute version for Part 1:

attrs is gone -- replace with inline invisible=, readonly=, required=
name_get() is deprecated -- use _compute_display_name() instead
<tree> becomes <list> -- rename in views and XPath targets
t-raw is removed -- use t-out everywhere
HTTP controllers: type="json" to type="jsonrpc" for Odoo RPC calls
Use Command objects instead of magic integer tuples for O2M/M2M writes
Wrap HTML in Markup() and always escape() user input
flush() and invalidate_cache() become flush_model() / invalidate_model()
SCSS: @import to @use, / to math.div() or calc()
Odoo 18+: models.Constraint, Python 3.12, stricter CSP, Dart Sass enforced
Odoo 19: Python 3.11 minimum, incremental tightening across the board
The 16-to-17 jump is the hardest. Plan twice the time you think you need. You’ll use it.

🔮 Coming in Part 2: The JavaScript and OWL Deep Dive
Part 1 covered the backend, XML, and SCSS. But we haven’t touched the other half of migration pain: JavaScript.

Part 2 will cover:

OWL 1.x to OWL 2.x: The complete component migration guide
Lifecycle methods to hooks (willStart to onWillStart, etc.)
Legacy widgets to OWL components (with full before/after examples)
moment.js to luxon (and the silent date bug that will haunt you)
The OWL service system (useService for ORM, notifications, actions)
Tour system migration
Props validation (array to schema)
Parent-child communication patterns
The 5 most common JavaScript migration mistakes
That wraps up the backend side of the Odoo migration journey. In Part 2, we’ll tackle the frontend — OWL components, services, and everything JavaScript. 🚀

If you found this guide useful, feel free to share it with fellow Odoo developers. 🔄 Questions, corrections, or war stories from your own migrations? Drop a comment — I’d love to hear from you. 💬