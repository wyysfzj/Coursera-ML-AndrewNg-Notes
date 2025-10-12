
# Smart Mock Execution Engine — Technical Overview (Extracted)

> **Source cross‑reference (image → sections):**
> - **Image 1**: §1 Purpose, §2 Core Runtime Components (table).
> - **Image 2**: §3 Technology Stack, §4 Endpoint Generation & Registration Flow (4.1–4.2).
> - **Image 4**: §4.3 Hot Update/Delete, §4.4 Startup Recovery, §5 Request Execution Pipeline (diagram narrative).
> - **Image 3**: §6 Script Execution Context + example, §7 Forward Proxy Integration, §8 Schema Validation (header).
> - **Image 6**: §8 Schema Validation (details), §9 Common Function Scripts, §10 Caching Layers, §11 Error & Interrupt Model.
> - **Image 7**: §14 Concurrency & Thread Safety, §15 Security Boundary, §16 Extensibility, §17 Typical Endpoint Script Lifecycle, §18 Key Design Properties.
> - **Image 8**: §12 OpenAPI Driven Provisioning, §13 Update/Refresh Sequences (13.1–13.3).
> - **Image 9**: §19 Minimal Script Authoring Reference, §20 Risks/Considerations, §21 Summary (part 1).
> - **Image 10**: §21 Summary (closing).

---

## 1. Purpose

The execution engine (module: `smart-mock-executor`) **dynamically loads, registers, executes, validates, forwards, and hot‑reloads mock API endpoints** at runtime **without restarting** the Spring Boot application launched by `SmartMockAdminLauncher`.  
It integrates with admin CRUD operations to reflect changes **immediately in memory**.

---

## 2. Core Runtime Components

| Concern | Class / Component (key name) |
|---|---|
| Endpoint lifecycle (load / update / delete / forward / schema / script) | `com.hsbc.openbanking.smartmockexecutor.core.MockEndpointManager` |
| Spring MVC dynamic handler (per endpoint) | `com.hsbc.openbanking.smartmockexecutor.core.impl.DefaultMockHandler` |
| Script cache (Groovy objects) | `com.hsbc.openbanking.smartmockexecutor.script.endpoint.EndpointHandlerScriptManager` |
| Script utility façade exposed as `$func` | `$func` (script helper facade) |
| Request execution metadata | `com.hsbc.openbanking.smartmockexecutor.common.model.HandlerExecutionMeta` |
| Forward proxy pipeline | `com.hsbc.openbanking.smartmockexecutor.forward.ProxyForwardProcessor` |
| Startup recovery (warm caches) | `com.hsbc.openbanking.smartmockexecutor.SmartMockExecutorRecover` |
| Common function script manager | `com.hsbc.openbanking.smartmockexecutor.script.commonfunc.CommonFuncScriptManager` |
| Common function wrapper validation | `com.hsbc.openbanking.smartmockexecutor.script.commonfunc.CommonFunc` |
| Environment param caching | Used via `$func.env` through cache managers (e.g., `ProjectEnvParamCacheManager`) |
| Data mapper script generator | `com.hsbc.openbanking.smartmock.common.script.DataMapperScriptGenerator` |
| Schema validation orchestrator | `com.hsbc.openbanking.smartmockexecutor.common.schema.EndpointSchemaManager` |
| Schema refresh proxy | `com.hsbc.openbanking.smartmockexecutor.manager.SchemaManagerProxy` |
| Endpoint in‑memory refresh proxy | `com.hsbc.openbanking.smartmockexecutor.manager.EndpointManagerProxy` |
| Group / Project cache orchestration | `com.hsbc.openbanking.smartmockexecutor.manager.GroupManagerProxy`, `ProjectManagerProxy` |
| OpenAPI — definitions (script + templates) | `com.hsbc.openbanking.smartmock.common.swagger.SwaggerYamlParser` |
| Script skeleton generation | `com.hsbc.openbanking.smartmockexecutor.script.ScriptGenerator` (referenced) |

---

## 3. Technology Stack

