---
layout: default
title: Communication Patterns
parent: Architecture
grand_parent: ADMS Overview
nav_order: 3
---

# Communication Patterns

## Overview

Communication patterns in ADMS define how microservices, client applications, and external systems exchange data. The architecture employs both synchronous request-response patterns for operations requiring immediate results and asynchronous messaging patterns for event-driven workflows, background processing, and system integration. Selecting appropriate communication patterns is critical for system performance, scalability, and resilience.

Synchronous communication is used when clients need immediate responses, such as querying network model data or submitting control operations that require confirmation. Asynchronous messaging is preferred for publishing events that multiple subscribers may consume, triggering background calculations, or integrating with external systems where real-time responses aren't required.

The system implements multiple communication protocols and technologies, each optimized for specific use cases. REST APIs provide simple, widely-understood endpoints for CRUD operations. gRPC offers high-performance binary communication for service-to-service calls. Message queues enable reliable asynchronous messaging with delivery guarantees. WebSockets provide bidirectional real-time communication for pushing updates to client applications.

## Synchronous Communication

### REST APIs

REST APIs are the primary synchronous communication mechanism for client-to-service and some service-to-service interactions. REST follows resource-oriented design principles with HTTP methods (GET, POST, PUT, DELETE) mapping to CRUD operations on resources like network equipment, measurements, alarms, and users.

Endpoints follow consistent URL patterns: `/api/equipment/{id}` for individual resources, `/api/equipment` for collections. Query parameters support filtering (`?feeder=F123`), pagination (`?offset=0&limit=50`), and field selection (`?fields=id,name,status`). Response formats use JSON with consistent error structures including error codes, messages, and validation details.

REST APIs are stateless with authentication handled through JWT tokens passed in Authorization headers. The API Gateway terminates TLS connections, validates tokens, enforces rate limits, and routes requests to backend services. Response caching at the gateway level improves performance for frequently requested data.

### gRPC Services

gRPC provides high-performance synchronous communication between backend services using Protocol Buffers for efficient binary serialization. gRPC is particularly well-suited for service-to-service calls requiring low latency and high throughput, such as calculation services calling network model services to retrieve topology data.

gRPC supports strongly-typed service contracts defined in `.proto` files, enabling compile-time type checking and automatic client/server code generation. The binary encoding reduces message sizes compared to JSON, and HTTP/2 multiplexing allows multiple concurrent requests over a single connection without head-of-line blocking.

For internal service communication, gRPC offers better performance than REST while maintaining a clear contract-based approach. However, gRPC is typically not exposed directly to browser clients due to limited browser support; a gRPC-Web proxy can translate between browser-friendly protocols and gRPC for services that need it.

## Asynchronous Communication

### Message Queues

Message queues provide reliable asynchronous communication with delivery guarantees, message persistence, and decoupling of producers and consumers. ADMS uses message queue systems like RabbitMQ (implementing AMQP protocol) or Apache Kafka (implementing a distributed log) for event-driven workflows.

Queue-based messaging follows point-to-point patterns where each message is consumed by a single consumer from a queue. This pattern is used for work distribution, such as calculation job requests where multiple calculation workers pull jobs from a queue for parallel processing. Queues provide load leveling, preventing service overload by buffering requests during traffic spikes.

Message acknowledgment semantics ensure reliable processing. Consumers acknowledge messages after successful processing; unacknowledged messages are redelivered. Dead letter queues capture messages that fail processing after retry attempts for analysis and manual intervention.

### Event Streaming

Event streaming with Apache Kafka publishes events to topics that multiple subscribers can consume independently. This publish-subscribe pattern is ideal for broadcasting state changes like network model updates, alarm generation, or measurement value changes where multiple services need to react.

Kafka provides an immutable, append-only log of events with topic partitioning for parallel consumption and consumer groups for load distribution. Events are retained for configurable time periods, enabling new services to replay historical events. Event sourcing patterns use Kafka to maintain complete audit trails of system state changes.

Stream processing frameworks like Kafka Streams or Apache Flink enable complex event processing, aggregations, and transformations on event streams. For example, calculating moving averages of measurements or correlating multiple alarm events to detect complex failure patterns.

### WebSockets

WebSockets enable bidirectional, real-time communication between client applications and backend services over persistent connections. Unlike HTTP request-response patterns, WebSocket connections remain open, allowing servers to push data to clients immediately when changes occur without polling.

ADMS uses WebSockets to stream real-time measurement updates, alarm notifications, and network state changes to operator workstations. When a field device measurement changes or an alarm is generated, the event is pushed through WebSocket connections to all subscribed clients within milliseconds. This provides operators with immediate situational awareness of grid conditions.

WebSocket connections are managed through a dedicated message gateway service that handles client connection lifecycle, subscription management, authentication, and message routing. The service consumes events from Kafka topics and pushes relevant events to subscribed clients based on their interest (geographic region, equipment type, alarm priority).

## Communication Patterns in Practice

### Request-Reply Pattern

