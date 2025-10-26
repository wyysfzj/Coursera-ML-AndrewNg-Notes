Here is a revised version of the SDK design, incorporating the expert recommendations to enhance security, resilience, and developer experience.

***

# HSBC SDK Design (Revised)

**User:** Anonymous (<os86@naifeistation.com>)

**Original Created:** 10/25/2025 21:56:01

**This Version (v2):** 10/26/2025 11:12:00

## Prompt:

the attached image show the SDK solution of SCB. SCB generate SDK for its account opening api and account opening page, Ctrip use this SDK in Ctrips'app. customer can open SCB account in Ctrip APP. Now I need produce the similar SDK for HSBC. please provide the detail design

## Response:

Below is a complete, implementation-ready design for an **HSBC Partner Account-Opening SDK (AOSDK), Version 2**. This design incorporates expert revisions to create a more secure, resilient, and flexible solution that mirrors the SCB/Ctrip pattern.

***

### 0) Executive summary

**Goal:** Provide a mobile SDK (iOS + Android) that launches an **HSBC-hosted account-opening (AO) experience** inside the partner app, prefilled with partner-provided data, and submits the application back to HSBC—securely, compliantly, and with excellent observability.

**Core design choices (Revised)**

*   **Hybrid UI**: Native SDK that wraps a **hosted HSBC AO Web** (WebView/Custom Tabs/SFSafariViewController).
*   **On-Demand Native Plugins**: **Optional, downloadable modules** (KYC doc scan, liveness, e-signature) to reduce friction, with seamless **web-based fallbacks** for all functions.
*   **Security**: OIDC + PKCE for mobile, **remotely-configurable Public Key Pinning**, device attestation (Play Integrity/App Attest) **reported to the server for policy decisions**, hardware-backed keystore, JWE/JWS payloads, and a **server-to-server prefill token** so PII never transits the client.
*   **Resilience**: **Mandatory SDK version checking** with a "force upgrade" mechanism and a stateless API design to handle token expiry.
*   **Partner-friendly**: Simple, stateless SDK API (`startAccountOpening(...)`), deep-link handling, and comprehensive samples.

***

### 1) End-to-end customer journey (mirrors your SCB flow)

**HSBC × Ctrip AO Journey**

1.  **Entry point** in Ctrip (e.g., Finance tab → “Open HSBC Account”).
2.  **Prefill consent & token**: Ctrip obtains user consent and requests an HSBC **Prefill Token** via its backend (server-to-server) using partner credentials.
3.  **Launch HSBC AO** in-app via the SDK: `HSBCAOSdk.start(...)`.
4.  **HSBC AO pages** open in a secure WebView (or Custom Tab/SafariView) inside Ctrip; prefilled fields appear (name, ID, address, etc.).
5.  **KYC**: SDK prompts for **on-demand download** of native KYC module. If successful, user performs high-friction tasks (doc capture, liveness) natively. If it fails (or is disabled), user seamlessly uses the **HSBC-hosted web KYC** flow inside the WebView.
6.  **Branch appointment** (if required) scheduled in-flow.
7.  **Submission**: Application submitted; SDK returns **result** (success with `applicationId`, pending, or error) to Ctrip.
8.  **Status & resume**: Customer can resume via deep link; Ctrip can show application status via **Application Status API**.

***

### 2) High-level architecture

```
[ Partner App (Ctrip) ]
     │
     │  AOSDK (HSBC) – iOS/Android
     │   ├─ Core (Session, Tokens, Config, Logger, Events)
     │   ├─ Security (OIDC+PKCE, Key Pinning, Attestation Reporter)
     │   ├─ UI Container (WebView / Custom Tab / SFSafariViewController)
     │   ├─ On-Demand KYC Loader (loads modules)
     │   ├─ Prefill Adapter (Token-based prefill)
     │   └─ Appointment & Status Adapters
     │
     ├─ [Optional Native KYC Modules (Downloaded On-Demand)]
     │   ├─ [Doc Scan Module]
     │   └─ [Liveness Module]
     │
Internet (TLS + Public Key Pinning, JWS/JWE)
     │
[ HSBC Partner Gateway ]
     ├─ OAuth/OIDC (Authz Code + PKCE)
     ├─ Prefill API (server-to-server)
     ├─ AO Web (responsive pages + web-based KYC fallbacks)
     ├─ Risk & Policy Engine (consumes attestation data)
     ├─ KYC/AML/Screening (internal or 3rd-party)
     ├─ Document Service (upload, OCR, classification)
     ├─ Application Orchestrator (BPMN/Rules)
     └─ Application Status API

```