- **Spring Boot + Spring MVC** dynamic `RequestMappingInfo` registration (hot add/remove).  
- **Groovy** runtime compilation (`GroovyUtil` via `ScriptGenerator`) — cached in `EndpointHandlerScriptManager`.  
- **MyBatis/MyBatis‑Plus** mappers (project, group, schema relations) — e.g., `MockEndpointMapper`.  
- **Liquibase** migrations (DB schema).  
- **In‑memory multi‑tier caches** (endpoints, templates, env params, mock data).  
- Optional **Redis** (`StringRedisTemplate`) for distributed coordination/cache sync (used inside `$func` utilities).  
- **Custom JSON / template engines** (File / JSON / Dynamic) to auto‑generate script/HTTP classes.  
- **OpenAPI v3 parsing** (`SwaggerYamlParser`) to auto‑generate endpoint script & templates.  
- **Pluggable extension system** (`PluginManager`, common function module).  
- **Concurrency** operators (exposed via `$func.concurrency`, manager not shown).  
- **Forward HTTP** execution using internal `HttpClientFactoryManager` + `SyncHttpClientFactory` (referenced in forward processor).  
- **Schema validation pipeline** assembled by `EndpointSchemaManager`.

---

## 4. Endpoint Generation & Registration Flow

### 4.1 Creation Trigger

Admin layer (`MockEndpointService.addMockEndpoint`) **persists** row → emits **in‑memory load** via executor handler → `EndpointManagerProxy.loadEndpointToMemory` → `MockEndpointManager.loadEndpoint`.

### 4.2 DB → Runtime Transformation

1. **Retrieve** `MockEndpointWholeInfoEntity` (joins project/group root paths).  
2. **Build full URL** (`projectRootPath + groupRootPath + endpoint.path`) using `AntPathMatcher` (see `setEndpointWholeUrl` in `MockEndpointManager`).  
3. **Construct `RequestMappingInfo`** with:  
   - **Path** = whole URL  
   - **Method** = enum converted to upper case  
   - **Consumes** = endpoint content type (fallback `*/*`)  
4. **Generate / compile Groovy handler script**:  
   - Script may already be stored (or **auto‑generated from OpenAPI** via `SwaggerYamlParser`).  
   - Success path uses `generateSuccessResponseAsDefaultScript` adding `$resp.status`, headers, body.  
   - Compiled and cached in `EndpointHandlerScriptManager`.  
5. **Optionally** build `EndpointSchemaValidator` via `EndpointSchemaManager.createSchemaValidator` if validation flags are active.  
6. **Instantiate** `DefaultMockHandler` with: endpoint ID, URL pattern, HTTP method, `HandlerExecutionMeta` (project/group/endpoint metadata, schema & forward flags), optional schema validator.  
7. **Register handler** into Spring MVC dispatcher (`RequestMappingHandlerMapping.registerMapping`).

### 4.3 Hot Update / Delete

- **Update** script / forward / schema: `EndpointManagerProxy.refreshEndpointInMemory` — internally:  
  - Replace cached Groovy object (`updateEndpointScript`).  
  - Refresh schema validator (`updateSchemaVerifierInEndpointHandler`).  
  - Refresh forward meta (`updateForwardProxyInEndpointHandler`).  
- **Delete**: `deleteEndpointsInMemory` → `MockEndpointManager.removeEndpoint` (unregister mapping + purge script cache).  
- **Forward bulk update**: `refreshEndpointsForwardProxyInMemory`.

### 4.4 Startup Recovery

`SmartMockExecutorRecover` iterates persisted projects/groups/endpoints to:  
- Rebuild caches (templates, env params, plugins, endpoints).  
- Re‑register mappings.  
- Reload common functions.

---

## 5. Request Execution Pipeline (narrative)

```
Client → Spring Dispatcher → DefaultMockHandler
      → (optional) SchemaValidator
      → Groovy Script ($req,$resp,$func)
      → (optional) ForwardProxy
      → Template/Env/Cache Managers
      → Write HTTP Response → Client
```

---

## 6. Script Execution Context

Inside `DefaultMockHandler#doMockHandling` (Groovy entry function = `ScriptGenerator.FUNCTION_NAME`).

### Objects exposed to Groovy

- **`$req`** — request abstraction (class `Req`, path assembled at `executor/core/*/req.java`): headers, body, params.  
- **`$resp`** — mutable response model (status, headers, body).  
- **`$func`** — instance of `$func` façade.

