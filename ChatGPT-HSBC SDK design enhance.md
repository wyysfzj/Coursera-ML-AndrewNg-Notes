# HSBC SDK design

**User:** Anonymous (os86@naifeistation.com)  
**Created:** 10/25/2025 21:56:01  
**Updated:** 10/27/2025 18:50:58  
**Exported:** 10/28/2025 0:58:00  
**Link:** [https://chatgpt.com/c/68fcd6f0-34a0-8329-9d83-3647aa1aac1d](https://chatgpt.com/c/68fcd6f0-34a0-8329-9d83-3647aa1aac1d)  

## Prompt:
![Uploaded image](https://chatgpt.com/backend-api/estuary/content?id=file_00000000405861f5a6a7dc37f82557a3&ts=489328&p=fs&cid=1&sig=8fcb4639ebe075b70bbc66cd0eb251cd8e02a6d680277dfa292ab596c76f7100&v=0)

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

## Response:
please explain detail of 3) packaging & platforms

## Prompt:
Great—here’s a deeper, implementation-level take on **3) Packaging & platforms** for the HSBC Partner Account-Opening SDK.

* * *

3) Packaging & platforms (detailed)
===================================

3.1 Android
-----------

### Artifact & min/target

*   **Artifact**: single `AAR` published to **Maven Central** and an **HSBC private repo**.
*   **Module split** (to keep size low):
    *   `com.hsbc.aosdk:core` (mandatory)
    *   `com.hsbc.aosdk:kyc-doc`, `…:kyc-liveness`, `…:kyc-nfc` (optional plugins)
*   **minSdk**: 23 (Android 6.0)  
    **targetSdk**: latest stable (keep within N-1)  
    **compileSdk**: latest stable

### Dependencies (kept thin, shaded where needed)

*   AndroidX (`browser` for Custom Tabs, `appcompat`, `lifecycle`).
*   `okhttp` + `okio` (network, cert pinning), `moshi`/`kotlinx.serialization`.
*   `jose4j` or Nimbus JOSE+JWT for JWS/JWE (shaded).
*   **No transitive UX frameworks**; all UI is within a lightweight Activity + WebView fallback.

### Distribution & import

**Gradle**

```gradle
repositories { mavenCentral(); maven { url "https://maven.hsbc.com/partner" } }
dependencies {
  implementation "com.hsbc.aosdk:core:1.0.0"
  implementation "com.hsbc.aosdk:kyc-doc:1.0.0"      // optional
}
```

### Manifest & config

*   **Intent-filter** for deep link:

```xml
<intent-filter>
  <action android:name="android.intent.action.VIEW"/>
  <category android:name="android.intent.category.DEFAULT"/>
  <category android:name="android.intent.category.BROWSABLE"/>
  <data android:scheme="ctrip" android:host="hsbc" android:pathPrefix="/callback"/>
</intent-filter>
```

*   **Network Security Config**: block user-added CAs; enable **pinning** (pin leaf + intermediate).
*   **Play Integrity**: `com.google.android.play.core.integrity` with “basicIntegrity + deviceIntegrity” checks.
*   **ProGuard/R8**: keep the SDK’s public API and model classes; obfuscate internals (mapping file provided to HSBC).

### Runtime behavior

*   **Primary container**: **Custom Tabs** (Chrome, or default browser).
*   **Fallback**: hardened **WebView** with SafeBrowsing, JS bridge isolated, CSP ensurelist.
*   **Token storage**: Android **Keystore** (StrongBox if available).
*   **Process death**: save session state (encrypted `SharedPreferences`) and **resume** via deep link.

### Size, performance, compatibility

*   **Core AAR** ≤ 3.5 MB; each KYC plugin packaged separately (≤ 15 MB combined, ABI-split via Gradle).
*   **ABI splits** enabled for native KYC libraries.
*   **Strict no-multidex requirement** for the SDK itself; compatible with multidex apps.

### App store declarations

*   **Data Safety**: template JSON provided (no PII collected by SDK; only privacy-safe telemetry if partner opts in).
*   **Permissions**:
    *   Core: none beyond Internet.
    *   KYC plugins: `CAMERA`, optional `NFC`, and `WRITE_EXTERNAL_STORAGE` **not used** (use MediaStore/SAF only).

* * *

3.2 iOS
-------

### Binary format & deployment target

*   **XCFramework** (fat, with slices for device + simulator), distributed via **CocoaPods** and **Swift Package Manager**.
*   **iOS** 14.0+ (ASWebAuthenticationSession availability), **Xcode** N-1 minimum.

### Module split

*   `HSBCAOSDKCore.xcframework` (mandatory)
*   `HSBCAOSDKKycDoc.xcframework`, `…KycLiveness.xcframework`, `…KycNfc.xcframework` (optional)

### Distribution & import

**SPM**

```swift
.package(url: "https://github.com/hsbc/hsbc-aosdk-ios.git", from: "1.0.0")
```

**CocoaPods**

```ruby
pod 'HSBCAOSDKCore', '1.0.0'
pod 'HSBCAOSDKKycDoc', '1.0.0'  # optional
```

### App capabilities & entitlements

*   **Associated Domains** (if universal links are used in addition to custom scheme).
*   **Keychain sharing** _not_ required (SDK uses its own default access group).
*   **App Attest** / DeviceCheck integration optional but recommended.

### Runtime behavior

*   **Primary container**: `ASWebAuthenticationSession` (best OIDC UX).
*   **Alternatives**: `SFSafariViewController`; hardened `WKWebView` fallback with pinning.
*   **Token storage**: **Keychain** (kSecAttrAccessibleAfterFirstUnlockThisDeviceOnly).
*   **Privacy manifests** (iOS 17+): shipped with the framework.
*   **ATS**: strict; allow only HSBC endpoints with TLS 1.2+; no arbitrary loads.

### App Store privacy

*   **Data types**: SDK telemetry is optional; no tracking; no linking to user identity.

* * *

3.3 Web (optional wrapper)
--------------------------

### When to use

*   Partners embedding AO in a Web app or in a **WebView**\-based superapp where a JS bridge is easier.

### Package

*   **NPM**: `@hsbc/aosdk-web`
*   **ESM + UMD** builds; tree-shakable; ≤ 25 KB gzipped.