***

### 3) Packaging & platforms

*   **Android**:

    *   **Core SDK**: AAR (min SDK 23+). \$\leq\$ 4MB. Uses **Custom Tabs** by default; falls back to secure WebView.
    *   **KYC Plugins**: Delivered as **Dynamic Feature Modules** (downloaded on-demand).
*   **iOS**:

    *   **Core SDK**: XCFramework (iOS 14+). \$\leq\$ 4MB. Uses **ASWebAuthenticationSession**.
    *   **KYC Plugins**: Delivered as **On-Demand Resources** (downloaded on-demand).
*   **Web (Future Consideration)**: A browser-based SDK is a **separate, major project** outside this initial scope, requiring its own threat model for token storage and XSS/CSRF.
*   **Binary distribution**: Distributed via a **private, credentialed package repository** (e.g., private Artifactory, GitHub Packages).

***

### 4) Security & compliance (Revised)

*   **Identity & Auth**

    *   OAuth 2.1 Authorization Code Flow with **PKCE**.
    *   **OIDC** for standardized ID tokens; **nonce** + **state** validated.
*   **Transport & Channel**

    *   TLS 1.2+ with **Public Key Pinning** (pins the SPKI of the leaf/intermediate).
    *   **Remotely Configurable Pinning Policy**: The SDK will fetch an updated pin-set (JWS-signed) from a config endpoint. This allows for emergency pin rotation or disabling pinning without a mandatory app update.
    *   **mTLS** for **partner-backend \$\rightarrow\$ HSBC** server-to-server APIs (Prefill, Webhook registration).
