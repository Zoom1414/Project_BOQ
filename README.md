# Construction BOQ & Labor Payment System

ระบบควบคุมงบประมาณและเบิกจ่ายค่าแรงตามงวดงานก่อสร้าง

The application includes the models, migrations, validation, budget calculations,
transactional approval services, and a working Master BOQ workspace. PostgreSQL is the
configured database; Ruby 3.4.1 is pinned in `.ruby-version`.

## Setup

```sh
bundle install
bin/rails db:prepare
gem install foreman # once, if not already installed
bin/dev
```

Run in a Ruby environment (WSL on this Windows workspace). PostgreSQL must be running,
with a role able to create the development and test databases. Use `DATABASE_URL`
if your credentials differ from local peer authentication. The migration preserves
existing projects/plans and assigns `LEGACY-P-<id>` / `LEGACY-HP-<id>` codes.

`bin/dev` builds Tailwind once before starting Rails and the CSS watcher together
using `Procfile.dev`. Stop an old Rails process with Ctrl+C before restarting it.
The default port is 3000; use `PORT=3001 bin/dev` to select another port.
If starting Rails alone, run `bin/rails tailwindcss:build` after changing templates
or styles. The layout explicitly loads the compiled `tailwind.css` asset.

Devise supplies login/password reset. Public registration is disabled. Create
accounts through `bin/rails console` with your own strong passwords:

```ruby
User.create!(name: "Administrator", email: "admin@your-company.com",
  password: "replace-with-a-unique-strong-password", role: :admin)
User.create!(name: "Engineer", email: "engineer@your-company.com",
  password: "replace-with-another-strong-password", role: :project_engineer)
```

Configure Action Mailer delivery and `default_url_options` before using password
reset outside development. `DEVISE_MAILER_SENDER` configures the sender.

## Master BOQ workspace

Open `/` or `/master_boq` after logging in. Select a project and house plan in the
sidebar, then create its Master BOQ if it does not exist. Dev, Admin, and Project
Engineer can create categories and items; User can view and export them.

The spreadsheet groups items by category with material and labor columns, sticky
headers, budget totals, pending PO quantities, and approved balances. Search by
item, code, category, contractor, or note; collapse categories or expand the workspace.
Click an item name to edit it in a Turbo Frame dialog. Saving updates the worksheet
and budget summary through Turbo Streams without a page reload. Totals are computed
on the server; editing does not permit changes to paid or used balances. Concurrent
edits return a conflict instead of silently overwriting newer values.

Export CSV downloads editable budget inputs with these headers:

```csv
category,code,name,unit,material_quantity,material_unit_price,labor_unit_price,contractor_name,note
```

Import accepts UTF-8 CSV up to 2 MB and 500 items. It creates categories as needed
and adds new items; it does not replace existing items or import paid/used balances.
An invalid row or duplicate item code within a category rolls back the entire file.
The backup button downloads a JSON snapshot of the selected BOQ and its balances;
restoring JSON is not implemented. Each successful form submission saves immediately.

## Purchasing, labor payments, and approvals

All authenticated pages share the SITEWORK workspace, project/plan selectors, and
navigation. Login and project/plan management use the same navy and amber theme.
Tables scroll horizontally on small screens; document details have a print layout.
Sidebar links navigate the `workspace_content` Turbo Frame with history enabled,
keeping the sidebar in place. Stimulus synchronizes the active menu and page title
after frame navigation and restores sidebar scroll on full visits.

The overview includes a work-progress donut and category bars. Values come from
each BOQ item's `progress_percentage`, weighted by its material plus labor budget;
zero-budget groups use a simple average. Empty BOQs show 0% and an empty chart.
These charts represent recorded work completion, independently of payments. Edit
an item's progress in Master BOQ to update the charts on the next overview visit.

