---
layout: default
title: Data Storage
parent: Architecture
grand_parent: ADMS Overview
nav_order: 4
---

# Data Storage

## Overview

Data storage in ADMS employs a polyglot persistence strategy where different types of data are stored in databases optimized for their specific access patterns, consistency requirements, and performance characteristics. The system manages multiple categories of data including structured network model data, time-series measurements, transient session state, and large binary files, each requiring different storage technologies and strategies.

Relational databases provide ACID transactions and referential integrity for critical network model data and configuration. Time-series databases optimize the storage and retrieval of millions of timestamped measurements with efficient compression and aggregation capabilities. In-memory caching reduces database load and provides sub-millisecond access to frequently read data. Message queues persist events for reliable delivery and replay capabilities.

Storage strategy decisions consider data volume, access patterns, consistency requirements, retention policies, and performance targets. Network model data requires strong consistency and is read frequently but updated infrequently. Measurement data is written continuously at high rates and queried for time-range aggregations. Cache data prioritizes speed over durability. Each storage system is configured, scaled, and backed up according to its specific requirements.

## Relational Databases

### PostgreSQL for Network Model

PostgreSQL serves as the primary relational database for network model data, configuration, user accounts, and application metadata. The database stores tens of thousands of network equipment objects with attributes including equipment type, location, electrical parameters, and connectivity relationships. The Common Information Model (CIM) class hierarchy maps to normalized table schemas with foreign key relationships maintaining topological integrity.

The network model schema includes tables for core equipment types (Substations, Feeders, Transformers, Switches, Meters) with inheritance relationships and junction tables for many-to-many connectivity. Indexes on equipment IDs, names, and geographic locations optimize query performance. Partial indexes on status fields enable efficient queries for specific operational states.

PostgreSQL's support for JSON columns enables flexible storage of equipment-specific attributes without schema modifications. The JSONB data type provides efficient indexing and querying of semi-structured data. Spatial extensions (PostGIS) support geographic queries for equipment location and service area calculations.

### Schema Design Principles

Database schemas follow third normal form (3NF) to eliminate redundancy while maintaining query performance through strategic denormalization where necessary. Equipment tables include computed columns for derived values like feeder assignments that would otherwise require expensive joins. Materialized views pre-compute complex aggregations for dashboard queries.

Partitioning strategies divide large tables (historical alarms, audit logs) by time ranges to improve query performance and enable efficient data archiving. Foreign key constraints ensure referential integrity between equipment objects and their relationships. Check constraints validate data ranges and enumerated values at the database level.

### Transaction Management

Database transactions ensure consistency for multi-step operations like network model imports where multiple equipment objects must be created atomically. Transaction isolation levels are configured based on operation requirements: read committed for most queries, serializable for critical control operations where phantom reads must be prevented.

Connection pooling (via PgBouncer or HikariCP) manages database connections efficiently, reusing connections across requests and preventing connection exhaustion under high load. Pool sizing is calculated based on concurrent user counts and average transaction durations.

## Time-Series Databases

### Measurement Data Storage

Time-series databases (InfluxDB or TimescaleDB extension for PostgreSQL) optimize storage and querying of measurement data collected from SCADA systems. Each measurement point generates values every 2-4 seconds, resulting in millions of data points per hour across thousands of measurement locations. Time-series databases compress this data efficiently while enabling fast aggregation queries.

Data is organized by measurement type (voltage, current, power, status) with tags for equipment identification and location. Tags are indexed for efficient filtering while field values store the actual measurements and quality indicators. Downsampling policies automatically aggregate high-resolution data into lower-resolution rollups (1-minute, 5-minute, 1-hour averages) for long-term retention.

Retention policies automatically delete old high-resolution data while preserving aggregated values, balancing storage costs with query capabilities. Real-time data (last 24 hours) is kept at full resolution, historical data beyond 30 days is aggregated to hourly values, and data beyond a year may be archived to cold storage.

### InfluxDB Configuration

```toml
[data]
dir = "/var/lib/influxdb/data"
wal-dir = "/var/lib/influxdb/wal"
cache-max-memory-size = "1g"
cache-snapshot-memory-size = "256m"

[[retention-policies]]
name = "realtime"
database = "adms_measurements"
duration = "24h"
replication = 1

[[retention-policies]]
name = "historical"
database = "adms_measurements"
duration = "365d"
replication = 1
shard-duration = "1w"

[[continuous-queries]]
name = "hourly_avg"
database = "adms_measurements"
query = "SELECT mean(value) INTO hourly_measurements FROM realtime_measurements GROUP BY time(1h), equipment_id"
```

## Caching Layer

### Redis for Distributed Caching

Redis provides in-memory caching for frequently accessed data, reducing database load and improving response times. The system caches current measurement values, user session state, equipment operational status, and computed calculation results. Cache entries include TTL (time-to-live) values for automatic expiration.

The cache-aside pattern is used where applications check the cache first and fall back to the database on cache misses, then populate the cache with retrieved data. This pattern is simple to implement and handles cache failures gracefully by falling back to the database. For critical real-time measurements, a write-through pattern updates both cache and database synchronously.

