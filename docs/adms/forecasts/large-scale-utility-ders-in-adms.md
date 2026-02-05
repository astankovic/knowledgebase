---
title: Large Scale Utility DERs in ADMS
description: Details configuration and forecasting for large-scale, weather-independent DERs like diesel generators or batteries in the NMS environment.
category: adms/forecasts
keywords: [large scale DERs, ADMS, utility DER forecasts, NMS, distributed generation, DERMS adapter]
source: user
created: 2026-02-05T01:04:19.814Z
updated: 2026-02-05T01:04:19.814Z
---

## Overview

Large scale utility DERs in NMS are defined as sizable, weather-unaffected units, such as diesel generators or large batteries, whose output does not depend on environmental conditions.

### Configuration

These DERs are forecasted individually, without aggregation, allowing for unique profiles even for nearby units. To exclude from weather zones, set the ZONE field in the PF_DIST_GEN table alias during model build, sourcing data from GIS or the Power Flow Engineering Data workbook. Configure a corresponding power source (e.g., Diesel, Gas, Battery).

### Forecasting

Provide forecasts for each unit over the current day plus six days, typically via external systems populating the PF_DER_FORECAST table. Options include:

- Keys to distributed generation profiles (e.g., on, off, peak shave).
- Direct hourly scaling factors, configurable via the distGenDefault SRS rule.

Maintain consistency in method (profiles or scaling) across all units and power sources. The NMS DERMS adapter supports CSV or RDBMS inputs from external systems (see Oracle Utilities Network Management System Adapters Guide).
