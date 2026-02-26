# FPMS MVP1 Enhancement Plan — Claude Edition

> **Author**: Claude (synthesized from Gemini, GPT-5.2, GPT-5.3 plans + independent codebase audit)
> **Date**: 2026-02-23
> **Goal**: (1) Close MVP1 Scope GAPs, (2) Align with FPMS SPEC 2.0 (interface-first where template data unavailable)
> **Discipline**: Strict sequential batches. Each batch MUST pass quality gate before next batch starts.

---

## 0. Execution Discipline

### 0.1 Iron Rules

1. **Batch order is MANDATORY** — no skipping, no reordering
2. **Quality gate per batch** — every batch ends with `ruff check && ruff format && pytest -q && alembic upgrade head` passing
3. **Forward-only migrations** — never `alembic downgrade`; for clean rebuild: delete `.db`, then `alembic upgrade head && python scripts/seed_dev.py`
4. **One concern per PR** — each batch = one merge-ready PR
5. **SQLite compatibility** — `server_default=sa.text(...)`, UUIDs as TEXT, no PG-only functions
6. **No scope creep** — frozen features stay frozen (see Section 0.3)
7. **Evidence required** — every batch records pass/fail of quality gate commands

### 0.2 Quality Gate (run after every batch)

```bash
cd backend
ruff check --fix . && ruff format .    # auto-fix lint
ruff check .                            # must pass clean
pytest -q                               # all tests pass
alembic upgrade head                    # migrations apply cleanly
python scripts/seed_dev.py              # seed runs without error
uvicorn app.main:app --port 8000 &      # server starts
sleep 3
curl -sf http://localhost:8000/healthz   # health check passes
kill %1
```

### 0.3 Frozen Features (NOT in any batch)

| Feature | Reason |
|---------|--------|
| PCT international + national plan | MVP1 out-of-scope |
| Annual fee batch, grace rules | MVP1 out-of-scope |
| Invalidation / litigation workflows | MVP1 out-of-scope |
| Dunning, bad debt, complex finance reports | MVP1 out-of-scope |
| Commission calculation & settlement | MVP1 out-of-scope |
| Full template builder UI | MVP1 out-of-scope |
| Full-text search / Elasticsearch | MVP1 out-of-scope |
| Multi-currency exchange rate engine | Post-MVP1 |
| Consulting/Search project management | MVP1 out-of-scope |
| Statistical/aggregate reports | Post-MVP1 |

---

## 1. Current Codebase Reality (Audit Results)

Before planning, I audited the actual code. The GAP analysis documents were written before several ENH tasks were completed. The real state is:

### Already Implemented (model + service + API)

| Feature | Model | Service | API | Notes |
|---------|-------|---------|-----|-------|
| TaskTemplate | `t_task_template` (code, name, enabled) | Referenced in TaskGenerationService | No CRUD API | **Missing**: deadline calc fields (add_days etc.) |
| TaskLog | `t_task_log` (task_id, action, from/to_status, remark) | `_create_task_log()` in service | No read API | **Missing**: `GET /tasks/{id}/logs` |
| Task worker/supervisor | `worker_id`, `supervisor_id`, `internal_due_date`, `base_date` on Task | `list_tasks_today(as_role=WORKER\|SUPERVISOR)` | `GET /tasks/today?as=worker` | Working |
| DocAttachment | `t_doc_attachment` (multi-file) | `add_attachment()`, `get_attachment_download()` | `POST/GET /documents/{id}/attachments` | Working |
| DocTemplate | `t_doc_template` (code, name, direction, enabled) | Referenced in document create | No CRUD API | **Missing**: SPEC config fields |
| Doc→Task auto-link | `TaskGenerationService.generate_from_document()` | Wired into `POST /documents` | Returns `X-Auto-Tasks-Created` header | **Missing**: template.offset_days field → RuntimeError |
| SystemParam | `t_system_param` (key, value, type, secret, updated_by) | Used in task sheet print | No CRUD API | **Missing**: list/upsert API |
| Bill direction + breakdowns | `direction` (AR/AP), `total_gov/service/misc` on Bill | — | — | Model complete |
| PaymentLine | `t_payment_line` (raw_amount, allocated_amt, balance_amt) | — | — | Model complete |
| Offset reversal | `is_reversed`, `reversed_at` on Offset | — | — | Model fields exist |
| Case sub-tables | T_CaseApplicant, T_CaseInventor, T_Priority as normalized tables | CRUD via CaseCreate/Update | Nested in case API | Working |
| FeeDraft breakdowns | `total_gov/service/misc` on FeeDraft | — | — | Model complete |
| FeeItem enhanced | `rate_id`, `year_no` on FeeItem | — | — | Model complete |

### Actually Missing (verified against code)

| Gap | Priority | Batch |
|-----|----------|-------|
| TaskTemplate: no `add_days`/`add_months`/`inner_offset_days`/remind offsets | P0 | A1 |
| TaskTemplate CRUD API + seed data | P0 | A1 |
| TaskLog read API (`GET /tasks/{id}/logs`) | P0 | A1 |
| Client: no T_ClientAddress / T_ClientContact tables | P0 | A2 |
| Case: missing ~20 NORMAL-type fields from SPEC | P1 | A3 |
| Document: no reply chain fields (reply_to_id, need_reply, reply_date) | P1 | B2 |
| SystemParam CRUD API | P1 | A4 |
| Advanced case search filters (client_id, case_type, date ranges) | P1 | A5 |
| DocTemplate CRUD API | P1 | B1 |
| DocTemplate SPEC config fields (status_effect, deadline_template_code, fee_draft_type) | P1 | B1 |
| FeeRate: no group/country/case_type/calc_mode dimensions | P2 | B4 |
| CaseReceipt: missing fee_code/year_no/is_arrears/invoice_no | P2 | B5 |

---

## 2. Batch Execution Plan

### Dependency Graph

```
A0 (Baseline Verify)
 └─→ A1 (TaskTemplate + TaskLog API)
      └─→ A2 (Client Address/Contact)
           └─→ A3 (Case Field Expansion)
                └─→ A4 (SystemParam API)
                     └─→ A5 (Advanced Case Search)
                          └─→ ═══ MVP1 SCOPE GATE ═══
                               └─→ B1 (DocTemplate Enhancement)
                                    ├─→ B2 (Doc Reply Chain + Auto Write-off)
                                    └─→ B3 (Doc→FeeDraft Linking Stub)
                                         └─→ B4 (FeeRate Dimensions + CalcMode Stub)
                                              └─→ B5 (CaseReceipt + Billing Polish)
                                                   └─→ B6 (Search & Filter Polish)
                                                        └─→ ═══ SPEC ALIGNMENT GATE ═══
```

