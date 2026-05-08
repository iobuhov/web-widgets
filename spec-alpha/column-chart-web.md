# Column Chart (column-chart-web)

## Purpose

The Column Chart widget renders a vertical bar chart using Plotly.js (v3.0+) to visualize one or more data series from Mendix data sources. It supports static (fixed) and dynamic (grouped) data sets, per-item color expressions, multiple aggregation functions, and grouped or stacked bar layouts. The widget does not require an entity context and does not support offline use.

## User Scenarios

### [P1] Display a simple column chart from a static data source
**Given** a Column Chart with one series, `dataSet=static`, X and Y attributes configured  
**When** the page is loaded  
**Then** a vertical bar chart is rendered with one column per data point; the Y axis always starts at zero  

#### Edge Cases
- Both axes are non-zoomable (`fixedrange: true`); the user cannot pan or zoom the chart
- When `barColor` expression is not set, Plotly uses its default color cycle
- When `barColor` expression is set, each column is colored by evaluating the expression against the corresponding data item (e.g., `if Value > 100 then 'red' else 'green'`)

### [P2] Display a grouped column chart with dynamic data
**Given** a Column Chart with one or more series, `dataSet=dynamic`, a `groupByAttribute` configured  
**When** data is loaded  
**Then** the chart groups data points by the `groupByAttribute` value and renders one column group per X value, with `barmode=group` (default)  

#### Edge Cases
- `groupByAttribute` accepts: String, Boolean, DateTime, Decimal, Enum, HashString, Integer, Long
- Switching between static and dynamic modes in Studio Pro completely changes the visible configuration properties

### [P3] Stack columns with `barmode=stack`
**Given** a Column Chart with multiple series and `barmode=stack`  
**When** data is loaded  
**Then** the columns are stacked vertically; the first series is drawn at the base and subsequent series are drawn on top  

#### Edge Cases
- Series ordering in Studio Pro affects the stacking order: the first series is at the bottom
- The Y axis still starts at zero when stacked

### [P4] Aggregate data values
**Given** a Column Chart with an `aggregationType` other than "none" configured  
**When** multiple data points share the same X value  
**Then** the widget aggregates the Y values per X group using the configured function  

#### Edge Cases
- Available aggregation types: none, count, sum, avg, min, max, median, mode, first, last
- Aggregation is performed by `@mendix/shared-charts` before passing data to Plotly

### [P5] Handle click actions on columns
**Given** a Column Chart with an `onClickAction` configured on a series  
**When** the user clicks a column  
**Then** the configured action executes for the clicked data point's Mendix object  

#### Edge Cases
- Click actions use `ListActionValue` — each data point triggers its own per-item action
- Click action configuration is separate for static and dynamic data sets

## Functional Requirements

- FR-001: The widget MUST render vertical bars using Plotly type `"bar"` with `orientation: "v"`.
- FR-002: The Y axis MUST always start at zero (`rangemode: "tozero"`).
- FR-003: Both X and Y axes MUST be non-zoomable (`fixedrange: true`).
- FR-004: When `barColor` is a list expression, the resolved value MUST be applied to `marker.color` per data point; when undefined, `marker.color` MUST be left undefined (Plotly default cycle applies).
- FR-005: The widget MUST support 10 aggregation types: none, count, sum, avg, min, max, median, mode, first, last.
- FR-006: The widget MUST support `barmode` values: `"group"` (default) and `"stack"`.
- FR-007: Series ordering MUST determine column stacking order: the first series is drawn at the lowest Z-order.
- FR-008: In Studio Pro, when `dataSet=static`, all dynamic-mode properties (dynamicDataSource, groupByAttribute, dynamicName, etc.) MUST be hidden, and vice versa.
- FR-009: In Studio Pro, X and Y attribute validation MUST produce errors when a data source is configured but attributes are not set, preventing app publishing.
- FR-010: When `advancedOptions=false` (default), the properties customLayout, customConfigurations, enableThemeConfig, and customSeriesOptions MUST be hidden.
- FR-011: The widget MUST auto-resize when placed inside a Mendix popup dialog.
- FR-012: The widget MUST work in both the Mendix classic (Dojo) and React clients.
- FR-013: The widget MUST NOT require an entity context (`needsEntityContext` is not set).

