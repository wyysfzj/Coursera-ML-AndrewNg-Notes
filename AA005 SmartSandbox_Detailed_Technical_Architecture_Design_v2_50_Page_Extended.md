# SmartSandbox Detailed Technical Architecture Design

## Developer Portal Integration with SmartMock-based Open Banking Simulation Platform

Version 2.0\
Document Type: Enterprise Architecture Design Paper\
Audience: - Architecture Review Board - Enterprise Architects - Solution
Architects - Security Architects - Open Banking Engineering Teams -
Platform Engineering Teams

------------------------------------------------------------------------

# Document Purpose

This document defines the target architecture for SmartSandbox, an
HSBC-owned Open Banking integration simulation platform.

The objective is to evolve the existing SmartMock execution engine and
Developer Portal capability into a production-grade sandbox platform
that enables third parties to validate complete business journeys
instead of isolated API calls.

The strategic evolution:

    Traditional API Sandbox

    API Request
        |
    Mock Response


    SmartSandbox

    Business Journey
        |
    API Orchestration
        |
    State Management
        |
    Dynamic Data
        |
    Events/Webhooks
        |
    Integration Validation

------------------------------------------------------------------------

# 1. Executive Summary

## 1.1 Strategic Context

Open Banking ecosystems require external developers to integrate with:

-   Account Information APIs
-   Payment APIs
-   Consent APIs
-   Authentication flows
-   Event notification mechanisms

The integration challenge is no longer only API connectivity.

The real challenge is:

"Can a partner successfully complete the end-to-end customer and
business journey?"

Examples:

-   Consent creation
-   Customer authorisation
-   Account access
-   Payment initiation
-   Payment status tracking
-   Webhook processing

------------------------------------------------------------------------

# 1.2 Strategic Proposal

The proposal introduces SmartSandbox:

    HSBC Developer Portal

            +

    SmartMock Execution Engine

            +

    Sandbox Control Plane

            +

    Scenario Simulation Platform

The result is an enterprise integration simulation capability.

------------------------------------------------------------------------

# 1.3 Strategic Advantages

## Business

-   Faster partner onboarding
-   Reduced production integration failures
-   Better developer experience
-   Lower support cost
-   Faster regulatory change adoption

## Technology

-   HSBC-owned capability
-   Reduced vendor dependency
-   Reusable simulation platform
-   AI-ready foundation

------------------------------------------------------------------------

# 2. Current State Assessment

# 2.1 Developer Portal Capability

The new HSBC Developer Portal already provides:

-   API catalogue
-   API documentation
-   developer registration
-   application lifecycle
-   credential management
-   API subscription
-   API testing entry point

Therefore, replacing DAC developer portal capability is not the main
challenge.

------------------------------------------------------------------------

# 2.2 Open Banking Capability

Existing capability:

## HKMA Open Banking Phase 3

Capabilities:

-   API onboarding
-   API publication
-   partner enablement
-   OpenAPI integration

## PSD2 Experience

Existing experience:

-   Account Information Service
-   Payment Initiation Service
-   Consent lifecycle
-   Regulatory API delivery

This provides a strong foundation.

------------------------------------------------------------------------

# 3. SmartMock Technical Foundation

# 3.1 Current SmartMock Role

SmartMock is the execution runtime.

Responsibilities:

-   API simulation
-   dynamic endpoint management
-   request processing
-   response generation
-   business logic execution

------------------------------------------------------------------------

# 3.2 SmartMock Logical Architecture

``` mermaid
flowchart TB

Request[Incoming API Request]

Router[Dynamic API Router]

Executor[Execution Engine]

Groovy[Business Logic Engine]

Template[Template Manager]

Data[Mock Data Manager]

Validator[Schema Validation]

Response[API Response]

Request --> Router
Router --> Executor

Executor --> Groovy
Groovy --> Template
Groovy --> Data
Groovy --> Validator

Executor --> Response
```

------------------------------------------------------------------------

# 3.3 Current SmartMock Capabilities

## Contract Capability

-   OpenAPI import
-   Swagger support
-   API generation

## Runtime Capability

