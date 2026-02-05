---
title: Forecasting Large Scale Utility DERs in ADMS
description: Configure and forecast large-scale, weather-independent DER units like diesel generators or batteries in NMS, providing individual unit predictions for enhanced power flow accuracy.
category: adms/forecasting
keywords: [large scale DER, ADMS, DER forecasts, NMS, PF_DER_FORECAST, distributed energy resources, utility DER]
source: user
created: 2026-02-05T03:14:33.851Z
updated: 2026-02-05T03:14:33.851Z
---

## Overview

Large scale utility DERs in NMS are defined as significant, weather-unaffected resources, such as diesel generators or large batteries, whose output is controlled independently. Unlike aggregated weather-affected DERs, these require unit-specific forecasts to capture varying operational scenarios.

### Configuration

To exclude a DER from weather zones, set the ZONE field in the PF_DIST_GEN table to an alias indicating non-weather dependency, sourced preferably from GIS or the Power Flow Engineering Data workbook. During model build, associate a power source (e.g., Diesel, Gas, Battery) with the unit.

### Forecasting

Forecast output for each unit over the current day plus six days, populated via external systems into the PF_DER_FORECAST table. Options include:

- Keys to distributed generation profiles (e.g., on, off, peak shave).
- Direct hourly scaling factors, configurable via the distGenDefault SRS rule.

Apply the same method (profiles or scaling) consistently across all units and sources. The NMS DERMS adapter supports CSV or RDBMS inputs from external forecast systems (see the Oracle Utilities Network Management System Adapters Guide).

This approach allows precise simulation of utility-scale DER contributions in ADMS, even for nearby units with differing forecasts.
