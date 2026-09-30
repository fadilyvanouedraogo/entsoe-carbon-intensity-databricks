# Power BI — D1 measures

Connect Power BI Desktop to the SQL warehouse (*Get data → Azure Databricks*), import the 7 tables of `energy.d1_gold`,
then create the relationships below, the two calculated tables and the measures (in a `D1 Measures` table).

## Relationships

| From (many) | To (one) | Active |
|---|---|---|
| `fact_carbon_hourly.date_key` | `dim_date.date_key` | yes |
| `fact_carbon_hourly.hour_key` | `dim_hour.hour_key` | yes |
| `fact_carbon_hourly.zone_code` | `dim_zone.zone_code` | yes |
| `fact_generation_hourly.date_key` | `dim_date.date_key` | yes |
| `fact_generation_hourly.hour_key` | `dim_hour.hour_key` | yes |
| `fact_generation_hourly.zone_code` | `dim_zone.zone_code` | yes |
| `fact_generation_hourly.psr_code` | `dim_production_type.psr_code` | yes |
| `fact_flow_hourly.date_key` | `dim_date.date_key` | yes |
| `fact_flow_hourly.hour_key` | `dim_hour.hour_key` | yes |
| `fact_flow_hourly.to_zone_code` | `dim_zone.zone_code` | yes |
| `fact_flow_hourly.from_zone_code` | `dim_zone.zone_code` | no (used with USERELATIONSHIP) |

Mark `dim_date` as date table (column `date`).

## Calculated tables

```dax
Factor method =
DATATABLE ( "Method", STRING, "Order", INTEGER, { { "Life-cycle (LCA)", 1 }, { "Direct combustion", 2 } } )

Neighbour =
SELECTCOLUMNS ( dim_zone, "neighbour_code", dim_zone[zone_code], "Neighbour", dim_zone[zone_label] )
```

`Factor method` is used as a slicer. `Neighbour` is not related to any table: the exchange measures use it with `TREATAS`.

## Parameters

```dax
Active method =
SELECTEDVALUE ( 'Factor method'[Method], "Life-cycle (LCA)" )
```

## Energy

```dax
Production (MWh) =
SUM ( fact_carbon_hourly[production_mwh] )
```
Format: `#,0`

```dax
Consumption (MWh) =
SUM ( fact_carbon_hourly[load_mwh] )
```
Format: `#,0`

```dax
Imports (MWh) =
SUM ( fact_carbon_hourly[imports_mwh] )
```
Format: `#,0`

```dax
Exports (MWh) =
SUM ( fact_carbon_hourly[exports_mwh] )
```
Format: `#,0`

```dax
Net imports (MWh) =
SUM ( fact_carbon_hourly[net_imports_mwh] )
```
Format: `#,0`

```dax
Import dependency =
DIVIDE ( [Net imports (MWh)], [Consumption (MWh)] )
```
Format: `0.0%`

```dax
Renewable share =
DIVIDE ( SUM ( fact_carbon_hourly[renewable_mwh] ), [Production (MWh)] )
```
Format: `0.0%`

```dax
Low-carbon share =
DIVIDE ( SUM ( fact_carbon_hourly[low_carbon_mwh] ), [Production (MWh)] )
```
Format: `0.0%`

## Emissions

```dax
Production emissions (t) =
IF ( [Active method] = "Direct combustion", SUM ( fact_carbon_hourly[emissions_production_direct_t] ), SUM ( fact_carbon_hourly[emissions_production_lca_t] ) )
```
Format: `#,0`

```dax
Consumption emissions (t) =
IF ( [Active method] = "Direct combustion", SUM ( fact_carbon_hourly[emissions_consumption_direct_t] ), SUM ( fact_carbon_hourly[emissions_consumption_lca_t] ) )
```
Format: `#,0`

```dax
Imported emissions (t) =
IF ( [Active method] = "Direct combustion", SUM ( fact_carbon_hourly[imported_emissions_direct_t] ), SUM ( fact_carbon_hourly[imported_emissions_lca_t] ) )
```
Format: `#,0`

## Intensity

```dax
Production intensity (gCO2/kWh) =
DIVIDE ( [Production emissions (t)] * 1000, [Production (MWh)] )
```
Format: `0`

```dax
Consumption intensity (gCO2/kWh) =
DIVIDE ( [Consumption emissions (t)] * 1000, CALCULATE ( [Consumption (MWh)], fact_carbon_hourly[is_flow_traced] = TRUE () ) )
```
Format: `0`

```dax
Consumption vs production gap (gCO2/kWh) =
[Consumption intensity (gCO2/kWh)] - [Production intensity (gCO2/kWh)]
```
Format: `+0;-0;0`

```dax
Hours below 100 g =
VAR threshold = 100
RETURN
    IF (
        [Active method] = "Direct combustion",
        CALCULATE ( COUNTROWS ( fact_carbon_hourly ), fact_carbon_hourly[ci_consumption_direct_g_kwh] < threshold ),
        CALCULATE ( COUNTROWS ( fact_carbon_hourly ), fact_carbon_hourly[ci_consumption_lca_g_kwh] < threshold )
    ) + 0
```
Format: `#,0`

```dax
% hours below 100 g =
DIVIDE ( [Hours below 100 g], CALCULATE ( COUNTROWS ( fact_carbon_hourly ), fact_carbon_hourly[is_flow_traced] = TRUE () ) )
```
Format: `0.0%`

```dax
Cleanest hour =
VAR t =
    FILTER (
        ADDCOLUMNS ( VALUES ( dim_hour[hour_label] ), "@ci", [Consumption intensity (gCO2/kWh)] ),
        NOT ISBLANK ( [@ci] )
    )
RETURN
    MAXX ( TOPN ( 1, t, [@ci], ASC ), dim_hour[hour_label] )
```

