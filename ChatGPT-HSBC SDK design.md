# HSBC SDK design

**User:** Anonymous (os86@naifeistation.com)  
**Created:** 10/25/2025 21:56:01  
**Updated:** 10/25/2025 22:02:46  
**Exported:** 10/25/2025 22:06:09  
**Link:** [https://chatgpt.com/c/68fcd6f0-34a0-8329-9d83-3647aa1aac1d](https://chatgpt.com/c/68fcd6f0-34a0-8329-9d83-3647aa1aac1d)  

## Prompt:
![Uploaded image](https://chatgpt.com/backend-api/estuary/content?id=file_00000000405861f5a6a7dc37f82557a3&ts=489277&p=fs&cid=1&sig=5cd3e57d0810068d0dc95cf1dfda35424b32bc2621286c191f039bf85d060ece&v=0)

the attached image show the SDK solution of SCB. SCB generate SDK for its account opening api and  account opening page, Ctrip use this SDK in Ctrips'app.  customer can open SCB account in Ctrip APP.  Now I need produce the similar SDK for HSBC. please provide the detail design

## Response:
Thought for 6m 42s

## Prompt:
Below is a complete, implementation‑ready design for an **HSBC Partner Account‑Opening SDK (AOSDK)** that mirrors (and improves on) the SCB/Ctrip pattern you showed—so a partner like Ctrip can let customers open an HSBC account _without leaving_ the partner app.

* * *

0) Executive summary
--------------------

**Goal:** Provide a mobile SDK (iOS + Android; optional Web) that launches an **HSBC‑hosted account‑opening (AO) experience** inside the partner app, prefilled with partner‑provided data, and submits the application back to HSBC—securely, compliantly, and with excellent observability.

**Core design choices**

*   **Hybrid UI**: Native SDK that wraps a **hosted HSBC AO Web** (WebView/Custom Tabs/SFSafariViewController) plus **optional native plugins** (ID doc scan, liveness, e‑signature) to reduce friction and handle device capabilities.
*   **Security**: OAuth 2.1 / OIDC + PKCE for mobile, certificate pinning, device attestation (Play Integrity/App Attest), hardware‑backed keystore/keychain, JWS/JWE payloads, and **server‑to‑server prefill token** so PII never transits the client in the clear.
*   **Event model**: Rich callbacks to the partner (progress, errors, completion) and **telemetry** to HSBC (privacy‑safe).
*   **Partner‑friendly**: Simple SDK API (`startAccountOpening(...)`), deep‑link handling, theming hooks, and comprehensive samples.

* * *

1) End‑to‑end customer journey (mirrors your SCB flow)
------------------------------------------------------

**HSBC × Ctrip AO Journey**

1.  **Entry point** in Ctrip (e.g., Finance tab → “Open HSBC Account”).
2.  **Prefill consent & token**: Ctrip obtains user consent and requests an HSBC **Prefill Token** via its backend (server‑to‑server) using partner credentials.
3.  **Launch HSBC AO** in‑app via the SDK: `HSBCAOSdk.start(...)`.
4.  **HSBC AO pages** open in a secure WebView (or Custom Tab/SafariView) inside Ctrip; prefilled fields appear (name, ID, address, etc.).
5.  **KYC**: Native plugin (doc capture + liveness) or HSBC hosted web KYC; results posted securely to HSBC.
6.  **Branch appointment** (if required by jurisdiction) scheduled in‑flow.
7.  **Submission**: Application submitted; SDK returns **result** (success with `applicationId`, pending, or error) to Ctrip.
8.  **Status & resume**: Customer can resume via deep link; Ctrip can show application status via **Application Status API**.

* * *

2) High‑level architecture
--------------------------

