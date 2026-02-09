# Testing Event-Driven Systems

> **TL;DR** — Event-driven systems are inherently harder to test than synchronous services because of asynchronous behavior, non-deterministic ordering, eventual consistency, and distributed state. A robust testing strategy requires multiple layers: fast unit tests for serialization, routing, and business logic; integration tests with real brokers (via Testcontainers or embedded brokers) for end-to-end message flow; contract tests (Pact, AsyncAPI, Schema Registry) for schema compatibility; saga-specific tests for compensation paths; chaos engineering for failure resilience; and performance tests for capacity planning. This page covers each layer with concrete tools, code examples, and anti-patterns to avoid.

---

## Table of Contents

- [Why This Matters](#why-this-matters)
- [Why Testing Event-Driven Systems Is Hard](#why-testing-event-driven-systems-is-hard)
- [Unit Testing Producers and Consumers](#unit-testing-producers-and-consumers)
- [Integration Testing with Real Brokers](#integration-testing-with-real-brokers)
- [Contract Testing for Event Schemas](#contract-testing-for-event-schemas)
- [Testing Saga and Choreography Patterns](#testing-saga-and-choreography-patterns)
- [Chaos Engineering for Messaging](#chaos-engineering-for-messaging)
- [Performance and Load Testing](#performance-and-load-testing)
- [Testing Anti-Patterns](#testing-anti-patterns)
- [A Practical Testing Strategy](#a-practical-testing-strategy)
- [Key Takeaways](#key-takeaways)
- [References](#references)

---

## Why This Matters

In a synchronous REST-based system, testing follows a predictable pattern: send a request, get a response, assert on the result. The feedback loop is immediate. Failures are visible. State changes are contained within the boundaries of a single request-response cycle.

Event-driven systems break all of those assumptions. A producer publishes an event and has no idea who will consume it, when they will consume it, or whether they will succeed. A consumer might process messages out of order, receive duplicates, or fail halfway through a multi-step workflow. The "result" of a single event might not be observable for seconds or minutes, and it might manifest as side effects in a completely different service's database.

If you test event-driven systems the same way you test REST APIs, you will have tests that are either flaky (because they race against async processing), incomplete (because they only cover the happy path), or misleading (because they mock away the very behavior that fails in production). This page gives you the strategies and tools to test these systems properly.

---

## Why Testing Event-Driven Systems Is Hard

Five characteristics make event-driven systems uniquely challenging to test:

**Asynchronous behavior.** The producer and consumer do not execute in the same call stack. There is no return value to assert on. You must wait for a side effect — a database write, a downstream message, a state change — and you do not know exactly when it will happen.

**Temporal coupling.** Tests may pass or fail depending on timing. A consumer that is slightly slow might not have processed the event by the time your assertion runs. This leads to the most dreaded category of test failures: intermittent flakes that pass locally but fail in CI.

**Non-deterministic ordering.** Kafka guarantees ordering within a partition but not across partitions. NATS Core provides no ordering guarantees at all. RabbitMQ delivers messages in FIFO order per queue, but competing consumers break that. Your tests must account for messages arriving in any order.

**Eventual consistency.** After publishing an event, the system is temporarily inconsistent. A query against the consumer's database immediately after publishing will likely return stale data. Tests must either wait for consistency or explicitly account for the consistency window.

**Distributed state.** The "correct state" of the system is spread across multiple services, each with its own database. Verifying end-to-end correctness requires inspecting state in multiple places — and those states converge asynchronously.

### The Testing Pyramid for Event-Driven Systems

The classic testing pyramid (many unit tests, fewer integration tests, fewest end-to-end tests) still applies, but the shape and content of each layer changes for messaging systems.

```mermaid
graph TB
    E2E["End-to-End / Chaos Tests<br/>(few, slow, high confidence)"]
    PERF["Performance & Load Tests<br/>(periodic, capacity planning)"]
    CONTRACT["Contract Tests<br/>(schema compatibility, Pact, AsyncAPI)"]
    INT["Integration Tests<br/>(Testcontainers, real brokers, async assertions)"]
    UNIT["Unit Tests<br/>(serialization, routing, business logic, idempotency)"]

    E2E --- PERF
    PERF --- CONTRACT
    CONTRACT --- INT
    INT --- UNIT

    style E2E fill:#e84393,stroke:#c2185b,color:#fff
    style PERF fill:#e84393,stroke:#c2185b,color:#fff
    style CONTRACT fill:#f078a0,stroke:#e84393,color:#fff
    style INT fill:#f078a0,stroke:#e84393,color:#fff
    style UNIT fill:#f8bbd0,stroke:#f078a0,color:#fff
```

At the base, **unit tests** verify that your serialization, routing logic, business logic, and idempotency handlers work correctly in isolation — no broker needed, fast execution. Above that, **integration tests** spin up real brokers to verify the end-to-end message flow: produce, consume, and verify side effects. **Contract tests** ensure that schema changes do not break the producer-consumer agreement. At the top, **chaos and end-to-end tests** validate resilience under failure conditions. **Performance tests** run periodically to catch throughput regressions and inform capacity planning.

---

## Unit Testing Producers and Consumers

Unit tests are the foundation. They run in milliseconds, require no infrastructure, and catch the majority of logic bugs. In event-driven systems, four areas deserve dedicated unit test coverage.

### Testing Serialization and Deserialization

Events cross process boundaries as bytes. Serialization bugs — a missing field, a renamed key, an incompatible type change — are among the most common causes of production failures in messaging systems.

```pseudocode
// Test: OrderCreated event serializes to the expected Avro/JSON format
function test_order_created_serialization():
    event = OrderCreated(
        order_id = "ord-123",
        customer_id = "cust-456",
        total_amount = 99.95,
        currency = "USD",
        created_at = "2025-01-15T10:30:00Z"
    )

    bytes = serialize(event, schema="order_created_v1")
    deserialized = deserialize(bytes, schema="order_created_v1")

    assert deserialized.order_id == "ord-123"
    assert deserialized.total_amount == 99.95
    assert deserialized.currency == "USD"

// Test: Unknown fields are tolerated (forward compatibility)
function test_deserialization_ignores_unknown_fields():
    json_with_extra_field = '{"order_id": "ord-123", "new_field": "unexpected"}'
    event = deserialize(json_with_extra_field, schema="order_created_v1")

    assert event.order_id == "ord-123"
    // new_field is silently ignored, no exception thrown

// Test: Missing optional fields get default values
function test_deserialization_defaults_optional_fields():
    json_missing_optional = '{"order_id": "ord-123", "customer_id": "cust-456"}'
    event = deserialize(json_missing_optional, schema="order_created_v1")

    assert event.currency == "USD"  // default value from schema
```

### Testing Routing Logic

Producers often determine the target topic, subject, or exchange based on the event type or content. A routing bug means messages end up in the wrong place — silently.

```pseudocode
// Test: Events are routed to the correct topic based on type
function test_routing_by_event_type():
    router = EventRouter(config=routing_config)

    assert router.resolve_topic(OrderCreated(...)) == "orders.created"
    assert router.resolve_topic(OrderShipped(...)) == "orders.shipped"
    assert router.resolve_topic(PaymentFailed(...)) == "payments.failed"

// Test: Kafka partition key is derived correctly
function test_partition_key_from_order_id():
    producer = OrderEventProducer(broker=mock_broker)
    event = OrderCreated(order_id="ord-789", ...)

    producer.publish(event)

    assert mock_broker.last_published_key == "ord-789"
    // Ensures all events for the same order go to the same partition
```

### Testing Consumer Business Logic in Isolation

The most important unit test: can your consumer process a message correctly when given valid input? Mock the broker entirely and test the handler function directly.

```pseudocode
// Test: Consumer handler processes an OrderCreated event correctly
function test_order_consumer_creates_shipment():
    db = InMemoryDatabase()
    handler = ShipmentHandler(database=db)

    event = OrderCreated(
        order_id = "ord-123",
        items = [Item("SKU-A", quantity=2), Item("SKU-B", quantity=1)],
        shipping_address = "123 Main St"
    )

    handler.handle(event)

    shipment = db.find_shipment_by_order("ord-123")
    assert shipment is not null
    assert shipment.status == "PENDING"
    assert shipment.item_count == 3
    assert shipment.address == "123 Main St"
```

### Testing Idempotency Handlers

At-least-once delivery means your consumer will see duplicates. Idempotency is not optional — and it must be tested.

```pseudocode
// Test: Processing the same event twice produces the same result
function test_idempotent_processing():
    db = InMemoryDatabase()
    handler = PaymentHandler(database=db, inbox=InMemoryInbox())

    event = PaymentRequested(
        message_id = "msg-001",
        order_id = "ord-123",
        amount = 50.00
    )

    handler.handle(event)  // First processing
    handler.handle(event)  // Duplicate delivery

    payments = db.find_payments_by_order("ord-123")
    assert len(payments) == 1  // Not 2
    assert payments[0].amount == 50.00

// Test: Different message IDs are processed independently
function test_different_messages_processed_independently():
    db = InMemoryDatabase()
    handler = PaymentHandler(database=db, inbox=InMemoryInbox())

    handler.handle(PaymentRequested(message_id="msg-001", order_id="ord-123", amount=50.00))
    handler.handle(PaymentRequested(message_id="msg-002", order_id="ord-456", amount=75.00))

    assert len(db.find_all_payments()) == 2
```

### Kafka Producer Unit Test Example

```pseudocode
// Using a mock Kafka producer (e.g., MockProducer in kafka-clients)
function test_kafka_producer_sends_to_correct_topic():
    mock_producer = MockKafkaProducer(
        key_serializer = StringSerializer,
        value_serializer = AvroSerializer(schema_registry=mock_registry)
    )
    service = OrderService(producer=mock_producer)

    service.place_order(order_id="ord-123", customer_id="cust-456", amount=99.95)

    assert mock_producer.history().size() == 1
    record = mock_producer.history()[0]
    assert record.topic == "orders.created"
    assert record.key == "ord-123"
    assert record.value.order_id == "ord-123"
    assert record.value.total_amount == 99.95
```

### Consumer Unit Test Example

```pseudocode
// Testing consumer deserialization and handler invocation
function test_kafka_consumer_deserializes_and_delegates():
    mock_handler = MockShipmentHandler()
    consumer = OrderEventConsumer(handler=mock_handler)

    raw_record = ConsumerRecord(
        topic = "orders.created",
        key = "ord-123",
        value = avro_encode(OrderCreated(order_id="ord-123", items=[...]))
    )

    consumer.on_message(raw_record)

    assert mock_handler.invocations.size() == 1
    assert mock_handler.invocations[0].order_id == "ord-123"
```

---

## Integration Testing with Real Brokers

Unit tests with mocks verify logic but cannot catch wiring issues: misconfigured serializers, wrong topic names, incompatible consumer group settings, or timing-dependent bugs. Integration tests spin up real brokers to verify the full produce-consume-verify cycle.

### Testcontainers

Testcontainers is the gold standard for integration testing with real infrastructure. It manages Docker containers as part of the test lifecycle — start before the test, tear down after.

```java
// Java/JUnit5: Kafka integration test with Testcontainers
@Testcontainers
class OrderEventIntegrationTest {

    @Container
    static KafkaContainer kafka = new KafkaContainer(
        DockerImageName.parse("confluentinc/cp-kafka:7.5.0")
    );

    @Test
    void orderCreatedEventFlowsToShipmentService() {
        // Configure producer with Testcontainer's bootstrap servers
        Properties producerProps = new Properties();
        producerProps.put("bootstrap.servers", kafka.getBootstrapServers());
        producerProps.put("key.serializer", StringSerializer.class);
        producerProps.put("value.serializer", JsonSerializer.class);

        // Produce an event
        try (KafkaProducer<String, OrderCreated> producer = new KafkaProducer<>(producerProps)) {
            OrderCreated event = new OrderCreated("ord-123", "cust-456", 99.95);
            producer.send(new ProducerRecord<>("orders.created", "ord-123", event)).get();
        }

        // Configure consumer
        Properties consumerProps = new Properties();
        consumerProps.put("bootstrap.servers", kafka.getBootstrapServers());
        consumerProps.put("group.id", "test-shipment-group");
        consumerProps.put("auto.offset.reset", "earliest");

        // Consume and verify
        try (KafkaConsumer<String, OrderCreated> consumer = new KafkaConsumer<>(consumerProps)) {
            consumer.subscribe(List.of("orders.created"));
            ConsumerRecords<String, OrderCreated> records = consumer.poll(Duration.ofSeconds(10));

            assertThat(records.count()).isEqualTo(1);
            assertThat(records.iterator().next().value().getOrderId()).isEqualTo("ord-123");
        }
    }
}
```

```java
// RabbitMQ with Testcontainers
@Container
static RabbitMQContainer rabbit = new RabbitMQContainer(
    DockerImageName.parse("rabbitmq:3.12-management")
);

// NATS with Testcontainers
@Container
static GenericContainer<?> nats = new GenericContainer<>(
    DockerImageName.parse("nats:2.10")
).withCommand("--jetstream").withExposedPorts(4222);
```

### Embedded Brokers

For faster feedback in JVM projects, embedded brokers run in-process without Docker.

```java
// Spring Kafka: EmbeddedKafka
@EmbeddedKafka(partitions = 1, topics = {"orders.created"})
@SpringBootTest
class OrderEventEmbeddedTest {

    @Autowired
    private KafkaTemplate<String, OrderCreated> kafkaTemplate;

    @Autowired
    private ShipmentRepository shipmentRepository;

    @Test
    void orderCreatedTriggersShipmentCreation() {
        kafkaTemplate.send("orders.created", "ord-123",
            new OrderCreated("ord-123", "cust-456", 99.95));

        // Wait for async consumer processing
        await().atMost(Duration.ofSeconds(5)).untilAsserted(() -> {
            Shipment shipment = shipmentRepository.findByOrderId("ord-123");
            assertThat(shipment).isNotNull();
            assertThat(shipment.getStatus()).isEqualTo("PENDING");
        });
    }
}
```

### Docker Compose Test Environments

For multi-broker or multi-service integration tests, Docker Compose provides a declarative environment.

```yaml
# docker-compose.test.yml
services:
  kafka:
    image: confluentinc/cp-kafka:7.5.0
    environment:
      KAFKA_NODE_ID: 1
      KAFKA_PROCESS_ROLES: broker,controller
      KAFKA_LISTENERS: PLAINTEXT://0.0.0.0:9092,CONTROLLER://0.0.0.0:9093
      KAFKA_CONTROLLER_QUORUM_VOTERS: 1@kafka:9093
      CLUSTER_ID: "test-cluster-001"

  schema-registry:
    image: confluentinc/cp-schema-registry:7.5.0
    depends_on: [kafka]
    environment:
      SCHEMA_REGISTRY_KAFKASTORE_BOOTSTRAP_SERVERS: kafka:9092

  nats:
    image: nats:2.10
    command: ["--jetstream", "--store_dir=/data"]

  rabbitmq:
    image: rabbitmq:3.12-management
    ports:
      - "5672:5672"
      - "15672:15672"
```

### End-to-End Message Flow Verification

The pattern for integration tests is always the same: **produce, wait, verify side effects**.

```mermaid
sequenceDiagram
    participant Test as Test Runner
    participant P as Producer
    participant B as Broker<br/>(Testcontainer)
    participant C as Consumer
    participant DB as Database

    Test->>P: Trigger event production
    P->>B: Publish OrderCreated
    B->>C: Deliver to consumer
    C->>DB: Write shipment record
    Test->>DB: Poll until shipment exists
    DB-->>Test: Shipment found
    Test->>Test: Assert shipment state

    Note over Test,DB: The polling step is critical —<br/>never use a fixed sleep()
```

### Handling Async Assertions

The most common mistake in integration testing is using `Thread.sleep()` to wait for async processing. This is fragile — too short and the test flakes, too long and the suite is slow. Use polling-based assertion libraries instead.

```java
// Awaitility (Java) — the standard for async assertions
await()
    .atMost(Duration.ofSeconds(10))
    .pollInterval(Duration.ofMillis(200))
    .untilAsserted(() -> {
        Shipment shipment = shipmentRepository.findByOrderId("ord-123");
        assertThat(shipment).isNotNull();
        assertThat(shipment.getStatus()).isEqualTo("PENDING");
    });

// CountDownLatch — for verifying that N events were consumed
CountDownLatch latch = new CountDownLatch(3);
consumer.setMessageListener(record -> {
    processRecord(record);
    latch.countDown();
});
producer.send(event1);
producer.send(event2);
producer.send(event3);
assertTrue(latch.await(10, TimeUnit.SECONDS));
```

```python
# Python: tenacity for polling assertions
from tenacity import retry, stop_after_delay, wait_fixed

@retry(stop=stop_after_delay(10), wait=wait_fixed(0.2))
def assert_shipment_created(db, order_id):
    shipment = db.find_shipment(order_id)
    assert shipment is not None
    assert shipment.status == "PENDING"
```

---

## Contract Testing for Event Schemas

Integration tests verify that your system works *today*. Contract tests verify that it will *keep working tomorrow* when schemas evolve. In event-driven systems, schema changes are the leading cause of cross-team production incidents.

### Why Schema Evolution Breaks Consumers

A producer adds a required field. A consumer has not been updated yet. Deserialization fails. Every message on the topic becomes a poison pill. The consumer's lag grows. Alerts fire. This is the most common failure mode in event-driven architectures at scale.

Contract testing prevents this by validating compatibility *before* deployment.

### Pact for Message Contracts

Pact is a consumer-driven contract testing framework. For messaging, the producer generates a contract (the "pact"), and the consumer verifies that it can process events matching that contract.

```mermaid
graph LR
    PROD["Producer<br/>generates pact"]
    BROKER["Pact Broker<br/>(stores contracts)"]
    CONS["Consumer<br/>verifies pact"]
    CI["CI Pipeline<br/>(can-i-deploy)"]

    PROD -->|"publish pact"| BROKER
    CONS -->|"fetch & verify"| BROKER
    BROKER -->|"compatibility check"| CI

    style PROD fill:#e84393,stroke:#c2185b,color:#fff
    style BROKER fill:#f078a0,stroke:#e84393,color:#fff
    style CONS fill:#e84393,stroke:#c2185b,color:#fff
    style CI fill:#f8bbd0,stroke:#f078a0,color:#fff
```

```pseudocode
// Consumer side: define what you expect from the producer
function test_consumer_can_process_order_created():
    pact = MessagePact(consumer="shipment-service", provider="order-service")

    pact.expects_to_receive("an OrderCreated event")
        .with_content({
            "order_id": like("ord-123"),      // type matching
            "customer_id": like("cust-456"),
            "total_amount": like(99.95),
            "items": each_like({"sku": "SKU-A", "quantity": 1})
        })

    // Verify that the consumer handler can process this shape
    pact.verify(handler=shipment_handler.handle)

// Producer side: verify that real events match the pact
function test_producer_honors_order_created_contract():
    pact = load_pact("shipment-service", "order-service")
    event = order_service.create_order(...)

    pact.verify_provider(event)
    // Ensures the real event matches the shape consumers expect
```

### Schema Registry Compatibility Testing

If you use a schema registry (Confluent, Apicurio, Karapace), compatibility is enforced at registration time. But you should also test compatibility in your CI pipeline *before* attempting to register.

```pseudocode
// CI pipeline: test schema compatibility before deployment
function test_schema_backward_compatible():
    registry = SchemaRegistryClient(url="http://localhost:8081")

    new_schema = load_schema("order_created_v2.avsc")

    result = registry.test_compatibility(
        subject = "orders.created-value",
        schema = new_schema
    )

    assert result.is_compatible == true
    // If false, the new schema would break existing consumers
```

| Compatibility Mode | What It Guarantees | When to Use |
|--------------------|--------------------|-------------|
| **Backward** | New consumers can read old messages | Default for most systems; upgrade consumers first |
| **Forward** | Old consumers can read new messages | When you cannot coordinate consumer upgrades |
| **Full** | Both backward and forward | Maximum safety; only add/remove optional fields |
| **None** | No guarantees | Development only; never in production |

### AsyncAPI for Event Contracts

AsyncAPI is the OpenAPI equivalent for event-driven systems. It documents your event channels, message formats, and schemas in a machine-readable specification that can be validated automatically.

```yaml
# asyncapi.yaml
asyncapi: 2.6.0
info:
  title: Order Service Events
  version: 1.0.0

channels:
  orders.created:
    publish:
      operationId: publishOrderCreated
      message:
        payload:
          type: object
          required: [order_id, customer_id, total_amount]
          properties:
            order_id:
              type: string
            customer_id:
              type: string
            total_amount:
              type: number
            currency:
              type: string
              default: "USD"
```

AsyncAPI specifications can be used to auto-generate documentation, validate messages at runtime, and generate consumer/producer code.

### Spring Cloud Contract for Messaging

For Spring Boot applications, Spring Cloud Contract provides messaging contract verification integrated directly into the build.

```pseudocode
// Groovy DSL contract definition
Contract.make {
    label("order_created")
    input {
        triggeredBy("createOrder()")
    }
    outputMessage {
        sentTo("orders.created")
        body([
            order_id: $(anyUuid()),
            customer_id: $(anyUuid()),
            total_amount: $(anyDouble()),
            created_at: $(anyIso8601DateTime())
        ])
    }
}
```

---

## Testing Saga and Choreography Patterns

Sagas are multi-step distributed transactions where each step is a message. Testing sagas is harder than testing individual consumers because you must verify the entire flow — including every compensation path when things go wrong.

### The Challenge

A saga with N steps has at least N+1 paths to test: the happy path, plus one failure-and-compensation path for each step that can fail. A five-step saga has at least six test scenarios. Miss a compensation path, and you have a silent data consistency bug.

```mermaid
graph TD
    START["Order Saga<br/>Start"]
    S1["Step 1:<br/>Reserve Inventory"]
    S2["Step 2:<br/>Charge Payment"]
    S3["Step 3:<br/>Create Shipment"]
    SUCCESS["Saga<br/>Completed"]

    C3["Compensate:<br/>Cancel Shipment"]
    C2["Compensate:<br/>Refund Payment"]
    C1["Compensate:<br/>Release Inventory"]
    FAILED["Saga<br/>Failed"]

    START --> S1
    S1 -->|"success"| S2
    S2 -->|"success"| S3
    S3 -->|"success"| SUCCESS

    S1 -->|"failure"| FAILED
    S2 -->|"failure"| C1
    S3 -->|"failure"| C2
    C2 --> C1
    C1 --> FAILED

    style START fill:#e84393,stroke:#c2185b,color:#fff
    style S1 fill:#f078a0,stroke:#e84393,color:#fff
    style S2 fill:#f078a0,stroke:#e84393,color:#fff
    style S3 fill:#f078a0,stroke:#e84393,color:#fff
    style SUCCESS fill:#e84393,stroke:#c2185b,color:#fff

    style C3 fill:#f8bbd0,stroke:#f078a0,color:#fff
    style C2 fill:#f8bbd0,stroke:#f078a0,color:#fff
    style C1 fill:#f8bbd0,stroke:#f078a0,color:#fff
    style FAILED fill:#f8bbd0,stroke:#f078a0,color:#fff
```

### Testing the Happy Path

```pseudocode
function test_order_saga_happy_path():
    // Arrange: all services will succeed
    inventory_service = MockInventoryService(always_succeeds=true)
    payment_service = MockPaymentService(always_succeeds=true)
    shipment_service = MockShipmentService(always_succeeds=true)

    saga = OrderSaga(inventory_service, payment_service, shipment_service)

    // Act
    result = saga.execute(order_id="ord-123", items=[...], amount=99.95)

    // Assert
    assert result.status == "COMPLETED"
    assert inventory_service.reserved("ord-123") == true
    assert payment_service.charged("ord-123") == true
    assert shipment_service.created("ord-123") == true
```

### Testing Compensation Paths

```pseudocode
// Test: Payment fails -> inventory reservation is released
function test_saga_compensates_when_payment_fails():
    inventory_service = MockInventoryService(always_succeeds=true)
    payment_service = MockPaymentService(fail_on_charge=true)
    shipment_service = MockShipmentService(always_succeeds=true)

    saga = OrderSaga(inventory_service, payment_service, shipment_service)

    result = saga.execute(order_id="ord-123", items=[...], amount=99.95)

    assert result.status == "FAILED"
    assert inventory_service.reserved("ord-123") == false  // Released
    assert payment_service.charged("ord-123") == false
    assert shipment_service.created("ord-123") == false     // Never reached

// Test: Shipment fails -> payment refunded AND inventory released
function test_saga_compensates_when_shipment_fails():
    inventory_service = MockInventoryService(always_succeeds=true)
    payment_service = MockPaymentService(always_succeeds=true)
    shipment_service = MockShipmentService(fail_on_create=true)

    saga = OrderSaga(inventory_service, payment_service, shipment_service)

    result = saga.execute(order_id="ord-123", items=[...], amount=99.95)

    assert result.status == "FAILED"
    assert inventory_service.reserved("ord-123") == false   // Released
    assert payment_service.refunded("ord-123") == true      // Refunded
    assert shipment_service.created("ord-123") == false
```

### Testing Orchestrator Sagas

For orchestrator-based sagas, you can test the orchestrator as a state machine: given a current state and an incoming event, verify the next state and the command emitted.

```pseudocode
function test_orchestrator_state_transitions():
    orchestrator = OrderSagaOrchestrator()

    // Initial command
    command = orchestrator.start(order_id="ord-123")
    assert command.type == "ReserveInventory"
    assert orchestrator.state == "AWAITING_INVENTORY"

    // Inventory reserved
    command = orchestrator.on_event(InventoryReserved(order_id="ord-123"))
    assert command.type == "ChargePayment"
    assert orchestrator.state == "AWAITING_PAYMENT"

    // Payment failed -> compensate
    command = orchestrator.on_event(PaymentFailed(order_id="ord-123"))
    assert command.type == "ReleaseInventory"
    assert orchestrator.state == "COMPENSATING"
```

---

## Chaos Engineering for Messaging

Unit and integration tests verify that the system works when everything goes right. Chaos engineering verifies that the system degrades gracefully when things go wrong — and in distributed messaging, things *always* go wrong eventually.

### What to Test

| Failure Scenario | What to Verify |
|-----------------|----------------|
| **Broker node crash** | Producers buffer or retry; consumers rebalance and resume from last committed offset |
| **Network partition** (producer to broker) | No silent message loss; producers detect failure and retry or report error |
| **Network partition** (broker to consumer) | Consumer detects disconnection, reconnects, resumes processing without duplicates (or handles duplicates idempotently) |
| **Slow consumer** | Broker applies backpressure or consumer lag grows; no OOM on broker side |
| **Message duplication** | Consumer idempotency handler correctly deduplicates |
| **Out-of-order delivery** | Consumer handles reordered messages without corrupting state |
| **Consumer rebalancing** (Kafka) | No messages lost during rebalance; offset commit is correct |
| **Disk full on broker** | Producers receive errors; old messages are retained per policy; no data corruption |
| **Schema registry unavailable** | Producers and consumers fail fast with clear errors; cached schemas used if available |

### Tools for Chaos Testing

**Toxiproxy** (from Shopify) is the most popular tool for simulating network-level failures in tests. It acts as a TCP proxy between your application and the broker, allowing you to inject latency, sever connections, or corrupt data.

```pseudocode
// Toxiproxy: simulate network partition between producer and Kafka
function test_producer_handles_network_partition():
    proxy = Toxiproxy.create(name="kafka", upstream="kafka:9092", listen="0.0.0.0:19092")

    producer = KafkaProducer(bootstrap_servers="localhost:19092")

    // Normal operation — message delivered
    producer.send("orders.created", event1)
    assert_message_consumed("orders.created", event1)

    // Inject failure: cut the connection
    proxy.add_toxic("timeout", type="timeout", attributes={"timeout": 0})

    // Producer should detect failure (timeout or exception)
    result = producer.send("orders.created", event2)
    assert result.is_error == true

    // Remove failure: connection restored
    proxy.remove_toxic("timeout")

    // Producer should recover
    producer.send("orders.created", event3)
    assert_message_consumed("orders.created", event3)
```

**Other tools:**

- **tc (traffic control)** — Linux kernel tool for injecting latency, packet loss, and bandwidth limits at the network level
- **Chaos Monkey** (Netflix) — Randomly terminates instances in a cluster to test resilience
- **Litmus Chaos** — Kubernetes-native chaos engineering platform with messaging-specific experiments
- **Custom fault injection** — Wrap your producer/consumer with middleware that randomly fails, delays, or duplicates based on configurable probability

### Testing Consumer Behavior During Kafka Rebalancing

Kafka consumer group rebalancing is one of the most common sources of production incidents. During a rebalance, partitions are reassigned to consumers. If offset commit timing is wrong, you get either message loss or mass reprocessing.

```pseudocode
function test_consumer_survives_rebalance():
    // Start 2 consumers in the same group
    consumer1 = start_consumer(group="test-group", topics=["orders"])
    consumer2 = start_consumer(group="test-group", topics=["orders"])

    // Produce messages
    for i in 1..100:
        producer.send("orders", key=str(i), value=OrderCreated(order_id=str(i)))

    // Wait for initial consumption
    await_until(total_consumed(consumer1, consumer2) >= 100)

    // Kill consumer2 to trigger rebalance
    consumer2.close()

    // Produce more messages
    for i in 101..200:
        producer.send("orders", key=str(i), value=OrderCreated(order_id=str(i)))

    // Verify: consumer1 picks up all partitions, no messages lost
    await_until(total_consumed(consumer1) >= 200)

    // Verify: no duplicates (or duplicates are handled idempotently)
    assert unique_messages_processed() == 200
```

---

## Performance and Load Testing

Functional correctness is necessary but not sufficient. A system that processes messages correctly at 100 msg/s but falls over at 10,000 msg/s is not production-ready. Performance testing determines your actual capacity and identifies bottlenecks before they become incidents.

### Broker-Specific Tools

**Kafka:**

```bash
# Producer performance test: 1M messages, 1KB each, to 'perf-test' topic
kafka-producer-perf-test \
    --topic perf-test \
    --num-records 1000000 \
    --record-size 1024 \
    --throughput -1 \
    --producer-props bootstrap.servers=localhost:9092

# Consumer performance test: consume from 'perf-test' topic
kafka-consumer-perf-test \
    --topic perf-test \
    --messages 1000000 \
    --bootstrap-server localhost:9092 \
    --group perf-test-group
```

**NATS:**

```bash
# NATS bench: 1M messages, 256 bytes, 2 publishers, 2 subscribers
nats bench test-subject \
    --msgs 1000000 \
    --size 256 \
    --pub 2 \
    --sub 2 \
    --js  # Use JetStream for persistent messaging benchmark
```

**RabbitMQ:**

```bash
# PerfTest: 1 producer, 2 consumers, 1KB messages, 60 second run
rabbitmq-perf-test \
    --producers 1 \
    --consumers 2 \
    --size 1024 \
    --time 60 \
    --uri amqp://guest:guest@localhost:5672
```

### Key Metrics to Measure

| Metric | What It Tells You | Target |
|--------|-------------------|--------|
| **Throughput** (msg/s) | Maximum sustainable message rate | Must exceed peak production load by 2-3x |
| **Latency p50** | Median end-to-end latency | Baseline performance; should be stable |
| **Latency p99** | Tail latency (worst 1%) | Affects user experience; must be within SLA |
| **Latency p99.9** | Extreme tail latency | Important for latency-sensitive workloads |
| **Consumer lag** | How far behind the consumer is | Should stay near zero under sustained load |
| **Broker CPU / Memory** | Resource saturation | Headroom for burst traffic |
| **Replication lag** (Kafka) | Delay between leader and followers | Must stay low for durability guarantees |

### Capacity Planning Through Load Testing

Run load tests at increasing throughput until the system degrades. The point where latency p99 exceeds your SLA is your effective capacity ceiling. Plan for 2-3x headroom above your expected peak load.

```pseudocode
// Automated capacity test: ramp up until SLA breach
function find_capacity_ceiling():
    for rate in [1000, 5000, 10000, 25000, 50000, 100000]:
        results = run_load_test(rate_per_second=rate, duration=300)

        log("Rate: {rate}/s, p50: {results.p50}ms, p99: {results.p99}ms, lag: {results.lag}")

        if results.p99 > SLA_THRESHOLD_MS:
            log("Capacity ceiling reached at {rate}/s")
            return rate

    log("System handled all tested rates")
```

---

## Testing Anti-Patterns

Knowing what *not* to do is as important as knowing the right approach. These anti-patterns are common in teams new to testing event-driven systems.

| Anti-Pattern | Problem | Better Approach |
|-------------|---------|-----------------|
| **Mocking the broker in integration tests** | Misses serialization, configuration, and timing bugs that only manifest with real infrastructure | Use Testcontainers or embedded brokers for integration tests; reserve mocks for unit tests only |
| **Ignoring message ordering in tests** | Tests pass because messages happen to arrive in order locally, but fail in production with multiple partitions | Explicitly test with multiple partitions and out-of-order delivery; verify your consumer handles reordering |
| **Using `sleep()` for async assertions** | Too short: flaky tests. Too long: slow test suite. Always wrong. | Use polling libraries (Awaitility, tenacity) that retry assertions until a timeout |
| **Testing only the happy path** | Compensation logic, DLQ routing, and idempotency are never verified | Write explicit tests for every failure path, duplicate delivery, and schema mismatch scenario |
| **Not testing schema evolution** | A schema change that passes unit tests breaks consumers in production | Add schema compatibility checks to CI; use Pact or AsyncAPI for contract testing |
| **Single-partition test topics** | Hides concurrency and ordering bugs that appear with real partition counts | Use at least 3 partitions in integration test topics to surface real concurrency behavior |
| **No consumer lag assertions** | Tests pass but consumer is silently falling behind, losing messages to retention expiry | Assert that consumer lag returns to zero within a bounded time after producing test messages |
| **Shared test infrastructure** | Tests interfere with each other; non-deterministic failures | Each test gets its own topic (or Testcontainer instance) for full isolation |

---

## A Practical Testing Strategy

Putting it all together: here is the layered strategy that gives you confidence at every level, from fast local feedback to pre-production resilience validation.

```mermaid
graph TB
    subgraph "Layer 1: Unit Tests (milliseconds)"
        U1["Serialization /<br/>Deserialization"]
        U2["Routing Logic"]
        U3["Consumer<br/>Business Logic"]
        U4["Idempotency<br/>Handlers"]
    end

    subgraph "Layer 2: Integration Tests (seconds)"
        I1["Testcontainers /<br/>Embedded Brokers"]
        I2["End-to-End<br/>Message Flow"]
        I3["Async Assertion<br/>Verification"]
    end

    subgraph "Layer 3: Contract Tests (seconds)"
        CT1["Pact / AsyncAPI<br/>Verification"]
        CT2["Schema Registry<br/>Compatibility"]
    end

    subgraph "Layer 4: Chaos & Performance (minutes)"
        CH1["Toxiproxy<br/>Fault Injection"]
        CH2["Load & Capacity<br/>Testing"]
    end

    U1 --- I1
    U2 --- I1
    U3 --- I1
    U4 --- I1
    I1 --- CT1
    I2 --- CT1
    I3 --- CT1
    CT1 --- CH1
    CT2 --- CH1

    style U1 fill:#f8bbd0,stroke:#f078a0,color:#fff
    style U2 fill:#f8bbd0,stroke:#f078a0,color:#fff
    style U3 fill:#f8bbd0,stroke:#f078a0,color:#fff
    style U4 fill:#f8bbd0,stroke:#f078a0,color:#fff

    style I1 fill:#f078a0,stroke:#e84393,color:#fff
    style I2 fill:#f078a0,stroke:#e84393,color:#fff
    style I3 fill:#f078a0,stroke:#e84393,color:#fff

    style CT1 fill:#e84393,stroke:#c2185b,color:#fff
    style CT2 fill:#e84393,stroke:#c2185b,color:#fff

    style CH1 fill:#e84393,stroke:#c2185b,color:#fff
    style CH2 fill:#e84393,stroke:#c2185b,color:#fff
```

**Layer 1: Unit tests (run on every commit).** Test serialization round-trips for every event type. Test routing logic (correct topic/subject for each event). Test consumer handler business logic with mocked dependencies. Test idempotency by calling handlers twice with the same message ID. Target: hundreds of tests, sub-second total runtime.

**Layer 2: Integration tests (run on every PR).** Use Testcontainers to spin up real Kafka, NATS, or RabbitMQ instances. Produce events, consume them, and assert on side effects using polling assertions. Test with realistic partition counts (3+). Test consumer group rebalancing by starting and stopping consumers. Target: tens of tests, minutes total runtime.

**Layer 3: Contract tests (run on every PR that changes schemas).** Verify schema compatibility against the schema registry using CI checks. Run Pact verification to ensure producer output matches consumer expectations. Validate AsyncAPI specs against actual message formats. Gate deployments with `pact-broker can-i-deploy`. Target: fast, but blocks deployment if broken.

**Layer 4: Chaos and performance tests (run before releases).** Use Toxiproxy to simulate broker failures, network partitions, and slow consumers. Run load tests to verify throughput meets SLA under sustained traffic. Test consumer recovery after broker restart. Measure latency percentiles and consumer lag under load. Target: periodic, minutes to hours, high confidence.

---

## Key Takeaways

1. **Event-driven systems require a fundamentally different testing approach** than synchronous services. Asynchronous behavior, non-deterministic ordering, and eventual consistency make traditional "call and assert" patterns insufficient.

2. **Unit tests are your fastest feedback loop.** Test serialization, routing, business logic, and idempotency in isolation. These catch the majority of bugs and run in milliseconds. Mock the broker at this layer — that is what mocks are for.

3. **Integration tests with real brokers are non-negotiable.** Testcontainers and embedded brokers eliminate the "works on my machine" problem. Always use polling-based assertions (Awaitility, tenacity) instead of `sleep()`.

4. **Contract testing prevents the most common cross-team failure:** schema evolution that breaks consumers. Use Pact, AsyncAPI, or Schema Registry compatibility checks in CI, and gate deployments on compatibility.

5. **Every saga compensation path must be explicitly tested.** A saga with N steps has at least N+1 test scenarios. Untested compensation logic is the most likely source of silent data corruption.

6. **Chaos engineering validates what integration tests cannot:** behavior under failure. Use Toxiproxy to simulate broker crashes, network partitions, and slow consumers. Test consumer rebalancing explicitly.

7. **Performance testing is capacity planning.** Use broker-native tools (kafka-producer-perf-test, nats bench, rabbitmq-perf-test) to find your throughput ceiling. Plan for 2-3x headroom above expected peak load.

8. **Avoid the common anti-patterns:** mocking brokers in integration tests, using `sleep()` for async assertions, testing only happy paths, ignoring schema compatibility, and using single-partition test topics. Each one hides bugs that surface in production.

---

## References

- Testcontainers. ["Getting Started"](https://testcontainers.com/getting-started/) — The standard library for managing Docker containers in integration tests across Java, Go, .NET, Python, and more.
- Testcontainers. ["Kafka Module"](https://testcontainers.com/modules/kafka/) — Kafka-specific Testcontainers module with configuration examples.
- Pact Foundation. ["Pact Documentation"](https://docs.pact.io/) — Consumer-driven contract testing for HTTP and messaging.
- Pact Foundation. ["Pact for Messages"](https://docs.pact.io/getting_started/how_pact_works#non-http-testing-message-pact) — Message-specific Pact documentation for event-driven systems.
- AsyncAPI Initiative. ["AsyncAPI Specification"](https://www.asyncapi.com/docs/reference/specification/latest) — The OpenAPI equivalent for event-driven architectures.
- Confluent. ["Schema Registry Overview"](https://docs.confluent.io/platform/current/schema-registry/index.html) — Schema management and compatibility enforcement for Kafka.
- Confluent. ["Schema Evolution and Compatibility"](https://docs.confluent.io/platform/current/schema-registry/fundamentals/schema-evolution.html) — Backward, forward, and full compatibility rules.
- Shopify. ["Toxiproxy"](https://github.com/Shopify/toxiproxy) — TCP proxy for simulating network failures in tests.
- Awaitility. ["Usage Guide"](https://github.com/awaitility/awaitility/wiki/Usage) — Polling-based async assertion library for Java.
- Spring. ["Spring Cloud Contract"](https://spring.io/projects/spring-cloud-contract) — Contract testing framework with messaging support for Spring Boot.
- Spring. ["EmbeddedKafka"](https://docs.spring.io/spring-kafka/reference/testing.html) — In-process Kafka broker for Spring Kafka integration tests.
- Apache Kafka. ["Performance Testing Tools"](https://kafka.apache.org/documentation/#basic_ops_producer_perf) — Built-in producer and consumer performance test tools.
- NATS. ["nats bench"](https://docs.nats.io/using-nats/nats-tools/nats_cli#benchmarking) — NATS CLI benchmarking tool.
- RabbitMQ. ["PerfTest"](https://rabbitmq.github.io/rabbitmq-perf-test/stable/htmlsingle/) — RabbitMQ throughput testing tool.
- Netflix. ["Chaos Monkey"](https://netflix.github.io/chaosmonkey/) — Resilience testing through random instance termination.
- LitmusChaos. ["Litmus Documentation"](https://litmuschaos.io/) — Kubernetes-native chaos engineering platform.
- Richardson, C. (2018). *Microservices Patterns*. Manning Publications. — Comprehensive coverage of saga testing and distributed transaction strategies.

---

*Back to [Testing Strategies](./README.md)*
