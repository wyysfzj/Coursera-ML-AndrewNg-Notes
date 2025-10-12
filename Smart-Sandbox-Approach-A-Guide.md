
# Smart Sandbox (Approach A) — Build Guide (Codex/Copilot Edition)

This guide upgrades the **Smart Mock Execution Engine** into a full **Developer‑Portal Sandbox runtime** using **Approach A**:

> **Approach A**: reuse **Smart Mock** as the sandbox runtime and **embed** sandbox features inside it (tenantization, scenarios, chaos/latency, idempotency, clock, webhook emitter, logs/replay, control‑plane APIs).  
> Goal: **From the portal**, a developer picks an API → creates a sandbox → gets a base URL → runs requests with scenario/chaos headers → views logs → (optionally) triggers webhooks — **all powered by Smart Mock**.

This file contains **end‑to‑end steps**, **granular tasks**, **ready‑to‑paste prompts** for Codex/Copilot, and **acceptance checks**.

---

## 0) Prereqs & Getting Started

- JDK **21**, Git, Docker Desktop (optional for Postgres).  
- VS Code extensions: **GitHub Copilot**, **Copilot Chat**, **Extension Pack for Java**.  
- Download and unzip the starter: `smart-mock-starter.zip` (place at repo root).  
- Keep `Smart-Mock-Execution-Engine-Overview.md` open in a pinned tab for Copilot context.

Verify the starter boots:

```bash
./gradlew :executor-admin:bootRun
curl http://localhost:8080/healthz
```

---

## 1) What we will build

**Per-sandbox runtime** (single Smart‑Mock instance, tenantized by `sandboxId`):

```
Ingress (dev123.sbx.api.example.com)
 → Smart‑Mock (tenantized)
    ├ Endpoint/caches keyed by sandboxId
    ├ $func.sbx.(scenario|chaos|idempotency|clock|webhook)
    ├ Request/Response log + Replay
 → Postgres schema per sandbox (or prefixed tables for local dev)
```

**Control plane (Minimal)**

```
POST   /sandboxes                   # create (register schema, import OpenAPI, seed)
POST   /sandboxes/{id}/seed         # seed/refresh fixtures
POST   /sandboxes/{id}/events/{type}# enqueue webhook
POST   /sandboxes/{id}/clock/offset # time travel (minutes)
GET    /sandboxes/{id}/status       # health/links
DELETE /sandboxes/{id}              # teardown / TTL clean
```

---

## 2) Build Path Overview

We’ll first complete the **Smart Mock core (Sprint A–D)** quickly (T01–T14 essentials), then add **Sandbox features** (S01–S16). If you already completed T01–T22 you can jump straight to **Sandbox Sprint**.

> Keep a tight **micro‑loop** with Copilot: open target file → paste prompt → accept → run acceptance.

---

## 3) Essentials Recap (core Smart Mock tasks)

If not done yet, implement T01–T14 from the “Smart Mock step‑by‑step” document:

- T01–T06: project, DTOs, `$req/$resp`, exceptions, dynamic mapping, Groovy compile/bind.  
- T07–T09: `$func` base utils, caches, startup recovery.  
- T10–T12: forward proxy, schema validation, pipeline.  
- T13–T14: OpenAPI import, admin CRUD & hot update.

Acceptance: you can import a spec and call a live endpoint returning a template JSON; hot update changes behavior immediately.

---

## 4) Sandbox Sprint (Approach A) — Tasks & Prompts

> **Prefix “S”** denotes Sandbox tasks. Each task includes a **prompt** crafted for Copilot/Codex. Paste into Chat while the target file is open.

### S01 — Tenant Context & Filters
**Goal**: Recognize `sandboxId` from `X-Sandbox-Id` header or subdomain; add correlation ID.

**Files**:  
`executor-core/.../tenant/{TenantContext,TenantContextHolder}.java`  
`executor-core/.../web/filter/{SandboxTenantFilter,CorrelationIdFilter}.java`

