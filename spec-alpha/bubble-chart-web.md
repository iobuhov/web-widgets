# BubbleChart

## Purpose

The BubbleChart widget renders a Plotly-based scatter chart in "markers" mode where each data point is a circle whose diameter encodes a third numeric dimension — the bubble size. It is intended for comparative data visualizations where three variables must be represented simultaneously: two position axes (X, Y) and one magnitude axis (size). Common use cases include market analysis (revenue vs. growth vs. market share), resource allocation (effort vs. impact vs. team size), and performance dashboards. The widget supports multiple data series, per-series aggregation, dynamic groupBy expansion, optional per-item click actions, and a full Plotly advanced options surface for custom configurations.

## User Scenarios

### [P1] Display a multi-series bubble chart with auto-scaled bubbles

**Given** two entries in `lines`, each with a static data source bound to a list of objects having numeric X, Y, and size attributes, and `autosize = true`  
**When** the page loads and data becomes available  
**Then** both series render as circle marker scatter series; the largest bubble in each series is scaled so that its diameter occupies approximately `sizeref`% of the chart's average dimension; series are rendered in the order they appear in `lines`

#### Edge Cases

- When `autosize = false`, `sizeref` acts as a direct scale factor: each data unit maps to a bubble radius of `1 / (sizeref / 100)` in Plotly units.
- When `lines` is empty or all data sources are still loading, the widget renders nothing — no placeholder or loading indicator.
- When a data item has a `null` size attribute, Plotly receives `null` for that point's size — the point is rendered as a zero-size marker.

---

### [P2] Color a series with a dynamic expression

**Given** a line has `staticMarkerColor` or `dynamicMarkerColor` configured as a list expression returning a color string  
**When** the chart renders  
**Then** the series marker color is set to the value returned by the expression for the first item in the series; all markers in the series share the same color

#### Edge Cases

- When no marker color is configured, the series uses Plotly's default color assignment.
- The marker color expression is evaluated per-item, but only the first item's value is used as the uniform series color — individual bubble color variation within a series is not supported.

---

### [P3] Use dynamic groupBy to auto-expand a single data source into multiple series

**Given** a line's `dataSet = "dynamic"` and `dynamicDataSource` and `groupByAttribute` are configured  
**When** the data source loads  
**Then** one chart series is created per distinct value of `groupByAttribute`; each group is named by its `groupByAttribute` value; groups are ordered by first appearance in the data source

#### Edge Cases

- When all items belong to one group, a single series is rendered.
- When a data item has a `null` groupBy attribute, it is treated as its own group with the name displayed as the null-equivalent string.

---

### [P4] Execute a per-item click action

**Given** a line has an `onClick` action configured  
**When** the user clicks a bubble  
**Then** the action is executed with the corresponding data source item as context

#### Edge Cases

- Click actions fire for each bubble independently — there is no chart-level click action.
- When no click action is configured on a line, clicking bubbles in that series is a silent no-op.

---

## Functional Requirements

- FR-001: The system MUST render each entry in `lines` as a Plotly scatter trace with `type: "scatter"`, `mode: "markers"`, and `symbol: ["circle"]`.
- FR-002: Axes MUST be fixed-range (`fixedrange: true`) — end users MUST NOT be able to zoom in or out on either axis.
- FR-003: Both axes MUST display a zero line and grid lines colored `#d7d7d7`.
- FR-004: Bubble size MUST use `sizemode: "diameter"` — size values are interpreted as bubble diameter, not area.
- FR-005: When `autosize = true`, the system MUST compute a `sizeref` such that the largest bubble occupies approximately `sizeref`% of the chart's average (width + height / 2) dimension.
- FR-006: When `autosize = false`, the system MUST use `sizeref` directly as a Plotly sizeref value.
- FR-007: The system MUST accept only Decimal, Long, and Integer attribute types for bubble size; string or datetime size attributes MUST be rejected at design time.
- FR-008: The system MUST support X and Y axis attributes of types: String, Enum, DateTime, Decimal, Integer, Long, AutoNumber.
- FR-009: The system MUST support per-series aggregation of type: none, count, sum, avg, min, max, median, mode, first, last.
- FR-010: The system MUST support dynamic groupBy: when `dataSet = "dynamic"` and `groupByAttribute` is configured, one Plotly trace MUST be created per distinct group value.
- FR-011: When a data source is configured for a line, X attribute, Y attribute, and size attribute MUST all be configured; Studio Pro MUST report configuration errors for any missing attribute.
- FR-012: The system MUST render nothing (no placeholder, no loading indicator) when all data sources are empty or still loading.
- FR-013: The chart MUST be responsive (`responsive: true` in Plotly config) — it MUST fill its configured container dimensions.
- FR-014: The widget MUST NOT require an entity context (`needsEntityContext` is absent).
- FR-015: The widget MUST function in offline-enabled Mendix applications (`offlineCapable = true`).
- FR-016: Advanced options (customLayout, customConfigurations, customSeriesOptions, enableThemeConfig) MUST be hidden by default and only visible when `enableAdvancedOptions = true`.
- FR-017: Studio Pro MUST show a configuration error (not a warning) when a data source is configured but a required axis or size attribute is missing.

## Props Reference

