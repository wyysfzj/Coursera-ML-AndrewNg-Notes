# AI‑EOS Consent Setup (Go) — v1.2 Patch Notes

This patch upgrades the teaching repository and standards from v1.1 to v1.2.

## What changed (high‑signal)

### 1) Conformance Suite: INV‑05 and INV‑06 are now required behavior gates
- **INV‑05 Expiry sanity** is asserted by tests:
  - invalid RFC3339 => 400
  - near-expiry (< now+60s) => 400
- **INV‑06 Challenge quality** is asserted by tests:
  - challenge must be URL-safe base64 charset
  - challenge must be non-empty, sufficiently long
  - challenges must be unique across multiple creations (extremely low collision risk)

### 2) Multi‑client conformance for INV‑03
- Added a second demo client `demo-client-2` with its own deterministic Ed25519 key.
- Conformance now asserts:
  - consent created by `demo-client` cannot be read by `demo-client-2`
  - response must be **404** (non-leak default)

### 3) Evidence output upgraded: `results.jsonl` extended schema
`tools/eos/eos.sh` now emits JSON lines with:
- `check`
- `result`
- `severity`
- `maps` (links back to constitutional invariants)
- optional `notes`

Example:
```json
{"check":"conformance","result":"pass","severity":"BLOCKER","maps":["INV-01","INV-02","INV-03","INV-04","INV-05","INV-06"]}
```

### 4) EOS runner now produces conformance and pii_scan results in `results.jsonl`
- `conformance` runs invariant suite in atomic mode
- `pii_scan` is a deterministic heuristic guardrail against “log the entire request”

### 5) Tooling: multi-client signing support
`cmd/sign` now supports:
- `--client-id demo-client`
- `--client-id demo-client-2`

Added demo:
- `scripts/demo_atomic_multiclient.sh`

## Files of interest
- Standards:
  - `docs/standards/Conformance_Suite_Spec_v1.md` (v1.2)
  - `docs/standards/Traceability_Matrix_Consent_Setup_v1.md` / `.csv` (v1.2)
- Conformance tests:
  - `tests/invariants_test.go`
- EOS runner:
  - `tools/eos/eos.sh`
  - `tools/eos/gates/pii_scan.sh`
- Security registry / demo keys:
  - `internal/security/registry.go`
- Signer:
  - `cmd/sign/main.go`

## Quick verification
```bash
go test ./...
EOS_MODE=atomic go test ./tests -run TestInvariants -v
bash tools/eos/gates/pii_scan.sh
```

## Notes
This remains a teaching-grade Open Banking-style consent setup.  
Production profiles (FAPI, mTLS, PAR, JARM, SSA, etc.) are intentionally out of scope.