---

## Batch A0 — Baseline Verification

**Goal**: Verify the 4 MVP1 success criteria actually work end-to-end. Fix any blockers found.

**Duration**: ~1 hour. NO new features.

### Verification Script

```bash
# === MVP1 Success Criterion 1: Case CRUD ===
# Login
FPMS_TOKEN=$(curl -sf -X POST http://localhost:8000/api/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"admin123"}' | python3 -c "import sys,json; print(json.load(sys.stdin)['access_token'])")

# Create case
CASE_ID=$(curl -sf -X POST http://localhost:8000/api/v1/cases \
  -H "Authorization: Bearer $FPMS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"case_no":"TEST-A0-001","case_type":"NORMAL","patent_category":"INV","flow_dir":"CN_DOMESTIC","title_cn":"基线验证案件"}' \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['id'])")

# Search case
curl -sf "http://localhost:8000/api/v1/cases?q=TEST-A0" \
  -H "Authorization: Bearer $FPMS_TOKEN" | python3 -c "import sys,json; d=json.load(sys.stdin); assert d['total']>=1"

# Get case detail
curl -sf "http://localhost:8000/api/v1/cases/$CASE_ID" \
  -H "Authorization: Bearer $FPMS_TOKEN" | python3 -c "import sys,json; assert json.load(sys.stdin)['case_no']=='TEST-A0-001'"

echo "✅ Criterion 1 PASSED: Case CRUD works"

# === MVP1 Success Criterion 2: Document + Task auto-link ===
# Create document (IN direction)
DOC_ID=$(curl -sf -X POST http://localhost:8000/api/v1/documents \
  -H "Authorization: Bearer $FPMS_TOKEN" \
  -H "Content-Type: application/json" \
  -d "{\"case_id\":\"$CASE_ID\",\"direction\":\"IN\",\"doc_date\":\"2026-02-23\",\"title\":\"OA Notification\"}" \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['id'])")
# Note: Auto-task depends on TaskTemplate with offset_days — may fail until A1

echo "✅ Criterion 2 PARTIAL: Document created (auto-task requires Batch A1)"

# === MVP1 Success Criterion 3: Fee draft → Bill → Payment → Offset ===
# (Verify these endpoints respond correctly)
curl -sf "http://localhost:8000/api/v1/fees/drafts?page=1" \
  -H "Authorization: Bearer $FPMS_TOKEN" > /dev/null && echo "  Fee drafts endpoint: OK"
curl -sf "http://localhost:8000/api/v1/billing/bills?page=1" \
  -H "Authorization: Bearer $FPMS_TOKEN" > /dev/null && echo "  Bills endpoint: OK"
curl -sf "http://localhost:8000/api/v1/billing/payments?page=1" \
  -H "Authorization: Bearer $FPMS_TOKEN" > /dev/null && echo "  Payments endpoint: OK"

echo "✅ Criterion 3 PASSED: Fee→Bill→Payment chain accessible"

# === MVP1 Success Criterion 4: Word bill template ===
echo "⚠️  Criterion 4: Manual verification — requires template file + bill data"
```

### Agent Team Prompt (A0)

> **Do NOT use a team for A0.** Run this as a single agent or manually.
> Execute the verification script above.
> If any endpoint returns 404/500, investigate and fix the root cause.
> Document results in `artifacts/A0-baseline/summary.md`.
> Do NOT add new features. Only fix broken existing functionality.

---

## Batch A1 — TaskTemplate Enhancement + TaskLog API

**Goal**: Make the Doc→Task auto-generation actually work by adding deadline calculation fields to TaskTemplate, providing CRUD API for TaskTemplate, adding TaskLog read API, and seeding 2-3 starter templates.

**Why first**: This is the #1 blocker for MVP1 Success Criterion 2 ("OA incoming → auto-creates deadline task"). The `TaskGenerationService` already exists and is wired into `POST /documents`, but crashes with `RuntimeError: TaskTemplate missing offset_days mapping` because the model has no such field.

### Scope

**Backend changes:**

1. **Migration**: Add fields to `t_task_template`:
   - `add_days: Integer, nullable=True` — days to add to base_date for due_date calculation
   - `add_months: Integer, nullable=True, server_default=text("0")` — months to add
   - `inner_offset_days: Integer, nullable=True` — days before due_date for internal_due_date
   - `default_worker_role: String(32), nullable=True` — role hint for auto-assignment
   - `description: Text, nullable=True`

2. **`backend/app/modules/tasks/schemas.py`**: Add TaskTemplate schemas:
   - `TaskTemplateCreateIn(code, name, add_days, add_months, inner_offset_days, default_worker_role, description)`
   - `TaskTemplateUpdateIn(name?, add_days?, add_months?, inner_offset_days?, default_worker_role?, enabled?, description?)`
   - `TaskTemplateOut(id, code, name, add_days, add_months, inner_offset_days, default_worker_role, enabled, description, created_at, updated_at)`

3. **`backend/app/modules/tasks/service.py`**: Add functions:
   - `list_task_templates(db, enabled_only=False) -> list[TaskTemplate]`
   - `get_task_template(db, template_id) -> TaskTemplate`
   - `create_task_template(db, data) -> TaskTemplate`
   - `update_task_template(db, template_id, data) -> TaskTemplate`
   - `list_task_logs(db, task_id) -> list[TaskLog]`

4. **`backend/app/modules/tasks/api.py`**: Add endpoints:
   - `GET /task-templates` — list all templates
   - `POST /task-templates` — create template (perm: `TaskTemplate.Create`)
   - `PUT /task-templates/{id}` — update template (perm: `TaskTemplate.Edit`)
   - `GET /tasks/{task_id}/logs` — list task audit logs (perm: `Task.Read`)

5. **`backend/app/modules/tasks/task_generation_service.py`**: Fix `_get_offset_days()` to read `template.add_days` (the actual field name). Update `generate_from_document()` to also calculate `internal_due_date` using `template.inner_offset_days`.

