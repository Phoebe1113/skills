---
name: "frappe-syntax-doctypes"
description: "Use when creating or modifying Frappe/ERPNext DocType definitions, choosing fieldtypes, naming rules, child tables, Custom Fields — incl. mapping Excel columns into DocTypes (v16)."
---

# DocType JSON Design (Frappe v14–v16)

Source: OpenAEC-Foundation/Frappe_Claude_Skill_Package (LGPL-3.0), adapted.

DocTypes define both the data model (DB schema) and the form view. ALWAYS design DocTypes before writing controller logic.

## Top-Level Properties

| Property | Purpose |
|---|---|
| `name` | Singular identifier ("Sales Invoice") |
| `module` | App module |
| `is_submittable` | Draft → Submitted → Cancelled |
| `is_tree` | NestedSet hierarchy (lft/rgt) |
| `is_virtual` | No DB table; custom backend |
| `issingle` | Single settings document |
| `istable` | Child table DocType |
| `track_changes` | Version history |
| `allow_rename` / `allow_copy` / `allow_import` | Rename / Duplicate / Data Import |
| `naming_rule` + `autoname` | Naming method + pattern |
| `title_field`, `search_fields`, `show_title_field_in_link` | Display/search |
| `sort_field`, `sort_order` | Default sort |

## Common Fieldtypes

| Fieldtype | Stores | DB Column |
|---|---|---|
| Data | Text ≤140 chars | VARCHAR(140) |
| Link | Ref to another DocType | VARCHAR(140) |
| Dynamic Link | Ref to any DocType | VARCHAR(140) |
| Select | One of options (newline-separated) | VARCHAR(140) |
| Table | Child rows | Separate table |
| Table MultiSelect | Multi-select link rows | Separate table |
| Check | 0/1 | TINYINT |
| Int | Integer | INT |
| Float | Decimal (9 places) | DECIMAL |
| Currency | Money | DECIMAL |
| Percent | Percentage | DECIMAL |
| Date / Datetime / Time | Dates | DATE / DATETIME / TIME |
| Small Text / Long Text / Text Editor | Text / HTML | TEXT / LONGTEXT |
| Attach / Attach Image | File URL | VARCHAR / TEXT |
| Section Break / Column Break / Tab Break | Layout only | none |

### Excel column → fieldtype heuristic
- Text ≤140 chars, codes, names → Data (or Link if values exist in another DocType)
- Fixed small set of values → Select
- Whole numbers → Int; quantities/decimals → Float; money → Currency
- Excel dates (serial numbers!) → Date; convert serials before import
- Long notes → Small Text / Long Text
- Yes/No, ✓ → Check
- Repeating row groups under one header record → child DocType (istable=1) + Table field
- fieldname: lowercase snake_case ASCII (strip Vietnamese diacritics), label keeps original text

## Essential Field Properties

`reqd`, `unique`, `search_index`, `in_list_view`, `in_standard_filter`, `allow_on_submit`, `read_only`, `hidden`, `depends_on` (`eval:doc.status=="Active"`), `mandatory_depends_on`, `read_only_depends_on`, `fetch_from` (`customer.customer_name`), `fetch_if_empty`, `options`, `default` (`Today`, `__user`), `description`, `collapsible`, `insert_after`.

## Which DocType Type?

```
Store data?
├─ YES: multiple records?
│  ├─ YES: submit/cancel? → is_submittable=1
│  │  └─ NO: hierarchy? → is_tree=1 ; embedded in parent? → istable=1 ; else Standard
│  └─ NO → Single (issingle=1)
└─ NO, external source → Virtual (is_virtual=1)
```

## Naming Rules

| naming_rule | autoname | Output |
|---|---|---|
| Set by User | (empty) | typed manually |
| Autoincrement | (empty) | 1, 2, 3 |
| By Fieldname | `field:item_code` | field value |
| By Naming Series | `naming_series:` | INV-2026-00001 |
| Expression | `PRE-.#####` or `INV-{YYYY}-{#####}` | PRE-00001 |
| Random | `hash` | random |
| UUID (v16) | (empty) | uuid |
| By Script | — | controller `autoname()` |

NEVER use Autoincrement in production (gaps after deletes) — use Expression or Naming Series.

## Child Tables

Child DocType has `istable=1`; parent field `fieldtype: Table`, `options: <Child DocType>`. Child rows get `parent`, `parenttype`, `parentfield`, `idx`.

```python
doc = frappe.get_doc("Sales Invoice", "INV-001")
doc.append("items", {"item_code": "ITEM-001", "qty": 5, "rate": 100.0})
doc.save()
```

NEVER create a child without `istable=1`; NEVER point a Table field at a non-child DocType.

## Single DocType

`issingle=1`, stored in `tabSingles`.
```python
settings = frappe.get_single("My Settings")
value = frappe.db.get_single_value("My Settings", "some_field")
```

## Tree DocType

`is_tree=1` adds `lft`, `rgt`, `parent_<doctype>`, `old_parent`. Define `parent_field`. NEVER edit lft/rgt manually; use `frappe.utils.nestedset.rebuild_tree()`.

## Virtual DocType

`is_virtual=1`; implement `db_insert`, `load_from_db`, `db_update`, `delete`, static `get_list`, `get_count`, `get_stats`. NEVER use `frappe.db.*` for its data.

## Customization without an app

```python
from frappe.custom.doctype.custom_field.custom_field import create_custom_fields
create_custom_fields({
    "Sales Invoice": [dict(fieldname="custom_tracking", label="Tracking ID",
                           fieldtype="Data", insert_after="naming_series")]
}, update=True)

from frappe.custom.doctype.property_setter.property_setter import make_property_setter
make_property_setter("Sales Invoice", "customer", "reqd", 1, "Check")
```

Creating a new DocType programmatically:
```python
doc = frappe.get_doc({
    "doctype": "DocType", "name": "My Record", "module": "My App", "custom": 1,
    "naming_rule": "Expression", "autoname": "REC-.#####",
    "fields": [{"fieldname": "title", "label": "Title", "fieldtype": "Data", "reqd": 1, "in_list_view": 1}],
    "permissions": [{"role": "System Manager", "read": 1, "write": 1, "create": 1, "delete": 1}],
})
doc.insert()
```
(`custom: 1` = stored in DB only; without it, needs developer_mode and an app module to write files.)

## Data Masking (v16+)

`"mask": 1` + `permlevel` hides values from users lacking mask permission; Administrator always sees.

## Type Stubs

NEVER edit code between `# begin: auto-generated types` and `# end: auto-generated types`.

## Critical Rules

1. Singular DocType names; DB table is `tab<Name>`.
2. Data/Link/Select ≤140 chars.
3. `search_index=1` on frequently filtered fields; `in_standard_filter=1` for user filters.
4. `fetch_if_empty=1` with `fetch_from` unless overwrite intended.
5. `depends_on` uses `eval:` JS syntax, not Python.
6. Don't `allow_on_submit` calc-affecting child fields without recalculating totals.