### API (mirrors mobile)

```ts
import { init, startAccountOpening, onEvent } from '@hsbc/aosdk-web';

init({ environment: 'SANDBOX', partnerId, clientId, redirectUri, locale });
onEvent((name, attrs) => analytics.log(name, attrs));

const res = await startAccountOpening({ prefillToken, productCode, campaignId });
```

### Security

*   **CSP** sample provided (script-src self; connect-src HSBC; frame-ancestors none).
*   **postMessage** channel with origin checks if AO is launched in an iframe; otherwise top-level redirect + return via `redirectUri`.

* * *

3.4 Versioning, distribution & rollout
--------------------------------------

*   **SemVer**: `MAJOR.MINOR.PATCH`
    *   Minor = additive features; Patch = bug/security fixes.
*   **Release channels**
    *   **Stable**: default (`1.x`).
    *   **Preview**: `1.x-rcN` for partners willing to test next release.
*   **Delivery**
    *   Android: Maven Central + HSBC private artifactory.
    *   iOS: CocoaPods trunk + SPM Git tag releases.
    *   Web: npm + GitHub Releases.
*   **Upgrade policy**
    *   Maintain **binary compatibility** for a whole MINOR line.
    *   Deprecations announced ≥ 90 days ahead; N-1 major supported ≥ 12 months.

* * *

3.5 Configuration & theming
---------------------------

*   **Environment** baked at runtime via `HSBCInitConfig(environment=SANDBOX|PRODUCTION)`.
*   **Remote feature flags** (signed JSON) pulled on init, cached with TTL 1 day.
*   **Theming hooks** (safe area: toolbar color/title/close labels/font scale). HSBC Web forms keep canonical styling for compliance.

* * *

3.6 Hardening & integrity (both platforms)
------------------------------------------

*   **Cert pinning**: multiple SPKI pins (current + next), fail-closed with opt-in “grace window” for region outages.
*   **Jailbreak/Root detection** with _signal-based_ checks; policy decides deny/warn/allow.
*   **Replay protection**: PKCE for OIDC, nonce/state, JWS on bridge messages.
*   **Crash & logs**: no PII; redaction by default; partner can disable telemetry entirely.

* * *

3.7 Testing & samples
---------------------

*   **Sample apps**: Kotlin + Swift demo with:
    *   Deep link wiring, event listener, result handling, background/resume.
    *   Toggle between **Custom Tab/ASWebAuth** and **WebView/WKWebView** fallback.
*   **Device matrix**:
    *   Android 8, 10, 12, 14 across mid/low-tier HW; Chrome present/absent.
    *   iOS 14, 15, 16, 17; VoiceOver; Large Text; Right-to-Left sanity check.
*   **CI/CD**:
    *   Build XCFramework + AAR with **reproducible builds**; sign with HSBC key.
    *   SBOM (CycloneDX) artifacts attached to every release.

* * *

3.8 Legal & licensing
---------------------

*   **License**: partner EULA; redistribution limited to approved apps.
*   **3rd-party notices**: bundled `THIRD_PARTY_NOTICES.*` & OSS attributions tool-generated.
*   **Export controls**: cryptography declarations prepared (US/EU/SG/HK/CN as applicable).

* * *

### Quick start snippets

**Android init**

```kotlin
HSBCAOSdk.initialize(
  applicationContext,
  HSBCInitConfig(
    environment = Environment.SANDBOX,
    partnerId = "ctrip_cn",
    clientId = "ctrip-hsbc-ao-mobile",
    redirectScheme = "ctrip",
    locale = Locale.SIMPLIFIED_CHINESE
  )
)
```

**iOS init**

```swift
let cfg = HSBCInitConfig(
  environment: .sandbox,
  partnerId: "ctrip_cn",
  clientId: "ctrip-hsbc-ao-ios",
  redirectScheme: "ctrip",
  locale: .init(identifier: "zh-Hans-CN")
)
HSBCAOSdk.initialize(config: cfg)
```

* * *

If you’d like, I can generate:

*   a **starter Android sample app** `zip` and an **iOS sample workspace** with all the above wired,
*   a **checklist** your partner team can use to certify each integration before UAT.

## Response:
give me a detail word description to explain above because I am not faimlier with mobile package

## Prompt:
Absolutely — let’s go step-by-step and explain the **“Packaging & Platforms”** part in plain words, so you can understand what each term means and how it connects to the HSBC SDK you’re designing.

* * *

🧩 What “Packaging & Platforms” Means
-------------------------------------

When a bank (like HSBC) provides an SDK for partners (like Ctrip), developers need to **install it into their mobile apps**.  
“Packaging & Platforms” describes **how the SDK is built, delivered, and runs** on different types of devices (Android, iOS, Web).

Think of it as:

> How the SDK is wrapped, distributed, installed, and kept secure on different app platforms.

* * *

📱 Android (for phones running Google’s OS)
-------------------------------------------

### 1️⃣ Packaging format — “AAR”

*   The SDK is delivered as an **AAR file** (“Android Archive”).
*   It’s like a ZIP that contains code, images, and configuration.
*   Developers don’t download it manually — they add a line in their project’s configuration (called **Gradle**) and the SDK is pulled automatically from a repository (like “Maven Central”).

### 2️⃣ Components

To keep it modular, we divide it:

*   `core`: the essential SDK (handles authentication, API calls, showing HSBC’s web page)
*   `kyc-doc`, `kyc-liveness`, `kyc-nfc`: optional add-ons for identity verification (e.g. taking ID photos, liveness check, NFC passport reading)

### 3️⃣ System requirements

*   Works on Android 6.0 (2015) or later (most phones).
*   Uses the latest Android development tools.
*   Can run on phones and tablets.

### 4️⃣ How developers add it

In plain English, developers just add:

```gradle
implementation "com.hsbc.aosdk:core:1.0.0"
```

to their project.  
That’s how apps like Ctrip will include the SDK.

### 5️⃣ How it runs

*   When the Ctrip app launches HSBC’s account-opening flow, the SDK opens a secure **browser view inside the app**.
*   It uses Google Chrome if possible (“Custom Tabs”), or a secure built-in WebView if Chrome isn’t available.
*   All communication uses encrypted HTTPS with **certificate pinning** (only trusts HSBC’s servers).
*   Sensitive info like tokens are stored securely in the phone’s **Keystore**, which is hardware-protected.