6. **`backend/scripts/seed_dev.py`**: Seed 2-3 starter TaskTemplates:
   - `OA_REPLY` — code="OA_REPLY", name="OA答复期限", add_days=120, inner_offset_days=14
   - `GRANT_FEE` — code="GRANT_FEE", name="授权登记费", add_days=60, inner_offset_days=7

7. **`backend/app/modules/rbac/service.py`** (or seed): Register permissions `TaskTemplate.Create`, `TaskTemplate.Edit`, `TaskTemplate.Read` and bind to Admin role.

**Non-scope**: No frontend changes. No DocTemplate changes. No fee linking.

### Files Modified

| File | Change |
|------|--------|
| `backend/alembic/versions/a1_task_template_fields.py` | NEW migration |
| `backend/app/modules/tasks/models.py` | Add fields to TaskTemplate |
| `backend/app/modules/tasks/schemas.py` | Add TaskTemplate + TaskLog schemas |
| `backend/app/modules/tasks/service.py` | Add CRUD + list_task_logs |
| `backend/app/modules/tasks/api.py` | Add 4 endpoints |
| `backend/app/modules/tasks/task_generation_service.py` | Fix offset_days → add_days |
| `backend/app/api/router.py` | Ensure task-templates routes included |
| `backend/scripts/seed_dev.py` | Seed starter templates + permissions |
| `backend/tests/test_task_template.py` | NEW: test CRUD + auto-generation |

### Acceptance Criteria

- [ ] `GET /api/v1/task-templates` returns seeded templates
- [ ] `POST /api/v1/task-templates` creates a template with `add_days=90`
- [ ] `GET /api/v1/tasks/{id}/logs` returns audit entries
- [ ] Creating a document with direction=IN and title containing "OA" auto-creates a task with correct `due_date = doc_date + add_days`
- [ ] Auto-created task has `internal_due_date = due_date - inner_offset_days`
- [ ] Duplicate document does NOT create duplicate task (idempotency)
- [ ] Quality gate passes

### Agent Team Prompt (A1)

```
You are implementing Batch A1 of the FPMS MVP1 Enhancement Plan.

## Context
FPMS is an IP law firm management system. Backend: FastAPI + SQLAlchemy + SQLite.
The TaskTemplate table exists but lacks deadline calculation fields.
The TaskGenerationService exists and is wired into POST /documents, but crashes
because TaskTemplate has no `add_days` field.

## Your Task
1. Create an Alembic migration adding these columns to t_task_template:
   - add_days: Integer, nullable=True
   - add_months: Integer, nullable=True, server_default=text("0")
   - inner_offset_days: Integer, nullable=True
   - default_worker_role: String(32), nullable=True
   - description: Text, nullable=True
   Remember: SQLite compatible. Use `op.add_column` with `batch_alter_table`.

2. Update backend/app/modules/tasks/models.py — add the new mapped_columns to TaskTemplate.

3. Update backend/app/modules/tasks/schemas.py — add:
   - TaskTemplateCreateIn, TaskTemplateUpdateIn, TaskTemplateOut

4. Update backend/app/modules/tasks/service.py — add:
   - list_task_templates(db, enabled_only) -> list
   - get_task_template(db, template_id) -> TaskTemplate
   - create_task_template(db, data) -> TaskTemplate
   - update_task_template(db, template_id, data) -> TaskTemplate
   - list_task_logs(db, task_id) -> list[TaskLog]

5. Update backend/app/modules/tasks/api.py — add endpoints:
   - GET /task-templates (perm: TaskTemplate.Read)
   - POST /task-templates (perm: TaskTemplate.Create)
   - PUT /task-templates/{id} (perm: TaskTemplate.Edit)
   - GET /tasks/{task_id}/logs (perm: Task.Read)

6. Fix backend/app/modules/tasks/task_generation_service.py:
   - _get_offset_days() should read template.add_days
   - generate_from_document() should set internal_due_date = due_date - timedelta(days=template.inner_offset_days) when inner_offset_days is not None

7. Update backend/scripts/seed_dev.py — seed 2 TaskTemplates:
   - OA_REPLY: add_days=120, inner_offset_days=14
   - GRANT_FEE: add_days=60, inner_offset_days=7
   Also seed permissions: TaskTemplate.Create, TaskTemplate.Edit, TaskTemplate.Read (bind to Admin role)

8. Write tests in backend/tests/test_task_template.py

## Constraints
- Use pattern: Depends(require_perm("X")) for permission enforcement
- UUIDs: str(uuid4()), stored as String(36)
- Error handling: raise_business_error(code, message, status_code=...)
- Follow existing code patterns in tasks/service.py and tasks/api.py
- Do NOT modify any files outside the listed scope

## Quality Gate (MUST pass)
cd backend && ruff check --fix . && ruff format . && ruff check . && pytest -q
```

---

## Batch A2 — Client Multi-Address & Contact

**Goal**: Add T_ClientAddress and T_ClientContact tables so a client can have multiple addresses (billing, mailing, etc.) and contacts, per MVP1 scope requirement "Client with addresses & contacts".

### Scope

**Backend changes:**

1. **Migration**: Create tables:
   - `t_client_address(id, client_id FK, address_type: "BILLING"|"MAILING"|"GENERAL", address_line1, address_line2, city, province, postal_code, country_code, is_default)`
   - `t_client_contact(id, client_id FK, contact_name, title, phone, mobile, email, is_primary)`

2. **`backend/app/modules/masterdata/clients/models.py`**: Add `ClientAddress` and `ClientContact` ORM models.

3. **`backend/app/modules/masterdata/clients/schemas.py`**: Add schemas for nested address/contact in client detail response + create/update inputs.

4. **`backend/app/modules/masterdata/clients/service.py`**: Extend client detail to include addresses/contacts. Add CRUD for addresses and contacts as sub-resources.

5. **`backend/app/modules/masterdata/clients/api.py`**: Add endpoints:
   - `GET /clients/{id}/addresses` + `POST` + `PUT/{addr_id}` + `DELETE/{addr_id}`
   - `GET /clients/{id}/contacts` + `POST` + `PUT/{contact_id}` + `DELETE/{contact_id}`

**Non-scope**: No frontend changes. No changes to billing address resolution logic.

### Files Modified

