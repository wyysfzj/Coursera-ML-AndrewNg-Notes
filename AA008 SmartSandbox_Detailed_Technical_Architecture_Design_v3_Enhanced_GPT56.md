# SmartSandbox Detailed Technical Architecture Design v3

## HSBC Developer Portal + SmartMock based Open Banking Integration Simulation Platform

## Executive Summary

SmartSandbox evolves Open Banking sandbox capability from API-level
simulation into a business integration simulation platform.

Traditional sandbox:

    API Request -> Mock Response

SmartSandbox:

    Business Journey
     -> API Orchestration
     -> Dynamic Business State
     -> Events/Webhooks
     -> Integration Validation

The solution combines:

-   HSBC Developer Portal
-   SmartMock execution engine
-   HKMA Open Banking Phase 3 capability
-   PSD2/Open Banking implementation experience
-   Kubernetes-based isolated sandbox architecture

------------------------------------------------------------------------

# 1. Strategic Context

Open Banking integration failures are usually caused by:

-   incorrect consent lifecycle handling
-   unexpected customer states
-   payment workflow issues
-   webhook processing problems
-   missing negative scenarios

A modern sandbox must validate:

"Can a partner complete the real business journey?"

not only:

"Can a partner call the API?"

------------------------------------------------------------------------

# 2. Current HSBC Capability

## Developer Portal

Existing capability:

-   API catalogue
-   API documentation
-   developer onboarding
-   application registration
-   credential management
-   API subscription

The portal already covers the developer experience normally provided by
commercial sandbox portals.

## Open Banking Capability

Proven capability:

-   HKMA Open Banking Phase 3 API onboarding
-   PSD2 implementation experience
-   consent lifecycle support
-   enterprise API delivery

------------------------------------------------------------------------

# 3. SmartMock Foundation

SmartMock is the execution engine.

Capabilities:

-   OpenAPI import
-   dynamic endpoint generation
-   runtime update
-   schema validation
-   custom business logic
-   dynamic mock data
-   proxy capability

Architecture:

``` mermaid
flowchart TB
Request[API Request] --> Router[Dynamic Router]
Router --> Engine[Execution Engine]
Engine --> Script[Business Logic]
Script --> Template[Templates]
Script --> Data[Mock Data]
Script --> Validation[Schema Validation]
Engine --> Response[API Response]
```

------------------------------------------------------------------------

# 4. DAC Comparison

The comparison is not DAC versus a mock server.

The correct comparison is:

    Commercial Open Banking Sandbox

    versus

    HSBC Owned Integration Simulation Platform

  Capability            DAC            HSBC SmartSandbox
  --------------------- -------------- --------------------------
  Developer Portal      Strong         Existing HSBC capability
  API Catalogue         Strong         Existing HSBC capability
  HKMA APIs             Strong         Proven
  PSD2                  Strong         Proven
  API Testing           Strong         Strong
  Consent Journey       Supported      Customizable
  Dynamic Data          Configurable   Advanced
  Business Scenario     Limited        Core capability
  Custom Logic          Limited        Strong
  Strategic Ownership   Vendor         HSBC

------------------------------------------------------------------------

# 5. Target Architecture

``` mermaid
flowchart TB
Developer[TPP Developer]
Portal[HSBC Developer Portal]
Control[SmartSandbox Control Plane]
Manager[Sandbox Manager]
Runtime[SmartMock Runtime]
Scenario[Scenario Engine]
Data[Dynamic Data Engine]
Webhook[Webhook Engine]
Observe[Observability]

Developer --> Portal
Portal --> Control
Control --> Manager
Manager --> Runtime
Runtime --> Scenario
Runtime --> Data
Runtime --> Webhook
Runtime --> Observe
```

------------------------------------------------------------------------

# 6. Sandbox Isolation

Design principle:

One partner:

    One Sandbox
    One Runtime
    Independent Data
    Independent Configuration

Recommended deployment:

-   Kubernetes namespace per sandbox
-   isolated credentials
-   isolated data
-   independent lifecycle

------------------------------------------------------------------------

# 7. Open Banking Journey Simulation

Example:

``` mermaid
sequenceDiagram
TPP->>Sandbox: Create Consent
Sandbox-->>TPP: Consent Created
TPP->>Sandbox: Authorise Consent
Sandbox-->>TPP: Approved
TPP->>Sandbox: Get Account
Sandbox-->>TPP: Account Data
TPP->>Sandbox: Initiate Payment
Sandbox-->>TPP: Payment Accepted
Sandbox->>TPP: Webhook Event
```

------------------------------------------------------------------------

# 8. Scenario Engine

Supports:

## Positive scenarios

-   consent approved
-   payment completed

## Negative scenarios

-   payment rejected
-   timeout
-   retry

## Regulatory scenarios

-   consent expired
-   consent revoked
-   authentication failure

------------------------------------------------------------------------

# 9. Dynamic Data Engine

Entities:

-   customer
-   account
-   transaction
-   consent
-   payment

Example:

``` yaml
customer:
  segment: premium

account:
  currency: HKD
  balance: 100000

payment:
  status: pending
```

------------------------------------------------------------------------

# 10. Authoring Model

## Standard Mode

For normal developers:

-   OpenAPI import
-   templates
-   rules
-   scenario configuration

## Advanced Mode

For expert users:

-   custom business logic
-   Groovy extension
-   domain simulation

Controls:

-   sandbox execution
-   timeout
-   audit
-   versioning
-   approval

------------------------------------------------------------------------

# 11. Security Architecture

Security principles:

-   no production dependency
-   tenant isolation
-   least privilege
-   complete audit

Threat controls:

  Threat              Mitigation
  ------------------- ----------------------
  Script abuse        Restricted execution
  Data leakage        Sandbox isolation
  Resource abuse      Quota/rate limits
  Credential misuse   IAM integration

------------------------------------------------------------------------

# 12. Market Positioning

## DigitalAPI/DAC

Strength:

-   Open Banking specialization
-   mature sandbox capability

Limitation:

-   less HSBC-specific business simulation

## WireMock Cloud

Strength:

-   powerful mocking engine

Limitation:

-   not complete Open Banking journey simulation

## Microcks

Strength:

-   open standards

Limitation:

-   requires additional platform engineering

------------------------------------------------------------------------

# 13. Roadmap

## Phase 0

SmartMock hardening:

-   security controls
-   audit
-   sandbox context

## Phase 1

SmartSandbox MVP:

-   Sandbox Manager
-   isolated runtime
-   dynamic data
-   scenarios
-   webhook
-   replay

## Phase 2

Advanced Simulation:

-   external API virtualization
-   third-party OpenAPI import
-   complex journeys
-   CI integration

## Phase 3

Intelligent Sandbox:

-   AI scenario generation
-   agent-based testing
-   autonomous integration validation

------------------------------------------------------------------------

# Final Recommendation

SmartSandbox should become:

> HSBC-owned Open Banking Integration Simulation Platform

The strategic value:

    Developer Portal
    +
    SmartMock Runtime
    +
    Open Banking Expertise
    +
    Scenario Simulation

    =
    Enterprise Integration Simulation Capability