-   dynamic endpoint registration
-   runtime refresh
-   hot update

## Behaviour Capability

-   Groovy scripting
-   extension functions
-   custom response logic

## Integration Capability

-   proxy
-   downstream simulation

------------------------------------------------------------------------

# 4. Limitations of API Sandbox Model

Traditional sandbox:

    Request
     |
     |
    Response

Problem:

No understanding of:

-   business state
-   customer lifecycle
-   workflow
-   events

------------------------------------------------------------------------

# 5. SmartSandbox Vision

SmartSandbox changes the model:

    API Testing

            ↓

    Business Journey Simulation

            ↓

    Integration Readiness Validation

------------------------------------------------------------------------

# 6. Target Architecture

# 6.1 Logical Architecture

``` mermaid
flowchart TB

Developer[TPP Developer]

Portal[HSBC Developer Portal]

Control[SmartSandbox Control Plane]

Manager[Sandbox Manager]

Provision[Provisioning Service]

Runtime[SmartMock Sandbox Runtime]

Scenario[Scenario Engine]

Data[Dynamic Data Engine]

Webhook[Webhook Engine]

Replay[Observability and Replay]

Developer --> Portal

Portal --> Control

Control --> Manager

Manager --> Provision

Provision --> Runtime

Runtime --> Scenario
Runtime --> Data
Runtime --> Webhook
Runtime --> Replay
```

------------------------------------------------------------------------

# 7. Component Design

# 7.1 Sandbox Manager

Purpose:

Central lifecycle controller.

Responsibilities:

-   create sandbox
-   delete sandbox
-   suspend sandbox
-   clone sandbox
-   reset data
-   manage lifecycle

Example API:

    POST /sandbox

    GET /sandbox/{id}

    DELETE /sandbox/{id}

------------------------------------------------------------------------

# 7.2 Sandbox Runtime

Each sandbox contains:

-   SmartMock instance
-   configuration
-   scenario definitions
-   data store
-   webhook configuration

Isolation principle:

    Partner A

    Namespace A

    Runtime A


    Partner B

    Namespace B

    Runtime B

------------------------------------------------------------------------

# 8. Kubernetes Deployment Architecture

``` mermaid
flowchart TB

Cluster[Sandbox Kubernetes Cluster]

NS1[Namespace Partner A]

NS2[Namespace Partner B]

Runtime1[SmartMock Runtime]

Runtime2[SmartMock Runtime]

DB1[Partner A Data]

DB2[Partner B Data]

Cluster --> NS1
Cluster --> NS2

NS1 --> Runtime1
NS1 --> DB1

NS2 --> Runtime2
NS2 --> DB2
```

------------------------------------------------------------------------

# 9. Consent Journey Simulation

``` mermaid
sequenceDiagram

TPP->>Sandbox: Create Consent

Sandbox-->>TPP: Consent Created

TPP->>Sandbox: Authorise Consent

Sandbox-->>TPP: Consent Approved

TPP->>Sandbox: Get Account

Sandbox-->>TPP: Account Information

TPP->>Sandbox: Payment Request

Sandbox-->>TPP: Payment Accepted

Sandbox->>TPP: Payment Webhook
```

------------------------------------------------------------------------

# 10. Scenario Engine Design

The scenario engine provides business state simulation.

Supported scenarios:

## Positive

-   approved consent
-   successful payment
-   completed transaction

## Negative

-   rejected consent
-   payment failure
-   timeout

## Regulatory

-   consent expired
-   consent revoked
-   authentication failure

------------------------------------------------------------------------

# 11. State Machine Model

Example:

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

# 12. Dynamic Data Engine

Purpose:

Provide realistic business data.

Entities:

-   customer
-   account
-   transaction
-   consent
-   payment

Example:

``` yaml
customer:
  type: corporate
  segment: premium

account:
  currency: HKD
  balance: 100000

payment:
  status: pending
```

------------------------------------------------------------------------

# 13. Webhook Simulation Architecture

