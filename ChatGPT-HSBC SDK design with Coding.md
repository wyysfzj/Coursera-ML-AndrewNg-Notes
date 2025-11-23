# HSBC SDK design

**User:** Anonymous (os86@naifeistation.com)  
**Created:** 10/25/2025 21:56:01  
**Updated:** 11/23/2025 14:50:12  
**Exported:** 11/23/2025 15:08:58  
**Link:** [https://chatgpt.com/c/68fcd6f0-34a0-8329-9d83-3647aa1aac1d](https://chatgpt.com/c/68fcd6f0-34a0-8329-9d83-3647aa1aac1d)  

## Prompt:
![Uploaded image](https://chatgpt.com/backend-api/estuary/content?id=file_00000000405861f5a6a7dc37f82557a3&ts=489966&p=fs&cid=1&sig=a33890a384828782074052da3ad5715ad510e04ce95f4b27df463f3acb924bd1&v=0)

the attached image show the SDK solution of SCB. SCB generate SDK for its account opening api and  account opening page, Ctrip use this SDK in Ctrips'app.  customer can open SCB account in Ctrip APP.  Now I need produce the similar SDK for HSBC. please provide the detail design

## Response:
Thought for 6m 42s

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

## Prompt:
please explain detail of 3) packaging & platforms

## Response:
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

## Prompt:
give me a detail word description to explain above because I am not faimlier with mobile package

## Response:
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

## Prompt:
generate both

## Response:
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

## Prompt:
please also provide detail word explain for Security & compliance

## Response:
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

## Prompt:
please provide detail word explain for Integration patterns (2 options)

## Response:
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

## Prompt:
how about HSBC built page of AO joureny , and feed into SDK

## Response:
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

## Prompt:
both, partner handout with Markdown

## Response:
**Hsbc Sdk Partner Handout**

# HSBC Partner Account Opening SDK – Partner Integration Handout

---

## 1. Overview

This document explains how HSBC’s **Account Opening (AO)** journey is delivered through the **HSBC Partner SDK**, which allows partners (like Ctrip) to enable users to open HSBC accounts **directly inside their own mobile or web apps**.

The approach described here is called **Hosted AO Journey** — HSBC builds and hosts all AO web pages, while the SDK provides a secure, branded container that partners embed.

---

## 2. Business Value

**For Partners:**  
- Offer HSBC account opening in your own app without building or maintaining banking pages.  
- Increase engagement and conversion by keeping users inside your app.  
- Integrate once, and automatically get future HSBC AO feature updates.

**For HSBC:**  
- Maintain full compliance and brand control across all partner integrations.  
- Centralize product logic, eligibility, and KYC flows for all markets.  
- Achieve fast rollout across multiple partners and jurisdictions.

---

## 3. End-to-End Flow

1. **Partner backend → HSBC Prefill API (mTLS)**  
   - Partner sends user consent + profile info directly to HSBC.  
   - HSBC returns a **prefillToken** (short-lived, single-use, no PII).

2. **Partner app → SDK.startAccountOpening()**  
   - Partner calls the SDK method with `prefillToken`, `productCode`, `locale`, etc.

3. **SDK → HSBC AO Pages**  
   - SDK launches HSBC’s hosted AO pages inside a secure browser view (Custom Tabs / SafariView / WebView fallback).  
   - SDK adds runtime metadata and verifies integrity before loading the HSBC domain.

4. **HSBC AO web → Prefill redemption**  
   - HSBC pages call backend APIs to fetch prefilled user info securely.

5. **In-app KYC (via SDK bridge)**  
   - AO pages request device actions (camera, liveness, NFC) using the SDK bridge.  
   - SDK executes natively and returns signed results.

6. **Application submission & result**  
   - AO pages submit to HSBC backend → return application status.  
   - SDK emits the final result to the partner app.

---

## 4. Technical Architecture

```
[ Partner App ]
    ↓
HSBC Partner SDK (Android/iOS/Web)
    ↓
Secure Container (Custom Tab / SafariView / WebView)
    ↓
HSBC AO Web Pages (Hosted by HSBC)
    ↓
HSBC Partner Gateway (Prefill API, KYC, Orchestrator, Status)
```

---

## 5. Integration Steps for Partners

| Step | Description | Responsibility |
|------|--------------|----------------|
| 1 | Register with HSBC Partner Onboarding | Partner + HSBC |
| 2 | Obtain credentials, mTLS certs, and sandbox access | HSBC |
| 3 | Implement **Prefill API** (server-to-server, no PII on device) | Partner backend |
| 4 | Integrate **SDK** into app | Partner mobile/web team |
| 5 | Configure `redirectScheme`, `partnerId`, and `clientId` | Partner |
| 6 | Implement `AOResult` handling and event listener | Partner |
| 7 | Test and certify integration in sandbox | Partner + HSBC |
| 8 | Go live and monitor through HSBC dashboards | Both |

---

## 6. Security & Compliance Summary

| Security Layer | Description |
|----------------|-------------|
| **Identity** | OAuth 2.1 Authorization Code Flow + PKCE ensures each session is unique and secure. |
| **Data privacy** | Partner never handles PII; only short-lived prefillToken. |
| **Transport** | TLS 1.2+, certificate pinning, and mTLS for partner backend calls. |
| **Device integrity** | Root/jailbreak detection, Play Integrity (Android), App Attest (iOS). |
| **Encryption** | JWS (signing) + JWE (encryption) for payloads. |
| **Audit & Consent** | User consent captured and timestamped in HSBC systems. |

All data handling is aligned with GDPR, PDPA, and local data residency laws (China CSL, HK PDPO, etc.).

---

## 7. User Experience Ownership

| Aspect | Ownership |
|---------|------------|
| Frame (toolbar, color, language) | Partner (via SDK config) |
| AO web pages (forms, disclosures, steps) | HSBC |
| Legal copy and compliance text | HSBC |
| KYC plugins (camera, NFC, liveness) | HSBC SDK (native modules) |
| Error handling and AO results | SDK + HSBC APIs |

---

## 8. Advantages of HSBC-Built AO Pages

✅ **Fast deployment** — no need to rebuild flows for each partner or product change.  
✅ **Always compliant** — HSBC legal copy and process stay up to date centrally.  
✅ **Consistent UX** — single HSBC journey across all partners.  
✅ **Secure by design** — zero PII on device, signed SDK ↔ page messages.  
✅ **Future-proof** — HSBC can update AO steps or validations without partner app releases.

---

## 9. Typical Timeline

| Phase | Duration | Notes |
|--------|-----------|-------|
| Setup & Onboarding | 1–2 weeks | Credentials, sandbox, certs |
| SDK Integration | 2–3 weeks | Add SDK, handle result, UI test |
| Prefill API Integration | 1–2 weeks | mTLS, consent logic |
| UAT & Certification | 2 weeks | HSBC-led tests |
| Go Live | 1 week | Production rollout |

**Total:** 6–8 weeks for a complete, compliant integration.

---

## 10. Support and Governance

- HSBC Partner Tech Support: integration guidance and sandbox troubleshooting.  
- HSBC Security Team: performs penetration testing and certifies final build.  
- Remote configuration allows HSBC to disable or patch features instantly if issues occur.

---

## 11. Summary

By embedding the HSBC SDK, partners can:
- Launch the full HSBC AO journey **inside their own app**.
- Deliver a smooth user experience while **HSBC retains compliance control**.
- Reduce integration time, risk, and cost.

This pattern — *HSBC-built pages fed into the SDK* — is the **recommended and supported model** for all new HSBC digital partnerships.

---

**Contact:**  
HSBC Partner Engineering Team  
📧 partner-tech@hsbc.com  
🌐 https://developer.hsbc.com/partners

---

gpt-5-1-pro cannot edit documents

Here’s your **Markdown partner handout** for the Hosted AO Journey (“HSBC-built pages fed into SDK”) — complete and ready to share or publish in your developer documentation.

You can review it in the canvas above, titled **“HSBC SDK Partner Handout.”**  
It includes:

*   A concise business-facing overview.
*   Step-by-step integration process.
*   Technical and security summary.
*   Ready-to-use contact and rollout sections.

Would you like me to add the **second document** (the **technical interface spec** for the JS bridge — with request/response schema, error codes, and examples) next?

## Prompt:
now I will start to build a MVP of SDK for iOS according to above design with VS code and Copilot. please provide the details guide, guide me step by step from developer environment set up to testing

## Response:
Thought for 9m 16s

Awesome—let’s build an **MVP iOS SDK** (Hosted AO: “HSBC‑built pages fed into the SDK”) using **VS Code + Copilot**. Below is a **complete, step‑by‑step guide** from environment setup to packaging and testing. I’ll give you:

*   A pragmatic MVP scope (what we will and won’t build today)
*   Environment + tools setup (for VS Code + iOS)
*   Repository structure
*   A **Swift Package SDK** you can code with Copilot (copy‑paste ready)
*   A simple **demo app** to test the SDK
*   Unit/UI testing plan
*   How to archive an **XCFramework** for distribution
*   Hardening backlog (what to add after MVP)
*   Checklists + Copilot prompts at each step

* * *

0) MVP scope (2 weeks target)
-----------------------------

**We will implement now**

*   `HSBCAOSdk.initialize(config:)`
*   `HSBCAOSdk.setEventListener(_:)`
*   `HSBCAOSdk.startAccountOpening(from:params:completion:)`
*   Secure in‑app container using **WKWebView** (so we can do the JS bridge in the MVP)
*   A **JS bridge** for AO pages: `request(...)` / `emit(...)`
*   A **fake AO page** (local HTML) that simulates HSBC pages, to prove the end‑to‑end flow
*   Result handling: `Completed / Pending / Cancelled / Failed`

**We will stub now (add later)**

*   Real OIDC/PKCE (will come with ASWebAuthenticationSession)
*   Real prefill redemption (we’ll pass a demo token)
*   Certificate pinning & attestation (will add once we switch to real endpoints)
*   Native KYC plugins (we’ll simulate success for doc/liveness/NFC)

> This is the quickest path to a working SDK you can hand to internal stakeholders and iterate.

* * *

1) Environment setup
--------------------

1.  **macOS + Xcode**
    *   Install **Xcode 15+** (latest stable) from App Store.
    *   Accept license:
        ```bash
        sudo xcodebuild -license accept
        ```
    *   Install command‑line tools:
        ```bash
        xcode-select --install
        ```
2.  **VS Code**
    *   Install **VS Code**.
    *   Extensions:
        *   **Swift** (by Apple)
        *   **CodeLLDB** (debugging)
        *   **ESLint**/**Prettier** (if you’ll touch JS/HTML test page)
        *   **GitHub Copilot** (enabled)
3.  **Command‑line helpers (optional)**
    *   Homebrew: [https://brew.sh](https://brew.sh)
    *   `brew install jq yq` (handy for scripts)
4.  **Git**
    ```bash
    git config --global user.name "Your Name"
    git config --global user.email you@example.com
    ```

* * *

2) Create the repo
------------------

```bash
mkdir hsbc-ao-sdk-ios
cd hsbc-ao-sdk-ios
git init
```

Recommended layout:

```
hsbc-ao-sdk-ios/
├─ sdk/                      # Swift Package (the SDK)
│  ├─ Package.swift
│  └─ Sources/HSBCAOSDKCore/
│     ├─ HSBCAOSdk.swift
│     ├─ HSBCAOViewController.swift
│     ├─ HSBCWebBridge.swift
│     └─ Resources/
│        └─ ao_test.html     # fake AO page for MVP
├─ examples/
│  └─ PartnerDemoApp/        # Xcode app that uses the SDK
├─ scripts/
│  ├─ build_xcframework.sh
│  └─ make_release.sh
└─ README.md
```

> You’ll edit Swift in **VS Code**, but you’ll still run & sign via Xcode behind the scenes.

* * *

3) Create the SDK as a Swift Package
------------------------------------

From `hsbc-ao-sdk-ios/`:

```bash
cd sdk
swift package init --type library
```

Open `Package.swift` and replace with:

```swift
// swift-tools-version: 5.9
import PackageDescription

let package = Package(
    name: "HSBCAOSDKCore",
    defaultLocalization: "en",
    platforms: [.iOS(.v14)],
    products: [
        .library(name: "HSBCAOSDKCore", targets: ["HSBCAOSDKCore"])
    ],
    targets: [
        .target(
            name: "HSBCAOSDKCore",
            resources: [
                .process("Resources/ao_test.html")
            ]
        ),
        .testTarget(
            name: "HSBCAOSDKCoreTests",
            dependencies: ["HSBCAOSDKCore"]
        )
    ]
)
```

Create `Resources/ao_test.html` (temporary fake AO page). **Paste this**:

```html
<!doctype html>
<html>
<head>
<meta charset="utf-8" />
<meta name="viewport" content="width=device-width, initial-scale=1" />
<title>HSBC AO Test Page</title>
<style>
body { font-family: -apple-system, system-ui; padding: 16px; }
button { padding: 12px 16px; margin: 8px 0; }
pre { background: #f6f8fa; padding: 8px; border-radius: 6px; }
</style>
<script>
(function(){
  // Simple bridge bootstrap for MVP using WKWebView message handler
  window.hsbcBridge = {
    _pending: {},
    request: function(name, payload) {
      const id = Math.random().toString(36).slice(2);
      window.webkit?.messageHandlers?.hsbcBridge?.postMessage({ kind: 'request', id, name, payload });
      return new Promise((resolve, reject) => { this._pending[id] = {resolve, reject}; setTimeout(()=> reject(new Error('timeout')), 15000); });
    },
    emit: function(name, payload) {
      window.webkit?.messageHandlers?.hsbcBridge?.postMessage({ kind: 'event', name, payload });
    },
    _resolve: function(id, result){ if(this._pending[id]){ this._pending[id].resolve(result); delete this._pending[id]; } },
    _reject: function(id, error){ if(this._pending[id]){ this._pending[id].reject(error); delete this._pending[id]; } }
  };
})();
</script>
</head>
<body>
  <h2>HSBC AO – Test Page</h2>
  <p>This simulates the HSBC‑hosted AO journey for SDK MVP.</p>

  <button onclick="hsbcBridge.emit('ready', {features:['doc','liveness','nfc']})">Emit READY</button>
  <button onclick="hsbcBridge.request('performLiveness', {}).then(r => alert('Liveness: '+JSON.stringify(r)))">Request Liveness</button>
  <button onclick="hsbcBridge.request('captureDocument', {type:'ID_FRONT'}).then(r => alert('Doc: '+JSON.stringify(r)))">Capture Document</button>
  <button onclick="hsbcBridge.emit('aoCompleted', {applicationId:'APP-TEST-123', state:'PENDING'})">Emit Completed</button>

  <p>Incoming messages are resolved by native code with: <code>hsbcBridge._resolve(id, result)</code></p>
</body>
</html>
```

* * *

4) Implement the SDK (copy/paste these MVP files)
-------------------------------------------------

**`Sources/HSBCAOSDKCore/HSBCAOSdk.swift`**

```swift
import Foundation
import UIKit

public struct HSBCInitConfig {
    public enum Environment { case sandbox, production }
    public let environment: Environment
    public let partnerId: String
    public let clientId: String
    public let redirectScheme: String
    public let locale: String

    public init(environment: Environment,
                partnerId: String,
                clientId: String,
                redirectScheme: String,
                locale: String = "zh-CN") {
        self.environment = environment
        self.partnerId = partnerId
        self.clientId = clientId
        self.redirectScheme = redirectScheme
        self.locale = locale
    }
}

public struct AOStartParams {
    public let prefillToken: String
    public let productCode: String
    public let campaignId: String?

    public init(prefillToken: String, productCode: String, campaignId: String? = nil) {
        self.prefillToken = prefillToken
        self.productCode = productCode
        self.campaignId = campaignId
    }
}

public enum AOResult {
    case completed(applicationId: String)
    case pending(applicationId: String)
    case cancelled
    case failed(code: String, message: String, recoverable: Bool)
}

public typealias AOEventListener = (_ name: String, _ attributes: [String: Any]) -> Void

public enum HSBCAOSdk {
    private static var config: HSBCInitConfig?
    private static var eventListener: AOEventListener?

    public static func initialize(config: HSBCInitConfig) {
        self.config = config
    }

    public static func setEventListener(_ listener: @escaping AOEventListener) {
        self.eventListener = listener
    }

    static func emit(_ name: String, _ attrs: [String: Any] = [:]) {
        self.eventListener?(name, attrs)
    }

    /// Presents the AO container (WKWebView MVP) and resolves with AOResult when page emits completion.
    public static func startAccountOpening(from presenter: UIViewController,
                                           params: AOStartParams,
                                           completion: @escaping (AOResult) -> Void) {
        guard let config = self.config else {
            completion(.failed(code: "SDK_NOT_INITIALIZED", message: "Call initialize(config:) first.", recoverable: false))
            return
        }
        let vc = HSBCAOViewController(config: config, params: params) { result in
            completion(result)
        }
        let nav = UINavigationController(rootViewController: vc)
        nav.modalPresentationStyle = .formSheet
        presenter.present(nav, animated: true)
        emit("AO_BEGIN", ["product": params.productCode, "campaignId": params.campaignId ?? ""])
    }
}
```

**`Sources/HSBCAOSDKCore/HSBCWebBridge.swift`**

```swift
import Foundation
import WebKit

final class HSBCWebBridge: NSObject, WKScriptMessageHandler {
    private weak var webView: WKWebView?
    private let onEvent: (String, [String: Any]) -> Void
    private let onComplete: (AOResult) -> Void

    init(webView: WKWebView,
         onEvent: @escaping (String, [String: Any]) -> Void,
         onComplete: @escaping (AOResult) -> Void) {
        self.webView = webView
        self.onEvent = onEvent
        self.onComplete = onComplete
    }

    func userContentController(_ userContentController: WKUserContentController,
                               didReceive message: WKScriptMessage) {
        guard message.name == "hsbcBridge" else { return }
        guard let body = message.body as? [String: Any] else { return }

        let kind = body["kind"] as? String
        switch kind {
        case "event":
            let name = body["name"] as? String ?? "UNKNOWN"
            let payload = body["payload"] as? [String: Any] ?? [:]
            handleEvent(name: name, payload: payload)

        case "request":
            let id = body["id"] as? String ?? UUID().uuidString
            let name = body["name"] as? String ?? "UNKNOWN"
            let payload = body["payload"] as? [String: Any] ?? [:]
            handleRequest(id: id, name: name, payload: payload)

        default: break
        }
    }

    private func handleEvent(name: String, payload: [String: Any]) {
        HSBCAOSdk.emit(name, payload)

        // MVP: react only to AO completion event from test page
        if name == "aoCompleted" {
            let appId = payload["applicationId"] as? String ?? "APP-UNKNOWN"
            let state = (payload["state"] as? String ?? "").uppercased()
            if state == "PENDING" {
                onComplete(.pending(applicationId: appId))
            } else {
                onComplete(.completed(applicationId: appId))
            }
        } else if name == "error" {
            let code = payload["code"] as? String ?? "UNKNOWN"
            let msg = payload["message"] as? String ?? "Unknown error"
            onComplete(.failed(code: code, message: msg, recoverable: true))
        }
    }

    private func handleRequest(id: String, name: String, payload: [String: Any]) {
        // MVP: stub native features and resolve back to page
        switch name {
        case "performLiveness":
            // Simulate async success
            DispatchQueue.main.asyncAfter(deadline: .now() + 0.8) {
                self.resolve(id: id, result: ["result": "PASS", "score": 0.98])
                HSBCAOSdk.emit("KYC_COMPLETED", ["method": "LIVENESS", "result": "PASS"])
            }
        case "captureDocument":
            let type = (payload["type"] as? String) ?? "UNKNOWN"
            DispatchQueue.main.asyncAfter(deadline: .now() + 0.6) {
                self.resolve(id: id, result: ["docType": type, "quality": "GOOD"])
                HSBCAOSdk.emit("KYC_STARTED", ["method": "DOC"])
            }
        case "scanNFC":
            DispatchQueue.main.asyncAfter(deadline: .now() + 0.6) {
                self.resolve(id: id, result: ["chipRead": true, "dg1": "…"])
            }
        default:
            self.reject(id: id, error: ["code":"UNSUPPORTED","message":"Request not supported in MVP"])
        }
    }

    private func resolve(id: String, result: [String: Any]) {
        guard let webView else { return }
        if let data = try? JSONSerialization.data(withJSONObject: result),
           let json = String(data: data, encoding: .utf8) {
            let js = "window.hsbcBridge._resolve('\(id)', \(json));"
            webView.evaluateJavaScript(js, completionHandler: nil)
        }
    }

    private func reject(id: String, error: [String: Any]) {
        guard let webView else { return }
        if let data = try? JSONSerialization.data(withJSONObject: error),
           let json = String(data: data, encoding: .utf8) {
            let js = "window.hsbcBridge._reject('\(id)', \(json));"
            webView.evaluateJavaScript(js, completionHandler: nil)
        }
    }
}
```

**`Sources/HSBCAOSDKCore/HSBCAOViewController.swift`**

```swift
import UIKit
import WebKit

final class HSBCAOViewController: UIViewController, WKNavigationDelegate {
    private let config: HSBCInitConfig
    private let params: AOStartParams
    private let onComplete: (AOResult) -> Void

    private var webView: WKWebView!
    private var bridge: HSBCWebBridge!

    init(config: HSBCInitConfig, params: AOStartParams, onComplete: @escaping (AOResult) -> Void) {
        self.config = config
        self.params = params
        self.onComplete = onComplete
        super.init(nibName: nil, bundle: nil)
        self.title = "HSBC Account Opening"
    }

    @available(*, unavailable)
    required init?(coder: NSCoder) { fatalError("init(coder:) has not been implemented") }

    override func viewDidLoad() {
        super.viewDidLoad()
        view.backgroundColor = .systemBackground

        // Close button
        navigationItem.leftBarButtonItem = UIBarButtonItem(barButtonSystemItem: .close, target: self, action: #selector(closeTapped))

        // WebView + bridge
        let contentController = WKUserContentController()
        contentController.add(self, name: "hsbcBridge")

        let configWK = WKWebViewConfiguration()
        configWK.userContentController = contentController
        configWK.preferences.javaScriptEnabled = true

        let wv = WKWebView(frame: .zero, configuration: configWK)
        wv.navigationDelegate = self
        wv.translatesAutoresizingMaskIntoConstraints = false
        view.addSubview(wv)
        NSLayoutConstraint.activate([
            wv.topAnchor.constraint(equalTo: view.topAnchor),
            wv.bottomAnchor.constraint(equalTo: view.bottomAnchor),
            wv.leadingAnchor.constraint(equalTo: view.leadingAnchor),
            wv.trailingAnchor.constraint(equalTo: view.trailingAnchor)
        ])
        self.webView = wv

        self.bridge = HSBCWebBridge(webView: wv, onEvent: { name, attrs in
            HSBCAOSdk.emit(name, attrs)
        }, onComplete: { [weak self] result in
            self?.complete(result)
        })

        loadAOTestPage()
    }

    @objc private func closeTapped() {
        complete(.cancelled)
    }

    private func complete(_ result: AOResult) {
        dismiss(animated: true) { [onComplete] in
            onComplete(result)
        }
    }

    private func loadAOTestPage() {
        // MVP: load the embedded HTML resource; later, replace with real AO URL.
        guard let url = Bundle.module.url(forResource: "ao_test", withExtension: "html") else {
            complete(.failed(code: "RESOURCE_MISSING", message: "ao_test.html not found", recoverable: false))
            return
        }
        // Append minimal query parameters as per design (simulated)
        var comps = URLComponents(url: url, resolvingAgainstBaseURL: false)!
        var q = [
            URLQueryItem(name: "prefillToken", value: params.prefillToken),
            URLQueryItem(name: "product", value: params.productCode),
            URLQueryItem(name: "locale", value: config.locale),
            URLQueryItem(name: "campaignId", value: params.campaignId ?? "")
        ]
        comps.queryItems = q
        webView.loadFileURL(comps.url!, allowingReadAccessTo: url.deletingLastPathComponent())
    }

    // Optional: navigation hooks for error handling
    func webView(_ webView: WKWebView, didFail nav: WKNavigation!, withError error: Error) {
        HSBCAOSdk.emit("AO_ERROR", ["message": error.localizedDescription])
    }
}

extension HSBCAOViewController: WKScriptMessageHandler {
    func userContentController(_ userContentController: WKUserContentController, didReceive message: WKScriptMessage) {
        bridge.userContentController(userContentController, didReceive: message)
    }
}
```

> ✅ At this point, the SDK compiles as a Swift Package and exposes the three main APIs. The WKWebView loads a local test page and the bridge is live.

* * *

5) Create the demo app to test the SDK
--------------------------------------

There isn’t an official CLI to generate an iOS app. Easiest way:

1.  Open **Xcode** → **File > New > Project… > iOS App**
    *   Product Name: `PartnerDemoApp`
    *   Interface: SwiftUI (or UIKit if you prefer)
    *   Save it in `hsbc-ao-sdk-ios/examples/PartnerDemoApp`
2.  Add the SDK package (local):
    *   In the app project: **Project Navigator > Package Dependencies > + > Add Local…**
    *   Choose `hsbc-ao-sdk-ios/sdk`.
3.  In the app’s code (SwiftUI `ContentView.swift`), wire the SDK:

```swift
import SwiftUI
import HSBCAOSDKCore

struct ContentView: View {
    @State private var status: String = "Idle"

    var body: some View {
        VStack(spacing: 16) {
            Text("Partner Demo App")
                .font(.title3)
            Button("Initialize SDK") {
                let cfg = HSBCInitConfig(environment: .sandbox,
                                         partnerId: "ctrip_cn",
                                         clientId: "ctrip-hsbc-ao-ios",
                                         redirectScheme: "ctrip",
                                         locale: "zh-CN")
                HSBCAOSdk.initialize(config: cfg)
                HSBCAOSdk.setEventListener { name, attrs in
                    print("Event:", name, attrs)
                }
                status = "SDK initialized"
            }
            Button("Start Account Opening") {
                guard let top = topController() else { return }
                let params = AOStartParams(prefillToken: "ptok_demo_123",
                                           productCode: "HSBC_CN_RP_CURR",
                                           campaignId: "ctrip_oct_campaign")
                HSBCAOSdk.startAccountOpening(from: top, params: params) { result in
                    switch result {
                    case .completed(let id): status = "Completed: \(id)"
                    case .pending(let id):   status = "Pending: \(id)"
                    case .cancelled:         status = "Cancelled"
                    case .failed(let c, let m, _): status = "Failed: \(c) \(m)"
                    }
                }
            }
            Text(status).padding(.top, 20)
        }
        .padding()
    }

    private func topController(base: UIViewController? = nil) -> UIViewController? {
        let keyWindow = UIApplication.shared.connectedScenes
            .flatMap { ($0 as? UIWindowScene)?.windows ?? [] }
            .first { $0.isKeyWindow }
        var top = keyWindow?.rootViewController
        while let presented = top?.presentedViewController { top = presented }
        return top
    }
}
```

4.  **Run** on iOS Simulator (any iPhone).
    *   Tap **Initialize SDK**
    *   Tap **Start Account Opening** → The WKWebView opens the test page
    *   Tap “Emit READY”, “Request Liveness”, “Capture Document”, “Emit Completed” to see callbacks.

* * *

6) Unit testing (minimal)
-------------------------

Create `Tests/HSBCAOSDKCoreTests/HSBCAOSDKCoreTests.swift`:

```swift
import XCTest
@testable import HSBCAOSDKCore

final class HSBCAOSDKCoreTests: XCTestCase {
    func testInitialize() {
        let cfg = HSBCInitConfig(environment: .sandbox, partnerId: "p", clientId: "c", redirectScheme: "app")
        HSBCAOSdk.initialize(config: cfg)
        // If no crash, it’s fine for MVP
        XCTAssertTrue(true)
    }
}
```

Run:

```bash
cd sdk
swift test
```

> For UI/bridge tests, you’d use **XCTest + XCUI** in the demo app project. Keep that for the next iteration.

* * *

7) Build an XCFramework (for distribution to partners later)
------------------------------------------------------------

Create `scripts/build_xcframework.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail

PRODUCT="HSBCAOSDKCore"
WORKSPACE="$(pwd)/.."
OUT_DIR="$WORKSPACE/dist"
SDK_DIR="$WORKSPACE/sdk"

rm -rf "$OUT_DIR"
mkdir -p "$OUT_DIR"

# Build archives for device and simulator
xcodebuild archive \
  -scheme "$PRODUCT" \
  -destination "generic/platform=iOS" \
  -archivePath "$OUT_DIR/ios_devices.xcarchive" \
  SKIP_INSTALL=NO BUILD_LIBRARY_FOR_DISTRIBUTION=YES

xcodebuild archive \
  -scheme "$PRODUCT" \
  -destination "generic/platform=iOS Simulator" \
  -archivePath "$OUT_DIR/ios_sim.xcarchive" \
  SKIP_INSTALL=NO BUILD_LIBRARY_FOR_DISTRIBUTION=YES

# Create XCFramework
xcodebuild -create-xcframework \
  -framework "$OUT_DIR/ios_devices.xcarchive/Products/Library/Frameworks/$PRODUCT.framework" \
  -framework "$OUT_DIR/ios_sim.xcarchive/Products/Library/Frameworks/$PRODUCT.framework" \
  -output "$OUT_DIR/$PRODUCT.xcframework"

echo "XCFramework at: $OUT_DIR/$PRODUCT.xcframework"
```

> Open the **Package** in Xcode once so it creates a scheme for `HSBCAOSDKCore` (Product → Scheme → Manage Schemes → Shared). Then run the script.

* * *

8) Hardening backlog (post‑MVP)
-------------------------------

Add these incrementally:

1.  **Switch container to `ASWebAuthenticationSession`** for OIDC login (best UX) and keep `WKWebView` for bridge‑enabled AO steps (or expose a hybrid).
2.  **Certificate pinning** for any `WKWebView`/`URLSession` calls you own (not possible inside ASWebAuth).
3.  **App Transport Security** (ATS) hardened; allow only HSBC domains with TLS1.2+.
4.  **Privacy manifest** (iOS 17+): `PrivacyInfo.xcprivacy` declaring data collection = none.
5.  **JWS/JWE** for bridge payloads (use CryptoKit) if the page expects signatures.
6.  **Root/jailbreak detection** (signal‑based) and **App Attest/DeviceCheck** (verdict included as header).
7.  **Remote config** fetch (signed JSON) to toggle features per region.
8.  **Error catalog** mapping to `.failed(code:message:recoverable:)`.
9.  **KYC plugins**: document capture (AVFoundation), liveness (ML model), NFC (CoreNFC).
10.  **Localization & accessibility** pass (WCAG, large text, VoiceOver).

* * *

9) Developer checklists
-----------------------

**Before coding**

*    Xcode 15+ installed and CLI tools set
*    VS Code + Swift extension + Copilot enabled
*    Git repo scaffolded

**SDK “done” criteria (MVP)**

*    `initialize`, `setEventListener`, `startAccountOpening` public APIs
*    Presents WKWebView container and loads `ao_test.html`
*    JS bridge supports `request` + `emit`, returns completion
*    Demo app receives `.completed/.pending/.cancelled/.failed`
*    Unit tests pass; demo app runs on simulator

**Pre‑release**

*    Share the scheme; archive builds for Device + Simulator
*    `HSBCAOSDKCore.xcframework` generated
*    README with integration steps written

* * *

10) Copilot prompts (paste into VS Code to speed you up)
--------------------------------------------------------

*   **Create the public SDK API file**
    > “Create a Swift file `HSBCAOSdk.swift` for a public SDK with types `HSBCInitConfig`, `AOStartParams`, enum `AOResult`, and a `HSBCAOSdk` namespace exposing `initialize(config:)`, `setEventListener(_:)`, and `startAccountOpening(from:params:completion:)`.”
*   **Create a WKWebView container with a JS bridge**
    > “Create `HSBCAOViewController` that displays a WKWebView full‑screen, injects a script message handler named `hsbcBridge`, handles a close button, and loads a local resource `ao_test.html` with query params for token/product/locale/campaignId. On JS events, forward to a bridge helper.”
*   **Bridge handler**
    > “Create `HSBCWebBridge` as a `WKScriptMessageHandler` that routes `{kind:'event'| 'request', name, payload, id}`; simulate `performLiveness`, `captureDocument`, `scanNFC` requests and resolve them with `window.hsbcBridge._resolve(id, result)`.”
*   **Demo app usage**
    > “In the SwiftUI demo app, add a view with two buttons: one to initialize the SDK, one to call `startAccountOpening`. Show the latest status as text.”
*   **Unit test**
    > “Write a minimal XCTest that initializes the SDK config and asserts no crash.”

* * *

11) What testing looks like (manual)
------------------------------------

*   Launch demo app → tap **Initialize SDK** → tap **Start Account Opening**
*   In the test page:
    *   Tap **Emit READY** → check Xcode console logs for `ready` event.
    *   Tap **Request Liveness** → alert shows simulated PASS.
    *   Tap **Capture Document** → alert shows simulated GOOD.
    *   Tap **Emit Completed** → SDK dismisses and returns `.pending(APP-TEST-123)`.

If all of that works, your **SDK MVP is working** end‑to‑end.

* * *

12) Next step options (I can deliver next)
------------------------------------------

*   A **CI workflow** (GitHub Actions) to build the package and export the XCFramework artifact.
*   A **Privacy manifest** (`PrivacyInfo.xcprivacy`) + ATS sample.
*   An **ASWebAuthenticationSession** prototype for the real OIDC step (with a fake issuer to demo the callback).
*   A **sample Postman collection** + a mock Prefill API so your demo can go “online.”

* * *

If you want, tell me which parts you’d like me to **auto‑generate** (scripts, manifest, ATS, CI) and I’ll produce the exact files you can drop into your repo.

## Prompt:
please

## Response:
Thought for 54s

awesome — i’ll drop in everything you asked for so you can copy-paste straight into your repo: **CI workflow, privacy manifest, ATS config, an ASWebAuthenticationSession prototype, a tiny mock Prefill/Status API (Node/Express) + Postman collection, and a couple of helper scripts.**

below I reference the repo layout you already created:

```
hsbc-ao-sdk-ios/
├─ sdk/
│  ├─ Package.swift
│  └─ Sources/HSBCAOSDKCore/…
├─ examples/
│  └─ PartnerDemoApp/
├─ scripts/
│  ├─ build_xcframework.sh
│  └─ make_release.sh             ← NEW
├─ ci/
│  └─ fastlane/                   ← OPTIONAL (future)
├─ mock/
│  ├─ server.js                   ← NEW (Node mock)
│  ├─ package.json                ← NEW
│  └─ .env.example                ← NEW
├─ postman/
│  └─ HSBC-Partner-APIs.postman_collection.json   ← NEW
└─ .github/workflows/
   └─ ios-sdk.yml                 ← NEW (GitHub Actions CI)
```

* * *

1) GitHub Actions CI (build, test, XCFramework artifact)
========================================================

**File:** `.github/workflows/ios-sdk.yml`

```yaml
name: iOS SDK – Build, Test & XCFramework

on:
  push:
    branches: [ main ]
    tags: [ 'v*' ]
  pull_request:
    branches: [ main ]

jobs:
  build-test-archive:
    runs-on: macos-14

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Select Xcode 15
        run: |
          sudo xcode-select -s /Applications/Xcode_15.4.app || true
          xcodebuild -version

      - name: Cache SPM
        uses: actions/cache@v4
        with:
          path: |
            ~/Library/Developer/Xcode/DerivedData
            ~/Library/Caches/org.swift.swiftpm
          key: ${{ runner.os }}-spm-${{ hashFiles('**/Package.resolved') }}

      - name: Build & Test (Swift Package)
        working-directory: sdk
        run: |
          swift package resolve
          swift build -v
          swift test -v

      - name: Prepare Xcode scheme (first time only)
        run: |
          # Ensure a shared scheme exists for HSBCAOSDKCore
          /usr/libexec/PlistBuddy -c "Add :SchemeUserState array" sdk/.swiftpm/xcode/package.xcworkspace/../../../.fake.plist 2>/dev/null || true
        shell: bash

      - name: Create XCFramework
        working-directory: scripts
        run: bash build_xcframework.sh

      - name: Upload XCFramework artifact
        uses: actions/upload-artifact@v4
        with:
          name: HSBCAOSDKCore.xcframework
          path: dist/HSBCAOSDKCore.xcframework

      - name: Create GitHub Release (on tag)
        if: startsWith(github.ref, 'refs/tags/v')
        uses: softprops/action-gh-release@v2
        with:
          files: |
            dist/HSBCAOSDKCore.xcframework
            README.md
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