```
[ Partner App (Ctrip) ]
     │
     │  AOSDK (HSBC) – iOS/Android
     │   ├─ Core (Session, Tokens, Config, Consent, Logger, Events)
     │   ├─ Security (OIDC+PKCE, Pinning, Attestation, DeviceChecks)
     │   ├─ UI Container (WebView / Custom Tab / SFSafariViewController)
     │   ├─ Native KYC Plugins (Doc Scan, NFC, Liveness) [optional]
     │   ├─ Prefill Adapter (Token-based prefill)
     │   └─ Appointment & Status Adapters
     │
Internet (TLS + Pinning, JWS/JWE)
     │
[ HSBC Partner Gateway ]
     ├─ OAuth/OIDC (Authz Code + PKCE)
     ├─ Prefill API (server-to-server)
     ├─ AO Web (responsive pages)
     ├─ KYC/AML/Screening (internal or 3rd-party)
     ├─ Document Service (upload, OCR, classification)
     ├─ Appointment Service
     ├─ Application Orchestrator (BPMN/Rules)
     ├─ Notifications (SMS/email/push webhooks)
     └─ Application Status API
```

* * *

3) Packaging & platforms
------------------------

*   **Android**: AAR (min SDK 23+). Uses **Custom Tabs** by default; falls back to secure WebView with certificate pinning and SafeBrowsing enabled.
*   **iOS**: XCFramework (iOS 14+). Uses **SFSafariViewController** or **ASWebAuthenticationSession**; falls back to WKWebView with pinning.
*   **Web (optional)**: A lightweight JS wrapper to manage the same deep‑link & event semantics when the partner integrates via a web container.
*   **Binary distribution**: Maven Central / CocoaPods + private artifact repo for partners.
*   **Obfuscation**: R8/ProGuard (Android), LLVM obfuscation (iOS) for sensitive logic.

* * *

4) Security & compliance
------------------------

*   **Identity & Auth**
    *   OAuth 2.1 Authorization Code Flow with **PKCE** (public mobile clients).
    *   **OIDC** for standardized ID tokens; **nonce** + **state** validated.
    *   **App‑to‑Web SSO** using system browser sessions where possible; else token binding inside SDK.
*   **Transport & Channel**
    *   TLS 1.2+ with **certificate pinning** (leaf + intermediate).
    *   **mTLS** for **partner‑backend → HSBC** server‑to‑server APIs (Prefill, Webhook registration).