``` mermaid
flowchart LR

Event[Business Event]

Queue[Event Queue]

Emitter[Webhook Emitter]

Partner[TPP Endpoint]

Audit[Audit Store]

Event --> Queue

Queue --> Emitter

Emitter --> Partner

Emitter --> Audit
```

------------------------------------------------------------------------

# 14. Authoring Model

# 14.1 Standard Mode

Target:

Most developers.

Capabilities:

-   OpenAPI import
-   templates
-   rules
-   scenarios
-   data configuration

------------------------------------------------------------------------

# 14.2 Advanced Mode

Target:

Power users.

Capabilities:

-   custom business logic
-   Groovy extension
-   domain simulation

Security:

-   restricted execution
-   audit
-   approval
-   versioning

------------------------------------------------------------------------

# 15. Security Architecture

Security principles:

-   tenant isolation
-   least privilege
-   no production access
-   full auditability

------------------------------------------------------------------------

# 16. Threat Model

Threats:

## Malicious Script

Mitigation:

-   sandbox execution
-   restricted APIs
-   timeout

## Data Leakage

Mitigation:

-   isolated database
-   synthetic data

## Resource Abuse

Mitigation:

-   quota
-   rate limit
-   lifecycle control

------------------------------------------------------------------------

# 17. Identity and Access Management

Model:

    Developer

     |
    Application

     |
    Sandbox Credential

     |
    Runtime Authorization

Support:

-   OAuth
-   API Key
-   mTLS
-   FAPI profile

------------------------------------------------------------------------

# 18. Data Architecture

Recommended:

Schema-per-sandbox.

Future:

Dedicated database for premium partners.

------------------------------------------------------------------------

# 19. Observability Architecture

Collect:

-   API logs
-   scenario execution
-   webhook delivery
-   business events

Metrics:

-   sandbox usage
-   failed journeys
-   integration quality

------------------------------------------------------------------------

# 20. CI/CD Integration

Future capability:

Partner pipeline:

    OpenAPI Spec

         |

    Sandbox Provision

         |

    Automated Journey Test

         |

    Integration Report

------------------------------------------------------------------------

# 21. Market Comparison

## DigitalAPI/DAC

Strength:

-   Open Banking focus
-   mature sandbox

Limitation:

-   less HSBC-specific simulation

## WireMock Cloud

Strength:

-   powerful mocking

Limitation:

-   less business workflow focus

## Microcks

Strength:

-   open standard

Limitation:

-   requires platform construction

## Postman

Strength:

-   developer adoption

Limitation:

-   lightweight simulation

## ReadyAPI

Strength:

-   enterprise virtualization

Limitation:

-   tool-centric

------------------------------------------------------------------------

# 22. Architecture Decision Records

## ADR-001

Decision:

SmartMock remains execution engine.

Reason:

Existing capability and investment.

## ADR-002

Decision:

Namespace-per-sandbox.

Reason:

Security and operational balance.

## ADR-003

Decision:

Scenario-first simulation.

Reason:

Business journey is more important than endpoint response.

------------------------------------------------------------------------

# 23. Roadmap

# Phase 0

SmartMock Enterprise Hardening

Deliver:

-   security
-   audit
-   sandbox context

# Phase 1

SmartSandbox MVP

Deliver:

-   sandbox manager
-   isolated runtime
-   dynamic data
-   scenarios
-   webhook
-   replay

# Phase 2

Advanced Simulation

Deliver:

-   external API virtualization
-   third-party OpenAPI import
-   CI integration

# Phase 3

Intelligent Sandbox

Deliver:

-   AI scenario generation
-   agent-based testing
-   autonomous integration validation

------------------------------------------------------------------------

# 24. Operational Model

Responsibilities:

Platform Team:

-   infrastructure
-   runtime
-   security

API Teams:

-   contracts
-   scenarios

Partners:

-   testing

------------------------------------------------------------------------

# 25. Final Recommendation

SmartSandbox should become:

"HSBC-owned Open Banking Integration Simulation Platform"

The strategic value:

    Developer Portal

    +

    SmartMock

    +

    Open Banking Expertise

    +

    Scenario Simulation

    =

    Differentiated Partner Ecosystem Capability
