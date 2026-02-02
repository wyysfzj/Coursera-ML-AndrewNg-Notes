# Atomic Task Writing Specification v1.0

## Purpose

This document defines the **official writing standard** for Atomic Tasks used in the FPMS MVP project.

Atomic Tasks are designed to:
- Be deterministic and reproducible
- Eliminate ambiguity and runtime decision-making
- Enable consistent results when executed by different developers or AI tools (e.g. Copilot/Codex)
- Align implementation strictly with design documents

This specification applies to:
- `tasks/backend/BE-xx-yy.md`
- `tasks/frontend/FE-xx-yy.md`

It does NOT apply to:
- README files
- Task index / summary files
- Design documents

---

## Core Principles (MUST)

All Atomic Tasks MUST follow these rules:

1. **One task = one file = one responsibility**
2. **Task scope MUST be deterministic**
3. **Design documents are the single source of truth**
4. **All fields / behavior MUST be explicitly enumerated**
5. **If a task conflicts with design docs, design docs win**

Any task violating these principles MUST be rejected in review.

---

## Prohibited Language (STRICT)

Atomic Tasks MUST NOT contain any of the following:

- “if you want…”
- “optionally…”
- “you may choose…”
- “ask me to add…”
- “audit fields” (without explicit enumeration)
- “etc.” / “…” / “and so on”

Any appearance of the above indicates a broken task.

---

## Required File Structure

Every Atomic Task file MUST follow this exact structure and order:


<TaskID> — <Short Title>
Design references
Target
Scope decision (MVP1 – FIXED)
Models / APIs / Components to implement (EXACT)
Non-scope (explicitly excluded)
Prompt

The order MUST NOT be changed.

---

## Section-by-Section Rules

### 1. Title


BE-02-01 — Case core model (T_Case)

Rules:
- Must include TaskID
- Must name the concrete responsibility (model / api / ui)
- Avoid vague words such as “basic”, “initial”, “some”

---

### 2. Design references (MANDATORY)


Design references

backend/app/modules/cases/docs/case_01_db.md

backend/app/modules/cases/docs/case_03_rules.md


Rules:
- Must list real file paths
- At least one reference is required
- These documents are the **only** source of truth

---

### 3. Target (Atomic Anchor)


Target

File: backend/app/modules/cases/models.py

Atomic rule: modify/create ONLY this file.


Rules:
- Exactly ONE file
- No “and related files”
- No conditional language

---

### 4. Scope decision (MVP1 – FIXED)


Scope decision (MVP1 – FIXED)

Scope is FIXED for MVP1.

No additional fields, tables, or logic are allowed.

All decisions in this section are FINAL.


Rules:
- MUST contain the word **FIXED**
- MUST explicitly deny expansion
- This section exists to stop Copilot from asking questions

---

### 5. Models / APIs / Components to implement (EXACT)

This is the most important section.

#### Backend example (Models)


Models to implement (EXACT)
T_Case

Fields:

id

case_no

case_type

patent_category

flow_dir

client_id

title_cn

title_en

status

recv_date

filing_date

created_at

updated_at


Rules:
- Fields MUST be fully enumerated
- No placeholders like “audit fields”
- Enums / nullable rules MUST be stated explicitly

#### Frontend example (Components)


Components to implement (EXACT)
CaseList.vue

Render columns: case_no, title_cn, status

Data source: GET /api/v1/cases

UI skeleton only


---

### 6. Non-scope (explicitly excluded)


Non-scope (explicitly excluded)

No APIs

No services

No validation logic

No database migrations


Rules:
- MUST exist
- Minimum 3 items
- Used to prevent accidental over-implementation

---

### 7. Prompt (Copilot Instruction)


Prompt

Implement the above EXACTLY.

Requirements:

Follow design references as the single source of truth.

Implement ONLY what is listed in this task.

Do NOT add optional features.

Do NOT ask clarification questions.


Rules:
- MUST include “EXACTLY”
- MUST forbid clarification questions
- MUST forbid optional behavior

---

## Review Checklist (for humans & AI)

Before approving or executing a task, verify:

- [ ] No conditional language exists
- [ ] All fields/components are explicitly listed
- [ ] Non-scope section is present
- [ ] Design references exist and are relevant
- [ ] Exactly one target file is defined

If any check fails, the task MUST be fixed before execution.

---

## Versioning

- **v1.0** — Initial formal specification (MVP1)

Future changes MUST be versioned and reviewed.

---

## Final Note

Atomic Tasks are **instructions, not conversations**.

If a task allows multiple interpretations, it is not atomic and MUST be rewritten.