## Props Reference

| Name | Type | Default | Caption | Description |
|------|------|---------|---------|-------------|
| `series` | `SeriesType[]` | — | Series | List of data series. Each series configures its own data source, attributes, aggregation, color, and click action. |
| `barmode` | `"group"` \| `"stack"` | `"group"` | Bar mode | Layout of multiple series: group (side by side) or stack (stacked vertically). |
| `xAxisLabel` | `string` | — | X axis label | Optional label shown below the X axis. |
| `yAxisLabel` | `string` | — | Y axis label | Optional label shown to the left of the Y axis. |
| `showLegend` | `boolean` | `true` | Show legend | Displays the series legend. |
| `gridLines` | `GridLinesEnum` | `"none"` | Grid lines | Controls grid line display. |
| `advancedOptions` | `boolean` | `false` | Advanced options | Exposes customLayout, customConfigurations, enableThemeConfig, and per-series customSeriesOptions. |
| `showPlaygroundSlot` | `boolean` | — | Show playground slot | Enables the Chart Playground widget slot for developer mode editing. |
| `widthUnit` | `"percentage"` \| `"pixels"` | `"percentage"` | Width unit | Unit for the width dimension. |
| `width` | `number` | `100` | Width | Width value. |
| `heightUnit` | `"percentage"` \| `"pixels"` \| `"percentageOfParent"` | `"percentage"` | Height unit | Unit for the height dimension. |
| `height` | `number` | `75` | Height | Height value. |

**Series object properties:**

| Name | Type | Caption | Description |
|------|------|---------|-------------|
| `dataSet` | `"static"` \| `"dynamic"` | Data set | Static: fixed list; Dynamic: grouped by attribute. |
| `staticDataSource` / `dynamicDataSource` | `ListValue` | Data source | The Mendix list providing data. |
| `groupByAttribute` | `ListAttributeValue` | Group by | Attribute used to group dynamic data. Accepts String, Boolean, DateTime, Decimal, Enum, HashString, Integer, Long. |
| `staticXAttribute` / `dynamicXAttribute` | `ListAttributeValue` | X attribute | Attribute mapped to the X axis. Accepts String, DateTime, Decimal, Integer, Long, AutoNumber. |
| `staticYAttribute` / `dynamicYAttribute` | `ListAttributeValue` | Y attribute | Attribute mapped to the Y axis. Accepts String, DateTime, Decimal, Integer, Long, AutoNumber. |
| `aggregationType` | `AggregationEnum` | Aggregation | One of: none, count, sum, avg, min, max, median, mode, first, last. |
| `staticBarColor` / `dynamicBarColor` | `ListExpressionValue<string>` | Column color | Expression evaluated per data item; sets `marker.color`. Supports conditional expressions. |
| `staticOnClickAction` / `dynamicOnClickAction` | `ListActionValue` | On click | Per-item action executed when a column is clicked. |
| `tooltipHoverText` | `ListExpressionValue<string>` | Tooltip text | Custom hover text per data point. |
| `staticName` / `dynamicName` | `string` / `ListExpressionValue<string>` | Series name | Name displayed in the legend. |

## Changelog

**6.2.1** (2025-07-15) — Updated `@mendix/shared-charts` dependency.

**6.2.0** — Fixed aggregate calculation broken by Plotly 3.0 upgrade.

**6.0.0** — Updated Plotly.js to v3.0 (breaking change).

**5.0.1** — Fixed chart not resizing inside Mendix popup dialogs.

## Open Questions

> Could not be determined from source code alone — requires human review

- [ ] What is the expected behavior when multiple series with `aggregationType != "none"` and different `groupByAttribute` values are combined?
- [ ] Is there a documented maximum number of data points before Plotly performance degrades in the Mendix client?
- [ ] The `heightUnit: "percentageOfParent"` option — what is its exact parent element (the widget container, page, etc.)?