### 6️⃣ Security extras

*   Detects if the phone is rooted or compromised.
*   Encrypts everything it stores.
*   Keeps SDK code small and optimized.
*   No user data is leaked — even if the phone has other apps installed.

* * *

🍎 iOS (for iPhone and iPad)
----------------------------

### 1️⃣ Packaging format — “XCFramework”

*   This is Apple’s equivalent of Android’s AAR.
*   It includes compiled code that can run on iPhones, iPads, or even the iOS simulator (for testing).

### 2️⃣ Installation options

Developers can install it using:

*   **Swift Package Manager (SPM)** (Apple’s built-in package system), or
*   **CocoaPods** (the older but popular one).

Example for Swift Package Manager:

```swift
.package(url: "https://github.com/hsbc/hsbc-aosdk-ios.git", from: "1.0.0")
```

### 3️⃣ System requirements

*   iOS 14 or later.
*   Xcode (Apple’s developer tool) up to the latest or one version back.

### 4️⃣ How it works inside the app

*   The SDK opens HSBC’s secure account-opening webpage using `ASWebAuthenticationSession` (Apple’s secure browser container).
*   Uses **Keychain** (hardware-secure storage) for saving temporary data.
*   Uses **App Attest** (Apple’s integrity check) to confirm the app isn’t tampered with.

### 5️⃣ iOS-specific security

*   Enforces **App Transport Security (ATS)** — only HTTPS connections allowed.
*   Verifies the HSBC domain certificate.
*   Optional “App Attest” feature ensures the SDK is running in a legitimate app build.

* * *

💻 Web (optional SDK version)
-----------------------------

If a partner (like a fintech website or a WeChat Mini App) wants to embed the same journey on the web, HSBC provides a **JavaScript package** via NPM (`@hsbc/aosdk-web`).

### How it works:

*   The partner’s web app loads the HSBC SDK.
*   Calls a simple function like:
    ```js
    startAccountOpening({ prefillToken, productCode })
    ```
*   It opens HSBC’s web journey inside an iframe or a new browser tab.
*   Uses secure message passing (`postMessage`) between pages.
*   No personal data stored in the browser.

* * *

⚙️ Versioning and Distribution
------------------------------

### Semantic Versioning (“SemVer”)

*   Version numbers look like **1.2.3**
    *   **1** = major version (breaking change)
    *   **2** = minor version (new features, no break)
    *   **3** = patch version (bug fix)

### Distribution Channels

*   Android: via Maven Central or HSBC’s private repo.
*   iOS: via SwiftPM or CocoaPods.
*   Web: via NPM registry.

### Release Policy

*   Minor updates don’t break existing apps.
*   HSBC supports the previous major version for at least 12 months.
*   Pre-release versions (`1.1.0-rc1`) allow partners to test upcoming changes.

* * *

🎨 Configuration & Theming
--------------------------

The SDK lets the partner control a few visual aspects:

*   Button color, toolbar text, or language (e.g. English, Chinese).
*   HSBC’s forms remain branded (for compliance), but Ctrip can make the surrounding experience match its app style.

* * *

🔐 Hardening and Integrity Across Platforms
-------------------------------------------

For **Android and iOS**, the SDK includes:

*   **Certificate pinning** — only talks to HSBC’s servers.
*   **Integrity checks** — detects jailbreak/root.
*   **Replay protection** — prevents someone from reusing old session tokens.
*   **Encrypted logs** — for troubleshooting but without personal data.

* * *

🧪 Testing & Developer Support
------------------------------

HSBC provides:

*   **Demo apps** (Android + iOS) showing integration.
*   Test users for sandbox mode (e.g., one that always passes KYC, one that always fails).
*   Continuous Integration (CI) builds for each platform with reproducible binaries.

* * *

📦 In simple terms
------------------

| Platform | Package Type | Where Developers Get It | Secure Container | Secure Storage | Optional Add-Ons |
| --- | --- | --- | --- | --- | --- |
| **Android** | `.AAR` | Maven Central | Chrome Custom Tab / WebView | Android Keystore | KYC camera, NFC |
| **iOS** | `.XCFramework` | SwiftPM / CocoaPods | ASWebAuthSession / SafariView | Keychain | KYC camera, NFC |
| **Web** | `.JS (NPM)` | npm registry | Browser / iframe | Local memory only | None |

* * *

If you’d like, I can also:

*   Draw a **diagram** showing how each SDK type fits between HSBC and partner apps.
*   Write a **shorter summary for management** (non-technical, business-level).

Would you like me to generate that next?

## Response:
generate both

## Prompt:
Perfect 👍 Below are **both outputs** you requested:

* * *

🎨 **1️⃣ Diagram – SDK Packaging & Platforms Overview**
=======================================================

Here’s a **clear visual architecture diagram** that you can import into Draw.io, Lucidchart, or Miro.  
It shows how the SDKs fit between partner apps (like Ctrip) and HSBC systems.

```mermaid
graph TD
    A[Partner App (e.g. Ctrip)] -->|Integrates| B[HSBC AO SDK]
    B --> B1[Android SDK (.AAR)\n- Custom Tabs / WebView\n- Secure Keystore]
    B --> B2[iOS SDK (.XCFramework)\n- ASWebAuthSession / SafariView\n- Keychain Storage]
    B --> B3[Web SDK (.JS via NPM)\n- Secure Iframe / Browser]
    B1 -->|API Calls & Events| C[HSBC Partner Gateway]
    B2 -->|OAuth + HTTPS (PKCE, Pinning)| C
    B3 -->|postMessage + HTTPS| C
    C --> D[HSBC AO Web + Services]
    D --> D1[KYC / AML / Screening]
    D --> D2[Application Orchestrator]
    D --> D3[Branch Appointment System]
    D --> D4[Status & Notification APIs]

    classDef hsbc fill:#E8F3FF,stroke:#005EB8,stroke-width:2px;
    class C,D,D1,D2,D3,D4 hsbc;
```

This diagram illustrates that:

*   Partners embed the SDK.
*   SDK handles the security, UI, and network layer.
*   HSBC backend provides all the business logic and compliance checks.