**Prompt**  
> Create `TenantContext(sandboxId, tenantId?)` with `ThreadLocal` holder. Implement `SandboxTenantFilter` that reads `X-Sandbox-Id` or subdomain `{id}.sbx.` and sets it in the context; clear in `finally`. Add `CorrelationIdFilter` that ensures `X-Correlation-Id` (UUID) is present in request MDC and response headers. Order: correlation first, sandbox second. Provide unit tests for header and host parsing.

**Acceptance**: Calls without header but with `dev123.sbx.local` set `sandboxId=dev123`; correlation ID always present.

---

### S02 — Tenantize Managers & Caches
**Goal**: All caches/managers key off `sandboxId` (isolation).

**Files**:  
`MockEndpointManager`, `EndpointHandlerScriptManager`, `MockTemplateCacheManager`, `MockDataManager`, `ProjectEnvParamCacheManager`, `MockDataMapperManager`, `PluginManager`

**Prompt**  
> Refactor internal maps to `ConcurrentMap<String sandboxId, ConcurrentMap<...>>`. Add helper `currentSandbox()` from `TenantContextHolder`. Ensure all register/unregister/swap operations acquire a **per-sandbox** `ReentrantLock`. Keep public APIs unchanged; route internal reads/writes by sandbox.

**Acceptance**: Adding endpoints/templates in `sbxA` does not leak to `sbxB` during tests.

---

### S03 — Sandbox Tables (Liquibase)
**Goal**: Persistence for sandbox runtime state.

**Files**:  
`executor-admin/src/main/resources/db/changelog/002-sandbox.yaml`

**DDL** (for Postgres; use schema‑per‑sandbox **or** single schema with sandboxId column).

```
sbx_env_param   (sandbox_id, k, v jsonb, pk(sandbox_id,k))
sbx_idempotency (sandbox_id, key pk, req_hash bytea, resp jsonb, created_at timestamptz)
sbx_webhook_event(id uuid pk, sandbox_id, type, payload jsonb, status enum(PENDING,DELIVERED,FAILED,DLQ), attempts int, next_at timestamptz, created_at timestamptz)
sbx_reqlog      (id bigserial pk, sandbox_id, ts timestamptz, method, path, req jsonb, resp jsonb, status int, latency_ms int, corr_id)
sbx_mock_data   (sandbox_id, namespace, k, v jsonb, pk(sandbox_id,namespace,k))
```

**Prompt**  
> Add a Liquibase changeSet `002-sandbox.yaml` creating the five tables above with indexes on `(sandbox_id)`, `(next_at)`, `(ts)`, `(key)`. Wire it into `db.changelog-master.yaml`. For local dev we’ll use a single schema (no schema‑per‑sandbox).

**Acceptance**: Liquibase applies; indexes exist.

---

### S04 — Sandbox Repositories
**Goal**: Read/write per‑sandbox rows.

**Files**:  
`executor-core/repo/{EnvParamRepo,IdempotencyRepo,WebhookEventRepo,ReqLogRepo,MockDataRepo}.java`

**Prompt**  
> Implement repositories using Spring JDBC. Each method takes `String sandboxId` and translates into `WHERE sandbox_id=?`. Provide methods: env.get/set, idempotency.find/save, webhook.nextDue(now)/markDelivered/retryOrDlq, reqlog.append/findLatest(limit), mock.find/update/delete. Write minimal tests with H2 if Postgres is unavailable.

**Acceptance**: CRUD works; selecting `nextDue()` yields due events.

---

### S05 — `$func.sbx` (Scenario & Chaos)
**Goal**: Header‑driven scenario selection + chaos toggles.

**Files**:  
`executor-core/sbx/{ScenarioService,ChaosService,SandboxFacade}.java`  
Expose as `$func.sbx.scenario` and `$func.sbx.chaos`.

**Prompt**  
> Implement `ScenarioService.resolve(headers, default)` reading `X-Sandbox-Scenario` (fallback default). Implement `ChaosService.applyFromHeader(headers)` parsing `X-Chaos` like `latency=200,5xx=2%`. Delay if latency>0; with probability error% throw `SmartMockInterruptException(500,…)`. Add Groovy usage docs. Wire into a new `$func.sbx` facade available to scripts.

**Acceptance**: Scenario header switches behavior; chaos can inject delay/errors.

---

