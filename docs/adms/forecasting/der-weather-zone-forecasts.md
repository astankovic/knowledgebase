---
title: Weather Zone Forecasts for DER in ADMS
description: Learn how to configure weather zones in NMS for forecasting weather-affected DER resources like PV and wind, including profile selection and temperature adjustments for loads.
category: adms/forecasting
keywords: [weather zones, ADMS, DER forecasts, NMS, PF_WEATHER_ZONE, distributed energy resources]
source: user
created: 2026-02-05T03:14:32.598Z
updated: 2026-02-05T03:14:32.598Z
---

## Overview

Weather zones in the Oracle Utilities Network Management System (NMS) enable utilities to model and forecast distributed energy resources (DER) affected by local weather conditions, such as solar PV or wind generation. This functionality allows for differentiated profiles across zones to reflect real-time and forecasted output, while also adjusting load profiles based on temperature.

### Configuration

To implement weather zones, map each weather-affected DER and load resource to a zone in the NMS data model. Preferably source this from GIS, or use the Power Flow Engineering Data workbook if needed. During model build, populate the ZONE columns in the PF_DIST_GEN and PF_LOADS tables with values matching entries in the PF_WEATHER_ZONE table. Only weather-dependent DER technologies (e.g., PV, Wind) should be mapped; independent sources like diesel generators are excluded.

### Forecasting

Provide forecasts for each zone covering the current day plus six days ahead, specifying generation profiles for power sources and temperature data to adjust loads. Options include:

- Manual setup via the Weather Zone Forecast tool for daily profiles per zone and source, used in real-time and power flow solutions.
- Automated population of the PF_WEATHER_ZONE_FORECAST table hourly via external interfaces (e.g., weather feeds) for granular control, bypassing the tool.
- Direct hourly scaling factors instead of profiles, configurable via the distGenDefault SRS rule.

Consistency is required: use the same method (profiles or scaling) across all zones and sources. Temperature forecasts can integrate via product adapters to weather servers or custom utility-specific solutions (see the Oracle Utilities Network Management System Adapters Guide).

This setup ensures accurate modeling of weather impacts on DER and loads in ADMS simulations.
