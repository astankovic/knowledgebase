---
layout: default
title: System Overview
parent: Architecture
grand_parent: ADMS Overview
nav_order: 1
---

# System Overview

## Overview

The ADMS system architecture follows a modern multi-tier design that separates concerns across presentation, application, and data layers. This layered approach enables independent scaling, technology choices appropriate to each layer's requirements, and clear separation of responsibilities. The architecture supports deployment models ranging from traditional on-premises data centers to public cloud infrastructure and hybrid configurations.

The system is designed to handle the demanding requirements of utility distribution grid operations: processing millions of real-time measurements per second, maintaining network models with hundreds of thousands of equipment objects, supporting hundreds of concurrent operators, and executing complex power system calculations with sub-second response times. High availability and fault tolerance are achieved through redundant components, automatic failover mechanisms, and distributed data replication.

Modern containerization and orchestration technologies enable the platform to scale dynamically based on load, deploy updates with zero downtime, and maintain consistent environments across development, testing, and production. The architecture leverages cloud-native patterns while supporting deployment flexibility to meet utility security and regulatory requirements.

## System Layers

### Presentation Layer

The presentation layer consists of web-based client applications accessed through modern browsers. These single-page applications (SPAs) are built with frameworks like React or Angular and communicate with backend services through REST APIs and WebSocket connections for real-time data streaming. The clients render interactive network diagrams, dashboards, data grids, and analytical visualizations.

Client applications are served through a Content Delivery Network (CDN) or web servers with caching to minimize latency and bandwidth. Authentication tokens are managed client-side, and secure HTTPS connections protect data in transit. Progressive web app capabilities enable offline functionality for critical operations.

### Application Layer

The application layer implements business logic through microservices deployed in containers. Key service categories include:

- **Network Model Services**: Manage the CIM-based network model, equipment attributes, connectivity, and topology
- **Real-Time Data Services**: Ingest SCADA measurements, perform state estimation, detect topology changes
- **Calculation Services**: Execute power flow, short circuit, optimization, and other analytical calculations
- **Event Processing Services**: Handle alarm generation, notification routing, and event correlation
- **Integration Services**: Connect to external systems through various protocols and data formats

An API Gateway provides a unified entry point for client requests, handling authentication, authorization, rate limiting, request routing, and response aggregation. Service mesh technology manages service-to-service communication with features like load balancing, circuit breaking, and distributed tracing.

### Data Layer

The data layer employs multiple specialized databases optimized for different data types and access patterns:

- **Relational Database (PostgreSQL)**: Stores network model data, configuration, user accounts, and transactional data
- **Time-Series Database (InfluxDB or TimescaleDB)**: Optimizes storage and retrieval of historical measurements and time-stamped events
- **In-Memory Cache (Redis)**: Provides low-latency access to frequently accessed data, session state, and real-time values
- **Message Queue (Kafka or RabbitMQ)**: Enables asynchronous communication and event streaming between services

Database replication ensures high availability and enables read scaling. Backup strategies include continuous database replication to standby servers and periodic snapshots to object storage.

## Deployment Architecture

### On-Premises Deployment

Traditional on-premises deployment places all infrastructure within utility data centers with direct control over hardware, networking, and security. Kubernetes clusters run on physical or virtual machines, with storage provided by SAN or NAS systems. This model provides maximum control and aligns with utilities requiring air-gapped networks for critical infrastructure protection.

High availability is achieved through multi-server redundancy, with primary and secondary data centers providing disaster recovery. Network connections to SCADA systems and control centers use dedicated secure networks isolated from corporate IT networks.

### Cloud Deployment

Cloud deployment leverages managed services from providers like AWS, Azure, or Google Cloud. Kubernetes is provided through managed services (EKS, AKS, GKE), databases through managed PostgreSQL and Redis services, and storage through object storage (S3, Azure Blob, Google Cloud Storage). This model reduces operational overhead and provides elastic scaling capabilities.

Cloud deployments must address utility security requirements through virtual private clouds (VPCs), private networking, encryption at rest and in transit, and compliance certifications (SOC 2, ISO 27001). Some utilities adopt hybrid models where SCADA integration occurs on-premises while analytical and planning functions run in the cloud.

### Hybrid Deployment

Hybrid architectures split functionality across on-premises and cloud environments. Real-time operations typically remain on-premises for security and latency requirements, while analytical applications, historical data storage, and development/test environments leverage cloud scalability and cost benefits. Secure VPN or dedicated interconnect services link the environments.

## Scalability Patterns

### Horizontal Scaling

Microservices are designed to scale horizontally by deploying additional container instances behind load balancers. Stateless service design ensures requests can be handled by any instance. Kubernetes automatically scales services based on CPU, memory, or custom metrics like queue depth or request latency.

Calculation services are particularly suited to horizontal scaling as multiple instances can process calculation requests in parallel. Real-time data ingestion can be partitioned across instances by data source or geographic region.

### Vertical Scaling

Certain components benefit from vertical scaling with larger compute resources. Database servers, in-memory caches, and calculation engines performing complex matrix operations can utilize additional CPU cores and memory. Kubernetes allows per-service resource allocation to optimize cost and performance.

### Database Scaling

Read replicas distribute query load across multiple database instances. Time-series data is partitioned by time range (daily, weekly) and aged out based on retention policies. Caching reduces database load for frequently accessed data. Connection pooling prevents database connection exhaustion under high concurrent load.

## Code Examples

### Kubernetes Service Definition

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: network-model-service
spec:
  replicas: 3
  selector:
    matchLabels:
      app: network-model
  template:
    metadata:
      labels:
        app: network-model
    spec:
      containers:
      - name: network-model
        image: adms/network-model:1.2.0
        ports:
        - containerPort: 8080
        resources:
          requests:
            memory: "1Gi"
            cpu: "500m"
          limits:
            memory: "2Gi"
            cpu: "1000m"
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
```

### Load Balancer Configuration

```nginx
upstream adms_backend {
    least_conn;
    server backend1.adms.local:8080 max_fails=3 fail_timeout=30s;
    server backend2.adms.local:8080 max_fails=3 fail_timeout=30s;
    server backend3.adms.local:8080 max_fails=3 fail_timeout=30s;
}

server {
    listen 443 ssl http2;
    server_name adms.utility.com;

    ssl_certificate /etc/ssl/certs/adms.crt;
    ssl_certificate_key /etc/ssl/private/adms.key;
    ssl_protocols TLSv1.2 TLSv1.3;

    location /api/ {
        proxy_pass http://adms_backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_connect_timeout 5s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
    }
}
```

## Best Practices

- Design services to be stateless enabling straightforward horizontal scaling and simplified load balancing
- Implement health checks for all services to enable automatic failure detection and recovery
- Use resource limits and requests in Kubernetes to prevent resource contention and enable efficient bin packing
- Deploy across multiple availability zones or data centers to protect against infrastructure failures
- Implement circuit breakers to prevent cascading failures when dependent services are unavailable
- Monitor system metrics (CPU, memory, latency, error rates) and configure alerts for anomalies
- Perform regular disaster recovery drills to validate backup and restore procedures
- Use infrastructure as code (Terraform, CloudFormation) to maintain consistent and reproducible environments
