---
layout: default
title: Message Bus
parent: Integrations
grand_parent: ADMS Overview
nav_order: 5
---

# Message Bus

## Overview

Enterprise message bus integration provides asynchronous, event-driven data exchange between ADMS and external systems. Unlike synchronous request-response patterns, message-based integration decouples systems temporally and spatially: publishers send messages without knowing who will consume them, consumers process messages at their own pace, and systems operate independently without direct dependencies. This loose coupling enhances scalability, resilience, and enables complex event-driven workflows.

Message bus technologies implement publish-subscribe patterns where event producers publish messages to topics, and interested consumers subscribe to receive relevant events. Messages persist in the bus until consumed, providing delivery guarantees even when consumers are temporarily offline. The bus handles message routing, replication for high availability, and scaling across distributed clusters.

ADMS leverages message bus integration for multiple scenarios: publishing real-time alarm events to enterprise notification systems, consuming outage management system events to coordinate restoration, broadcasting network model changes to downstream analytics applications, and implementing event sourcing patterns for audit trails and system integration. The asynchronous nature enables ADMS to scale independently of external system performance while maintaining reliable data exchange.

## Message Broker Technologies

### Apache Kafka

Apache Kafka is a distributed event streaming platform designed for high-throughput, fault-tolerant messaging at scale. Kafka organizes messages into topics divided into partitions for parallel processing. Messages are append-only logs retained for configurable durations (days to weeks), enabling consumers to replay historical events or new consumers to process past events.

Kafka's architecture provides horizontal scalability by distributing topic partitions across broker nodes. Producer applications write messages to topic partitions, with partitioning keys determining partition assignment for ordering guarantees. Consumer groups enable parallel consumption where each partition is consumed by one consumer in the group, automatically rebalancing when consumers join or leave.

ADMS uses Kafka for high-volume event streaming like measurement updates (millions per hour), alarm events, and audit logs. Kafka's retention capabilities support event sourcing where the complete history of system state changes is preserved. Kafka Streams enables real-time stream processing for aggregations, filtering, and enrichment.

### RabbitMQ

RabbitMQ is a message broker implementing Advanced Message Queuing Protocol (AMQP), providing flexible routing, delivery acknowledgments, and transactional guarantees. Unlike Kafka's log-based approach, RabbitMQ uses traditional message queuing where messages are removed after consumption. This pattern suits request-response workflows, job queues, and scenarios where message retention isn't required.

RabbitMQ exchanges route messages to queues based on routing patterns. Direct exchanges route messages with specific routing keys to matching queues. Topic exchanges support wildcard routing (`alarms.critical.*`, `alarms.*.substation-a`). Fanout exchanges broadcast messages to all bound queues for pub-sub patterns.

ADMS uses RabbitMQ for work distribution like calculation job queues, command and control message flows requiring acknowledgment, and integration with systems expecting traditional message queuing semantics. RabbitMQ's transaction support ensures exactly-once processing for critical operations.

## Message Patterns

### Event Publishing

ADMS publishes events to notify external systems of significant state changes: equipment status changes, alarm generation/clearing, calculation completion, configuration updates. Events include complete context (equipment ID, old/new values, timestamp, user) enabling consumers to react without querying back to ADMS.

Event schemas are versioned to enable evolution without breaking consumers. Schema registries (Confluent Schema Registry, Apicurio) store Avro or JSON schemas with versioning and compatibility rules. Producers and consumers reference schemas by ID, and the registry ensures compatibility between schema versions.

Event publishing follows at-least-once delivery semantics where duplicate events may occur during failures. Consumers implement idempotency through deduplication or idempotent operation design. Event IDs enable duplicate detection. Some scenarios use exactly-once semantics (Kafka transactions) for critical operations requiring guaranteed unique delivery.

### Command Messages

External systems send command messages to trigger ADMS operations: execute calculation, import network model file, execute switching sequence. Commands are messages requiring explicit handling and response, often using request-reply patterns where command responses are published to reply topics.

