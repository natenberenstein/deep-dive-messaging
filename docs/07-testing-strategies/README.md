# Testing Strategies

> **Testing event-driven systems is fundamentally harder than testing synchronous services.** In a REST-based architecture, you call an endpoint and assert on the response — the feedback loop is immediate and deterministic. In an event-driven architecture, the producer does not know who consumes its events, consumers process messages asynchronously (possibly minutes later), ordering is not guaranteed across partitions or subjects, and the same message may be delivered more than once. You cannot simply assert on a return value — you must wait for side effects to propagate through a distributed, eventually consistent pipeline. This section covers the strategies, tools, and patterns that make event-driven systems testable at every layer.

---

## Concept Map

```mermaid
graph TB
    TS["Testing<br/>Strategies"]

    UNIT["Unit<br/>Testing"]
    INT["Integration<br/>Testing"]
    CONTRACT["Contract<br/>Testing"]

    SAGA["Saga<br/>Testing"]
    CHAOS["Chaos<br/>Engineering"]
    PERF["Performance<br/>Testing"]

    SERIAL["Serialization &<br/>Routing"]
    IDEMP["Idempotency<br/>Handlers"]
    TC["Testcontainers"]
    ASYNC["Async<br/>Assertions"]

    PACT["Pact &<br/>AsyncAPI"]
    SCHEMA["Schema<br/>Compatibility"]

    TS --> UNIT
    TS --> INT
    TS --> CONTRACT
    TS --> SAGA
    TS --> CHAOS
    TS --> PERF

    UNIT --> SERIAL
    UNIT --> IDEMP
    INT --> TC
    INT --> ASYNC
    CONTRACT --> PACT
    CONTRACT --> SCHEMA

    style TS fill:#e84393,stroke:#c2185b,color:#fff
    style UNIT fill:#e84393,stroke:#c2185b,color:#fff
    style INT fill=#e84393,stroke:#c2185b,color:#fff
    style CONTRACT fill:#e84393,stroke:#c2185b,color:#fff

    style SAGA fill:#f078a0,stroke:#e84393,color:#fff
    style CHAOS fill:#f078a0,stroke:#e84393,color:#fff
    style PERF fill:#f078a0,stroke:#e84393,color:#fff
    style SERIAL fill:#f078a0,stroke:#e84393,color:#fff
    style IDEMP fill:#f078a0,stroke:#e84393,color:#fff
    style TC fill:#f078a0,stroke:#e84393,color:#fff
    style ASYNC fill:#f078a0,stroke:#e84393,color:#fff
    style PACT fill:#f078a0,stroke:#e84393,color:#fff
    style SCHEMA fill:#f078a0,stroke:#e84393,color:#fff
```

---

## Pages in This Section

| # | Page | What You Will Learn |
|---|------|---------------------|
| 1 | [Testing Event-Driven Systems](./testing-event-driven-systems.md) | Why testing async systems is uniquely hard, unit and integration testing strategies for producers and consumers, contract testing with Pact and AsyncAPI, saga testing, chaos engineering for messaging, performance and load testing, anti-patterns to avoid, and a practical layered testing strategy. |

---

## Suggested Reading Order

1. **Read [Testing Event-Driven Systems](./testing-event-driven-systems.md)** from start to finish. The page is structured to build from fast, isolated unit tests up through integration, contract, chaos, and performance testing — mirroring the layered strategy you should adopt in your own projects.

2. **After completing this section**, revisit the patterns from [Event-Driven Microservices](../04-patterns-and-comparisons/event-driven-microservices.md) with a testing lens — every pattern discussed there (Sagas, Outbox, CQRS) has corresponding test strategies covered here.

---

## Prerequisites

This section assumes familiarity with the concepts from the previous sections:

- **[Messaging Foundations](../01-messaging-foundations/README.md)** — delivery semantics, acknowledgment patterns, idempotency
- **[Apache Kafka](../02-apache-kafka/README.md)** — partitions, consumer groups, offsets (needed for Kafka-specific test examples)
- **[NATS](../03-nats/README.md)** — subjects, JetStream consumers (needed for NATS-specific test examples)
- **[Patterns & Comparisons](../04-patterns-and-comparisons/README.md)** — Sagas, Outbox, CQRS, schema evolution (tested in this section)

---

*Next up: [Testing Event-Driven Systems](./testing-event-driven-systems.md)*
