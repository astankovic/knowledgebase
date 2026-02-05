---
title: Demand Response Groups in ADMS
description: Covers setup and forecasting for demand response groups in NMS to simulate load reduction scenarios using DR profiles.
category: adms/forecasts
keywords: [demand response groups, ADMS, DR forecasts, NMS, distributed generation, DERMS adapter]
source: user
created: 2026-02-05T01:04:21.202Z
updated: 2026-02-05T01:04:21.202Z
---

## Overview

Demand response groups (DR groups) in NMS enable modeling of aggregated DR resources with specific profiles for real-time and forecasted scenarios, such as load drops during peak periods.

### Configuration

Map DR resources to groups using GIS data or the Power Flow Engineering Data workbook. During model build, populate the ZONE column in PF_DIST_GEN, referencing the PF_DEMAND_RESPONSE_GROUPS table.

### Forecasting

Forecast output for each group over the current day plus six days, populated via external systems into the PF_DEMAND_RESPONSE_FORECAST table. Options include:

- Keys to profiles (e.g., on, off).
- Hourly scaling factors, configurable via the distGenDefault SRS rule.

Ensure consistent methods across all groups and power sources. The NMS DERMS adapter handles CSV or RDBMS forecasts from external sources (see Oracle Utilities Network Management System Adapters Guide).