### S06 — `$func.sbx` (Idempotency)
**Goal**: Deterministic replays using `Idempotency-Key`.

**Files**:  
`executor-core/sbx/IdempotencyService.java` (uses `sbx_idempotency`)  
Expose as `$func.sbx.idempotency.remember(key){ ... }`

**Prompt**  
> Implement `remember(key, Supplier<T>)` which checks `(sandboxId,key)`; if present, return stored `resp`, else execute supplier, persist `req_hash + resp` and return. Provide `hashCurrentRequest()` helper using method+path+body. Unit test that repeated calls return identical body & status.

**Acceptance**: Same key returns same response across calls.

---

### S07 — `$func.sbx` (Clock / Time Travel)
**Goal**: Per‑sandbox clock offset (minutes).

**Files**:  
`executor-core/sbx/ClockService.java` (stores offset in `sbx_env_param`)  
Expose as `$func.sbx.clock.now()`

**Prompt**  
> Implement `now()` that returns `Instant.now().plus(offset)`. Add admin method to set offset (minutes). Provide Groovy helper usage and verify through a test that advancing the clock changes timestamps produced by scripts.

**Acceptance**: Offset affects `$func.sbx.clock.now()`.

---

### S08 — Webhook Emitter & Signing
**Goal**: Queue → deliver with retry/backoff; manual redelivery.

**Files**:  
`executor-core/sbx/WebhookEmitter.java` (enqueue), `executor-core/sbx/WebhookWorker.java` (@Scheduled)  
`executor-admin/admin/WebhookAdminController.java` (redeliver/test)

**Prompt**  
> Implement `emit(type,payload,opts)` to insert into `sbx_webhook_event`. Add `WebhookWorker` running every 2s: fetch `nextDue()` events, POST to configured endpoint with HMAC header `X-Signature`, mark DELIVERED on 2xx else backoff with jitter and increment attempts; move to DLQ after N attempts. Add `POST /internal/webhooks/redeliver/{id}` and `POST /sandboxes/{id}/events/{type}` to enqueue manual events.

**Acceptance**: Events deliver, retry, and can be redelivered via API.

---

### S09 — Request/Response Logging + Replay
**Goal**: Store redacted call snapshots & generate curl replay.

**Files**:  
`executor-core/web/ReqLogInterceptor.java`, `executor-admin/admin/ReqLogController.java`

**Prompt**  
> Add an interceptor around `DefaultMockHandler` that measures latency, captures req/resp (redact secrets), writes to `sbx_reqlog`. Expose `GET /internal/reqlog?limit=50` returning recent entries and `GET /internal/reqlog/{id}/replay` returning a ready-to-run cURL command using the same headers (except auth) and the sandbox base URL.

**Acceptance**: You can fetch last N entries and get a cURL replay command.

---

### S10 — Sandbox Admin API (Control Plane Lite)
**Goal**: Minimal APIs for the portal to create/seed/manage a sandbox.

**Files**:  
`executor-admin/admin/SandboxAdminController.java`

**Prompt**  
> Implement:  
> - `POST /sandboxes {openApiVersion, scenarioPack, ttlHours}` → create sandbox: register in DB, ensure rows initialized in all `sbx_*` tables, set env param `OPENAPI_VERSION`, call OpenAPI import, run `SeedingService`. Return `{sandboxId, baseUrl, apiKey?}`.  
> - `POST /sandboxes/{id}/seed` → re-run seeding.  
> - `POST /sandboxes/{id}/clock/offset {minutes}` → set offset.  
> - `GET /sandboxes/{id}/status` → return service health + counts.  
> - `DELETE /sandboxes/{id}` → mark expired + cleanup.  

**Acceptance**: A sandbox can be created and immediately called with its base URL.

---

### S11 — Seeding Service & Scenario Packs
**Goal**: Deterministic fixture data & scenario pack defaults.

**Files**:  
`executor-core/seed/SeedingService.java` + `resources/fixtures/{version}/...`

**Prompt**  
> Implement `SeedingService.seed(sandboxId, version, pack)` that loads JSON fixtures from `classpath:/fixtures/{version}/{pack}/` into `sbx_mock_data` and env params (e.g., `API_DOMAIN`). Provide three packs: `happy_path`, `insufficient_funds`, `rate_limit`. Include golden webhook payloads. Return counts seeded.

