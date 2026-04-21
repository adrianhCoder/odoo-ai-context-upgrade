# Odoo 18 to Odoo 19 Future Migration Guide

While the transition to Odoo 18 required specific ORM and OWL adjustments, Odoo 19 introduces even more structural breaking changes for custom add-ons. Below is a summarized list of core technical changes you will need to account for when transitioning your custom modules from version 18 to version 19.

## 1. Core Framework & Python

### Python Environment
- **Requirement**: Odoo 19 strictly requires **Python 3.10+**. For production performance gains, Python 3.12 is highly recommended. 
- **Legacy Namespace Deprecation**: The legacy `odoo.osv` namespace is now officially deprecated. Any legacy modules holding onto this old backbone will completely break.
- **Config Key Renamed**: In your `odoo.conf` file, the legacy `xmlrpc_port` configuration key is officially renamed to `http_port`.
- **Decorators**: The `@api.returns` decorator has been removed from the ORM.

## 2. ORM & Database Changes

### Inventory & Stock Refactoring
- **Removal of `stock.valuation.layer`**: The `stock.valuation.layer` model has been completely removed. Inventory valuation logic has been directly merged and is now handled natively on `stock.move`. Custom modules interacting with accounting stock valuations will need significant refactoring.

### Deprecated Contextual Access
- Bypassing the environment using `record._cr`, `record._context`, and `record._uid` is now deprecated. You must use `record.env.cr`, `record.env.context`, etc.

### Sequence & Indexing 
- The `_sequence` attribute of `Model` objects has been removed. Odoo now strictly utilizes the PostgreSQL default sequence behaviors for primary keys.

## 3. Frontend & OWL (Odoo Web Library)

### Kanban Views
- The `<kanban-box>` component has been replaced by the more standardized `<card>` component. All XML Kanban views must be updated to replace `<kanban-box>` with `<card>`.

### Template Rendering
- The `t-esc` directive in QWeb XML templates is now fully deprecated. You must swap all occurrences of `t-esc` to the safer `t-out` directive for rendering variable content.

### Inner Template Inheritance
- Odoo 19 introduces `mode="inner"` inheritance. This allows developers to explicitly replace only the inner content of a targeted template while intrinsically preserving the outer structural wrapper.

### ORM Service Data Format (`searchRead`)
- **Strict Method Arguments:** The `searchRead` method signature strictly requires the fields array to be passed directly as the 3rd argument `(model, domain, fields, kwargs)`, avoiding objects. If passed wrongly inside bounds, Odoo 19 will consistently throw `Invalid fields list: [object Object]`.
- **Data Encapsulation:** Always defensively unwrap potential dictionaries returned by the ORM service in JS. Odoo 19 standardizes pagination metadata wrapping, returning objects like `{ records: [...], length: int }` instead of generic arrays.
```javascript
// Safely unpack responses
const result = await this.orm.searchRead('model.name', [], ['id']);
const safeArray = Array.isArray(result) ? result : (result.records || []);
```

## 4. Security & Routing

### Python Controller Routing
- In `@http.route()` decorators, using `type='json'` must be updated to `type='jsonrpc'`. This enforces strict JSON-RPC protocol compliance.

### Security Groups
- The `category_id` field in the `res.groups` model has been removed. Odoo 19 introduces a completely newly redesigned privilege-based system for managing hierarchical security groups.

---
*Reference: Official Odoo 19 Developer Framework Changelogs and OCA guidelines.*
