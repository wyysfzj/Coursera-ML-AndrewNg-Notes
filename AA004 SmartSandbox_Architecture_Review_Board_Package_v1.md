# SmartSandbox Architecture Review Board Package

## HSBC Developer Portal + SmartMock Based Open Banking Integration Simulation Platform

**Document Type:** Architecture Review Board Submission\
**Version:** 1.0\
**Audience:** Enterprise Architecture Board, Security Architecture,
Platform Architecture, Open Banking Technology Leadership

------------------------------------------------------------------------

# 1. Executive Decision Summary

## 1.1 Decision Required

This paper requests Architecture Review Board approval for the evolution
of HSBC Open Banking sandbox capability from a traditional API sandbox
model into an enterprise-grade Integration Simulation Platform.

The proposed platform:

-   integrates HSBC Developer Portal capability;
-   reuses SmartMock as the execution engine;
-   introduces SmartSandbox control-plane capabilities;
-   provides isolated partner environments;
-   supports end-to-end Open Banking journey simulation.

The decision is not simply:

    DAC replacement

The strategic decision is:

    API Sandbox

    versus

    Enterprise Integration Simulation Platform

------------------------------------------------------------------------

# 2. Executive Summary

## 2.1 Vision

SmartSandbox enables external developers and internal teams to validate
complete business journeys before production integration.

Traditional sandbox:

    Request
     |
    Response

SmartSandbox:

    Business Journey

    Consent Lifecycle

    API Orchestration

    Dynamic Business State

    Events/Webhooks

    Integration Validation

------------------------------------------------------------------------

# 3. Business Drivers

## 3.1 Open Banking Complexity

Modern Open Banking integration requires:

-   API connectivity
-   authentication
-   consent management
-   customer state
-   payment lifecycle
-   asynchronous events
-   error handling

A partner may successfully call APIs but still fail production
integration.

------------------------------------------------------------------------

## 3.2 Strategic Drivers

The platform addresses:

-   faster TPP onboarding;
-   reduced integration failures;
-   reduced support effort;
-   faster regulatory API adoption;
-   reusable simulation capability.

------------------------------------------------------------------------

# 4. Current State Assessment

# 4.1 HSBC Developer Portal

Existing capabilities:

-   API catalogue
-   documentation
-   developer registration
-   application management
-   credential lifecycle
-   API subscription
-   API testing entry point

The portal already provides capabilities normally supplied by external
sandbox vendors.

------------------------------------------------------------------------

# 4.2 Open Banking Capability

Existing proven capability:

## HKMA Open Banking Phase 3

-   API onboarding
-   API publication
-   partner integration

## PSD2 Experience

-   Account Information Service
-   Payment Initiation Service
-   Consent lifecycle
-   regulatory API delivery

------------------------------------------------------------------------

# 5. SmartMock Technical Foundation

## 5.1 Role

SmartMock is the execution engine.

It provides:

-   dynamic API simulation;
-   OpenAPI import;
-   runtime endpoint generation;
-   schema validation;
-   custom behaviour;
-   dynamic data.

------------------------------------------------------------------------

## 5.2 SmartMock Architecture

``` mermaid
flowchart TB

Request[API Request]

Router[Dynamic Router]

Engine[Execution Engine]

Script[Business Logic]

Template[Template Manager]

Data[Mock Data Manager]

Validation[Schema Validation]

Response[Response]

Request --> Router
Router --> Engine
Engine --> Script
Script --> Template
Script --> Data
Script --> Validation
Engine --> Response
```

------------------------------------------------------------------------

# 6. Current Sandbox Market Assessment

## 6.1 DAC / DigitalAPI

Strengths:

-   Open Banking focus
-   developer portal
-   sandbox environment
-   standard onboarding

Limitations:

-   less HSBC-specific customization;
-   less ownership of simulation roadmap.

------------------------------------------------------------------------

## 6.2 Market Position

  Capability            Commercial Sandbox   SmartSandbox
  --------------------- -------------------- --------------------------
  API testing           Strong               Strong
  Developer portal      Strong               Existing HSBC capability
  Consent journey       Supported            Customizable
  Dynamic state         Medium               Advanced
  Scenario simulation   Medium               Core capability
  Custom logic          Limited              Strong
  Strategic ownership   Vendor               HSBC

