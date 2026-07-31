# Executive Overview and Context

# Digital Experience Measurement Capability Initiative

## From MI Metric Discussion to Digital Platform + API Journey Measurement Architecture

## 1. Executive Interpretation

Based on the MI Metric discussion and attached materials, this
initiative should not be interpreted as a traditional reporting or
dashboard exercise.

The strategic objective is to establish a Digital Experience Measurement
capability that converts digital interaction data into customer journey
intelligence and continuous optimisation.

The evolution is:

    Digital Activity Measurement
            |
            v
    Customer Journey Understanding
            |
            v
    Experience Intelligence
            |
            v
    Continuous Product Optimisation

The capability should answer:

-   How are customers using digital platforms?
-   Are customers successfully completing their intended journeys?
-   Where do customers drop off?
-   Why do failures happen?
-   Which improvements create measurable customer value?

------------------------------------------------------------------------

## 2. Current Challenge

HSBC digital platforms generate large amounts of data:

-   Web interactions
-   Mobile interactions
-   API events
-   Backend service events
-   Transaction events
-   Partner integration events
-   Operational telemetry

However, these signals are usually separated by technology boundaries.

Traditional monitoring may show:

    Payment API availability: 99.9%
    Latency: within SLA
    Error rate: acceptable

But customer experience may show:

    Payment completion decreased
    Consent abandonment increased
    Customer satisfaction reduced

The missing capability is the connection:

    Technical Event
            |
            v
    Customer Journey Impact
            |
            v
    Business Outcome

------------------------------------------------------------------------

## 3. Strategic Insight

The organisation does not only need more data.

It needs a common measurement model that transforms digital interactions
into meaningful experience intelligence.

The target capability connects:

    Customer Intent

            |

    Business Journey

            |

    Digital Interaction

            |

    API Events

            |

    Experience Metrics

            |

    Optimisation Actions

------------------------------------------------------------------------

## 4. Why Open Banking Is the Ideal Pilot

Open Banking provides an excellent proof point because it is:

### Customer Journey Driven

Example OBIE Payment Journey:

    Payment Intent

            |

    Select Open Banking

            |

    Choose Bank Account

            |

    Authentication

            |

    Consent Approval

            |

    Payment Initiation

            |

    Payment Completion

### API Native

The journey involves:

-   Consent APIs
-   Account APIs
-   Payment APIs
-   ASPSP integrations
-   TPP interactions

This creates the bridge between:

    API Platform Engineering
    +
    Customer Experience Measurement

### Multi-dimensional Failure Model

A technical failure:

    ASPSP timeout

creates business impact:

    Customer abandons payment
    Payment completion decreases
    Trust decreases

------------------------------------------------------------------------

## 5. From MI Metrics to Experience Intelligence

Traditional MI focuses on:

-   Transaction volume
-   Usage statistics
-   Availability
-   Operational reporting

The target capability introduces:

-   Journey success rate
-   Completion rate
-   Drop-off analysis
-   Time-to-value
-   Customer effort measurement
-   Experience improvement tracking

------------------------------------------------------------------------

## 6. Measurement Hierarchy

### Layer 1 --- Technical Observability

Answers:

"Is the platform working?"

Measures:

-   API availability
-   Latency
-   Error rate
-   Infrastructure health

### Layer 2 --- Digital Behaviour Analytics

Answers:

"How are customers interacting?"

Measures:

-   Sessions
-   Clicks
-   User actions
-   Channel behaviour

### Layer 3 --- Journey Intelligence

Answers:

"Can customers achieve their goals?"

Measures:

-   Journey completion
-   Funnel progression
-   Drop-off points
-   Failure reasons

### Layer 4 --- Business Experience Intelligence

Answers:

"What should we improve?"

Measures:

-   Customer outcome
-   Product effectiveness
-   Business impact

------------------------------------------------------------------------

## 7. Core Architectural Gap

The current pattern:

    API Monitoring
    +
    Digital Analytics
    +
    Business Reporting

is insufficient.

The required capability is:

    API Event

            |

    Journey Context

            |

    Experience Metric

            |

    Business Decision

------------------------------------------------------------------------

## 8. API Events Are Signals, Not Insights

Raw API event:

``` json
{
 "endpoint":"/payments",
 "status":500
}
```

provides technical information only.

Journey-aware event:

``` json
{
 "event":"payment_failed",
 "journey":"open_banking_payment",
 "stage":"authentication",
 "failureReason":"ASPSP_TIMEOUT",
 "customerImpact":"journey_abandonment"
}
```

creates customer experience intelligence.

------------------------------------------------------------------------

## 9. Enterprise Vision

The long-term target is:

    Enterprise Digital Experience Intelligence Platform

Reusable across:

-   Open Banking
-   Retail Banking
-   CMB
-   CIB
-   Wealth
-   Lending

Common model:

    Customer Journey
            |
    Digital Events
            |
    Canonical Event Model
            |
    Experience Metrics
            |
    Analytics
            |
    Optimisation

------------------------------------------------------------------------

## 10. Conclusion

The MI Metric discussion represents an opportunity to move HSBC from
reactive reporting to proactive digital experience management.

The architecture proposal that follows defines:

**Digital Experience Measurement Capability --- Open Banking Pilot**

with the strategic positioning:

**Digital Platform + API Journey Measurement Architecture**

The objective is to transform API-level interaction events into
enterprise customer experience intelligence and establish a reusable
foundation for future digital journeys.