> this: builds & tests the Swift package, produces an **XCFramework**, uploads it as a build artifact, and attaches it to **tagged releases** (`v1.0.0`, etc.).

* * *

2) Privacy Manifest (iOS 17+)
=============================

**File:** `sdk/Sources/HSBCAOSDKCore/PrivacyInfo.xcprivacy`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" 
  "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <!-- No tracking -->
  <key>NSPrivacyTracking</key>
  <false/>

  <!-- SDK does not collect data; partner may opt-in to privacy-safe events -->
  <key>NSPrivacyCollectedDataTypes</key>
  <array/>

  <!-- SDK does not link data to user identity -->
  <key>NSPrivacyCollectedDataTypesLinkedToUser</key>
  <array/>

  <!-- SDK does not use required reason APIs for tracking -->
  <key>NSPrivacyTrackingDomains</key>
  <array/>
</dict>
</plist>
```

> keep this file in the **SDK target** so it’s embedded for partners automatically.

* * *

3) ATS (App Transport Security) + App-Bound Domains
===================================================

**File:** `examples/PartnerDemoApp/Info.plist` (add these keys)

```xml
<key>NSAppTransportSecurity</key>
<dict>
  <key>NSAllowsArbitraryLoads</key><false/>
  <key>NSExceptionDomains</key>
  <dict>
    <key>sandbox.partner.hsbc.com</key>
    <dict>
      <key>NSIncludesSubdomains</key><true/>
      <key>NSExceptionAllowsInsecureHTTPLoads</key><false/>
      <key>NSTemporaryExceptionMinimumTLSVersion</key><string>TLSv1.2</string>
      <key>NSThirdPartyExceptionRequiresForwardSecrecy</key><true/>
    </dict>
    <key>ao.sandbox.partner.hsbc.com</key>
    <dict>
      <key>NSIncludesSubdomains</key><true/>
      <key>NSExceptionAllowsInsecureHTTPLoads</key><false/>
      <key>NSTemporaryExceptionMinimumTLSVersion</key><string>TLSv1.2</string>
      <key>NSThirdPartyExceptionRequiresForwardSecrecy</key><true/>
    </dict>
  </dict>
</dict>

<!-- WKAppBoundDomains restricts WKWebView to bank domains (tighten later) -->
<key>WKAppBoundDomains</key>
<array>
  <string>sandbox.partner.hsbc.com</string>
  <string>ao.sandbox.partner.hsbc.com</string>
</array>
```

> when you switch from the local `ao_test.html` to real AO pages, this protects network policy.

* * *

4) ASWebAuthenticationSession prototype (OIDC step)
===================================================

Add a small auth helper to **optionally** perform OIDC login via system browser **before** loading AO pages inside WKWebView.

**File:** `sdk/Sources/HSBCAOSDKCore/OIDCAuth.swift`

```swift
import Foundation
import AuthenticationServices

final class OIDCAuth: NSObject {
    private var session: ASWebAuthenticationSession?
    private var callbackURLScheme: String

    init(redirectScheme: String) {
        self.callbackURLScheme = redirectScheme
    }

    func authorize(authURL: URL, completion: @escaping (Result<URL, Error>) -> Void) {
        let session = ASWebAuthenticationSession(
            url: authURL,
            callbackURLScheme: callbackURLScheme
        ) { callbackURL, error in
            if let url = callbackURL {
                completion(.success(url))
            } else {
                completion(.failure(error ?? NSError(domain: "OIDC", code: -1)))
            }
        }
        session.presentationContextProvider = self
        session.prefersEphemeralWebBrowserSession = true
        self.session = session
        session.start()
    }
}

extension OIDCAuth: ASWebAuthenticationPresentationContextProviding {
    func presentationAnchor(for session: ASWebAuthenticationSession) -> ASPresentationAnchor {
        ASPresentationAnchor()
    }
}
```

**(optional) wire it in**: in `HSBCAOSdk.startAccountOpening`, call `OIDCAuth.authorize(authURL: …)` first (against your test issuer) and, on success, proceed to present `HSBCAOViewController`. For MVP you can skip until you have a real issuer.

* * *

5) Mock Partner APIs (Prefill + Status) — Node/Express
======================================================

**Directory:** `mock/`

**File:** `mock/package.json`

```json
{
  "name": "hsbc-mock-partner",
  "version": "1.0.0",
  "type": "module",
  "scripts": {
    "start": "node server.js"
  },
  "dependencies": {
    "dotenv": "^16.4.5",
    "express": "^4.19.2",
    "helmet": "^7.1.0",
    "morgan": "^1.10.0",
    "uuid": "^9.0.1"
  }
}
```

**File:** `mock/.env.example`

```
PORT=8081
MOCK_LATENCY_MS=150
```

**File:** `mock/server.js`

```js
import express from "express";
import helmet from "helmet";
import morgan from "morgan";
import { config as dotenv } from "dotenv";
import { randomUUID } from "crypto";

dotenv();
const app = express();
app.use(helmet());
app.use(express.json());
app.use(morgan("tiny"));

const PORT = process.env.PORT || 8081;
const SLEEP = parseInt(process.env.MOCK_LATENCY_MS || "0", 10);

// simulate async latency
const wait = (ms) => new Promise((r) => setTimeout(r, ms));

/**
 * POST /partner/v1/prefill
 * Body: { consent:{…}, applicant:{…}, product, trackingContext? }
 * Returns: { prefillToken, expiresIn }
 */
app.post("/partner/v1/prefill", async (req, res) => {
  await wait(SLEEP);
  const prefillToken = `ptok_${randomUUID()}`;
  return res.json({ prefillToken, expiresIn: 900 });
});

/**
 * GET /partner/v1/applications/:id/status
 * Returns one of: PENDING | APPROVED | REJECTED | MORE_INFO
 */
app.get("/partner/v1/applications/:id/status", async (req, res) => {
  await wait(SLEEP);
  const { id } = req.params;
  const states = ["PENDING", "APPROVED", "REJECTED", "MORE_INFO"];
  const status = states[id.length % states.length];
  return res.json({ status, updatedAt: new Date().toISOString(), nextActions: [] });
});

app.listen(PORT, () => console.log(`Mock Partner API running on :${PORT}`));
```

**Run it locally**

```bash
cd mock
cp .env.example .env
npm i
npm start
# -> http://localhost:8081/partner/v1/prefill
```

> use this mock to drive your **demo prefill** flow and later replace it with a real partner server.

* * *

6) Postman collection (Prefill + Status)
========================================

**File:** `postman/HSBC-Partner-APIs.postman_collection.json`

```json
{
  "info": { "name": "HSBC Partner APIs (Mock)", "schema": "https://schema.getpostman.com/json/collection/v2.1.0/collection.json" },
  "item": [
    {
      "name": "Prefill (Mock)",
      "request": {
        "method": "POST",
        "header": [{ "key": "Content-Type", "value": "application/json" }],
        "url": { "raw": "http://localhost:8081/partner/v1/prefill", "protocol": "http", "host": ["localhost"], "port": "8081", "path": ["partner","v1","prefill"] },
        "body": {
          "mode": "raw",
          "raw": "{\n  \"partnerId\": \"ctrip_cn\",\n  \"consent\": {\"type\":\"HSBC_ACCOUNT_OPENING\",\"version\":\"2025-01\",\"timestamp\":\"2025-10-29T00:00:00Z\",\"captureChannel\":\"ctrip_app\",\"evidenceId\":\"consent-uuid-123\"},\n  \"applicant\": {\"fullName\": {\"given\":\"Wei\",\"family\":\"Zhang\"},\"dob\":\"1992-08-12\",\"nationality\":\"CN\",\"idDocument\": {\"type\":\"CN_ID\",\"number\":\"X1234567\",\"expiry\":\"2034-08-12\"},\"mobile\":\"+8613812345678\"},\n  \"product\": \"HSBC_CN_RP_CURR\",\n  \"trackingContext\": {\"partnerUserId\":\"u-001\"}\n}"
        }
      }
    },
    {
      "name": "Application Status (Mock)",
      "request": {
        "method": "GET",
        "url": { "raw": "http://localhost:8081/partner/v1/applications/APP-TEST-123/status", "protocol": "http", "host": ["localhost"], "port": "8081", "path": ["partner","v1","applications","APP-TEST-123","status"] }
      }
    }
  ]
}
```

* * *

7) Release helper script
========================

**File:** `scripts/make_release.sh`

```bash
#!/usr/bin/env bash
set -euo pipefail

if [ $# -ne 1 ]; then
  echo "Usage: $0 vMAJOR.MINOR.PATCH (e.g., v1.0.0)"
  exit 1
fi

TAG="$1"

# build xcframework
bash "$(dirname "$0")/build_xcframework.sh"

# create tag & push
git add -A
git commit -m "chore(release): ${TAG}" || true
git tag -a "${TAG}" -m "Release ${TAG}"
git push origin main --tags

echo "Tag ${TAG} pushed. CI will publish artifacts to the GitHub Release."
```

* * *

8) Copilot prompts (paste these to generate missing glue quickly)
=================================================================

*   **Wire OIDC prototype**  
    “In `HSBCAOSdk.startAccountOpening`, before presenting `HSBCAOViewController`, instantiate `OIDCAuth` with `redirectScheme`. Call `authorize(authURL:)` using a sample `URL(string: 'https://id.sandbox.partner.hsbc.com/oauth2/authorize?...')!`. On success, proceed to present the WKWebView; on failure, call completion with `.failed`.”
*   **Switch from local HTML to real AO URL**  
    “Modify `loadAOTestPage()` to build a `URLRequest` to `https://ao.sandbox.partner.hsbc.com/web/start?prefillToken=...&product=...&locale=...&campaignId=...`. Add custom header `X-HSBC-Partner-Id`, and handle navigation delegate failures cleanly.”
*   **Add simple jailbreak/root signals**  
    “Create a `DeviceIntegrity.swift` helper with static checks for suspicious file paths, write permissions to system dirs, and URL scheme handling. Expose a method that returns a boolean verdict.”

* * *

9) Quick test plan
==================

1.  **Unit tests** (already added) — run `swift test`
2.  **Manual demo** — in PartnerDemoApp:
    *   Initialize SDK → Start AO → Play with the test page buttons; verify events and result.
3.  **Mock API** — run the Node server, hit Prefill & Status via Postman; wire your demo to consume `prefillToken` from `localhost:8081`.
4.  **CI** — push to `main`; verify the workflow builds, tests, and uploads the XCFramework artifact.
5.  **Tag** — run `bash scripts/make_release.sh v0.1.0`; check GitHub Release includes the XCFramework.

* * *

10) What to do next (when MVP compiles)
=======================================

*   Replace `ao_test.html` with the **real AO start URL** and enable ATS/WKAppBoundDomains.
*   Introduce **pinning** for any native requests (CryptoKit + TrustKit or your own evaluator).
*   Add **PrivacyInfo.xcprivacy** to your SDK target’s **Copy Bundle Resources** (Xcode does this if the file lives under the target).
*   Start migrating the container to **ASWebAuthenticationSession** for OIDC login + **WKWebView** only for the bridge steps (hybrid model).
*   Add **error catalog** mapping to align with HSBC’s server codes.

* * *

if you want, i can also generate:

*   a **SwiftPM plugin** that automates XCFramework creation,
*   a **Fastlane lane** for local “build & zip & notarize” (if you plan to distribute outside the App Store),
*   or a **tiny iOS UI test** that presses the buttons in `ao_test.html` via WebKit automation.

just tell me which and i’ll drop those files too.

## Prompt:
please

## Response:
Thought for 44s

you got it — here are the three extras you asked for, ready to paste into your repo:

*   a **SwiftPM plugin** to one-command build an XCFramework,
*   a **Fastlane lane** to build & zip (and notes + template for notarization, if you ever ship a macOS framework),
*   a tiny **iOS UI test** that drives the WKWebView test page end-to-end.

* * *

1) SwiftPM plugin — “build xcframework” (one command)
=====================================================

**Repo layout additions**

```
hsbc-ao-sdk-ios/
└─ sdk/
   ├─ Package.swift                      ← update (below)
   └─ Plugins/
      └─ BuildXCFrameworkPlugin/
         ├─ PackagePlugin.swift          ← NEW
         └─ Resources/
             └─ build_xcframework.sh     ← NEW
```

**sdk/Package.swift — add the plugin target**

```swift
// swift-tools-version: 5.9
import PackageDescription

let package = Package(
    name: "HSBCAOSDKCore",
    defaultLocalization: "en",
    platforms: [.iOS(.v14)],
    products: [
        .library(name: "HSBCAOSDKCore", targets: ["HSBCAOSDKCore"])
    ],
    targets: [
        .target(
            name: "HSBCAOSDKCore",
            resources: [
                .process("Sources/HSBCAOSDKCore/Resources/ao_test.html"),
                .process("Sources/HSBCAOSDKCore/PrivacyInfo.xcprivacy")
            ]
        ),
        .testTarget(
            name: "HSBCAOSDKCoreTests",
            dependencies: ["HSBCAOSDKCore"]
        ),
        // 👇 plugin target
        .plugin(
            name: "BuildXCFrameworkPlugin",
            capability: .command(
                intent: .custom(verb: "build-xcframework",
                                 description: "Builds an XCFramework for HSBCAOSDKCore"),
                permissions: [
                    .writeToPackageDirectory(reason: "Create dist/ and XCFramework output")
                ]
            ),
            path: "Plugins/BuildXCFrameworkPlugin"
        )
    ]
)
```

**sdk/Plugins/BuildXCFrameworkPlugin/PackagePlugin.swift**

```swift
import PackagePlugin
import Foundation

@main
struct BuildXCFrameworkPlugin: CommandPlugin {
    func performCommand(context: PluginContext, arguments: [String]) async throws {
        let packageDir = context.package.directory.string
        let script = context.pluginWorkDirectory.appending(subpath: "build_xcframework.sh")
        // Copy script from plugin resources to work dir (first run)
        try copyScriptIfNeeded(from: "build_xcframework.sh",
                               to: script.string,
                               resources: context.pluginWorkDirectory)

        let process = Process()
        process.currentDirectoryURL = URL(fileURLWithPath: packageDir).deletingLastPathComponent()
        process.executableURL = URL(fileURLWithPath: "/bin/bash")
        process.arguments = [script.string]
        try process.run()
        process.waitUntilExit()
        guard process.terminationStatus == 0 else {
            throw PluginError("build_xcframework.sh failed with exit code \(process.terminationStatus)")
        }
        print("✅ XCFramework created under dist/HSBCAOSDKCore.xcframework")
    }

    private func copyScriptIfNeeded(from resource: String, to dest: String, resources: Path) throws {
        let fm = FileManager.default
        if fm.fileExists(atPath: dest) { return }
        let bundleDir = resources.string // Plugin work dir (contains resources copied by SwiftPM)
        let src = (bundleDir as NSString).appendingPathComponent(resource)
        try fm.copyItem(atPath: src, toPath: dest)
        try chmod(path: dest, mode: 0o755)
    }

    private func chmod(path: String, mode: Int32) throws {
        if chmod(path, mode) != 0 {
            throw PluginError("chmod failed for \(path)")
        }
    }
}

struct PluginError: Error, CustomStringConvertible {
    let description: String
    init(_ d: String) { description = d }
}
```

**sdk/Plugins/BuildXCFrameworkPlugin/Resources/build\_xcframework.sh**

```bash
#!/usr/bin/env bash
set -euo pipefail

ROOT="$(cd "$(dirname "${BASH_SOURCE[0]}")"/../../.. && pwd)"
OUT="$ROOT/../../dist"
SCHEME="HSBCAOSDKCore"

mkdir -p "$OUT"

# Ensure Xcode project for SPM exists (first run may generate it)
xcodebuild -list -workspace /dev/null >/dev/null 2>&1 || true

echo "🔨 Archiving (device)…"
xcodebuild archive \
  -scheme "$SCHEME" \
  -destination "generic/platform=iOS" \
  -archivePath "$OUT/ios_devices.xcarchive" \
  SKIP_INSTALL=NO BUILD_LIBRARY_FOR_DISTRIBUTION=YES

echo "🔨 Archiving (simulator)…"
xcodebuild archive \
  -scheme "$SCHEME" \
  -destination "generic/platform=iOS Simulator" \
  -archivePath "$OUT/ios_sim.xcarchive" \
  SKIP_INSTALL=NO BUILD_LIBRARY_FOR_DISTRIBUTION=YES

echo "📦 Creating XCFramework…"
xcodebuild -create-xcframework \
  -framework "$OUT/ios_devices.xcarchive/Products/Library/Frameworks/$SCHEME.framework" \
  -framework "$OUT/ios_sim.xcarchive/Products/Library/Frameworks/$SCHEME.framework" \
  -output "$OUT/$SCHEME.xcframework"

echo "✅ Done: $OUT/$SCHEME.xcframework"
```

**Run it (from `sdk/`)**

```bash
swift package plugin build-xcframework --allow-writing-to-package-directory
```

* * *

2) Fastlane lane — build & zip (and notarization template)
==========================================================

> iOS **XCFrameworks do not require notarization**. Notarization applies to macOS binaries. I’m giving you:
> 
> *   a **working build & zip lane** for your iOS XCFramework,
> *   a **template notarize lane** (commented) you can adapt if you ever build a macOS framework/tool.
>     

**Repo additions**

```
hsbc-ao-sdk-ios/
└─ ci/
   └─ fastlane/
      ├─ Fastfile             ← NEW
      └─ Appfile              ← NEW (empty or metadata)
```

**ci/fastlane/Fastfile**

```ruby
default_platform(:ios)

platform :ios do
  desc "Build and zip HSBCAOSDKCore.xcframework"
  lane :build_zip_xcframework do
    root = File.expand_path('../..', __dir__)
    Dir.chdir("#{root}/scripts") do
      sh "bash build_xcframework.sh"
    end
    Dir.chdir("#{root}/dist") do
      sh "rm -f HSBCAOSDKCore.xcframework.zip"
      sh "zip -r HSBCAOSDKCore.xcframework.zip HSBCAOSDKCore.xcframework"
      sh "shasum -a 256 HSBCAOSDKCore.xcframework.zip > HSBCAOSDKCore.xcframework.zip.sha256"
    end
    UI.success "XCFramework zipped under dist/"
  end

  # --------- macOS notarization template (NOT needed for iOS) ----------
  # lane :notarize_macos_artifact do |options|
  #   zip_path = options[:zip] || "../dist/MyMacBinary.zip"
  #   apple_id = ENV["APPLE_ID"]
  #   team_id  = ENV["APPLE_TEAM_ID"]
  #   itc_pass = ENV["APP_SPECIFIC_PASSWORD"] # app-specific password
  #
  #   # Submit for notarization (Xcode 13+: notarytool is recommended)
  #   sh "xcrun notarytool submit '#{zip_path}' --apple-id '#{apple_id}' --team-id '#{team_id}' --password '#{itc_pass}' --wait"
  #   UI.success "Notarization complete."
  # end
end
```

**ci/fastlane/Appfile**

```ruby
# Reserved for future metadata; lanes above use env vars.
```

**Run locally**

```bash
cd ci/fastlane
bundle exec fastlane build_zip_xcframework
# (or `fastlane build_zip_xcframework` if fastlane is installed globally)
```

> For macOS notarization (if ever needed), set env vars `APPLE_ID`, `APPLE_TEAM_ID`, `APP_SPECIFIC_PASSWORD` and uncomment the lane.

* * *

3) Tiny iOS UI test — drives the WKWebView test page
====================================================

**Add UI test target** to `examples/PartnerDemoApp`:

*   In Xcode: _File → New → Target → iOS UI Testing Bundle_  
    Name it: **PartnerDemoAppUITests**

**examples/PartnerDemoApp/PartnerDemoAppUITests/PartnerDemoAppUITests.swift**

```swift
import XCTest

final class PartnerDemoAppUITests: XCTestCase {
    var app: XCUIApplication!

    override func setUp() {
        continueAfterFailure = false
        app = XCUIApplication()
        app.launch()
    }

    func testAOEndToEndMVP() {
        // 1) Initialize SDK
        let initButton = app.buttons["Initialize SDK"]
        XCTAssertTrue(initButton.waitForExistence(timeout: 5))
        initButton.tap()

        // 2) Start AO
        let startButton = app.buttons["Start Account Opening"]
        XCTAssertTrue(startButton.waitForExistence(timeout: 5))
        startButton.tap()

        // 3) Interact with WKWebView content (the test HTML page)
        // WKWebView exposes accessibility elements when buttons have text.
        // Find by label text we used in ao_test.html:
        let readyBtn = app.webViews.buttons["Emit READY"]
        XCTAssertTrue(readyBtn.waitForExistence(timeout: 5))
        readyBtn.tap()

        let livenessBtn = app.webViews.buttons["Request Liveness"]
        XCTAssertTrue(livenessBtn.waitForExistence(timeout: 5))
        livenessBtn.tap()

        // The alert comes from `alert()` in the page
        let okAlertButton = app.alerts.buttons.firstMatch
        if okAlertButton.waitForExistence(timeout: 3) { okAlertButton.tap() }

        let docBtn = app.webViews.buttons["Capture Document"]
        XCTAssertTrue(docBtn.waitForExistence(timeout: 5))
        docBtn.tap()
        if okAlertButton.waitForExistence(timeout: 3) { okAlertButton.tap() }

        // Complete the flow
        let completeBtn = app.webViews.buttons["Emit Completed"]
        XCTAssertTrue(completeBtn.waitForExistence(timeout: 5))
        completeBtn.tap()

        // Back in the host app, confirm status label updates
        let statusText = app.staticTexts.containing(NSPredicate(format: "label CONTAINS[c] 'Pending' OR label CONTAINS[c] 'Completed'"))
        XCTAssertTrue(statusText.element.waitForExistence(timeout: 5))
    }
}
```

**Enable UI testing scheme**

*   In Xcode, select **PartnerDemoApp** workspace.
*   Product → Test (⌘U). The test should launch simulator and run through the HTML buttons.

* * *

Quick sanity checklist
----------------------

*    `swift package plugin build-xcframework --allow-writing-to-package-directory` succeeds and creates `dist/HSBCAOSDKCore.xcframework`
*    `fastlane build_zip_xcframework` zips and SHA-256 checksums the XCFramework
*    UI test runs in simulator and updates the status label after “Emit Completed”
*    CI workflow uploads the XCFramework artifact on PRs/pushes

* * *

if you want, i can also add:

*   a **SwiftPM command plugin** to zip + checksum as part of the same `build-xcframework` verb,
*   a **GitHub Release** notes generator that pulls version + changelog,
*   a **CocoaPods podspec** stub so partners can consume the SDK via Pods as well.

say the word and i’ll drop those too.

## Prompt:
yes , please

## Response:
Perfect — here’s the final layer for your iOS SDK build-and-release toolchain:

✅ **SwiftPM plugin** (extends the existing one to include zip + checksum)  
✅ **GitHub release notes generator**  
✅ **CocoaPods podspec** stub for partner consumption

* * *

🧩 1) Extend the SwiftPM Plugin — zip & checksum in one command
---------------------------------------------------------------

**File:** `sdk/Plugins/BuildXCFrameworkPlugin/PackagePlugin.swift` (replace the previous one)

```swift
import PackagePlugin
import Foundation

@main
struct BuildXCFrameworkPlugin: CommandPlugin {
    func performCommand(context: PluginContext, arguments: [String]) async throws {
        let packageDir = context.package.directory.string
        let root = URL(fileURLWithPath: packageDir).deletingLastPathComponent().path
        let outDir = "\(root)/dist"
        let scheme = "HSBCAOSDKCore"

        print("🔨 Building XCFramework…")
        try run("bash", args: ["scripts/build_xcframework.sh"], at: root)

        print("📦 Zipping & checksum…")
        let xcframeworkPath = "\(outDir)/\(scheme).xcframework"
        let zipPath = "\(outDir)/\(scheme).xcframework.zip"
        try? FileManager.default.removeItem(atPath: zipPath)
        try run("zip", args: ["-r", zipPath, "\(scheme).xcframework"], at: outDir)
        try run("shasum", args: ["-a", "256", zipPath, ">", "\(zipPath).sha256"], at: outDir)

        print("✅ Done: \(zipPath)")
    }

    private func run(_ cmd: String, args: [String], at dir: String) throws {
        let process = Process()
        process.executableURL = URL(fileURLWithPath: "/usr/bin/env")
        process.currentDirectoryURL = URL(fileURLWithPath: dir)
        process.arguments = [cmd] + args
        try process.run()
        process.waitUntilExit()
        guard process.terminationStatus == 0 else {
            throw NSError(domain: "BuildXCFrameworkPlugin", code: Int(process.terminationStatus))
        }
    }
}
```

**Usage:**

```bash
swift package plugin build-xcframework --allow-writing-to-package-directory
```

→ builds, zips, and generates a SHA-256 checksum in one command.

* * *

🧾 2) GitHub Release Notes Generator
------------------------------------

**File:** `.github/workflows/release-notes.yml`

