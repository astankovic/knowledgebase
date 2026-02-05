---
title: DER and Weather Zone Forecasts Overview in ADMS
description: Overview of DER forecast categories in NMS, including weather affected DERs, utility scale DERs, and demand response groups, with configuration basics.
category: adms/forecasting
keywords: [DER forecasts, ADMS, NMS, distributed generation, forecast categories]
source: user
created: 2026-02-05T04:00:37.597Z
updated: 2026-02-05T04:00:37.597Z
---

## Overview

A utility has the ability to provide forecast data for various types of DER resources that are present within the electrical network. The NMS categorizes DERs into the following forecasts categories:
- Weather Affected DERs
- Utility Scale DERs
- Demand Response Groups

The following sections will describe how to configure them, and provide the required forecast data.

## Weather Zones

A utility has the ability to configure different weather zones within the NMS data model such that different distributed generation profiles can be used within each of the different zones to characterize the real-time and forecasted generation output. For example, if the "coastal" zone is cloudy, a user could set the forecast for PV to use a cloudy profile in the coastal zone, but all other zones would use the clear profile. The weather zone configuration is also used by loads to adjust load profiles as temperature changes. DERs technology types (for example, PV, Wind, and so on) that are mapped to weather zones should be weather affected DERs only. For example, a large utility scale diesel generator would not be mapped to a weather zone since its forecast and output is independent of weather conditions.

If a utility wishes to configure weather zone functionality, data needs to exist that maps each distributed generation and load resource in the NMS data model to one of the applicable zones. The preferable location for this data would be the GIS, but, if needed, the data could be placed in the customer specific Power Flow Engineering Data workbook. This data needs to be brought across during the model build into the ZONE columns of the PF_DIST_GEN and PF_LOADS tables. The value populated will need to map to a configured zone in the PF_WEATHER_ZONE table.

Once Weather Zones have been configured, the utility has the ability to provide a forecast for each zone that forecasts how the DER resources for each power source will behave for the current day plus the next six days. This forecast should also include a temperature forecast for each applicable zone; this will be used to adjust load profiles as temperature changes.

During implementation, a utility needs to determine how they would like to use the weather zone forecast functionality.
- They could have an administrator use the Weather Zone Forecast tool to set the generation profile daily for each zone and power source. The power flow solutions will then use the profiles that have been set for the day in the real-time solutions and forecasts.
- For more advanced forecasting, they could use an external interface (for example, weather feed) to populate the PF_WEATHER_ZONE_FORECAST table for each hour of the day with what generation profile to use. This would allow for more granular refinement of the power flow solutions. In this scenario the utility would not use the Weather Zone Forecast tool to set the weather forecasts.
- Alternatively, they could provide direct scaling factors to use for each hour instead of what profile to use for each hour; the input method can be configured with the distGenDefault SRS rule.

Whichever method is used, the entire forecast for each zone must be provided with the same method (profiles or scaling factor) for each power source.

Temperature forecasts can be provided for US customers using a product adapter that interfaces to a weather data server. Alternatively, an implementer could build a custom adapter that would provide temperature forecasts on a zone basis from an alternate resource such as utility specific forecasts (see the Oracle Utilities Network Management System Adapters Guide for more information).
