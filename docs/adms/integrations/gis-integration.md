---
layout: default
title: GIS Integration
parent: Integrations
grand_parent: ADMS Overview
nav_order: 1
---

# GIS Integration

## Overview

Geographic Information System (GIS) integration provides ADMS with geospatial context for electrical network equipment, enabling visualization of network topology on maps, spatial analysis of service areas, and synchronization of asset data between enterprise GIS and operational ADMS systems. GIS serves as the authoritative source for equipment locations, connectivity, and physical attributes, while ADMS adds operational data including real-time status, measurements, and control capabilities.

Modern utility GIS platforms like ESRI ArcGIS maintain comprehensive asset inventories including substations, overhead and underground lines, transformers, switches, and customer service points. This data is stored in spatial databases with geographic coordinates, map layers, and topological relationships. ADMS integrations consume this geospatial data, transforming GIS representations into electrical network models suitable for power system calculations and real-time operations.

Integration patterns range from periodic bulk synchronization where the entire network model is imported nightly from GIS, to incremental updates where only changed equipment is synchronized, to real-time integration where ADMS queries GIS services on-demand for specific equipment details. The pattern selection depends on data volumes, change frequency, network size, and operational requirements for model currency.

## GIS Data Sources

### ESRI ArcGIS Integration

ESRI ArcGIS is the dominant GIS platform in North American utilities, providing comprehensive asset management and mapping capabilities. ADMS integrates with ArcGIS through multiple mechanisms: feature services for querying spatial data, map services for rendering map layers, and geodatabase exports for bulk data extraction.

ArcGIS REST API provides programmatic access to feature services, enabling ADMS to query equipment by geographic extent, attribute filters, or object IDs. Queries return GeoJSON or other spatial formats with equipment attributes and geometries. The API supports pagination for large result sets and spatial relationship queries like "find all equipment within 500 meters of a point."

ArcGIS Online or ArcGIS Enterprise deployments expose web services that ADMS consumes. Authentication uses OAuth 2.0 or token-based authentication. For bulk synchronization, ADMS can execute stored procedures or queries against the underlying geodatabase (SQL Server or PostgreSQL with PostGIS extension), extracting equipment data directly.

### Open Geospatial Standards

For interoperability with non-ESRI GIS systems or to avoid vendor lock-in, ADMS can integrate using open standards from the Open Geospatial Consortium (OGC). Web Map Service (WMS) provides rendered map images for background visualization. Web Feature Service (WFS) provides vector feature data in GML (Geography Markup Language) or GeoJSON formats, enabling equipment queries and updates.

WFS-T (Transactional WFS) extends WFS with edit capabilities, allowing ADMS to write operational data back to GIS such as as-operated switch positions or outage-related equipment status. These standards enable ADMS to work with diverse GIS vendors including open-source platforms like GeoServer and QGIS.

## Data Synchronization

### Network Model Alignment

The critical challenge in GIS integration is aligning GIS asset representations with ADMS electrical network models. GIS data is organized geographically (map layers, spatial relationships) while ADMS requires electrical topology (connectivity, phases, impedances). The integration layer maps GIS objects to CIM classes, extracts electrical attributes, and constructs topological relationships.

For example, a GIS overhead line may be represented as a polyline geometry with attributes for voltage, conductor type, and length. The integration transforms this into ADMS conductor segments with electrical impedance calculated from conductor specifications and length. GIS connectivity (lines connected to transformers at points) becomes ADMS topology (conductor terminals connected to transformer terminals).

Discrepancies between GIS and electrical connectivity require business logic. GIS may show physical conduit paths while electrical connectivity follows circuit paths. Switching devices (normally-open vs normally-closed switches) affect operational topology differently than physical topology. The integration implements utility-specific rules to resolve these differences.

### Change Detection

Incremental synchronization improves efficiency by transferring only equipment that changed since the last sync. Change detection mechanisms include:

- **Timestamp-based**: GIS objects include last-modified timestamps; ADMS queries for records modified since last sync
- **Version-based**: GIS maintains version numbers incremented on updates; ADMS tracks synchronized versions
- **Change tables**: GIS logs changes to separate audit tables that ADMS queries
- **Database triggers**: GIS database triggers populate change queues for real-time notification

Change detection reduces data transfer volumes and processing time, enabling more frequent synchronization for near-real-time model currency. However, it requires GIS schema support for timestamps/versions and careful handling of deletions (tombstone records or explicit deletion logs).

## Spatial Queries

### Location-Based Operations

ADMS leverages GIS spatial capabilities for operational functions. "Find equipment near a storm location" queries identify potentially affected infrastructure. "Find all customers downstream of a faulted transformer" traces network connectivity combined with spatial analysis. These queries use spatial databases (PostGIS) with R-tree or GiST indexes for efficient spatial lookups.