------------------------------------------------------------------------

# 7. Target Architecture

## 7.1 Architecture Principles

## Principle 1

Sandbox is a simulation environment, not a mock server.

## Principle 2

One partner receives one isolated sandbox.

## Principle 3

Journey-first simulation.

------------------------------------------------------------------------

# 8. Logical Architecture

``` mermaid
flowchart TB

Developer[TPP Developer]

Portal[HSBC Developer Portal]

Control[SmartSandbox Control Plane]

Manager[Sandbox Manager]

Provision[Provisioning Service]

Namespace[Kubernetes Namespace]

Runtime[SmartMock Runtime]

Scenario[Scenario Engine]

Data[Dynamic Data Engine]

Webhook[Webhook Engine]

Observe[Observability]

Developer --> Portal
Portal --> Control
Control --> Manager
Manager --> Provision
Provision --> Namespace
Namespace --> Runtime

Runtime --> Scenario
Runtime --> Data
Runtime --> Webhook
Runtime --> Observe
```

------------------------------------------------------------------------

# 9. Control Plane Design

## 9.1 Sandbox Manager

Responsibilities:

-   create sandbox;
-   destroy sandbox;
-   clone sandbox;
-   reset sandbox;
-   manage lifecycle;
-   enforce quota.

Example API:

    POST /sandboxes

    GET /sandboxes/{id}

    DELETE /sandboxes/{id}

------------------------------------------------------------------------

## 9.2 Provisioning Workflow

``` mermaid
sequenceDiagram

Developer->>Portal: Create Sandbox Request

Portal->>SandboxManager: Provision Sandbox

SandboxManager->>Kubernetes: Create Namespace

Kubernetes->>Runtime: Deploy SmartMock

Runtime->>Data: Initialize Dataset

Runtime-->>Portal: Sandbox Ready

Portal-->>Developer: URL and Credentials
```

------------------------------------------------------------------------

# 10. Runtime Architecture

Each sandbox contains:

-   SmartMock runtime;
-   scenario configuration;
-   mock data;
-   webhook configuration;
-   monitoring integration.

Isolation:

    Partner A

    Namespace A

    Runtime A


    Partner B

    Namespace B

    Runtime B

------------------------------------------------------------------------

# 11. Open Banking Journey Simulation

## 11.1 Consent Flow

``` mermaid
sequenceDiagram

TPP->>Sandbox: Create Consent

Sandbox-->>TPP: Consent Created

TPP->>Sandbox: Authorisation

Sandbox-->>TPP: Approved

TPP->>Sandbox: Account Access

Sandbox-->>TPP: Account Data

TPP->>Sandbox: Payment

Sandbox-->>TPP: Payment Accepted

Sandbox->>TPP: Webhook Event
```

------------------------------------------------------------------------

# 12. Scenario Engine Design

## 12.1 Purpose

Support realistic business behaviour.

Examples:

-   happy path;
-   failure path;
-   timeout;
-   retry;
-   compliance scenario.

------------------------------------------------------------------------

## 12.2 State Model

``` mermaid
stateDiagram-v2

[*] --> ConsentCreated

ConsentCreated --> Approved

Approved --> AccountAccessible

AccountAccessible --> PaymentPending

PaymentPending --> Completed

PaymentPending --> Failed

Failed --> [*]

Completed --> [*]
```

------------------------------------------------------------------------

# 13. Dynamic Data Architecture

Managed entities:

-   customer;
-   account;
-   transaction;
-   consent;
-   payment.

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

# 14. Webhook Architecture

``` mermaid
flowchart LR

Event[Business Event]

Queue[Event Queue]

Emitter[Webhook Emitter]

Partner[TPP Endpoint]

Audit[Audit]

Event --> Queue
Queue --> Emitter
Emitter --> Partner
Emitter --> Audit
```

------------------------------------------------------------------------

# 15. Authoring Model

## 15.1 Standard Mode

Target:

