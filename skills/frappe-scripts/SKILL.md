---
name: "frappe-scripts"
description: "Use when writing ERPNext/Frappe Server Scripts or Client Scripts (form JS): doc events, API, scheduler, permission query, sandbox rules (no imports), frm events, buttons, filters."
---

# Frappe Server + Client Scripts (v16, compact)

Scripts live in the DB (not in git). >50 lines, needs imports, or must survive rebuilds as code → move to custom app (frappe-impl-customapp).

## Server Script

Enable (v15+ off by default): `bench set-config -g server_script_enabled 1`

**NO `import` ever** (sandbox). Use: `frappe.parse_json/as_json`, `frappe.utils.today/now_datetime/add_days/date_diff/flt/cint/cstr`, `frappe.make_get_request/make_post_request`. No `open/eval/exec`, no `re/os`.

| Type | Variables |
|---|---|
| Document Event | `doc` |
| API (`/api/method/<name>`) | `frappe.form_dict` → set `frappe.response["message"]` |
| Scheduler (cron) | — ; end with `frappe.db.commit()` |
| Permission Query | `user` → set `conditions` (SQL string) |

UI event → hook: Before Save=`validate`, After Save=`on_update`, Before/After Submit=`before_submit/on_submit`, Before/After Cancel, Before Delete=`on_trash`.

Rules: never `doc.save()` in Before Save (loop); never `commit()` in doc events; SQL always parameterized `frappe.db.sql(q, {"x": v}, as_dict=1)`; stop with `frappe.throw(_("..."))`.

```python
# Before Save on Sales Invoice
if doc.grand_total < 0:
    frappe.throw("Tổng tiền không được âm")
doc.requires_approval = 1 if doc.grand_total > 10000000 else 0
```
```python
# Permission Query
conditions = "" if "System Manager" in frappe.get_roles(user) else f"`tabSales Invoice`.owner = {frappe.db.escape(user)}"
```

## Client Script (JS)

UI checks only; ALWAYS back critical rules server-side (Data Import/API bypass JS).

```js
frappe.ui.form.on('Sales Order', {
  setup(frm) {                       // filters go here
    frm.set_query('city', () => ({ filters: { state: frm.doc.state } }));
  },
  refresh(frm) {                     // buttons + initial state here
    if (!frm.is_new() && frm.doc.docstatus === 1)
      frm.add_custom_button(__('Tạo phiếu'), () => {/*...*/}, __('Create'));
    frm.toggle_display('discount', frm.doc.customer_group === 'VIP');
  },
  state(frm) { frm.set_value('city', ''); },   // clear dependent field
  async validate(frm) {
    if (flt(frm.doc.total) <= 0) frappe.throw(__('Total phải > 0'));  // throw stops save, msgprint doesn't
  }
});
frappe.ui.form.on('Sales Order Item', {          // child table
  qty: calc, rate: calc, items_remove: calc
});
function calc(frm, cdt, cdn) {
  const r = locals[cdt][cdn];
  if (r) frappe.model.set_value(cdt, cdn, 'amount', flt(r.qty) * flt(r.rate));
  frm.set_value('total', (frm.doc.items || []).reduce((s, i) => s + flt(i.amount), 0));
}
```

Server call: `const r = await frappe.call({method: 'app.api.fn', args: {...}}); r.message`.

Rules: `flt()` for numbers; handle `<table>_remove`; no `refresh_field` after `set_value`; buttons only in `refresh`; never query inside loops.