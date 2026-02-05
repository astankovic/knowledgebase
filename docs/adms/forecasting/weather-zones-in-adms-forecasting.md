---
title: Weather Zones for DER Forecasts in ADMS
description: Learn how to configure weather zones in NMS for forecasting weather-affected DER resources like PV and wind, including temperature impacts on loads.
category: adms/forecasting
keywords: [weather zones, ADMS, DER forecasts, temperature forecast, NMS]
source: user
created: 2026-02-05T03:29:53.508Z
updated: 2026-02-05T03:29:53.508Z
---

## Overview

Weather zones in the Oracle Utilities Network Management System (NMS) allow utilities to model varying weather conditions across different areas of the electrical network. This is essential for accurate forecasting of weather-affected Distributed Energy Resources (DERs), such as solar PV and wind generation, and for adjusting load profiles based on temperature changes.

### Configuration

To enable weather zone functionality, map each distributed generation and load resource in the NMS data model to an applicable zone. Preferably source this data from the GIS; otherwise, use the customer-specific Power Flow Engineering Data workbook. During model build, populate the ZONE columns in the PF_DIST_GEN and PF_LOADS tables with values that correspond to entries in the PF_WEATHER_ZONE table.

Only weather-affected DER technology types (e.g., PV, Wind) should be mapped to zones. Resources independent of weather, like large diesel generators, should not be assigned.

### Forecasting

Once configured, provide forecasts for each zone covering the current day plus the next six days. This includes generation profiles for each power source and temperature forecasts to adjust load profiles.

Utilities can implement forecasting in several ways:

- Use the Weather Zone Forecast tool for daily manual profile setting by an administrator.
- Integrate an external weather feed to populate the PF_WEATHER_ZONE_FORECAST table hourly for granular control.
- Apply direct scaling factors per hour, configurable via the distGenDefault SRS rule.

Consistency is required: use the same method (profiles or scaling factors) across all zones and power sources.

### Temperature Forecasts

Temperature data can be sourced via a product adapter interfacing with a weather data server or a custom adapter for utility-specific forecasts. Refer to the Oracle Utilities Network Management System Adapters Guide for details.