Command messages include correlation IDs linking responses to requests, enabling asynchronous request-reply. The requester publishes commands with unique correlation ID and subscribes to responses matching that ID. ADMS processes commands, performs operations, and publishes responses with matching correlation IDs.

Command validation occurs before execution: checking permissions, validating parameters, and verifying system state. Invalid commands generate error responses without execution. Valid commands execute asynchronously with progress updates and final completion/failure responses.

### Request/Reply Pattern

The request-reply pattern implements synchronous-style communication over asynchronous messaging for operations requiring responses without REST API overhead. The requester publishes request messages to a request topic with reply-to topic name and correlation ID. The service processes requests and publishes responses to the reply-to topic.

Temporary or exclusive reply queues (RabbitMQ) ensure responses reach only the requester. Kafka achieves similar results using correlation IDs where requesters filter responses by ID. Timeouts handle cases where responses never arrive, considering messages failed after configured duration (30-60 seconds).

## Topics and Queues

### Topic Organization

Kafka topics are organized hierarchically by domain and entity type: `adms.alarms.generated`, `adms.measurements.updated`, `adms.network-model.changed`. Hierarchical naming enables topic-based subscriptions with wildcards and clear organization as topics proliferate.

Topic naming conventions include namespace (adms), domain (alarms, measurements, network-model), and event type (generated, updated, changed). Environment prefixes distinguish development/test/production topics: `dev.adms.alarms.generated`. Version suffixes support migration to new schemas: `adms.alarms.generated.v2`.

Topic configuration includes partition count (determines parallelism), replication factor (determines fault tolerance), and retention period (determines event storage duration). High-volume topics use more partitions for throughput. Critical topics use higher replication for availability.

### Consumer Groups

Kafka consumer groups enable parallel consumption where each partition is consumed by one consumer instance in the group. Adding consumers scales consumption proportionally up to the number of partitions. Consumer group coordination automatically rebalances partition assignments when consumers join or leave.

Consumer groups maintain offset tracking indicating the last processed message for each partition. Committed offsets enable consumers to resume after restart without reprocessing or skipping messages. Offset management strategies include auto-commit (periodic automatic commits), manual commit (explicit commit after processing), and manual offset management (storing offsets externally).

Multiple consumer groups subscribe to the same topic independently, each maintaining their own offsets. This enables diverse consumers (real-time dashboard, batch analytics, external system integration) to process events at different rates for different purposes without interfering.

## Code Examples

### Kafka Event Producer

```java
@Service
public class AlarmEventPublisher {

    @Autowired
    private KafkaTemplate<String, AlarmEvent> kafkaTemplate;

    private static final String TOPIC = "adms.alarms.generated";

    public void publishAlarmGenerated(Alarm alarm) {
        AlarmEvent event = AlarmEvent.builder()
            .eventId(UUID.randomUUID().toString())
            .eventType("ALARM_GENERATED")
            .timestamp(Instant.now())
            .alarmId(alarm.getId())
            .equipmentId(alarm.getEquipmentId())
            .severity(alarm.getSeverity())
            .message(alarm.getMessage())
            .build();

        ProducerRecord<String, AlarmEvent> record = new ProducerRecord<>(
            TOPIC,
            alarm.getEquipmentId(), // partition key
            event
        );

        // Add headers
        record.headers().add("eventType", "ALARM_GENERATED".getBytes());
        record.headers().add("version", "1".getBytes());

        kafkaTemplate.send(record)
            .addCallback(
                result -> logger.info("Published alarm event: {} to partition {}",
                    event.getEventId(), result.getRecordMetadata().partition()),
                ex -> logger.error("Failed to publish alarm event: {}",
                    event.getEventId(), ex)
            );
    }
}
```

### Kafka Event Consumer