| Page | Path | Workflow |
| --- | --- | --- |
| Overview | `/dashboard` | Selected plan budget, approved material/labor spending, pending requests |
| Create PO | `/purchase_orders/new` | BOQ picker, outside-BOQ items, editable quantities/prices, live totals |
| PO history | `/purchase_orders` | Status, supplier, amount, document detail and printing |
| Pending PU | `/purchase_orders/pending` | Admin opens a Turbo Frame dialog to enter actual prices |
| PU history | `/purchase_orders/history` | Approved receipts, PU/PO references, actual dates and totals |
| Labor draw | `/labor_draw_requests/new` | Admin or Engineer selects contractor; assigned BOQ rows load in a Turbo Frame |
| DV history | `/labor_draw_requests` | Approved monthly totals with expandable documents and all-request history |
| Approvals | `/approvals` | Pending, approved, rejected, and all PO/DV documents |

PO and DV creation is available to Admin and Project Engineer. The requester is
always the signed-in account. Engineer DV requests remain pending; Admin DV
submissions immediately approve and deduct BOQ balances in one transaction through
`SubmitLaborDrawService`. Only Admin can approve or reject other requests.

The labor form shows available balances, optional budget details, a fill-remaining
shortcut, live totals, and a sticky submit button. Switching contractors preserves
entered amounts during the current page visit. A unique submission key makes
retries of the same form idempotent. No draft is saved until submission.

PO and Engineer DV submissions return Turbo Stream budget warnings for overruns
without deducting balances. Every approval rechecks current balances under locks.
Over-budget approval requires an explicit Admin checkbox and reason, including
immediate Admin draws. Failed immediate approval rolls back the whole submission.
Rejection requires a note and never changes balances. Approval supports an optional
Admin note stored atomically with the audit and balance changes.
PO, PU, and DV numbers use monthly counters protected by database row locks. PU
completion saves actual prices, the receipt date/number, approval audit, and BOQ
deductions in one transaction. Retrying approval never deducts twice. Actual purchase
prices appear beside the original PO prices on the printable document.

Outside-BOQ PO lines are explicitly labeled and have no BOQ balance to deduct. They
still require Admin approval and contribute to approved spending in the dashboard.
DV requests always refer to BOQ items assigned to the selected contractor. DV monthly
summaries include only approved requests, grouped by request date; pending requests
remain visible in the all-request table. No bank transfer or external payment occurs.

## Architecture

```text
Project -> HousePlan -> MasterBoq -> BoqCategory -> BoqItem
Project / HousePlan -> PurchaseOrder -> PoItem -> BoqItem
Project / HousePlan / User -> LaborDrawRequest -> LaborDrawItem -> BoqItem
```

All requested fields are included. Foreign keys, unique indexes, required columns,
status constraints, and model validations protect the hierarchy. Project codes are
globally unique; plan codes are unique within a project, item codes within a category,
and each plan has one master BOQ. Each BOQ item appears at most once per PO/DV.
Referenced records and approved documents cannot be deleted through models.

Money uses `decimal(18,2)`, quantities use `decimal(18,4)`, and progress is 0–100.
Approval adds `approved_by`, `approved_at`, `budget_override`, `override_reason`, and
optimistic locking. Bangkok is the application time zone.

### Calculations

- `material_total = round(material_quantity * material_unit_price, 2)`.
- `labor_total = round(material_quantity * labor_unit_price, 2)`: the requested
  schema has one quantity, shared by material and labor calculations.
- Remaining quantity = planned quantity minus approved used quantity.
- Remaining labor = planned labor total minus approved paid amount.
- Master BOQ budgets roll up item totals after item creation, changes, and deletion.
- PO totals use actual material price when present (including 0), otherwise the
  estimate. Each material/labor line component rounds to cents.
- DV `requested_amount` is an explicit installment, not necessarily quantity × rate.
  Snapshot fields show budget, paid-before-this-draw, and remaining-after-this-draw.
  Approval refreshes those snapshots under locks.

`BoqItem#pending_labor_amount` combines pending DV installments and pending-PU labor.
`pending_material_quantity` counts pending-PU quantities. Drafts/rejections are
excluded. Pending figures are informational and do **not** reserve or deduct budget.
Approval always rechecks the latest committed balance.

PO approval consumes both quantity and PO labor, per BR-01/02. If a PO should only
commit labor for later DV payment, that requires a separate commitment ledger; do
not enter the same payment in both documents.

### Roles

