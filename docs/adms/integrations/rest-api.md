---
layout: default
title: REST API
parent: Integrations
grand_parent: ADMS Overview
nav_order: 4
---

# REST API

## Overview

ADMS exposes RESTful APIs enabling external systems to query network data, submit control operations, retrieve measurement histories, and access calculation results. REST (Representational State Transfer) APIs use standard HTTP methods and JSON data formats, providing simple integration for web applications, mobile apps, and enterprise systems without specialized protocol libraries or middleware.

The API design follows RESTful principles with resource-oriented endpoints, stateless requests, standard HTTP status codes, and HATEOAS (Hypermedia as the Engine of Application State) for discoverability. Resources represent domain entities: equipment, measurements, alarms, users, and calculation jobs. HTTP methods map naturally to CRUD operations: GET for retrieval, POST for creation, PUT/PATCH for updates, DELETE for removal.

API implementations balance simplicity with enterprise requirements including authentication, authorization, rate limiting, versioning, error handling, and comprehensive documentation. The API Gateway pattern centralizes cross-cutting concerns while routing requests to appropriate backend microservices. OpenAPI (Swagger) specifications provide machine-readable API contracts enabling automatic client generation and interactive documentation.

## API Design

### Resource Endpoints

Resources are accessed through hierarchical URL paths reflecting domain relationships. Top-level collections represent entity types: `/api/equipment`, `/api/measurements`, `/api/alarms`. Individual resources append identifiers: `/api/equipment/{equipmentId}`. Nested resources reflect parent-child relationships: `/api/substations/{substationId}/equipment` lists equipment within a substation.

Query parameters enable filtering, pagination, sorting, and field selection. Examples:
- `/api/equipment?type=TRANSFORMER&voltage=12.47` filters by equipment type and voltage
- `/api/measurements?startTime=2024-01-01T00:00:00Z&endTime=2024-01-02T00:00:00Z` retrieves time-bounded measurements
- `/api/alarms?severity=CRITICAL&acknowledged=false&offset=20&limit=10` paginates unacknowledged critical alarms

Resource representations use JSON with consistent structures. Collections wrap items in arrays with metadata: `{ "items": [...], "total": 1523, "offset": 0, "limit": 50 }`. Individual resources include links to related resources following HATEOAS principles.

### HTTP Methods

Standard HTTP methods provide semantic operations:

- **GET**: Retrieve resources (idempotent, cacheable, safe)
- **POST**: Create new resources, trigger operations (non-idempotent)
- **PUT**: Replace entire resource (idempotent)
- **PATCH**: Partially update resource (idempotent)
- **DELETE**: Remove resource (idempotent)

Method semantics guide client behavior and server implementation. Idempotent operations (GET, PUT, PATCH, DELETE) can be retried safely without unintended effects. POST operations may create duplicates on retry, requiring idempotency keys or duplicate detection.

### Versioning Strategy

API versioning enables evolution without breaking existing clients. Version strategies include:

- **URI versioning**: `/api/v1/equipment` vs `/api/v2/equipment` (explicit, visible, but increases URL complexity)
- **Header versioning**: `Accept: application/vnd.adms.v1+json` (cleaner URLs, less visible)
- **Query parameter**: `/api/equipment?version=1` (simple but non-standard)

ADMS implements URI versioning for clarity. Major versions indicate breaking changes (removed fields, changed semantics). Minor versions add features with backward compatibility. Deprecated versions are supported for transition periods (6-12 months) with sunset headers indicating removal dates.

## Authentication & Authorization

### OAuth 2.0 Integration

API authentication uses OAuth 2.0 access tokens issued by the identity provider. Clients authenticate users, obtain access tokens, and include tokens in API requests via `Authorization: Bearer {token}` headers. The API Gateway validates tokens, extracts user identity and roles, and enforces access policies before routing requests.

Client credentials flow enables service-to-service authentication where external systems obtain tokens with client ID and secret. Resource owner password flow (discouraged) allows username/password authentication directly for legacy integrations. Authorization code flow with PKCE provides secure authentication for web and mobile applications.

Token validation verifies signatures using public keys from the identity provider, checks expiration times, and validates audience claims ensuring tokens are intended for ADMS APIs. Failed validation returns `401 Unauthorized` responses with error details.

### Rate Limiting

Rate limiting protects APIs from abuse and ensures fair resource allocation. Limits are configured per client (identified by API key or OAuth client ID) with tiered limits based on subscription levels. Common limits: 1000 requests/hour for standard clients, 10000 requests/hour for premium clients.