*   **Data protection**
    *   **Prefill Token** pattern: partner never injects raw PII into the app. Partner backend posts PII → HSBC Prefill API → returns **single‑use, short‑TTL token** to the partner; the token is passed to the SDK, which exchanges it server‑side for prefill data. (No PII leaks on device.)
    *   On‑device secrets (refresh tokens when applicable) stored in **Android Keystore**/**iOS Keychain** (hardware backed).
    *   All events limited to **least‑privilege, privacy‑safe** payloads; PII redaction by default.
*   **Device integrity**
    *   Root/jailbreak detection; Play Integrity API / App Attest.
*   **Crypto**
    *   JWT/JWS for request non‑repudiation; JWE for PII payloads. Bank‑approved curves and ciphers only.
*   **Regulatory**
    *   GDPR/PDPA/China CSL/PII residency controls via **regional endpoints**.
    *   **Consent capture**: SDK requires partner to present HSBC terms and obtain explicit consent; **consent artifacts** posted to HSBC.
*   **Threat modeling**
    *   STRIDE coverage: replay (nonce + exp), MITM (pinning), injection (strict CSP on AO Web), clickjacking (XFO/CSP), code tamper (attestation), token theft (PKCE + binding).

* * *

5) Integration patterns (2 options)
-----------------------------------

### A) **Hosted AO (recommended, fastest)**

*   UI lives on HSBC AO Web; SDK provides secure container, identity, events, and native KYC hooks.
*   Pros: rapid rollout, consistent compliance, flexible content updates.
*   Cons: limited partner theming beyond supported hooks.

### B) **Native AO (deeper embed)**

*   Forms built natively; SDK orchestrates APIs (eligibility, KYC, docs, submit).
*   Pros: best UX control.
*   Cons: higher change surface; needs strict versioning and frequent updates when HSBC forms/rules change.

> Start with **A**, offer selected **native plugins** to reduce friction (camera, NFC for ePassports, liveness).

* * *

6) Public SDK interface (proposed)
----------------------------------

### Common types

```kotlin
// Kotlin (Android)
data class HSBCInitConfig(
  val environment: Environment,       // SANDBOX | PRODUCTION
  val partnerId: String,              // issued by HSBC
  val clientId: String,               // OIDC client id
  val redirectScheme: String,         // e.g., ctrip://hsbc/callback
  val locale: Locale = Locale.SIMPLIFIED_CHINESE,
  val theme: HSBCTheme? = null,       // theming hooks
  val attestation: AttestationConfig? = null
)

data class AOStartParams(
  val prefillToken: String,           // obtained server-to-server
  val campaignId: String? = null,
  val productCode: String,            // e.g., "HSBC_CN_RP_CURR"
  val trackingContext: Map<String,String> = emptyMap()
)

sealed class AOResult {
  data class Completed(val applicationId: String): AOResult()
  data class Pending(val applicationId: String): AOResult()
  object Cancelled: AOResult()
  data class Failed(val error: AOError): AOResult()
}

data class AOError(
  val code: String,                   // see error codes
  val message: String,
  val recoverable: Boolean
)

interface AOEventListener {
  fun onEvent(name: String, attributes: Map<String,Any>)
}
```

### Android usage

```kotlin
HSBCAOSdk.initialize(
  context = applicationContext,
  config = HSBCInitConfig(
    environment = Environment.SANDBOX,
    partnerId = "ctrip_cn",
    clientId = "ctrip-hsbc-ao-mobile",
    redirectScheme = "ctrip"
  )
)

HSBCAOSdk.setEventListener(object: AOEventListener {
  override fun onEvent(name: String, attributes: Map<String, Any>) {
    // e.g., AO_BEGIN, KYC_STARTED, DOC_UPLOADED, AO_SUBMITTED
    analytics.log(name, attributes)
  }
})

lifecycleScope.launch {
  val result = HSBCAOSdk.startAccountOpening(
    activity = this@CtripActivity,
    AOStartParams(
      prefillToken = prefillTokenFromServer,
      productCode = "HSBC_CN_RP_CURR",
      campaignId = "ctrip_oct_campaign"
    )
  )
  when (result) {
    is AOResult.Completed -> showSuccess(result.applicationId)
    is AOResult.Pending   -> showPending(result.applicationId)
    is AOResult.Cancelled -> showCancelled()
    is AOResult.Failed    -> showError(result.error)
  }
}
```

### iOS usage (Swift)

```swift
let config = HSBCInitConfig(
  environment: .sandbox,
  partnerId: "ctrip_cn",
  clientId: "ctrip-hsbc-ao-ios",
  redirectScheme: "ctrip"
)
HSBCAOSdk.initialize(config: config)

HSBCAOSdk.setEventListener { name, attributes in
  Analytics.logEvent(name, parameters: attributes)
}

HSBCAOSdk.startAccountOpening(
  from: self,
  params: AOStartParams(
    prefillToken: prefillTokenFromServer,
    campaignId: "ctrip_oct_campaign",
    productCode: "HSBC_CN_RP_CURR"
  )
) { result in
  switch result {
  case .completed(let appId): self.showSuccess(appId)
  case .pending(let appId):   self.showPending(appId)
  case .cancelled:            self.showCancelled()
  case .failed(let err):      self.showError(err)
  }
}
```

### Deep link handler

```kotlin
// AndroidManifest: <data android:scheme="ctrip" android:host="hsbc" android:pathPrefix="/callback" />
override fun onNewIntent(intent: Intent) {
  if (HSBCAOSdk.canHandle(intent.data)) {
    HSBCAOSdk.handleDeepLink(intent.data!!)
  }
}
```

* * *

7) Data contract (prefill & consent)
------------------------------------

**Prefill request** (partner → HSBC, server‑to‑server)

```
POST /partner/v1/prefill
Headers: mTLS, Authorization: Bearer <partner_access_token>
Body (JWE):
{
  "partnerId": "ctrip_cn",
  "consent": {
    "type": "HSBC_ACCOUNT_OPENING",
    "version": "2025-01",
    "timestamp": "2025-10-22T06:43:00Z",
    "captureChannel": "ctrip_app",
    "evidenceId": "consent-uuid-123"
  },
  "applicant": {
    "fullName": { "given":"Wei", "family":"Zhang" },
    "dob": "1992-08-12",
    "nationality": "CN",
    "idDocument": { "type":"CN_ID", "number":"...", "expiry":"2034-08-12" },
    "mobile": "+86-13....",
    "email": "user@example.com",
    "address": { "line1":"...", "city":"...", "province":"...", "postalCode":"..." }
  },
  "employment": { "status":"EMPLOYED", "employer":"..." },
  "pep": false,
  "tax": [{ "country":"CN", "tin":"..." }],
  "product": "HSBC_CN_RP_CURR",
  "marketingOptIn": false,
  "trackingContext": { "partnerUserId":"..." }
}
```

**Response**

```
200 OK
{
  "prefillToken": "ptok_eyJhbGciOi... (JWS)",
  "expiresIn": 900
}
```

> The **prefillToken** is **single‑use** and **short‑lived** (e.g., 15 minutes). No PII is returned to the partner mobile app.

**Required/optional fields**

*   **Required**: full name, dob, nationality, idDocument.{type,number}, mobile, product, consent.
*   **Optional**: email, address, employment, tax, marketingOptIn.
*   **Validation**: ISO 3166 for countries, E.164 for phone, RFC 5322 for email (relaxed), Unicode NFC normalization for names.

* * *

8) HSBC partner‑facing APIs (server side)
-----------------------------------------

> All server APIs use **mTLS**, JWT client assertion, and rate limiting. Request payloads that contain PII are **JWE** encrypted with HSBC public key.

### 8.1 Authorization (OAuth 2.1 for partners)

*   `POST /oauth2/token` — partner credentials grant → **partner\_access\_token** to call prefill/status/webhook registration.
*   `POST /oauth2/introspect` — token introspection (optional).

### 8.2 Prefill

*   `POST /partner/v1/prefill` → returns `prefillToken` (above).

### 8.3 Application status

*   `GET /partner/v1/applications/{applicationId}/status`
    *   Response: `{ "status": "PENDING|APPROVED|REJECTED|MORE_INFO", "updatedAt": "...", "nextActions":[...] }`

### 8.4 Webhook registration (optional)

*   `POST /partner/v1/webhooks` — register event destinations (application.updated, kyc.completed).
*   HSBC signs events with **HSBK‑Sig** header (Ed25519); partner verifies signature.

* * *

9) In‑app AO Web endpoints (SDK‑handled, not partner)
-----------------------------------------------------

*   `GET /ao/start?prefillToken=...&locale=...&product=...&campaignId=...`
*   OIDC login occurs via system browser/ASWebAuthSession; AO session created.
*   AO web pulls prefilled data by redeeming `prefillToken` (server‑to‑server).
*   **Native plugin hooks** exposed via `window.hsbcBridge.postMessage(...)` for:
    *   `captureDocument({type:"ID_FRONT"})`
    *   `performLiveness()`
    *   `scanNFC({docType:"passport"})`
    *   `openAppointmentScheduler({branchId?...})`
*   Bridge responses are signed JSON objects (JWS) the AO web posts back to HSBC services.

* * *

10) KYC plugins (optional)
--------------------------

*   **Document capture**
    *   Edge detection, glare control, motion guidance, MRZ read (if passport), on‑device quality score, encrypted upload.
*   **Liveness**
    *   Passive liveness with on‑device pre‑check; server inference.
*   **NFC**
    *   ICAO 9303 passport chip read (if present).
*   All plugins can be **disabled** at runtime by HSBC policy (feature flags) or the partner’s jurisdiction.

* * *

11) Branch appointment module
-----------------------------

*   `POST /ao/appointment/availability`
*   `POST /ao/appointment/book`
*   SDK bridge opens a native date/time selector when requested; returns a signed booking object (`appointmentId`, `slot`, `branch`, `qr` if needed).

* * *

12) Events & analytics
----------------------

**SDK → Partner events** (privacy‑safe):

| Event name | When | Attributes |
| --- | --- | --- |
| `AO_BEGIN` | AO UI opened | `product`, `campaignId` |
| `AO_FORM_PREFILLED` | Prefill redeemed | `prefillSource: "partner"`, `fieldsPrefilledCount` |
| `KYC_STARTED` | First KYC step launched | \`method: DOC |
| `KYC_COMPLETED` | All KYC steps done | \`result: PASS |
| `APPOINTMENT_BOOKED` | Branch slot reserved | `branchId`, `slotStart` |
| `AO_SUBMITTED` | Application submitted | `applicationId`, `submitLatencyMs` |
| `AO_EXITED` | User exits | \`reason: CANCEL |
| `AO_ERROR` | Error occurred | `code`, `message` (sanitized), \`recoverable: true |

> All events are **redacted** (no PII) and contain a **correlationId** so HSBC can reconcile logs.

* * *

13) Error model
---------------

**Top‑level error codes (examples)**

*   `NET_TIMEOUT`, `NET_UNREACHABLE`
*   `AUTH_FAILED`, `AUTH_CANCELLED`, `AUTH_EXPIRED`
*   `PREFILL_INVALID`, `PREFILL_EXPIRED`, `PREFILL_REDEEMED`
*   `KYC_CAMERA_DENIED`, `KYC_QUALITY_LOW`, `KYC_SERVER_ERROR`
*   `AO_UNAVAILABLE`, `AO_POLICY_BLOCKED` (jurisdiction/product)
*   `UNKNOWN`

**SDK strategy**

*   Retry with exponential backoff where safe (idempotent GETs, token refresh).
*   Offer **graceful recovery UI** and deep‑link to resume.
*   All failures surfaced via `AOResult.Failed` and `AO_ERROR` event.

* * *

14) Theming, i18n, accessibility
--------------------------------

*   **Theme hooks**: toolbar color, title text, close/cancel labels; HSBC form styling remains consistent for compliance/brand.
*   **Locales**: `zh-CN`, `en-HK`, `en-GB`, etc. Locale passed to AO Web; fallback to device language.
*   **Accessibility**: WCAG 2.1 AA; dynamic type; screen reader labels; color contrast; reduced motion support.

* * *

15) Versioning & compatibility
------------------------------

*   **SemVer** for SDK (e.g., `1.3.0`).
*   **Backward compatible** public API for minor releases.
*   **Deprecation policy**: N−1 major version supported for ≥ 12 months.
*   **Runtime feature flags** from HSBC (kill switches, A/B toggles) via signed config fetched at start.

* * *

16) Observability & support
---------------------------

*   **HSBC telemetry** (opt‑in, privacy‑safe): perf metrics (TTFB for AO web, KYC step durations), error rates, device profile (OS only).
*   **Partner runbook**: FAQ codes, known issues, escalation paths.
*   **Crash reporting**: breadcrumbs without PII.

* * *

17) Onboarding steps for Ctrip (partner)
----------------------------------------

1.  **Registration**
    *   Exchange legal agreements, scopes, and data minimization matrix.
    *   HSBC issues `partnerId`, OIDC `clientId`, and mTLS certs for partner backend.
2.  **Environments**
    *   Sandbox endpoints and test data; later production endpoints.
3.  **Prefill integration**
    *   Implement `/partner/v1/prefill` server‑to‑server with consent evidence.
    *   Store only **prefillToken**, not raw PII in Ctrip systems beyond what they already hold under their privacy policy.
4.  **SDK integration**
    *   Add dependency (AAR/XCFramework), set redirect scheme, wire deep link.
    *   Implement event listener and result handling.
5.  **KYC plugin choice**
    *   Decide hosted vs native; if native, add doc scan dependency and camera permissions.
6.  **Brand checks**
    *   Apply allowed theming; run through HSBC UX acceptance checklist.
7.  **Security tests**
    *   Pen test on partner build; root/jailbreak checks; attestation working.
8.  **Go‑live checklist**
    *   Webhook (optional), status API consumption, monitoring dashboards, rollback plan.

* * *

18) Example AO sequence (text sequence diagram)
-----------------------------------------------

```
Ctrip App        Ctrip Backend       HSBC Partner GW             HSBC AO Web & Services
    |                  |                    |                                    |
(1) User taps "Open HSBC Account"                                               |
    |                  |                    |                                    |
(2) Consent for data use ->                                                     |
    |---- POST /prefill (PII+consent, mTLS) --->|                               |
    |                  |<--- prefillToken ------|                                |
    |                                                                            |
(3) HSBCAOSdk.start(prefillToken, product)                                       |
    |---------------- Open AO Web --------------------------->|                  |
    |                                                        |-- redeem prefillToken --> [Prefill Store]
    |                                                        |<-- prefilled fields ------|
    |<---- OIDC auth (ASWebAuth/Custom Tab) ----->|                                     |
    |                                                        |-- KYC (plugins/hosted) -->|
    |                                                        |-- Appointment (optional) ->|
    |                                                        |-- Submit Application ------> [Orchestrator]
    |                                                        |<-- applicationId & state ---|
    |<---------------- AOResult.Completed(applicationId) ------------------------------- |
```

* * *

19) Non‑functional requirements
-------------------------------

*   **SLA**: 99.9% AO Web availability; 99.95% Partner GW; P95 page interactive < 3.0s on 4G.
*   **Rate limits**: per partner throttle with burst credits; clear `429` semantics and `Retry-After`.
*   **Resilience**: multi‑region active/active, circuit breakers, grace mode (queue KYC uploads offline with encryption).
*   **Size**: SDK ≤ 3.5 MB (w/o native KYC); native KYC plugin ≤ 15 MB (models included).
*   **Privacy**: data minimization; expunge on device after completion; logs redacted; configurable retention.

* * *

20) Sample sandbox configuration (to give engineers a running start)
--------------------------------------------------------------------

*   **Environments**
    *   SANDBOX Base URL: `https://sandbox.partner.hsbc.com`
    *   OIDC Issuer: `https://id.sandbox.partner.hsbc.com`
*   **Scopes**: `openid ao.start ao.status`
*   **Redirect URIs**: `ctrip://hsbc/callback`
*   **Products**: `HSBC_CN_RP_CURR` (RMB current account), `HSBC_HK_MCA` (HK multi‑currency)
*   **Test personas**
    *   `Zhang Wei` (eligible), `Chen Li` (KYC refer), `Liu Min` (sanctions hit → reject)

* * *

21) Deliverables checklist
--------------------------

*   ✅ **SDK binaries** (Android/iOS), release notes, SemVer tags
*   ✅ **Integration guide** (this document’s content in developer portal format)
*   ✅ **Code samples** (Android + iOS demo apps)
*   ✅ **Partner server sample** (Node/Java/Kotlin snippets for Prefill & Webhooks)
*   ✅ **Postman collection** for Partner APIs
*   ✅ **Threat model** + pen‑test scope & test cases
*   ✅ **Compliance pack**: consent text, privacy notices, data flows, DPIA template

* * *

22) Build plan (phased)
-----------------------

**Phase 1 – Hosted AO with prefill token**  
Deliver core SDK, OIDC, pinning, event model, AO Web, status API.

**Phase 2 – Native KYC plugins & appointments**  
Add doc capture, liveness, NFC read; appointment adapter.

**Phase 3 – Enhancements**  
Save/resume cards in partner app, richer analytics, A/B testing hooks, and federated SSO with partner IDP if desired.

* * *

### Appendix A — Minimal partner server code sketch (Prefill) _(illustrative)_

```pseudo
POST /hsbc/prefill
  assert user_consent == true
  payload = {
    partnerId, consent, applicant, product, trackingContext
  }
  jwe = JWE.encrypt(payload, hsbcPublicKey)
  resp = httpsClient.mtls()
        .post("https://sandbox.partner.hsbc.com/partner/v1/prefill",
              headers={Authorization: "Bearer " + partnerAccessToken},
              body=jwe)
  return resp.prefillToken
```

* * *

If you want, I can also provide:

*   a ready‑to‑paste **Android Gradle** dependency block and **iOS CocoaPods/SwiftPM** snippets,
*   a **Postman collection JSON** for the Partner APIs,
*   a **consent text** template localized for zh‑CN and en‑HK.

But the design above is complete enough for HSBC and Ctrip engineering teams to begin implementation immediately.



---
Powered by [ChatGPT Exporter](https://www.chatgptexporter.com)