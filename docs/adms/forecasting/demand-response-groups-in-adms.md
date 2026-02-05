---
title: Demand Response Groups in ADMS
description: Setup and forecasting for demand response groups in NMS, including mapping resources and using external systems for profiles or scaling factors.
category: adms/forecasting
keywords: [demand response groups, ADMS, NMS, DR profiles, PF_DEMAND_RESPONSE_FORECAST, PF_DEMAND_RESPONSE_GROUPS]
source: user
created: 2026-02-05T04:00:40.909Z
updated: 2026-02-05T04:00:40.909Z
---

## Demand Response Groups

A utility has the ability to configure different demand response groups (DR groups) within the NMS data model such that different DR profiles can be used within each of the different groups to characterize the real-time and forecasted DR scenarios. For example, the ability to simulate a DR group dropping load for 2 hours during a peak load day.

If a utility wishes to configure demand response group functionality, data would need to exist that maps each DR resource in the NMS data model to one of the applicable demand response groups. The preferable location for this data would be the GIS, but, if needed, the data could be placed the utility's Power Flow Engineering Data workbook. This data will need to be brought across during the model build into the ZONE column of the PF_DIST_GEN table. The value populated will need to map to a configured zone in the PF_DEMAND_RESPONSE_GROUPS table.

Once the demand response groups have been configured, the utility has the ability to provide a forecast for each demand response group that forecasts the output for the current day plus the next six days. It is expected that an external system would be used to populate the forecasts for these units and the data is stored with NMS DB table PF_DEMAND_RESPONSE_FORECAST. This table can have a key corresponding to a distributed generation profile (for example, on, off, and so on). Alternatively, a utility could provide direct scaling factors to use for each hour instead of what profile to use for each hour; the input method can be configured with the distGenDefault SRS Rule. With either method, the entire forecast for all units must be provided with the same method (profiles or scaling factor) for each power source. The NMS has a DERMS adapter product that is capable of taking a CSV or RDBMS based forecast from an external system for use by the NMS (see the Oracle Utilities Network Management System Adapters Guide for more information).
