# TimeSeriesChart

## Purpose

The Time Series Chart widget renders one or more line series plotted over a DateTime x-axis using Plotly.js (via `@mendix/shared-charts`). It is designed for displaying data that changes over time — sensor readings, KPI trends, financial data, or any metric with a DateTime x-axis. The y-axis is intentionally non-zoomable; only the x-axis supports panning and zooming. Multiple data series can be configured per chart, each with independent static or dynamic (grouped) data sources, optional fill areas, custom colors, and click actions. The widget supports offline use and requires an entity context.

## User Scenarios

### [P1] Display multiple time-series lines
**Given** a chart with two or more configured series, each bound to a datasource with DateTime x-attribute and numeric y-attribute  
**When** the page loads and data is available  
**Then** each series renders as a line (or line+markers) over the shared DateTime x-axis, with optional fill areas and a legend  

#### Edge Cases
- When `enableFillArea = true` (default), the first series fills to the zero baseline; subsequent series fill to the line of the previous series (`fill: "tonexty"`), creating stacked shading.
- When `aggregationType` is set to anything other than `"none"`, x-values are converted from `Date` objects to ISO strings internally; Plotly accepts both formats.

### [P2] Interactive range slider
**Given** the chart with `showRangeSlider = true` (default)  
**When** the user drags the range slider handles below the chart  
**Then** the visible x-axis range updates to show only the selected time window  

### [P3] Dynamic (grouped) series
**Given** a chart configured with `dataSet = "dynamic"` and a `groupByAttribute`  
**When** the data loads  
**Then** the single datasource is split into one line per unique value of `groupByAttribute` (e.g., one line per sensor ID)  

### [P4] Click action on data point
**Given** a series with a configured `onClick` action  
**When** the user clicks a data point on the chart  
**Then** the Mendix action executes in the context of the clicked data item  

## Functional Requirements

- FR-001: The x-axis MUST be typed as `"date"` — only `DateTime` attributes are accepted for the x-axis.
- FR-002: The y-axis MUST have `fixedrange: true` — y-axis zoom and pan are disabled; only x-axis interaction is permitted.
- FR-003: `yAxisRangeMode` MUST default to `"tozero"`; values of `"normal"` (auto) and `"nonnegative"` MUST also be supported; this option MUST be hidden unless `enableAdvancedOptions` is enabled.
- FR-004: Fill area MUST use `fill: "tonexty"` (fill to previous trace), not fill to x-axis. The first series effectively fills to zero; subsequent series fill to the series below.
- FR-005: Line mode MUST map: `lineStyle: "line"` → Plotly `mode: "lines"`; `lineStyle: "lineWithMarkers"` or `"custom"` → Plotly `mode: "lines+markers"`.
- FR-006: Interpolation MUST map: `"linear"` → Plotly `line.shape: "linear"`; `"spline"` → Plotly `line.shape: "spline"`.
- FR-007: Colors (`lineColor`, `markerColor`, `fillColor`) MUST be per-series `DynamicValue<string>` expressions — uniform for the entire series, not per data point.
- FR-008: `markerColor` property MUST be hidden in Studio Pro unless `lineStyle === "lineWithMarkers"`.
- FR-009: `fillColor` property MUST be hidden in Studio Pro unless `enableFillArea` is true.
- FR-010: Ten aggregation types MUST be supported: none, count, sum, avg, min, max, median, mode, first, last.
- FR-011: Dynamic series MUST support `groupByAttribute` to split a single datasource into multiple lines by attribute value.
- FR-012: The range slider MUST default to visible (`showRangeSlider: true`).
- FR-013: Advanced options (`customLayout`, `customConfigurations`, `customSeriesOptions`, `enableThemeConfig`, `yAxisRangeMode`) MUST be gated behind `enableAdvancedOptions` toggle.
- FR-014: The chart MUST be responsive (`configOptions: { responsive: true }`).
- FR-015: The widget MUST support CSS class and inline style customization.
- FR-016: System MUST support offline use (`offlineCapable="true"`) and requires entity context.

## Props Reference

### Chart-level

