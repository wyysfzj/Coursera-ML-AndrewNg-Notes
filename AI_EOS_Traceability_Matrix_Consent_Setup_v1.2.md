# AI‑EOS Traceability Matrix  
## Consent Setup Chapter — v1.2

**Goal**: Provide a complete, auditable mapping from:
- **Constitutional invariants** (INV‑xx)  
to
- **Atomic Tasks** (OB‑AT‑xx)  
to
- **Evidence** (Evidence Bundle fields)  
to
- **Gates** (static/behavior/diff)  
to
- **Implementation locations** (repo paths)

This matrix is what allows a platform/governance team to answer:  
> “Which task introduced the control, what evidence proves it, and what gate enforces it?”

---

## 1. Invariant → Task → Gate mapping (primary)

| Invariant | Non‑negotiable rule | Implemented by Atomic Tasks | Required Evidence (bundle) | Gates (deterministic) | Primary code locations |
|---|---|---|---|---|---|
| **INV‑01** Request integrity | Missing/invalid signature => 401; never trust headers alone | **OB‑AT‑03**, OB‑AT‑05, OB‑AT‑06, **OB‑AT‑09** | `outputs/*_conformance.log`, evidence logs `EVIDENCE_SIGNATURE_VERIFIED`, `results.jsonl` | **B‑INV‑01** (behavior), S‑VET/S‑FMT (static), D‑SCOPE (diff) | `internal/security/*`, `internal/atomic/consent/signature.go`, `tests/invariants_test.go` |
| **INV‑02** Idempotency non‑malleable | Idempotency‑Key binds request hash; mismatch => 409 | **OB‑AT‑04**, OB‑AT‑05, **OB‑AT‑09** | `outputs/*_conformance.log`, `results.jsonl`, `git/diff.patch` | **B‑INV‑02** (behavior), D‑SCOPE | `internal/atomic/consent/idempotency.go`, `internal/atomic/consent/store.go`, `tests/invariants_test.go` |
| **INV‑03** Client binding on reads | Client mismatch must not leak existence; default 404 | **OB‑AT‑06**, **OB‑AT‑09** | `outputs/*_conformance.log`, `results.jsonl` | B‑INV‑03 (behavior; v1.1+ expands multi‑client), D‑SCOPE | `internal/atomic/consent/service.go`, `tests/invariants_test.go` |
| **INV‑04** PII minimization | No customer_id in responses; no raw body logging | **OB‑AT‑07**, OB‑AT‑05, **OB‑AT‑09** | `outputs/*_conformance.log`, `outputs/*_pii_scan.log`, `results.jsonl` | **S‑PII‑SCAN** (static), B‑INV‑04 (behavior) | `internal/httpapi/*`, `internal/atomic/consent/*`, `internal/audit/*` |
| **INV‑05** Expiry sanity | expires_at must be RFC3339 and sufficiently in future | **OB‑AT‑02**, OB‑AT‑05, OB‑AT‑09 (v1.1+) | `outputs/*_conformance.log` (when enabled) | B‑INV‑05 (behavior; v1.2 required) | `internal/consent/model.go` (`ValidateExpiry`) |
| **INV‑06** Challenge quality | challenge must be cryptographically random | OB‑AT‑05, OB‑AT‑09 (v1.1+) | optional uniqueness evidence | B‑INV‑06 (behavior; v1.2 required) | `internal/atomic/consent/util.go` |

---

## 2. Task → Evidence Bundle obligations (secondary)

| Atomic Task | Primary objective | Mandatory bundle fields | Typical gates to expect |
|---|---|---|---|
| OB‑AT‑03 | signature verification | commands.jsonl, outputs/*_vet.log, outputs/*_test.log, results.jsonl | S‑FMT, S‑VET, B‑INV‑01, D‑SCOPE |
| OB‑AT‑04 | idempotency hash binding | commands.jsonl, git/diff.patch, outputs/*_test.log, results.jsonl | B‑INV‑02, D‑SCOPE |
| OB‑AT‑07 | audit/redaction | outputs/*_pii_scan.log, results.jsonl | S‑PII‑SCAN, B‑INV‑04 |

---

## 3. Non‑Atomic track (training harness mapping)

| Non‑Atomic anti‑pattern | How it manifests | What gate catches it |
|---|---|---|
| Accept missing signatures | POST succeeds without X‑Signature | B‑INV‑01 |
| Idempotency malleability | same key + different payload still returns success | B‑INV‑02 |
| Raw request logging | logs include JSON request/customer_id | S‑PII‑SCAN (atomic codebase), plus policy review; Non‑Atomic kept for training only |

---

## 4. Audit question examples (how to use the matrix)

1) “Where is INV‑02 enforced?”  
→ OB‑AT‑04 + B‑INV‑02; verify via `tests/invariants_test.go` and Evidence Bundle results.

2) “Which task introduced PII controls?”  
→ OB‑AT‑07, and gates S‑PII‑SCAN + B‑INV‑04.

3) “Can we ship without passing B‑INV‑01?”  
→ Constitution says **no**; exception process forbids disabling INV‑01 for production.