Rate limit information is communicated in response headers:
- `X-RateLimit-Limit: 1000` - Maximum requests per window
- `X-RateLimit-Remaining: 847` - Remaining requests in current window
- `X-RateLimit-Reset: 1609459200` - Unix timestamp when limit resets

Exceeded limits return `429 Too Many Requests` with `Retry-After` header indicating when clients can retry. Rate limiting algorithms include token bucket (smooth rate), leaky bucket (fixed capacity), and sliding window (precise limits over rolling periods).

## Response Formats

### JSON Structure

Responses use consistent JSON structures with standard fields. Success responses include requested data with appropriate HTTP status (200 OK, 201 Created, 204 No Content). Resource representations include self-referential links and related resource links following HATEOAS.

Example equipment response:
```json
{
  "id": "TRANS-001",
  "name": "Substation A - Transformer 1",
  "type": "POWER_TRANSFORMER",
  "voltage": 12.47,
  "status": "IN_SERVICE",
  "lastUpdated": "2024-01-15T10:30:00Z",
  "_links": {
    "self": { "href": "/api/equipment/TRANS-001" },
    "measurements": { "href": "/api/equipment/TRANS-001/measurements" },
    "substation": { "href": "/api/substations/SUB-A" }
  }
}
```

### Pagination

Large collections use pagination to limit response sizes and improve performance. Offset-based pagination uses `offset` and `limit` parameters: `/api/equipment?offset=100&limit=50` retrieves items 101-150. Cursor-based pagination uses opaque cursors for stable pagination: `/api/equipment?cursor=eyJpZCI6MTAwfQ&limit=50`.

Pagination metadata appears in response: `{ "items": [...], "offset": 100, "limit": 50, "total": 1523, "next": "/api/equipment?offset=150&limit=50" }`. The `next` link enables clients to iterate collections without constructing URLs manually.

### Error Responses

Error responses use appropriate HTTP status codes with structured error bodies. Client errors (4xx) indicate invalid requests: 400 Bad Request (malformed syntax), 401 Unauthorized (missing authentication), 403 Forbidden (insufficient permissions), 404 Not Found (resource doesn't exist), 422 Unprocessable Entity (validation failure).

Server errors (5xx) indicate service problems: 500 Internal Server Error (unexpected condition), 503 Service Unavailable (temporary overload or maintenance). Error bodies include machine-readable error codes and human-readable messages:

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Equipment validation failed",
    "details": [
      {
        "field": "voltage",
        "issue": "Voltage must be positive",
        "value": -12.47
      }
    ]
  }
}
```

## Code Examples

### API Controller

```java
@RestController
@RequestMapping("/api/v1/equipment")
public class EquipmentController {

    @Autowired
    private EquipmentService equipmentService;

    @GetMapping
    public ResponseEntity<PagedResponse<Equipment>> listEquipment(
            @RequestParam(required = false) String type,
            @RequestParam(required = false) Double voltage,
            @RequestParam(defaultValue = "0") int offset,
            @RequestParam(defaultValue = "50") int limit) {

        EquipmentFilter filter = EquipmentFilter.builder()
            .type(type)
            .voltage(voltage)
            .build();

        Page<Equipment> page = equipmentService.findEquipment(filter, offset, limit);

        PagedResponse<Equipment> response = PagedResponse.<Equipment>builder()
            .items(page.getContent())
            .total(page.getTotalElements())
            .offset(offset)
            .limit(limit)
            .next(buildNextUrl(offset, limit, page.getTotalElements()))
            .build();

        return ResponseEntity.ok()
            .cacheControl(CacheControl.maxAge(60, TimeUnit.SECONDS))
            .body(response);
    }

    @GetMapping("/{id}")
    public ResponseEntity<Equipment> getEquipment(@PathVariable String id) {
        return equipmentService.findById(id)
            .map(equipment -> ResponseEntity.ok()
                .eTag(equipment.getVersion())
                .body(equipment))
            .orElse(ResponseEntity.notFound().build());
    }

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public Equipment createEquipment(@Valid @RequestBody EquipmentCreate request) {
        Equipment equipment = equipmentService.create(request);

        // Add Location header
        URI location = ServletUriComponentsBuilder
            .fromCurrentRequest()
            .path("/{id}")
            .buildAndExpand(equipment.getId())
            .toUri();

        return ResponseEntity.created(location).body(equipment);
    }