Redis data structures optimize specific use cases: hashes store equipment attribute collections, sorted sets maintain measurement time series for recent values, pub/sub channels broadcast real-time value changes to subscribed services, and bitmaps efficiently track operational states across thousands of equipment items.

### Cache Invalidation Strategies

Cache invalidation ensures clients receive current data after updates. Event-driven invalidation listens for network model update events and removes affected cache entries. Time-based expiration automatically removes cache entries after configured durations. For critical data, the write-through pattern keeps cache and database consistent.

Cache stampede prevention uses distributed locks to ensure only one service instance updates the cache after expiration, preventing multiple simultaneous database queries. Probabilistic early expiration randomizes TTL values to prevent synchronized expiration of many cache entries.

### Cache Configuration

```yaml
redis:
  host: redis-cluster.adms.local
  port: 6379
  password: ${REDIS_PASSWORD}
  database: 0
  pool:
    max-active: 50
    max-idle: 20
    min-idle: 5
  timeout: 2000ms
  cache:
    measurement-values:
      ttl: 60s
      max-entries: 100000
    equipment-status:
      ttl: 300s
      max-entries: 50000
    calculation-results:
      ttl: 3600s
      max-entries: 10000
```

## Backup and Recovery

### Backup Strategies

Production databases employ continuous replication to standby servers providing near-real-time disaster recovery capabilities. Streaming replication in PostgreSQL ships write-ahead logs (WAL) to replica servers that maintain synchronized copies of the database. In case of primary failure, a replica can be promoted to primary within minutes.

Periodic snapshots provide point-in-time recovery capabilities. Full backups run daily during low-activity windows, with incremental backups every hour capturing changes since the last backup. Backups are encrypted and stored in geographically separate locations with retention periods meeting regulatory requirements (typically 7 years for utility data).

Time-series data backup strategies differ due to data volume. Continuous backup of raw measurement data is often impractical, so systems backup downsampled data and rely on the raw data retention policies. Critical calculation results and alarm histories are backed up fully.

### Recovery Procedures

Recovery procedures are documented and tested regularly through disaster recovery drills. Recovery Time Objective (RTO) targets specify maximum acceptable downtime (typically 4 hours for ADMS). Recovery Point Objective (RPO) defines maximum acceptable data loss (typically 15 minutes for critical operational data).

Hot standby configurations enable near-instantaneous failover with minimal data loss. Warm standby systems require manual promotion but provide cost-effective redundancy. Backup restoration procedures are automated and tested quarterly to validate recovery capabilities and timing.

## Code Examples

### PostgreSQL Schema

```sql
CREATE TABLE equipment (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    equipment_type VARCHAR(50) NOT NULL,
    feeder_id UUID REFERENCES feeders(id),
    voltage_level DECIMAL(10,2),
    latitude DECIMAL(10,8),
    longitude DECIMAL(11,8),
    attributes JSONB,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_equipment_type ON equipment(equipment_type);
CREATE INDEX idx_equipment_feeder ON equipment(feeder_id);
CREATE INDEX idx_equipment_location ON equipment USING GIST (ll_to_earth(latitude, longitude));
CREATE INDEX idx_equipment_attributes ON equipment USING GIN (attributes);

CREATE TABLE connectivity (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    from_equipment_id UUID NOT NULL REFERENCES equipment(id),
    to_equipment_id UUID NOT NULL REFERENCES equipment(id),
    connection_type VARCHAR(50),
    UNIQUE(from_equipment_id, to_equipment_id)
);
```

### Time-Series Query

```sql
-- InfluxQL query for measurement aggregation
SELECT mean(value) AS avg_voltage,
       max(value) AS max_voltage,
       min(value) AS min_voltage
FROM voltage_measurements
WHERE equipment_id = 'TRANS-001'
  AND time >= now() - 24h
GROUP BY time(1h)
```

### Redis Cache Access

```java
@Service
public class EquipmentCache {

    @Autowired
    private RedisTemplate<String, Equipment> redisTemplate;

    private static final String CACHE_PREFIX = "equipment:";
    private static final Duration TTL = Duration.ofMinutes(5);

    public Equipment getEquipment(String id) {
        String key = CACHE_PREFIX + id;
        Equipment cached = redisTemplate.opsForValue().get(key);

        if (cached != null) {
            return cached;
        }

        // Cache miss - fetch from database
        Equipment equipment = equipmentRepository.findById(id);
        if (equipment != null) {
            redisTemplate.opsForValue().set(key, equipment, TTL);
        }
        return equipment;
    }

    public void invalidate(String id) {
        redisTemplate.delete(CACHE_PREFIX + id);
    }
}
```

## Best Practices

- Use relational databases for data requiring ACID transactions, referential integrity, and complex queries
- Employ time-series databases for high-volume timestamped data with efficient compression and retention policies
- Implement caching for frequently read data to reduce database load and improve response times
- Design database schemas with proper indexes on frequently queried columns while avoiding over-indexing
- Partition large tables by time ranges or functional boundaries to improve query performance
- Configure connection pools appropriately based on concurrent load to prevent connection exhaustion
- Implement automated backup strategies with geographically separate storage and regular recovery testing
- Monitor database performance metrics including query execution times, connection pool utilization, and cache hit rates
- Use database migrations tools (Flyway, Liquibase) for version-controlled schema evolution