| Name | Type | Default | Caption | Description |
|------|------|---------|---------|-------------|
| `lines` | list of LinesType | _(required)_ | Data series | One or more bubble chart data series. Each entry maps to one Plotly scatter trace (or multiple when using dynamic groupBy). |
| `lines.dataSet` | enum | — | Data set | `"static"` (single list) or `"dynamic"` (grouped by attribute). |
| `lines.staticDataSource` | list source | — | Data source | Data source for static mode. |
| `lines.dynamicDataSource` | list source | — | Data source | Data source for dynamic mode. |
| `lines.groupByAttribute` | attribute | — | Group by | Attribute to group items into series (dynamic mode only). |
| `lines.xAttribute` | attribute | — | X axis attribute | Numeric, string, enum, or datetime attribute for the X position. |
| `lines.yAttribute` | attribute | — | Y axis attribute | Numeric, string, enum, or datetime attribute for the Y position. |
| `lines.staticSizeAttribute` | ListAttributeValue\<Big\> | — | Bubble size attribute | Decimal/Long/Integer attribute for bubble diameter (static mode). |
| `lines.dynamicSizeAttribute` | ListAttributeValue\<Big\> | — | Bubble size attribute | Decimal/Long/Integer attribute for bubble diameter (dynamic mode). |
| `lines.autosize` | boolean | true | Auto size | When true, normalizes largest bubble to `sizeref`% of chart dimension. When false, uses `sizeref` as a direct Plotly scale factor. |
| `lines.sizeref` | integer | 10 | Size reference | Percentage of chart dimension for the largest bubble (autosize=true), or direct Plotly sizeref (autosize=false). Hidden when autosize=true. |
| `lines.aggregationType` | enum | `"none"` | Aggregation | Aggregation function applied to Y values per X group: none/count/sum/avg/min/max/median/mode/first/last. |
| `lines.staticMarkerColor` | ListExpressionValue\<string\> | — | Marker color | Color expression for the series (static mode). Returns a CSS color string. First item's value used as uniform series color. |
| `lines.dynamicMarkerColor` | ListExpressionValue\<string\> | — | Marker color | Color expression for the series (dynamic mode). |
| `lines.hoverText` | expression | — | Hover text | Tooltip text displayed when hovering over a bubble. |
| `lines.onClickAction` | ListActionValue | — | On click | Action executed when a bubble is clicked, with the source item as context. |
| `lines.customSeriesOptions` | string | `""` | Custom series options | Raw Plotly trace JSON merged into this series' trace options. Only available when `enableAdvancedOptions = true`. |
| `xAxisLabel` | string | — | X axis label | Label displayed on the X axis. |
| `yAxisLabel` | string | — | Y axis label | Label displayed on the Y axis. |
| `showLegend` | boolean | true | Show legend | Renders the Plotly chart legend. |
| `gridLines` | enum | — | Grid lines | `"none"`, `"horizontal"`, `"vertical"`, or `"both"`. |
| `enableAdvancedOptions` | boolean | false | Enable advanced options | Shows advanced configuration properties: customLayout, customConfigurations, customSeriesOptions, enableThemeConfig. |
| `customLayout` | string | — | Custom layout | Raw Plotly layout JSON merged into chart layout. Requires `enableAdvancedOptions = true`. |
| `customConfigurations` | string | — | Custom configurations | Raw Plotly config JSON. Requires `enableAdvancedOptions = true`. |
| `enableThemeConfig` | boolean | false | Enable theme config | Loads Plotly config from theme folder JSON files. Requires `enableAdvancedOptions = true`. |
| `showPlaygroundSlot` | boolean | false | Show playground slot | Reveals the `playground` widget slot for developer configuration exploration. |
| `playground` | ReactNode | — | Playground | Widget slot rendered alongside the chart for interactive configuration (developer use). |
| `widthUnit` | enum | `"percentage"` | Width unit | `"percentage"` or `"pixels"`. |
| `width` | integer | — | Width | Widget width in the selected unit. |
| `heightUnit` | enum | — | Height unit | `"percentageOfWidth"`, `"pixels"`, or `"percentageOfParent"`. |
| `height` | integer | — | Height | Widget height in the selected unit. |

## Changelog

### [6.2.1] - 2025-07-15
- Updated shared charts dependency.

### [6.2.0] - 2025-06-03
- Fixed: Aggregate removal issue with Plotly 3.0.

### [6.0.0] - 2025-02-28
- Updated: Plotly.js to v3.0 (breaking upgrade).

## Open Questions

> Could not be determined from source code alone — requires human review

- [ ] What is the behavior when `aggregationType != "none"` is combined with bubble size — is the size attribute also aggregated (e.g. summed), or is it taken from the first item in the group?
- [ ] Can hover text reference the bubble size value? The draft shows hoverText as a general expression but does not confirm whether the size attribute's formatted value is accessible in the expression editor.
- [ ] When `autosize = false`, what are the practical min/max usable `sizeref` values? The property is declared as `integer` but there are no documented minimum/maximum constraints in the XML or validation code.
- [ ] Are there plans to support per-item (variable) marker colors within a single series, rather than using the first item's color for all bubbles?
- [ ] The version history skips 4.x entirely. Were those versions released under a different package name or distribution channel?