| File | Change |
|------|--------|
| `backend/alembic/versions/a2_client_address_contact.py` | NEW migration |
| `backend/app/modules/masterdata/clients/models.py` | Add ClientAddress, ClientContact |
| `backend/app/modules/masterdata/clients/schemas.py` | Add address/contact schemas |
| `backend/app/modules/masterdata/clients/service.py` | Add CRUD functions |
| `backend/app/modules/masterdata/clients/api.py` | Add sub-resource endpoints |
| `backend/tests/test_client_address.py` | NEW tests |

### Acceptance Criteria

- [ ] Create a client, add 2 addresses (BILLING + MAILING), verify both returned in `GET /clients/{id}`
- [ ] Add 2 contacts to client, verify returned
- [ ] Delete an address, verify removed
- [ ] Quality gate passes

### Agent Team Prompt (A2)

```
You are implementing Batch A2 of the FPMS MVP1 Enhancement Plan.

## Context
FPMS backend: FastAPI + SQLAlchemy + SQLite.
The Client model exists at backend/app/modules/masterdata/clients/models.py
with fields: id, client_code, name_cn, name_en, email, client_type, default_currency, is_active.
MVP1 scope requires "Client with addresses & contacts" but currently only single inline fields exist.

## Your Task
1. Create migration adding t_client_address and t_client_contact tables:

   t_client_address:
   - id: String(36) PK (UUID)
   - client_id: String(36) FK -> t_client.id ON DELETE CASCADE, NOT NULL
   - address_type: String(16), NOT NULL, server_default="GENERAL"  (BILLING|MAILING|GENERAL)
   - address_line1: Text, nullable
   - address_line2: Text, nullable
   - city: String(128), nullable
   - province: String(128), nullable
   - postal_code: String(20), nullable
   - country_code: String(10), nullable, server_default="CN"
   - is_default: Boolean, NOT NULL, server_default=text("0")
   - AuditMixin columns (created_at, updated_at, created_by, updated_by)

   t_client_contact:
   - id: String(36) PK (UUID)
   - client_id: String(36) FK -> t_client.id ON DELETE CASCADE, NOT NULL
   - contact_name: String(200), NOT NULL
   - title: String(100), nullable
   - phone: String(50), nullable
   - mobile: String(50), nullable
   - email: String(254), nullable
   - is_primary: Boolean, NOT NULL, server_default=text("0")
   - AuditMixin columns

2. Add ORM models to clients/models.py (use UUIDPrimaryKeyMixin + AuditMixin)
3. Add Pydantic schemas to clients/schemas.py
4. Add service functions to clients/service.py
5. Add API endpoints to clients/api.py:
   - GET/POST /clients/{id}/addresses
   - PUT/DELETE /clients/{id}/addresses/{addr_id}
   - GET/POST /clients/{id}/contacts
   - PUT/DELETE /clients/{id}/contacts/{contact_id}
6. Write tests

## Constraints
- Permissions: use existing Client.Read / Client.Edit permissions
- Follow existing patterns in the clients module
- SQLite compatible migrations (batch_alter_table for existing table changes)

## Quality Gate (MUST pass)
cd backend && ruff check --fix . && ruff format . && ruff check . && pytest -q
```

---

## Batch A3 — Case Field Expansion (NORMAL Type)

**Goal**: Add the most-needed SPEC fields for NORMAL case type to T_Case. Split into low-risk (dates/publication) and medium-risk (control flags/agent assignment) groups per GPT52's risk stratification.

### Scope

**Migration**: Add columns to `t_case`:

**Group 1 — Publication/Grant dates (low risk)**:
- `pub_date: Date, nullable`
- `pub_no: String(64), nullable`
- `grant_date: Date, nullable`
- `grant_no: String(64), nullable`
- `patent_no: String(64), nullable`
- `valid_until: Date, nullable`

**Group 2 — Specification details (low risk)**:
- `spec_pages: Integer, nullable`
- `claim_count: Integer, nullable`
- `has_exam_request: Boolean, nullable, server_default=text("0")`

**Group 3 — Agent assignment (medium risk)**:
- `primary_agent_id: String(36), FK -> t_user.id, nullable`
- `second_agent_id: String(36), FK -> t_user.id, nullable`
- `draftor_id: String(36), FK -> t_user.id, nullable`

**Group 4 — Control flags (medium risk)**:
- `is_fee_monitor: Boolean, nullable, server_default=text("0")`
- `fee_reduction: String(16), nullable` — (NONE|PARTIAL|FULL)
- `applicant_kind: String(16), nullable` — (INDIVIDUAL|ENTITY|UNIV|GOV)

**Schema updates**: Extend CaseCreate, CaseUpdateFull, CaseDetail, CaseListItem with new fields. Extend CaseUpdateLimited whitelist: add `spec_pages`, `claim_count`.

**Non-scope**: No PCT fields. No FromCountry/ToCountry (NORMAL only). No frontend changes.

### Files Modified

| File | Change |
|------|--------|
| `backend/alembic/versions/a3_case_field_expansion.py` | NEW migration |
| `backend/app/modules/cases/models.py` | Add 15 columns to Case |
| `backend/app/modules/cases/schemas.py` | Extend all case schemas |
| `backend/app/modules/cases/service.py` | Ensure new fields flow through CRUD |
| `backend/tests/test_case_fields.py` | NEW tests for new fields |

### Agent Team Prompt (A3)

```
You are implementing Batch A3 of the FPMS MVP1 Enhancement Plan.

## Context
FPMS backend: FastAPI + SQLAlchemy + SQLite.
T_Case currently has: id, case_no, case_type, patent_category, flow_dir, client_id,
title_cn, title_en, app_no, status, recv_date, filing_date.
SPEC 2.0 requires ~50 fields. We add the most important 15 for NORMAL case type only.

## Your Task
1. Create migration adding 15 columns to t_case (use batch_alter_table for SQLite):
   - pub_date: Date, nullable
   - pub_no: String(64), nullable
   - grant_date: Date, nullable
   - grant_no: String(64), nullable
   - patent_no: String(64), nullable
   - valid_until: Date, nullable
   - spec_pages: Integer, nullable
   - claim_count: Integer, nullable
   - has_exam_request: Boolean, nullable, server_default=text("0")
   - primary_agent_id: String(36), nullable (FK to t_user.id, SET NULL on delete)
   - second_agent_id: String(36), nullable (FK to t_user.id, SET NULL on delete)
   - draftor_id: String(36), nullable (FK to t_user.id, SET NULL on delete)
   - is_fee_monitor: Boolean, nullable, server_default=text("0")
   - fee_reduction: String(16), nullable
   - applicant_kind: String(16), nullable

2. Update backend/app/modules/cases/models.py — add mapped_columns

3. Update backend/app/modules/cases/schemas.py:
   - Add new fields to CaseCreate (all optional)
   - Add new fields to CaseUpdateFull
   - Add spec_pages, claim_count to CaseUpdateLimited whitelist
   - Add new fields to CaseDetail and CaseListItem

4. Update backend/app/modules/cases/service.py if needed (ensure new fields flow through)

5. Write tests verifying new fields persist through create→get cycle

## Constraints
- All new columns MUST be nullable (backward compatible)
- SQLite FK: use String(36) not ForeignKey for agent IDs if batch_alter_table doesn't support FK in SQLite; alternatively, skip FK constraint in migration and rely on app-level validation
- Do NOT add PCT, invalidation, or litigation fields
- Do NOT change existing field definitions

## Quality Gate (MUST pass)
cd backend && ruff check --fix . && ruff format . && ruff check . && pytest -q
```