### `$func` utilities (selected)

- `template.file/json/dyn(...)` → Template load (via `MockTemplateCacheManager`).  
- `mock.find/update/delete(...)` → Mock data (via `MockDataManager`).  
- `env.get(...)` → Environment parameters cache.  
- `http.sync(...)` → HTTP client factory for downstream calls (when not using auto‑forward).  
- `dataMapper.apply(...)` → Script generated by `DataMapperScriptGenerator`.  
- `plugin.exec(...)` → Plugin invocation via `PluginManager`.  
- `error(...)` → Interrupt helpers raising `SmartMockInterruptException`.  
- Common functions dynamically added via `CommonFuncScriptManager` validated by `CommonFunc`.

### Example (Auto‑generated OpenAPI success pattern)

```groovy
$resp.status = 200
$resp.headers.'content-type' = 'application/json'
$resp.body = $func.template.json('Some_Endpoint_200_application_json')
```

---

## 7. Forward Proxy Integration

When endpoint meta (`HandlerExecutionMeta.isActiveForward`) is **true**, handler consults:

- Populated forward domain & original path variables.  
- `ProxyForwardProcessor`:
  - Replaces `{variable}` placeholders using incoming path/query.  
  - Merges query string.  
  - Acquires project‑specific `SyncHttpClientFactory`.  
  - Executes downstream request; response mapped into current response (status, headers, body).  
  - Failure to resolve client factory throws `SmartMockEndpointRunningException`.

---

## 8. Schema Validation

If endpoint flags enable validation:

1. `EndpointSchemaManager.createSchemaValidator` builds validator chain using schema + dependency refs (joins via `MockEndpointSchemaRefMapper`).  
2. On schema change → `SchemaManagerProxy.refreshSchemaManager` computes impacted endpoints → `MockEndpointManager.updateSchemaVerifierInEndpointHandler`.  
3. Validator executes **before** script; failures raise mapped exceptions with enriched metadata.

---

## 9. Common Function Scripts

- Stored **per project** in DB, loaded & compiled into `CommonFuncScriptManager`.  
- Wrapper `CommonFunc` enforces **first parameter type is `Object`** (the `$func` context).  
- Refresh trigger path: admin update → executor proxy (e.g., `CommonFunctionManagerProxy.refreshCommonFuncScript`).

---

## 10. Caching Layers (High‑Level)

| Cache | Manager |
|---|---|
| Endpoint Groovy handlers | `EndpointHandlerScriptManager` |
| Endpoint metadata / handler mapping | `MockEndpointManager` |
| Templates (json/file/dyn) | `MockTemplateCacheManager` |
| Mock data per group | `MockDataManager` |
| Environment parameters | `ProjectEnvParamCacheManager` |
| Data mapper compiled rules | `MockDataMapperManager` |
| Plugins | `PluginManager` |

_All refreshed through corresponding Proxy or ExecutorHandler pathways invoked from admin services._

---

## 11. Error & Interrupt Model

- Base exception: `BaseSmartMockException`.  
- **Script runtime**: `SmartMockScriptRunningException`.  
- **Endpoint runtime**: `SmartMockEndpointRunningException`.  
- **Business validation**: `SmartMockBusinessException`.  
- **Interrupt (early return)**: `SmartMockInterruptException` (thrown via `$func.error(...)`, carries custom status/headers/body).  
- Metadata enrichment added using `HandlerExecutionMeta` for observability.

---

## 12. OpenAPI Driven Provisioning

`SwaggerYamlParser`:

1. Iterates **paths**.  
2. For each (method, path) builds `MockEndpointDefinition` (method, path, contentType, auto name).  
3. Generates response templates (`JsonTemplateDefinition`) + **default success script** lines (`generateSuccessResponseAsDefaultScript`).  
4. Admin import (`/mock-admin/dev-ops/openapi/import`) persists endpoints + templates; executor loads them immediately.

---

## 13. Update / Refresh Sequences

### 13.1 Endpoint Update (Script / Flags / Schemas)
`Admin Update Request` → `MockEndpointService.update` → DB update + SchemaRef refresh → `EndpointExecutorHandler.refresh` → `EndpointManagerProxy.refresh` → `MockEndpointManager.update*`.