```dax
Dirtiest hour =
VAR t =
    FILTER (
        ADDCOLUMNS ( VALUES ( dim_hour[hour_label] ), "@ci", [Consumption intensity (gCO2/kWh)] ),
        NOT ISBLANK ( [@ci] )
    )
RETURN
    MAXX ( TOPN ( 1, t, [@ci], DESC ), dim_hour[hour_label] )
```

```dax
Cleanest hour intensity (gCO2/kWh) =
MINX ( VALUES ( dim_hour[hour_label] ), [Consumption intensity (gCO2/kWh)] )
```
Format: `0`

```dax
Dirtiest hour intensity (gCO2/kWh) =
MAXX ( VALUES ( dim_hour[hour_label] ), [Consumption intensity (gCO2/kWh)] )
```
Format: `0`

```dax
Evening vs midday spread (gCO2/kWh) =
VAR evening = CALCULATE ( [Consumption intensity (gCO2/kWh)], dim_hour[hour_key] >= 18 && dim_hour[hour_key] <= 21 )
VAR midday = CALCULATE ( [Consumption intensity (gCO2/kWh)], dim_hour[hour_key] >= 11 && dim_hour[hour_key] <= 14 )
RETURN evening - midday
```
Format: `0`

## Sources

```dax
Generation (MWh) =
SUM ( fact_generation_hourly[generation_mwh] )
```
Format: `#,0`

```dax
Source emissions (t) =
IF ( [Active method] = "Direct combustion", SUM ( fact_generation_hourly[emissions_direct_t] ), SUM ( fact_generation_hourly[emissions_lca_t] ) )
```
Format: `#,0`

```dax
Source share =
DIVIDE ( [Generation (MWh)], CALCULATE ( [Generation (MWh)], REMOVEFILTERS ( dim_production_type ) ) )
```
Format: `0.0%`

```dax
Source colour =
SELECTEDVALUE ( dim_production_type[color_hex] )
```

```dax
Gas share of emissions =
DIVIDE (
    CALCULATE ( [Source emissions (t)], dim_production_type[psr_code] = "B04" ),
    CALCULATE ( [Source emissions (t)], REMOVEFILTERS ( dim_production_type ) )
)
```
Format: `0%`

```dax
Emission factor (g/kWh) =
IF ( [Active method] = "Direct combustion",
    SELECTEDVALUE ( dim_production_type[ef_direct_g_kwh] ),
    SELECTEDVALUE ( dim_production_type[ef_lifecycle_g_kwh] ) )
```
Format: `0`

## Exchanges

```dax
Inflows (MWh) =
CALCULATE ( SUM ( fact_flow_hourly[flow_mwh] ), USERELATIONSHIP ( fact_flow_hourly[to_zone_code], dim_zone[zone_code] ) )
```
Format: `#,0`

```dax
Outflows (MWh) =
CALCULATE ( SUM ( fact_flow_hourly[flow_mwh] ), USERELATIONSHIP ( fact_flow_hourly[from_zone_code], dim_zone[zone_code] ) )
```
Format: `#,0`

```dax
Inflow from neighbour (MWh) =
CALCULATE (
    SUM ( fact_flow_hourly[flow_mwh] ),
    REMOVEFILTERS ( dim_zone ),
    TREATAS ( VALUES ( dim_zone[zone_code] ), fact_flow_hourly[to_zone_code] ),
    TREATAS ( VALUES ( Neighbour[neighbour_code] ), fact_flow_hourly[from_zone_code] )
)
```
Format: `#,0`

```dax
Outflow to neighbour (MWh) =
CALCULATE (
    SUM ( fact_flow_hourly[flow_mwh] ),
    REMOVEFILTERS ( dim_zone ),
    TREATAS ( VALUES ( dim_zone[zone_code] ), fact_flow_hourly[from_zone_code] ),
    TREATAS ( VALUES ( Neighbour[neighbour_code] ), fact_flow_hourly[to_zone_code] )
)
```
Format: `#,0`

```dax
CO2 from neighbour (t) =
CALCULATE (
    IF ( [Active method] = "Direct combustion", SUM ( fact_flow_hourly[embedded_emissions_direct_t] ), SUM ( fact_flow_hourly[embedded_emissions_lca_t] ) ),
    REMOVEFILTERS ( dim_zone ),
    TREATAS ( VALUES ( dim_zone[zone_code] ), fact_flow_hourly[to_zone_code] ),
    TREATAS ( VALUES ( Neighbour[neighbour_code] ), fact_flow_hourly[from_zone_code] )
)
```
Format: `#,0`

```dax
Inflow intensity (gCO2/kWh) =
DIVIDE ( [CO2 from neighbour (t)] * 1000, [Inflow from neighbour (MWh)] )
```
Format: `0`

```dax
Net inflow from neighbour (MWh) =
[Inflow from neighbour (MWh)] - [Outflow to neighbour (MWh)]
```
Format: `#,0`

```dax
CO2 imported via flows (t) =
CALCULATE ( IF ( [Active method] = "Direct combustion", SUM ( fact_flow_hourly[embedded_emissions_direct_t] ), SUM ( fact_flow_hourly[embedded_emissions_lca_t] ) ), USERELATIONSHIP ( fact_flow_hourly[to_zone_code], dim_zone[zone_code] ) )
```
Format: `#,0`

```dax
CO2 exported via flows (t) =
CALCULATE ( IF ( [Active method] = "Direct combustion", SUM ( fact_flow_hourly[embedded_emissions_direct_t] ), SUM ( fact_flow_hourly[embedded_emissions_lca_t] ) ), USERELATIONSHIP ( fact_flow_hourly[from_zone_code], dim_zone[zone_code] ) )
```
Format: `#,0`
