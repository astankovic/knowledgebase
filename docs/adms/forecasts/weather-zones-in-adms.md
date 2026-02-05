---
title: Weather Zones for DER Forecasts in ADMS
description: Explains how to configure weather zones in NMS for weather-affected DERs and loads, including forecast integration for generation profiles and temperature adjustments.
category: adms/forecasts
keywords: [weather zones, ADMS, DER forecasts, NMS, temperature forecasts, distributed generation]
source: user
created: 2026-02-05T01:04:18.627Z
updated: 2026-02-05T01:04:18.627Z
---

## Overview

Weather zones in the Oracle Utilities Network Management System (NMS) allow utilities to model varying weather impacts on distributed energy resources (DERs) and loads across different areas of the electrical network.

### Configuration

A utility can configure different weather zones within the NMS data model to apply distinct distributed generation profiles for real-time and forecasted output. For instance, in a 'coastal' zone with cloudy conditions, PV forecasts might use a cloudy profile, while other zones use a clear profile. Weather zones also adjust load profiles based on temperature changes. Only weather-affected DER technologies (e.g., PV, Wind) should be mapped to zones; utility-scale diesel generators, unaffected by weather, are excluded.

To enable this, map each distributed generation and load resource to a zone using data from GIS or the Power Flow Engineering Data workbook. During model build, populate the ZONE columns in PF_DIST_GEN and PF_LOADS tables, referencing values in the PF_WEATHER_ZONE table.

### Forecasting

Once configured, provide forecasts for each zone covering the current day plus six days ahead. Include generation behavior for each power source and temperature forecasts to adjust loads. Options include:

- Manual daily setup via the Weather Zone Forecast tool for profiles per zone and source.
- External interfaces (e.g., weather feeds) to populate PF_WEATHER_ZONE_FORECAST hourly.
- Direct scaling factors per hour, configurable via the distGenDefault SRS rule.

Use consistent methods (profiles or scaling) across all power sources per zone. Temperature forecasts can integrate via product adapters to weather servers or custom utility-specific adapters (see Oracle Utilities Network Management System Adapters Guide).
