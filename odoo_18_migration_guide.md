# Odoo 17 to Odoo 18 Migration Guide: Code-Breaking Changes

When migrating custom Odoo modules from version 17 to version 18, several critical code-breaking changes must be addressed to ensure compatibility. 

## 1. XML View Changes

### `tree` to `list` tag and XPATH Selectors
The `<tree>` tag in XML views has been renamed to `<list>`. Any custom views defining or inheriting a `<tree>` view must be updated.
- **Before:** `<tree string="My Tree"> ... </tree>`
- **After:** `<list string="My List"> ... </list>`

> [!WARNING]
> This critically affects **xpath expressions** as well. Xpaths that previously targeted `expr="//tree"` or `expr="//field[@name='line_ids']/tree"` must be updated to target `list` instead.
- **Before:** `<xpath expr="//field[@name='order_line']/tree" position="inside">`
- **After:** `<xpath expr="//field[@name='order_line']/list" position="inside">`

### Removal of `attrs` and `states` in XML
The legacy `attrs` dictionary and `states` attributes have been simplified. You must now use direct attributes (like `invisible`, `readonly`, `required`) with boolean expressions.
- **`attrs` Replacement:**
  - **Before:** `attrs="{'invisible': [('state', '=', 'draft')]}"`
  - **After:** `invisible="state == 'draft'"`
- **`states` Replacement:**
  - **Before:** `<button states="draft" .../>`
  - **After:** `<button invisible="state != 'draft'" .../>`

### Settings XML Structure
The structure for defining configuration settings has been simplified, replacing older structures with specific `app` and `block` groupings.

---

## 2. Python & Framework Changes

### Removal of `name_get()`
While `name_get()` was deprecated in V17, it has been completely removed in Odoo 18.
- **Action Required:** Switch to using a computed `display_name` field. Ensure records display correctly in dropdowns, breadcrumbs, and across the UI by defining a proper compute method for `display_name`.

### Removal of `states` on Fields
The `states` parameter on Python field definitions is removed. Conditional logic applied via the `states` attribute must now be handled directly in the UI XML views (using `readonly=...`, `invisible=...`, etc.) or through other Python logic overriding `fields_view_get`.

### API Decorators
The `@api.multi` decorator has been completely removed.
- **Action Required:** Remove it. Depending on the method, ensure you iterate over `self` if needed, as most ORM methods handle recordsets by default now.

### ORM Command Objects replacing Magic Tuples
The legacy "magic tuples" used for one-to-many and many-to-many relational writes are deprecated and officially replaced by `Command` objects in Odoo 17/18.
- **Action Required:** Instead of tuples, import `Command` and use its descriptive methods.
- **Before:** `order.write({'line_ids': [(0, 0, {'name': 'New'})]})`
- **After:** 
```python
from odoo import Command
order.write({'line_ids': [Command.create({'name': 'New'})]})
```
*(Similarly, `(6, 0, [id])` -> `Command.set([id])`, `(2, id, 0)` -> `Command.delete(id)`, etc.)*

### HTML Escaping in message_post (XSS Prevention)
Odoo 18 strictly enforces safe HTML rendering to prevent XSS. Previously, concatenating strings in `message_post` blocks was acceptable. Now, all bodies containing HTML must be explicitly wrapped in `Markup` and variables must be escaped.
- **Action Required:** Use `%` string formatting alongside `Markup` and `escape()`. **Do not** use `f-strings` directly inside `Markup()` as it bypasses the safety mechanism.
- **Before:**
```python
from markupsafe import Markup
body = Markup(f"<p>Order: {self.name}</p>")
self.message_post(body=body)
```
- **After:**
```python
from markupsafe import Markup, escape
body = Markup("<p>Order: %s</p>") % escape(self.name)
self.message_post(body=body)
```

### Scheduled Actions (ir.cron) - Field Removals
The `numbercall` field (used to define how many times a cron runs, or `-1` for infinite) and the `doall` field have been **completely removed** from the `ir.cron` model in Odoo 18.
- **Action Required:** Remove the `<field name="numbercall">-1</field>` and `<field name="doall" eval="..." />` tags entirely from your `data/cron.xml` definitions. Odoo 18 assumes scheduled actions recur infinitely by default.