```yaml
name: Generate Release Notes

on:
  push:
    tags:
      - "v*"

jobs:
  generate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Extract tag
        id: tag
        run: echo "tag=${GITHUB_REF#refs/tags/}" >> $GITHUB_OUTPUT

      - name: Generate changelog
        run: |
          echo "## Release ${{ steps.tag.outputs.tag }}" > release_notes.md
          echo "" >> release_notes.md
          git log -10 --pretty=format:'- %s (%h)' >> release_notes.md
          echo "" >> release_notes.md
          echo "Artifacts:" >> release_notes.md
          echo "- \`HSBCAOSDKCore.xcframework.zip\`" >> release_notes.md

      - name: Publish release
        uses: softprops/action-gh-release@v2
        with:
          body_path: release_notes.md
          files: |
            dist/HSBCAOSDKCore.xcframework.zip
            dist/HSBCAOSDKCore.xcframework.zip.sha256
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

> Every tag push (e.g., `v1.0.0`) triggers this workflow, creates release notes from recent commits, and uploads your zipped XCFramework and checksum.

* * *

📦 3) CocoaPods Podspec stub
----------------------------

**File:** `sdk/HSBCAOSDKCore.podspec`

```ruby
Pod::Spec.new do |s|
  s.name             = 'HSBCAOSDKCore'
  s.version          = '1.0.0'
  s.summary          = 'HSBC Partner Account Opening SDK for iOS (Hosted AO Journey)'
  s.description      = <<-DESC
    A lightweight, secure SDK for integrating HSBC-hosted Account Opening journeys
    into partner iOS applications. Supports OIDC login, WKWebView container, and
    native KYC plugin extensions.
  DESC
  s.homepage         = 'https://github.com/hsbc/hsbc-ao-sdk-ios'
  s.license          = { :type => 'Commercial', :text => '© 2025 HSBC. All rights reserved.' }
  s.author           = { 'HSBC Partner Engineering' => 'partner-tech@hsbc.com' }
  s.platform         = :ios, '14.0'
  s.swift_versions   = ['5.9']
  s.source           = { :git => 'https://github.com/hsbc/hsbc-ao-sdk-ios.git', :tag => s.version.to_s }
  s.vendored_frameworks = 'dist/HSBCAOSDKCore.xcframework'
  s.requires_arc     = true
  s.resources        = ['Sources/HSBCAOSDKCore/Resources/**/*']
  s.resource_bundles = {
    'HSBCAOSDKCore' => ['Sources/HSBCAOSDKCore/Resources/**/*']
  }
end
```

**Usage for partners:**

```ruby
pod 'HSBCAOSDKCore', '~> 1.0.0'
```

When you tag a new release, update the version number here (and in your GitHub tag).

* * *

✅ Quick summary of your full toolchain
--------------------------------------

| Layer | Tool | Purpose |
| --- | --- | --- |
| **Local dev** | SwiftPM plugin | `swift package plugin build-xcframework` to build+zip+checksum |
| **CI/CD** | GitHub Actions (`ios-sdk.yml`) | Builds & uploads XCFramework artifact |
| **Release notes** | GitHub Action (`release-notes.yml`) | Auto-generates notes on tag push |
| **Packaging** | Fastlane | Local build+zip lane |
| **Distribution** | CocoaPods podspec | Enables public/partner installation |
| **Verification** | UI Tests | Automated simulator test for MVP |

* * *

Would you like me to now add:

1.  A **Carthage** or **SwiftPM remote binary target** spec (so partners can pull the XCFramework directly via URL), or
2.  A **release README template** that auto-links to the SDK doc, checksum, and integration guide?

## Prompt:
please

## Response:
Thought for 50s

Awesome — here are both deliverables you asked for:

* * *

1) SwiftPM **Remote Binary Target** (pull the XCFramework via URL)
==================================================================

### A) Publisher setup (your repo)

1.  Make sure your CI uploads:
    *   `dist/HSBCAOSDKCore.xcframework.zip`
    *   `dist/HSBCAOSDKCore.xcframework.zip.sha256`
2.  Compute checksum (already in your pipeline, but here’s the local command):

```bash
swift package compute-checksum dist/HSBCAOSDKCore.xcframework.zip
# -> paste the 64-char SHA256 in the manifest below
```

3.  Create a tiny **wrapper Package.swift** (so partners can add by URL). Put this in a separate repo like `hsbc/hsbc-aosdk-ios-binary` (or under `/binary` branch):

**Package.swift**

```swift
// swift-tools-version: 5.9
import PackageDescription

let package = Package(
    name: "HSBCAOSDKCoreBinary",
    platforms: [.iOS(.v14)],
    products: [
        .library(name: "HSBCAOSDKCore", targets: ["HSBCAOSDKCore"])
    ],
    targets: [
        .binaryTarget(
            name: "HSBCAOSDKCore",
            url: "https://github.com/hsbc/hsbc-ao-sdk-ios/releases/download/v1.0.0/HSBCAOSDKCore.xcframework.zip",
            checksum: "<PASTE_SWIFT_COMPUTE_CHECKSUM_RESULT>"
        )
    ]
)
```

> On each release/tag, bump the version, URL, and checksum.

### B) Partner usage

**Xcode → Add Package Dependency**:  
`https://github.com/hsbc/hsbc-aosdk-ios-binary.git` (choose version `1.0.0+`)

or in **Package.swift**:

```swift
.package(url: "https://github.com/hsbc/hsbc-aosdk-ios-binary.git", from: "1.0.0")
```

* * *

2) **Carthage** Binary Distribution
===================================

Carthage supports **binary-only** frameworks via a JSON “binary project” descriptor.

### A) Publisher setup

Add a JSON file to your repo (e.g., at:  
`https://raw.githubusercontent.com/hsbc/hsbc-ao-sdk-ios/main/HSBCAOSDKCore.json`).

**HSBCAOSDKCore.json**

```json
{
  "1.0.0": {
    "url": "https://github.com/hsbc/hsbc-ao-sdk-ios/releases/download/v1.0.0/HSBCAOSDKCore.xcframework.zip",
    "checksum": "SHA256: <PASTE_SHA256_OF_ZIP>"
  },
  "1.1.0": {
    "url": "https://github.com/hsbc/hsbc-ao-sdk-ios/releases/download/v1.1.0/HSBCAOSDKCore.xcframework.zip",
    "checksum": "SHA256: <PASTE_SHA256_OF_ZIP>"
  }
}
```

*   The `checksum` supports `SHA256: <value>` form.
*   Update this file for every release.

### B) Partner usage

**Cartfile**

```
binary "https://raw.githubusercontent.com/hsbc/hsbc-ao-sdk-ios/main/HSBCAOSDKCore.json" ~> 1.0
```

Then:

```bash
carthage update --use-xcframeworks
```

* * *

3) **Release README** Template (drop into each GitHub Release & repo root)
==========================================================================

Save as `RELEASE.md` (and paste into the GitHub Release body).

````markdown
# HSBCAOSDKCore iOS – vX.Y.Z

> HSBC Partner Account-Opening SDK (Hosted AO Journey) – iOS binary distribution.

## 📦 What’s in this release
- `HSBCAOSDKCore.xcframework.zip`
- `HSBCAOSDKCore.xcframework.zip.sha256`
- Changelog (below)

## ✅ Integrity (verify before use)
```bash
# expected (from .sha256 file):
cat HSBCAOSDKCore.xcframework.zip.sha256

# actual:
shasum -a 256 HSBCAOSDKCore.xcframework.zip
# compare – they must match
````

📥 Installation
---------------

### Option A — Swift Package Manager (remote binary)

**Add Package**: `https://github.com/hsbc/hsbc-aosdk-ios-binary.git` (version ≥ X.Y.Z)

Or in `Package.swift`:

```swift
.package(url: "https://github.com/hsbc/hsbc-aosdk-ios-binary.git", from: "X.Y.Z")
```

### Option B — CocoaPods

`Podfile`:

```ruby
pod 'HSBCAOSDKCore', '~> X.Y.Z'
```

### Option C — Carthage (binary)

`Cartfile`:

```
binary "https://raw.githubusercontent.com/hsbc/hsbc-ao-sdk-ios/main/HSBCAOSDKCore.json" ~> X.Y
```

```bash
carthage update --use-xcframeworks
```

### Option D — Direct download

1.  Download `HSBCAOSDKCore.xcframework.zip`
2.  Verify SHA256 (see above)
3.  Unzip and drag `HSBCAOSDKCore.xcframework` into your app target (Embed & Sign)

🛠 Minimum requirements
-----------------------

*   iOS 14.0+
*   Xcode 15+
*   Swift 5.9
*   ATS enabled (TLS 1.2+, HSBC domains only)
*   WKAppBoundDomains configured for HSBC AO domains

🔐 Security notes
-----------------

*   No tracking, no PII stored by the SDK
*   TLS + certificate pinning (for native calls)
*   OIDC + PKCE recommended via ASWebAuthenticationSession
*   Device integrity checks (App Attest/DeviceCheck) optional
*   Privacy manifest included

🚀 Getting started (snippet)
----------------------------

```swift
import HSBCAOSDKCore

let cfg = HSBCInitConfig(
  environment: .sandbox,
  partnerId: "your_partner_id",
  clientId: "your_client_id",
  redirectScheme: "yourapp",
  locale: "zh-CN"
)
HSBCAOSdk.initialize(config: cfg)
HSBCAOSdk.setEventListener { name, attrs in print(name, attrs) }

HSBCAOSdk.startAccountOpening(from: topController,
  params: AOStartParams(prefillToken: "<from your backend>", productCode: "HSBC_CN_RP_CURR")) { result in
    print(result)
}
```

🧩 Changelog (X.Y.Z)
--------------------

*   Feature: …
*   Fix: …
*   Security: …
*   SDK size: … MB

📚 Docs
-------

*   Integration guide: `/docs/ios/getting-started.md`
*   Security & compliance summary: `/docs/security.md`
*   AO bridge protocol: `/docs/ao-js-bridge.md`

🆘 Support
----------

*   partner-tech@hsbc.com (include SDK version, iOS version, device, steps)

```

---

## Where to put these files

- **SwiftPM binary wrapper**: new repo (e.g., `hsbc-ao-sdk-ios-binary`).
- **Carthage JSON**: in your main SDK repo root, published via `raw.githubusercontent.com`.
- **RELEASE.md**: in the main SDK repo root *and* pasted into each GitHub Release body.

---

If you want, I can also generate:
- a **docs/ios/getting-started.md** and **docs/ao-js-bridge.md** skeleton,  
- a **CHANGELOG.md** template following Keep a Changelog + Semantic Versioning.
```

## Prompt:
please review the above design we can found that we can generate and build the frame work at first. this framework SDk suitable for any kind of business scenario + associated openAPI.  currently we use account opening as example, base in this framework, if we want to build a SDK for money transfer. we just need provide the pages of money transfer business journey, and provide the end poit for open api set with spec, and also process sequence . please generate or revise  low level the design for this approch

## Response:
Love this direction. Below is a **low-level, implementation-ready design** for a **generic HSBC Partner SDK framework** that supports **any business journey** (e.g., Account Opening, Money Transfer) by swapping in pages + a declarative journey manifest + OpenAPI bindings—without changing the SDK core.

I’ve kept it concrete for iOS (Swift), but the abstractions map cleanly to Android/Web.

* * *

HSBC Partner SDK – Generic Journey Framework (Low-Level Design)
===============================================================

0) Goals
--------

*   **One SDK, many journeys**: AO, Money Transfer (MT), Card Apply, Address Update, etc.
*   **Bank-owned pages** (hosted) with **native bridge** for device capabilities.
*   **Declarative orchestration**: a **Journey Manifest** drives step flow, API calls, error handling.
*   **OpenAPI-bound**: endpoints, schemas, auth & retry derived from OpenAPI specs.
*   **Security unchanged**: OIDC+PKCE, pinning, attestation, JWS/JWE, consent capture.
*   **Minimal partner work**: integrate SDK once; select journey with parameters.

* * *

1) Key Concepts
---------------

*   **Journey**: a named business flow, e.g., `"account_opening"`, `"money_transfer"`.
*   **Journey Manifest (JM)**: signed JSON that declares steps, page URLs, bridge invocations, and OpenAPI call bindings.
*   **OpenAPI Bundle (OAB)**: bank-published OpenAPI (YAML/JSON) with operationIds used in the manifest.
*   **Runtime Engine (RE)**: SDK component that loads JM+OAB, enforces policy, navigates steps, executes bound API calls, and coordinates the JS bridge.
*   **Bridge**: secure message bus between bank pages and the SDK native code.
*   **Plugins**: optional native modules (camera/NFC/liveness, biometrics, file picker, geo, etc.).

* * *

2) Public SDK Interfaces (Swift)
--------------------------------

```swift
public struct HSBCInitConfig {
    public enum Environment { case sandbox, production }
    public let environment: Environment
    public let partnerId: String
    public let clientId: String
    public let redirectScheme: String
    public let locale: String
    public let remoteConfigURL: URL?      // signed config (per region)
    public let featureFlags: [String: Bool]

    // … init(…)
}

public struct JourneyStartParams {
    public let journeyId: String          // e.g. "account_opening", "money_transfer"
    public let contextToken: String       // prefill token / session token / transfer token
    public let productCode: String?
    public let campaignId: String?
    public let trackingContext: [String:String]
}

public enum JourneyResult {
    case completed(payload: [String: Any])     // e.g., {applicationId}, {transferId}
    case pending(payload: [String: Any])
    case cancelled
    case failed(code: String, message: String, recoverable: Bool)
}

public typealias JourneyEventListener = (_ name: String, _ attributes: [String: Any]) -> Void

public enum HSBCPartnerSdk {
    public static func initialize(config: HSBCInitConfig)
    public static func setEventListener(_ listener: @escaping JourneyEventListener)

    /// Generic entry for any journey (AO, MT, etc.)
    public static func startJourney(from presenter: UIViewController,
                                    params: JourneyStartParams,
                                    completion: @escaping (JourneyResult) -> Void)
}
```

* * *

3) Runtime Architecture (SDK Internal)
--------------------------------------

```
HSBCPartnerSdk.startJourney()
   └─ RuntimeEngine
      ├─ ConfigLoader (local + signed remote)
      ├─ ManifestLoader (fetch JM for journeyId; verify JWS)
      ├─ OpenAPIResolver (load OAB; map operationIds → REST calls)
      ├─ PolicyGuard (pinning, ATS, origin checks, attestation)
      ├─ ContainerController (ASWebAuth + WKWebView hybrid)
      │   └─ Bridge (JS <-> Native)
      ├─ StateMachine (Steps, Transitions, Guards, Effects)
      ├─ ApiClient (OpenAPI execution, auth headers, retries)
      └─ Plugins (KYC, NFC, Biometrics, Files, Camera, etc.)
```

* * *

4) Journey Manifest (JM) – **Signed JSON** (bank-authored)
----------------------------------------------------------

> **Purpose**: Declaratively defines the flow, UI pages, bridge calls, API calls, transitions, and error handling.

```json
{
  "manifestVersion": "1.0.0",
  "journeyId": "money_transfer",
  "oapiBundle": "https://ao.sandbox.partner.hsbc.com/oapi/transfer-v1.yaml",
  "locale": ["zh-CN", "en-HK"],
  "startStep": "mt_intro",
  "steps": {
    "mt_intro": {
      "type": "web",
      "url": "https://ao.sandbox.partner.hsbc.com/mt/intro",
      "on": {
        "next": { "to": "mt_recipient" },
        "cancel": { "to": "end_cancel" }
      }
    },
    "mt_recipient": {
      "type": "web",
      "url": "https://ao.sandbox.partner.hsbc.com/mt/recipient",
      "bindings": [
        { "onEvent": "validateRecipient",
          "call": { "operationId": "ValidateRecipient", "argsFrom": "event.payload" },
          "onSuccessEmit": "recipientValid",
          "onErrorEmit": "recipientInvalid" }
      ],
      "on": {
        "recipientValid": { "to": "mt_amount" },
        "back": { "to": "mt_intro" }
      }
    },
    "mt_amount": {
      "type": "web",
      "url": "https://ao.sandbox.partner.hsbc.com/mt/amount",
      "on": {
        "proceed": { "to": "mt_review" },
        "back": { "to": "mt_recipient" }
      }
    },
    "mt_review": {
      "type": "web",
      "url": "https://ao.sandbox.partner.hsbc.com/mt/review",
      "bindings": [
        { "onEvent": "calculateFees",
          "call": { "operationId": "CalculateFee", "argsFrom": "event.payload" },
          "onSuccessEmit": "feesReady" }
      ],
      "on": {
        "confirm": { "to": "mt_biometric" },
        "back": { "to": "mt_amount" }
      }
    },
    "mt_biometric": {
      "type": "native",
      "plugin": "biometricAuth",
      "params": { "reason": "Authorize transfer" },
      "on": {
        "success": { "to": "mt_submit" },
        "failure": { "to": "mt_review" }
      }
    },
    "mt_submit": {
      "type": "server",
      "call": { "operationId": "CreateTransfer", "argsFrom": "session" },
      "on": {
        "success": { "to": "end_complete", "emit": { "name": "MT_SUBMITTED" } },
        "error":   { "to": "end_error" }
      }
    },
    "end_complete": { "type": "terminal", "result": { "status": "COMPLETED" } },
    "end_cancel":   { "type": "terminal", "result": { "status": "CANCELLED" } },
    "end_error":    { "type": "terminal", "result": { "status": "FAILED" } }
  },
  "security": {
    "pinning": true,
    "allowedOrigins": ["https://ao.sandbox.partner.hsbc.com"],
    "attestation": { "ios": true }
  },
  "headers": {
    "X-HSBC-Partner-Id": "$partnerId",
    "X-HSBC-Client-Id": "$clientId",
    "X-HSBC-Locale": "$locale",
    "X-HSBC-Context": "$contextToken"
  },
  "signature": "JWS-DETACHED-SIGNATURE"
}
```

*   **type**: `"web"` (hosted page), `"native"` (plugin step), `"server"` (headless API call), `"terminal"`.
*   **bindings**: map **page events** (`onEvent`) to **OpenAPI calls**.
*   **on**: state transitions on events or outcomes.

> The SDK Runtime validates the manifest signature, enforces **allowedOrigins**, and injects **headers** + **query params**.

* * *

5) OpenAPI Bundle (OAB) – Example (Money Transfer)
--------------------------------------------------

```yaml
openapi: 3.0.3
info: { title: HSBC Transfer API, version: 1.0.0 }
servers:
  - url: https://api.sandbox.partner.hsbc.com
paths:
  /transfer/validateRecipient:
    post:
      operationId: ValidateRecipient
      requestBody: { required: true, content: { application/json: { schema: { $ref: '#/components/schemas/Recipient' } } } }
      responses:
        '200': { description: OK, content: { application/json: { schema: { $ref: '#/components/schemas/RecipientValidation' } } } }
  /transfer/calculateFee:
    post:
      operationId: CalculateFee
      requestBody: { required: true, content: { application/json: { schema: { $ref: '#/components/schemas/FeeRequest' } } } }
      responses:
        '200': { description: OK, content: { application/json: { schema: { $ref: '#/components/schemas/FeeResponse' } } } }
  /transfer:
    post:
      operationId: CreateTransfer
      requestBody: { required: true, content: { application/json: { schema: { $ref: '#/components/schemas/TransferRequest' } } } }
      responses:
        '201': { description: Created, content: { application/json: { schema: { $ref: '#/components/schemas/Transfer' } } } }
components:
  schemas:
    Recipient: { type: object, properties: { name: {type: string}, accountNumber: {type: string}, bankCode: {type: string}, country: {type: string} }, required: [name, accountNumber, bankCode, country] }
    RecipientValidation: { type: object, properties: { valid: {type: boolean}, reason: {type: string} }, required: [valid] }
    FeeRequest: { type: object, properties: { amount: {type: number, format: double}, currency: {type: string}, route: {type: string} }, required: [amount, currency] }
    FeeResponse: { type: object, properties: { fee: {type: number, format: double}, rate: {type: number, format: double} }, required: [fee] }
    TransferRequest: { type: object, properties: { recipient: { $ref: '#/components/schemas/Recipient' }, amount: {type: number}, currency: {type: string}, memo: {type: string} }, required: [recipient, amount, currency] }
    Transfer: { type: object, properties: { transferId: {type: string}, status: {type: string}, submittedAt: {type: string, format: date-time} }, required: [transferId, status] }
```

The **Runtime** resolves `operationId` → full HTTP call (method, path, body schema), injects auth headers, and executes with retries.

* * *

6) Bridge Protocol (Page ↔ SDK)
-------------------------------

### Message format (JSON)

```json
// Page → SDK: request
{ "kind":"request", "id":"<uuid>", "name":"performBiometric", "payload": { "reason":"Authorize transfer" } }

// SDK → Page: resolve
{ "kind":"resolve", "id":"<uuid>", "result": { "biometric":"PASS", "ts": "..." }, "sig": "<JWS>" }

// Page → SDK: event (fire-and-forget)
{ "kind":"event", "name":"calculateFees", "payload": { "amount": 500, "currency":"HKD" } }
```

*   All **SDK → Page** payloads are JWS-signed; page verifies signature with HSBC-published key.
*   The SDK **validates origin** and only exposes `hsbcBridge` to **allowedOrigins** from JM.
*   **Standard bridge capabilities**:
    *   `performBiometric`, `captureDocument`, `scanNFC`, `openFilePicker`, `getDeviceInfo`, `getLocale`, `openAppointment`, etc.
    *   `invokeApi(operationId, args)` (optional) – _if HSBC chooses to let the page ask the SDK to call a protected endpoint_; typically we prefer **bindings** in JM.

* * *

7) State Machine (Runtime Engine)
---------------------------------

*   **Node** = step (web/native/server/terminal)
*   **Edge** = transition with **guard** (predicate) and **effect** (emit/result/call)
*   **Event sources**: page events, plugin results, API outcomes, user nav (back/close), timeouts.

Pseudo-types:

```swift
struct Step {
    enum Kind { case web(url: URL), native(plugin: String, params: [String:Any]), server(operationId: String), terminal(result: [String:Any]) }
    var on: [String: Transition]   // eventName -> Transition
    var bindings: [Binding]        // optional page-event to API call
}

struct Transition {
    var to: String?                // next step id
    var emit: BridgeEvent?         // optional event to page/SDK
    var guardExpr: Expr?           // predicate over context/session
}

struct Binding {
    var onEvent: String
    var call: ApiCallBinding       // operationId + arg mapping
    var onSuccessEmit: String?
    var onErrorEmit: String?
}
```

*   **Session context**: shared KV (no PII stored unless policy allows).
*   **Arg mapping** (`argsFrom`) supports **JSONPath** / **key paths** (`event.payload.amount` → request body).

* * *

8) Money Transfer (MT) Journey – Low-Level Example
--------------------------------------------------

### A) Start parameters

```swift
let params = JourneyStartParams(
  journeyId: "money_transfer",
  contextToken: "mtok_eyJhbGciOi...",     // server-to-server minted
  productCode: "HSBC_HK_MT_DOMESTIC",
  campaignId: "summer_rewards",
  trackingContext: ["partnerUserId":"u-123"]
)
HSBCPartnerSdk.startJourney(from: vc, params: params) { result in … }
```

### B) Events emitted (privacy-safe)

*   `MT_BEGIN`, `RECIPIENT_VALID`, `FEE_CALCULATED`, `MT_BIOMETRIC_OK`, `MT_SUBMITTED`
*   Errors: `MT_ERROR` with code `NET_TIMEOUT|AUTH_EXPIRED|PINNING_FAIL|VALIDATION_FAIL|UNKNOWN`

### C) Result mapping

*   `end_complete` → `.completed(payload: { "transferId": "...", "status": "SUBMITTED" })`
*   `end_error` → `.failed(code: "…", message: "…", recoverable: true|false)`

* * *

9) Security & Policy (framework-level)
--------------------------------------

*   **Same as AO**:
    *   OIDC+PKCE (ASWebAuthenticationSession), **contextToken** minted server-side.
    *   **Pinning** for native calls; **ATS** restricted; **WKAppBoundDomains**.
    *   **Origin** whitelist per JM; **JWS** on outbound bridge data.
    *   **Attestation** verdict injected via headers to pages/APIs.
*   **Scopes** (per journey): `mt.validate`, `mt.fee`, `mt.create` separate from AO scopes.
*   **Consent**: JM can mark steps as **consent gates** (the container must block progression until consent event is received).

* * *

10) Packaging & Versioning
--------------------------

*   **Single SDK** artifact (`HSBCPartnerSdk`) with:
    *   `Core` (Runtime Engine, Container, Bridge, State Machine, ApiClient).
    *   `Plugins` (biometric/doc/nfc…) as optional subspecs or binary targets.
*   **Journeys shipped server-side** (JM + OAB), signed & cached by SDK (TTL).
*   **SemVer**: Major = public API changes; Minor = plugin additions; Patch = fixes.
*   **Remote flags** can disable a journey or plugin at runtime (signed config).

* * *

11) Telemetry & Observability
-----------------------------

*   **SDK → HSBC** (privacy-safe): render time, step durations, error codes, retry counts, biometrics usage (boolean), container kind (ASWebAuth vs WKWebView).
*   **Partner → SDK**: event listener mirrors journey events without PII (for funnel metrics).
*   **Correlation**: unique `correlationId` per session used across SDK + AO Web + APIs.

* * *

12) Error Model (unified)
-------------------------

| Code | Layer | Meaning | Recovery |
| --- | --- | --- | --- |
| `AUTH_EXPIRED` | Auth | OIDC/session expired | refresh or restart |
| `PINNING_FAIL` | Transport | cert mismatch | block & report |
| `NET_TIMEOUT` | Network | no response | retry with backoff |
| `ORIGIN_BLOCKED` | Bridge | page origin not allowed | terminate |
| `VALIDATION_FAIL` | Server | schema/field error | show details, correct |
| `RATE_LIMITED` | Server | 429 | wait & retry |
| `UNKNOWN` | Any | unspecified | capture + report |

The JM can attach **error → transition** maps (e.g., `VALIDATION_FAIL` → stay on page; show inline errors).

* * *

13) Building Another Journey: “Money Transfer” Playbook
-------------------------------------------------------

1.  **Define OpenAPI** (validate recipient, calc fee, create transfer).
2.  **Author pages** (`/mt/intro`, `/mt/recipient`, `/mt/amount`, `/mt/review`) with the **standard bridge**.
3.  **Produce JM**:
    *   sequence: intro → recipient → amount → review → biometric (native) → submit (server)
    *   bindings for `validateRecipient` and `calculateFees`
    *   transitions on success/failure/back
4.  **Sign JM** with HSBC key; publish to regional CDN.
5.  **Mint contextToken** server-to-server with consent and scope.
6.  **Partners** call `startJourney(journeyId: "money_transfer", …)` — **no SDK update needed**.

* * *

14) Sample Code Snippets (iOS)
------------------------------

### A) Generic entry

```swift
HSBCPartnerSdk.startJourney(from: self,
   params: JourneyStartParams(journeyId: "money_transfer",
                              contextToken: mtok,
                              productCode: "HSBC_HK_MT_DOMESTIC",
                              campaignId: nil,
                              trackingContext: [:])) { res in
    switch res {
    case .completed(let p): print("Transfer OK:", p)
    case .pending(let p):   print("Pending:", p)
    case .cancelled:        print("Cancelled")
    case .failed(let c, let m, _): print("Failed:", c, m)
    }
}
```

### B) Standard bridge names for MT pages

*   `validateRecipient` → triggers `ValidateRecipient`
*   `calculateFees` → triggers `CalculateFee`
*   `performBiometric` (native) → returns `{ biometric: "PASS" }`
*   `submitTransfer` may be a **server step** to keep submission off the page.

* * *

15) Testing Strategy
--------------------

*   **Contract tests** for JM validation (schema + signature).
*   **OpenAPI conformance** (operationIds, request/response shape).
*   **Bridge tests** (origin checks, JWS verification).
*   **State machine tests** (step coverage, error transitions).
*   **UI tests** (WKWebView buttons → events → transitions).
*   **Chaos**: drop network mid-flow; recover with deterministic transitions.

* * *

16) Migration from AO MVP
-------------------------

*   Rename API surface from `startAccountOpening` → **`startJourney`**.
*   Keep AO JM as the first published journey (`journeyId: "account_opening"`).
*   Add MT JM + OAB; re-use existing container/bridge/security.
*   Partners do **no code changes**; they pass another `journeyId`.

* * *

17) Deliverables Checklist
--------------------------

*   ✅ SDK Core (Runtime, Container, Bridge, State Machine, ApiClient)
*   ✅ Plugin: `biometricAuth` (MVP)
*   ✅ JM schema + validator & signer (bank CI job)
*   ✅ AO JM + OAB (existing)
*   ✅ **MT JM + OAB (new)**
*   ✅ Remote config service (signed)
*   ✅ Sample pages for MT (intro/recipient/amount/review)
*   ✅ Example partner demo toggling AO ↔ MT

* * *

### TL;DR

You now have a **journey-agnostic SDK**: the **framework is stable**, while **each business scenario** (AO, Money Transfer) is shipped as a **signed manifest + pages + OpenAPI bundle**. Security controls, packaging, CI, and partner integration **don’t change**—only the **journey artefacts** do.

If you want, I can generate:

*   a **JSON Schema** for the Journey Manifest,
*   a **validator tool** (Node/Python script) that signs and verifies JM,
*   or a **Money Transfer demo JM + pages** you can drop into a sandbox.

## Prompt:
please verify above the design again, see any update and enhance

## Response:
Thought for 1m 58s

Great call—treating the SDK as a **generic journey framework** is the right abstraction. I re‑reviewed everything you’ve built (AO MVP + genericization plan) and below is a **tight, low‑level, implementation‑ready revision** with **concrete enhancements**. I’ve kept it practical for iOS first, but the design is platform‑agnostic and mirrors cleanly to Android/Web.

* * *

A. What stays correct (no change)
---------------------------------

*   **Hosted pages + native SDK container + secure bridge** is the right baseline for speed, compliance, and rollout control.
*   **Server‑to‑server token pattern** (“contextToken” / “prefillToken”) keeps PII off device—keep it.
*   **Runtime‑driven orchestration** (manifest) lets you add new business journeys (AO, Money Transfer, Cards, …) **without** shipping a new SDK.
*   **OpenAPI‑based bindings** keep the network surface formalized and testable.

* * *

B. What to add / enhance (summary)
----------------------------------

1.  **Version negotiation & capability handshake** (SDK ↔ page) to prevent silent incompatibilities.
2.  **Journey Manifest v1.1**: explicit schema, per‑step **allowed bridge calls**, **guards**, **effects**, and **idempotency keys**.
3.  **Bridge RPC v1.1**: signed envelopes, anti‑replay (nonce+ts), and **per‑step call allow‑list** enforced by the SDK.
4.  **State snapshots & resume**: durable, encrypted, PII‑free resume tokens.
5.  **Hybrid container**: OIDC in `ASWebAuthenticationSession` + step pages in `WKWebView` with **cookie/SSO handoff**.
6.  **Security tightening**: origin pinning, WKAppBoundDomains, CSP guidance, attestation in headers, JWS on bridge responses, optional JWE for sensitive plugin returns.
7.  **OpenTelemetry** & W3C `traceparent` propagation across SDK ↔ page ↔ HSBC APIs.
8.  **Performance budgets & prefetch** with safe asset caching + first paint targets.
9.  **Formal error catalog** + deterministic transitions; **COMPLIANCE\_HOLD** and **MORE\_INFO** states (useful for MT).
10.  **Change management**: manifest/API **compatibility rules** and safe rollout (canary, phased, kill‑switch).
11.  **Money Transfer domain specifics**: FX quote integrity, cut‑off windows, sanctions/Travel Rule checkpoints.

Everything below expands these and gives you drop‑in spec text and payload shapes.

* * *

C. Revised low‑level design
---------------------------

### C1) Public API (generic, stable)