```java
@Service
public class AlarmEventConsumer {

    @Autowired
    private NotificationService notificationService;

    @KafkaListener(
        topics = "adms.alarms.generated",
        groupId = "alarm-notification-service",
        concurrency = "3"
    )
    public void handleAlarmEvent(
            @Payload AlarmEvent event,
            @Header(KafkaHeaders.RECEIVED_PARTITION_ID) int partition,
            @Header(KafkaHeaders.OFFSET) long offset) {

        logger.info("Processing alarm event {} from partition {} offset {}",
            event.getEventId(), partition, offset);

        try {
            // Process event
            if (event.getSeverity() == Severity.CRITICAL) {
                notificationService.sendCriticalAlarmNotification(event);
            }

            // Event processing successful
            logger.info("Successfully processed alarm event: {}", event.getEventId());

        } catch (Exception e) {
            logger.error("Failed to process alarm event: {}", event.getEventId(), e);
            throw e; // Trigger retry or dead letter queue
        }
    }
}
```

### RabbitMQ Message Publisher

```java
@Service
public class CalculationRequestPublisher {

    @Autowired
    private RabbitTemplate rabbitTemplate;

    private static final String EXCHANGE = "adms.calculation.requests";
    private static final String ROUTING_KEY = "powerflow";

    public String submitCalculationRequest(PowerFlowRequest request) {
        String correlationId = UUID.randomUUID().toString();

        Message message = MessageBuilder
            .withBody(serializeRequest(request))
            .setContentType("application/json")
            .setCorrelationId(correlationId)
            .setReplyTo("adms.calculation.replies")
            .setExpiration("60000") // 60 seconds timeout
            .build();

        rabbitTemplate.send(EXCHANGE, ROUTING_KEY, message);

        logger.info("Published calculation request with correlation ID: {}", correlationId);
        return correlationId;
    }

    public CompletableFuture<CalculationResult> submitAndWait(PowerFlowRequest request) {
        String correlationId = submitCalculationRequest(request);

        return CompletableFuture.supplyAsync(() -> {
            // Wait for response with correlation ID
            return awaitResponse(correlationId, Duration.ofSeconds(60));
        });
    }
}
```

### Event Stream Processing

```java
@Component
public class AlarmAggregationStream {

    @Autowired
    public void buildPipeline(StreamsBuilder builder) {
        // Create stream from alarm events topic
        KStream<String, AlarmEvent> alarmStream = builder.stream(
            "adms.alarms.generated",
            Consumed.with(Serdes.String(), alarmEventSerde())
        );

        // Group by equipment and count alarms in 5-minute windows
        KTable<Windowed<String>, Long> alarmCounts = alarmStream
            .groupBy(
                (key, alarm) -> alarm.getEquipmentId(),
                Grouped.with(Serdes.String(), alarmEventSerde())
            )
            .windowedBy(TimeWindows.of(Duration.ofMinutes(5)))
            .count();

        // Filter for equipment with more than 10 alarms (alarm flooding)
        alarmCounts
            .filter((windowed, count) -> count > 10)
            .toStream()
            .map((windowed, count) -> {
                AlarmFloodingEvent event = new AlarmFloodingEvent(
                    windowed.key(),
                    count,
                    windowed.window().startTime(),
                    windowed.window().endTime()
                );
                return new KeyValue<>(windowed.key(), event);
            })
            .to("adms.alarms.flooding", Produced.with(Serdes.String(), floodingEventSerde()));
    }
}
```

## Best Practices

- Design event schemas carefully as they become contracts between systems; use schema evolution strategies for compatibility
- Include complete event context to enable consumers to process events without calling back to publishers
- Use partitioning keys strategically to ensure related events are ordered and processed by the same consumer
- Implement idempotent consumer logic to handle at-least-once delivery semantics safely
- Configure appropriate message retention periods balancing storage costs with replay requirements
- Monitor consumer lag (difference between latest message and consumer position) to detect processing slowdowns
- Use dead letter queues or topics to capture messages that fail processing repeatedly for investigation
- Implement circuit breakers in consumers to prevent cascading failures from overwhelming publishers
- Version event schemas explicitly and maintain compatibility to avoid breaking existing consumers
- Test message processing thoroughly including failure scenarios, duplicate messages, and out-of-order delivery