* * *

🧭 **2️⃣ Business / Management Summary (Non-Technical)**
========================================================

Objective
---------

Enable third-party partners (like Ctrip) to let users **open an HSBC account directly within their app**, securely and seamlessly — without redirecting users to external sites or physical branches.

* * *

Key Concept
-----------

HSBC provides a **Partner Account-Opening SDK (Software Development Kit)**.  
It’s a pre-built “toolbox” partners plug into their mobile or web app.

Think of it as:

> “HSBC’s secure account-opening engine, wrapped and ready for easy integration.”

* * *

Platforms Covered
-----------------

| Platform | What HSBC Provides | Purpose |
| --- | --- | --- |
| **Android SDK (.AAR)** | For apps built on Android phones | Partners like Ctrip include it to start HSBC account-opening flow securely within their app |
| **iOS SDK (.XCFramework)** | For iPhone and iPad apps | Works via Apple’s secure browser environment |
| **Web SDK (.JS via NPM)** | For websites or mini-programs | Allows HSBC’s account opening to appear inside a partner’s web page |

* * *

Business Benefits
-----------------

✅ **Frictionless experience** — customers open an HSBC account without leaving the partner app.  
✅ **Instant data sharing** — partner passes verified customer data securely to HSBC (no re-entry).  
✅ **Faster onboarding** — KYC and document checks happen digitally.  
✅ **Strong compliance** — all encryption, consent, and audit trails are built-in.  
✅ **Brand control** — HSBC maintains regulatory UX and branding, while partner app provides its familiar environment.

* * *

How It Works (Simplified Flow)
------------------------------

1.  User taps “Open HSBC Account” in the partner app.
2.  Partner app calls HSBC SDK → launches HSBC’s secure web flow.
3.  HSBC verifies identity, does KYC, and completes account creation.
4.  SDK returns success/failure result back to partner.
5.  Partner app can show “Application successful” screen or follow up via HSBC’s status API.

* * *

Integration Steps for Partner
-----------------------------

1.  **Register as HSBC Partner** → receive credentials & sandbox environment.
2.  **Integrate SDK** into their app (Android/iOS/Web).
3.  **Implement Prefill API** → send user consent and info securely to HSBC.
4.  **Test in sandbox** using HSBC’s sample users.
5.  **Go live** with full monitoring and HSBC certification.

* * *

Governance and Security Highlights
----------------------------------

*   All traffic uses **TLS 1.2+ with certificate pinning**.
*   Sensitive data is encrypted with HSBC keys (never stored on device).
*   SDK detects unsafe devices (rooted/jailbroken).
*   Partners get **prefill tokens**, not raw customer data (ensuring privacy).
*   Compliance with **GDPR / PDPA / local data protection** requirements.

* * *

Business Outcome
----------------

This SDK solution:

*   **Transforms onboarding** — partners become digital acquisition channels for HSBC.
*   **Speeds up go-to-market** — reusable SDK, easy integration.
*   **Improves conversion** — customers complete the process in familiar apps.
*   **Maintains HSBC security & compliance** across all touchpoints.

* * *

If you’d like, I can:

*   Generate a **PowerPoint summary slide** (executive-ready, visual summary).
*   Or a **technical design doc section** for the “SDK Packaging & Platforms” chapter (markdown format, to append to your architecture doc).

Would you like me to produce one or both of those next?

## Response:
please also provide detail word explain for Security & compliance

## Prompt:
Excellent — let’s now go through the **Security & Compliance** section in full human-readable detail, without assuming any mobile or backend security background.  
This section explains _how and why_ the HSBC SDK keeps user data, banking systems, and partner apps safe.

* * *

🛡️ HSBC Partner SDK — Security & Compliance (Detailed Explanation)
===================================================================

* * *

1️⃣ Overall Security Objective
------------------------------

The goal of HSBC’s Partner SDK (used by apps like **Ctrip**) is to:

> Allow customers to open an HSBC account **inside a partner app** while maintaining **bank-level security** and **regulatory compliance**.

That means:

*   Data is always protected — at rest, in transit, and in processing.
*   The partner cannot see or tamper with the confidential banking flow.
*   HSBC maintains end-to-end control over authentication, consent, and KYC (Know Your Customer) validation.

* * *

2️⃣ Identity & Authentication
-----------------------------

### 🔒 OAuth 2.1 + OpenID Connect (OIDC)

The SDK uses industry standards for secure login and authorization:

*   **OAuth 2.1 Authorization Code Flow with PKCE**
    *   Ensures every session is unique and cannot be replayed.
    *   “PKCE” adds a proof key to block attackers from stealing tokens.
*   **OIDC (OpenID Connect)**
    *   Allows HSBC to identify users securely, without sharing passwords.
*   Each partner (like Ctrip) has their own **client ID** and **partner ID** issued by HSBC.  
    This ensures one partner cannot impersonate another.

### 🧩 Token-based Sessions

When Ctrip’s app launches the HSBC SDK:

*   It exchanges a **Prefill Token** (temporary, single-use key) with HSBC.
*   The SDK uses this token to fetch customer prefilled data (like name or ID) from HSBC — **not from the app**.
*   This design ensures **Ctrip never handles sensitive PII** directly.

* * *

3️⃣ Transport & Network Security
--------------------------------

### 🔐 Encrypted Communication

*   All communication between SDK and HSBC servers uses **TLS 1.2 or newer**, same as online banking.
*   This prevents eavesdropping or man-in-the-middle attacks.

### 📍 Certificate Pinning

*   The SDK “pins” HSBC’s real SSL certificate fingerprints.
    *   If someone tries to intercept with a fake HSBC site, the SDK instantly blocks the connection.
*   This makes phishing or traffic hijacking nearly impossible.

### 🔄 Mutual TLS (mTLS)

*   Partner servers (not mobile apps) use **mTLS**, meaning both client and server verify each other’s identity with digital certificates.
*   This ensures even partner-to-bank communication is strictly authenticated.

* * *

4️⃣ Data Protection & Privacy
-----------------------------

### 🧱 Data Minimization