```swift
public struct HSBCInitConfig {
  public enum Environment { case sandbox, production }
  public let environment: Environment
  public let partnerId: String
  public let clientId: String
  public let redirectScheme: String
  public let locale: String
  public let remoteConfigURL: URL?        // signed config
  public let featureFlags: [String: Bool] // local overrides
}

public struct JourneyStartParams {
  public let journeyId: String            // "account_opening", "money_transfer", …
  public let contextToken: String         // minted S2S by partner→HSBC
  public let productCode: String?
  public let campaignId: String?
  public let trackingContext: [String:String]
}

public enum JourneyResult {
  case completed(payload: [String:Any])
  case pending(payload: [String:Any])
  case cancelled
  case failed(code: String, message: String, recoverable: Bool)
}

public enum HSBCPartnerSdk {
  public static func initialize(config: HSBCInitConfig)
  public static func setEventListener(_ listener: @escaping (_ name: String, _ attrs: [String:Any]) -> Void)
  public static func startJourney(from: UIViewController,
                                  params: JourneyStartParams,
                                  completion: @escaping (JourneyResult) -> Void)
}
```

> **No partner API changes** when you add new journeys. All configurability is in manifests.

* * *

### C2) Boot flow & capability handshake

**Step 0**: SDK init → fetch **signed remote config** (per region).  
**Step 1**: `startJourney` → download **JM** (journey manifest), **OAB** (OpenAPI bundle).  
**Step 2**: Present **ASWebAuthenticationSession** for OIDC (ephemeral), then hand off session to `WKWebView`.  
**Step 3**: When the first page loads, SDK sends **ClientHello** (bridge v1.1):

```json
{
  "kind":"hello",
  "bridgeVersion":"1.1",
  "sdkApiVersion":"1.0",
  "journeyId":"money_transfer",
  "sdkCapabilities":["biometric","nfc","filePicker"],
  "locale":"zh-CN",
  "ts":"2025-11-02T09:34:18Z",
  "nonce":"f7c1f9e9…",
  "traceparent":"00-<traceid>-<spanid>-01",
  "sig":"<JWS>"
}
```

**Page** responds with **ServerHello** including `minimumBridgeVersion`, `features`, and **policy knobs** (e.g., requireBiometric). SDK validates **compatibility** before continuing.

* * *

### C3) Journey Manifest (JM) v1.1 (signed)

**Key changes vs v1.0**

*   `bridgeAllow`: explicit allow‑list of bridge calls per step.
*   `guard` / `effect` blocks: deterministic transitions.
*   `idempotencyKey`: for `server` steps.
*   `resumePolicy`: when/how to snapshot.

**JM skeleton**

```json
{
  "manifestVersion": "1.1.0",
  "journeyId": "money_transfer",
  "oapiBundle": "https://api.sandbox.partner.hsbc.com/oapi/transfer-v1.yaml",
  "startStep": "mt_intro",
  "headers": {
    "X-HSBC-Partner-Id": "$partnerId",
    "X-HSBC-Client-Id": "$clientId",
    "X-HSBC-Locale": "$locale",
    "X-HSBC-Context": "$contextToken",
    "X-HSBC-Attest": "$attestation"            // DeviceCheck/App Attest verdict (base64)
  },
  "security": {
    "pinning": true,
    "allowedOrigins": ["https://ao.sandbox.partner.hsbc.com"],
    "csp": "default-src 'self' https://ao.sandbox.partner.hsbc.com; frame-ancestors 'none';"
  },
  "resumePolicy": { "snapshotOn": ["stepEnter","serverSuccess"] },
  "steps": {
    "mt_intro": {
      "type": "web",
      "url": "https://ao.sandbox.partner.hsbc.com/mt/intro",
      "bridgeAllow": [],
      "on": { "next": {"to":"mt_recipient"}, "cancel":{"to":"end_cancel"} }
    },
    "mt_recipient": {
      "type": "web",
      "url": "https://ao.sandbox.partner.hsbc.com/mt/recipient",
      "bridgeAllow": ["validateRecipient"],
      "bindings": [{
        "onEvent": "validateRecipient",
        "call": { "operationId":"ValidateRecipient", "argsFrom":"event.payload" },
        "onSuccessEmit":"recipientValid",
        "onErrorEmit":"recipientInvalid"
      }],
      "on": {
        "recipientValid": {"to":"mt_amount"},
        "recipientInvalid": {"to":"mt_recipient", "effect":{"emit":"showInlineErrors"}},
        "back": {"to":"mt_intro"}
      }
    },
    "mt_amount": {
      "type": "web",
      "url": "https://ao.sandbox.partner.hsbc.com/mt/amount",
      "bridgeAllow": ["calculateFees"],
      "bindings": [{
        "onEvent":"calculateFees",
        "call":{"operationId":"CalculateFee","argsFrom":"event.payload"},
        "onSuccessEmit":"feesReady"
      }],
      "on": { "proceed":{"to":"mt_review"}, "back":{"to":"mt_recipient"} }
    },
    "mt_biometric": {
      "type": "native",
      "plugin": "biometricAuth",
      "params": { "reason":"Authorize transfer" },
      "on": { "success":{"to":"mt_submit"}, "failure":{"to":"mt_review"} }
    },
    "mt_submit": {
      "type": "server",
      "idempotencyKey": "$session.correlationId",
      "call": { "operationId":"CreateTransfer", "argsFrom":"session" },
      "on": {
        "success": { "to":"end_complete", "effect":{"emit":"MT_SUBMITTED"} },
        "error":   { "to":"end_error" }
      }
    },
    "end_complete": { "type":"terminal", "result":{"status":"COMPLETED"} },
    "end_cancel":   { "type":"terminal", "result":{"status":"CANCELLED"} },
    "end_error":    { "type":"terminal", "result":{"status":"FAILED"} }
  },
  "signature": "JWS-DETACHED"
}
```

> **JSON Schema**: keep a schema in CI that validates the manifest (types, operationIds exist, `bridgeAllow` subset of known calls, terminal steps have no outgoing edges).

* * *

### C4) Bridge RPC v1.1 (security‑tight)

**Envelope for all SDK→Page messages**

```json
{
  "kind": "resolve" | "event" | "hello",
  "id": "…",                       // only for request/resolve
  "name": "performBiometric" | "ready" | …,
  "payload": {…},
  "meta": {
    "ts": "2025-11-02T09:35:52Z",
    "nonce": "4c3d…",
    "sdkVersion": "1.0.0",
    "bridgeVersion": "1.1",
    "traceparent": "00-…"
  },
  "sig": "<JWS over (name,payload,meta)>"
}
```

**Anti‑replay**

*   SDK keeps a short‑lived **nonce window** per step; rejects duplicates/out‑of‑window.
*   Page validates `sig` against HSBC SDK public key (published per region).

**Per‑step allow‑list**

*   If a page calls an unlisted `request` for the current step, SDK **rejects** with `code: BRIDGE_FORBIDDEN`.

* * *

### C5) State machine + resume

*   Each **step enter** and **server success** emits a **snapshot**:
    *   `snapshot = { stepId, sessionContextSubset, ts, ttl }`
    *   Stored encrypted in Keychain with **ThisDeviceOnly** attribute (iOS).
*   **Resume token** is a short opaque string the partner may store (PII‑free).
*   `startJourney` can accept an optional `resumeToken` to fast‑forward.

* * *

### C6) Hybrid container (SSO handoff)

*   **Login**: `ASWebAuthenticationSession` (OIDC + PKCE), ephemeral if possible.
*   **Step pages**: `WKWebView` with:
    *   **WKAppBoundDomains** = HSBC domains only.
    *   Cookie/SSO handoff: rely on the issuer’s **SameSite=None; Secure** cookies **or** a bank‑issued **session handoff token** redeemed by the step page on first load.
    *   **Fail‑safe**: if SSO handoff fails, retry `ASWebAuth` and reload.

* * *

### C7) Security concrete updates

*   **Origin & CSP**: JM declares `allowedOrigins` and a **minimum CSP**; step pages send `Content‑Security‑Policy` headers.
*   **Pinning**: native requests pinned (dual pins current+next); WebView inherits ATS + AppBound.
*   **Attestation**: DeviceCheck/App Attest verdict serialized and sent in `X‑HSBC‑Attest` header.
*   **JWS everywhere**: All SDK→Page replies signed; sensitive plugin outputs optionally **JWE**.
*   **Per‑journey scopes**: AO vs MT separated; principle of least privilege.
*   **Kill switch**: Remote config can disable a journey or plugin per region/partner instantly.

* * *

### C8) Observability & SLOs

*   **Trace propagation**: SDK generates a `traceparent` on start, passes to page (hello) and to APIs (HTTP header), enabling full E2E tracing.
*   **Metrics** (privacy‑safe):
    *   Time to first paint (TTFP), step dwell times, error rates, retry counts, bridge latency, plugin success rates.
*   **SLOs** (targets):
    *   First AO/MT page paint ≤ **1200 ms** on 4G; p95 server step ≤ **800 ms**; session resume ≤ **400 ms**.

* * *

### C9) Performance & prefetch

*   **Asset prefetch** (icons/CSS) allowed via an HSBC **asset manifest** (signed), fetched on init; cached via `WKURLSchemeHandler` with TTL.
*   Avoid prefetching **HTML with PII**; all dynamic content stays server‑side.

* * *

### C10) Error catalog (normalized)

| Code | Layer | Notes / Transition guidance |
| --- | --- | --- |
| `AUTH_EXPIRED` | Auth | Re‑run OIDC, reload current step |
| `PINNING_FAIL` | Transport | Terminate; show secure error |
| `NET_TIMEOUT` | Network | Backoff/retry; surface to page via event |
| `ORIGIN_BLOCKED` | Bridge | Terminate; misconfiguration |
| `VALIDATION_FAIL` | Server | Stay on page; show inline errors |
| `RATE_LIMITED` | Server | Observe `Retry‑After` |
| `COMPLIANCE_HOLD` | Orchestr. | New for MT/AO; emit `MORE_INFO` nextActions |
| `MORE_INFO` | Orchestr. | Move to **document/extra‑info** step |
| `UNKNOWN` | Any | Log & safe error |

JM can include an `errorMap` per step to override defaults.

* * *

D. Money Transfer (MT) domain add‑ons
-------------------------------------

1.  **FX quotes**
    *   `GetQuote` API with `quoteId` and **lifetime/expiry**; the **review** page must show the quote and the server step must **validate the same quoteId**.
    *   If expired → transition back to amount with `QUOTE_EXPIRED`.
2.  **Cut‑off windows & scheduling**
    *   Manifest can include `timePolicies` → if after cut‑off, route to **schedule step**.
    *   `CreateTransfer` supports `executionDate` and idempotency key.
3.  **Sanctions/Travel Rule**
    *   Server step `PreScreen` before create; outcomes: `PASS`, `HOLD`, `REJECT`.
    *   `HOLD` → `COMPLIANCE_HOLD` state with `nextActions` (upload docs).
4.  **Confirmation of Payee (CoP)**
    *   Optional binding from `recipient` step; show match/mismatch warnings per market.
5.  **Statuses**
    *   `.completed(payload:{transferId, status:"SUBMITTED|BOOKED"})`
    *   `.pending(payload:{transferId, status:"HOLD", nextActions:[…]})`

* * *

E. Compatibility & rollout rules
--------------------------------

*   **Version matrix**
    *   `bridgeVersion`: 1.1 (min accepted 1.0).
    *   `manifestVersion`: >= 1.1 to use `bridgeAllow`, `resumePolicy`.
    *   `sdkApiVersion`: 1.x → no breaking public API until 2.0.
*   **Rollout**
    *   Publish JM as **canary** (5% traffic) via remote config; monitor SLOs; promote to 100%.
    *   **Kill‑switch** if error budget exceeded.
    *   Breaking manifest changes **must** bump `manifestVersion` and **reject** on older SDKs during handshake with a clear `UPGRADE_REQUIRED` reason.

* * *

F. Testing matrix (what to automate)
------------------------------------

*   **JM Schema validation** (CI), signature check, `operationId` existence in OAB.
*   **Bridge contract** (golden tests for envelopes, sig, nonce, TTL).
*   **State machine** property tests (no dead ends; terminals reachable).
*   **Resume** tests (snapshot encode/decode; reentry on every step).
*   **Offline/Retry** (drop network mid‑call; idempotency honored).
*   **Perf** (TTFP, step latency) with synthetic pages.
*   **Security**: origin spoof attempts, invalid sig, replayed nonce, pinning failure path.

* * *

G. Concrete changes to your current MVP
---------------------------------------

1.  **Introduce `journeyId` + `contextToken`** in the public API now (AO uses `account_opening`, MT uses `money_transfer`).
2.  **Add ClientHello/ServerHello** bridge messages and **reject** on version mismatch.
3.  Enforce **per‑step `bridgeAllow`** from the manifest (deny others with `BRIDGE_FORBIDDEN`).
4.  Sign all SDK→Page responses with **JWS** (CryptoKit), include `ts` + `nonce`.
5.  Add **resume snapshots** (Keychain ThisDeviceOnly; redact PII).
6.  Implement **hybrid container**: OIDC via `ASWebAuthenticationSession`, then `WKWebView` with SSO handoff token (or SameSite cookie).
7.  Add error codes `COMPLIANCE_HOLD` and `MORE_INFO` in the result map.
8.  Add **traceparent** propagation; start emitting privacy‑safe metrics.

* * *

H. Example: MT Journey manifest delta (snippets)
------------------------------------------------

**Add FX quote validation to review → submit path**

```json
"mt_review": {
  "type":"web",
  "url":"https://ao.sandbox.partner.hsbc.com/mt/review",
  "bridgeAllow":["requestQuote"],
  "bindings":[
    {"onEvent":"requestQuote","call":{"operationId":"GetQuote","argsFrom":"event.payload"},
     "onSuccessEmit":"quoteReady","onErrorEmit":"quoteError"}
  ],
  "on":{
    "confirm":{"to":"mt_submit", "guard":{"expr":"session.quoteId != null && now < session.quoteExpiry"} , "else":{"to":"mt_amount","effect":{"emit":"QUOTE_EXPIRED"}}},
    "back":{"to":"mt_amount"}
  }
}
```

* * *

I. Governance & docs to ship with the framework
-----------------------------------------------

*   **JM JSON Schema** + signer/validator CLI.
*   **Bridge protocol** reference (v1.1) with envelope examples and allowed methods.
*   **OpenAPI review** checklist (idempotency, error shapes, rate limits).
*   **Security hardening guide** (ATS, pinning, AppBound, CSP examples).
*   **Operational runbook** (kill‑switch, canary, rollbacks, SLO monitoring).
*   **Partner quick starts** (how to pick a journey, how to pass `contextToken`, how to handle results).

* * *

### TL;DR

You now have a **hardened, journey‑agnostic SDK** with:

*   **Versioned handshake**, **signed bridge**, **deterministic state machine**, **resume**, and **OpenTelemetry**.
*   **Money Transfer** is just **pages + manifest + OpenAPI**—no SDK change.
*   Stronger security, better resilience, and clear rollout/compat rules.

If you want, I can output:

*   a **JM v1.1 JSON Schema**,
*   a **bridge v1.1 spec sheet** (method list + envelopes + error codes),
*   or a **sample MT manifest + mock APIs** aligned to this revision.

## Prompt:
 now i start to developer this hardened, journey‑agnostic SDK for iOS, and will use VSCode + OpenAI codex , please give me detail step by step guide from scratch , with the details prompt for Codex

## Response:
Thought for 2m 5s

Perfect — here’s a **from‑scratch, step‑by‑step build guide** to implement the **hardened, journey‑agnostic iOS SDK** in **VS Code** using **OpenAI Codex / Copilot** as your code generator.  
For each step I include:

*   **Goal** (what you’ll achieve)
*   **Commands** (what to run)
*   **Files / structure**
*   **“Prompt for Codex”** blocks you can paste directly into Copilot Chat (or as file‑header comments) to generate code
*   **Acceptance checks** so you know it’s done

> Scope: iOS 14+, Swift 5.9+, Xcode 15+. We’ll implement **SDK v1.1** features: Journey Manifest (JM) v1.1, OpenAPI resolver, hybrid container (ASWebAuth + WKWebView), signed bridge handshake, idempotency, retries, resume tokens, and telemetry.

* * *

0) Prereqs (one time)
---------------------

**Install & configure**

```bash
# Xcode
xcode-select --install
sudo xcodebuild -license accept

# VS Code extensions
# - Swift (Apple)
# - GitHub Copilot / Copilot Chat (or your OpenAI Codex client)
# - CodeLLDB (debugging)
```

**Create workspace**

```bash
mkdir hsbc-partner-sdk-ios
cd hsbc-partner-sdk-ios
git init
```

* * *

1) Repository skeleton
----------------------

**Goal:** multi‑package layout that separates core, plugins, tools, demo app, tests.

```
hsbc-partner-sdk-ios/
├─ sdk/
│  ├─ Package.swift
│  └─ Sources/
│     ├─ HSBCPartnerSDKCore/                # core target
│     │  ├─ Public/                         # public API only
│     │  │  └─ HSBCPartnerSdk.swift
│     │  ├─ Runtime/                        # Engine pieces
│     │  │  ├─ RuntimeEngine.swift
│     │  │  ├─ StateMachine.swift
│     │  │  ├─ SessionManager.swift
│     │  │  └─ EventBus.swift
│     │  ├─ Security/
│     │  │  ├─ KeyStore.swift
│     │  │  ├─ JWS.swift
│     │  │  └─ AttestationCollector.swift
│     │  ├─ Network/
│     │  │  ├─ ApiClient.swift
│     │  │  └─ OpenAPIResolver.swift
│     │  ├─ Manifest/
│     │  │  ├─ ManifestModels.swift
│     │  │  ├─ ManifestLoader.swift
│     │  │  └─ ManifestSchema.json
│     │  ├─ Container/
│     │  │  ├─ HybridContainer.swift        # ASWebAuth + WKWebView
│     │  │  ├─ Bridge.swift                 # JS bridge & handshake
│     │  │  └─ WebPolicies.swift            # ATS/AppBound/origin guard
│     │  ├─ Telemetry/
│     │  │  └─ Telemetry.swift              # traceparent + metrics
│     │  └─ Resources/
│     │     └─ ao_test.html                 # local test page
│     └─ HSBCPlugins/                       # optional plugin target
│        └─ BiometricAuth.swift
│
├─ examples/
│  └─ PartnerDemoApp/                       # Xcode demo app
├─ tests/
│  ├─ Unit/
│  │  ├─ ManifestTests.swift
│  │  ├─ BridgeHandshakeTests.swift
│  │  ├─ StateMachineTests.swift
│  │  └─ ApiClientTests.swift
│  └─ UI/
│     └─ PartnerDemoAppUITests.swift
│
├─ scripts/
│  ├─ build_xcframework.sh
│  └─ make_release.sh
└─ .github/workflows/ios-sdk.yml
```

**Create Swift package**

```bash
cd sdk
swift package init --type library
```

**Prompt for Codex — create `sdk/Package.swift`**

> Create a Swift Package manifest (Swift 5.9) named `HSBCPartnerSDK`. Platforms: iOS 14+. Products:
> 
> *   library `HSBCPartnerSDKCore` target `HSBCPartnerSDKCore`
> *   library `HSBCPlugins` target `HSBCPlugins`  
>     Targets: `HSBCPartnerSDKCore` (process Resources/ao\_test.html and PrivacyInfo.xcprivacy), `HSBCPlugins`, and a test target `HSBCPartnerSDKCoreTests`. No external dependencies.
>     

* * *

2) Public API surface
---------------------

**Goal:** a stable entry point; no business specifics.

**Prompt for Codex — `Public/HSBCPartnerSdk.swift`**

> Write a Swift file that defines:
> 
> *   `HSBCInitConfig { environment: enum sandbox|production, partnerId, clientId, redirectScheme, locale, remoteConfigURL: URL?, featureFlags: [String:Bool], telemetryOptIn: Bool }`.
> *   `JourneyStartParams { journeyId, contextToken, resumeToken: String?, productCode: String?, campaignId: String?, trackingContext: [String:String], capabilitiesOverride: [String:Bool]? }`.
> *   `JourneyResult` enum: `.completed(payload:[String:Any])`, `.pending(payload:[String:Any])`, `.cancelled`, `.failed(code:String,message:String,recoverable:Bool)`.
> *   `HSBCPartnerSdk` enum with static: `initialize(config:)`, `setEventListener(_:)`, `startJourney(from:params:completion:)`.  
>     Implement `startJourney` to hand off to a `RuntimeEngine.start(...)`.  
>     Document each API with SwiftDoc; mark as `public`.
>     

**Acceptance**

*   Package builds; types visible from demo app.

* * *

3) Event bus & telemetry (journey‑agnostic)
-------------------------------------------

**Prompt for Codex — `Runtime/EventBus.swift`**

> Create a tiny event bus with:
> 
> *   `JourneyEvent { name:String, attributes:[String:Any], ts:Date, correlationId:String }`
> *   `EventBus` singleton to register a single external listener and to `emit(_:)`.
> *   Helper methods for canonical events: `journeyBegin`, `stepEnter`, `apiCall`, `error`, `result`.  
>     Ensure thread safety via serial queue.
>     

**Prompt for Codex — `Telemetry/Telemetry.swift`**

> Implement `Telemetry` with functions to generate W3C `traceparent` and `tracestate`, and to time spans (`withSpan(name:attributes:_:)`). Include minimal no-op backend but keep the API.

**Acceptance**

*   You can call `EventBus.shared.emit(...)` from anywhere; telemetry returns a `traceparent` string.

* * *

4) Security basics: KeyStore + JWS helper
-----------------------------------------

**Prompt for Codex — `Security/KeyStore.swift`**

> Implement `KeyStore` to hold a set of pinned public keys for verifying JWS (HSBC manifest keys).
> 
> *   Store `kid -> SecKey`.
> *   Provide `update(keys:)`, `key(for kid:)`.
> *   Add in-memory map; load initial keys from a bundled JSON (stub allowed).
>     

**Prompt for Codex — `Security/JWS.swift`**

> Implement a minimal JWS (compact serialization) verifier and signer using CryptoKit with P-256.
> 
> *   `struct JWS { static func verify(compact:String, key:SecKey) throws -> Data /* payload */ }`
> *   `JwsSigner` that signs with Secure Enclave `P256.Signing.PrivateKey` and returns compact string with header `{ "alg":"ES256","kid":"<kid>" }`.  
>     DO NOT use external libs. Add clear error types.
>     

**Acceptance**

*   Unit test verifies that signing then verifying returns the original payload (same process, local key).

* * *

5) Remote config & manifest loader (JM v1.1)
--------------------------------------------

**Prompt for Codex — `Manifest/ManifestModels.swift`**

> Define Codable structs for JM v1.1:  
> Top-level fields: `manifestVersion:String, minSdk:String, requiredCapabilities:[String], journeyId:String, oapiBundle:URL, locales:[String], startStep:String, security:{allowedOrigins:[URL], pinning:Bool, attestation:[String:Bool]?, requireHandshake:Bool}, headers:[String:String], experiments:{key:String, variants:[String], weights:[Double]}?, steps:[String:Step], signature:String`.  
> `Step` has: `type: enum web|native|server|terminal`, `url:URL?`, `plugin:String?`, `params:[String:AnyCodable]?`, `timeoutMs:Int?`, `bindings:[Binding]?`, `on:[String:Transition]?`, `result:[String:AnyCodable]?`.  
> `Binding { onEvent:String, call:{ operationId:String, argsFrom:String?, headers:[String:String]? }, onSuccessEmit:String?, onErrorEmit:String? }`  
> `Transition { to:String?, emit:BridgeEvent?, guardExpr:String? }`  
> Include a tiny `AnyCodable` to carry untyped JSON values.

**Prompt for Codex — `Manifest/ManifestLoader.swift`**

> Implement `ManifestLoader`:
> 
> *   `func load(journeyId:String, contextToken:String, config:HSBCInitConfig, keyStore:KeyStore) async throws -> Manifest`
> *   Downloads the manifest URL derived from `remoteConfig` (use a stub URL resolver for now), verifies detached JWS in `signature` using `KeyStore`, validates `minSdk`, ensures `allowedOrigins` is not empty.
> *   Load `oapiBundle` later (return URL string for now).
> *   Provide `validateAgainstSchema(schemaURL: URL)`; stub schema check with basic required field checks (no external lib).
>     

**Acceptance**

*   Loader fetches a local JSON file (for now), verifies a test signature (you can bypass verification with a `TODO:` comment), returns a decoded `Manifest`.

* * *

6) OpenAPI resolver (operationId → HTTP call)
---------------------------------------------

**Prompt for Codex — `Network/OpenAPIResolver.swift`**

> Implement `OpenAPIResolver` that loads an OpenAPI 3.0 JSON/YAML (use only JSON now) and builds a mapping from `operationId -> (method:String, path:String)`.  
> Provide: `func resolve(operationId:String) -> (method:String, path:String)?`.  
> Add a helper to apply path and base URL to form a `URLRequest` given `args` (as `[String:Any]`) and extra headers.

**Prompt for Codex — `Network/ApiClient.swift`**

> Implement `ApiClient` with:
> 
> *   init(baseURL: URL, resolver: OpenAPIResolver, pinningEnabled: Bool)
> *   `func call(operationId:String, body:[String:Any]?, headers:[String:String], idempotencyKey:String?) async throws -> (status:Int, headers:[AnyHashable:Any], body:Data)`
> *   Add retries with exponential backoff for 408/429/5xx (max 3 attempts), honoring `Retry-After`.
> *   If `idempotencyKey` non-nil, set header `X-Idempotency-Key`.
> *   Inject `traceparent` header from `Telemetry`.
>     

**Acceptance**

*   A unit test that resolves a tiny fake OAS json and successfully builds a request.

* * *

7) Session & resume
-------------------

**Prompt for Codex — `Runtime/SessionManager.swift`**

> Implement `SessionManager` (singleton) with:
> 
> *   `correlationId:String` (UUID)
> *   store: `contextToken:String`, `resumeToken:String?`, `stepPointer:String?`, `idempotencyKey:String` per journey.
> *   Persist a **snapshot** `{journeyId, stepPointer, idempotencyKey, ts}` into Keychain with kSecAttrAccessibleAfterFirstUnlockThisDeviceOnly.
> *   `func saveSnapshot()`, `func loadSnapshot(resumeToken:String) -> Snapshot?`.  
>     **No PII in snapshots.**
>     

**Acceptance**

*   Unit test: save, kill process (simulate by re-instantiating), load snapshot. (You can stub Keychain with in-memory store in tests.)

* * *

8) Hybrid container & web policies
----------------------------------

**Prompt for Codex — `Container/HybridContainer.swift`**

> Implement `HybridContainer` that exposes:
> 
> *   `func signInIfNeeded(authURL:URL, redirectScheme:String) async throws -> URL` using `ASWebAuthenticationSession` with `prefersEphemeralWebBrowserSession = true`.
> *   `func presentStepWebView(on:UIViewController, url:URL, allowedOrigins:[URL], bridge:Bridge) -> UIViewController` that returns a modal VC hosting a WKWebView configured with: JavaScript enabled, no back/forward gestures, navigation delegate that enforces origin checks (`allowedOrigins`), and blocks new windows. Use `WKWebViewConfiguration` and `WKUserContentController`.
> *   Requires iOS 14+.
>     

**Prompt for Codex — `Container/WebPolicies.swift`**

> Provide helpers:
> 
> *   `func isAllowed(origin:URL, allowed:[URL]) -> Bool` (exact host match and https only)
> *   WKAppBoundDomains note in a comment (configured in Info.plist in the demo app).
> *   ATS guidance comment: TLS 1.2+, no arbitrary loads.
>     

**Acceptance**

*   Demo app can present a WKWebView with the local `ao_test.html` and origin check passes (file URL allowed for dev only).

* * *

9) Bridge (signed handshake + privileged calls)
-----------------------------------------------

**Prompt for Codex — `Container/Bridge.swift`**

> Implement a `Bridge` class that:
> 
> *   Sets up a `WKScriptMessageHandler` named `"hsbcBridge"`.
> *   Maintains handshake state: notReady → ready.
> *   Accepts page events `{kind:'event', name:'bridge_hello', payload:{origin,pageNonce}}`, validates `origin` via `WebPolicies`, collects device attestation via `AttestationCollector.collect()` (stub), and replies with `{kind:'event', name:'bridge_ready', payload:{ sessionProofJws, features:[...] }}` using `JWS` signer.
> *   For `kind:'request'`, allow only if `ready == true` and `name` ∈ `allowedMethodsForStep` (passed in).
> *   Implement privileged stubs:
>     *   `performBiometric` → uses `LocalAuthentication` to evaluate policy; returns `{ biometric:"PASS" }`.
> *   All SDK→Page responses are wrapped in a signed envelope: `{kind:'resolve', id, name, payload, meta:{ts,nonce,bridgeVersion,sdkVersion,traceparent}, sig}`.
> *   Include detailed error codes: `BRIDGE_FORBIDDEN`, `ORIGIN_BLOCKED`, `HANDSHAKE_REQUIRED`.
>     

**Acceptance**

*   UI test clicks buttons in `ao_test.html` and sees successful request/resolve with READY emitted first.

* * *

10) State machine (steps, transitions, bindings)
------------------------------------------------

**Prompt for Codex — `Runtime/StateMachine.swift`**

> Implement a simple state machine:
> 
> *   Holds current `stepId`, a map of `steps`.
> *   `func handleEvent(name:String, payload:[String:Any])` looks for transitions in `steps[current].on[name]`, evaluates optional `guardExpr` (support `==`, `!=`, `>`, `<`, `&&`, `||` over numeric/string/bool using a tiny evaluator), executes `effect.emit` if present, and moves to `to`.
> *   Support `timeoutMs`: schedule a timer on step enter that triggers event `"timeout"`.
> *   `bindings`: map `onEvent` to `ApiClient.call` with `argsFrom` keypath (support dot-paths like `event.payload.amount`).  
>     Emit `STEP_ENTER`/`STEP_EXIT` via `EventBus`.
>     

**Acceptance**

*   Unit tests drive through a tiny dummy manifest with 3 steps and confirm transitions.

* * *

11) Runtime engine (wire it all)
--------------------------------

**Prompt for Codex — `Runtime/RuntimeEngine.swift`**

> Implement `RuntimeEngine.start(from: UIViewController, params: JourneyStartParams, completion:@escaping(JourneyResult)->Void)`:
> 
> 1.  Load remote config (stub ok) and JM via `ManifestLoader`.
> 2.  Resolve OAS via `OpenAPIResolver` and create `ApiClient`.
> 3.  Create `HybridContainer`, run `signInIfNeeded(...)` if required by JM (add a boolean for now).
> 4.  Present first page with `presentStepWebView(...)`, attach `Bridge` with `allowedMethodsForStep`.
> 5.  Wire Bridge → StateMachine by translating page events to `handleEvent`, and `bindings` to API calls.
> 6.  On terminal step, map to `JourneyResult` (completed/pending/failed) and call `completion`.
> 7.  Save snapshots (`SessionManager.saveSnapshot`) on step enter and on successful server call.
> 8.  Emit canonical events to `EventBus`.
>     

**Acceptance**

*   Demo flows from intro → recipient → amount → review → biometric → submit → end\_complete using the local test page (mock the API client to return canned responses).

* * *

12) Plugins: Biometric (MVP) + interface
----------------------------------------

