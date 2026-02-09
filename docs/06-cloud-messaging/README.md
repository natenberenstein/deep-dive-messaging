# Cloud Messaging

Every messaging system we have covered so far — Kafka, NATS, RabbitMQ — can be self-hosted. You provision servers, manage upgrades, tune configurations, handle failovers, and monitor cluster health yourself. For teams with deep infrastructure expertise and workloads that justify the operational investment, self-hosting is the right call. But for everyone else — teams that need to move fast, scale elastically, or avoid hiring a dedicated platform engineer just to keep the message bus running — cloud-managed messaging services shift that operational burden to the cloud provider. AWS, Google Cloud, and Azure each offer a portfolio of managed messaging services that trade customization and portability for reduced operational overhead and tighter integration with the rest of the cloud ecosystem. Understanding these services, their trade-offs, and when managed messaging makes sense is essential for any modern architecture decision.

---

## Concept Map

```mermaid
graph TB
    CM["Cloud-Managed<br/>Messaging"]

    AWS["AWS<br/>Services"]
    GCP["Google Cloud<br/>Services"]
    AZ["Azure<br/>Services"]

    SQS["Amazon SQS"]
    SNS["Amazon SNS"]
    EB["Amazon<br/>EventBridge"]
    MSK["Amazon MSK"]

    PS["Cloud<br/>Pub/Sub"]
    PSL["Pub/Sub<br/>Lite"]

    SB["Azure<br/>Service Bus"]
    EH["Azure<br/>Event Hubs"]
    EG["Azure<br/>Event Grid"]

    CM --> AWS
    CM --> GCP
    CM --> AZ

    AWS --> SQS
    AWS --> SNS
    AWS --> EB
    AWS --> MSK

    GCP --> PS
    GCP --> PSL

    AZ --> SB
    AZ --> EH
    AZ --> EG

    style CM fill:#1abc9c,stroke:#16a085,color:#fff
    style AWS fill:#1abc9c,stroke:#16a085,color:#fff
    style GCP fill:#1abc9c,stroke:#16a085,color:#fff
    style AZ fill=#1abc9c,stroke:#16a085,color:#fff

    style SQS fill:#48c9b0,stroke:#1abc9c,color:#fff
    style SNS fill:#48c9b0,stroke:#1abc9c,color:#fff
    style EB fill:#48c9b0,stroke:#1abc9c,color:#fff
    style MSK fill:#48c9b0,stroke:#1abc9c,color:#fff
    style PS fill:#48c9b0,stroke:#1abc9c,color:#fff
    style PSL fill:#48c9b0,stroke:#1abc9c,color:#fff
    style SB fill:#48c9b0,stroke:#1abc9c,color:#fff
    style EH fill:#48c9b0,stroke:#1abc9c,color:#fff
    style EG fill:#48c9b0,stroke:#1abc9c,color:#fff
```

---

## Pages in This Section

| # | Page | What You Will Learn |
|---|------|---------------------|
| 1 | [Cloud-Managed Messaging Services](./cloud-messaging-services.md) | The build-vs-buy trade-off for messaging infrastructure. AWS services (SQS, SNS, EventBridge, MSK), Google Cloud services (Pub/Sub, Pub/Sub Lite), and Azure services (Service Bus, Event Hubs, Event Grid). Cross-cloud comparison, vendor lock-in strategies, and common architectural patterns with managed services. |

---

## Suggested Reading Order

1. **Read [Cloud-Managed Messaging Services](./cloud-messaging-services.md)** to understand the full landscape of managed messaging across the three major cloud providers, how they compare to each other and to self-hosted alternatives, and when managed messaging is the right architectural choice.

2. **After completing this section**, revisit [Choosing a Messaging System](../04-patterns-and-comparisons/choosing-a-messaging-system.md) with cloud-managed options in mind — many of the same trade-offs (latency, throughput, ordering, durability) apply, but the operational calculus changes significantly when the cloud provider handles infrastructure.

---

## Prerequisites

This section assumes familiarity with the concepts from earlier sections:

- **[Messaging Foundations](../01-messaging-foundations/README.md)** — delivery semantics, messaging paradigms, broker architecture
- **[Apache Kafka](../02-apache-kafka/README.md)** — partitions, consumer groups, log-based storage (helpful for understanding MSK and Event Hubs)
- **[Patterns & Comparisons](../04-patterns-and-comparisons/README.md)** — event-driven patterns and system selection trade-offs

If you skipped the technology-specific sections, you can still follow along — we recap key concepts where needed — but understanding Kafka, NATS, and RabbitMQ will make the managed-vs-self-hosted comparison more concrete.

---

*Next up: [Cloud-Managed Messaging Services](./cloud-messaging-services.md)*
