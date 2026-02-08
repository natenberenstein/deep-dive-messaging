# Tools and Frameworks

> **A reference guide to the tooling ecosystem around Apache Kafka and NATS.** This page catalogs the CLIs, operators, UIs, stream processors, client libraries, and supporting tools you will encounter when building and operating messaging systems.

---

## How to Use This Guide

This is a reference page, not a tutorial. Use it to:

- **Discover tools** you did not know existed for a specific task (monitoring, schema management, CDC, etc.).
- **Compare options** when evaluating tooling for a new project.
- **Find official links** quickly — every entry links to the relevant repository or documentation.

Tools are organized by ecosystem (Kafka, NATS, cross-platform), then by category within each ecosystem.

---

## Apache Kafka Ecosystem

The Kafka ecosystem is the largest and most mature of any messaging system. The table below covers the tools you are most likely to encounter in production, from the official Apache project through the Confluent commercial ecosystem to community-driven alternatives.

| Tool | Category | Description |
|------|----------|-------------|
| [Kafka CLI Tools](https://kafka.apache.org/documentation/#quickstart) (`kafka-topics.sh`, `kafka-console-producer.sh`, `kafka-console-consumer.sh`, etc.) | CLI | The built-in command-line tools that ship with Apache Kafka. Essential for topic management, debugging, and ad-hoc message production/consumption. Every Kafka operator should know these cold. |
| [Kafka Connect](https://kafka.apache.org/documentation/#connect) | Data Integration | A framework for streaming data between Kafka and external systems (databases, search indexes, file systems, cloud storage) using pre-built or custom connectors. Handles offset tracking, serialization, and fault tolerance so you do not have to. |
| [Kafka Streams](https://kafka.apache.org/documentation/streams/) | Stream Processing | A Java/Scala library for building stateful stream processing applications directly on Kafka — no separate cluster required. Supports KStreams (record streams), KTables (changelog streams), joins, windowed aggregations, and exactly-once processing. |
| [ksqlDB](https://ksqldb.io/) | Stream Processing | A SQL engine for stream processing on Kafka. Lets you create materialized views, streaming ETL pipelines, and event-driven microservices using familiar SQL syntax. Developed by Confluent; requires the Confluent platform. |
| [Confluent Schema Registry](https://docs.confluent.io/platform/current/schema-registry/index.html) | Schema Management | A central registry for Avro, Protobuf, and JSON schemas used in Kafka messages. Enforces schema compatibility rules (backward, forward, full) to prevent producers from breaking consumers. Effectively required for any serious Kafka deployment. |
| [Confluent Platform](https://www.confluent.io/product/confluent-platform/) | Platform | The commercial distribution of Kafka from Confluent. Adds Schema Registry, ksqlDB, a REST proxy, RBAC, Audit Logs, Tiered Storage, and multi-region replication on top of open-source Kafka. Available self-managed or as [Confluent Cloud](https://www.confluent.io/confluent-cloud/). |
| [Strimzi](https://strimzi.io/) | Kubernetes Operator | A CNCF sandbox project that provides Kubernetes operators for running Apache Kafka, Kafka Connect, Kafka MirrorMaker, and Schema Registry on Kubernetes. The most widely adopted way to run Kafka on K8s. |
| [Redpanda](https://redpanda.com/) | Kafka-Compatible Broker | A Kafka-compatible streaming platform written in C++ (not JVM-based). Drop-in replacement for Kafka that eliminates ZooKeeper, reduces tail latency, and simplifies operations. Uses the Kafka protocol, so existing clients and tools work unmodified. |
| [Kafdrop](https://github.com/obsidiandynamics/kafdrop) | UI / Monitoring | A lightweight web UI for viewing Kafka topics, consumer groups, messages, and broker metadata. Simple to deploy (single Docker container) and useful for development and debugging. |
| [AKHQ](https://akhq.io/) (formerly KafkaHQ) | UI / Monitoring | A more full-featured Kafka UI than Kafdrop — supports topic management, consumer group monitoring, Schema Registry browsing, Kafka Connect management, and ACL viewing. Good for teams that need a single pane of glass. |
| [Conduktor](https://www.conduktor.io/) | UI / Platform | A commercial Kafka management platform with a desktop app and web console. Provides topic browsing, consumer lag monitoring, schema management, Kafka Connect management, and data governance features. Free tier available for individual use. |
| [Debezium](https://debezium.io/) | Change Data Capture (CDC) | An open-source CDC platform built on Kafka Connect. Captures row-level changes from databases (PostgreSQL, MySQL, MongoDB, SQL Server, Oracle, etc.) and streams them into Kafka topics. The standard tool for database-to-Kafka integration. |
| [MirrorMaker 2](https://kafka.apache.org/documentation/#georeplication) | Replication | The built-in tool for replicating data across Kafka clusters, typically used for geo-replication and disaster recovery. Built on Kafka Connect, it replicates topics, consumer group offsets, and ACLs. Replaces the original MirrorMaker with a more robust architecture. |
| [Cruise Control](https://github.com/linkedin/cruise-control) | Operations | LinkedIn's open-source tool for Kafka cluster balancing and self-healing. Automatically rebalances partition replicas across brokers to maintain even load distribution, and detects/fixes broker failures. |

---

## NATS Ecosystem

The NATS ecosystem is deliberately smaller and more focused than Kafka's. NATS ships as a single binary that includes core messaging, JetStream persistence, key-value store, and object store — so fewer external tools are needed. The tools below cover what you will need for development, operations, and production deployment.

| Tool | Category | Description |
|------|----------|-------------|
| [nats CLI](https://github.com/nats-io/natscli) | CLI | The official NATS command-line tool. Manages servers, streams, consumers, key-value stores, and object stores. Also supports benchmarking (`nats bench`), message pub/sub, request-reply testing, and account management. The single most important tool in the NATS ecosystem. |
| [nats-server](https://github.com/nats-io/nats-server) | Server | The NATS server itself — a single, statically compiled Go binary (~20 MB) that includes core NATS, JetStream, and all clustering/gateway/leaf-node functionality. Zero external dependencies. Starts in milliseconds. |
| [nats-top](https://github.com/nats-io/nats-top) | Monitoring | A `top`-like tool for monitoring NATS server connections, subscriptions, and message rates in real time. Useful for quick operational visibility without setting up a full monitoring stack. |
| [NATS Helm Charts](https://github.com/nats-io/k8s/tree/main/helm/charts/nats) | Kubernetes | Official Helm charts for deploying NATS and NATS with JetStream on Kubernetes. Supports clustering, TLS, authentication, and resource configuration out of the box. |
| [NATS Kubernetes Operator](https://github.com/nats-io/nats-operator) | Kubernetes | A Kubernetes operator for managing NATS clusters as custom resources. Handles scaling, upgrades, and configuration changes declaratively. Note: for most deployments, the Helm chart is the recommended approach; the operator is suited for more advanced lifecycle management. |
| [nsc (NATS Security Credentials)](https://github.com/nats-io/nsc) | Security | A CLI tool for managing NATS accounts, users, and authorization using NKeys and JWTs. Essential for setting up NATS's decentralized authentication and multi-tenancy model, where account JWTs can be distributed without server restarts. |
| [nats-surveyor](https://github.com/nats-io/nats-surveyor) | Monitoring | A Prometheus exporter for NATS metrics. Collects server stats, JetStream info, and connection data, then exposes them as Prometheus metrics. Pair with Grafana dashboards (included in the repo) for production monitoring. |
| [Benthos / Bento](https://www.benthos.dev/) | Stream Processing | A declarative stream processor that connects to NATS (core and JetStream) as both input and output. Define processing pipelines in YAML — supports mapping, filtering, enrichment, batching, and routing. Also connects to Kafka, HTTP, databases, and dozens of other systems, making it useful as a bridge. |

### NATS Client Libraries

NATS maintains official client libraries across all major languages. Each library supports core NATS, JetStream, key-value, and object store APIs.

| Library | Language | Repository |
|---------|----------|------------|
| nats.go | Go | [github.com/nats-io/nats.go](https://github.com/nats-io/nats.go) |
| nats.java | Java | [github.com/nats-io/nats.java](https://github.com/nats-io/nats.java) |
| nats.py | Python | [github.com/nats-io/nats.py](https://github.com/nats-io/nats.py) |
| nats.js | JavaScript / TypeScript | [github.com/nats-io/nats.js](https://github.com/nats-io/nats.js) |
| nats.net | .NET (C#) | [github.com/nats-io/nats.net](https://github.com/nats-io/nats.net) |
| nats.rs | Rust | [github.com/nats-io/nats.rs](https://github.com/nats-io/nats.rs) |

> **All NATS client libraries follow the same API conventions.** If you know how to use one, you can pick up another quickly. The [NATS by Example](https://natsbyexample.com/) site shows the same patterns implemented in multiple languages side by side.

---

## Cross-Platform and General Tools

These tools are not specific to Kafka or NATS but are commonly used alongside them in messaging-oriented architectures.

| Tool | Category | Description |
|------|----------|-------------|
| [Prometheus](https://prometheus.io/) + [Grafana](https://grafana.com/) | Monitoring | The standard open-source monitoring stack. Both Kafka (via JMX Exporter or Confluent Metrics) and NATS (via nats-surveyor) export Prometheus-compatible metrics. Grafana provides dashboards and alerting. Community-maintained dashboards exist for both Kafka and NATS. |
| [AsyncAPI](https://www.asyncapi.com/) | API Specification | An open standard for defining event-driven APIs — the equivalent of OpenAPI/Swagger for messaging. Supports Kafka, NATS, AMQP, WebSocket, and other protocols. Use it to document your message channels, payload schemas, and broker bindings. Enables code generation for producers, consumers, and documentation. |
| [CloudEvents](https://cloudevents.io/) | Event Format Standard | A CNCF specification for describing event data in a common, interoperable format. Defines a minimal set of metadata attributes (source, type, id, time) that every event should carry. Supported by Kafka, NATS, and most cloud providers' event services. Reduces coupling between producers and consumers. |
| [Testcontainers](https://testcontainers.com/) | Integration Testing | A library (available for Java, Go, Python, Node.js, .NET, Rust) that spins up real Docker containers for integration tests. Provides first-class modules for [Kafka](https://testcontainers.com/modules/kafka/), [Redpanda](https://testcontainers.com/modules/redpanda/), and NATS — so you can write tests against real brokers instead of mocks. |
| [Docker Compose](https://docs.docker.com/compose/) | Local Development | Docker Compose files are the standard way to run Kafka or NATS locally for development. See [Confluent's cp-all-in-one](https://github.com/confluentinc/cp-all-in-one) for a full Kafka stack, or the [NATS Docker image](https://hub.docker.com/_/nats) for a single-container NATS server with JetStream enabled via `--js` flag. |
| [kcat (formerly kafkacat)](https://github.com/edenhill/kcat) | CLI | A versatile command-line Kafka producer, consumer, and metadata inspection tool written in C. Faster and more scriptable than the Java-based Kafka CLI tools. Useful for piping data in and out of Kafka in shell scripts and CI pipelines. |
| [Buf](https://buf.build/) | Schema Management | A modern toolchain for working with Protocol Buffers. Useful when Protobuf is your message serialization format — provides linting, breaking change detection, and a schema registry that complements or replaces Confluent Schema Registry for Protobuf schemas. |

---

## Client Libraries Comparison

The table below compares the primary Kafka and NATS client libraries across major languages. All links point to the official or most widely adopted library for each combination.

| Language | Kafka Client | NATS Client |
|----------|-------------|-------------|
| **Go** | [confluent-kafka-go](https://github.com/confluentinc/confluent-kafka-go) — CGo wrapper around librdkafka. High performance. Also: [segmentio/kafka-go](https://github.com/segmentio/kafka-go) — pure Go, no CGo dependency. | [nats.go](https://github.com/nats-io/nats.go) — official, pure Go. Full JetStream, KV, and object store support. |
| **Java** | [Apache Kafka Client](https://mvnrepository.com/artifact/org.apache.kafka/kafka-clients) — the official Java client, maintained as part of the Apache Kafka project. The reference implementation that all other clients are measured against. | [nats.java](https://github.com/nats-io/nats.java) — official. Supports core NATS, JetStream, KV, and object store. |
| **Python** | [confluent-kafka-python](https://github.com/confluentinc/confluent-kafka-python) — Python wrapper around librdkafka. Recommended for production. Also: [aiokafka](https://github.com/aio-libs/aiokafka) — asyncio-native, pure Python. | [nats.py](https://github.com/nats-io/nats.py) — official, asyncio-native. Full JetStream support. |
| **JavaScript / TypeScript** | [KafkaJS](https://github.com/tulios/kafkajs) — pure JavaScript, no native dependencies. The most popular Kafka client for Node.js. Also: [confluent-kafka-javascript](https://github.com/confluentinc/confluent-kafka-javascript) — Confluent's official Node.js client wrapping librdkafka. | [nats.js](https://github.com/nats-io/nats.js) — official. Works in Node.js, Deno, and browsers (via WebSocket). Full JetStream support. |
| **Rust** | [rust-rdkafka](https://github.com/fede1024/rust-rdkafka) — Rust bindings for librdkafka. The most mature Kafka client for Rust. | [nats.rs](https://github.com/nats-io/nats.rs) — official, async Rust. Built on Tokio. Full JetStream, KV, and object store support. |
| **.NET (C#)** | [confluent-kafka-dotnet](https://github.com/confluentinc/confluent-kafka-dotnet) — .NET wrapper around librdkafka. The standard Kafka client for .NET. | [nats.net](https://github.com/nats-io/nats.net) — official. Modern .NET (v2 rewrite). Full JetStream, KV, and object store support. |

### Notes on Kafka Client Strategy

Most production Kafka clients across languages are wrappers around **[librdkafka](https://github.com/confluentinc/librdkafka)**, a high-performance C library maintained by Confluent. This means:

- **Performance is consistent** across languages, since the heavy lifting happens in C.
- **Feature availability** tends to land in librdkafka first, then propagate to language wrappers.
- **The trade-off** is a CGo/FFI dependency, which complicates builds and cross-compilation. Pure-language alternatives (kafka-go, KafkaJS, aiokafka) avoid this at the cost of some features or performance.

### Notes on NATS Client Strategy

All NATS clients are **pure implementations** in their respective languages — no shared C library. This means:

- **No native dependencies** — simpler builds, easier cross-compilation, and no CGo/FFI issues.
- **Consistent API design** across all languages, following the same patterns and naming conventions.
- **The trade-off** is that each client must independently implement the full NATS protocol, so new features may arrive at slightly different times across languages.

---

*This tools reference is maintained as part of the [deep-dive-messaging](../docs/01-messaging-foundations/README.md) knowledge base. Suggest additions by opening a pull request.*