### Universal Product Type `type='product'` Deprecation
In Odoo 18, physical items strictly tracked in inventory no longer use the explicit `'product'` type value. Instead, `type='consu'` applies seamlessly to both Consumable and Storable items, offset strictly by an internal `is_storable` boolean.
- **Action Required:** Change all Python domains, ORM JS hook domain calls, XML attributes, and Wizard hardcoded domains checking for `('type', '=', 'product')` to use `('is_storable', '=', True)`.

### Javascript ORM Hooks (`searchRead` Signature Validation)
The OWL (Odoo Web Library) Javascript component API in Odoo 18 strictly validates the arguments sent to the native `this.orm.searchRead()` method.
- **Action Required:** In Odoo 18, the `fields` array **must** be explicitly passed as the 3rd argument (a primitive array of strings). You cannot group `fields` inside the `kwargs` dictionary. Doing so will immediately crash the client with an `Invalid fields list: [object Object]` exception triggered at `validatePrimitiveList`. Any other options like `order` or `limit` must go into the 4th argument (`kwargs`).
- **Incorrect (Crashes Odoo 18):**
```javascript
this.orm.searchRead('res.partner', [['is_company', '=', true]], {
    fields: ['id', 'name'],
    order: 'name asc'
});
```
- **Correct (Odoo 18 Standard):**
```javascript
this.orm.searchRead('res.partner', [['is_company', '=', true]], ['id', 'name'], {
    order: 'name asc'
});
```

### Javascript ORM Hooks (Return Data Structure Wrapper)
Depending on the specific version and context execution within Odoo 18, `searchRead` may return a flat Javascript `Array` or a wrapped object containing pagination metadata (like `{ records: [...], length: ... }`).
- **Action Required:** Implement defensive checks in JS code executing `searchRead` to securely extract the array of records without breaking the execution flow or throwing `TypeError: ... is not iterable`.
- **Example:**
```javascript
const result = await this.orm.searchRead('product.product', [], ['id']);
const productsList = Array.isArray(result) ? result : (result.records || result.data || []);
```

---

## 3. Critical Schema / Model Refactors

### Security Groups `category_id` Removal
In Odoo 18, the security group hierarchy was drastically refactored. The `category_id` field has been entirely stripped from the `res.groups` model in favor of a new privilege clustering system. 
- **Action Required:** Remove all `<field name="category_id" ref="..."/>` mappings from your custom `security.xml` files defining `res.groups` records. If left in, the database will crash on initialization with a `ValueError: Invalid field 'category_id' in 'res.groups'`.

### Mexican Localization (Pedimentos) Refactor
Odoo 18 overhauled the `l10n_mx_edi` models used for Mexican Electronic Invoicing (Pedimentos/Aduanas). Specifically, models like `l10n_mx_edi.customs.regime` and `l10n_mx_edi.customs.document.type` are strictly abstracted or missing structurally without rigid dependency trees.
- **Action Required:** If your custom code historically linked directly to these models using `fields.Many2one('l10n_mx_edi.customs.regime', ...)`, the registry will immediately crash with an `AssertionError: unknown comodel_name`. Migrate these fields to portable `fields.Char()` variables and strip `_id` tails from XML view templates.

### UI Menu Access Rights (`groups_id` -> `group_ids`)
The field mapping security groups to `ir.ui.menu` models was renamed to enforce Odoo's pluralization standard on Many2Many fields.
- **Action Required:** Change `<field name="groups_id">` to `<field name="group_ids">` in any XML where you are appending or replacing groups on a `<record model="ir.ui.menu">`. Failure to do so crashes the database with `ValueError: Invalid field 'groups_id' in 'ir.ui.menu'`.

---

## 4. General Requirements

- **Python Version:** Odoo 18 requires Python 3.11+.
- **PostgreSQL Version:** PostgreSQL 13+ is required.
- **Security:** Ensure that security rules (`ir.model.access.csv` and `ir.rule`) are strictly defined. Older workarounds in views may no longer function.
