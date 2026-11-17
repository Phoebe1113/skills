---
name: "frappe-impl-customapp"
description: "Use when building or changing a custom Frappe/ERPNext v16 app: bench new-app, structure, hooks, patches, fixtures, apps.json, GitHub push, deploy to the Docker-based ERPNext."
---

# Frappe Custom App — Implementation (v16 focus)

Source: OpenAEC-Foundation/Frappe_Claude_Skill_Package (LGPL-3.0), adapted.

## USER RULE (always apply)

- EVERY custom app MUST be listed in `apps.json` and pushed to GitHub.
- NEVER create/edit an app directly inside a running container — a rebuilt image loses it.
- Flow: develop in a git repo → push to GitHub → add `{"url": "https://github.com/<org>/<app>", "branch": "main"}` to apps.json → rebuild image (frappe_docker `APPS_JSON_BASE64`) → `bench --site <site> install-app <app>` → `migrate`.

## Need an app at all?

```
Add fields to existing DocType → Custom Field + Property Setter (no app)
Simple validation/automation <50 lines → Server/Client Script (no app)
Complex logic, new DocTypes, Python imports, integrations → custom app
```
Start with the simplest option; but anything that must survive rebuilds as code belongs in the app (fixtures).

## Step 1: Create

```bash
bench new-app my_app   # in a dev bench, NOT in the production container
```
`my_app/__init__.py` must have `__version__ = "0.0.1"`.

## Step 2: pyproject.toml

```toml
[build-system]
requires = ["flit_core >=3.4,<4"]
build-backend = "flit_core.buildapi"

[project]
name = "my_app"
authors = [{ name = "Yoshita", email = "dev@example.com" }]
requires-python = ">=3.11"
readme = "README.md"
dynamic = ["version"]
dependencies = ["openpyxl>=3.1"]   # PyPI only

[tool.bench.frappe-dependencies]
frappe = ">=16.0.0,<17.0.0"
erpnext = ">=16.0.0,<17.0.0"
```
NEVER put frappe/erpnext in `[project].dependencies`.

## Step 3: hooks.py

```python
app_name = "my_app"
app_title = "My App"
app_publisher = "Yoshita"
app_license = "MIT"
required_apps = ["frappe", "erpnext"]
fixtures = []
```

## Step 4: Modules

`modules.txt` lists modules; `My App` → `my_app/`. 1–5 DocTypes: one module; more: split by business area. Each DocType in exactly one module.

## Step 5: DocTypes

Create in a dev site with `developer_mode=1` so JSON is written into the app:
```bash
bench --site dev.localhost set-config developer_mode 1
bench --site dev.localhost new-doctype "My Document" --module "My App"
```
Generates `doctype/my_document/{.json,.py,.js,test_*.py}`.

## Step 6: Hooks

```python
doc_events = {"Sales Invoice": {"validate": "my_app.events.sales_invoice.validate"}}

# v16 preferred way to extend a DocType class
extend_doctype_class = {"Sales Invoice": "my_app.overrides.sales_invoice.CustomSalesInvoice"}
# ALWAYS call super().method() in overrides

scheduler_events = {
    "daily": ["my_app.tasks.daily_cleanup"],
    "cron": {"0 9 * * 1-5": ["my_app.tasks.morning_report"]},
}
```

## Step 7: Patches

```python
# my_app/patches/v1_0/populate_defaults.py
import frappe
def execute():
    if not frappe.db.has_column("My DocType", "target_field"):
        return
    offset, size = 0, 1000
    while rows := frappe.get_all("My DocType", limit_page_length=size, limit_start=offset):
        for r in rows:
            frappe.db.set_value("My DocType", r.name, "target_field", "default", update_modified=False)
        frappe.db.commit()
        offset += size
```
```ini
# patches.txt
[pre_model_sync]
[post_model_sync]
my_app.patches.v1_0.populate_defaults
```
Guard clause, batch 1000, commit per batch, test on a DB copy first.

## Step 8: Fixtures (keeps UI customizations in git)

```python
fixtures = [
    {"dt": "Custom Field", "filters": [["module", "=", "My App"]]},
    {"dt": "Property Setter", "filters": [["module", "=", "My App"]]},
    {"dt": "Role", "filters": [["name", "in", ["My App User"]]]},
    {"dt": "Workflow", "filters": [["document_type", "=", "My DocType"]]},
]
```
```bash
bench --site dev.localhost export-fixtures --app my_app
```
Filter to your app only; never transactional data. Fixtures auto-import on migrate.

## Step 9: Dev cycle

```
schema/hooks/patches changed → bench --site S migrate
JS/CSS changed              → bench build --app my_app
Python changed              → bench --site S clear-cache (prod: bench restart)
```

## Step 10: Tests

```bash
bench --site dev.localhost run-tests --app my_app
```

## Step 11: Release & deploy

```bash
cd apps/my_app && git add -A && git commit -m "..." && git push
```
Bump `__version__` (MAJOR breaking / MINOR feature / PATCH fix). Then update apps.json, rebuild image, `install-app` (first time) and `migrate`.

## v16 notes

- Python ≥3.11; `extend_doctype_class` preferred over override_doctype_class
- Data masking per field; UUID naming rule; Chrome-based PDF (wkhtmltopdf deprecated)

## ALWAYS / NEVER

ALWAYS: bench new-app (no hand-made structure); `__version__` + `dynamic=["version"]`; filtered fixtures; versioned patch folders; test install on a fresh site; apps.json + GitHub.
NEVER: edit core ERPNext files; hardcode site values (use a Settings DocType); delete fields without a backup patch; create apps inside the production container.