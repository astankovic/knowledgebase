---
title: 2030.5 Protocol Support in ADMS
slug: adms-2030-5-support
category: adms/protocols
keywords: [2030.5, ADMS, FEP server, IEEE, protocol]
created: 2026-02-02
source: user
---

## Summary
Overview of 2030.5 protocol support in ADMS via FEP server, based on 2030.5.1 standard and IEEE documentation.

## Content

### Protocol Overview
- **Name**: 2030.5 (also known as IEEE 2030.5)
- **Purpose**: Supports smart energy profile applications, including electric vehicle supply equipment (EVSE), metering, and demand response over networks.
- **Standards Basis**:
  | Standard | Description | Source |
  |----------|-------------|--------|
  | 2030.5.1 | Core implementation standard | Official specification |
  | IEEE 2030.5 | Reference documentation | IEEE publications |

### Implementation in ADMS
- **Deployment**: Hosted over FEP (Front-End Processor) server.
- **Integration**: ADMS leverages FEP for protocol termination, enabling communication with 2030.5-compliant devices.
- **Key Features**:
  - Secure RESTful API over HTTPS.
  * Resource model for DER (Distributed Energy Resources) management.
  - Authentication via client certificates or OAuth.
  - Supports HTTP/2 for efficiency.

### Configuration
- Enable in ADMS via FEP server settings:
  ```
  # Example FEP config snippet (pseudo-syntax)
  protocol 2030.5 {
    standard 2030.5.1;
    port 443;
    cert /path/to/server.crt;
    key /path/to/server.key;
  }
  ```
- Verify support: Query FEP logs for "2030.5 handshake" events.

### Supported Operations
| Operation | Endpoint Example | Description |
|-----------|------------------|-------------|
| GET /DeviceStatus | Status monitoring | Retrieve device operational state |
| POST /SetPoint | Control | Send setpoints to inverters/EVSE |
| SUBSCRIBE /Events | Event streaming | Real-time notifications |

### Limitations
- Requires IEEE-compliant endpoints.
- No legacy protocol fallback.

### References
- IEEE Std 2030.5™-2018
- 2030.5.1 Profile Specification (OCPP/SunSpec aligned)
