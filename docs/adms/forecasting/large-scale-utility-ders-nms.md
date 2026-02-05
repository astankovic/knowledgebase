---
title: Large Scale Utility DER Forecasts in NMS
description: Configure forecasts for large-scale, weather-independent DER units like diesel generators or batteries in NMS, including individual unit forecasting and external system integration.
category: adms/forecasting
keywords: [utility scale DERs, DER forecasts, NMS, distributed energy resources, ADMS, PF_DER_FORECAST]
source: user
created: 2026-02-05T05:21:45.491Z
updated: 2026-02-05T05:21:45.491Z
---

## Overview

Large scale utility DERs are categorized as units that are large in scale and unaffected by weather. An example would be a diesel generator or large battery that has an output that is unaffected by weather. For these units, the forecast is provided for each individual unit and they are not aggregated together like DERs within weather zones. It may be entirely plausible to have two large batteries in close proximity to each other but their forecasts could be drastically different.

If a utility wishes to categorize a DER resource as being outside of a weather zone the ZONE field within the PF_DIST_GEN database table should be configured within the alias of the unit. The preferable location for this data would be the GIS, but if needed the data could be entered in the utility's Power Flow Engineering Data workbook. This data will need to be brought across during the model build. A corresponding power source also needs to be configured for the unit (for example, Diesel, Gas, Battery, and so on).

Once the large scale DERs have been configured, the utility has the ability to provide a forecast for each unit that forecasts the output for the current day plus the next six days. It is expected that an external system would be used to populate the forecasts for these units and the data is stored in the PF_DER_FORECAST database table. Within this table a key corresponding to a distributed generation profile can be provided (for example, on, off, peak shave, and so on). Alternatively, a utility could provide direct scaling factors to use for each hour instead of what profile to use for each hour, the input method can be configured with the distGenDefault SRS rule. With either method the entire forecast for all units must be provided with the same method (profiles or scaling factor) for each power source. The NMS has a DERMS adapter product that is capable of taking a CSV or RDBMS based forecast from an external system for use by the NMS (see the Oracle Utilities Network Management System Adapters Guide for more information).