### 13.2 Schema Change
`Schema Add/Update/Delete` → `EndpointSchemaManager.observe` → `SchemaManagerProxy.refresh` → `MockEndpointManager.updateSchemaVerifierInEndpointHandler`.

### 13.3 Common Function Script Change
`Admin CommonFunc Update` → Persist → `CommonFunctionManagerProxy.refresh` → `CommonFuncScriptManager` → affects future executions.

---

## 14. Concurrency & Thread Safety

- Endpoint script & mapping modifications guarded by **`ReentrantLock`** in `MockEndpointManager`, ensuring **atomic register/unregister/script swap**.  
- Script cache operations synchronized in `EndpointHandlerScriptManager`.  
- `$func.concurrency` (manager not shown) provides higher‑level async utilities for scripts.

---

## 15. Security Boundary

- Enforcement (roles/permissions) occurs in **Admin controllers** (e.g., `@PermissionVerify`).  
- Executor layer **assumes** upstream/front‑end has authenticated external traffic; no additional RBAC checks inside scripts.

---

## 16. Extensibility

| Extension Point | Mechanism |
|---|---|
| New utility for scripts | Add field & nested util in `$func` |
| New response templates | Extend template manager or OpenAPI parser |
| New validation layer | Inject before Groovy invocation inside `DefaultMockHandler` |
| New DSL helpers | Compile via `CommonFuncScriptManager` |

---

## 17. Typical Endpoint Script Lifecycle

1. Author script (manual or OpenAPI default).  
2. Saved via Admin → validated (`GroovyUtil.genGroovyObject` in service path).  
3. Persisted script referenced by endpoint ID.  
4. Executor compiles & caches at load.  
5. Request invokes compiled method; `$func` utilities orchestrate dynamic data/templates/downstream calls.  
6. Updates trigger **safe replacement**; failures roll back to previous Groovy object (see `updateEndpointScript`).

---

## 18. Key Design Properties

| Property | Approach |
|---|---|
| Zero‑restart evolution | Dynamic Spring MVC mapping + Groovy recompilation |
| Safe script update | Lock + previous object fallback |
| Deterministic path resolution | Project + Group root + Endpoint relative path |
| Backward compatibility | Default success script when no response schema |
| Isolation | Endpoint script context limited to `$req/$resp/$func` |
| Observability | Enriched exception metadata (project/group/endpoint) |
| Performance | In‑memory caches; no reflection per request beyond Groovy dispatch |

---

## 19. Minimal Script Authoring Reference

| Goal | Snippet |
|---|---|
| Static JSON body | ```groovy\n$resp.status = 200; $resp.body = [ok:true]\n``` |
| Use JSON template | ```groovy\n$resp.body = $func.template.json('TemplateName')\n``` |
| Forward downstream | ```groovy\n def r = $func.http.get(\"${domain}/x\"); $resp.body = r.body // (or enable endpoint forward flag)\n``` |
| Interrupt with error | ```groovy\n $func.error(400, ['x-trace':'abc'], [message:'Bad input'])\n``` |
| Read env param | ```groovy\n def apiKey = $func.env.get('API_KEY')\n``` |
| Use mock data | ```groovy\n def item = $func.mock.find('customer','C123')\n``` |

---

## 20. Risks / Considerations

- **Script security** (Groovy sandboxing not shown; potential for arbitrary code execution).  
- Forward proxy depends on correct factory initialization; missing client yields runtime failure.  
- Multi content‑types in OpenAPI request bodies currently rejected (single or empty allowed).  
- Schema validation only active when flag set; silent skip otherwise.

---

## 21. Summary

The execution engine converts persisted endpoint definitions into **live Spring MVC handlers** through dynamic mapping + Groovy script compilation.  
A layered set of managers (endpoint, schema, template, env, mock data, plugins) coupled with proxy refresh components enables **granular hot reload**.  
OpenAPI ingestion accelerates provisioning by generating both endpoint scripts and reusable response templates.  
The design emphasizes runtime mutability, isolation of scripting utilities, and structured validation / forwarding capabilities.