For operations requiring immediate confirmation, such as executing a remote control operation on a breaker, the request-reply pattern uses synchronous HTTP or gRPC calls. The client sends a control request, the service validates the request, executes the operation, and returns a response indicating success or failure with details.

Timeout configurations prevent clients from waiting indefinitely for responses. Idempotent operation design ensures that retry attempts don't cause duplicate state changes. For operations with longer execution times, the service may return immediately with a job identifier, and the client polls for completion status or subscribes to a WebSocket notification.

### Fire-and-Forget Pattern

For operations that don't require immediate confirmation, the fire-and-forget pattern publishes a message to a queue or topic and continues without waiting for acknowledgment. This pattern is used for triggering background calculations, logging audit events, or notifying external systems of changes.

The pattern improves system responsiveness by not blocking on non-critical operations. However, it requires careful error handling since the sender doesn't receive direct feedback if processing fails. Services typically publish status events to topics that interested parties can subscribe to for progress tracking.

### Event-Driven Workflows

Complex workflows are orchestrated through event-driven patterns where services react to events published by other services. For example, when a network model update event is published, multiple services may react: the calculation service re-executes affected calculations, the real-time data service updates topology, and the client UI service invalidates cached diagram data.

This loose coupling allows adding new functionality without modifying existing services. Workflow orchestration can be implemented through choreography (services autonomously react to events) or orchestration (a coordinator service explicitly manages workflow steps).

## Code Examples

### REST API Client

```java
@Service
public class EquipmentClient {

    private final RestTemplate restTemplate;

    public EquipmentClient(RestTemplate restTemplate) {
        this.restTemplate = restTemplate;
    }

    public Equipment getEquipment(String id) {
        String url = "http://network-model-service/api/equipment/{id}";
        return restTemplate.getForObject(url, Equipment.class, id);
    }

    public List<Equipment> searchEquipment(String feederName) {
        String url = "http://network-model-service/api/equipment?feeder={feeder}&limit=100";
        EquipmentList response = restTemplate.getForObject(url, EquipmentList.class, feederName);
        return response.getItems();
    }

    public Equipment updateEquipment(String id, Equipment equipment) {
        String url = "http://network-model-service/api/equipment/{id}";
        restTemplate.put(url, equipment, id);
        return getEquipment(id);
    }
}
```

### Kafka Event Producer

```java
@Service
public class AlarmEventProducer {

    @Autowired
    private KafkaTemplate<String, AlarmEvent> kafkaTemplate;

    private static final String ALARM_TOPIC = "adms.alarms.generated";

    public void publishAlarmEvent(Alarm alarm) {
        AlarmEvent event = AlarmEvent.builder()
            .alarmId(alarm.getId())
            .equipmentId(alarm.getEquipmentId())
            .severity(alarm.getSeverity())
            .message(alarm.getMessage())
            .timestamp(Instant.now())
            .build();

        ProducerRecord<String, AlarmEvent> record = new ProducerRecord<>(
            ALARM_TOPIC,
            alarm.getEquipmentId(), // key for partitioning
            event
        );

        kafkaTemplate.send(record).addCallback(
            result -> logger.info("Published alarm event: {}", alarm.getId()),
            ex -> logger.error("Failed to publish alarm event: {}", alarm.getId(), ex)
        );
    }
}
```

### WebSocket Message Handler

```javascript
class RealtimeDataClient {
  constructor(url) {
    this.ws = new WebSocket(url);
    this.subscribers = new Map();

    this.ws.onopen = () => {
      console.log('WebSocket connected');
      this.sendAuth();
    };

    this.ws.onmessage = (event) => {
      const message = JSON.parse(event.data);
      this.handleMessage(message);
    };

    this.ws.onerror = (error) => {
      console.error('WebSocket error:', error);
    };

    this.ws.onclose = () => {
      console.log('WebSocket closed, reconnecting...');
      setTimeout(() => this.reconnect(), 5000);
    };
  }

  subscribe(measurementId, callback) {
    this.subscribers.set(measurementId, callback);
    this.send({
      type: 'subscribe',
      measurementId: measurementId
    });
  }

  handleMessage(message) {
    if (message.type === 'measurement') {
      const callback = this.subscribers.get(message.measurementId);
      if (callback) {
        callback(message.value, message.timestamp);
      }
    }
  }

  send(data) {
    if (this.ws.readyState === WebSocket.OPEN) {
      this.ws.send(JSON.stringify(data));
    }
  }
}
```

## Best Practices

- Use synchronous REST/gRPC for operations requiring immediate responses and interactive user operations
- Employ asynchronous messaging for events, notifications, and operations that can be processed in the background
- Implement timeouts for all synchronous calls to prevent indefinite blocking
- Design idempotent operations to safely handle retries without duplicating state changes
- Use message acknowledgment and dead letter queues for reliable asynchronous processing
- Partition Kafka topics to enable parallel consumption and scale message throughput
- Implement WebSocket reconnection logic and subscription recovery for resilient real-time connections
- Monitor message queue depths and processing latencies to detect performance bottlenecks
- Document message schemas and API contracts with versioning for backward compatibility
