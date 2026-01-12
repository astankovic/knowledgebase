---
layout: default
title: Microservices Architecture
parent: Architecture
grand_parent: ADMS Overview
nav_order: 2
---

# Microservices Architecture

## Overview

ADMS employs a microservices architecture pattern where the system is decomposed into small, loosely-coupled services that implement specific business capabilities. Each microservice owns its data, exposes well-defined APIs, can be developed and deployed independently, and communicates with other services through lightweight protocols. This architectural approach provides flexibility, scalability, and resilience while enabling teams to work autonomously on different services.

The microservices architecture aligns with domain-driven design principles, where service boundaries correspond to bounded contexts within the utility distribution management domain. Services are organized around business capabilities like network model management, real-time data processing, calculation execution, and event handling rather than technical layers.

This decomposition enables different services to use technologies best suited to their requirements. Calculation services may use C++ for performance, network model services may use Java with JPA for database interaction, and integration services may use Node.js for high-throughput I/O operations. Services can be scaled independently based on load patterns, with calculation services requiring more compute resources while integration services need more network bandwidth.

## Core Services

### Network Model Service

The Network Model Service manages the electrical network model representing all distribution equipment, connectivity, and attributes based on the Common Information Model (CIM) standard. This service provides CRUD operations for network objects (substations, feeders, transformers, switches, meters), maintains topological relationships, validates model consistency, and supports versioning for model changes.

The service exposes REST APIs for querying and modifying the network model, with support for both individual object operations and bulk updates. It maintains the authoritative version of the network model and publishes change events to notify dependent services of model updates. The service implements caching strategies to optimize read performance for frequently accessed network regions.

### Real-Time Data Service

The Real-Time Data Service ingests measurements from SCADA systems, processes telemetry data, and maintains the current operational state of the distribution network. It receives measurements for voltages, currents, power flows, breaker positions, and other operational data points, typically at 2-4 second update rates from thousands of field devices.

This service performs data validation, bad data detection, and state estimation to determine the most likely electrical state of the network given potentially inconsistent measurements. It maintains an in-memory cache of current values for low-latency access and publishes value change events through message queues. The service also implements data compression and efficient storage of historical measurements in time-series databases.

### Calculation Engine Service

The Calculation Engine Service executes complex power system calculations including power flow analysis, short circuit analysis, voltage optimization, and network reconfiguration. These calculations use the network model and real-time measurements to analyze grid conditions, identify constraints, and recommend operational actions.

The service implements a queue-based job execution model where calculation requests are queued, executed by worker processes, and results are stored and published to requesting clients. Calculation jobs can be triggered on-demand by operators, scheduled periodically, or automatically triggered by network events. The service supports horizontal scaling by deploying multiple calculation workers to process jobs in parallel.

### Event Processing Service

The Event Processing Service handles alarm generation, event correlation, notification routing, and operator acknowledgment. It monitors real-time data for limit violations, equipment state changes, and system anomalies, generating alarms when conditions warrant operator attention. The service implements alarm priority ranking, suppression logic to reduce alarm flooding, and correlation rules to group related alarms.

Complex Event Processing (CEP) capabilities enable the service to detect patterns across multiple data streams, such as cascading equipment failures or abnormal load patterns. The service maintains alarm history, tracks operator acknowledgments and comments, and supports custom notification rules for escalation to supervisors or emergency responders.

### User Session Service

The User Session Service manages authentication, authorization, and user session state. It integrates with enterprise authentication systems (LDAP, Active Directory, OAuth providers), validates user credentials, issues JWT tokens, and maintains active session tracking. The service implements role-based access control (RBAC), enforcing permissions for operations based on user roles and context.

Session state including user preferences, dashboard configurations, and active views is cached in Redis for fast access across load-balanced service instances. The service publishes user activity events for audit logging and provides session management APIs for administrators to view active users and terminate sessions if needed.

## Service Communication

### API Contracts

Services expose well-defined API contracts using OpenAPI specifications for REST endpoints and Protocol Buffer definitions for gRPC services. These contracts are versioned, with support for multiple API versions to enable backward compatibility during service upgrades. API gateways enforce contracts and provide automatic API documentation through Swagger UI.

### Service Discovery

Services register themselves with a service registry (Consul, Eureka, or Kubernetes DNS) upon startup, advertising their network location and capabilities. Client services query the registry to discover service instances, enabling dynamic routing as services scale or relocate. Health checks ensure only healthy service instances receive traffic.