---

## Batch A4 — SystemParam CRUD API

**Goal**: Expose the existing T_SystemParam table via API endpoints so admins can manage system parameters.

### Scope

1. **`backend/app/modules/system/service.py`**: Add functions:
   - `list_params(db) -> list[SystemParam]`
   - `get_param(db, key) -> SystemParam | None`
   - `upsert_param(db, key, value, value_type, description, is_secret, actor_id) -> SystemParam`

2. **`backend/app/modules/system/schemas.py`**: Add/update schemas:
   - `SystemParamOut(param_key, param_value, value_type, description, is_secret, updated_at)` — mask value when is_secret=True
   - `SystemParamUpsertIn(param_key, param_value, value_type?, description?, is_secret?)`

3. **`backend/app/modules/system/api.py`**: Add endpoints:
   - `GET /system/params` — list all params (perm: `System.Read`)
   - `PUT /system/params/{key}` — upsert param (perm: `System.Admin`)

4. **`backend/scripts/seed_dev.py`**: Seed default params:
   - `case_no_prefix` = "CN"
   - `default_currency` = "CNY"
   - `bill_no_prefix` = "INV"
   - `task_sheet_template_path` = "templates/task_sheet.docx"

**Non-scope**: No UI. No complex parameter validation engine.

### Agent Team Prompt (A4)

```
You are implementing Batch A4 of the FPMS MVP1 Enhancement Plan.

## Context
T_SystemParam already exists at backend/app/models/system_param.py with fields:
id (Integer PK), param_key, param_value, value_type, description, is_secret,
updated_by_user_id, updated_at, created_at.

The system module exists at backend/app/modules/system/ with api.py, schemas.py, service.py.
Currently only healthz endpoint exists.

## Your Task
1. Add service functions to backend/app/modules/system/service.py:
   - list_params(db) -> list[SystemParam]
   - get_param(db, key) -> SystemParam | None
   - upsert_param(db, key, value, ..., actor_id) -> SystemParam
     (upsert = insert if not exists, update if exists, matching on param_key)

2. Add/update schemas in backend/app/modules/system/schemas.py:
   - SystemParamOut — when is_secret=True, return param_value as "******"
   - SystemParamUpsertIn(param_key, param_value, value_type="string", description=None, is_secret=False)

3. Add endpoints to backend/app/modules/system/api.py:
   - GET /system/params (perm: System.Read) — returns list of SystemParamOut
   - PUT /system/params/{key} (perm: System.Admin) — upsert a parameter

4. Seed default params in backend/scripts/seed_dev.py

5. Register permissions System.Read, System.Admin in seed (bind to Admin role)

6. Write tests

## Constraints
- Import SystemParam from app.models.system_param
- Secret masking in SCHEMA layer (the Out schema), not in service
- Do NOT modify the SystemParam model
- Follow existing API patterns in system/api.py

## Quality Gate (MUST pass)
cd backend && ruff check --fix . && ruff format . && ruff check . && pytest -q
```

---

## Batch A5 — Advanced Case Search

**Goal**: Add filter parameters to `GET /api/v1/cases` so users can search by client, status, case_type, date ranges, and agent.

### Scope

1. **`backend/app/modules/cases/service.py`**: Extend `list_cases()` to accept and apply filters:
   - `client_id: str | None`
   - `status: str | None`
   - `case_type: str | None`
   - `patent_category: str | None`
   - `flow_dir: str | None`
   - `filing_date_from: date | None`
   - `filing_date_to: date | None`
   - `primary_agent_id: str | None`

2. **`backend/app/modules/cases/api.py`**: Add Query parameters to `GET /cases`.

3. **Tests**: Verify filtering works correctly.

**Non-scope**: No frontend filter panel. No full-text search.

### Agent Team Prompt (A5)

```
You are implementing Batch A5 of the FPMS MVP1 Enhancement Plan.

## Context
GET /api/v1/cases currently supports: q (keyword search), page, page_size, status.
Need to add: client_id, case_type, patent_category, flow_dir, filing_date_from,
filing_date_to, primary_agent_id as additional query filters.

## Your Task
1. Read current cases/service.py and cases/api.py to understand existing patterns
2. Extend the list_cases service function with new filter parameters
3. Add new Query params to the GET /cases endpoint
4. Write tests that create cases with different attributes and verify filters work

## Constraints
- All new filters are optional (Query(default=None))
- Date filters use >= and <= comparison
- Follow existing filter pattern in service.py
- Do NOT change the response schema

## Quality Gate (MUST pass)
cd backend && ruff check --fix . && ruff format . && ruff check . && pytest -q
```

---

## ═══ MVP1 SCOPE GATE ═══

After Batch A5, verify all 4 MVP1 success criteria pass:

```bash
# Run full baseline verification script from A0
# Criterion 2 should now work: Doc(IN, "OA") → auto-creates Task with correct due_date
```

If all pass, proceed to Phase B. If any fail, fix before continuing.

---

## Batch B1 — DocTemplate Enhancement + CRUD API

**Goal**: Add SPEC configuration fields to T_DocTemplate so it can drive downstream automation (status changes, task generation, fee draft generation). Provide CRUD API. Seed 3-5 common templates.

### Scope