Spatial queries use geometric predicates: contains (point in polygon for customer in service area), intersects (lines crossing regions), within distance (equipment near location), and touches (equipment connectivity). Query results combine spatial relationships with electrical topology for comprehensive operational awareness.

### Geographic Analysis

Service area calculations determine which customers are supplied by specific network segments using spatial joins between customer locations and network topology. Outage impact assessment identifies affected customers by tracing de-energized network segments and spatially selecting customers within those service areas.

Proximity analysis supports crew dispatch by finding nearest available crews to outage locations. Heat maps visualize data densities like outage frequency by geographic region. These analytical capabilities combine ADMS operational data with GIS spatial analysis tools.

## Code Examples

### ArcGIS REST API Query

```java
@Service
public class ArcGISClient {

    @Value("${arcgis.feature-service-url}")
    private String featureServiceUrl;

    @Value("${arcgis.api-token}")
    private String apiToken;

    private final RestTemplate restTemplate;

    public List<Equipment> queryEquipmentByExtent(BoundingBox extent) {
        String url = String.format(
            "%s/query?where=1=1&geometry=%s&geometryType=esriGeometryEnvelope" +
            "&spatialRel=esriSpatialRelIntersects&outFields=*&f=geojson&token=%s",
            featureServiceUrl,
            formatExtent(extent),
            apiToken
        );

        GeoJsonFeatureCollection response = restTemplate.getForObject(url, GeoJsonFeatureCollection.class);
        return response.getFeatures().stream()
            .map(this::convertToEquipment)
            .collect(Collectors.toList());
    }

    private String formatExtent(BoundingBox extent) {
        return String.format("{\"xmin\":%f,\"ymin\":%f,\"xmax\":%f,\"ymax\":%f}",
            extent.getMinX(), extent.getMinY(), extent.getMaxX(), extent.getMaxY());
    }

    private Equipment convertToEquipment(GeoJsonFeature feature) {
        Map<String, Object> properties = feature.getProperties();
        return Equipment.builder()
            .id(properties.get("OBJECTID").toString())
            .name(properties.get("FACILITYID").toString())
            .equipmentType(properties.get("SUBTYPE").toString())
            .latitude(getLatitude(feature.getGeometry()))
            .longitude(getLongitude(feature.getGeometry()))
            .build();
    }
}
```

### PostGIS Spatial Query

```sql
-- Find all transformers within 1km of a point location
SELECT
    t.equipment_id,
    t.equipment_name,
    t.voltage_level,
    ST_Distance(
        t.location::geography,
        ST_SetSRID(ST_MakePoint(-122.419, 37.775), 4326)::geography
    ) as distance_meters
FROM equipment t
WHERE ST_DWithin(
    t.location::geography,
    ST_SetSRID(ST_MakePoint(-122.419, 37.775), 4326)::geography,
    1000
)
AND t.equipment_type = 'TRANSFORMER'
ORDER BY distance_meters;

-- Find customers affected by outage on a feeder using spatial join
SELECT
    c.customer_id,
    c.customer_name,
    c.service_address
FROM customers c
INNER JOIN service_areas sa ON ST_Within(c.location, sa.geometry)
WHERE sa.feeder_id = 'FEEDER_123'
  AND sa.energized = false;
```

### Coordinate System Transformation

```python
from pyproj import Transformer

class CoordinateTransformer:
    def __init__(self, source_crs='EPSG:2227', target_crs='EPSG:4326'):
        # Example: California State Plane to WGS84
        self.transformer = Transformer.from_crs(source_crs, target_crs, always_xy=True)

    def transform_point(self, x, y):
        """Transform from source CRS to target CRS"""
        lon, lat = self.transformer.transform(x, y)
        return {'latitude': lat, 'longitude': lon}

    def transform_equipment_batch(self, equipment_list):
        """Transform coordinates for multiple equipment records"""
        for equipment in equipment_list:
            coords = self.transform_point(equipment['x'], equipment['y'])
            equipment['latitude'] = coords['latitude']
            equipment['longitude'] = coords['longitude']
        return equipment_list
```

## Best Practices

- Establish GIS as the single source of truth for equipment location and physical attributes
- Implement change detection mechanisms for efficient incremental synchronization
- Validate GIS data quality before import, checking for missing coordinates, duplicate IDs, and invalid attributes
- Handle coordinate reference system (CRS) transformations correctly, documenting all CRS conversions
- Design integration to handle partial updates gracefully without corrupting the network model
- Implement spatial indexing (R-tree, GiST) for efficient spatial queries on large datasets
- Cache frequently accessed GIS data in ADMS to reduce real-time query latency
- Monitor GIS service availability and implement fallback mechanisms for critical operations
- Maintain data lineage tracking which GIS version/date each ADMS equipment record originated from
- Test spatial queries with diverse geographic patterns to ensure correct results across service territory
