---
title: Demand Response Groups Forecasts in ADMS
description: Set up demand response groups in NMS to forecast and simulate load reductions or adjustments, mapping DR resources to groups for scenario-based power flow analysis.
category: adms/forecasting
keywords: [demand response groups, ADMS, DER forecasts, NMS, PF_DEMAND_RESPONSE_FORECAST, distributed energy resources, DR groups]
source: user
created: 2026-02-05T03:14:35.121Z
updated: 2026-02-05T03:14:35.121Z
---

## Overview

Demand response (DR) groups in NMS facilitate the configuration and forecasting of aggregated DR resources, enabling simulations of load drops or adjustments during peak periods using specific profiles.

### Configuration

Map DR resources to groups in the NMS data model, ideally from GIS or the Power Flow Engineering Data workbook. During model build, populate the ZONE column in the PF_DIST_GEN table with values matching the PF_DEMAND_RESPONSE_GROUPS table.

### Forecasting

Provide group forecasts for the current day plus six days, populated by external systems into the PF_DEMAND_RESPONSE_FORECAST table. Options include:

- Keys to DR profiles (e.g., on, off).
- Direct hourly scaling factors, configurable via the distGenDefault SRS rule.

Maintain consistency in method (profiles or scaling) across all groups and sources. The NMS DERMS adapter handles CSV or RDBMS forecasts from external systems (see the Oracle Utilities Network Management System Adapters Guide).

For example, simulate a DR group reducing load for 2 hours on a peak day. This enhances ADMS's ability to model responsive demand in network scenarios.