*   The partner never sees raw PII (Personally Identifiable Information).
*   Instead, the partner’s backend sends prefilled user data (with consent) directly to HSBC — **server to server**.
*   HSBC returns a **Prefill Token** to the partner app, which the SDK uses.  
    → This token has **no personal data**, only a reference.

### 🔐 On-device Storage Security

*   The SDK stores any temporary session or token securely:
    *   **Android**: stored in the **Android Keystore**, hardware-encrypted.
    *   **iOS**: stored in the **Keychain**, Apple’s encrypted vault.
*   These keys cannot be read even if someone gains root access.

### 🚫 No Persistent Logs or Caching

*   Sensitive forms, screenshots, or debug data are **never stored** on device.
*   Any logs that exist (for troubleshooting) are **PII-free** and **rotated frequently**.

* * *

5️⃣ Device Integrity Checks
---------------------------

### 🧠 Root / Jailbreak Detection

*   The SDK automatically checks if the device is rooted or jailbroken.
*   If detected, the SDK either:
    *   Blocks the flow entirely (if policy requires), or
    *   Shows a warning and disables data-sensitive actions.

### ✅ Attestation APIs

*   On Android: uses **Play Integrity API** to ensure the app and OS are genuine.
*   On iOS: uses **App Attest / DeviceCheck** for the same reason.
*   Prevents fraud from fake or emulated devices.

* * *

6️⃣ Application Security
------------------------

### 🔏 Secure Coding Practices

*   HSBC SDK is built with **OWASP MASVS** (Mobile App Security Verification Standard).
*   Sensitive code (e.g., cryptography) is **obfuscated**, meaning it can’t be reverse-engineered.
*   HSBC performs **static & dynamic security scanning** before release.

### 🧾 Signed & Verified Binaries

*   Every SDK version is **digitally signed** by HSBC.
*   Partners cannot modify or tamper with the SDK.
*   Apps using the SDK must be re-signed with valid partner keys before going live.

* * *

7️⃣ Cryptography & Token Handling
---------------------------------

### 🔑 JWS and JWE

*   All sensitive payloads are **digitally signed (JWS)** and **encrypted (JWE)**.
    *   JWS → ensures message integrity (it hasn’t been changed).
    *   JWE → ensures confidentiality (nobody can read it).

### 🧬 Keys

*   HSBC rotates encryption keys regularly.
*   Partners use their own unique public/private key pairs for mTLS and signing.

### ⏳ Token Lifetimes

*   Short-lived (minutes) tokens to minimize exposure.
*   Tokens are bound to device and session → no replay.

* * *

8️⃣ Regulatory Compliance
-------------------------

### 🌍 Privacy Regulations

Compliant with global and local laws:

*   **GDPR (EU)** — consent, right to forget, data minimization.
*   **PDPA (Singapore / Hong Kong)** — proper handling of personal data.
*   **China Cybersecurity Law (CSL)** — data residency; Chinese customer data stays within China.

### 📋 Consent Management

*   Before data transfer, SDK ensures:
    *   The customer explicitly accepts HSBC’s Terms and Privacy Notice.
    *   Partner captures consent proof (timestamp + evidence ID).
*   HSBC stores these consent records for audit.

### 🔎 Data Residency

*   SDK calls regional HSBC endpoints (e.g., `hsbc.cn` for China, `hsbc.com.hk` for Hong Kong).
*   No cross-border data transfer unless legally permitted.

* * *

9️⃣ Threat Detection & Monitoring
---------------------------------

*   HSBC continuously monitors SDK traffic for anomalies:
    *   Suspicious IPs
    *   Repeated failed KYC attempts
    *   Abnormal session patterns
*   HSBC’s Security Operations Center (SOC) can disable compromised SDK tokens in real-time.

* * *

🔟 Compliance Audits & Testing
------------------------------

*   SDK and APIs undergo:
    *   **Penetration testing** (external security firms simulate attacks)
    *   **Source code review** for each release
    *   **Vulnerability scanning** (SAST, DAST)
    *   **Compliance audits** (PCI DSS, ISO 27001 alignment)
*   Each partner integration goes through a **“go-live certification”** that checks:
    1.  Secure network calls
    2.  Proper consent collection
    3.  No data leak via logs
    4.  Correct sandbox → production switch procedure

* * *

✅ Summary: Why This Is Secure
-----------------------------

| Layer | Protection | Description |
| --- | --- | --- |
| **Data in transit** | 🔒 TLS + Pinning + mTLS | All data encrypted and authenticated |
| **Data at rest** | 🔐 Keystore / Keychain | Encrypted hardware storage |
| **Authentication** | 🔑 OAuth 2.1 + PKCE | Prevents hijacking or reuse of tokens |
| **Integrity** | 🧩 Root/jailbreak checks + Attestation | Blocks unsafe devices |
| **Confidentiality** | 🔏 JWE encryption | Only HSBC can read payloads |
| **Auditability** | 📜 Consent + Logging | Full trace for regulators |
| **Compliance** | 🏦 GDPR, PDPA, CSL | Meets data protection laws globally |

* * *

💡 Simple Analogy
-----------------

Imagine you’re letting Ctrip “host a small HSBC branch inside their app.”

*   The **SDK** is the “secure HSBC counter” inside Ctrip — fully controlled by HSBC.
*   Ctrip provides the doorway (customer entry), but not the back-office.
*   Every document, photo, or ID the customer provides goes **straight to HSBC’s vault** — Ctrip can never peek inside.
*   If anyone tampers with the vault, HSBC knows immediately and shuts access down.

* * *

Would you like me to now:

*   Produce a **Markdown section** you can directly insert into your HSBC Developer Portal Architecture Document (with headings, YAML stubs, etc.),  
    or
*   Generate a **PowerPoint summary slide** (visual and executive-friendly, with icons and bullets)?

## Response:
please provide detail word explain for Integration patterns (2 options)

## Prompt:
Absolutely—here’s a clear, detailed, non-code explanation of the two **Integration Patterns** for the HSBC Partner Account-Opening SDK, plus how to choose, migrate, and govern them.

1) Pattern A — Hosted AO (recommended, fastest)
===============================================

What it is
----------