*   **Data protection**

    *   **Prefill Token** pattern: Partner backend posts PII \$\rightarrow\$ HSBC Prefill API \$\rightarrow\$ returns **single-use, short-TTL token**. The SDK only ever sees this token. (No PII leaks on device.)
    *   On-device secrets (refresh tokens) stored in **Android Keystore**/**iOS Keychain** (hardware backed).
*   **Device integrity**

    *   **Server-Side Policy**: The SDK **collects** device integrity signals (Play Integrity token, App Attest assertion, root/jailbreak status) and **passes them to the HSBC Partner Gateway** (e.g., in the OIDC request).
    *   The **HSBC server** (Risk & Policy Engine) makes the final decision whether to block the flow, not the client. This allows for flexible, real-time risk-based decisions.
*   **Crypto**

    *   JWT/JWS for request non-repudiation; JWE for PII payloads.
*   **Regulatory**

    *   GDPR/PDPA/etc. controls via **regional endpoints**.
    *   **Consent capture**: SDK requires partner to present HSBC terms; **consent artifacts** posted to HSBC.
*   **Threat modeling**

    *   STRIDE coverage, plus specific mitigation for: replay (nonce + exp), MITM (key pinning), injection (strict CSP on AO Web), code tamper (attestation *reporting*), token theft (PKCE + binding).

***

### 5) Integration patterns

*   **Hosted AO (Recommended)**: UI lives on HSBC AO Web. SDK provides secure container, identity, events, and *optional* native KYC hooks.

    *   Pros: Rapid rollout, consistent compliance, flexible content updates.
    *   Cons: Limited partner theming.

***

### 6) Public SDK interface (Revised)

The API is now **stateless**. The `HSBCInitConfig` is passed into the `start` method, removing the need for a global `initialize` call. This makes the SDK more flexible for partners supporting multiple regions (e.g., HSBC China and HSBC Hong Kong) with different configs.

#### Common types

Kotlin

```
// Kotlin (Android)
data class HSBCInitConfig(
  val environment: Environment,       // SANDBOX | PRODUCTION
  val partnerId: String,              // issued by HSBC
  val clientId: String,               // OIDC client id
  val redirectScheme: String,         // e.g., ctrip://hsbc/callback
  val locale: Locale = Locale.SIMPLIFIED_CHINESE,
  val theme: HSBCTheme? = null
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

#### Android usage (Revised)

Kotlin

```
// No global initialize() needed.
// Set listener once (optional)
HSBCAOSdk.setEventListener(object: AOEventListener {
  override fun onEvent(name: String, attributes: Map<String, Any>) {
    analytics.log(name, attributes)
  }
})

// Call startAccountOpening with the specific config for this flow
lifecycleScope.launch {
  val config = HSBCInitConfig(
    environment = Environment.SANDBOX,
    partnerId = "ctrip_cn",
    clientId = "ctrip-hsbc-ao-mobile",
    redirectScheme = "ctrip"
  )
  
  val params = AOStartParams(
    prefillToken = prefillTokenFromServer,
    productCode = "HSBC_CN_RP_CURR"
  )

  val result = HSBCAOSdk.startAccountOpening(
    activity = this@CtripActivity,
    config = config,
    params = params
  )
  
  when (result) {
    is AOResult.Completed -> showSuccess(result.applicationId)
    is AOResult.Pending   -> showPending(result.applicationId)
    is AOResult.Cancelled -> showCancelled()
    is AOResult.Failed    -> {
      if (result.error.code == "PREFILL_TOKEN_EXPIRED") {
        // Silently get a new token and retry
        val newPrefillToken = getNewPrefillToken()
        val newParams = params.copy(prefillToken = newPrefillToken)
        HSBCAOSdk.startAccountOpening(activity, config, newParams) // Simplified retry
      } else if (result.error.code == "SDK_UPDATE_REQUIRED") {
        promptUserToUpdateApp()
      } else {
        showError(result.error)
      }
    }
  }
}

```

#### iOS usage (Swift, Revised)

Swift

```
// Set listener once (optional)
HSBCAOSdk.setEventListener { name, attributes in
  Analytics.logEvent(name, parameters: attributes)
}

// Call startAccountOpening with the specific config for this flow
let config = HSBCInitConfig(
  environment: .sandbox,
  partnerId: "ctrip_cn",
  clientId: "ctrip-hsbc-ao-ios",
  redirectScheme: "ctrip"
)

let params = AOStartParams(
  prefillToken: prefillTokenFromServer,
  productCode: "HSBC_CN_RP_CURR"
)

HSBCAOSdk.startAccountOpening(
  from: self,
  config: config,
  params: params
) { result in
  switch result {
  case .completed(let appId): self.showSuccess(appId)
  case .pending(let appId):   self.showPending(appId)
  case .cancelled:            self.showCancelled()
  case .failed(let err):
    if err.code == "PREFILL_TOKEN_EXPIRED" {
      // Silently get a new token and retry
      self.getNewPrefillToken { newPrefillToken in
        let newParams = AOStartParams(prefillToken: newPrefillToken, ...)
        HSBCAOSdk.startAccountOpening(from: self, config: config, params: newParams) { ... }
      }
    } else if err.code == "SDK_UPDATE_REQUIRED" {
      self.promptUserToUpdateApp()
    } else {
      self.showError(err)
    }
  }
}

```

***

### 7) Data contract (prefill & consent)

(No change to this section; the server-to-server contract remains the same.)

```
POST /partner/v1/prefill
Headers: mTLS, Authorization: Bearer <partner_access_token>
Body (JWE):
{
  ...
  "applicant": { ... },
  ...
}

Response 200 OK:
{
  "prefillToken": "ptok_eyJhbGciOi... (JWS)",
  "expiresIn": 900 // 15 minutes
}

```

***

### 8) HSBC partner-facing APIs (server side)

(No change to this section.)

***

### 9) In-app AO Web endpoints (SDK-handled)

(Revised to include explicit bridge security)

*   `GET /ao/start?prefillToken=...&locale=...&product=...`
*   **Bridge Security**: The native `window.hsbcBridge.postMessage(...)` interface will be **domain-locked**. It will *only* be injected into the `WKWebView` / `WebView` if the loaded URL's domain matches the **exact, whitelisted HSBC AO Web domain** provided in the SDK's configuration. All other web content (e.g., 3rd-party links) will have no access to the native bridge.
*   **Native plugin hooks**:

    *   `captureDocument({type:"ID_FRONT"})`
    *   `performLiveness()`
    *   `scanNFC({docType:"passport"})`

***

### 10) KYC plugins (Revised)

*   **On-Demand Loading**: Plugins (Doc capture, Liveness, NFC) are **not bundled** in the core SDK. They will be loaded as **Android Dynamic Features** or **iOS On-Demand Resources** at the point they are needed. This keeps the partner's initial app install size minimal.
*   **Web-Based Fallback**: For **every** native plugin function, a 100% web-based equivalent *must* exist on the HSBC AO Web. If a native plugin fails to download (no network), fails to load (crash), or is remotely disabled by HSBC, the SDK will **seamlessly and automatically fall back** to the web-based equivalent inside the WebView. The user journey is not broken.

***

### 11) Branch appointment module

(No change to this section.)

***

### 12) Events & analytics (Revised)

Telemetry is split into two mandatory categories for support and legal reasons.

| **Event name**       | **When**              | **Category**   | **Attributes**                   |
| :------------------- | :-------------------- | :------------- | :------------------------------- |
| `AO_BEGIN`           | AO UI opened          | **Analytic**   | `product`, `campaignId`          |
| `AO_FORM_PREFILLED`  | Prefill redeemed      | **Analytic**   | `fieldsPrefilledCount`           |
| `KYC_STARTED`        | First KYC step        | **Analytic**   | `method: NATIVE` or `WEB`        |
| `KYC_COMPLETED`      | All KYC steps done    | **Analytic**   | `result: PASS`                   |
| `AO_SUBMITTED`       | Application submitted | **Analytic**   | `applicationId`                  |
| `AO_EXITED`          | User exits            | **Analytic**   | `reason: CANCEL`                 |
| `AO_ERROR`           | Error occurred        | **Diagnostic** | `code`, `message`, `recoverable` |
| `SDK_SECURITY_EVENT` | e.g., Pinning fail    | **Diagnostic** | `type: PINNING_FAIL`             |

*   **Analytic Telemetry**: Performance/UX metrics. Can be toggled by privacy flags.
*   **Diagnostic Telemetry**: PII-free operational events (errors, crashes, security failures). This is **mandatory and non-optional** as it is required for HSBC to support the SDK and monitor for fraud or security attacks.

***

### 13) Error model (Revised)

Added new, specific, actionable error codes for partners.

**Top-level error codes (examples)**

*   `NET_TIMEOUT`, `NET_UNREACHABLE`
*   `AUTH_FAILED`, `AUTH_CANCELLED`
*   `PINNING_VIOLATION` (A security event)
*   `PREFILL_INVALID`
*   **`PREFILL_TOKEN_EXPIRED`** (Recoverable; partner *must* handle this by getting a new token and retrying)
*   `KYC_CAMERA_DENIED`
*   `KYC_MODULE_DOWNLOAD_FAILED` (Recoverable; SDK will fall back to web)
*   **`SDK_UPDATE_REQUIRED`** (Non-recoverable; partner must prompt user to update the app)
*   `AO_POLICY_BLOCKED` (e.g., server blocked a jailbroken device)
*   `UNKNOWN`

***

### 14) Theming, i18n, accessibility

(No change to this section.)

***

### 15) Versioning & compatibility (Revised)

*   **SemVer** for SDK (e.g., `1.3.0`).
*   **Mandatory Version Check**: On every `startAccountOpening` call, the SDK will make a lightweight, signed call to an HSBC config endpoint. This endpoint will return the `minRequiredVersion`. If the current SDK version is less than this minimum, the SDK **will refuse to run** and will immediately return the `AOResult.Failed` with error code `SDK_UPDATE_REQUIRED`. This is a critical security mechanism to remotely disable vulnerable SDKs.
*   **Deprecation policy**: N-1 major version supported for \$\geq\$ 12 months, *contingent on passing the mandatory version check.*

***

### 16) Observability & support (Revised)

*   **HSBC Telemetry**:

    *   **Mandatory Diagnostic Telemetry**: PII-free error rates, security events, and crash reports.
    *   **Opt-in Analytic Telemetry**: Performance metrics (page timings, KYC step durations), device profile (OS only).
*   **Partner Runbook**: FAQ codes (including `PREFILL_TOKEN_EXPIRED` and `SDK_UPDATE_REQUIRED`), escalation paths.

***

### 17) Onboarding steps for Ctrip (partner)

(Revised to include new partner responsibilities)

1.  **Registration**

    *   Exchange legal agreements, scopes, and data minimization matrix.
    *   HSBC issues `partnerId`, OIDC `clientId`, and mTLS certs.
2.  **Environments**

    *   Access to **private package repository** (e.g., Artifactory) credentials.
    *   Sandbox endpoints and test data.
3.  **Prefill integration**

    *   Implement `/partner/v1/prefill` server-to-server with consent evidence.
4.  **SDK integration**

    *   Add dependency from private repo, set redirect scheme, wire deep link.
    *   Implement event listener and result handling.
5.  **Critical Error Handling**

    *   Implement the client-side logic to handle `AOResult.Failed(error.code == "PREFILL_TOKEN_EXPIRED")`. This logic **must** silently fetch a new token from the Ctrip backend and retry the `startAccountOpening` call to avoid a poor user experience.
    *   Implement the client-side logic to handle `AOResult.Failed(error.code == "SDK_UPDATE_REQUIRED")` by showing a dialog prompting the user to update their app from the App Store / Play Store.
6.  **Brand checks**

    *   Apply allowed theming; run through HSBC UX acceptance checklist.
7.  **Security tests**

    *   Pen test on partner build; verify attestation data is being sent.
8.  **Go-live checklist**

    *   Webhook (optional), status API consumption, monitoring dashboards.

***

### 18) Example AO sequence (text sequence diagram)

(No change to this section; the high-level flow is the same.)

```
Ctrip App        Ctrip Backend       HSBC Partner GW             HSBC AO Web & Services
    |                  |                    |                                    |
(1) User taps "Open HSBC Account"                                               |
    |                  |                    |                                    |
(2) Consent for data use ->                                                     |
    |---- POST /prefill (PII+consent, mTLS) --->|                               |
    |                  |<--- prefillToken ------|                                |
    |                                                                            |
(3) HSBCAOSdk.start(config, params)                                              |
    |----(SDK calls config endpoint -> version OK)---->|                          |
    |----(SDK passes attestation token)--------------->|                          |
    |                                                 | (Server policy check OK) |
    |---------------- Open AO Web --------------------------->|                  |
    ...                                                                          ...

```

***

### 19) Non-functional requirements (Revised)

*   **SLA**: 99.9% AO Web availability; 99.95% Partner GW; P95 page interactive < 3.0s on 4G.
*   **Rate limits**: per partner throttle with burst credits.
*   **Resilience**: multi-region active/active, circuit breakers.
*   **Size**:

    *   **Core SDK**: \$\leq\$ 4 MB.
    *   **On-Demand KYC Modules**: \$\leq\$ 25 MB (downloaded only if/when needed).
*   **Privacy**: data minimization; expunge on device after completion; logs redacted.

***

### 20) Sample sandbox configuration

(No change to this section.)

***

### 21) Deliverables checklist

(No change to this section.)

***

### 22) Build plan (phased)

(No change to this section.)