1. **Migration**: Add columns to `t_doc_template`:
   - `status_effect: String(32), nullable` — case status to set when doc registered (e.g., "ACCEPTED", "OA1")
   - `status_restore: String(32), nullable` — status to restore on doc delete (rollback)
   - `deadline_template_code: String(64), nullable` — TaskTemplate.code to use for auto-task
   - `fee_draft_type: String(32), nullable` — FeeDraft.draft_type to auto-create (e.g., "GRANT_FEE")
   - `fee_item_list: Text, nullable` — JSON array of default fee items
   - `need_reply: Boolean, nullable, server_default=text("0")` — does this doc type require a reply?
   - `reply_to_template_code: String(64), nullable` — which template is the expected reply
   - `input_fields: Text, nullable` — JSON list of additional input field definitions

2. **`backend/app/modules/documents/schemas.py`**: Add DocTemplate CRUD schemas.

3. **`backend/app/modules/documents/service.py`**: Add DocTemplate CRUD functions.

4. **`backend/app/modules/documents/api.py`**: Add endpoints:
   - `GET /doc-templates` (perm: `DocTemplate.Read`)
   - `POST /doc-templates` (perm: `DocTemplate.Create`)
   - `PUT /doc-templates/{id}` (perm: `DocTemplate.Edit`)
   - `GET /doc-templates/{id}` (perm: `DocTemplate.Read`)

5. **Seed 5 templates** (actual template files not required — config only):
   - `OA_IN` — direction=IN, need_reply=True, deadline_template_code="OA_REPLY", status_effect="OA1"
   - `OA_OUT` — direction=OUT, reply_to_template_code="OA_IN"
   - `ACCEPTANCE_NOTICE` — direction=IN, status_effect="ACCEPTED"
   - `GRANT_NOTICE` — direction=IN, status_effect="GRANT_PENDING", fee_draft_type="GRANT_FEE"
   - `CLIENT_IN` — direction=IN, need_reply=False