    @PutMapping("/{id}")
    public ResponseEntity<Equipment> updateEquipment(
            @PathVariable String id,
            @Valid @RequestBody Equipment equipment,
            @RequestHeader(value = "If-Match", required = false) String ifMatch) {

        // Optimistic locking with ETag
        if (ifMatch != null && !equipmentService.checkVersion(id, ifMatch)) {
            return ResponseEntity.status(HttpStatus.PRECONDITION_FAILED).build();
        }

        Equipment updated = equipmentService.update(id, equipment);
        return ResponseEntity.ok()
            .eTag(updated.getVersion())
            .body(updated);
    }

    @DeleteMapping("/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    public void deleteEquipment(@PathVariable String id) {
        equipmentService.delete(id);
    }

    @ExceptionHandler(ValidationException.class)
    public ResponseEntity<ErrorResponse> handleValidationError(ValidationException ex) {
        ErrorResponse error = ErrorResponse.builder()
            .code("VALIDATION_ERROR")
            .message(ex.getMessage())
            .details(ex.getValidationErrors())
            .build();

        return ResponseEntity.status(HttpStatus.UNPROCESSABLE_ENTITY).body(error);
    }
}
```

### API Client

```javascript
class ADMSApiClient {
  constructor(baseUrl, accessToken) {
    this.baseUrl = baseUrl;
    this.accessToken = accessToken;
  }

  async getEquipment(equipmentId) {
    const response = await fetch(`${this.baseUrl}/api/v1/equipment/${equipmentId}`, {
      headers: {
        'Authorization': `Bearer ${this.accessToken}`,
        'Accept': 'application/json'
      }
    });

    if (!response.ok) {
      throw await this.handleError(response);
    }

    return response.json();
  }

  async listEquipment(filters = {}, pagination = {}) {
    const params = new URLSearchParams({
      offset: pagination.offset || 0,
      limit: pagination.limit || 50,
      ...filters
    });

    const response = await fetch(`${this.baseUrl}/api/v1/equipment?${params}`, {
      headers: {
        'Authorization': `Bearer ${this.accessToken}`,
        'Accept': 'application/json'
      }
    });

    if (!response.ok) {
      throw await this.handleError(response);
    }

    return response.json();
  }

  async createEquipment(equipment) {
    const response = await fetch(`${this.baseUrl}/api/v1/equipment`, {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${this.accessToken}`,
        'Content-Type': 'application/json'
      },
      body: JSON.stringify(equipment)
    });

    if (!response.ok) {
      throw await this.handleError(response);
    }

    return response.json();
  }

  async handleError(response) {
    const error = await response.json();
    const apiError = new Error(error.message);
    apiError.code = error.code;
    apiError.status = response.status;
    apiError.details = error.details;
    return apiError;
  }
}
```

### OpenAPI Specification

```yaml
openapi: 3.0.0
info:
  title: ADMS API
  version: 1.0.0
  description: REST API for Advanced Distribution Management System

servers:
  - url: https://adms.utility.com/api/v1
    description: Production server

security:
  - BearerAuth: []

paths:
  /equipment:
    get:
      summary: List equipment
      parameters:
        - name: type
          in: query
          schema:
            type: string
        - name: voltage
          in: query
          schema:
            type: number
        - name: offset
          in: query
          schema:
            type: integer
            default: 0
        - name: limit
          in: query
          schema:
            type: integer
            default: 50
            maximum: 100
      responses:
        '200':
          description: Equipment list
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/EquipmentList'

  /equipment/{equipmentId}:
    get:
      summary: Get equipment by ID
      parameters:
        - name: equipmentId
          in: path
          required: true
          schema:
            type: string
      responses:
        '200':
          description: Equipment details
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Equipment'
        '404':
          description: Equipment not found

components:
  securitySchemes:
    BearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT

  schemas:
    Equipment:
      type: object
      properties:
        id:
          type: string
        name:
          type: string
        type:
          type: string
        voltage:
          type: number
```

## Best Practices

- Design resource-oriented APIs with clear hierarchical URL structures reflecting domain relationships
- Use standard HTTP methods and status codes consistently across all endpoints
- Implement comprehensive input validation and return detailed error messages with field-level details
- Version APIs explicitly in URLs to enable evolution without breaking existing clients
- Document APIs thoroughly using OpenAPI specifications with examples and error scenarios
- Implement pagination for all collection endpoints to prevent large response sizes
- Use ETags for optimistic locking enabling concurrent updates with conflict detection
- Apply rate limiting to protect against abuse and ensure fair resource allocation
- Enable CORS (Cross-Origin Resource Sharing) for browser-based clients with appropriate origin restrictions
- Monitor API usage patterns, error rates, and performance metrics to identify issues and optimization opportunities