**Prompt for Codex — `HSBCPlugins/BiometricAuth.swift`**

> Provide `public protocol JourneyPlugin { var name:String { get } func canHandle(_ method:String) -> Bool; func handle(_ method:String, params:[String:Any]) async throws -> [String:Any] }`.  
> Implement `BiometricAuthPlugin` using `LocalAuthentication` for FaceID/TouchID; method `performBiometric(reason:String)` returning `{"biometric":"PASS"}` or throws.

**Acceptance**

*   Bridge can delegate to the plugin when `performBiometric` is requested.

* * *

13) Error taxonomy & mapping
----------------------------

**Prompt for Codex — extend `Network/ApiClient.swift` & `Runtime/RuntimeEngine.swift`**

> Add a canonical error enum with codes: `AUTH_EXPIRED`, `PINNING_FAIL`, `ORIGIN_BLOCKED`, `NET_TIMEOUT`, `RATE_LIMITED`, `VALIDATION_FAIL`, `IDEMPOTENT_REPLAY`, `SCA_REQUIRED`, `UNKNOWN`.  
> Map HTTP 401/403 to `AUTH_EXPIRED`, 408 to `NET_TIMEOUT`, 409 (with idempotency) to `IDEMPOTENT_REPLAY`, 422/400 with field errors to `VALIDATION_FAIL`. Ensure `RuntimeEngine` converts these to `JourneyResult.failed(..., recoverable:)` or to a transition defined in JM `errorMap` (you can keep default mapping for now).

**Acceptance**

*   A failing API call yields a deterministic SDK error with one of the above codes.

* * *

14) Demo app to test everything
-------------------------------

Create an Xcode app in `examples/PartnerDemoApp/`, add the package locally, and wire:

**Prompt for Codex — `examples/PartnerDemoApp/ContentView.swift`**

> Produce a SwiftUI view with:
> 
> *   Button “Initialize SDK”
> *   Button “Start Money Transfer” calling `startJourney(journeyId:"money_transfer", contextToken:"mtok_demo", productCode:nil, …)`
> *   A status text bound to latest `JourneyResult` and printed events from `EventBus`.
>     

**Acceptance**

*   You can run, initialize SDK, start the journey, interact with test page buttons, and receive `.completed`/`.pending`.

* * *

15) Tests (unit + UI)
---------------------

**Prompt for Codex — `tests/Unit/ManifestTests.swift`**

> Test that a sample JM v1.1 JSON decodes and enforces `minSdk` and non-empty `allowedOrigins`.

**Prompt for Codex — `tests/Unit/BridgeHandshakeTests.swift`**

> Simulate `bridge_hello` from a permitted origin and verify the SDK responds with `bridge_ready` including a JWS.

**Prompt for Codex — `tests/Unit/StateMachineTests.swift`**

> Assert transitions occur and `timeout` fires.

**Prompt for Codex — `tests/UI/PartnerDemoAppUITests.swift`**

> Automate tapping demo buttons, pressing the READY/biometric/complete buttons in WKWebView, and validate the status label.

* * *

16) CI & packaging
------------------

You already have CI in earlier steps. Ensure:

*   `ios-sdk.yml` builds & tests; uploads `HSBCAOSDKCore.xcframework`
*   SwiftPM plugin `build-xcframework` available locally
*   Privacy manifest `PrivacyInfo.xcprivacy` is part of the target
*   `Info.plist` in demo app has ATS strict and `WKAppBoundDomains` for your HSBC domains (or localhost for dev)

* * *

17) Harden & extend (checklist)
-------------------------------

*    Replace local `ao_test.html` with real HSBC page URLs per JM; enforce `allowedOrigins`.
*    Enable **pinning** for native calls (dual pins current+next).
*    Implement **attestation** collector (DeviceCheck/App Attest) and add `X‑HSBC‑Device‑Attestation` header.
*    JWS signing with **Secure Enclave** keys; rotate **KID** via remote config.
*    Add **idempotency key** per submission; map HTTP 409 to prior success.
*    Add **resumeToken** plumbing in public API and state reentry.
*    `traceparent` end‑to‑end; measure p95 step latency and TTFP.
*    Expand **guardExpr** evaluator or replace with a safe mini‑DSL.
*    JSONPath extraction for `argsFrom` (start with dot‑path; add array indexes later).

* * *

18) Ready‑to‑paste **Codex one‑shot prompts** (use these as you code)
---------------------------------------------------------------------

You can drop any of these into Copilot Chat:

**A. Generate public API**

```
Create Public/HSBCPartnerSdk.swift for an iOS Swift package target.
Expose a public SDK with:
- HSBCInitConfig { environment: sandbox|production; partnerId, clientId, redirectScheme, locale, remoteConfigURL: URL?, featureFlags: [String:Bool], telemetryOptIn: Bool }
- JourneyStartParams { journeyId, contextToken, resumeToken: String?, productCode: String?, campaignId: String?, trackingContext: [String:String], capabilitiesOverride: [String:Bool]? }
- JourneyResult enum (completed/pending/cancelled/failed)
- HSBCPartnerSdk with initialize, setEventListener, startJourney. startJourney delegates to RuntimeEngine.start.
Swift 5.9, iOS 14+, no external deps. Add SwiftDoc comments.
```

**B. Build Bridge with handshake**

```
Implement Container/Bridge.swift for WKWebView:
- Register "hsbcBridge" message handler
- Handshake: on event {name:"bridge_hello", payload:{origin,pageNonce}}, verify origin via WebPolicies, then reply event {name:"bridge_ready", payload:{sessionProofJws, features:[...]}}, signed with ES256 using JWS helper.
- For kind:'request', allow only if handshake completed AND method is in allowedMethodsForStep.
- Implement privileged method "performBiometric" using LocalAuthentication.
- All SDK->page replies include a signed envelope with meta {ts,nonce,bridgeVersion:"1.1",sdkVersion:"1.0.0",traceparent}.
```

**C. State machine with bindings**

```
Implement Runtime/StateMachine.swift:
- Holds current stepId and step table from Manifest.
- handleEvent(name,payload) finds Transition; evaluate guardExpr (tiny evaluator for ==,!=,>,<,&&,|| with dot-path lookups); execute effect.emit; move to 'to'.
- On step enter, start an optional timeoutMs timer; on fire, emit "timeout".
- For bindings, when page emits Binding.onEvent, call ApiClient with argsFrom mapping (dot-path extractor) and emit onSuccessEmit or onErrorEmit back to the page.
- Emit EventBus events for STEP_ENTER/EXIT.
```

**D. ApiClient retries + idempotency**

```
Implement Network/ApiClient.swift call(operationId,body,headers,idempotencyKey):
- Resolve method+path via OpenAPIResolver.
- Build URLRequest with JSON body and headers + traceparent.
- Retry policy: 408/429/5xx with exponential backoff and jitter (3 attempts), read Retry-After when present.
- If idempotencyKey != nil, set X-Idempotency-Key header.
- Map response codes to canonical error codes; throw structured errors with code and httpStatus.
```

**E. Session snapshots**

```
Implement Runtime/SessionManager.swift saveSnapshot/loadSnapshot:
- Save {journeyId, stepPointer, idempotencyKey, ts} into Keychain as JSON, class kSecClassGenericPassword, kSecAttrAccessibleAfterFirstUnlockThisDeviceOnly.
- Provide generateResumeToken() that returns a random opaque string mapped to the saved snapshot.
- No PII.
```

* * *

19) What “done” looks like
--------------------------

*   ✅ You can initialize the SDK, start **money\_transfer**, see the page load (local HTML or HSBC dev URL), trigger **biometric**, and complete → `JourneyResult.completed`.
*   ✅ Negative tests: origin blocked, handshake missing, API 429 with retry, timeout path, resume after kill.
*   ✅ CI builds `HSBCPartnerSDKCore.xcframework` and runs tests.

* * *

If you want, I can also generate:

*   A **sample Money Transfer manifest v1.1** + a tiny **mock API** (Node) aligned to `operationId`s,
*   A **JM validator CLI** (Node/Python) that checks signature + schema + OAS operationIds,
*   A **ready-made demo HSBC page** (HTML/JS) that uses the bridge (`bridge_hello`, `performBiometric`, etc.) for local testing.

Just say the word and I’ll drop those assets in the same step‑by‑step style.

## Prompt:
please generate with same step by step style, not code directly , use prompt

## Response:
awesome — here’s a **from-scratch, step-by-step pack** that gives you _only prompts_ you can paste into VS Code Copilot / Codex to generate the artifacts. No code is printed here; each step has:

*   **Goal**
*   **Where to create files**
*   **Prompt for Codex** (paste as-is)
*   **Acceptance checks**

The pack covers:

1.  Money-Transfer Journey Manifest v1.1 (JM)
2.  Mock Transfer APIs (Node/Express) aligned to `operationId`s
3.  JM Validator CLI (schema + signature + OAS opId cross-check)
4.  Demo HSBC Journey Pages (HTML/JS) wired to the bridge (`bridge_hello`, `performBiometric`, …)

* * *

1) Money-Transfer Journey Manifest v1.1 (JM)
============================================

1.1 Create folder
-----------------

```
hsbc-partner-sdk-ios/
└─ assets/
   └─ journeys/
      └─ money_transfer/
         ├─ manifest.mt.v1_1.json         # to generate
         └─ README.md
```

1.2 Prompt for Codex — **manifest generation**
----------------------------------------------

> **Task:** Create a JSON manifest file named `manifest.mt.v1_1.json` for the **money\_transfer** journey, compliant with **JM v1.1**.  
> **Requirements:**
> 
> *   Top-level fields: `manifestVersion:"1.1.0"`, `minSdk:"1.0.0"`, `journeyId:"money_transfer"`, `locales:["zh-CN","en-HK"]`, `oapiBundle:"https://api.sandbox.partner.hsbc.com/oapi/transfer-v1.json"`, `startStep:"mt_intro"`.
> *   `security`: `allowedOrigins:["https://ao.sandbox.partner.hsbc.com"]`, `pinning:true`, `attestation:{ios:true}`, `requireHandshake:true`.
> *   `headers` include partnerId, clientId, locale, contextToken, attestation placeholders (e.g., `"X-HSBC-Context":"$contextToken"`).
> *   Steps:
>     *   **mt\_intro** (web): URL `/mt/intro`, on `next→mt_recipient`, `cancel→end_cancel`.
>     *   **mt\_recipient** (web): URL `/mt/recipient`, `bridgeAllow:["validateRecipient"]`. Binding: `onEvent:"validateRecipient"` calls `operationId:"ValidateRecipient"`, `argsFrom:"event.payload"`, emits `recipientValid` or `recipientInvalid`. Transitions: `recipientValid→mt_amount`, `recipientInvalid→mt_recipient` with effect emit `showInlineErrors`, `back→mt_intro`.
>     *   **mt\_amount** (web): URL `/mt/amount`, `bridgeAllow:["calculateFees"]`. Binding: `onEvent:"calculateFees"` calls `operationId:"CalculateFee"`, `argsFrom:"event.payload"`, emits `feesReady`. `proceed→mt_review`, `back→mt_recipient`.
>     *   **mt\_review** (web): URL `/mt/review`, `bridgeAllow:["requestQuote"]`. Binding: `onEvent:"requestQuote"` calls `operationId:"GetQuote"`, emits `quoteReady` or `quoteError`. Transition on `confirm` guarded by `session.quoteId != null && now < session.quoteExpiry` to `mt_biometric`, else `to:"mt_amount"` with effect emit `QUOTE_EXPIRED`.
>     *   **mt\_biometric** (native): plugin `biometricAuth`, `params:{reason:"Authorize transfer"}`, on `success→mt_submit`, `failure→mt_review`.
>     *   **mt\_submit** (server): `idempotencyKey:"$session.correlationId"`, call `operationId:"CreateTransfer"`, `argsFrom:"session"`, `success→end_complete` with effect emit `MT_SUBMITTED`, `error→end_error`.
>     *   **end\_complete** (terminal): `result:{status:"COMPLETED"}`
>     *   **end\_cancel** (terminal): `result:{status:"CANCELLED"}`
>     *   **end\_error** (terminal): `result:{status:"FAILED"}`
> *   `resumePolicy`: snapshot on `["stepEnter","serverSuccess"]`.
> *   `signature`: include a **placeholder** string `"JWS-DETACHED-PLACEHOLDER"` (the validator will later replace this with a real JWS).  
>     **Output:** Valid, pretty-printed JSON saved to `assets/journeys/money_transfer/manifest.mt.v1_1.json`.
>     

1.3 Acceptance
--------------

*   File exists, JSON validates, contains all steps and fields above.
*   `bridgeAllow` present per web step; `idempotencyKey` present in `mt_submit`.

* * *

2) Mock Transfer APIs (Node/Express)
====================================

2.1 Create folder
-----------------

```
hsbc-partner-sdk-ios/
└─ mock/
   └─ money_transfer/
      ├─ package.json
      ├─ server.js
      ├─ .env.example
      └─ README.md
```

2.2 Prompt for Codex — **Node project scaffold**
------------------------------------------------

> Initialize a Node/Express service for Money Transfer mock APIs aligned to OpenAPI operationIds: `ValidateRecipient`, `CalculateFee`, `GetQuote`, `CreateTransfer`.  
> **Artifacts to generate:** `package.json`, `server.js`, `.env.example`, and a short `README.md` with run instructions.  
> **Behavior:**
> 
> *   `POST /transfer/validateRecipient` → body `{name,accountNumber,bankCode,country}`; return `{valid:boolean, reason?:string}`; make some bankCodes invalid for demo.
> *   `POST /transfer/calculateFee` → body `{amount,currency,route}`; return `{fee:number, rate:number}` with simple tiers; random small jitter.
> *   `POST /transfer/quote` (for `GetQuote`) → body `{amount,currency}`; return `{quoteId, rate, fee, expiresAt}` where `expiresAt` is ~60s in the future.
> *   `POST /transfer` (for `CreateTransfer`) → body includes `recipient`, `amount`, `currency`, optional `quoteId`; enforce **idempotency** via `X-Idempotency-Key` header; first call returns `{transferId,status:"SUBMITTED"}`, subsequent same key returns identical response; if quote expired → `409 QUOTE_EXPIRED`.  
>     **Non-functional:**
> *   Add `helmet`, `morgan`, JSON body parsing, `.env` with `PORT` and `MOCK_LATENCY_MS`.
> *   Latency injection via `MOCK_LATENCY_MS`.
> *   Start command: `npm start`.  
>     **Save files under:** `mock/money_transfer/`.
>     

2.3 Acceptance
--------------

*   `npm i && npm start` boots server, prints port.
*   Endpoints respond and idempotency works.

* * *

3) JM Validator CLI (schema + signature + OAS opId cross-check)
===============================================================

3.1 Create folder
-----------------

```
hsbc-partner-sdk-ios/
└─ tools/
   └─ jm-validator/
      ├─ package.json
      ├─ index.js
      ├─ schema.jm.v1_1.json
      ├─ keys/
      │  ├─ manifest.pub.pem
      │  └─ manifest.priv.pem             # example only; in real life store securely
      └─ README.md
```

3.2 Prompt for Codex — **schema file**
--------------------------------------

> Generate `schema.jm.v1_1.json` (JSON Schema Draft 7) that validates a JM v1.1 manifest with: required top-level fields (`manifestVersion`, `minSdk`, `journeyId`, `oapiBundle`, `startStep`, `security.allowedOrigins`, `steps`, `signature`).
> 
> *   `steps.*.type` enum: `web|native|server|terminal`.
> *   If `type=="web"`, require `url`; if `type=="native"`, require `plugin`; if `type=="server"`, require `call.operationId`.
> *   `bridgeAllow` (array of strings) allowed only for `web` steps.
> *   `resumePolicy.snapshotOn` array limited to `["stepEnter","serverSuccess"]`.  
>     Save to `tools/jm-validator/schema.jm.v1_1.json`.
>     

3.3 Prompt for Codex — **CLI implementation**
---------------------------------------------

> Create a Node CLI `index.js` with `#!/usr/bin/env node` that:
> 
> *   Accepts args: `--manifest <file> --oas <file-or-url> --pubkey <pem> [--sign <privkey>]`
> *   Loads and validates the manifest JSON against `schema.jm.v1_1.json` (use `ajv`).
> *   If `--oas` is provided, load the OpenAPI JSON and verify that every `operationId` used in `bindings` or `server` steps exists.
> *   If `--pubkey` provided, verify the manifest’s `signature` as **detached JWS over the canonicalized manifest without the signature field**.
> *   If `--sign` provided, replace the `signature` with a new detached JWS using the private key (ES256).
> *   Exit non-zero with a readable error list if validation fails.  
>     Save to `tools/jm-validator/index.js`. Also generate `package.json` with dependencies `ajv`, `node-jose` (or `jose`), `yargs`, `node-fetch`, and a README with usage examples.
>     

3.4 Acceptance
--------------

*   `node tools/jm-validator/index.js --manifest assets/journeys/money_transfer/manifest.mt.v1_1.json --oas https://api.sandbox.partner.hsbc.com/oapi/transfer-v1.json --pubkey tools/jm-validator/keys/manifest.pub.pem` prints **OK**.
*   `--sign tools/jm-validator/keys/manifest.priv.pem` updates signature.

* * *

4) Demo HSBC Journey Pages (HTML/JS) using the bridge
=====================================================

4.1 Create folder
-----------------

```
hsbc-partner-sdk-ios/
└─ demo-pages/
   └─ money_transfer/
      ├─ intro.html
      ├─ recipient.html
      ├─ amount.html
      ├─ review.html
      ├─ shared/
      │  ├─ bridge.js
      │  └─ styles.css
      └─ README.md
```

4.2 Prompt for Codex — **bridge helper (browser-side)**
-------------------------------------------------------

> Create `demo-pages/money_transfer/shared/bridge.js` that exposes:
> 
> *   `hsbc.sendEvent(name, payload)` → posts `{kind:'event', name, payload}` via `window.webkit.messageHandlers.hsbcBridge.postMessage(...)` if available.
> *   `hsbc.request(name, payload)` → returns a Promise and posts `{kind:'request', id, name, payload}`; resolves on SDK’s `resolve` message.
> *   On load, automatically fire `bridge_hello` event with `{origin: location.origin, pageNonce: crypto random}`.
> *   Add an event listener `hsbc.on(name, handler)` to handle SDK→Page events.
> *   Include input validation and error logging; no bank secrets in code.
>     

4.3 Prompt for Codex — **intro page**
-------------------------------------

> Create `demo-pages/money_transfer/intro.html` that:
> 
> *   Imports `shared/bridge.js` and `shared/styles.css`.
> *   Renders a short headline and a button “Start Transfer”.
> *   On click → `hsbc.sendEvent('next', {})`.
> *   Also render a “Cancel” button that emits `cancel`.
> *   Display a small console area that logs any incoming SDK events (e.g., `bridge_ready`).
>     

4.4 Prompt for Codex — **recipient page**
-----------------------------------------

> Create `recipient.html` with a simple form `{name, accountNumber, bankCode, country}`.
> 
> *   On submit, call `hsbc.request('validateRecipient', formData)`;
> *   If success → emit `recipientValid`; if error → emit `recipientInvalid` and show inline error.
> *   Add a back button emitting `back`.
>     

4.5 Prompt for Codex — **amount page**
--------------------------------------

> Create `amount.html` with fields `{amount, currency, route}`.
> 
> *   On change or on “Calculate Fees” click → `hsbc.request('calculateFees', {amount, currency, route})`;
> *   Show `{fee, rate}`; on success allow “Proceed” which emits `proceed`; back emits `back`.
>     

4.6 Prompt for Codex — **review page**
--------------------------------------

> Create `review.html` that shows recipient, amount, currency, and a “Get Quote” button → `hsbc.request('requestQuote', {...})`.
> 
> *   On success, show `quoteId`, `rate`, `fee`, and a countdown until `expiresAt`.
> *   “Confirm” emits `confirm`; if quote expired, show message and emit no event.
> *   “Back” emits `back`.
>     

4.7 Acceptance
--------------

*   Pages load in a normal browser (without SDK) and don’t error; buttons log intended actions.
*   When embedded in WKWebView with your SDK, they should progress through the journey via events/requests.

* * *

5) Wire demo pages to your SDK (dev-only)
=========================================

5.1 Prompt for Codex — **add a local file origin allow-list for dev**
---------------------------------------------------------------------

> In `Container/WebPolicies.swift`, add a **dev-only** mode that allows `file://` origins for local testing; controlled by a `featureFlags["allowFileOriginDev"] == true`. Document that production must use HTTPS HSBC domains only.

5.2 Prompt for Codex — **manifest override to local pages (dev)**
-----------------------------------------------------------------

> Create a developer README snippet that explains how to edit the JM URLs to point to `file:///…/demo-pages/money_transfer/*.html` for testing, and how to revert back to hosted URLs for real environments.

* * *

6) End-to-end acceptances (what to test after generation)
=========================================================

1.  **Validator**
    *   Sign `manifest.mt.v1_1.json`, verify signature, schema, and OAS operationIds → **OK**.
2.  **Mock APIs**
    *   `POST /transfer/validateRecipient` returns `{valid:true}` for happy path; some bankCodes invalid.
    *   `POST /transfer/quote` returns a quote with `expiresAt ~ 60s`.
    *   `POST /transfer` honors idempotency; repeat with same `X-Idempotency-Key` returns same result.
3.  **SDK + Demo pages**
    *   Start journey `"money_transfer"` → intro → recipient (validate) → amount (fees) → review (quote) → biometric (native) → submit (server) → `.completed`.
    *   Simulate expired quote → review page emits `QUOTE_EXPIRED` → transitions back to amount.
    *   Kill app after amount → resume token allows continuation at review (or amount), per snapshot policy.
4.  **Security controls (dev simulation)**
    *   With `allowFileOriginDev=false`, file URL origin is blocked.
    *   With production origin mismatched, bridge handshake fails (`ORIGIN_BLOCKED`).
    *   Unlisted bridge methods return `BRIDGE_FORBIDDEN`.

* * *

Bonus: one-liner master prompt (paste to Codex to generate everything stepwise)
-------------------------------------------------------------------------------

> “We are building a hardened, journey-agnostic iOS SDK. Generate, file by file, the following artifacts using the instructions I’ll paste: (1) `assets/journeys/money_transfer/manifest.mt.v1_1.json` (JM v1.1 per spec, with placeholder detached JWS), (2) `mock/money_transfer` Node/Express server implementing `ValidateRecipient`, `CalculateFee`, `GetQuote`, `CreateTransfer` with idempotency, (3) `tools/jm-validator` CLI that validates manifest against schema, verifies signature, and cross-checks operationIds with OAS, (4) `demo-pages/money_transfer` HTML pages + `shared/bridge.js` that speak the bridge protocol (`bridge_hello`, `request`, `event`). For each file, I’ll give you a targeted prompt. Do not print code until I ask for each file; confirm readiness and then emit the code block only for that file.”

* * *

When you’re ready, start with **Step 1.2** and paste the **manifest generation prompt** into Copilot/Codex. I can sit on your shoulder for each step and tune the prompts if the generator goes off-spec.

## Prompt:
when I creat folder hsbc-partner-sdk-ios/
├─ sdk/, cd sdk, execute command Create a Swift Package manifest (Swift 5.9) named HSBCPartnerSDK. Platforms: iOS 14+. Products:

library HSBCPartnerSDKCore target HSBCPartnerSDKCore

library HSBCPlugins target HSBCPlugins
Targets: HSBCPartnerSDKCore (process Resources/ao_test.html and PrivacyInfo.xcprivacy), HSBCPlugins, and a test target HSBCPartnerSDKCoreTests. No external dependencies.  did get the assocaited result. please check

## Response:
Thought for 47s

Got you. It sounds like you ran a _natural-language_ “command” and nothing happened. On macOS you must either (a) run real shell commands, or (b) paste a prompt into Copilot/Codex to generate the `Package.swift` file, then save it. Here’s a quick rescue plan that will 100% get your Swift package building.

* * *

1) Verify you’re in the right place
-----------------------------------

In Terminal:

```bash
cd hsbc-partner-sdk-ios
mkdir -p sdk
cd sdk
```

Initialize a library package (this creates a starter `Package.swift`):

```bash
swift package init --type library
```

> If you already did this, you can overwrite the `Package.swift` in the next step.

* * *

2) Create the folders the manifest will reference
-------------------------------------------------

SwiftPM only accepts resource paths that **exist** under the **target** directory.

```bash
mkdir -p Sources/HSBCPartnerSDKCore/Public
mkdir -p Sources/HSBCPartnerSDKCore/Resources
mkdir -p Sources/HSBCPlugins
mkdir -p Tests/HSBCPartnerSDKCoreTests
# stub resource files so the build doesn't fail
: > Sources/HSBCPartnerSDKCore/Resources/ao_test.html
: > Sources/HSBCPartnerSDKCore/PrivacyInfo.xcprivacy
```

* * *

3) Use Codex to generate the manifest (prompt)
----------------------------------------------

Open **VS Code** → open the `sdk/` folder → open Copilot/Codex Chat and paste:

> **Prompt:**  
> “Generate a Swift Package manifest (`Package.swift`) for Swift 5.9 named `HSBCPartnerSDK`. Platforms: iOS 14+.  
> Products:
> 
> *   library `HSBCPartnerSDKCore` built from target `HSBCPartnerSDKCore`
> *   library `HSBCPlugins` built from target `HSBCPlugins`  
>     Targets:
> *   target `HSBCPartnerSDKCore` that **processes** two resources: `Resources/ao_test.html` and `PrivacyInfo.xcprivacy` (both relative to the target directory `Sources/HSBCPartnerSDKCore/`)
> *   target `HSBCPlugins` (no resources)
> *   test target `HSBCPartnerSDKCoreTests` depending on `HSBCPartnerSDKCore`  
>     No external dependencies. Use `defaultLocalization: "en"`.”
>     

Copy the generated content into `sdk/Package.swift` (overwriting the starter).

> Tip: In SwiftPM, resource paths are **relative to the target’s source dir**. With our layout, the correct entries look like:
> 
> ```swift
> .process("Resources/ao_test.html"),
> .process("PrivacyInfo.xcprivacy")
> ```
> 
> (because both files live under `Sources/HSBCPartnerSDKCore/...`)

* * *

4) Quick sanity build
---------------------

```bash
swift build -v
```

You should see a successful build. If you get an error like:

*   **“no such file or directory: …/Resources/ao\_test.html”**  
    → Make sure the file exists **exactly** at `Sources/HSBCPartnerSDKCore/Resources/ao_test.html`.
*   **“found nil for target HSBCPartnerSDKCore”**  
    → Confirm your folder is `Sources/HSBCPartnerSDKCore/` (spelling and case matter).
*   **“invalid resource path”**  
    → Ensure you used `.process("Resources/ao_test.html")` and **not** an absolute path.

* * *

5) (Optional) Minimal test file so `swift test` passes
------------------------------------------------------

Use Codex to generate a tiny test:

> **Prompt:**  
> “Create `Tests/HSBCPartnerSDKCoreTests/HSBCPartnerSDKCoreTests.swift` with a single `XCTestCase` that calls `XCTAssertTrue(true)` so tests pass.”

Then run:

```bash
swift test -v
```

* * *

6) If VS Code doesn’t pick it up in Xcode
-----------------------------------------

Open the package in Xcode (good for building the XCFramework later):

