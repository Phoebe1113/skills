---
name: "frappe-testing"
description: "Use when testing a custom ERPNext/Frappe v16 app: unit/integration tests, test fixtures, running bench run-tests, a staging test site, and GitHub Actions CI."
---

# Frappe App Testing (v16, compact)

Source: OpenAEC-Foundation/Frappe_Claude_Skill_Package (LGPL-3.0), condensed.

## 1. Test site (never test on production)
```bash
bench new-site test.local --admin-password admin --install-app erpnext
bench --site test.local install-app my_app
bench --site test.local set-config allow_tests true
bench --site test.local set-config developer_mode 1
```
Docker: run in a separate compose project/stack, or `docker compose exec backend bench new-site test.local ...` on the same image (app comes from apps.json, never edited in-container). Copy prod data to staging: `backup --with-files` on prod → `restore` on test site.

## 2. Run tests
| Task | Command |
|---|---|
| Whole app | `bench --site test.local run-tests --app my_app` |
| One DocType | `... run-tests --doctype "My Doc"` |
| One method | `... run-tests --doctype "My Doc" --test test_submit` |
| One module | `... run-tests --module my_app.my_module.doctype.my_doc.test_my_doc` |
| Stop at first fail | `--failfast` |
| CI report | `--junit-xml-output report.xml` |
| Coverage | `--coverage` |

## 3. Base classes (v15+)
- `from frappe.tests import UnitTestCase` — pure logic, no DB.
- `from frappe.tests import IntegrationTestCase` — DB, doc lifecycle, permissions. Each test runs in a transaction rolled back afterwards.
- `FrappeTestCase` = old v14 wrapper, avoid in new code.
(Older import path `frappe.tests.classes` also works.)

File: `doctype/my_doc/test_my_doc.py` — must start with `test_`.

```python
import frappe
from frappe.tests import IntegrationTestCase

def make_doc(**kw):
    return frappe.get_doc({"doctype": "My Doc", "title": "_Test A", **kw}).insert()

class TestMyDoc(IntegrationTestCase):
    def test_lifecycle(self):
        d = make_doc(qty=5)
        self.assertEqual(d.docstatus, 0)
        d.submit(); self.assertEqual(d.docstatus, 1)
        d.cancel(); self.assertEqual(d.docstatus, 2)

    def test_validation(self):
        with self.assertRaises(frappe.ValidationError):
            make_doc(qty=-1)

    def test_permission(self):
        frappe.set_user("test1@example.com")
        self.assertFalse(frappe.has_permission("My Doc", "delete"))

    def tearDown(self):
        frappe.set_user("Administrator")
```
Useful helpers: `self.assertDocumentEqual(expected_dict, doc)`, `with self.set_user("x@y.com"):`, `with self.change_settings("Stock Settings", {"allow_negative_stock": 1}):`, `with self.freeze_time("2026-01-01"):`.

## 4. Test data
- Prefix values with `_Test` to separate from real data.
- Static: `test_records.json` in the doctype folder (linked records auto-created).
- Complex: helper function guarded by `frappe.flags.<name>_created`.
- ERPNext standard test records exist (`_Test Company`, `_Test Customer`, `_Test Item`) when erpnext tests are set up; safer to create your own.
- Excel import tool: keep small sample .xlsx in `my_app/tests/fixtures/`, assert row counts, fieldtype conversion (dates, numbers, diacritics) and error rows.

## 5. GitHub Actions CI (.github/workflows/ci.yml)
```yaml
name: CI
on: [push, pull_request]
jobs:
  test:
    runs-on: ubuntu-latest
    services:
      mariadb:
        image: mariadb:11.4
        env: { MARIADB_ROOT_PASSWORD: root }
        ports: ["3306:3306"]
        options: --health-cmd="healthcheck.sh --connect" --health-interval=5s --health-retries=10
      redis-cache: { image: redis:alpine, ports: ["13000:6379"] }
      redis-queue: { image: redis:alpine, ports: ["11000:6379"] }
    steps:
      - uses: actions/checkout@v4
        with: { path: app }
      - uses: actions/setup-python@v5
        with: { python-version: "3.12" }
      - uses: actions/setup-node@v4
        with: { node-version: "20" }
      - run: pip install frappe-bench
      - run: |
          bench init --skip-assets --skip-redis-config-generation --frappe-branch version-16 ~/b
          cd ~/b
          bench set-config -g db_root_password root
          bench set-config -g redis_cache redis://localhost:13000
          bench set-config -g redis_queue redis://localhost:11000
          bench get-app erpnext --branch version-16 --skip-assets
          bench get-app --skip-assets my_app $GITHUB_WORKSPACE/app
          bench setup requirements --dev
          bench new-site test_site --db-root-password root --admin-password admin --no-mariadb-socket
          bench --site test_site install-app erpnext my_app
          bench --site test_site set-config allow_tests true
          bench --site test_site run-tests --app my_app --failfast
```
Drop the erpnext lines if the app only needs frappe. Push → tests run automatically before the app goes into apps.json/rebuild.

## Rules
- Never run tests on the production site (tests write data).
- Don't call `frappe.db.commit()` in tests (breaks rollback isolation).
- Reset user in tearDown; one behaviour per test method.
- Test both happy path and validation errors.