### Resilience Patterns

Circuit breaker patterns (implemented with libraries like Hystrix or Resilience4j) protect services from cascading failures. When a downstream service becomes unavailable, the circuit breaker opens, immediately returning failure responses rather than waiting for timeouts. After a cooldown period, the circuit breaker allows test requests to determine if the downstream service has recovered.

Retry logic with exponential backoff handles transient failures. Bulkhead patterns isolate thread pools for different dependencies to prevent resource exhaustion. Timeout configurations ensure long-running operations don't block service threads indefinitely.

## Service Boundaries

### Bounded Contexts

Service boundaries align with domain-driven design bounded contexts. The network model context includes equipment types, connectivity, and attributes. The real-time operations context includes measurements, alarms, and control operations. The integration context handles external system communication. Clear boundaries prevent tight coupling and enable independent evolution of services.

### Data Ownership

Each service owns its data with exclusive write access to its database tables or schemas. Other services cannot directly access another service's database; all data access occurs through APIs. This enforces loose coupling and enables services to change internal data representations without affecting clients. Event publishing allows services to share data changes with interested subscribers.

## Code Examples

### Service Health Check

```java
@RestController
@RequestMapping("/health")
public class HealthController {

    @Autowired
    private DatabaseHealthIndicator databaseHealth;

    @Autowired
    private RedisHealthIndicator redisHealth;

    @GetMapping
    public ResponseEntity<HealthStatus> health() {
        boolean isDatabaseHealthy = databaseHealth.check();
        boolean isRedisHealthy = redisHealth.check();

        if (isDatabaseHealthy && isRedisHealthy) {
            return ResponseEntity.ok(new HealthStatus("UP", "Service is healthy"));
        } else {
            HealthStatus status = new HealthStatus("DOWN", "Service dependencies unavailable");
            return ResponseEntity.status(503).body(status);
        }
    }

    @GetMapping("/ready")
    public ResponseEntity<String> readiness() {
        // Check if service has completed initialization
        if (serviceInitialized) {
            return ResponseEntity.ok("Ready");
        }
        return ResponseEntity.status(503).body("Not ready");
    }
}
```

### Circuit Breaker Implementation

```java
@Service
public class NetworkModelClient {

    private final RestTemplate restTemplate;
    private final CircuitBreaker circuitBreaker;

    public NetworkModelClient(RestTemplate restTemplate, CircuitBreakerFactory factory) {
        this.restTemplate = restTemplate;
        this.circuitBreaker = factory.create("network-model-service");
    }

    public Equipment getEquipment(String equipmentId) {
        return circuitBreaker.run(
            () -> restTemplate.getForObject(
                "http://network-model-service/api/equipment/{id}",
                Equipment.class,
                equipmentId
            ),
            throwable -> {
                // Fallback: return cached equipment or default
                logger.warn("Circuit breaker opened for network model service", throwable);
                return getCachedEquipment(equipmentId);
            }
        );
    }
}
```

### Event Publishing

```java
@Service
public class AlarmService {

    @Autowired
    private KafkaTemplate<String, AlarmEvent> kafkaTemplate;

    private static final String ALARM_TOPIC = "adms.alarms";

    public void publishAlarm(Alarm alarm) {
        AlarmEvent event = new AlarmEvent(
            alarm.getId(),
            alarm.getEquipmentId(),
            alarm.getSeverity(),
            alarm.getMessage(),
            Instant.now()
        );

        kafkaTemplate.send(ALARM_TOPIC, alarm.getId(), event)
            .addCallback(
                result -> logger.info("Alarm event published: {}", alarm.getId()),
                ex -> logger.error("Failed to publish alarm event", ex)
            );
    }
}
```

## Best Practices

- Define clear service boundaries based on business capabilities and domain contexts
- Keep services small and focused on single responsibilities to minimize complexity
- Design APIs with backward compatibility to enable independent service deployment
- Implement comprehensive logging and distributed tracing for troubleshooting across services
- Use asynchronous communication for operations that don't require immediate responses
- Implement idempotent operations to handle duplicate requests safely
- Version APIs explicitly and support multiple versions during transition periods
- Monitor service dependencies and implement circuit breakers to prevent cascading failures
- Maintain service documentation including API contracts, deployment procedures, and operational runbooks
