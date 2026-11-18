---
name: "frappe-print"
description: "Use when creating ERPNext/Frappe Print Formats, Jinja templates, Letter Heads or generating PDFs (v16)."
---

# Frappe Print Formats (v16, compact)

## Which kind
| Need | Use |
|---|---|
| Hide/show fields only | Standard format + field `print_hide` / Customize Form |
| Drag-and-drop | Print Format Builder (or Print Designer app, v15+) |
| Full control | Custom HTML (Jinja), `print_format_type: Jinja` |
| Report print | JS microtemplate `{%= %}` — NEVER mix with Jinja `{{ }}` |

## Jinja template
```html
<div class="print-format">
  <h2>{{ doc.name }}</h2>
  <p>Khách hàng: {{ doc.customer_name }} — Ngày: {{ frappe.format(doc.posting_date, {'fieldtype':'Date'}) }}</p>
  <table class="table table-bordered">
    <tr><th>#</th><th>Mã</th><th>SL</th><th>Thành tiền</th></tr>
    {% for row in doc.items %}
    <tr><td>{{ row.idx }}</td><td>{{ row.item_code }}</td>
        <td class="text-right">{{ row.get_formatted("qty") }}</td>
        <td class="text-right">{{ row.get_formatted("amount", doc) }}</td></tr>
    {% endfor %}
  </table>
  <p>Tổng: {{ doc.get_formatted("grand_total") }}</p>
  {% if doc.terms %}<div>{{ doc.terms }}</div>{% endif %}
</div>
<style>.print-format{font-size:11pt}.print-format table{width:100%}</style>
```
Available: `doc`, `frappe`, `_()`, `frappe.db.get_value()`, `frappe.utils.*` (e.g. `money_in_words`), `get_formatted()`. Page break: `<div class="page-break"></div>`.

## PDF
```python
pdf = frappe.get_print("Sales Invoice", name, print_format="My Format", as_pdf=True, no_letterhead=0)
frappe.attach_print("Sales Invoice", name, print_format="My Format")  # for emails
```
URL: `/api/method/frappe.utils.print_format.download_pdf?doctype=X&name=Y&format=Z&no_letterhead=0`
v16: Chrome-based PDF generator (wkhtmltopdf deprecated) → modern CSS works; set page size/margins in Print Settings.

## Letter Head
Letter Head DocType: header/footer HTML or image; `is_default`. Template may use `{{ doc }}` in footer (e.g. page info).

## Ship in app
`my_app/<module>/print_format/<name>/<name>.json` with `"standard": "Yes"`, `"module"`, `"doc_type"`, `"html"` — create with developer_mode on. Site-only formats: add to fixtures.

## Rules
- No `frappe.get_doc()` inside loops (N+1); prepare data in Python.
- No heavy logic in Jinja; no huge base64 images (use URL + `max-width:100%`).
- Use relative widths / `.print-format`, not hardcoded page px.
- Vietnamese text: ensure a font with diacritics (default Inter / Noto Sans OK).