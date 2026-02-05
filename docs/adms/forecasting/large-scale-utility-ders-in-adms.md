---
title: Large Scale Utility DERs in ADMS Forecasting
description: Configure and forecast large-scale, weather-independent DERs like diesel generators and batteries in NMS, including individual unit predictions.
category: adms/forecasting
keywords: [utility scale DERs, ADMS, DER forecasts, NMS, distributed generation]
source: user
created: 2026-02-05T03:29:54.680Z
updated: 2026-02-05T03:29:54.680Z
---

## Overview

Large scale utility DERs are significant, weather-unaffected resources such as diesel generators or large batteries. Unlike weather-affected DERs, these are forecasted individually rather than aggregated, allowing for precise control over their output.

### Configuration

To exclude a DER from weather zones, set the ZONE field in the PF_DIST_GEN table to an alias indicating it's outside zoning (e.g., via GIS or Power Flow Engineering Data workbook during model build). Configure a corresponding power source (e.g., Diesel, Gas, Battery).

### Forecasting

Provide forecasts for each unit over the current day plus six days, populated via an external system into the PF_DER_FORECAST table. Options include:

- Keys to distributed generation profiles (e.g., on, off, peak shave).
- Direct hourly scaling factors, configurable with the distGenDefault SRS rule.

Maintain consistency in method (profiles or scaling) across all units and power sources.

The NMS DERMS adapter supports CSV or RDBMS inputs from external systems. See the Oracle Utilities Network Management System Adapters Guide for implementation details.