*   Xcode → File → Open → choose the **sdk/** folder (not the repo root).
*   In **Product → Scheme → Manage Schemes**, make sure `HSBCPartnerSDKCore` is shared.

* * *

7) Quick checklist (what “correct” looks like)
----------------------------------------------

*   `sdk/Package.swift` exists and references:
    *   product libraries `HSBCPartnerSDKCore`, `HSBCPlugins`
    *   target `HSBCPartnerSDKCore` with resources:
        *   `Resources/ao_test.html`
        *   `PrivacyInfo.xcprivacy`
    *   target `HSBCPlugins`
    *   test target `HSBCPartnerSDKCoreTests`
*   File layout:
    *   `Sources/HSBCPartnerSDKCore/Resources/ao_test.html`
    *   `Sources/HSBCPartnerSDKCore/PrivacyInfo.xcprivacy`
*   `swift build` and `swift test` both succeed.

* * *

If it still fails, tell me the exact error text
-----------------------------------------------

If you hit an error message, paste the **full error line** and I’ll pinpoint the fix (it’s usually a path or casing mismatch in SwiftPM).

## Prompt:
please also verify the 后续 task from 2) to 19) , regenerate step by step with prompt follow above approach

## Response:
Thought for 13m 51s

Great — I’ll **verify and regenerate** the **subsequent (后续) steps 2 → 19** in the _same “no code, prompt‑only”_ style you asked for.  
Each step includes:

*   **Goal** (what you’ll achieve)
*   **Where to create files** (paths to keep SwiftPM happy)
*   **Prompt for Codex** (paste directly into Copilot/Codex Chat)
*   **Acceptance checks** (how you know it’s correct)

> ⚠️ Reminder: keep resource files under `Sources/<TargetName>/...` or SwiftPM will reject them. Spelling and casing matter.

* * *

2) Public API surface (generic, stable)
---------------------------------------

**Goal**  
Expose a public SDK entry point that does **not** mention any specific business journey.

**Where**  
`sdk/Sources/HSBCPartnerSDKCore/Public/HSBCPartnerSdk.swift`

**Prompt for Codex**

> **Create `Public/HSBCPartnerSdk.swift`** (Swift 5.9, iOS 14+, no external deps).  
> Define:
> 
> *   `public struct HSBCInitConfig { environment: enum { sandbox, production }; partnerId, clientId, redirectScheme, locale; remoteConfigURL: URL?; featureFlags: [String:Bool]; telemetryOptIn: Bool }`
> *   `public struct JourneyStartParams { journeyId: String; contextToken: String; resumeToken: String?; productCode: String?; campaignId: String?; trackingContext: [String:String]; capabilitiesOverride: [String:Bool]? }`
> *   `public enum JourneyResult { case completed(payload:[String:Any]); case pending(payload:[String:Any]); case cancelled; case failed(code:String,message:String,recoverable:Bool) }`
> *   `public enum HSBCPartnerSdk` with:
>     *   `initialize(config:)`
>     *   `setEventListener((_ name:String,_ attrs:[String:Any])->Void)`
>     *   `startJourney(from: UIViewController, params: JourneyStartParams, completion: @escaping(JourneyResult)->Void)` that delegates to a `RuntimeEngine.start(...)`.  
>         Add SwiftDoc comments on all symbols. Keep API surface minimal and generic.
>         

**Acceptance**

*   `swift build` succeeds.
*   These types are visible when the demo app imports `HSBCPartnerSDKCore`.

* * *

3) Event bus & telemetry (privacy‑safe)
---------------------------------------

**Goal**  
Emit structured, privacy‑safe events and generate `traceparent` for distributed tracing.

**Where**

*   `sdk/Sources/HSBCPartnerSDKCore/Runtime/EventBus.swift`
*   `sdk/Sources/HSBCPartnerSDKCore/Telemetry/Telemetry.swift`

**Prompt for Codex**

> **Create `Runtime/EventBus.swift`** with:
> 
> *   `struct JourneyEvent { name:String, attributes:[String:Any], ts:Date, correlationId:String }`
> *   `final class EventBus` singleton, thread‑safe, allowing one external listener (set by `HSBCPartnerSdk.setEventListener`) and internal `emit(_:)` calls.
> *   Helpers: `journeyBegin`, `stepEnter`, `apiCall`, `bridge`, `error`, `result`.
>     
> 
> **Create `Telemetry/Telemetry.swift`** with:
> 
> *   `func newTraceparent() -> String` (W3C Trace Context 00 format; random trace/span ids)
> *   `func withSpan(name:String, attributes:[String:Any], _ block: () -> Void)`; no‑op backend but capture timings for logs/events.  
>     Keep payloads PII‑free.
>     

**Acceptance**

*   You can call `EventBus.shared.emit(...)` from anywhere.
*   `newTraceparent()` returns a `00-<trace>-<span>-01` string.

* * *

4) Security basics: KeyStore + JWS helper
-----------------------------------------

**Goal**  
Verify/supply JWS signatures (ES256) and manage bank public keys.

**Where**

*   `sdk/Sources/HSBCPartnerSDKCore/Security/KeyStore.swift`
*   `sdk/Sources/HSBCPartnerSDKCore/Security/JWS.swift`

**Prompt for Codex**

> **Create `Security/KeyStore.swift`**: in‑memory map `kid:String -> SecKey` with `update(keys:)` and `key(for:)`. Load initial keys from a small bundled JSON (ok to stub for now), and expose a method to refresh from remote config later.
> 
> **Create `Security/JWS.swift`**: minimal ES256 JWS helper **without external libs** using CryptoKit.
> 
> *   `enum JWSError: Error {...}`
> *   `struct JWS { static func verify(compact:String, key: SecKey) throws -> Data /* payload */ }`
> *   `struct JWSSigner { init(privateKey: P256.Signing.PrivateKey, kid:String); func sign(payload: Data) throws -> String /* compact */ }`  
>     Only support **compact serialization**; Base64URL; header `{ "alg":"ES256","kid":"..." }`.
>     

**Acceptance**

*   A quick unit test can sign and verify a payload in‑memory.
*   No external lib imports.

* * *

5) Remote config & Manifest loader (JM v1.1)
--------------------------------------------

**Goal**  
Load and validate a signed Journey Manifest v1.1 and basic schema rules.

**Where**

*   `sdk/Sources/HSBCPartnerSDKCore/Manifest/ManifestModels.swift`
*   `sdk/Sources/HSBCPartnerSDKCore/Manifest/ManifestLoader.swift`
*   `sdk/Sources/HSBCPartnerSDKCore/Manifest/ManifestSchema.json` (optional placeholder)

**Prompt for Codex**

> **Create `Manifest/ManifestModels.swift`**: Codable structs for JM v1.1:
> 
> *   Top level: `manifestVersion, minSdk, journeyId, oapiBundle: URL, locales:[String]?, startStep, headers:[String:String], security:{allowedOrigins:[URL], pinning:Bool, attestation:[String:Bool]?, requireHandshake:Bool}, resumePolicy:{snapshotOn:[String]}?, steps:[String:Step], signature:String`
> *   `Step` includes: `type: enum web|native|server|terminal`, `url:URL?`, `plugin:String?`, `params:[String:AnyCodable]?`, `timeoutMs:Int?`, `bindings:[Binding]?`, `on:[String:Transition]?`, `result:[String:AnyCodable]?`, `bridgeAllow:[String]?`, `idempotencyKey:String?`
> *   `Binding { onEvent:String, call:{ operationId:String, argsFrom:String?, headers:[String:String]? }, onSuccessEmit:String?, onErrorEmit:String? }`
> *   `Transition { to:String?, emit:BridgeEvent?, guardExpr:String? }`  
>     Include a minimal `AnyCodable` to carry arbitrary JSON.
>     
> 
> **Create `Manifest/ManifestLoader.swift`**:
> 
> *   `func load(journeyId:String, contextToken:String, config:HSBCInitConfig, keyStore:KeyStore) async throws -> Manifest`
> *   Resolve manifest URL from `remoteConfigURL` + `journeyId` (stub ok).
> *   Fetch manifest JSON (allow local file URL in dev), verify **detached JWS** signature (temporarily allow bypass behind a feature flag), validate `minSdk` and that `security.allowedOrigins` is non‑empty.
> *   Optional: `validateAgainstSchema(...)` (stub: sanity checks).  
>     Return the decoded manifest.
>     

**Acceptance**

*   Loader returns a Manifest from a local JSON during dev.
*   It fails when `allowedOrigins` is empty or `minSdk` is higher than current.

* * *

6) OpenAPI resolver & API client (retries & idempotency)
--------------------------------------------------------

**Goal**  
Map `operationId` → (method, path) and call HSBC APIs robustly.

**Where**

*   `sdk/Sources/HSBCPartnerSDKCore/Network/OpenAPIResolver.swift`
*   `sdk/Sources/HSBCPartnerSDKCore/Network/ApiClient.swift`

**Prompt for Codex**

> **Create `Network/OpenAPIResolver.swift`**:
> 
> *   Load a JSON OpenAPI 3.0 document (YAML support later).
> *   Build `operationId -> (method:String, path:String)` map.
> *   Provide `resolve(operationId:)`.
> *   Helper: build `URLRequest` from baseURL, method, path, body `[String:Any]?`, extra headers.
>     
> 
> **Create `Network/ApiClient.swift`**:
> 
> *   `init(baseURL: URL, resolver: OpenAPIResolver, pinningEnabled: Bool)`
> *   `async func call(operationId:String, body:[String:Any]?, headers:[String:String], idempotencyKey:String?) throws -> (status:Int, headers:[AnyHashable:Any], body:Data)`
> *   Add **exponential backoff + jitter** for 408/429/5xx (max 3 tries); observe `Retry-After`.
> *   Inject headers: `traceparent` (from Telemetry), and `X-Idempotency-Key` if provided.
> *   Map HTTP codes to canonical errors (to be defined in step 13).
>     

**Acceptance**

*   Resolves an operationId from a tiny fake OAS JSON and builds a `URLRequest`.
*   Retries occur on simulated 429/500.

* * *

7) Session & resume (PII‑free snapshots)
----------------------------------------

**Goal**  
Allow safe resume with PII‑free state saved in Keychain.

**Where**  
`sdk/Sources/HSBCPartnerSDKCore/Runtime/SessionManager.swift`

**Prompt for Codex**

> **Create `Runtime/SessionManager.swift`**:
> 
> *   Singleton with `correlationId:String` per session, `contextToken`, `resumeToken:String?`, `stepPointer:String?`, `idempotencyKey:String`.
> *   `func saveSnapshot(journeyId:String, stepId:String)` and `func loadSnapshot(resumeToken:String) -> Snapshot?`.
> *   Use Keychain (kSecClassGenericPassword, `kSecAttrAccessibleAfterFirstUnlockThisDeviceOnly`).
> *   Snapshots store **no PII**; only `{journeyId, stepPointer, idempotencyKey, ts}`.
>     

**Acceptance**

*   Saving & loading snapshot works (mock Keychain in tests if needed).
*   Resume token is opaque.

* * *

8) Hybrid container & web policies (SSO handoff)
------------------------------------------------

**Goal**  
Use `ASWebAuthenticationSession` for OIDC + `WKWebView` for steps with origin guard.

**Where**

*   `sdk/Sources/HSBCPartnerSDKCore/Container/HybridContainer.swift`
*   `sdk/Sources/HSBCPartnerSDKCore/Container/WebPolicies.swift`

**Prompt for Codex**

> **Create `Container/HybridContainer.swift`**:
> 
> *   `async func signInIfNeeded(authURL:URL, redirectScheme:String) throws -> URL` using `ASWebAuthenticationSession` (`prefersEphemeralWebBrowserSession = true`).
> *   `func presentStepWebView(on:UIViewController, url:URL, allowedOrigins:[URL], bridge:Bridge) -> UIViewController` to present a modal WKWebView with strict config (JS enabled, no new windows, no nav gestures), navigation delegate that **enforces origin** and blocks non‑HTTPS.
>     
> 
> **Create `Container/WebPolicies.swift`**:
> 
> *   `func isAllowed(origin:URL, allowed:[URL]) -> Bool` (host+scheme match; https only).
> *   Comment: Prod uses ATS (TLS 1.2+), `WKAppBoundDomains` must list HSBC domains; dev can allow `file://` behind feature flag.
>     

**Acceptance**

*   You can present a WKWebView in demo and origin checks are enforced.
*   ASWebAuth returns to app via redirect scheme (you’ll wire later).

* * *

9) Bridge v1.1 (handshake, signed envelopes, allow‑list)
--------------------------------------------------------

**Goal**  
Secure JS bridge with handshake and per‑step method allow‑list.

**Where**  
`sdk/Sources/HSBCPartnerSDKCore/Container/Bridge.swift`

**Prompt for Codex**

> **Create `Container/Bridge.swift`** for WKWebView:
> 
> *   Register `hsbcBridge` message handler.
> *   States: `.notReady`, `.ready`.
> *   On page event `{kind:'event', name:'bridge_hello', payload:{origin,pageNonce}}`:
>     *   Verify origin via `WebPolicies`.
>     *   Collect attestation (stubbed `AttestationCollector.collect()` returns a string).
>     *   Reply `{kind:'event', name:'bridge_ready', payload:{ sdkCapabilities:[...], sessionProofJws }, meta:{ts,nonce,bridgeVersion:'1.1',sdkVersion:'1.0.0',traceparent}, sig:<JWS>}`
> *   For `{kind:'request', id, name, payload}`:
>     *   Reject if not `.ready` or `name` not in `allowedMethodsForStep` (error `BRIDGE_FORBIDDEN`).
>     *   Route `performBiometric` to plugin; return signed `resolve` envelope.
> *   All SDK→Page responses include `sig` (ES256 JWS) over `(name,payload,meta)` with a fresh nonce.
>     

**Acceptance**

*   The page can `bridge_hello` → receives `bridge_ready`.
*   Calling an unlisted method returns an error event with `BRIDGE_FORBIDDEN`.

* * *

10) State machine (steps, transitions, bindings)
------------------------------------------------

**Goal**  
Deterministic flow control driven by manifest.

**Where**  
`sdk/Sources/HSBCPartnerSDKCore/Runtime/StateMachine.swift`

**Prompt for Codex**

> **Create `Runtime/StateMachine.swift`**:
> 
> *   Holds `currentStepId:String` and `steps:[String:Step]`.
> *   `handleEvent(name:String, payload:[String:Any])`:
>     *   Find transition in `steps[current].on[name]`.
>     *   Evaluate optional `guardExpr` with a tiny evaluator (support `==, !=, >, <, &&, ||` and lookups like `session.quoteId`).
>     *   Execute optional `effect.emit` back to page/bridge.
>     *   Move to `to` and emit `STEP_ENTER/EXIT` events via EventBus.
> *   On step enter: start `timeoutMs` timer that emits `"timeout"`.
> *   `bindings`: when the page emits `Binding.onEvent`, call `ApiClient` with args from `event.payload` (dot‑path extractor), set idempotency key if defined on step, then emit `onSuccessEmit` or `onErrorEmit`.
>     

**Acceptance**

*   A simple 3‑step manifest transitions correctly; timeout triggers expected event.
*   Binding calls trigger ApiClient and emit events.

* * *

11) Runtime engine (wire all subsystems)
----------------------------------------

**Goal**  
Tie everything together and produce a `JourneyResult`.

**Where**  
`sdk/Sources/HSBCPartnerSDKCore/Runtime/RuntimeEngine.swift`

**Prompt for Codex**

> **Create `Runtime/RuntimeEngine.swift`** with:
> 
> *   `start(from: UIViewController, params: JourneyStartParams, completion:@escaping(JourneyResult)->Void)`  
>     Steps:
>     1.  Load remote config (stub ok), then `ManifestLoader.load(...)`.
> 2.  Resolve OAS (`OpenAPIResolver`) and create `ApiClient` with base URL from OAS servers.
> 3.  If JM requires auth: call `HybridContainer.signInIfNeeded(...)` (stub URL ok).
> 4.  Build `Bridge` with `allowedMethodsForStep` from JM; present first step via `presentStepWebView(...)`.
> 5.  Wire Bridge → StateMachine (`handleEvent`), and StateMachine → Bridge (effects).
> 6.  Save snapshots on step enter and server success via `SessionManager`.
> 7.  On terminal step, map to `JourneyResult` and call `completion`.
> 8.  Emit canonical events via EventBus at each milestone.
>     

**Acceptance**

*   You can run a dummy flow end‑to‑end with local pages and see `JourneyResult.completed`.

* * *

12) Plugins: Biometric (MVP) & plugin interface
-----------------------------------------------

**Goal**  
Standard plugin interface and a working biometric plugin.

**Where**  
`sdk/Sources/HSBCPlugins/BiometricAuth.swift`

**Prompt for Codex**

> **Create `HSBCPlugins/BiometricAuth.swift`**:
> 
> *   `public protocol JourneyPlugin { var name:String { get } ; func canHandle(_ method:String) -> Bool ; func handle(_ method:String, params:[String:Any]) async throws -> [String:Any] }`
> *   `public final class BiometricAuthPlugin: JourneyPlugin` using `LocalAuthentication` to evaluate policy; accept method `performBiometric` with `reason:String`, return `{"biometric":"PASS"}` or throw an error with code `SCA_REQUIRED` / `USER_CANCELLED`.
> *   Expose a way for the Bridge to register plugins (e.g., `PluginRegistry.shared`).
>     

**Acceptance**

*   Calling `performBiometric` through the Bridge resolves to success on FaceID/TouchID‑enabled simulator or returns a sensible error.

* * *

13) Error taxonomy & mapping (normalized)
-----------------------------------------

**Goal**  
Consistent SDK error codes and mapping from HTTP/network conditions.

**Where**

*   `sdk/Sources/HSBCPartnerSDKCore/Network/ApiClient.swift` (augment)
*   `sdk/Sources/HSBCPartnerSDKCore/Runtime/RuntimeEngine.swift` (map to JourneyResult)

**Prompt for Codex**

> **Extend ApiClient & Runtime**:
> 
> *   Define `enum SdkErrorCode: String { AUTH_EXPIRED, PINNING_FAIL, ORIGIN_BLOCKED, NET_TIMEOUT, RATE_LIMITED, VALIDATION_FAIL, IDEMPOTENT_REPLAY, SCA_REQUIRED, COMPLIANCE_HOLD, MORE_INFO, UNKNOWN }`
> *   Map HTTP codes: 401/403→`AUTH_EXPIRED`; 408→`NET_TIMEOUT`; 409 with idempotency→`IDEMPOTENT_REPLAY`; 422/400 with field errors→`VALIDATION_FAIL`; 429→`RATE_LIMITED`.
> *   RuntimeEngine converts these into `.failed(code:message:recoverable:)` or into JM‑declared transitions (default is to surface as failed with `recoverable` true for 408/429).
>     

**Acceptance**

*   Simulated 429 triggers retries; after max retries, you get `.failed(code:"RATE_LIMITED",…)`.
*   An unrecognized error returns `.failed(code:"UNKNOWN",…)`.

* * *

14) Demo app (to drive and see results)
---------------------------------------

**Goal**  
A tiny app that initializes the SDK, starts a journey, and displays results.

**Where**  
`examples/PartnerDemoApp/` (Xcode project created via Xcode UI)

**Prompt for Codex**

> **In `ContentView.swift`** (SwiftUI):
> 
> *   Add two buttons: “Initialize SDK” and “Start Money Transfer”.
> *   On initialize: create `HSBCInitConfig` (sandbox, partnerId/clientId demo values, redirect scheme, locale), call `HSBCPartnerSdk.initialize` and `setEventListener` (print events).
> *   On start: call `HSBCPartnerSdk.startJourney` with `journeyId:"money_transfer"`, `contextToken:"mtok_demo"`, show the returned `JourneyResult` in a `Text` below.
> *   Implement `topController()` helper to get the presenting VC if needed.  
>     Keep UI minimal.
>     

**Acceptance**

*   You can run, click, see the web container load your local/demo pages, and get a `.completed` or `.pending`.

* * *

15) Tests (unit + UI)
---------------------

**Goal**  
Confidence that core contracts behave correctly.

**Where**

*   `sdk/Tests/HSBCPartnerSDKCoreTests/ManifestTests.swift`
*   `sdk/Tests/HSBCPartnerSDKCoreTests/BridgeHandshakeTests.swift`
*   `sdk/Tests/HSBCPartnerSDKCoreTests/StateMachineTests.swift`
*   `examples/PartnerDemoApp/PartnerDemoAppUITests/PartnerDemoAppUITests.swift`

**Prompt for Codex**

> **Create four tests**:
> 
> 1.  `ManifestTests`: load a sample JM v1.1 JSON, assert required fields, `minSdk` check (pass/fail cases), `allowedOrigins` non‑empty.
> 2.  `BridgeHandshakeTests`: simulate a `bridge_hello` from allowed origin and assert `bridge_ready` with signed envelope; simulate blocked origin returns `ORIGIN_BLOCKED`.
> 3.  `StateMachineTests`: dummy steps A→B→C via events; `timeoutMs` triggers an event; `guardExpr` blocks or allows transition.
> 4.  **UI Test**: drive the demo app—tap Initialize; Start; in the WKWebView press test buttons (READY/biometric/confirm); assert status text contains “Completed” or “Pending”.
>     

**Acceptance**

*   `swift test` passes.
*   Xcode UI test runs to completion on simulator.

* * *

16) CI & packaging (artifact you can hand over)
-----------------------------------------------

**Goal**  
Build, test, and publish XCFramework on pushes/tags.

**Where**

*   `.github/workflows/ios-sdk.yml`
*   `sdk/Plugins/BuildXCFrameworkPlugin/...` (if not already)
*   `scripts/build_xcframework.sh` (already in earlier steps)

**Prompt for Codex**

> **Generate `.github/workflows/ios-sdk.yml`** that:
> 
> *   Runs on PRs and tags `v*`.
> *   Selects Xcode 15.
> *   Caches SPM.
> *   Runs `swift build` & `swift test` in `sdk/`.
> *   Executes `scripts/build_xcframework.sh`.
> *   Uploads `dist/HSBCPartnerSDKCore.xcframework` as artifact.
> *   On tag, creates a GitHub Release and attaches the XCFramework and checksum.  
>     Ensure job name, steps, and paths match our repo layout.
>     

**Acceptance**

*   A push triggers CI, producing a downloadable artifact.
*   Tagging `v0.1.0` auto‑creates a release.

* * *

17) Harden & extend checklist (security backlog doc)
----------------------------------------------------

**Goal**  
Track all hardening tasks in one place.

**Where**  
`SECURITY_BACKLOG.md` at repo root

**Prompt for Codex**

> **Create `SECURITY_BACKLOG.md`** listing:
> 
> *   Cert pinning (dual pins current+next) for native requests.
> *   ATS hardened; WKAppBoundDomains; CSP example.
> *   Attestation (DeviceCheck/App Attest) header `X‑HSBC‑Attest`.
> *   JWS/JWE for all Bridge responses (ES256; optional JWE for sensitive plugin outputs).
> *   Resume token encryption & rotation.
> *   Kill‑switch via signed remote config.
> *   Per‑journey scopes; least privilege.
> *   OpenTelemetry metrics and budgets (TTFP ≤ 1200ms on 4G; p95 server step ≤ 800ms).
> *   Chaos tests for offline/retry/idempotency.  
>     Include owner, priority, and status columns.
>     

**Acceptance**

*   The doc exists; your team can use it as a living backlog.

* * *

18) Ready‑to‑paste “macro” prompts (fast‑track generation)
----------------------------------------------------------

**Goal**  
Have a few big, “one‑shot” prompts you can reuse to generate modules quickly.

**Where**  
`docs/DEV_MACRO_PROMPTS.md`

**Prompt for Codex**

> **Create `docs/DEV_MACRO_PROMPTS.md`** with 4 macro prompts:
> 
> 1.  **Public API + EventBus + Telemetry** (single prompt describing all three files and their contracts).
> 2.  **Manifest + Loader + OpenAPI + ApiClient** (manifest models/loader, resolver, client with retries/idempotency).
> 3.  **HybridContainer + WebPolicies + Bridge v1.1** (ASWebAuth, WKWebView, handshake, signed envelopes, allow‑list).
> 4.  **StateMachine + RuntimeEngine + SessionManager** (transitions, bindings, snapshots, completion mapping).  
>     Each macro prompt must restate file paths and acceptance checks.
>     

**Acceptance**

*   The doc contains clear, copy‑ready macro prompts.

* * *

19) What “done” looks like (self‑checklist)
-------------------------------------------

**Goal**  
Objective end‑to‑end definition of done.

**Where**  
`DONE_CHECKLIST.md`

**Prompt for Codex**

> **Create `DONE_CHECKLIST.md`** that lists:
> 
> *   ✅ Build: `swift build`, `swift test` pass locally and in CI.
> *   ✅ Demo: Journey “money\_transfer” runs intro→recipient→amount→review→biometric→submit→completed on simulator.
> *   ✅ Errors: Simulated 429/500 retries; 401→AUTH\_EXPIRED; origin blocked path; handshake required path.
> *   ✅ Security: ATS/WKAppBoundDomains set in demo; JWS signing on Bridge responses on; feature flag to allow `file://` origins **off** by default.
> *   ✅ Packaging: XCFramework built; checksum generated; release tag attaches artifact.
> *   ✅ Docs: SECURITY\_BACKLOG, DEV\_MACRO\_PROMPTS present and up to date.
>     

**Acceptance**

*   The checklist reflects reality after your first successful end‑to‑end run.

* * *

### Final notes

*   Keep using **prompts** in Copilot/Codex exactly as above; let the generator create the files and then run the **Acceptance** checks.
*   If any prompt yields incomplete code, **re‑prompt** with “regenerate only the missing parts” and restate the file path.
*   Any SwiftPM error usually boils down to **incorrect file paths** or **resource placement** — double‑check before debugging logic.

If you get stuck on any single step (2 → 19), paste the exact error or behavior and I’ll help you adjust the prompt precisely.

## Prompt:
please review all of above context, regenerate the detail low lever design doc, The doc contains clear , well structure, professional

## Response:
Thought for 9m 4s

Below is a consolidated **low‑level design document** for the **HSBC Journey‑Agnostic iOS SDK** based on everything we’ve discussed. It’s structured so you can share it with architects, security, and developers as a formal design.

* * *

HSBC Journey‑Agnostic Partner SDK (iOS)
=======================================

**Low‑Level Design Document (LLD)**

* * *

1\. Introduction
----------------

### 1.1 Purpose

This document defines the **low‑level technical design** of the **HSBC Journey‑Agnostic Partner SDK for iOS**. The SDK enables partners (e.g. Ctrip) to embed HSBC business journeys—such as **Account Opening (AO)** and **Money Transfer (MT)**—directly inside their apps while HSBC retains control over:

*   Business flows and UI pages
*   Security, compliance, and KYC
*   API contracts and orchestration

Journeys are configured via **signed manifests** and **OpenAPI specifications**, so the SDK itself is **generic** and does not need to change for each new business scenario.

### 1.2 Scope

This LLD covers:

*   **iOS SDK** design (Swift, iOS 14+)
*   Journey‑agnostic runtime and orchestration
*   Security and compliance mechanisms
*   Data contracts (manifest, bridge, APIs)
*   Example journey: **Money Transfer (MT)**
*   Extensibility to other journeys (e.g. AO, card application)

Other platforms (Android/Web) should mirror the same abstractions but are out of scope for this document.

* * *

2\. Terminology
---------------

*   **Journey** – A business flow (e.g., `account_opening`, `money_transfer`).
*   **Journey Manifest (JM)** – Signed JSON that declares steps, transitions, bindings, and security policy for a given journey.
*   **OpenAPI Bundle (OAB)** – OpenAPI 3 specification with `operationId`s referenced by the manifest.
*   **Runtime Engine (RE)** – Core SDK orchestration component that interprets JM + OAB and runs the journey.
*   **Bridge** – Secure JS/native communication layer between HSBC hosted pages and the SDK.
*   **Plugins** – Optional native modules (e.g., biometric auth, document capture).
*   **Context Token** – Short‑lived, server‑to‑server minted token containing consent and context (e.g., prefill or transfer session).
*   **Resume Token** – Opaque token (PII‑free) that allows resuming a partially completed journey.

* * *

3\. Requirements
----------------

### 3.1 Functional

1.  Allow partner apps to start any supported HSBC journey by calling a **single public API**:
    *   `startJourney(journeyId:..., contextToken:..., ...)`.
2.  Load **HSBC‑hosted web pages** in an in‑app container (no external Safari app jump).
3.  Use **signed manifests** to describe:
    *   Which pages to load
    *   How steps transition
    *   Which APIs to call and when
    *   Which native capabilities can be invoked
4.  Support **multiple journey types** with no SDK code changes (only new manifests and APIs).
5.  Provide a **completion result** summarizing the outcome:
    *   Completed / Pending / Cancelled / Failed (+ error code).
6.  Support **Resume** for partially completed journeys (where allowed by HSBC policy).
7.  Expose **privacy‑safe events** to the partner for analytics.

### 3.2 Non‑Functional

*   iOS 14+, Swift 5.9+
*   Strong security (TLS, pinning, JWS, attestation)
*   GDPR, PDPA, CSL, and local data laws alignment
*   High availability; graceful network failure handling (retries, backoff)
*   Backward compatible public API (SemVer)
*   Minimal footprint; modular plugins; efficient memory & CPU usage

* * *

4\. High‑Level Architecture
---------------------------

### 4.1 Component Overview

```text
Partner iOS App
  └─ HSBCPartnerSdk (public API)
      ├─ RuntimeEngine
      │   ├─ ManifestLoader
      │   ├─ StateMachine
      │   ├─ SessionManager
      │   ├─ ApiClient + OpenAPIResolver
      │   └─ Telemetry + EventBus
      ├─ Container (HybridContainer)
      │   ├─ Auth (ASWebAuthenticationSession)
      │   └─ Web (WKWebView + WebPolicies)
      ├─ Bridge (JS ↔ Native)
      ├─ Security
      │   ├─ KeyStore
      │   └─ JWS / AttestationCollector
      └─ Plugins (e.g. BiometricAuth)
HSBC Infrastructure
  ├─ Journey Manifests (JM)
  ├─ OpenAPI Bundles (OAB)
  ├─ AO / MT hosted web flows
  ├─ Partner API Gateway (OAuth2, KYC, MT, etc.)
  └─ Remote Config Service
```

The partner integrates only with `HSBCPartnerSdk`. All journey specifics are **externalized** into JM + OAB and HSBC’s hosted pages.

* * *

5\. Public SDK Interface
------------------------

### 5.1 Initialization & Configuration

**Type:** `HSBCInitConfig`

Fields:

*   `environment: { sandbox | production }`
*   `partnerId: String` – HSBC‑assigned partner identifier
*   `clientId: String` – OAuth client id for the mobile app
*   `redirectScheme: String` – URL scheme for OIDC callbacks (e.g., `ctrip`)
*   `locale: String` – BCP‑47, e.g., `"zh-CN"`
*   `remoteConfigURL: URL?` – optional HSBC remote config endpoint
*   `featureFlags: [String: Bool]` – local toggle overrides (primarily for dev)
*   `telemetryOptIn: Bool` – partner opt‑in for telemetry

### 5.2 Journey Start Parameters

**Type:** `JourneyStartParams`

*   `journeyId: String` – e.g., `"account_opening"`, `"money_transfer"`
*   `contextToken: String` – server-issued context token (prefill/transfer)
*   `resumeToken: String?` – token from previous partial session
*   `productCode: String?` – optional product identifier
*   `campaignId: String?` – marketing campaign tracking
*   `trackingContext: [String:String]` – additional non‑PII correlation fields
*   `capabilitiesOverride: [String:Bool]?` – optional override of available features

### 5.3 Journey Result

**Type:** `JourneyResult`

*   `.completed(payload: [String: Any])`
*   `.pending(payload: [String: Any])`
*   `.cancelled`
*   `.failed(code: String, message: String, recoverable: Bool)`

Payload is journey-specific (e.g., `{applicationId}` for AO, `{transferId, status}` for MT).

### 5.4 Public API

**Enum:** `HSBCPartnerSdk`

*   `initialize(config: HSBCInitConfig)`
*   `setEventListener((name: String, attributes: [String: Any]) -> Void)`
*   `startJourney(from presenter: UIViewController, params: JourneyStartParams, completion: @escaping (JourneyResult) -> Void)`

The SDK maintains no static global state beyond configuration, event listener, and session identifiers.

* * *

6\. Internal Components
-----------------------

### 6.1 EventBus & Telemetry

**EventBus**: mediator for internal events and external partner listener.

*   `JourneyEvent { name, attributes, ts, correlationId }`
*   API:
    *   `emit(_ event: JourneyEvent)`
    *   Helpers: `journeyBegin`, `stepEnter`, `apiCall`, `bridge`, `error`, `result`.

**Telemetry**:

*   Generates **W3C `traceparent`** per journey session.
*   Simple timing wrappers (`withSpan(...)`) for internal metrics.
*   Privacy‑safe only; no PII or account numbers logged.

* * *

### 6.2 Security: KeyStore & JWS

**KeyStore**:

*   Map: `kid -> SecKey`
*   Initial keys are bundled; additional keys fetched via remote config.
*   Used for validating:
    *   JM signatures
    *   Remote configs (if signed)

**JWS Helper**:

*   Supports **ES256** (P‑256 + SHA‑256).
*   Operations:
    *   `sign(payload, kid) -> compact JWS`
    *   `verify(compactJWS, key) -> payload`
*   Used for:
    *   JM detached signatures (bank signs)
    *   Bridge responses (SDK signs)

* * *

### 6.3 Manifest Management

**Models**:

*   `Manifest` – top‑level JM object.
*   `Step` – individual step definition.
*   `Binding` – event → API call binding.
*   `Transition` – event‑driven state change.

**Loader (`ManifestLoader`)** responsibilities:

1.  Resolve manifest URL from `remoteConfigURL` and `journeyId`.
2.  Fetch JM JSON (HTTP GET; dev may allow local `file://`).
3.  Validate:
    *   Mandatory fields present.
    *   `minSdk` ≤ SDK version.
    *   `security.allowedOrigins` non‑empty.
4.  Verify JWS `signature` using `KeyStore`.
5.  Optionally validate against a JM JSON Schema (CI enforces strict schema).

**Manifest Version**: `1.1.0`

*   Adds: `bridgeAllow`, `resumePolicy`, `idempotencyKey`.

* * *

### 6.4 OpenAPI Resolution & ApiClient

**OpenAPIResolver**:

*   Loads OpenAPI 3 JSON.
*   Builds map: `operationId -> (method, path)`.
*   Basic validation: all operationIds referenced by JM exist.

**ApiClient** responsibilities:

*   Build `URLRequest` from:
    *   Base URL (from OAS `servers` list).
    *   `operationId` resolution.
    *   Body (JSON) from arguments.
    *   Headers (auth, traceparent, idempotency).
*   Support:
    *   TLS, ATS compliance.
    *   Optional certificate pinning (native only).
    *   Retries: 408, 429, and 5xx with exponential backoff and jitter.
    *   Respect `Retry-After` header where present.