**Non-scope**: No actual cascade execution logic (that's B2/B3). This batch only stores the configuration.

### Agent Team Prompt (B1)

```
You are implementing Batch B1 of the FPMS MVP1 Enhancement Plan.

## Context
T_DocTemplate exists at backend/app/modules/documents/models.py with:
id, code, name, direction, enabled.
Need to add SPEC 2.0 configuration fields that will drive automation in later batches.

## Your Task
1. Migration: add 8 columns to t_doc_template (all nullable, SQLite compatible)
2. Update models.py
3. Add schemas: DocTemplateCreateIn, DocTemplateUpdateIn, DocTemplateOut
4. Add service CRUD functions
5. Add API endpoints: GET/POST /doc-templates, GET/PUT /doc-templates/{id}
6. Seed 5 common doc templates in seed_dev.py
7. Register permissions: DocTemplate.Read, DocTemplate.Create, DocTemplate.Edit
8. Tests

## Key Design Decision
These fields are CONFIG ONLY in this batch. The actual automation logic
(status_effect → update case status, deadline_template_code → create task,
fee_draft_type → create fee draft) will be implemented in B2 and B3.

## Quality Gate (MUST pass)
cd backend && ruff check --fix . && ruff format . && ruff check . && pytest -q
```

---

## Batch B2 — Document Reply Chain + Auto Write-off

**Goal**: Add reply chain fields to T_Document. When an OUT document is registered as a reply to an IN document, auto-close the related task.

### Scope

1. **Migration**: Add columns to `t_document`:
   - `reply_to_id: String(36), FK -> t_document.id, nullable`
   - `need_reply: Boolean, nullable, server_default=text("0")`
   - `reply_date: Date, nullable`

2. **Service logic** in `create_document()`:
   - If document has `reply_to_id` and direction=OUT:
     - Find the original IN document
     - Find any OPEN task linked to that IN document (`Task.document_id = reply_to_id`)
     - Auto-close that task (status → DONE, write TaskLog with action="AUTO_WRITEOFF")
     - Set original document's `reply_date = new document's doc_date`

3. **DocTemplate cascade** (from B1 config):
   - When creating a document with a `doc_template_id`:
     - If template has `status_effect`, update `Case.status` to that value
     - If template has `need_reply=True`, set `Document.need_reply = True`
     - Log: create TaskLog with action="STATUS_CHANGE" for traceability

4. **Schema updates**: Add `reply_to_id`, `need_reply`, `reply_date` to DocumentCreateIn, DocumentUpdateIn, DocumentOut.

**Non-scope**: No fee draft auto-creation (that's B3).

### Agent Team Prompt (B2)

```
You are implementing Batch B2 of the FPMS MVP1 Enhancement Plan.

## Context
This is the core "OA lifecycle" automation:
1. Register OA incoming (direction=IN, template=OA_IN) → status_effect="OA1" → auto task
2. Register OA reply (direction=OUT, template=OA_OUT, reply_to_id=OA_doc_id) → auto close task

T_Document currently has no reply chain fields.
DocTemplate already has status_effect, deadline_template_code, need_reply from B1.
TaskGenerationService already auto-creates tasks from documents (fixed in A1).

## Your Task
1. Migration: add reply_to_id, need_reply, reply_date to t_document
2. Update Document model
3. Update schemas
4. Update create_document service:
   a. If doc_template_id provided and template.status_effect is set:
      - Update Case.status = template.status_effect
   b. If doc has reply_to_id and direction=OUT:
      - Find OPEN tasks where Task.document_id = reply_to_id
      - Close each: status=DONE, done_at=now, write TaskLog(action="AUTO_WRITEOFF")
      - Set original_doc.reply_date = new_doc.doc_date
5. Tests: Full OA lifecycle test:
   - Create case → register OA_IN doc → verify task auto-created → register OA_OUT reply → verify task auto-closed

## Key Business Rule
Auto write-off ONLY closes tasks linked to the specific replied-to document.
It does NOT close all tasks on the case.

## Quality Gate (MUST pass)
cd backend && ruff check --fix . && ruff format . && ruff check . && pytest -q
```

---

## Batch B3 — Document→FeeDraft Linking (Interface-First)

**Goal**: When a document is registered with a DocTemplate that has `fee_draft_type` configured, auto-create a FeeDraft linked to the case. This is interface-first — the actual fee items come from template config, but if template has no fee_item_list, just create an empty draft.

### Scope

1. **New service**: `backend/app/modules/documents/fee_linking_service.py`
   - `maybe_create_fee_draft(db, document, template) -> FeeDraft | None`
   - If template.fee_draft_type is set:
     - Create FeeDraft(case_id, client_id from case, draft_type=template.fee_draft_type, currency from case/system param)
     - If template.fee_item_list (JSON) is set, parse and create FeeItems
     - Return created draft
   - If template.fee_draft_type is None, return None (no-op)

2. **Wire into `create_document()`**: After document creation, if template has fee_draft_type, call `maybe_create_fee_draft()`.

3. **Response enhancement**: Return `X-Auto-Fee-Draft-Created: {draft_id}` header when draft auto-created.

**Non-scope**: No fee calculation engine. No fee rate lookup. Just creates draft structure.

### Agent Team Prompt (B3)

```
You are implementing Batch B3 of the FPMS MVP1 Enhancement Plan.

## Context
DocTemplate now has fee_draft_type and fee_item_list fields (from B1).
When a document is registered using a template like GRANT_NOTICE (fee_draft_type="GRANT_FEE"),
the system should auto-create a FeeDraft for the case.

## Your Task
1. Create backend/app/modules/documents/fee_linking_service.py:
   - maybe_create_fee_draft(db, document, template) -> FeeDraft | None
   - Only creates if template.fee_draft_type is set
   - Creates FeeDraft with: case_id, client_id (from Case), draft_type, currency="CNY", status="OPEN"
   - If template.fee_item_list is a JSON string, parse it and create FeeItem rows
   - fee_item_list format: [{"fee_code":"REG_FEE","fee_name":"登记费","fee_type":"GOV","amount":200}]
   - If fee_item_list is empty/null, create draft with no items (user fills manually)

2. Wire into documents/service.py create_document():
   - After document is created, if doc_template_id is set, load template
   - If template.fee_draft_type is set, call maybe_create_fee_draft()
   - This should NOT block document creation on fee draft failure — wrap in try/except, log warning

3. Add X-Auto-Fee-Draft-Created header to document create response when draft created

4. Tests: Register GRANT_NOTICE doc → verify FeeDraft auto-created

## Constraints
- Import FeeDraft, FeeItem from app.modules.fees.models
- fee_item_list is stored as JSON text in DocTemplate, parse with json.loads()
- If fee_item_list is malformed, log warning and skip — do not crash
- Do NOT modify fee module models or schemas

## Quality Gate (MUST pass)
cd backend && ruff check --fix . && ruff format . && ruff check . && pytest -q
```

---

## Batch B4 — FeeRate Dimensions + CalcMode Stub

**Goal**: Add multi-dimensional fields to T_FeeRate and a CalcMode stub so the fee rate table can eventually support automated fee calculation.

### Scope

1. **Migration**: Add columns to `t_fee_rate`:
   - `rate_group: String(32), nullable` — grouping (DOMESTIC, PCT, ANNUITY)
   - `country_code: String(10), nullable`
   - `case_type: String(32), nullable`
   - `patent_category: String(32), nullable`
   - `calc_mode: String(16), nullable, server_default=text("'FIXED'")` — FIXED|PER_CLAIM|PER_PAGE|TIER
   - `calc_params: Text, nullable` — JSON for complex calc rules
   - `allow_reduction: Boolean, nullable, server_default=text("0")`
   - `effective_from: Date, nullable`
   - `effective_to: Date, nullable`

2. **Update schemas and service** to expose new fields.

3. **Stub calc function**: `calculate_fee_amount(rate, case) -> Decimal` that currently only supports FIXED mode (returns `rate.default_amount`). Other modes return the same with a TODO log.

**Non-scope**: No actual PER_CLAIM/PER_PAGE/TIER calculation logic. No UI changes.

### Agent Team Prompt (B4)

```
You are implementing Batch B4 of the FPMS MVP1 Enhancement Plan.

## Context
T_FeeRate at backend/app/modules/fees/models.py currently has:
id, fee_code, fee_name, fee_type, currency, default_amount, enabled.
SPEC requires multi-dimensional rate lookup and calculation modes.

## Your Task
1. Migration: add 9 columns to t_fee_rate
2. Update FeeRate model
3. Update fee schemas to include new fields in response
4. Add stub function in fees/service.py:
   calculate_fee_amount(rate: FeeRate, case: Case | None = None) -> Decimal
   - FIXED: return rate.default_amount
   - Other modes: log warning, return rate.default_amount as fallback
5. Tests

## Constraints
- All new columns nullable (backward compatible)
- calc_params is JSON text, not parsed in this batch
- Do NOT implement actual PER_CLAIM/PER_PAGE logic — just the stub

## Quality Gate (MUST pass)
cd backend && ruff check --fix . && ruff format . && ruff check . && pytest -q
```

---

## Batch B5 — CaseReceipt Enrichment + Billing Polish

**Goal**: Add missing fields to T_CaseReceipt and polish billing module.

### Scope

1. **Migration**: Add columns to `t_case_receipt`:
   - `fee_code: String(64), nullable`
   - `year_no: Integer, nullable`
   - `is_arrears: Boolean, nullable, server_default=text("0")`
   - `invoice_no: String(64), nullable`
   - `is_commissionable: Boolean, nullable, server_default=text("0")`

2. **Update schemas and service** for CaseReceipt to include new fields.

3. **Offset reversal service**: Add `reverse_offset(db, offset_id, actor_id)` function that:
   - Sets `offset.is_reversed = True`, `offset.reversed_at = now()`
   - Restores bill balance: `bill.balance += offset.offset_amt`
   - Updates bill status if needed (SETTLED → PARTIALLY_SETTLED or UNSETTLED)

4. **API**: Add `POST /billing/offsets/{offset_id}/reverse` endpoint.

**Non-scope**: No prepayment management. No bad debt. No dunning.

### Agent Team Prompt (B5)

```
You are implementing Batch B5 of the FPMS MVP1 Enhancement Plan.

## Context
T_CaseReceipt exists with: id, case_id, fee_type, currency, receivable_amt, received_amt, last_receipt_date.
T_Offset exists with: is_reversed, reversed_at fields already in model.
Need to add financial detail fields to CaseReceipt and implement offset reversal.

## Your Task
1. Migration: add fee_code, year_no, is_arrears, invoice_no, is_commissionable to t_case_receipt
2. Update CaseReceipt model and schemas
3. Add reverse_offset() service function in billing/service.py:
   - Validate offset exists and is_reversed=False
   - Set is_reversed=True, reversed_at=datetime.utcnow()
   - Restore bill.balance += offset.offset_amt
   - If bill.balance == bill.amount: bill.status = "UNSETTLED"
   - Elif bill.balance > 0: bill.status = "PARTIALLY_SETTLED"
4. Add POST /billing/offsets/{offset_id}/reverse endpoint (perm: Billing.Edit)
5. Tests

## Quality Gate (MUST pass)
cd backend && ruff check --fix . && ruff format . && ruff check . && pytest -q
```

---

## Batch B6 — Search & Filter Enhancement

**Goal**: Add advanced search filters to document and task list endpoints. Add case_no/client_name joins to task and document list responses.

### Scope

1. **Document list** (`GET /documents`): already has direction, doc_template_id, case_id, date_from, date_to filters. Add:
   - `client_id` filter (join through Case.client_id)

2. **Task list** (`GET /tasks`): already has status, case_id, worker_id, supervisor_id, due_from, due_to. Add:
   - `client_id` filter (join through Case.client_id)
   - Include `client_name` in task list response (join Case → Client)

3. **Document list response**: Include `case_no` (already joined in task list, apply same pattern).

**Non-scope**: No full-text search. No Elasticsearch. No frontend filter panel.

### Agent Team Prompt (B6)

```
You are implementing Batch B6 of the FPMS MVP1 Enhancement Plan.

## Context
Task list already batch-resolves case_no. Document list does not include case_no.
Neither includes client_name in the response.
Need to add cross-entity joins for better search results.

## Your Task
1. Update documents/service.py list_documents():
   - Add client_id filter (join Document → Case → client_id)

2. Update documents/api.py GET /documents:
   - Add client_id Query parameter
   - Include case_no in each document item (batch-resolve like tasks do)

3. Update tasks/api.py GET /tasks:
   - Add client_id Query parameter (join Task → Case → client_id)
   - Include client_name in response items (batch-resolve Case → Client)

4. Tests

## Quality Gate (MUST pass)
cd backend && ruff check --fix . && ruff format . && ruff check . && pytest -q
```

---

## ═══ SPEC ALIGNMENT GATE ═══

After Batch B6, verify the complete chain works:

```bash
# Full E2E: Case → Doc(OA_IN) → auto-task → Doc(OA_OUT, reply) → auto-close-task
# Full E2E: Case → Doc(GRANT_NOTICE) → auto-fee-draft → bill → payment → offset → reverse
# Advanced search: cases by client_id + status + date range
# TaskTemplate CRUD + auto-generation
# SystemParam CRUD
# Client with addresses + contacts
```

---

## 3. Batch Execution Summary

| Batch | Name | Priority | Depends On | Est. Files | Key Deliverable |
|-------|------|----------|------------|------------|-----------------|
| **A0** | Baseline Verify | P0 | — | 0 | Verified working state |
| **A1** | TaskTemplate + TaskLog API | P0 | A0 | 9 | Doc→Task auto-link works |
| **A2** | Client Address/Contact | P0 | A1 | 6 | Multi-address/contact CRUD |
| **A3** | Case Field Expansion | P1 | A2 | 5 | 15 new SPEC fields on Case |
| **A4** | SystemParam API | P1 | A3 | 5 | Admin param management |
| **A5** | Advanced Case Search | P1 | A4 | 3 | 8 new filter params |
| | **═══ MVP1 GATE ═══** | | A5 | | All 4 success criteria pass |
| **B1** | DocTemplate Enhancement | P1 | A5 | 7 | Config-driven doc automation |
| **B2** | Reply Chain + Auto Write-off | P1 | B1 | 5 | OA lifecycle complete |
| **B3** | Doc→FeeDraft Linking | P2 | B1 | 4 | Fee draft auto-generation |
| **B4** | FeeRate Dimensions | P2 | B3 | 4 | Multi-dim rate lookup stub |
| **B5** | CaseReceipt + Offset Reversal | P2 | B4 | 4 | Financial detail + reversal |
| **B6** | Search & Filter Polish | P2 | B5 | 3 | Cross-entity search |
| | **═══ SPEC GATE ═══** | | B6 | | Full E2E chain verified |

**Total batches**: 12 (+ 2 gates)
**Total estimated files changed**: ~55

---

## 4. How to Execute: Agent Team Launch Protocol

### For each batch, use this pattern:

```bash
# Launch command (adjust batch ID and prompt)
# Use Claude Code agent team with the prompt from the corresponding batch section above
```

### Recommended team size per batch:

| Batch | Recommended Approach | Why |
|-------|---------------------|-----|
| A0 | Single agent, manual | Verification only, no code changes |
| A1 | 2-agent team (Backend + Test) | Most complex batch; migration + 4 endpoints + service fix |
| A2 | Single agent | Well-scoped sub-resource CRUD |
| A3 | Single agent | Migration + schema changes, straightforward |
| A4 | Single agent | Small scope, 2 endpoints |
| A5 | Single agent | Filter additions only |
| B1 | Single agent | Migration + CRUD, similar to A1 |
| B2 | 2-agent team (Service + Test) | Complex auto-write-off logic needs thorough testing |
| B3 | Single agent | New service file + wiring |
| B4 | Single agent | Migration + stub function |
| B5 | Single agent | Migration + reversal logic |
| B6 | Single agent | Query enhancement |

### Post-batch checklist:

```
□ Quality gate passes (ruff + pytest + alembic + seed + healthz)
□ New endpoints documented in API docstrings
□ Batch diff reviewed (no unintended changes)
□ Git commit with message: "feat(ENH): Batch {ID} — {name}"
```

---

## 5. Risk Mitigation

| Risk | Mitigation |
|------|-----------|
| Migration breaks existing data | All new columns nullable; test with existing .db before and after |
| TaskGenerationService RuntimeError | A1 fixes this by adding add_days field; tested with seeded template |
| Auto write-off closes wrong tasks | B2 uses document_id linkage (not case-wide); tested with specific scenario |
| Fee draft auto-creation fails | B3 wraps in try/except; does not block document creation |
| SQLite FK limitations in batch_alter_table | Use app-level validation for new FKs in migration if SQLite batch mode doesn't support them |
| Scope creep during implementation | Frozen features list enforced; each batch prompt has explicit non-scope |