| Role | Existing projects/plans | DV creation | Approval / override |
| --- | --- | --- | --- |
| Dev | Read/write | No | No |
| Admin | Read/write | Immediate approval | Yes |
| Project Engineer | Read/write | Own requests | No |
| User | Read | No | No |

Dev has no implicit financial approval privilege. Services authorize the freshly
loaded actor using Pundit. Project membership was not part of the schema, so
authenticated users can read all projects.

## Create and approve a labor draw

```ruby
item = BoqItem.first!
plan = item.master_boq.house_plan
engineer = User.project_engineer.first!
admin = User.admin.first!

draw = LaborDrawRequest.create!(
  dv_number: "DV-2026-0001", project: plan.project, house_plan: plan,
  user: engineer, contractor_name: item.contractor_name, request_date: Date.current,
  labor_draw_items_attributes: [
    { boq_item_id: item.id, work_description: "Concrete installment 1",
      quantity: 2, unit_price: item.labor_unit_price, requested_amount: 500 }
  ]
)

draw.budget_warnings # Structured warning data; no balance changes.
approved = ApproveLaborDrawService.new(draw, actor: admin).call

# Alternative for an over-budget pending draw, after an explicit Admin decision:
approved = ApproveLaborDrawService.new(
  draw, actor: admin, override: true, override_reason: "Approved scope variation"
).call
```

`call` returns a fresh approved record. It locks the persisted document and BOQ rows
in ID order, validates all lines, checks budgets, updates balances, and records
approval in one transaction. Duplicate approval is idempotent. Competing requests
serialize against shared BOQ rows. Exceptions roll back all writes.

Controllers handle `ApproveLaborDrawService::BudgetExceeded` (with `warnings`), `InvalidState`,
`OverrideReasonRequired`, `Pundit::NotAuthorizedError`, and normal Active Record
validation/stale-write errors without changing balances on failure.

Use `ApprovePurchaseOrderService.new(po, actor: admin).call` for a `pending_pu` PO.
Every PU line must have an actual material price before approval. Drafts/rejections
cannot be approved. Both services accept the same explicit override options.
Approved documents are immutable; reversals need a separate audited workflow.

### Turbo warning integration

PO/DV controllers use a tested helper and alert partial.
Place `<div id="budget_warnings"></div>` in the form and append this helper to the
Turbo Stream create/update response after saving a pending request:

```ruby
render turbo_stream: helpers.budget_warning_stream(@labor_draw_request)
```

The alert reads: "Warning: This request exceeds the Master BOQ budget limit. Require
Admin approval override." The helper clears old warnings when within budget.
Over-budget pending requests persist; approval requires an Admin override and reason.
Negative remaining values are intentionally supported after an override.

Only permit business inputs in controllers. Do not permit computed totals,
BOQ balances, audit fields, roles, `approval_in_progress`, or
`balance_update_in_progress`. Assign requester/actor from `current_user`. Model
callbacks protect ordinary saves; direct SQL, `update_columns`, `update_all`, and
bulk imports bypass them and must not be used for financial changes.

## Verification

```sh
RAILS_ENV=test bin/rails db:prepare
bin/rails test
bin/rails zeitwerk:check
bin/rubocop
```

Tests cover rounding/rollups, hierarchy/contractor validation, pending balances,
Turbo output, authentication/roles, PU actual prices, exact/over-budget approval,
audit fields, immutability, duplicate approvals, transaction rollback, and simultaneous
PostgreSQL approvals through separate connections.

Controller and import tests also cover BOQ editing, role restrictions, cross-BOQ
access, stale edits, CSV export protection, atomic CSV imports, PO/PU/DV workflows,
outside-BOQ lines, procurement rollback, repeated approval, and dashboard totals
for documents with multiple line items.

References: [Rails row locking](https://api.rubyonrails.org/classes/ActiveRecord/Locking/Pessimistic.html)
and [Devise configuration](https://github.com/heartcombo/devise).

#สร้าง Users
bin/rails console
User.create!(name: "Administrator", email: "admin@your-company.com", password: "รหัสผ่านที่ปลอดภัยของคุณ", role: :admin)

u = User.find_by!(email: "admin@your-company.com") //หาUser
u.update!(password: "รหัสผ่านใหม่อย่างน้อย12ตัว", role: :admin) //แก้รหัส