**Acceptance**: After seeding, `$func.mock.find('customer','C123')` returns seeded object.

---

### S12 — OpenAPI Version Pinning (per sandbox)
**Goal**: Each sandbox references a specific spec version.

**Files**:  
Update `OpenApiImportController` to accept `version` and read spec from `resources/openapi/{version}.yaml` or object store; set `sbx_env_param('OPENAPI_VERSION')`.

**Prompt**  
> Extend import to take `version` and persist it in `sbx_env_param`. On `POST /sandboxes`, pass the version to the import flow. Add an API `POST /sandboxes/{id}/openapi/refresh` that re-imports with pinned version.

**Acceptance**: Changing version updates endpoints and templates safely.

---

### S13 — Helm Chart (optional; for cluster)
**Goal**: Per‑sandbox chart to run in a namespace.

**Files**:  
`charts/smart-mock/` (values: `SANDBOX_ID`, `OPENAPI_VERSION`, `ingress.host`, `db.*`)

**Prompt**  
> Create a Helm chart that deploys a single sandbox release with `Deployment`, `Service`, `Ingress`, `ConfigMap`, `Secret`, and a seeding `Job`. Include a `NetworkPolicy` that only allows egress to configured hosts. Provide `values.example.yaml` and README with install commands.

**Acceptance**: `helm install` yields a working sandbox at `https://<host>` (cluster env).

---

### S14 — Security & Quotas
**Goal**: Guard rails for cost & abuse.

**Files**:  
`executor-admin/config/SecurityConfig.java`, `executor-core/limit/RateLimiter.java`, TTL job.

**Prompt**  
> Protect `/admin/**` with role `ADMIN`. Add a simple token-bucket rate limiter per sandbox (requests/minute) and enforce in `DefaultMockHandler`. Implement TTL cleanup (scheduled) that disables expired sandboxes and purges caches. On exceeded rate, return 429 with `retry-after` header.

**Acceptance**: High call rate is throttled; expired sandbox returns 410 Gone.

---

### S15 — Portal Integration Notes & Examples
**Goal**: Show how the Developer Portal interacts with APIs.

**Files**:  
`docs/portal-integration.md`

**Prompt**  
> Write a portal integration guide:  
> 1) `POST /sandboxes` → get `{sandboxId, baseUrl}`.  
> 2) Use `baseUrl` in the API explorer.  
> 3) Scenario Launcher UI sets `X-Sandbox-Scenario` & `X-Chaos`.  
> 4) Webhook tester calls `POST /sandboxes/{id}/events/{type}`.  
> 5) Logs page calls `GET /internal/reqlog?limit=N` + replay endpoint.  
> Provide sample JS `fetch()` snippets and curl commands.

**Acceptance**: A portal dev can follow it to wire their UI in a day.

---

### S16 — Showcase Groovy Scripts using `$func.sbx`
**Goal**: Prove the new DSL in action.

**Files**:  
`samples/scripts/PaymentCreate.groovy`

**Script example**:

```groovy
import com.hsbc.openbanking.smartmockexecutor.script.Req
import com.hsbc.openbanking.smartmockexecutor.script.Resp
import com.hsbc.openbanking.smartmockexecutor.func.FuncFacade

class PaymentCreate {
  Object handle(Req $req, Resp $resp, FuncFacade $func) {
    def scen = $func.sbx.scenario.resolve($req.headers.'X-Sandbox-Scenario', 'ok')
    $func.sbx.chaos.applyFromHeader($req.headers.'X-Chaos')
    switch(scen) {
      case 'insufficient_funds':
        $func.error(402, [:], [code:'insufficient_funds']); break
      case 'rate_limit':
        $func.error(429, ['retry-after':'1'], [code:'rate_limited']); break
      default:
        $resp.status = 200
        $resp.headers.'content-type' = 'application/json'
        $resp.body = $func.template.json('Payment_200_application_json')
    }
    $resp.body = $func.sbx.idempotency.remember($req.headers.'Idempotency-Key'){ $resp.body }
    $func.sbx.webhook.emit('payment.created', $resp.body, null)
    return null
  }
}
```

