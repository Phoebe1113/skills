---
name: "frappe-workflow"
description: "Use when designing or configuring an ERPNext/Frappe Workflow: approval states, transitions, roles, doc_status, conditions, e.g. for AS9100 QMS approval flows."
---

# Frappe Workflow (v16, compact)

## Design steps
1. List every stage of the document (e.g. Draft → Chờ duyệt → Đã duyệt / Từ chối).
2. Map each state to `doc_status`: 0 Draft (editable), 1 Submitted, 2 Cancelled.
   - All approval-in-progress states = 0. NEVER 1 for intermediate states (submitted can't go back).
   - Non-submittable DocType: all states 0.
3. Each state: `allow_edit` role, optional `update_field` (e.g. `status`) + `update_value`.
4. Transitions: state → action → next_state, `allowed` role, optional `condition`.
5. Must have a doc_status=0 state before activating; activating sets `workflow_state` field (creates it if missing). Only one active workflow per DocType.

## Example (Workflow JSON / fixture)
```json
{
 "doctype": "Workflow", "workflow_name": "Duyệt NCR", "document_type": "Non Conformance",
 "is_active": 1, "workflow_state_field": "workflow_state", "send_email_alert": 1,
 "states": [
  {"state": "Draft", "doc_status": "0", "allow_edit": "QC User"},
  {"state": "Chờ duyệt", "doc_status": "0", "allow_edit": "QA Manager"},
  {"state": "Đã duyệt", "doc_status": "1", "allow_edit": "QA Manager"},
  {"state": "Từ chối", "doc_status": "0", "allow_edit": "QC User"},
  {"state": "Đã hủy", "doc_status": "2", "allow_edit": "QA Manager"}
 ],
 "transitions": [
  {"state": "Draft", "action": "Gửi duyệt", "next_state": "Chờ duyệt", "allowed": "QC User"},
  {"state": "Chờ duyệt", "action": "Duyệt", "next_state": "Đã duyệt", "allowed": "QA Manager", "allow_self_approval": 0},
  {"state": "Chờ duyệt", "action": "Từ chối", "next_state": "Từ chối", "allowed": "QA Manager"},
  {"state": "Từ chối", "action": "Gửi lại", "next_state": "Chờ duyệt", "allowed": "QC User"},
  {"state": "Đã duyệt", "action": "Hủy", "next_state": "Đã hủy", "allowed": "QA Manager"}
 ]
}
```
States and Actions must exist as `Workflow State` / `Workflow Action Master` records (UI creates them; in fixtures export them too).

## Conditions
Python expression on `doc`: `doc.grand_total > 50000000`, `doc.department == "QA"`. Allowed helpers: `frappe.session.user`, `frappe.utils.*`. Use for amount-based approval tiers (two transitions, opposite conditions).

## Rules
- `allow_self_approval: 0` for real approvals (creator can't approve; Administrator exempt).
- Workflow overrides Submit/Cancel buttons; permissions still need submit/cancel rights on the role.
- Notifications: `send_email_alert` or Notification DocType on `workflow_state` change; pending items in Workflow Action list.
- Changing an active workflow: existing docs in a removed state get stuck → map them first (`frappe.db.set_value(..., "workflow_state", ...)` in a patch).
- Ship via fixtures: `{"dt": "Workflow", "filters": [["name","in",["Duyệt NCR"]]]}` + Workflow State/Action Master.