*   The partner app (e.g., Ctrip) embeds the **HSBC-hosted account-opening web journey** inside a secure container (Custom Tabs/SafariView/WKWebView).
*   The SDK handles identity (OIDC+PKCE), redeeming the **prefill token**, security hardening, events/callbacks, and optional **native KYC plugins** (camera, liveness, NFC).
*   Forms, validation, rules, and compliance UX live on HSBC servers (dynamic and centrally updated).

Why teams pick it
-----------------

*   **Speed to market**: minimal partner UI build; HSBC can change questions/flows without partner app updates.
*   **Compliance confidence**: regulated text, disclosures, and audit traces live on bank-controlled pages.
*   **Lower maintenance**: fewer mobile releases for every regulatory or product tweak.

Partner experience
------------------

*   Partner triggers one call (`startAccountOpening(...)`).
*   Receives progress events and a final **result** (completed/pending/cancelled/failed).
*   Partner can also query **Application Status API** and register **webhooks** for async updates.

UX characteristics
------------------

*   HSBC controls the content and layout; partner controls the frame (toolbar color/title/close) and locale.
*   Seamless in-app; user does not “leave” the partner app.
*   Optional native KYC reduces friction (e.g., higher doc-scan quality than pure web).

Security posture
----------------

*   Strongest by default: HSBC owns all forms and validations; **least chance of partner mis-implementation**.
*   PII never passes through partner app; **prefill token** pattern prevents on-device PII exposure.

Operational model
-----------------

*   HSBC ships backend flow updates anytime.
*   SDK updates only when mobile shell or plugins change (less frequent).
*   Feature flags allow **gradual rollout** and **jurisdiction-based toggles**.

Pros / Cons (quick view)
------------------------

*   ✅ Fastest to launch, safest for compliance, minimal partner dev.
*   ⚠️ Limited theming/control over inner form UX; web dependency for the core journey.

* * *

2) Pattern B — Native AO (deeper embed)
=======================================

What it is
----------

*   The partner app builds **native screens** (iOS/Android) for the AO steps (eligibility, personal info, address, KYC, review).
*   The SDK provides orchestration utilities: authentication, secure storage, API clients (eligibility, prefill redemption, doc upload, submit, status), and eventing.
*   HSBC exposes a **stable public schema** (field definitions, rules metadata) and **versioned APIs**.

Why teams pick it
-----------------

*   **Maximum UX control**: pixel-perfect integration with partner flows, A/B tests, cross-sell, and analytics.
*   **Deeper integration** with partner capabilities (eKYC already used elsewhere, stored addresses, profile autofill, app-wide design system).

Partner experience
------------------

*   Partner engineers implement forms and local validations using HSBC **schemas/rules**.
*   SDK handles secure calls and token management; partner wires callbacks and error states.
*   More work: partner must track **schema changes** and roll out app updates accordingly.

UX characteristics
------------------

*   Native performance, animations, accessibility consistent with partner’s app.
*   Can pre-validate and **reduce re-entry** by reusing partner profile data (with consent).

Security posture
----------------

*   Equivalent crypto/transport standards as Hosted AO, **but** greater shared responsibility:
    *   Partner must **not** log PII, must **follow schema** and **mask sensitive inputs**.
    *   Every UI change needs security review and regression testing.

Operational model
-----------------

*   HSBC publishes versioned field/rules metadata (e.g., `/metadata/v3`); backward-compatible for a window.
*   Partners update the app when metadata changes are breaking (e.g., new mandatory fields).
*   Requires **release coordination** for regulated copy or rule changes.

Pros / Cons (quick view)
------------------------

*   ✅ Best UX control, deeper partner reuse of existing assets.
*   ⚠️ Highest engineering & compliance overhead; app updates needed for many changes.

* * *

3) How to choose (decision guide)
=================================

| Criteria | Hosted AO | Native AO |
| --- | --- | --- |
| **Time to market** | ★★★★☆ (fast) | ★★☆☆☆ (slow) |
| **Compliance surface** | Minimal (HSBC-owned) | Shared (partner must track changes) |
| **UX control** | Medium (frame + locale + plugins) | High (full native UI) |
| **Maintenance** | Low | High |
| **Change velocity** | HSBC can update server-side instantly | Requires partner app releases |
| **Offline tolerance** | Low/Medium (web journey) | Medium (stage inputs offline, submit later) |
| **Risk of PII leakage** | Lowest (token pattern) | Low to Medium (if partner mis-handles) |
| **Internationalization** | Centralized in HSBC web | Shared across partner app |
| **A/B experiments** | HSBC-side | Partner-side (full control) |

**Default recommendation:** Start with **Hosted AO** and add **native KYC plugins** for best conversion. Move to **Native AO** only if business needs demand deeper integration and you have strong governance.

* * *

4) Data & API surface differences
=================================

**Common to both**

*   Prefill Token (S2S), OIDC+PKCE, encrypted uploads, Application Status API, webhooks, event callbacks, correlation IDs.

**Hosted AO**

*   Partner passes the token; HSBC web redeems it.
*   Most business logic server-side; SDK mainly containers + plugins.
*   Minimal UI API: open/close, theme, events, result.

**Native AO**

*   Partner consumes **metadata endpoints** (field lists, enums, validators, dynamic steps).
*   More APIs: eligibility, address verification, doc-type capabilities, submit, resume.
*   Requires robust **schema versioning** and backward compatibility strategy.

* * *

5) Observability & error handling
=================================

**Hosted AO**

*   Event stream from SDK (AO\_BEGIN, PREFILLED, KYC\_STARTED, SUBMITTED, ERROR).
*   Server-side telemetry (form drop-off, step latencies) fully under HSBC.
*   Partners see enough to support users without handling PII.

**Native AO**

*   Partner must instrument each native step.
*   HSBC provides error catalogs and remediation hints (e.g., `MORE_INFO: address proof`).
*   More potential edge cases (local validation vs server validation mismatches).

* * *

6) Theming, accessibility, localization
=======================================

**Hosted AO**

*   Theming: toolbar/close labels; HSBC pages have consistent brand and compliant wording.
*   Localization: HSBC owns translations and legal copy per market.
*   Accessibility: HSBC guarantees WCAG conformance of the hosted flow.

**Native AO**

*   Partner owns the full visual layer and accessibility—must meet HSBC standards.
*   HSBC supplies **legal copy bundles**; partner must keep them in sync.

* * *