**Acceptance**: All scenario branches work; idempotency yields identical body on replay; webhook is enqueued.

---

## 5) End‑to‑End Flow (Portal to Sandbox)

1. **Portal – Create Sandbox**  
   `POST /sandboxes { openApiVersion: "v1", scenarioPack:"happy_path", ttlHours:24 }` → `{ sandboxId, baseUrl }`

2. **Portal – Explore APIs**  
   Use `baseUrl` to call endpoints, add headers:  
   - `X-Sandbox-Scenario: ok|insufficient_funds|rate_limit`  
   - `X-Chaos: latency=200,5xx=2%`  
   - `Idempotency-Key: UUID`

3. **Portal – Inspect Logs & Replay**  
   `GET /internal/reqlog?limit=50` → show recent calls; `GET /internal/reqlog/{id}/replay` → one‑click replay.

4. **Portal – Trigger Webhooks**  
   `POST /sandboxes/{id}/events/payment.created` → send signed events with retries.

5. **Portal – Time Travel**  
   `POST /sandboxes/{id}/clock/offset { minutes: 60 }` → simulate future timestamps.

6. **Portal – Teardown**  
   `DELETE /sandboxes/{id}` → cleanup.

---

## 6) Quick E2E (local)

```bash
./gradlew :executor-admin:bootRun

# 1) Create sandbox
curl -s http://localhost:8080/sandboxes -H "Content-Type: application/json" \
  -d '{"openApiVersion":"v1","scenarioPack":"happy_path","ttlHours":24}'

# Assume {sandboxId, baseUrl} returned; set:
SBX=dev123
BASE=http://localhost:8080

# 2) Call API
curl -H "X-Sandbox-Id: $SBX" -H "Idempotency-Key: 1" \
     -H "X-Sandbox-Scenario: ok" \
     -X POST $BASE/v1/payments

# 3) Change scenario
curl -H "X-Sandbox-Id: $SBX" -H "X-Sandbox-Scenario: insufficient_funds" \
     -X POST $BASE/v1/payments -i

# 4) View logs
curl "$BASE/internal/reqlog?limit=5" -H "X-Sandbox-Id: $SBX"

# 5) Trigger webhook
curl -X POST "$BASE/sandboxes/$SBX/events/payment.created"
```

---

## 7) Known Limitations & Mitigations (Approach A)

- **Groovy sandboxing** not enforced by default → run in non‑prod project, restrict classpath or add `SecureASTCustomizer`.  
- **Forward proxy** requires correct client factory init → add health checks.  
- **Request body content‑types** limited (single/none) → extend OpenAPI ingestion for multi‑content.  
- **Schema validation** only when flag is on → document clearly in portal.

---

## 8) How to use this guide with Copilot (best practices)

- Always keep the **target file open** when prompting.  
- **Name classes and file paths** in prompts.  
- Add **acceptance criteria** and a **sample test/curl** into the prompt.  
- If Copilot produces partial code: “*Continue from where you left off; implement remaining methods and add imports only.*”  
- If the type system fights you: “*Fix compile errors; do not alter public method signatures.*”  
- Commit small, runnable changes every 10–15 minutes.

---

## 9) Deliverables Checklist (MVP)

- [ ] Sandbox Admin API (`/sandboxes`, `/seed`, `/events`, `/clock/offset`, `/status`, `DELETE`).  
- [ ] `$func.sbx` modules (scenario, chaos, idempotency, clock, webhook).  
- [ ] Tenantized managers + caches + per‑sandbox locks.  
- [ ] Request logs & replay endpoints.  
- [ ] OpenAPI import with version pinning per sandbox.  
- [ ] Seeding service & scenario packs.  
- [ ] Security (admin auth) + basic rate limiting + TTL cleanup.  
- [ ] Helm chart (optional for cluster).

---

**You’re ready to build the Smart Sandbox with Codex/Copilot.**  
Use this document as the driver for your VS Code sessions. For any task you’d like me to pre‑generate code for, name the task (e.g., **S05**, **S08**, **T13**) and I’ll drop full class files tailored to your current repo.