*   Returns:
    *   `status`, headers, body (raw Data).
*   Normalizes errors to `SdkErrorCode` (see §10).

* * *

### 6.5 SessionManager & Resume

**SessionManager**:

*   Tracks:
    *   `correlationId` – unique per journey instance.
    *   `stepPointer` – current step id.
    *   `idempotencyKey` – per “server” step (e.g., `CreateTransfer`).
*   Snapshots:
    *   Snapshot structure: `{journeyId, stepPointer, idempotencyKey, ts}`.
    *   Stored in Keychain with `ThisDeviceOnly`; no PII.
    *   `resumeToken` is an opaque reference (e.g. random key in Keychain).
*   Usage:
    *   `saveSnapshot` at step enter and after successful server calls.
    *   `loadSnapshot(resumeToken)` to fast‑forward on re‑entry.

* * *

### 6.6 Container (HybridContainer & WebPolicies)

**HybridContainer**:

*   Auth:
    *   `signInIfNeeded(authURL, redirectScheme)` using `ASWebAuthenticationSession`.
    *   Ephemeral sessions (no long‑lived cookies), where possible.
*   Web:
    *   `presentStepWebView(on: UIViewController, url: URL, allowedOrigins: [URL], bridge: Bridge) -> UIViewController`
    *   `WKWebViewConfiguration`:
        *   JavaScript enabled.
        *   No navigation gestures.
        *   Pop‑ups blocked.
    *   Navigation delegate:
        *   Enforces origin checks via `WebPolicies`.
        *   On disallowed navigation → error and termination.

**WebPolicies**:

*   `isAllowed(origin: URL, allowed: [URL]) -> Bool`:
    *   Scheme must be `https` (dev mode may allow `file://` behind flag).
    *   Host must match one of `allowedOrigins`.
*   ATS & WKAppBoundDomains configured at app-level for HSBC domains.

* * *

### 6.7 Bridge v1.1 (JS ↔ Native)

**Goals**:

*   Guarantee only HSBC pages call native APIs.
*   Ensure payload integrity and basic anti‑replay.
*   Enforce per‑step method allow‑list.

**Bridge States**:

*   `notReady` – handshake not completed.
*   `ready` – handshake OK; origin verified.

**Handshake**:

1.  Page initial JS sends:
    ```json
    { "kind":"event",
      "name":"bridge_hello",
      "payload":{ "origin": "<page origin>", "pageNonce": "<random>" } }
    ```
2.  Bridge:
    *   Validates origin against `Manifest.security.allowedOrigins`.
    *   Collects attestation (DeviceCheck/App Attest → base64 string).
    *   Responds with `bridge_ready` event:
    ```json
    {
      "kind":"event",
      "name":"bridge_ready",
      "payload":{
        "sdkCapabilities":["biometric","nfc","filePicker"], 
        "sessionProofJws":"<JWS>"
      },
      "meta":{
        "ts":"…",
        "nonce":"…",
        "bridgeVersion":"1.1",
        "sdkVersion":"1.0.0",
        "traceparent":"00-…"
      },
      "sig":"<ES256 over name+payload+meta>"
    }
    ```

**Requests & Responses**:

*   Page → SDK: `{kind:'request', id, name, payload}`
*   SDK: checks:
    *   `state == ready`
    *   `name` ∈ `bridgeAllow` for current step
*   If allowed:
    *   Dispatches to:
        *   Plugin (e.g. biometrics)
        *   Or to internal behavior (e.g. future camera plugin).
    *   Sends `resolve` envelope:
        ```json
        {
          "kind":"resolve",
          "id":"<id>",
          "name":"performBiometric",
          "payload":{"biometric":"PASS"},
          "meta":{"ts":"…","nonce":"…","bridgeVersion":"1.1","sdkVersion":"1.0.0","traceparent":"00-…"},
          "sig":"<JWS>"
        }
        ```
*   If disallowed:
    *   Respond with error payload `code:"BRIDGE_FORBIDDEN"`.

All envelopes SDK→Page are **JWS signed** to support integrity checks by HSBC pages.

* * *

### 6.8 Plugins

**Interface**:

*   `JourneyPlugin`:
    *   `name: String`
    *   `canHandle(method: String) -> Bool`
    *   `handle(method: String, params: [String:Any]) async throws -> [String:Any]`

**BiometricAuthPlugin**:

*   Implements `performBiometric` using `LocalAuthentication`.
*   Returns `{"biometric":"PASS"}` or throws with code `SCA_REQUIRED`, `USER_CANCELLED`, etc.

Bridge uses a global `PluginRegistry` to locate the plugin for a requested method.

* * *

7\. State Machine Design
------------------------

**StateMachine** interprets the `steps` defined in JM:

*   Holds:
    *   `currentStepId`
    *   `steps` map
*   `handleEvent(name, payload)`:
    1.  Find `Transition` in `steps[currentStepId].on[name]`.
    2.  Evaluate `guardExpr` (if any) using:
        *   Simple expression evaluator (`==, !=, >, <, &&, ||`).
        *   Access to a limited context (e.g. `session.quoteId`, `now`).
    3.  If guard passes:
        *   Execute `effect` (e.g., emit an event to Bridge).
        *   Move to `to` step:
            *   Emit `STEP_EXIT` and `STEP_ENTER` events.
            *   Start `timeout` if `timeoutMs` is configured.

**Bindings**:

*   When the page emits `Binding.onEvent` (e.g., `"validateRecipient"`):
    *   StateMachine triggers `ApiClient.call` for `Binding.call.operationId`.
    *   `argsFrom` indicates the path within `event.payload` or session context.
    *   On success/failure, events `onSuccessEmit` / `onErrorEmit` are emitted back to the page via Bridge and may trigger transitions.

* * *

8\. Journey Manifest v1.1 Specification (Summary)
-------------------------------------------------

### 8.1 Top-level

*   `manifestVersion: String`
*   `minSdk: String`
*   `journeyId: String`
*   `oapiBundle: URL`
*   `locales: [String]?`
*   `startStep: String`
*   `headers: [String:String]` – header templates; `"$contextToken"` etc. are replaced at runtime.
*   `security`:
    *   `allowedOrigins: [URL]`
    *   `pinning: Bool`
    *   `attestation: [String:Bool]?` (e.g., `{ "ios": true }`)
    *   `requireHandshake: Bool`
*   `resumePolicy`: optional (e.g. `{"snapshotOn":["stepEnter","serverSuccess"]}`)
*   `steps: [String: Step]`
*   `signature: String` – detached JWS over manifest body (without `signature`).

### 8.2 Step Types

*   `type: "web" | "native" | "server" | "terminal"`
*   `web`:
    *   `url: URL`
    *   `bridgeAllow: [String]` – permitted bridge request names
    *   `bindings: [Binding]?`
    *   `on: [eventName: Transition]`
*   `native`:
    *   `plugin: String`
    *   `params: [String:Any]`
    *   `on` transitions (e.g. `success`, `failure`)
*   `server`:
    *   `call.operationId: String`
    *   `idempotencyKey: String` – expression referencing session field (e.g., `$session.correlationId`)
    *   `on.success`, `on.error`
*   `terminal`:
    *   `result: [String:Any]` – used to compute `JourneyResult`

* * *

9\. Money Transfer Journey Example
----------------------------------

**JourneyId:** `"money_transfer"`

### 9.1 High-Level Flow

1.  **mt\_intro** – Intro page, user continues or cancels.
2.  **mt\_recipient** – Capture recipient details, validate via `ValidateRecipient`.
3.  **mt\_amount** – Enter amount and calculate fees (`CalculateFee`).
4.  **mt\_review** – Display summary and get FX quote (`GetQuote`).
5.  **mt\_biometric** – Native biometric confirmation.
6.  **mt\_submit** – Server call `CreateTransfer` (idempotent).
7.  **end\_complete** / **end\_cancel** / **end\_error** – Terminal states.

### 9.2 Example Use of Manifest Concepts

*   `bridgeAllow`:
    *   `mt_recipient`: `["validateRecipient"]`
    *   `mt_amount`: `["calculateFees"]`
    *   `mt_review`: `["requestQuote"]`
*   `bindings`:
    *   `mt_recipient.validateRecipient` → `ValidateRecipient`
    *   `mt_amount.calculateFees` → `CalculateFee`
    *   `mt_review.requestQuote` → `GetQuote`
*   Guarded transition in `mt_review`:
    *   On `confirm`:
        *   Guard: `session.quoteId != null && now < session.quoteExpiry`
        *   If true: `→ mt_biometric`
        *   Else: `→ mt_amount` with effect `emit("QUOTE_EXPIRED")`
*   `mt_submit`:
    *   `idempotencyKey: "$session.correlationId"` to ensure safe retries.

* * *

10\. Error Model & Mapping
--------------------------

**Internal enum:** `SdkErrorCode` (string)

Key codes:

*   `AUTH_EXPIRED` – OIDC tokens invalid or expired
*   `PINNING_FAIL` – TLS certificate pinning mismatch
*   `ORIGIN_BLOCKED` – page origin not allowed by manifest
*   `NET_TIMEOUT` – network timeout or unreachable
*   `RATE_LIMITED` – HTTP 429
*   `VALIDATION_FAIL` – HTTP 4xx with validation details
*   `IDEMPOTENT_REPLAY` – repeated operation with same idempotencyKey
*   `SCA_REQUIRED` – strong customer authentication failed
*   `COMPLIANCE_HOLD` – screening/AML hold
*   `MORE_INFO` – additional documents/info required
*   `UNKNOWN` – catch‑all

**Mapping to JourneyResult**:

*   Fatal (e.g., `PINNING_FAIL`, `ORIGIN_BLOCKED`) → `.failed(recoverable: false)`
*   Recoverable (e.g., `NET_TIMEOUT`, `RATE_LIMITED`) → `.failed(recoverable: true)` plus recommended UI messaging.
*   JM may override behavior via `errorMap` (not fully detailed here, but supported via transitions).

* * *

11\. Packaging & Distribution (iOS)
-----------------------------------

*   **Swift Package**:
    *   Name: `HSBCPartnerSDK`
    *   Targets: `HSBCPartnerSDKCore`, `HSBCPlugins`, tests.
    *   Resources: `ao_test.html`, `PrivacyInfo.xcprivacy`.
*   **XCFramework**:
    *   Built as a binary framework for distribution.
*   **Distribution options**:
    *   SwiftPM (source/Binary target).
    *   CocoaPods (podspec referencing XCFramework).
    *   Carthage (binary JSON descriptor).
*   **Versioning**:
    *   SemVer (`MAJOR.MINOR.PATCH`).
    *   `1.x` API stable; breaking changes require `2.0`.

* * *

12\. Extensibility: Adding New Journeys
---------------------------------------

To add a new journey (e.g., **Card Application**):

1.  HSBC defines a new **OpenAPI** bundle for required APIs.
2.  HSBC creates a **JM v1.1** manifest describing steps, transitions, bindings.
3.  HSBC develops hosted pages using the standard bridge (`bridge_hello`, requests, events).
4.  HSBC signs the manifest and publishes it plus OAB in remote config.
5.  Partner app simply calls:
    ```swift
    HSBCPartnerSdk.startJourney(
        from: presenter,
        params: JourneyStartParams(
            journeyId: "card_application",
            contextToken: "...",
            productCode: "...",
            campaignId: nil,
            trackingContext: [...]
        )
    )
    ```

No SDK update is required, assuming existing capabilities (bridge methods, plugins) suffice.

* * *

13\. Testing Strategy
---------------------

### 13.1 Unit Tests

*   **ManifestTests**:
    *   Decode sample JM; verify `minSdk`, `allowedOrigins` present.
*   **BridgeHandshakeTests**:
    *   Simulate `bridge_hello` from allowed and disallowed origins.
*   **StateMachineTests**:
    *   Test basic sequences, guard expressions, timeouts.
*   **ApiClientTests**:
    *   Verify operationId resolution; retry behavior on 429/500.

### 13.2 UI Tests

*   Demo App UI test:
    *   Initialize SDK.
    *   Start journey.
    *   Interact with WKWebView (local demo pages or staging AO/MT).
    *   Verify `JourneyResult` text updates.

### 13.3 Integration & Security Tests

*   **Pinning** tests (invalid certificate).
*   **Attestation** tests (simulated failure).
*   **Error path** coverage (429, 500, blocked origins, handshake failure).
*   **Resume** tests (kill app, relaunch with `resumeToken`).

* * *

14\. Operations & Rollout
-------------------------

*   **Remote config**:
    *   Controls:
        *   JM/OAS URLs.
        *   Journey enablement.
        *   Feature flags (plugins, experimental flows).
*   **Canary deployments**:
    *   Roll out new manifests to subset of traffic.
*   **Kill switches**:
    *   Disable a journey or plugin quickly in case of incident.
*   **SLOs**:
    *   TTFP (time to first paint) < 1200ms on 4G.
    *   p95 server call latency < 800ms.
    *   Error rate thresholds for auto rollback.

* * *

15\. Security Backlog (Post‑MVP Enhancements)
---------------------------------------------

The following hardening items should be tracked and delivered over time:

*   Full certificate pinning with dual pins per region.
*   JWE for sensitive plugin responses.
*   App Attest integration with HMAC binding to tokens.
*   Formal JM JSON Schema enforcement at CI & runtime.
*   More expressive guard expression DSL with static analysis.
*   OpenTelemetry export support for HSBC observability stack.

* * *

16\. Conclusion
---------------

This low‑level design defines a **journey‑agnostic, hardened SDK** that:

*   Treats the SDK as a **generic runtime and container**, not a business‑specific client.
*   Pushes journey complexity into **signed manifests** and **OpenAPI specs** owned by HSBC.
*   Enforces strong **security and compliance controls**, while allowing partners to embed rich HSBC journeys with a **single, stable API**.

This structure is sufficient to implement:

*   The current **Account Opening** journey.
*   The new **Money Transfer** journey.
*   Future journeys (e.g. card application, address updates) with minimal incremental effort.

If you’d like, I can next:

*   Convert this LLD into a **shorter architecture overview** for management, or
*   Extract specific sections into your internal template (e.g., “Security”, “Data Model”, “Sequence Diagrams”).

## Prompt:
please covert LLD , and also extract all section to template.

## Response:
Thought for 1m 56s

Got it. I’ll give you **two things** in one answer:

1.  A **converted High-Level Design (HLD)** version of the LLD – shorter, executive‑friendly, but still precise.
2.  A **clean reusable template** (headings + prompts) you can use for future SDKs/journeys.

I’ll keep both well‑structured and professional.

* * *

A. High‑Level Design (HLD) – HSBC Journey‑Agnostic iOS Partner SDK
==================================================================

1\. Overview
------------

HSBC is providing a **journey‑agnostic iOS SDK** that allows partners (e.g. Ctrip) to embed HSBC’s digital journeys directly inside their apps, such as:

*   Account Opening (AO)
*   Money Transfer (MT)
*   Future: Card Applications, Address Changes, etc.

The SDK is **generic**. Specific journeys are controlled by **server‑side configuration**, not by SDK code changes. This is achieved through:

*   **Journey Manifests** (JM): signed JSON files that describe the steps, transitions, and rules of a journey.
*   **OpenAPI specifications**: defining backend operations (`operationId`) and schemas.
*   **HSBC‑hosted web pages**: responsible for user interface and business validation, running inside a secure container with a JS bridge.

The partner integrates once with the SDK; HSBC can roll out new journeys and changes centrally.

* * *

2\. Objectives
--------------

### 2.1 Business Objectives

*   Enable partners to **offer HSBC products** (accounts, transfers, etc.) inside their own apps.
*   **Shorten time to market**: deliver new journeys without requiring partner app updates.
*   Maintain **HSBC brand, security and compliance** across all partner channels.

### 2.2 Technical Objectives

*   Single public API to start **any journey**.
*   Hosted HSBC pages for business logic and compliance text.
*   Journey configuration via **signed manifests** and **OpenAPI**.
*   Strong security: OIDC + PKCE, TLS + pinning, attestation, JWS/JWE, kill‑switches.
*   Privacy‑safe telemetry and end‑to‑end traceability.

* * *

3\. Architecture Summary
------------------------

### 3.1 Key Components

**On Device (iOS SDK)**

*   **HSBCPartnerSdk (Public API)**  
    Entry point for initialization and starting journeys.
*   **Runtime Engine**
    *   Loads Journey Manifest and OpenAPI.
    *   Runs a state machine over steps & transitions.
    *   Calls HSBC APIs via a generic ApiClient.
    *   Handles session, resume tokens, and result mapping.
*   **Container**
    *   OIDC login via `ASWebAuthenticationSession`.
    *   Presents HSBC pages via `WKWebView` with strong domain restrictions.
    *   Hosts the **JS bridge** between pages and the SDK.
*   **Bridge**
    *   Validates page origin.
    *   Performs handshake (`bridge_hello` / `bridge_ready`).
    *   Enforces per‑step allowed methods.
    *   Signs all SDK → page messages (JWS).
*   **Security & Plugins**
    *   KeyStore + JWS helper.
    *   Device attestation collector.
    *   Plugins such as biometric authentication (FaceID/TouchID).

**On HSBC Side**

*   **Journey Manifest service** – stores & signs JMs per journey and region.
*   **OpenAPI specifications** – published for AO/MT/backend services.
*   **Hosted AO/MT web journeys** – pages using the standard bridge.
*   **Partner API Gateway** – OAuth2, KYC, MT operations, etc.
*   **Remote Config** – toggles features, endpoints, and kill‑switches.

* * *

4\. Functional Design
---------------------

### 4.1 Public SDK API

*   `initialize(config: HSBCInitConfig)`  
    Sets environment, partner identity, locale, remote config URL, feature flags.
*   `setEventListener(_:)`  
    Allows partner to receive privacy‑safe events (e.g., `JOURNEY_BEGIN`, `STEP_ENTER`, `RESULT`).
*   `startJourney(from: UIViewController, params: JourneyStartParams, completion: @escaping (JourneyResult) -> Void)`  
    Generic call to start any journey (`account_opening`, `money_transfer`, etc.), given a `contextToken`.

### 4.2 Lifecycle at Runtime

1.  Partner calls `startJourney`.
2.  SDK:
    *   Fetches/validates the manifest for `journeyId`.
    *   Loads OpenAPI bundle and prepares the ApiClient.
    *   Runs OIDC login, if required.
    *   Opens the first HSBC journey page in a WKWebView.
3.  Page and SDK exchange events via the bridge:
    *   Page events drive the **state machine**.
    *   Bindings trigger backend calls by `operationId` from OpenAPI.
    *   Plugins handle native capabilities (e.g., biometrics).
4.  When a terminal step is reached, the SDK:
    *   Closes the container.
    *   Returns a `JourneyResult` (completed/pending/cancelled/failed).
    *   Emits final events for telemetry and partner analytics.

* * *

5\. Journey Configuration
-------------------------

### 5.1 Journey Manifest

*   Controlled by HSBC and signed using JWS.
*   Specifies:
    *   Steps (`web`, `native`, `server`, `terminal`).
    *   Allowed JS bridge methods per step.
    *   Bindings from page events to backend APIs (`operationId`).
    *   Step transitions and guard conditions.
    *   Resume policy (when to save snapshots).
    *   Security policy (allowed origins, pinning, attestation).

### 5.2 OpenAPI Integration

*   Each manifest references an OpenAPI bundle by URL.
*   All backend operations referenced in the manifest are defined by `operationId`.
*   The SDK uses OpenAPI to build HTTP requests and map responses.

* * *

6\. Security & Compliance
-------------------------

Key protections:

*   **Authentication**: OIDC Authorization Code + PKCE via system browser.
*   **Transport**: TLS 1.2+, ATS, certificate pinning (native), AppBound domains for WebView.
*   **Bridge**: origin validation; signed responses; per‑step allow‑list.
*   **Data protection**: PII minimized on device; snapshots are PII‑free and encrypted.
*   **Device integrity**: DeviceCheck/App Attest; root/jailbreak detection; verdicts sent to HSBC.
*   **Governance**: manifests and config are JWS‑signed; remote kill‑switches.

* * *

7\. Example Journeys
--------------------

### 7.1 Account Opening (AO)

*   Steps: eligibility, personal details, KYC documents, review, submit.
*   Hosted pages: AO‑specific.
*   APIs: create application, upload documents, schedule appointment, status.

### 7.2 Money Transfer (MT)

*   Steps:
    *   Intro → recipient → amount & fees → review & quote → biometric → submit.
*   APIs:
    *   `ValidateRecipient`, `CalculateFee`, `GetQuote`, `CreateTransfer`.
*   Features:
    *   FX quote with expiry.
    *   Biometric confirmation.
    *   Idempotent submission.

* * *

8\. Non‑Functional & Operations
-------------------------------

*   **Performance**: fast first paint, resilient retries, no blocking UI.
*   **Versioning**: SemVer; stable public API for 1.x.
*   **Observability**: `traceparent` across app/web/API; aggregated metrics.
*   **Deployment**:
    *   Swift Package (source or binary).
    *   XCFramework, CocoaPods, Carthage.
    *   CI builds, tests, signs, and publishes artifacts.

* * *

B. Design Document Template (Extracted Sections)
================================================

Below is a **generic template** you can reuse for future designs (for this SDK or other systems). You can copy this into Word, Confluence, or Markdown and fill in.

* * *

1\. Document Information
------------------------

*   **Title:**
*   **Version:**
*   **Author(s):**
*   **Date:**
*   **Status:** Draft / Review / Approved
*   **Stakeholders:** (Architecture, Security, Product, Partner, etc.)

* * *

2\. Introduction
----------------

### 2.1 Purpose

*   What is this system / SDK for?
*   What problem does it solve?

### 2.2 Scope

*   What is included in this design?
*   What is explicitly out of scope?

### 2.3 Audience

*   Who should read this document? (e.g., iOS devs, architects, security team, partner tech teams)

* * *

3\. Background & Objectives
---------------------------

### 3.1 Business Context

*   Business drivers (e.g., partner distribution, digital onboarding).
*   Existing solutions or limitations.

### 3.2 Objectives

*   Business objectives (e.g., conversion, time to market).
*   Technical objectives (e.g., journey-agnostic, high security, low maintenance).

### 3.3 Assumptions & Constraints

*   Platform constraints (iOS versions, Xcode versions).
*   Regulatory constraints (data locality, privacy).
*   Partner capabilities (e.g., supports deep links, OIDC).

* * *

4\. High‑Level Architecture
---------------------------

### 4.1 Architecture Overview

*   Diagram of major components (SDK, app, backend, web pages).
*   Short explanation of how they interact.

### 4.2 Key Design Concepts

*   Journey‑agnostic runtime.
*   Hosted pages + JS bridge.
*   Signed manifests + OpenAPI integration.

* * *

5\. Functional Requirements
---------------------------

### 5.1 Core Use Cases

*   List of journeys / flows the SDK must support (e.g., Account Opening, Money Transfer).

### 5.2 User Journeys

*   Short narrative of each journey from end‑user’s perspective.

### 5.3 Partner Integration Requirements

*   How the partner app integrates: initialization, journey start, handling results.

* * *

6\. Public API Design
---------------------

### 6.1 Configuration Types

*   Describe `InitConfig` (fields, meaning, defaults).
*   Describe `JourneyStartParams`.

### 6.2 API Methods

*   Initialization API(s).
*   Journey start method(s).
*   Event listener / callback registration.

### 6.3 Result Handling

*   Describe `Result` types and their semantics.
*   Error codes surfaced to partners.

* * *

7\. Internal Architecture
-------------------------

### 7.1 Components

For each internal component, define:

*   Purpose.
*   Responsibilities.
*   Key operations.
*   Interactions with other components.

Typical components:

*   Runtime Engine
*   Manifest Loader
*   State Machine
*   ApiClient & OpenAPI Resolver
*   Session Manager (snapshots/resume)
*   Telemetry & EventBus
*   Container (Auth + WebView)
*   Bridge
*   Plugin Framework

### 7.2 Class / Module Overview

*   Optional class diagram or list of main Swift types.
*   Mapping to source folders (e.g., `Runtime/`, `Manifest/`, `Network/`).

* * *

8\. Journey Configuration Model
-------------------------------

### 8.1 Journey Manifest (JM)

*   Versioning (e.g., v1.1).
*   Structure:
    *   Top‑level fields (id, minSdk, oapiBundle, security, startStep, etc.).
    *   Step types (`web`, `native`, `server`, `terminal`).
    *   Transitions and guards.
    *   Resume policy.
    *   Signature.

### 8.2 Step Types

*   Web step (URL, bindings, allowed bridge methods).
*   Native step (plugin, parameters).
*   Server step (operationId, idempotency key).
*   Terminal step (result payload).

### 8.3 Bindings & Transitions

*   How events from pages map to API calls and transitions.
*   Example binding (`validateRecipient`, `CalculateFee`, etc.)

* * *

9\. Bridge Protocol
-------------------

### 9.1 Overview

*   Purpose of the JS ↔ Native bridge.
*   Which components use it.

### 9.2 Message Types

*   `event`, `request`, `resolve`, `hello/ready`, `error`.
*   Required fields for each (id, name, payload, meta, sig).

### 9.3 Handshake

*   Initiation sequence.
*   Origin checks.
*   Capability negotiation (what features are available).

### 9.4 Security

*   Signing (JWS).
*   Anti‑replay mechanisms (nonce, timestamps).
*   Per‑step allowed method lists.

* * *

10\. API Integration (OpenAPI)
------------------------------

### 10.1 OpenAPI Usage

*   Where the OpenAPI bundle is hosted.
*   How the SDK loads and caches it.

### 10.2 OperationId Mapping

*   How operationIds used in JM resolve to actual HTTP calls.

### 10.3 Idempotency & Retry Policy

*   Which operations are idempotent.
*   Headers used for idempotency (e.g., `X-Idempotency-Key`).
*   Retry policy (status codes, backoff).

* * *

11\. Security & Compliance
--------------------------

### 11.1 Authentication & Authorization

*   Protocols (e.g., OIDC + PKCE).
*   Token types and lifetime.

### 11.2 Data Protection

*   Data in transit (TLS, pinning, ATS).
*   Data at rest on device (Keychain usage, snapshots).
*   PII handling strategy.

### 11.3 Device Integrity

*   Jailbreak/root detection.
*   Attestation (e.g., DeviceCheck/App Attest).
*   How the verdict is used in risk decisions.

### 11.4 Governance & Change Management

*   Manifest signing and distribution.
*   Remote configuration and kill switches.
*   Audit and logging (what is logged, where, and how long).

* * *

12\. Error Handling & Observability
-----------------------------------

### 12.1 Error Model

*   List of canonical error codes.
*   Mapping from HTTP/network/bridge failures to codes.

### 12.2 Surfacing Errors

*   How errors are returned via `Result`.
*   How errors are emitted via events.

### 12.3 Logging & Metrics

*   What events are instrumented.
*   Key metrics (step timing, completion rates, error rates).
*   Trace propagation (e.g., `traceparent` header).

* * *

13\. Non‑Functional Requirements
--------------------------------

### 13.1 Performance

*   Target times (TTFP, API latency).
*   Constraints (no blocking on main thread).

### 13.2 Reliability & Resilience

*   Retry strategies.
*   Offline/weak network behavior.
*   Expected availability.

### 13.3 Compatibility & Versioning

*   Minimum supported OS versions.
*   SemVer strategy for the SDK.

* * *

14\. Build, Packaging & Distribution
------------------------------------

### 14.1 Build Process

*   Tools (Xcode version, SwiftPM).
*   CI pipeline overview.

### 14.2 Packaging

*   SwiftPM (source and/or binary).
*   XCFramework packaging.
*   Podspec / Carthage support (if required).

### 14.3 Release Management

*   Tagging convention.
*   Release artifacts (XCFramework, checksums, docs).
*   Backward compatibility policy.

* * *

15\. Testing & Validation
-------------------------

### 15.1 Unit Tests

*   Coverage areas (manifest, bridge, state machine, ApiClient, etc.).

### 15.2 Integration Tests

*   End‑to‑end flows (AO, MT).
*   Mock services and pages.

### 15.3 UI Tests

*   Key UI flows in demo or reference app.

### 15.4 Security Testing

*   Pen testing.
*   Static/dynamic analysis.
*   Threat modeling (optional section).

* * *

16\. Risks & Mitigations
------------------------

*   Identify key risks (security, partner misuse, performance, UX).
*   Proposed mitigations per risk.

* * *

17\. Open Questions & Future Enhancements
-----------------------------------------

*   Design questions not yet resolved.
*   Planned enhancements (e.g., offline support, more plugins, richer DSL).

* * *

If you’d like, I can also:

*   Map this template into a **Word/PowerPoint outline** (titles + bullets), or
*   Help you fill in the template for a **second journey** (e.g., “Credit Card Application”) using the same structure.

## Prompt:
yes please do both of them

## Response:
Thought for 1m 46s

I’ll split this into **two parts**, as you asked:

1.  A **Word + PowerPoint–friendly outline** of the design (titles + bullets).
2.  A **filled-in template for a second journey: Credit Card Application**, using the same structure but more concise.

You can copy‑paste Part 1 into Word/PowerPoint, and Part 2 into your internal wiki or design library.

* * *

Part 1 – Word / PowerPoint Outline
==================================

1\. Word‑Style Outline (for a design document)
----------------------------------------------

You can paste this into Word and then flesh out each bullet.

* * *

### 1\. Document Information

*   Title: HSBC Journey‑Agnostic iOS Partner SDK – Design
*   Version: 1.0
*   Author(s):
*   Date:
*   Status: Draft / In review / Approved
*   Stakeholders:
    *   Architecture
    *   Security
    *   Mobile Engineering (iOS/Android)
    *   Digital Product
    *   Partner Engineering

* * *

### 2\. Introduction

**2.1 Purpose**

*   Describe the purpose of the SDK:
    *   Provide a journey‑agnostic iOS SDK enabling partners to embed HSBC digital journeys (Account Opening, Money Transfer, etc.) into their apps.
    *   Ensure HSBC retains control of security, compliance, and journey logic.

**2.2 Scope**

*   In scope:
    *   iOS SDK design and implementation.
    *   Journey configuration via manifests and OpenAPI.
    *   Example journeys: Account Opening (AO) and Money Transfer (MT).
*   Out of scope:
    *   Android/Web SDK details.
    *   Backend implementations of AO/MT APIs.

**2.3 Audience**

*   iOS developers
*   Architects
*   Security and Compliance teams
*   Partner technical teams / integration engineers

* * *

### 3\. Background & Objectives

**3.1 Business Context**

*   Partner channels increasingly used for acquisition.
*   Existing onboarding flows are siloed and slow to change.
*   Need centralized, governed journeys re‑used across partners.

**3.2 Objectives**

*   Business:
    *   Increase acquisition via partners.
    *   Improve conversion and reduce friction.
    *   Centralize journey control at HSBC.
*   Technical:
    *   Single SDK for multiple journeys.
    *   Server‑driven configuration via manifest + OpenAPI.
    *   Strong, bank‑grade security and compliance.

**3.3 Assumptions & Constraints**

*   Minimum iOS version: 14+
*   HSBC provides:
    *   Journey manifests (JM)
    *   OpenAPI specs (OAS)
    *   Hosted web journeys
*   Partner app supports:
    *   Deep links / custom URL schemes
    *   TLS 1.2+
    *   OIDC redirect handling

