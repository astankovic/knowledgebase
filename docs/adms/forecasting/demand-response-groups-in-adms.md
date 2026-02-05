---
title: Demand Response Groups for DER Forecasts in ADMS
description: Set up demand response groups in NMS to model and forecast DR scenarios, such as load dropping during peak periods.
category: adms/forecasting
keywords: [demand response groups, ADMS, DER forecasts, NMS, DR profiles]
source: user
created: 2026-02-05T03:29:55.858Z
updated: 2026-02-05T03:29:55.858Z
---

## Overview

Demand Response (DR) groups in NMS enable utilities to simulate and forecast DR events by grouping resources and applying specific profiles, such as load reduction during peak hours.

### Configuration

Map DR resources to groups in the NMS data model, preferably from GIS or the Power Flow Engineering Data workbook. During model build, populate the ZONE column in the PF_DIST_GEN table with values matching the PF_DEMAND_RESPONSE_GROUPS table.

### Forecasting

Forecast DR output for each group over the current day plus six days, using an external system to populate the PF_DEMAND_RESPONSE_FORECAST table. Methods include:

- Keys to profiles (e.g., on, off).
- Hourly scaling factors, set via the distGenDefault SRS rule.

Use the same method consistently for all groups and power sources.

The NMS DERMS adapter handles CSV or RDBMS forecasts from external sources. Consult the Oracle Utilities Network Management System Adapters Guide for more information.
