# AGENTS — FPMS MVP1 (Authoritative)

This file defines the authoritative execution rules for Codex/Agent working on this repository.
Agent MUST follow these rules without exception.

---

## 1) Atomic Task Discipline

- The atomic unit remains EXACTLY ONE task file path.
- One atomic task = one endpoint OR one service OR one doc change.
- Atomicity is enforced at the AGENT level, not at the whole-session level.
- In a multi-agent run, the lead/main thread MAY coordinate multiple atomic task files in one session, but each spawned agent MUST own exactly one explicit task file path.
- No single agent may implement more than one task file in the same execution.
- Do NOT implement multiple endpoints inside one atomic task unless the task file explicitly defines them as a single unit.
- Do NOT proactively refactor unrelated code.
- When asked “which task to implement”, request/confirm either:
  - one exact task file path, OR
  - an explicit batch manifest listing one exact task file path per agent.
- If no explicit batch manifest exists, default to ONE task file only.
- Two agents MUST NOT concurrently edit the same task file or the same shared ownership file.
- Shared ownership files (for example: router wiring, shared schemas, permission registries, common exports) require serialized ownership.
- An atomic task file MUST define EXACTLY ONE closure slice, not one module cluster.
- Every atomic task file MUST explicitly state:
  - exact closure slice
  - explicit non-closure statement
  - remaining follow-up task ids, or `None`
- If a task file cannot clearly answer “what exact behavior does this task close?”, it is NOT ready for implementation.
- A task file MUST NOT use broad acceptance wording such as:
  - “close the remaining module”
  - “finish the whole chain”
  - “complete backend/frontend parity”
  - “close the remaining feasible scope”
- Preferred atomic examples:
  - one endpoint behavior
  - one service rule
  - one page capability
  - one query / visibility slice
  - one final QA close audit
- Disallowed atomic examples:
  - one whole module cluster
  - one mixed backend+frontend mega task
  - one task that combines defaults + linkage + queries + dashboard

---

## 2) Phase Constraints

### Phase 3 (Domain APIs)
- No database schema changes.
- No Alembic migration edits (except Phase 0-EXT compatibility fixes explicitly requested).
- Use existing ORM models only.
- Implement endpoints only in module-level `api.py`.
- Preserve existing response envelope conventions.

### Phase 3.1 (API Extensions — `tasks/backend/apis_ext/`)
- Same constraints as Phase 3.
- Only implement tasks under `tasks/backend/apis_ext/`.

### Phase 3.5 (Business Logic — `tasks/backend/business_logic/`)
- Service-layer logic only.
- No schema changes.
- Allowed:
  - docxtpl rendering
  - context builders
  - Document → Task auto-generation
- API wiring may be updated ONLY where explicitly required by the task.

---

## 3) Router Wiring (Module-level, One-time)

- Router wiring is module-level and one-time.
- Do NOT rewire routers when adding endpoints within an already-wired module.
- Only add `include_router(...)` when entering a module for the FIRST time.
- Router wiring changes are limited to `backend/app/api/router.py` unless a task explicitly requires otherwise.

---

## 4) Authorization & Permission Enforcement

- Permission enforcement is mandatory for all protected endpoints.
- NEVER use `Depends(require_perm(...))` inside decorator `dependencies=[...]`.
- ALWAYS inject permission as a function parameter:

```python
_perm: None = Depends(require_perm("Title.Action"))

- Permission codes MUST follow Title.Action naming.

———

## 5) FastAPI Status Code Semantics (MANDATORY)

### 204 No Content

- MUST NOT return a response body.
- MUST NOT define response_model for 204 routes.
- Handler must end with:
    - return None OR
    - return Response(status_code=204)

### 201 Created

- MUST return created resource or its identifier (unless task says otherwise).

### GET

- MUST NOT require a request body.

### Error semantics (use consistently)

- 400: business validation failure (explicit message)
- 401: unauthenticated
- 403: permission denied
- 404: resource not found
- 409: conflict / missing configuration
- 422: validation error (FastAPI)

Before finishing any task, self-check:

- Is the response body compatible with declared status_code?

———

## 6) Ruff / Lint Discipline

Code MUST pass task-scoped lint/format checks by default.

### Default task-level verification

- ruff check --fix <task-allowlist-files>
- ruff format <task-allowlist-files>
- ruff check <task-allowlist-files>

### Full-repo verification

- ruff check --fix .
- ruff format .
- ruff check .

Full-repo verification is allowed ONLY when one of the following is true:

- final batch close
- explicit user request
- explicit task/runbook requirement

Imports must remain minimal and ordered; remove unused imports.

Rules:

- Do NOT run repo-wide Ruff write operations by default for a single atomic task.
- Do NOT modify files outside the task allowlist only because repo-wide Ruff touched them.
- In multi-agent execution, repo-wide Ruff write operations require serialized main-thread ownership.
- If a task is atomic and file-scoped, lint/format should be file-scoped by default.

———

## 7) Response Envelope Discipline

- Do NOT invent new response envelopes.
- Follow existing module conventions.

———

## 8) SQLite PoC Compatibility (MVP1 REQUIRED)

MVP1 PoC uses SQLite. Code and migrations MUST remain SQLite-compatible.

### 8.1 Timestamp defaults (REQUIRED)

- Do NOT use now() in migrations or server_default.
- Use server_default=sa.text("CURRENT_TIMESTAMP") for created_at/updated_at.

### 8.2 Autoincrement primary keys (REQUIRED)

- SQLite autoincrement works only with INTEGER PRIMARY KEY.
- For PoC, prefer Integer PK everywhere.
- Do NOT use BigInteger PK expecting autoincrement.
- Keep PK/FK types aligned (e.g., do NOT point String PKs from BigInteger FKs).
- If UUID PK is used, UUID MUST be generated in application code (uuid4) and stored as TEXT.

### 8.3 Foreign keys must be enabled (REQUIRED)

- SQLite foreign keys are not guaranteed unless enabled.
- Engine creation MUST set PRAGMA foreign_keys=ON on every connection (SQLAlchemy connect event).

### 8.4 Dialect-specific SQL is prohibited in PoC

Do NOT introduce PG-only functions/types:

- uuid_generate_v4(), gen_random_uuid()
- ILIKE
- date_trunc(...), timezone(...), interval '...'
- JSONB, ARRAY, CITEXT
  If needed, implement SQLite-safe alternatives (lower()+LIKE; app-side date math; store JSON as TEXT).

### 8.5 Do not rely on RETURNING for correctness

- Do not assume RETURNING support; use session.flush() to obtain PK values after insert.

### 8.6 Concurrency note (PoC)

- SQLite uses file locks. Keep write transactions short.
- If “database is locked” occurs, prefer reducing concurrency; WAL/retry may be considered later.

———

## 9) Forward-only Migrations Policy

Some migrations intentionally do NOT support downgrade.

- Do NOT attempt to rely on alembic downgrade base in dev workflows.
- For clean rebuild on SQLite dev:
    1. Delete the SQLite db file (and -wal/-shm if present)
    2. Run alembic upgrade head

———

## 10) Seeding (MVP1 Required)

- Seeding MUST be idempotent and bootstrap-safe.
- Avoid deadlocks: seed must be runnable when no users OR no Admin role exists.
- After any SQLite DB rebuild, seed MUST be re-run and token must be refreshed (log in again).

———

## 11) Required Agent Output (End of each execution)

Every agent MUST explicitly state:

- Which task/runbook was executed
- Which role executed it
- Which file(s) were modified
- Verification commands + expected status codes
- Evidence path under artifacts/<TASK-ID>/**
- Final per-task status: PASS / FAIL / BLOCKED
- Exact closure slice completed
- Explicit non-closure boundary respected

If the main thread coordinates multiple agents in one session, it MUST additionally state:

- Agent-to-task mapping
- Execution wave / batch order
- Any serialized shared-file decisions
- Per-task completion summary

Status policy:

- PASS means the task is claimed complete and has required evidence
- FAIL means the task was attempted but did not meet completion criteria
- BLOCKED means execution could not proceed or complete because of a dependency, scope issue, missing contract, or policy constraint

An agent MUST NOT mark a task PASS unless:

- required verification has run
- required evidence exists
- scope compliance has been checked
- the exact closure slice has been completed
- no second closure slice was silently absorbed into the same task

———

## 12) Evidence & Gates (EOS Bootstrap — MVP1 Enhancement)

To make AI-first execution auditable and deterministic, every task MAY create/update evidence under:

- artifacts/<TASK-ID>/**

This is non-product output and is explicitly allowed for all tasks, even when a task’s code scope is restricted to a single source file.

Rules:

- Do NOT store secrets/PII in artifacts logs.
- Artifacts should be generated by wrapper scripts (see scripts/evidence_run.sh).
- Evidence is OPTIONAL for exploratory, in-progress, or blocked work.
- Evidence is REQUIRED for tasks that claim completion or PASS:
    - artifacts/<TASK-ID>/results.jsonl (commands + rc)
    - artifacts/<TASK-ID>/summary.md (human-readable evidence summary)
    - artifacts/<TASK-ID>/git/diff.patch (scoped diff)
    - artifacts/<TASK-ID>/baseline_allowlist.diff when the task started from a dirty worktree
    - artifacts/<TASK-ID>/baseline_external_files.txt when the task started from a dirty worktree

Completion policy:

- An agent MUST NOT mark a task PASS without the required evidence artifacts.
- FAIL and BLOCKED may be reported without complete evidence, but any available evidence should still be attached.
- Reviewer and main thread MUST treat missing required evidence as non-completion.
- If dirty baseline artifacts are required but missing, the task MUST NOT be treated as cleanly accepted.

Gates:

- Task Gate: ./scripts/task_validate.sh <TASK-ID>
- Release Gate: ./scripts/release_gate.sh

———

## 13) Codex Multi-Agent Atomic Execution (MVP1 Execution)

This repository supports multi-agent execution with strict atomic-task discipline.

Team behavior is enforced by this file + task files + evidence gates.

### 13.1 Role Mapping (Supported)

| Business Role | Codex Executor | Responsibility |
|---|---|---|
| Team Lead / Project Manager | Main thread / default | Plan, assign task IDs, group tasks into safe waves, coordinate dependencies, final acceptance |
| Architect / Designer | explorer agent | Spec/contract analysis, module boundary checks, design decisions, impact analysis |
| Backend Developer | worker agent | Implement exactly one backend atomic task file per agent |
| Frontend Developer | worker agent | Implement exactly one frontend atomic task file per agent |
| Tester / Progress Monitor | monitor agent | Wait/poll long-running work, run task gates, collect evidence, report pass/fail/blockers |
| Reviewer | explorer agent or main thread | Independent scope review, acceptance validation, risk notes |

Notes:

- Built-in roles may be used directly (default, explorer, worker, monitor).
- One session MAY contain multiple atomic tasks only when each task has an explicit task file path and is assigned to a distinct agent or serialized wave.
- Acceptance remains per-task, not per-session.

### 13.2 Hard Constraints (Iron Rules + Quality Gate)

- Preserve all rules in Sections 1-12; this section is additive, not a replacement.
- Each spawned agent MUST target exactly one atomic task file path.
- Lead MAY coordinate multiple task file paths in one run only through an explicit batch manifest.
- No single agent may own more than one task file concurrently.
- No two agents may concurrently edit the same ownership file or shared support file.
- Each execution task file MUST define one exact closure slice only.
- Each execution task file MUST include an explicit non-closure statement.
- If a task file still reads like a cluster plan, it must be rewritten before execution.
- Shared ownership files that require serialization include:
    - backend/app/api/router.py
    - shared schema files
    - permission registry / permission constants
    - common exports / index files
    - any file explicitly listed in more than one task allowlist
- Frontend shared ownership files that require serialization include:
    - frontend/src/api/*.ts
    - frontend/src/api/*.types.ts
    - frontend router wiring / route registry files
    - shared stores
    - shared constants / display-text registries
    - common module exports / index files
- If two tasks require the same shared file, split out a dedicated follow-up task OR serialize them into different waves.
- Backend/Frontend dev agents MUST NOT modify each other's ownership files unless task explicitly allows.
- Reviewer cannot mark a task complete unless required gates pass and evidence exists.
- Reviewer cannot mark an item or batch `covered` merely because one representative slice passed; close decisions must be based on item-to-slice evidence.

### 13.3 Coordination Protocol (Wave-based, Multi-Agent Safe)

1. Lead defines an explicit batch manifest:

- one row = one task file path
- owner role
- allowed files
- required verification
- dependency notes
- exact closure slice
- explicit non-closure statement
- remaining follow-up task ids
- done definition

Plan-driven execution rule:

- If the user initiates work from a plan/batch document instead of explicit task file paths, the lead MUST first convert that plan into an explicit batch manifest.
- No real multi-agent implementation may begin until each execution agent has:
    - one exact task file path
    - one allowlist
    - one verification set
    - one exact closure slice
- Without such a manifest, execution MUST fall back to single-agent or planning-only mode.

2. Architect (explorer) freezes API contract / acceptance checklist for each task or for the batch if the contract is shared and identical.
3. Lead groups only NON-CONFLICTING tasks into the same execution wave.
4. Spawn one execution agent per task in the current wave:

- backend task → worker
- frontend task → worker
- validation / long-wait / evidence tracking → monitor

5. Monitor tracks completion and records evidence per task under:

- artifacts/<TASK-ID>/results.jsonl
- artifacts/<TASK-ID>/summary.md
- artifacts/<TASK-ID>/git/diff.patch
- artifacts/<TASK-ID>/baseline_allowlist.diff when required
- artifacts/<TASK-ID>/baseline_external_files.txt when required

6. Reviewer validates each completed task independently:

- scope compliance
- gate pass
- no regression
- no cross-task contamination
- exact closure slice completed
- explicit non-closure boundary respected

7. Lead closes tasks individually.

- A session may complete multiple tasks.
- Each task must independently pass before being marked done.

Mandatory handoff artifacts per task when claiming PASS:

- artifacts/<TASK-ID>/results.jsonl
- artifacts/<TASK-ID>/summary.md
- artifacts/<TASK-ID>/git/diff.patch
- dirty baseline artifacts when required

8. Final QA close audit for any batch MUST include an item-to-slice ledger.

The ledger MUST map each in-scope `Partially Implemented` item to:

- required slices
- implemented task ids
- evidence
- residual gap
- close decision

A batch MUST NOT be declared `complete` unless:

- all implementation tasks are `PASS`
- all required artifacts exist
- all task gates pass
- every in-scope item is `covered` in the ledger
- no residual gap remains inside the approved batch interpretation

9. Worker takeover and stall handling MUST preserve slice boundaries.

Before labeling a worker `idle` or `stalled`, lead/monitor MUST check:

- recent allowlist diff growth
- artifact timestamp changes
- running verification activity
- whether only evidence or gate closure is missing

Allowed takeover types:

- evidence-only takeover
- verification-only takeover
- limited slice-completion takeover, but only when the remaining work is still inside the SAME exact closure slice

Forbidden takeover types:

- silently completing a second slice inside the same task
- stretching a broad acceptance statement to fit a partial result
- using takeover to cross into another batch
- using takeover to avoid splitting a needed follow-up task

If the remaining work belongs to another slice, the correct action is to create a new follow-up task, not to finish it in place.

### 13.4 Required Verification by Role

- Backend task:
    - task-level Ruff on allowlist files
    - task-defined targeted tests
    - pytest -q only for:
        - final batch close
        - explicit user request
        - explicit task/runbook requirement
- Frontend task:
    - targeted lint/type/build checks when supported by the repo tooling
    - repo-wide npm run lint / npm run typecheck / npm run build only for:
        - final batch close
        - explicit user request
        - explicit task/runbook requirement
- Tester / Monitor:
    - ./scripts/task_validate.sh <TASK-ID>
    - targeted regression checks
    - release gate only when explicitly requested for batch close
- Reviewer:
    - Verify status-code semantics, permission enforcement, envelope consistency,
      SQLite compatibility, task allowlist compliance, and no cross-task/shared-file regression.

SQLite test concurrency rule:

- Agents MUST NOT run concurrent test jobs that write to the same SQLite database.
- If a test suite is known or likely to perform writes, it must run in serialized ownership.
- When “database is locked” risk exists, prefer targeted tests and serialized execution over parallel full-suite execution.

———

## 14) Frontend UI Language Iron Rule (MANDATORY)

- All user-facing UI text MUST be Simplified Chinese.
- This rule applies to:
    - page titles
    - menu labels
    - buttons
    - form labels/placeholders
    - validation/error/toast messages
    - empty states and helper texts
    - dialog titles/content/actions
- English is allowed only for non-UI technical values:
    - IDs, enum/code values, API field names, protocol terms, file paths, logs.
- If an existing page has mixed language, FE tasks touching that page MUST normalize visible text to Simplified Chinese within task scope.
- Reviewer MUST reject FE tasks that introduce or retain user-visible non-Chinese text without explicit task-level exception.