| Name | Type | Default | Caption | Description |
|------|------|---------|---------|-------------|
| `lines` | Object list | — | Series | List of time-series lines; each entry is one series |
| `xAxisLabel` | string (optional) | — | X axis label | Label text for the x-axis |
| `yAxisLabel` | string (optional) | — | Y axis label | Label text for the y-axis |
| `showLegend` | boolean | `true` | Show legend | Toggles the chart legend |
| `showRangeSlider` | boolean | `true` | Show range slider | Toggles the range slider below the chart |
| `gridLines` | `"none"` \| `"horizontal"` \| `"vertical"` \| `"both"` | — | Grid lines | Controls visible grid lines |
| `enableAdvancedOptions` | boolean | `false` | Advanced options | Unlocks advanced Plotly configuration properties |
| `yAxisRangeMode` | `"normal"` \| `"tozero"` \| `"nonnegative"` | `"tozero"` | Y-axis range mode | Controls y-axis floor; hidden unless `enableAdvancedOptions` is on |
| `widthUnit` | `"percentage"` \| `"pixels"` | `"percentage"` | Width unit | |
| `heightUnit` | `"percentageOfWidth"` \| `"pixels"` \| `"percentageOfParent"` | `"percentageOfWidth"` | Height unit | |

### Per-series (`lines[]`)

| Name | Type | Default | Caption | Description |
|------|------|---------|---------|-------------|
| `dataSet` | `"static"` \| `"dynamic"` | — | Dataset | Static: one datasource per series. Dynamic: one datasource split by `groupByAttribute` |
| `staticDataSource` / `dynamicDataSource` | ListValue | — | Data source | Mutually exclusive depending on `dataSet` |
| `groupByAttribute` | `ListAttributeValue<any>` (optional) | — | Group by | Dynamic mode only: attribute whose unique values define separate lines |
| `staticXAttribute` / `dynamicXAttribute` | `ListAttributeValue<Date>` | — | X attribute | **DateTime only** |
| `staticYAttribute` / `dynamicYAttribute` | various | — | Y attribute | Accepts String, Enum, DateTime, Decimal, Integer, Long, AutoNumber |
| `aggregationType` | enum (10 options) | `"none"` | Aggregation | Aggregation function applied to Y values sharing the same X timestamp |
| `interpolation` | `"linear"` \| `"spline"` | — | Interpolation | Line curve style |
| `lineStyle` | `"line"` \| `"lineWithMarkers"` \| `"custom"` | — | Line style | Controls whether markers are shown |
| `lineColor` | `DynamicValue<string>` (optional) | — | Line color | CSS color value; uniform per series |
| `markerColor` | `DynamicValue<string>` (optional) | — | Marker color | CSS color value; shown only when `lineStyle === "lineWithMarkers"` |
| `fillColor` | `DynamicValue<string>` (optional) | — | Fill color | CSS color value; shown only when `enableFillArea` is true |
| `enableFillArea` | boolean | `true` | Enable fill area | Fills the area below the line to the previous trace |
| `staticOnClickAction` / `dynamicOnClickAction` | ActionValue (optional) | — | On click | Mendix action executed in the context of the clicked data item |
| `customSeriesOptions` | string | `""` | Custom series options | Raw Plotly trace options (JSON); shown only when `enableAdvancedOptions` is on |

## Changelog

- **v6.2.1 (2025-07-15)**: Updated shared-charts dependency.
- **v6.2.0 (2025-06-03)**: Fixed aggregation being removed on Plotly 3.0 upgrade (Plotly transforms reorganized).
- **v6.0.0 (2025-02-28)**: Updated Plotly.js library to version 3.0 (breaking change).
- **v5.1.0 (2024-10-28)**: Changed bundling to allow security/compliance package scanning.
- **v5.0.1 (2024-10-15)**: Fixed auto-resize when chart is inside a popup dialog.

## Open Questions

> Could not be determined from source code alone — requires human review
- [ ] For `lineStyle: "custom"`, what additional customization is available beyond `"lines+markers"` mode? The XML enum suggests a custom mode but the Plotly data maps it identically to `"lineWithMarkers"`.
- [ ] How are String and Enum y-attribute types rendered on the y-axis — as categorical labels or numeric encodings?
