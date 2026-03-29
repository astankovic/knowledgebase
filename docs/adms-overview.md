---
title: ADMS Overview
tags: [overview, adms, ecostruxure]
components: [ADMS]
lastUpdated: 2025-11-08
---

# ADMS Overview

## What is ADMS?
The Advanced Distribution Management System (ADMS) is a core component of Schneider Electric's EcoStruxure (ESX) suite, designed for real-time monitoring, control, and optimization of electrical distribution networks. It integrates SCADA, outage management (OMS), and advanced analytics to enhance grid reliability, support DER integration, and enable proactive decision-making for distribution operators.

## Key Features
- **Real-Time Monitoring**: Supervises substations, feeders, and field devices via SCADA protocols (e.g., DNP3, IEC 104).
- **Outage Management**: Automates fault detection, isolation, and restoration using OMS workflows.
- **Network Modeling**: Maintains a dynamic network topology model for state estimation and load flow analysis.
- **DER Integration**: Supports distributed energy resources (e.g., solar, storage) through coordination with DERMS.
- **Analytics and Optimization**: Provides forecasting, volt/VAR control, and contingency analysis for efficient operations.

## Common Use Cases
- Fault location and service restoration during outages.
- Voltage regulation and power quality improvement.
- Integration with GIS for geospatial network visualization.
- Compliance reporting for regulatory standards (e.g., NERC).

## Architecture Components
- **Core Engine**: Handles real-time data processing and alarm management.
- **User Interfaces**: Web-based HMI for operators, mobile apps for field crews.
- **Integration Layer**: Connects with RTUs, IEDs, and external systems like CIS or EMS.
- **Database**: Stores historical data, events, and configurations (typically SQL-based).

## Getting Started
1. Ensure hardware prerequisites: Compatible servers with Windows/Linux OS, network connectivity to field devices.
2. Install via Schneider Electric's deployment tools; configure database connections and license keys.
3. Import network model from GIS or CIM format.
4. Test integrations with sample SCADA points.

For detailed configuration, troubleshooting, or specific modules (e.g., State Estimator), refer to related articles (none yet—suggest creating targeted guides).

## Related
- (To be added as articles are created, e.g., [DERMS Integration](derms-integration.md))

## Changelog
- [2025-11-08 10:16] Initial ADMS overview article created