Most developers.

Features:

-   OpenAPI import;
-   templates;
-   rules;
-   scenarios.

------------------------------------------------------------------------

## 15.2 Advanced Mode

Target:

Expert users.

Features:

-   custom logic;
-   Groovy extensions;
-   domain simulation.

Security:

-   sandbox execution;
-   timeout;
-   memory control;
-   audit;
-   approval.

------------------------------------------------------------------------

# 16. Security Architecture

## 16.1 Security Principles

-   no production dependency;
-   isolated tenant boundary;
-   least privilege;
-   audit everything.

------------------------------------------------------------------------

## 16.2 Threat Model

  Threat              Control
  ------------------- ----------------------
  Script abuse        Restricted execution
  Data leakage        Isolated data
  Resource abuse      Quota
  Credential misuse   IAM controls

------------------------------------------------------------------------

# 17. IAM Architecture

Identity layers:

    Developer

    Application

    Sandbox Credential

    Runtime Authorization

Support:

-   OAuth2;
-   mTLS;
-   API key;
-   FAPI profiles.

------------------------------------------------------------------------

# 18. Data Architecture

Recommended:

Schema-per-sandbox.

Future:

Dedicated database for premium partners.

------------------------------------------------------------------------

# 19. Kubernetes Architecture

``` mermaid
flowchart TB

Cluster[Sandbox Cluster]

A[Namespace A]

B[Namespace B]

RuntimeA[SmartMock A]

RuntimeB[SmartMock B]

DataA[Data A]

DataB[Data B]

Cluster --> A
Cluster --> B

A --> RuntimeA
A --> DataA

B --> RuntimeB
B --> DataB
```

------------------------------------------------------------------------

# 20. Non Functional Requirements

## Availability

Target:

Enterprise sandbox availability.

## Performance

Support:

-   multiple partner sandboxes;
-   concurrent API testing.

## Scalability

Scale independently:

-   control plane;
-   runtime;
-   data layer.

------------------------------------------------------------------------

# 21. Observability

Required:

-   API logs;
-   scenario logs;
-   webhook logs;
-   replay.

Metrics:

-   sandbox usage;
-   journey success;
-   integration failures.

------------------------------------------------------------------------

# 22. CI/CD Integration

Future:

    OpenAPI

    |

    Sandbox Provision

    |

    Automated Journey Test

    |

    Integration Report

------------------------------------------------------------------------

# 23. Architecture Decisions

## ADR-001

Decision:

SmartMock remains execution engine.

Reason:

Reuse existing investment.

------------------------------------------------------------------------

## ADR-002

Decision:

Namespace per sandbox.

Reason:

Balance isolation and cost.

------------------------------------------------------------------------

## ADR-003

Decision:

Journey-first design.

Reason:

Business integration success is objective.

------------------------------------------------------------------------

# 24. Delivery Roadmap

## Phase 0

SmartMock enterprise hardening.

Deliver:

-   security;
-   audit;
-   sandbox context.

------------------------------------------------------------------------

## Phase 1

SmartSandbox MVP.

Deliver:

-   Sandbox Manager;
-   isolation;
-   dynamic data;
-   scenarios;
-   webhook;
-   replay.

------------------------------------------------------------------------

## Phase 2

Advanced simulation.

Deliver:

-   external API virtualization;
-   partner OpenAPI import;
-   CI integration.

------------------------------------------------------------------------

## Phase 3

Intelligent Sandbox.

Deliver:

-   AI scenario generation;
-   autonomous testing agents.

------------------------------------------------------------------------

# 25. Risks

## Technical Risks

-   script security;
-   operational complexity;
-   scalability.

## Mitigation

-   restricted execution;
-   automation;
-   Kubernetes governance.

------------------------------------------------------------------------

# 26. Final Recommendation

Approve SmartSandbox as HSBC strategic capability.

The platform creates:

    HSBC Developer Portal

    +

    SmartMock Runtime

    +

    Open Banking Expertise

    +

    Business Simulation

    =

    Enterprise Integration Simulation Platform

This capability moves HSBC from providing API access to enabling partner
integration success.