7) Cost & governance (TCO view)
===============================

*   **Hosted AO** → lower engineering cost, lower compliance change-cost, centralized fixes.
*   **Native AO** → higher initial cost + recurring cost for every rules/copy change; needs a **joint change calendar**.

* * *

8) Jurisdictional nuance
========================

*   Markets with **fast-moving regulation** or **frequent product updates** → Hosted AO simplifies rollout.
*   Markets where partners demand **deep UX control** (super-apps, intense funnel optimization) → Native AO can be justified **if** a strong change-management process exists.

* * *

9) Migration paths
==================

A → B (start Hosted, evolve to Native)
--------------------------------------

1.  Launch Hosted AO to validate demand and operations.
2.  Introduce **native KYC plugins** to improve conversion.
3.  Expose **read-only status views** natively.
4.  Gradually replace selected steps with native screens (e.g., eligibility & prefill preview).
5.  Move to full Native AO once schema/rules APIs are stable and the partner is ready for ongoing upkeep.

B → A (rollback to Hosted)
--------------------------

*   Keep Hosted AO as a **feature-flagged fallback** for incident recovery or major breaking changes.
*   SDK supports switching at init via remote config (signed).

* * *

10) Typical delivery timelines
==============================

*   **Hosted AO**: 4–8 weeks (partner onboarding, prefill S2S, SDK wiring, UAT, go-live).
*   **Native AO**: 10–16+ weeks (all of the above **plus** native UI build, metadata wiring, copy reviews, accessibility, release cycles).

* * *

11) Readiness checklists
========================

Hosted AO – partner must:
-------------------------

*    Implement **server-to-server Prefill API** with consent evidence.
*    Add SDK and **deep-link** routing.
*    Wire **event listener** + result handling.
*    (Optional) Enable **native KYC plugins**.
*    Pass HSBC UAT (security headers, no PII in logs, network pinning active).

Native AO – partner must (in addition):
---------------------------------------

*    Build **native forms** per HSBC schema; mask/validate inputs locally.
*    Implement eligibility, address lookup/validation, document capture flows.
*    Subscribe to **metadata versioning**; handle **breaking changes**.
*    Localize legal copy; maintain accessibility and theming standards.
*    Produce evidence for **secure coding**, privacy manifests, and pen-test remediation.

* * *

12) Recommended starting stance
===============================

*   **Launch with Hosted AO**, plus **native document/liveness plugins** if you need higher completion rates.
*   Keep **Native AO** on the roadmap only if your partner has:
    *   A mature mobile engineering team,
    *   Robust compliance review cycles,
    *   Appetite for frequent, small updates aligned with HSBC schema changes.

* * *

If you want, I can turn this into:

*   a **one-page decision brief** for stakeholders, and
*   a **partner integration SOP** (operational checklist + RACI) you can drop into the developer portal.

## Response:
how about HSBC built page of AO joureny , and feed into SDK

## Prompt:
Thought for 8s

Great idea — that’s actually the **Hosted AO** pattern, but let’s design it very concretely as **“HSBC-built AO pages, delivered through the SDK.”** Below is a practical blueprint you can hand to both HSBC and partner teams.

1) What this means (in one line)
================================

HSBC owns and serves the **entire account-opening (AO) web journey**. The **SDK** is a lightweight, secure container that:

*   launches those pages in-app,
*   injects context (env, locale, product, campaign),
*   exposes a **native bridge** for device capabilities (camera, liveness, NFC),
*   returns a **structured result** (completed/pending/cancelled/error) to the partner app.

* * *

2) End-to-end flow (with “feed into SDK” steps)
===============================================

1.  **Partner backend → HSBC Prefill API** (mTLS)  
    Sends user consent + application data → gets back a **single-use prefillToken** (no PII on device).
2.  **Partner app → SDK.start()**  
    Partner calls `HSBCAOSdk.startAccountOpening()` with:
    *   `prefillToken`
*   `productCode`
*   `campaignId`
*   optional `trackingContext` (no PII)
*   optional capability flags (e.g., prefer native KYC)
    3.  **SDK opens HSBC AO page**
    *   Uses **ASWebAuthenticationSession / SFSafariView** (iOS) or **Custom Tabs** (Android); hardened **WebView** fallback.
*   SDK adds **runtime configuration** via query params + headers (env/locale/partnerId) and a signed session token.
    4.  **AO web redeems prefill** (server-to-server)  
    Pages load, HSBC backend calls prefill store with the token → prefilled fields appear.
5.  **Device capabilities via a JS bridge**  
    AO pages ask the SDK for **doc capture/liveness/NFC** when needed. SDK returns signed results to the page.
6.  **Submit & decision**  
    AO web submits to HSBC Orchestrator → returns an `applicationId` + state (approved/pending/refer).
7.  **SDK returns result to partner**  
    `AOResult.Completed | Pending | Cancelled | Failed` and emits privacy-safe events.

* * *

3) What the SDK needs from the HSBC-built pages
===============================================

3.1 AO URL contract
-------------------

*   Base: `https://ao.<region>.partner.hsbc.com/web/start`
*   Query params set by SDK:
    *   `prefillToken`
    *   `product` (e.g., `HSBC_CN_RP_CURR`)
    *   `locale` (e.g., `zh-CN`)
    *   `campaignId` (optional)
    *   `sdkVersion`, `platform` (for analytics)
*   Headers (SDK attaches):
    *   `X-HSBC-SDK-Signature` (JWS over params)
    *   `X-HSBC-Partner-Id`, `X-HSBC-Client-Id`
    *   `X-Device-Integrity` (attestation verdict)

3.2 JavaScript bridge (browser ↔ SDK)
-------------------------------------

The HSBC page includes a small script (bank-owned) that exposes async calls:

```ts
// Page → SDK
window.hsbcBridge.request('captureDocument', { type: 'ID_FRONT' })
window.hsbcBridge.request('performLiveness', {})
window.hsbcBridge.request('scanNFC', { docType: 'PASSPORT' })
window.hsbcBridge.request('scheduleAppointment', { branchId, slot })

// SDK → Page (events)
window.hsbcBridge.emit('kycProgress', { step: 'docUploaded' })
window.hsbcBridge.emit('error', { code, message })
window.hsbcBridge.emit('ready', { sdkVersion, features })
```

