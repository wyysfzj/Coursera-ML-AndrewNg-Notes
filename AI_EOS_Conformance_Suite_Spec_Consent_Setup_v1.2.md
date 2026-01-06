# AI‑EOS Conformance Suite Specification  
## Consent Setup Chapter — v1.2

**Status**: Normative (internal standard)  
**Audience**: Platform governance, security, risk/compliance, senior engineers  
**Purpose**: Define a deterministic **Conformance Suite** that proves the Consent Setup service complies with the **AI‑EOS Backend Constitution: Consent Setup Chapter**.

---

## 1. Definitions

### 1.1 Conformance Suite
A **Conformance Suite** is a bounded set of **deterministic checks** (gates) that collectively establish whether a service implementation satisfies required **constitutional invariants**.

### 1.2 Conformance Report
A **Conformance Report** is the machine‑readable output of the suite. In AI‑EOS, it is carried via:
- `artifacts/<TASK_ID>/outputs/*_conformance.log`
- `artifacts/<TASK_ID>/results.jsonl` (the only gate input surface)

### 1.3 Scope
This suite applies to the **production‑enabled implementation** of Consent Setup.
In the teaching repo, that is the **Atomic** path (`internal/atomic/...`) and its request/response surface (`internal/httpapi/...`).
The **Non‑Atomic** path exists for training and MUST be excluded from production conformance.

---

## 2. Design principles

1. **Deterministic**: same input yields same pass/fail.
2. **Invariant‑bound**: every check maps to an invariant (INV‑xx).
3. **Falsifiable**: failures produce evidence that can be reproduced.
4. **Non‑bypassable**: conformance failures block merge/release unless exception process is invoked.
5. **Machine‑consumable**: gates consume only `results.jsonl`.

---

## 3. Check taxonomy and severity

### 3.1 Taxonomy
- **Static checks**: structural properties (formatting, vetting, forbidden patterns).
- **Behavior checks**: runtime properties asserting invariants via tests.
- **Diff checks**: scope control (only allowed files changed per Atomic Task).

### 3.2 Severity levels
- **BLOCKER**: must pass for any merge to protected branches.
- **CRITICAL**: must pass for release; typically also for merge.
- **MAJOR**: should pass; exception requires explicit compensating controls.
- **MINOR**: informational; does not block by default.

---

## 4. Required checks (v1.2)

> Each check has: ID, type, invariant mapping, execution, pass/fail, evidence, severity.

### 4.1 Static checks

#### S‑FMT — gofmt clean
- **Type**: static
- **Maps**: Governance baseline
- **Execute**: `test -z "$(gofmt -l .)"`
- **Pass**: empty output
- **Evidence**: `outputs/*_format.log`, `results.jsonl` (`format`)
- **Severity**: BLOCKER

#### S‑VET — go vet clean
- **Type**: static
- **Execute**: `go vet ./...`
- **Pass**: exit code 0
- **Evidence**: `outputs/*_vet.log`, `results.jsonl` (`vet`)
- **Severity**: BLOCKER

#### S‑PII‑SCAN — forbid PII logging patterns (Atomic path only)
- **Type**: static
- **Maps**: INV‑04 (PII minimization)
- **Execute**: `bash tools/eos/gates/pii_scan.sh`
- **Pass**: no forbidden patterns found
- **Evidence**: `outputs/*_pii_scan.log`, `results.jsonl` (`pii_scan`)
- **Severity**: CRITICAL

> Note: This is a **heuristic static check** by design.  
> It is not a substitute for a full data classification program, but it is a deterministic gate against common AI‑introduced failures.

---

### 4.2 Behavior checks (invariant gates)

#### B‑INV‑01 — Signature required
- **Type**: behavior
- **Maps**: INV‑01
- **Execute**: `EOS_MODE=atomic go test ./tests -run TestInvariants -v`
- **Pass**: unsigned POST/GET return 401
- **Evidence**: `outputs/*_conformance.log` (test output), `results.jsonl` (`conformance`)
- **Severity**: BLOCKER

#### B‑INV‑02 — Idempotency binds request hash (mismatch => 409)
- **Type**: behavior
- **Maps**: INV‑02
- **Execute**: same as above (same conformance test suite)
- **Pass**: reuse same key with different payload => 409
- **Severity**: BLOCKER

#### B‑INV‑03 — Client binding on reads (mismatch non‑leak)
- **Type**: behavior
- **Maps**: INV‑03
- **Execute**: conformance test suite (extended in v1.1+ to include multi‑client)
- **Severity**: BLOCKER

#### B‑INV‑04 — No customer_id in responses
- **Type**: behavior
- **Maps**: INV‑04
- **Execute**: conformance test suite
- **Severity**: CRITICAL

#### B‑INV‑05 — Expiry sanity
- **Type**: behavior
- **Maps**: INV‑05
- **Execute**: conformance test suite (required in v1.2)
- **Severity**: BLOCKER

#### B‑INV‑06 — Challenge randomness
- **Type**: behavior
- **Maps**: INV‑06
- **Execute**: deterministic uniqueness + charset test (required v1.2)
- **Severity**: BLOCKER

---

### 4.3 Diff checks

#### D‑SCOPE — Atomic Task diff allowlist
- **Type**: diff
- **Maps**: AI‑EOS task governance
- **Execute**: `bash tools/eos/gates/diff_gate.sh "<allowlist>"`
- **Pass**: all changed files match allowlist patterns
- **Evidence**: `outputs/*_diff_gate.log`, `results.jsonl` (`diff_scope`)
- **Severity**: BLOCKER

---

## 5. How the suite integrates into Evidence Bundles

When running an Atomic Task under `make eos TASK=<id>`:

1. Record commands in `commands.jsonl`
2. Capture git evidence in `git/`
3. Capture raw tool outputs in `outputs/`
4. Emit machine results to `results.jsonl`

**Gates MUST consume only `results.jsonl`.**

---

## 6. Extending the suite

To add a new invariant (INV‑xx):
1. Define invariant in the Constitution chapter
2. Add (at least) one behavior gate asserting it
3. Add optional static gate(s) for common anti‑patterns
4. Add traceability (see `Traceability_Matrix_Consent_Setup_v1.md`)

---

## 7. Exception policy (summary)
- Exceptions are permitted only via an explicit governance process.
- Exceptions MUST NOT disable INV‑01 or INV‑02 in production.

---

## Appendix A — Recommended `results.jsonl` schema
Each line:
```json
{"check":"<id>","result":"pass|fail","severity":"BLOCKER|CRITICAL|MAJOR|MINOR","maps":["INV-xx", "..."],"notes":"optional"}
```
In v1.2, the teaching repo runner emits the **extended schema** by default.