* * *

### 4\. High‑Level Architecture

**4.1 Architecture Overview**

*   Components:
    *   Partner App
    *   iOS SDK (HSBCPartnerSdk)
    *   HSBC Hosted Pages (AO/MT)
    *   HSBC Backend APIs (AO, MT, KYC, etc.)
    *   Manifest & Config Services
*   Data Flow:
    *   Partner → SDK → HSBC hosted pages / APIs.

**4.2 Key Design Concepts**

*   Journey‑agnostic SDK runtime driven by manifests.
*   Hosted HSBC pages in an in‑app container.
*   JS bridge for native ↔ web communication.
*   OpenAPI‑based backend integration.

* * *

### 5\. Functional Requirements

**5.1 Core Use Cases**

*   Account Opening journey.
*   Money Transfer journey.
*   Future journeys (e.g., Credit Card Application).

**5.2 User Journey Summary**

*   End‑user taps “Open HSBC Account” or “Send Money with HSBC”.
*   Journey runs inside partner app via SDK container.
*   Result (success/pending/error) returned to partner app.

**5.3 Partner Integration Requirements**

*   Initialize SDK once per app launch.
*   Call `startJourney()` with journeyId and contextToken.
*   Handle `JourneyResult` and events for analytics.

* * *

### 6\. Public API Design

**6.1 Configuration Types**

*   HSBCInitConfig:
    *   environment
    *   partnerId
    *   clientId
    *   redirectScheme
    *   locale
    *   remoteConfigURL
    *   featureFlags
    *   telemetryOptIn
*   JourneyStartParams:
    *   journeyId
    *   contextToken
    *   resumeToken (optional)
    *   productCode
    *   campaignId
    *   trackingContext
    *   capabilitiesOverride

**6.2 API Methods**

*   `initialize(config:)`
*   `setEventListener(_:)`
*   `startJourney(from:params:completion:)`

**6.3 Result Handling**

*   JourneyResult:
    *   completed(payload)
    *   pending(payload)
    *   cancelled
    *   failed(code, message, recoverable)

* * *

### 7\. Internal Architecture

**7.1 Components**

*   Runtime Engine
*   Manifest Loader
*   State Machine
*   ApiClient & OpenAPIResolver
*   SessionManager
*   Telemetry & EventBus
*   Container (Hybrid: ASWebAuth + WKWebView)
*   Bridge
*   Plugins (Biometric, etc.)

**7.2 Interactions**

*   Runtime loads manifest & OAS.
*   StateMachine drives step flow.
*   ApiClient calls HSBC APIs by operationId.
*   Container presents pages and hosts the bridge.
*   Bridge consumes page events, calls plugins/ApiClient, and responds.

* * *

### 8\. Journey Configuration Model

**8.1 Manifest (JM)**

*   Manifest version.
*   Min SDK version.
*   Journey ID.
*   OpenAPI bundle URL.
*   Start step.
*   Security (allowedOrigins, pinning, attestation).
*   Headers template.
*   Steps map.

**8.2 Step Types**

*   Web step:
    *   url
    *   bridgeAllow
    *   bindings
    *   transitions
*   Native step:
    *   plugin
    *   params
    *   transitions
*   Server step:
    *   operationId
    *   idempotencyKey
    *   transitions
*   Terminal step:
    *   result payload

**8.3 Bindings & Transitions**

*   Bindings:
    *   onEvent → API operationId.
*   Transitions:
    *   eventName → nextStep (with optional guard expression and effects).

* * *

### 9\. Bridge Protocol

**9.1 Purpose**

*   Secure communication between HSBC pages and native SDK.

**9.2 Message Types**

*   `event`
*   `request`
*   `resolve`
*   `hello/ready`
*   `error`

**9.3 Handshake**

*   Page sends `bridge_hello` with origin and nonce.
*   Bridge validates origin and responds with `bridge_ready` and JWS‑signed envelope.

**9.4 Security**

*   Origin allowed list from manifest.
*   Signed responses (JWS).
*   Nonce and timestamp for anti‑replay.
*   Per‑step allowed method list.

* * *

### 10\. API Integration (OpenAPI)

**10.1 OpenAPI Usage**

*   SDK loads OAS once per journey.
*   Validates that all operationIds in manifest exist.

**10.2 OperationId Mapping**

*   operationId → HTTP method + path.
*   Build base URL from OAS servers.

**10.3 Idempotency & Retry Policy**

*   Use `X-Idempotency-Key` header from Journey/session context.
*   Retries on 408/429/5xx with exponential backoff and jitter.
*   Respect `Retry-After` where present.

* * *

### 11\. Security & Compliance

**11.1 Authentication & Authorization**

*   OIDC Authorization Code + PKCE in ASWebAuthenticationSession.

**11.2 Data Protection**

*   TLS with ATS; certificate pinning for native API calls.
*   PII minimized on device.
*   Session snapshots PII‑free and Keychain‑encrypted.

**11.3 Device Integrity**

*   Jailbreak/root checks.
*   DeviceCheck / App Attest verdict included in headers.

**11.4 Governance**

*   JM and remote config signed (JWS).
*   Kill‑switch for journeys/plugins.
*   Centralized audit logging.

* * *

### 12\. Error Handling & Observability

**12.1 Error Model**

*   SdkErrorCode: AUTH\_EXPIRED, PINNING\_FAIL, ORIGIN\_BLOCKED, NET\_TIMEOUT, RATE\_LIMITED, VALIDATION\_FAIL, IDEMPOTENT\_REPLAY, SCA\_REQUIRED, COMPLIANCE\_HOLD, MORE\_INFO, UNKNOWN.

**12.2 Surfacing Errors**

*   JourneyResult.failed(code, message, recoverable).
*   Events via EventBus for diagnostics.

**12.3 Logging & Metrics**

*   Events: journey begin/end, step enter/exit, API calls, errors.
*   Metrics: completion rate, latency, retry rates.
*   `traceparent` for distributed tracing.

* * *

### 13\. Non‑Functional Requirements

**13.1 Performance**

*   TTFP target, p95 API latency.
*   No heavy work on main thread.

**13.2 Reliability**

*   Graceful handling of poor networks.
*   Idempotent submissions.

**13.3 Compatibility**

*   iOS 14+, Xcode 15+.
*   SemVer for SDK.

* * *

### 14\. Build, Packaging & Distribution

**14.1 Build Process**

*   SwiftPM + Xcode.
*   CI pipeline for build, test, lint.

**14.2 Packaging**

*   XCFramework.
*   SwiftPM binary target.
*   Optional CocoaPods / Carthage.

**14.3 Release Management**

*   Tagging scheme: vMAJOR.MINOR.PATCH.
*   Release notes, artifacts, checksums.

* * *

### 15\. Testing & Validation

**15.1 Unit Tests**

*   Manifest loading & validation.
*   Bridge handshake.
*   State machine transitions.
*   ApiClient behavior.

**15.2 Integration Tests**

*   AO journey end‑to‑end.
*   MT journey end‑to‑end via mocks/staging.

**15.3 UI Tests**

*   Demo app with WKWebView interactions.

**15.4 Security Testing**

*   Pen tests, code scanning, threat modeling.

* * *

### 16\. Risks & Mitigations

*   List key risks: security, partner misuse, UX, performance.
*   Document mitigation strategies for each.

* * *

### 17\. Open Questions & Future Enhancements

*   Unresolved design questions.
*   Planned enhancements (e.g., more plugins, offline support).

* * *

2\. PowerPoint Slide Outline
----------------------------

You can map these to slides 1–10.

* * *

**Slide 1 – Title**

*   Title: HSBC Journey‑Agnostic iOS Partner SDK
*   Subtitle: Architecture & Design Overview
*   Presenter, Date

* * *

**Slide 2 – Business Context & Objectives**

*   Partner apps are strategic acquisition channels.
*   Need standardized HSBC journeys across partners.
*   Objectives:
    *   Single SDK for multiple journeys.
    *   Faster rollout of new products.
    *   Centralized governance and compliance.

* * *

**Slide 3 – Concept: Journey‑Agnostic SDK**

*   One SDK, many journeys:
    *   Account Opening
    *   Money Transfer
    *   Card Applications (future)
*   Server‑driven via:
    *   Journey Manifests (JM)
    *   OpenAPI Specs
    *   Hosted HSBC pages

* * *

**Slide 4 – High‑Level Architecture**

*   Partner App ↔ HSBCPartnerSdk (iOS) ↔ HSBC Hosted Pages ↔ HSBC APIs
*   Key components:
    *   Runtime Engine
    *   Container (Auth + Web)
    *   Bridge
    *   Plugins
*   Call‑flow diagram (visual).

* * *

**Slide 5 – Public API & Partner Integration**

*   `initialize(config:)`
*   `startJourney(journeyId, contextToken, …)`
*   `JourneyResult` (completed / pending / cancelled / failed)
*   What partner must implement:
    *   Obtain contextToken from backend.
    *   Handle results & events.

* * *

**Slide 6 – Journey Configuration (JM + OpenAPI)**

*   JM describes:
    *   Steps (`web`, `native`, `server`, `terminal`)
    *   Step transitions
    *   Bindings to operationIds
    *   Security (allowed origins, pinning)
*   OpenAPI:
    *   Defines all backend operations.

* * *

**Slide 7 – Security & Compliance**

*   OIDC + PKCE (ASWebAuthSession)
*   TLS + ATS + certificate pinning
*   Origin validation & signed bridge messages
*   Device integrity & attestation
*   Signed manifests & remote kill‑switches

* * *

**Slide 8 – Example: Money Transfer Journey**

*   Steps:
    *   Intro → Recipient → Amount & Fees → Review & Quote → Biometric → Submit
*   APIs:
    *   ValidateRecipient, CalculateFee, GetQuote, CreateTransfer
*   Features:
    *   FX quote expiry
    *   Biometric confirmation
    *   Idempotent transfer submission

* * *

**Slide 9 – Non‑Functional & Operations**

*   Performance targets (TTFP, latency).
*   Observability (traceparent, metrics).
*   CI/CD:
    *   Build & test.
    *   XCFramework packaging.
    *   Release & artifacts.

* * *

**Slide 10 – Next Steps & Open Questions**

*   Implementation plan / timeline.
*   Open questions (e.g., which markets first, which journeys next).
*   Decisions required from stakeholders.

* * *

Part 2 – Template Filled for a Second Journey: Credit Card Application
======================================================================

Below is a **concise design** for a **Credit Card Application** journey using the same structure. You can paste this into a separate doc and refine.

* * *

1\. Document Information
------------------------

*   **Title:** HSBC Credit Card Application Journey – iOS SDK Design
*   **Version:** 0.1
*   **Author(s):** \[Your Name\]
*   **Date:** \[DD/MM/YYYY\]
*   **Status:** Draft
*   **Stakeholders:** Architecture, Security, Cards Product, Mobile, Partner Tech

* * *

2\. Introduction
----------------

### 2.1 Purpose

Define how the **Credit Card Application (CCA)** journey is implemented on top of the **journey‑agnostic HSBCPartnerSdk for iOS**, reusing the same runtime, manifest, and bridge framework as Account Opening and Money Transfer.

### 2.2 Scope

*   In scope:
    *   CCA journey flow configuration (JM).
    *   Required backend APIs (OpenAPI).
    *   Hosted web pages and bridge usage.
*   Out of scope:
    *   Card decisioning engine internals.
    *   Risk rules and scoring configuration.

### 2.3 Audience

*   iOS developers implementing the SDK.
*   Cards product & operations.
*   Security & Architecture.
*   Partner technical teams using CCA.

* * *

3\. Background & Objectives
---------------------------

### 3.1 Business Context

*   HSBC wishes to acquire credit card customers through partner apps.
*   Current card application process is web‑only and disconnected from partner journeys.
*   Using the journey‑agnostic SDK, we can embed card applications in partner apps with HSBC maintaining full control.

### 3.2 Objectives

*   Provide a **Card Application Journey** accessible via SDK without new app releases.
*   Ensure strong compliance for disclosures and KYC/AML validation.
*   Support multiple card products and campaigns.

### 3.3 Assumptions & Constraints

*   Card eligibility and decisioning are provided by HSBC backend services.
*   Partners will provide minimal prefill data with customer consent.
*   Journey must comply with local credit rules (e.g., income verification, consent handling).

* * *

4\. High‑Level Architecture
---------------------------

*   Same architecture as AO/MT:
    *   Partner app calls `startJourney(journeyId: "credit_card_application", ...)`.
    *   SDK loads JM + OAS for CCA.
    *   HSBC card application pages run inside WKWebView.
    *   Backend provides eligibility, offers, and application submit APIs.

* * *

5\. Functional Requirements
---------------------------

### 5.1 Core Use Cases

*   New credit card application:
    *   Prefilled customer data (if available).
    *   Choose card product.
    *   Provide income/employment details.
    *   Upload supporting documents (if required).
    *   Accept terms and conditions.
    *   Submit application.

### 5.2 User Journey (High-Level)

1.  Intro & consent.
2.  Eligibility / product selection.
3.  Personal and contact details (prefilled / editable).
4.  Financial details (income, employer, liabilities).
5.  Supporting documents (if required).
6.  Review and confirm.
7.  Decision:
    *   Instant approval / conditional approval / pending / declined.

### 5.3 Partner Integration Requirements

*   Server‑to‑server prefill API to create `contextToken` with:
    *   Customer basic data.
    *   Consent evidence.
*   The partner app:
    *   Calls `startJourney` with `journeyId:"credit_card_application"` and `contextToken`.
    *   Handles final `JourneyResult` and optional application status polling.

* * *

6\. Public API Design (Reuse)
-----------------------------

*   Same as generic SDK:
    *   `HSBCInitConfig` – no CCA‑specific fields.
    *   `JourneyStartParams` – `journeyId = "credit_card_application"`.
    *   `JourneyResult` payload:
        *   On completed: `{applicationId, status: "APPROVED|PENDING|DECLINED"}`.

* * *

7\. Internal Architecture (Reuse)
---------------------------------

*   No new components; CCA uses:
    *   Runtime Engine
    *   CCA Journey Manifest
    *   CCA OpenAPI bundle
    *   Hosted CCA pages
    *   Bridge
    *   ApiClient
*   Document capture and other device capabilities can reuse existing plugins (if needed).

* * *

8\. CCA Journey Manifest Model
------------------------------

### 8.1 Steps (Conceptual)

Example step list (actual manifest to specify URLs and bindings):

1.  `cca_intro` (web) – Welcome, consent.
2.  `cca_eligibility` (web) – Check basic eligibility, call `CheckEligibility`.
3.  `cca_product_select` (web) – Select card product; call `ListCardProducts`.
4.  `cca_personal_details` (web) – Prefilled + editable details; call `ValidateApplicant`.
5.  `cca_financial_details` (web) – Income, employment, liabilities; call `ValidateFinancials`.
6.  `cca_documents` (native or web) – Optional document upload; call `UploadDocument`.
7.  `cca_review` (web) – Summary of selected product and data; call `CalculateOffer` if needed.
8.  `cca_submit` (server) – `CreateCardApplication` with idempotency key.
9.  `end_complete`, `end_pending`, `end_declined` (terminal).

### 8.2 Bindings & Transitions

*   `onEvent:"checkEligibility"` → `CheckEligibility` (OAS)
*   `onEvent:"validateApplicant"` → `ValidateApplicant`
*   `onEvent:"validateFinancials"` → `ValidateFinancials`
*   `onEvent:"calculateOffer"` → `CalculateOffer`
*   `cca_submit` → `CreateCardApplication`

Transitions defined by JM:

*   `eligibilityPassed` → `cca_product_select`
*   `eligibilityFailed` → `end_declined`
*   `submitSuccess` → `end_complete` (if instant decision)
*   `submitPending` → `end_pending`

* * *

9\. Bridge Protocol (CCA Usage)
-------------------------------

*   CCA pages use the same bridge methods:
    *   Fire `bridge_hello` on load.
    *   Use `hsbc.request("checkEligibility", {...})`, etc.
    *   Emit events like `eligibilityPassed`, `eligibilityFailed`, `submitSuccess`, `submitPending`.
*   Device capabilities:
    *   If local document capture plugin exists, steps such as `cca_documents` can call `captureDocument` via bridge.

* * *

10\. API Integration (CCA OpenAPI)
----------------------------------

Typical CCA backend operations (operationIds):

*   `CheckEligibility`
*   `ListCardProducts`
*   `ValidateApplicant`
*   `ValidateFinancials`
*   `CalculateOffer` (optional)
*   `UploadDocument` (optional)
*   `CreateCardApplication`
*   `GetCardApplicationStatus` (for polling)

All defined in an OpenAPI 3 spec; the manifest references them by operationId.

* * *

11\. Security & Compliance (CCA Focus)
--------------------------------------

*   Additional considerations:
    *   Clear consent wording for credit checks and bureau pulls.
    *   Data retention and privacy compliance for credit data.
    *   Extra KYC/AML checks for higher‑risk applicants.
*   Otherwise, same framework:
    *   Secure transport, bridge, device integrity, and manifest signing.

* * *

12\. Error Handling & Observability
-----------------------------------

*   Common SDK error codes reused.
*   CCA‑specific conditions:
    *   `eligibilityFailed` handled as a normal business outcome (not a technical error).
    *   `complianceHold` or `MORE_INFO` states for additional document requests.
*   Observability:
    *   Step drop‑off rates.
    *   Eligibility pass rate.
    *   Instant approval vs pending vs decline distribution.

* * *

13\. Non‑Functional Requirements (CCA)
--------------------------------------

*   Performance: similar to AO journey; no extra heavy operations on device.
*   Reliability: idempotent submission of card applications; safe retries on network failures.

* * *

14\. Build, Packaging & Distribution
------------------------------------

*   No change vs core SDK:
    *   CCA is introduced via new JM + OAS + hosted pages.
    *   No new SDK binary release required if capabilities already exist.

* * *

15\. Testing & Validation (CCA)
-------------------------------

*   Unit tests:
    *   Manifest validation for CCA.
    *   OperationId mapping and error handling.
*   Integration tests:
    *   End‑to‑end CCA flow in staging.
*   UI tests:
    *   CCA demo journey in test app.

* * *

16\. Risks & Mitigations (CCA)
------------------------------

Examples:

*   **Risk:** Incorrect eligibility logic in manifest → **Mitigation:** A/B testing and staged rollout.
*   **Risk:** Inconsistent product list between partner and HSBC → **Mitigation:** HSBC master data only; partner displays what the pages show.
*   **Risk:** Misuse of credit data → **Mitigation:** Strict logging, masking, and limited exposure in SDK.

* * *

17\. Open Questions & Future Enhancements (CCA)
-----------------------------------------------

*   Should we support in‑journey upsell (e.g., add-ons, loyalty programs)?
*   How to support targeted offers based on partner profile?
*   Should we expose status tracking in partner app after submission (separate journey)?

* * *

If you’d like, I can now:

*   Turn the **HLD** into a fully written “management deck” script (speaker notes per slide), or
*   Help you adapt the **same template** for another journey type (e.g., Loans, Insurance).

## Prompt:
Turn the HLD into a fully written “management deck” script (speaker notes per slide)

## Response:
Thought for 3m 11s

Here’s a full **management-ready deck script** you can drop into PowerPoint.  
For each slide I include:

*   **Slide title**
*   **Slide content (bullets)**
*   **Speaker notes** – what to say out loud (you can paste this into the Notes section).

You can adjust names (HSBC, Ctrip, etc.) and numbers to match your org.

* * *

Slide 1 – Title
---------------

**Title:**  
HSBC Journey‑Agnostic iOS Partner SDK

**Subtitle:**  
Architecture & Design Overview

**Content:**

*   Deliver HSBC journeys inside partner apps
*   One SDK, many products and markets
*   Secure, compliant, centrally controlled

**Speaker notes:**

> “Today I’ll walk through the architecture and design of our **journey‑agnostic iOS Partner SDK**.
> 
> The goal is simple: let partners embed HSBC journeys—like **Account Opening** and **Money Transfer**—directly inside their own apps, while **HSBC retains full control** over product logic, compliance text, and security.
> 
> We’ll look at what the SDK does, how it’s structured, how partners integrate, and how this scales to new journeys and markets with minimal extra work.”

* * *

Slide 2 – Why We Need This (Business Context)
---------------------------------------------

**Title:**  
Why a Journey‑Agnostic SDK?

**Content:**

*   Partners are key digital acquisition channels
*   Today’s integrations are one‑off and costly
*   Regulatory content must remain under HSBC control
*   Need “configure, not rebuild” for each new journey

**Speaker notes:**

> “Our partner ecosystem is a major growth lever: travel apps, lifestyle apps, and super‑apps all want to offer HSBC products.
> 
> Historically, each integration has been **custom**: custom screens, custom API integrations, custom QA, and custom approvals. That doesn’t scale. It slows down launches and increases risk.
> 
> At the same time, **regulatory content and KYC processes must remain under HSBC control**, not copied into each partner’s codebase.
> 
> The aim of this SDK is to turn integrations into a **configuration problem** instead of a **rebuild problem** – so adding a new journey or market doesn’t mean starting from scratch.”

* * *

Slide 3 – Concept: One SDK, Many Journeys
-----------------------------------------

**Title:**  
Concept: One SDK, Many Journeys

**Content:**

*   Single iOS SDK embedded in partner app
*   Journeys defined by **signed manifests** + **OpenAPI**
*   HSBC hosts all journey pages
*   SDK handles security, container, and integration

**Speaker notes:**

> “The key idea: we ship **one iOS SDK** that partners embed in their app.
> 
> Instead of hard‑coding product flows inside the SDK, we describe each journey using:
> 
> *   A **Journey Manifest**: a signed JSON document that defines steps, transitions, and which APIs to call.
> *   An **OpenAPI specification**: that defines the backend APIs by operationId.
>     
> 
> HSBC hosts all the journey pages—so forms, disclosures, and KYC screens live on our side.
> 
> The SDK’s job is to provide a **secure container**, handle the **authentication**, enforce **security policies**, and expose a **simple integration API** for partners.”

* * *

Slide 4 – High‑Level Architecture
---------------------------------

**Title:**  
High‑Level Architecture

**Content:**

*   Partner App
    *   Calls `initialize` and `startJourney`
*   HSBC iOS SDK
    *   Runtime engine, container, bridge, plugins
*   HSBC Backends
    *   Hosted pages, manifests, APIs, remote config

**Speaker notes:**

> “Architecturally, there are three main pieces:
> 
> **1\. Partner App** – they integrate our SDK and call two main methods: `initialize`, and `startJourney` with a `journeyId` and a `contextToken`.
> 
> **2\. HSBC iOS SDK** – this is our code. It loads the right **manifest** and **OpenAPI spec**, presents HSBC pages in a secure container, runs the step‑by‑step journey logic, and enforces our security rules.
> 
> **3\. HSBC Backends** – we host the journey UI pages, publish the manifests and OpenAPI specs, and expose the actual banking APIs for account opening, transfers, and so on.
> 
> This split means partners are responsible only for **embedding the SDK and passing tokens**; HSBC is responsible for the **journey itself**.”

* * *

Slide 5 – Partner Integration View
----------------------------------

**Title:**  
What Partners Do (and Don’t Do)

**Content:**

*   **Partner does:**
    *   Integrate SDK once
    *   Obtain `contextToken` backend‑to‑backend
    *   Call `startJourney(journeyId, contextToken)`
    *   Handle `JourneyResult` and events
*   **Partner does NOT:**
    *   Rebuild HSBC forms or logic
    *   Handle KYC rules and wording
    *   Call core banking APIs directly

**Speaker notes:**

> “From the partner’s perspective, integration is deliberately simple.
> 
> They do a **backend‑to‑backend call** to us to pre‑create context—this returns a `contextToken` that represents the customer, consents, and any prefill data.
> 
> In the app, they call `startJourney` with that `journeyId` and token. That’s it.
> 
> They **do not** build or manage HSBC forms, they don’t write any compliance copy, and they don’t call our core banking APIs directly. All of that remains on HSBC side, controlled by us.
> 
> At the end, they receive a simple `JourneyResult`—completed, pending, cancelled, or failed—plus identifiers like an applicationId or transferId.”

* * *

Slide 6 – Journey Configuration: Manifest & OpenAPI
---------------------------------------------------

**Title:**  
How Journeys Are Configured

**Content:**

*   **Journey Manifest (JM)**
    *   Steps (web/native/server/terminal)
    *   Page URLs, transitions, guards
    *   Which APIs to call and when
*   **OpenAPI Bundle**
    *   Backend API operations (`operationId`)
    *   Request/response schemas
*   SDK reads both → runs the journey

**Speaker notes:**

> “Under the hood, each journey is described by two configuration artifacts.
> 
> First, the **Journey Manifest**. This is a signed JSON file that defines:
> 
> *   The step flow (which pages or native steps exist),
> *   The transitions between steps,
> *   Which events trigger which API calls, and
> *   What the terminal outcomes look like.
>     
> 
> Second, the **OpenAPI bundle**. This describes the backend APIs by `operationId` and schema. The manifest references those operationIds.
> 
> When the SDK starts a journey, it downloads and validates the manifest and OpenAPI file, and then **executes the journey** as a state machine.
> 
> The big benefit: if we need to change the journey, we update the **manifest and pages**, not the partner’s app.”

* * *

Slide 7 – Security & Compliance Story
-------------------------------------

**Title:**  
Security & Compliance by Design

**Content:**

*   User auth: OIDC + PKCE via system browser
*   Transport: TLS, ATS, certificate pinning
*   Hosted pages: origin checks, JS bridge signing
*   Device: jailbreak checks, attestation sent to HSBC
*   Data: minimal PII on device, encrypted snapshots

**Speaker notes:**

> “For management, the most important part is the **security and compliance posture**.
> 
> *   Authentication uses **OIDC with PKCE**, handled by the system browser so credentials aren’t held by the partner app.
> *   All transport uses **TLS**, with Apple’s App Transport Security and certificate **pinning** for native API calls.
> *   Our hosted pages only run inside a restricted WebView bound to HSBC domains, and the JavaScript bridge is locked down: origin is verified, and all responses from the SDK are signed using **JWS** for integrity.
> *   We also capture **device integrity signals**—root/jailbreak and attestation—and pass those to our backends for risk decisions.
> *   On the device, we avoid storing PII. Any resume state is **PII‑free and encrypted in the Keychain**.
>     
> 
> The net effect is: partners get powerful journeys, but HSBC remains fully in control of security and compliance.”

* * *

Slide 8 – Example Journey: Account Opening
------------------------------------------

**Title:**  
Example: Account Opening Journey

**Content:**

*   Steps (conceptual):
    *   Intro & eligibility
    *   Personal details & contact
    *   KYC / document checks
    *   Review & consent
    *   Submit & decision
*   All rendered via HSBC pages in SDK container
*   Partner sees only a final result + applicationId

**Speaker notes:**

> “Let’s ground this in something concrete: **Account Opening**.
> 
> In this journey, our manifest defines steps like:
> 
> *   Introduction and eligibility,
> *   Data capture for personal and contact details,
> *   KYC document checks and liveness if required,
> *   A final review and consent screen, and then
> *   Submission and decision.
>     
> 
> All of these steps are implemented as **HSBC‑hosted pages** that run inside the SDK container. The SDK enforces the sequence and security, but the actual form fields, copy, and rules live on HSBC servers.
> 
> The partner app sees a smooth in‑app flow. At the end, they receive an `applicationId` and a status—such as approved, pending, or declined—through a single `JourneyResult`.”

* * *

Slide 9 – Example Journey: Money Transfer
-----------------------------------------

**Title:**  
Example: Money Transfer Journey

**Content:**

*   Steps:
    *   Intro → recipient → amount & fees
    *   Review & FX quote
    *   Biometric confirmation
    *   Submit transfer
*   APIs:
    *   ValidateRecipient, CalculateFee, GetQuote, CreateTransfer
*   Idempotent & FX‑safe

**Speaker notes:**

> “The same SDK also powers **Money Transfer** journeys.
> 
> The manifest defines a flow like:
> 
> *   Intro,
> *   Recipient details with validation,
> *   Amount and fee calculation,
> *   Review plus FX quote display,
> *   Biometric confirmation using the native plugin,
> *   And finally a server‑side submission of the transfer.
>     
> 
> The APIs are standardized via OpenAPI: `ValidateRecipient`, `CalculateFee`, `GetQuote`, and `CreateTransfer`.
> 
> We also enforce **idempotency** on the transfer submission step, and we handle FX quote expiry inside the manifest, so we don’t send transfers against expired quotes.
> 
> Again, partners simply call `startJourney("money_transfer")` and receive a result with a `transferId` and status.”

* * *

Slide 10 – Benefits for HSBC & Partners
---------------------------------------

**Title:**  
Key Benefits

**Content:**

*   For HSBC:
    *   Centralized control of journeys and compliance
    *   Faster rollout of new products and changes
    *   Reusable security and integration layer
*   For Partners:
    *   One‑time integration effort
    *   No need to build banking UI or handle KYC rules
    *   Better user experience (in‑app, consistent)

**Speaker notes:**

> “Summarizing benefits:
> 
> For **HSBC**, the SDK gives us:
> 
> *   Centralized control of all journeys and regulatory content,
> *   Faster time‑to‑market for new products and changes, because we update manifests and hosted pages, and
> *   A single, reusable security and integration layer across all partners and journeys.
>     
> 
> For **partners**, this is attractive because:
> 
> *   Integration is **one‑time** and relatively light—just initialize the SDK, get context tokens, and call `startJourney`.
> *   They don’t have to understand or replicate detailed banking rules and regulations.
> *   Their users stay inside the app and get a consistent HSBC experience.
>     
> 
> This bridges the gap between strong governance on our side and speed + convenience on theirs.”

* * *

Slide 11 – Delivery Plan & Next Steps
-------------------------------------

**Title:**  
Delivery Plan & Next Steps

**Content:**

*   Phase 1: SDK MVP + Account Opening
*   Phase 2: Money Transfer journey
*   Phase 3: Additional journeys (e.g., Cards)
*   Governance:
    *   Security sign‑off
    *   Manifest & release process
*   Ask from management:
    *   Confirm scope & priorities
    *   Approve rollout roadmap

**Speaker notes:**

> “In terms of delivery:
> 
> **Phase 1** is the SDK foundation plus the Account Opening journey with at least one pilot partner.
> 
> **Phase 2** brings in the Money Transfer journey, reusing the same SDK and infrastructure.
> 
> **Phase 3** is about adding further journeys such as Credit Card Application, address updates, or other servicing flows.
> 
> Alongside this, we need clear **governance**:
> 
> *   Security and architecture sign‑off on the SDK design,
> *   A controlled process for approving and signing manifests and OpenAPI changes,
> *   And a release process for new journeys and markets.
>     
> 
> The key asks today are:
> 
> *   Confirm this scope and approach,
> *   Prioritize which journeys and partners we target first, and
> *   Approve the rollout roadmap so we can move into full implementation and partner onboarding.”
>     

* * *

If you want, I can now:

*   Compress this into a **1–2 page executive summary script** for your senior leadership, or
*   Draft a **short add‑on slide** specifically about the Credit Card Application journey using the same style.



---
Powered by [ChatGPT Exporter](https://www.chatgptexporter.com)