**Security**

*   The SDK verifies **origin** of the AO page and only exposes the bridge there.
*   Every response payload from SDK → page is **JWS-signed** with SDK’s embedded public key known to HSBC.
*   Page verifies signatures before accepting results.

3.3 AO page responsibilities
----------------------------

*   Render all forms, validations, disclosures, and consent text (compliant, localized).
*   Drive step logic (rules engine, product variants, KYC paths).
*   Call the **bridge** only when device functions are needed.
*   Handle **error catalogs** (e.g., `MORE_INFO`, `ID_QUALITY_LOW`) and retry flows.
*   On completion, **POST** the application and then **notify SDK** via a final page-to-SDK event:
    ```js
    window.hsbcBridge.emit('aoCompleted', { applicationId, state: 'PENDING' })
    ```
    The SDK maps this to `AOResult.Completed/Pending` for the partner.

* * *

4) Config “fed into SDK” (runtime + remote)
===========================================

4.1 Local init config (partner code)
------------------------------------

```json
{
  "environment": "SANDBOX",
  "partnerId": "ctrip_cn",
  "clientId": "ctrip-hsbc-ao-mobile",
  "redirectScheme": "ctrip",
  "locale": "zh-CN",
  "features": {
    "preferNativeKyc": true,
    "allowResume": true
  }
}
```

4.2 Remote signed config (HSBC → SDK)
-------------------------------------

*   Hosted at `/.well-known/hsbc-aosdk-config.json` (per region).
*   Pulled by SDK on init (cached 24h; signed JWS).
*   Toggles:
    *   **endpoint URLs** (AO web, auth issuer, status API by region)
    *   **feature flags** (enableNFC, enableLiveness, gracePinning, fallbackWebViewAllowed)
    *   **AB tests** (variant ids)
    *   **jurisdiction policy** (age, residency, document types)
*   Lets HSBC **change behavior without a new SDK release**.

* * *

5) Theming and content boundaries
=================================

*   **Partner-controlled (via SDK)**: container title, toolbar color, close/cancel labels, language/locale.
*   **HSBC-controlled (in page)**: fields, validation, legal copy, step order, KYC instructions, error content.
*   Why: preserves **compliance & auditability** while letting partner blend the frame with their app.

* * *

6) Error & status contract (simple, predictable)
================================================

**SDK → Partner app (final result)**

```json
{
  "type": "Completed",
  "applicationId": "APP-2025-10-12345"
}
```

or

```json
{
  "type": "Failed",
  "error": { "code": "PREFILL_EXPIRED", "message": "Prefill token expired", "recoverable": true }
}
```

**Partner can poll status API (server-to-server)**

```
GET /partner/v1/applications/{applicationId}/status
→ { "status": "PENDING|APPROVED|REJECTED|MORE_INFO", "updatedAt": "..." }
```

* * *

7) Security posture highlights (why this is safer)
==================================================

*   **No PII on device**: only a short-lived prefill token goes into the SDK; the page redeems it server-side.
*   **Origin-scoped bridge**: only HSBC AO origin can call native functions.
*   **Pinning + attestation**: TLS pinning and device checks before loading AO.
*   **Signed messages end-to-end**: SDK ↔ page JSON is JWS-signed and verified.
*   **Remote kill-switches**: HSBC can disable plugins or cut traffic per region via remote config.

* * *

8) Ops model (who updates what)
===============================

*   **HSBC** updates: AO web pages, rules, translations, disclosures, KYC vendor routing, remote config.
*   **SDK** updates: only when container/security/core APIs change (low frequency).
*   **Partner** updates: rarely (SDK version bumps or adding optional plugins).  
    This keeps **change cost** primarily on HSBC (centralized) and **time-to-market** high.

* * *

9) A/B testing and analytics (privacy-safe)
===========================================

*   HSBC controls experiments inside the AO page (copy, step order, instruction variants).
*   SDK emits **privacy-safe events** (e.g., `AO_BEGIN`, `KYC_STARTED`, `AO_SUBMITTED`, `AO_ERROR`) with **no PII**.
*   Correlation ID allows HSBC to join SDK telemetry with AO web analytics, without leaking PII to partners.

* * *

10) Minimal deliverables to make this work
==========================================

**From HSBC (server & web):**

*   AO web app (responsive, WCAG, localized).
*   Prefill API + Status API + Webhook endpoints (mTLS).
*   JS bridge library (tiny, bank-owned).
*   Remote signed config service per region.
*   Error code catalog & consent packs (per jurisdiction).

**From HSBC (mobile):**

*   SDK containers (Android/iOS) + optional KYC plugins.
*   Sample apps (Android/iOS) and integration guide.

**From Partner:**

*   Server-to-server **Prefill** call + (optional) **Webhook** consumption.
*   Add SDK, wire deep link, handle final `AOResult`, and basic theming.

* * *

11) Pros & cons specific to this approach
=========================================

**Pros**

*   Fastest go-live; HSBC can change the journey without partner app releases.
*   Strongest compliance posture; single source of truth for forms and copy.
*   Least PII exposure risk; best for multi-market rollout.

**Cons**

*   Inner form theming is controlled by HSBC (limited customization).
*   Requires reliable in-app web container (the SDK handles fallbacks).

* * *

12) Migration & extensibility
=============================

*   Start here (Hosted AO).
*   If later you need deeper UX control on a specific step (e.g., Doc capture), you **keep the hosted journey** but **swap that step** to native via the bridge (plugin first).
*   If you ever need full native control, you can progress to **Native AO** step-by-step, because:
    *   The same **Prefill/Status** APIs and **event contracts** still apply.
    *   The same **security posture** (pinning, attestation, JWS/JWE) carries over.

* * *

TL;DR
-----

**Yes — HSBC builds the AO pages, and the SDK “feeds” them context, security, and device powers.**  
This gives you **speed, safety, and centralized control**, while partners keep a seamless in-app experience and a very small integration surface.

If you want, I can turn this into:

*   a **2-page partner handout** (PDF/Markdown), or
*   a **technical interface spec** for the JS bridge (request/response payloads and error codes).



---
Powered by [ChatGPT Exporter](https://www.chatgptexporter.com)