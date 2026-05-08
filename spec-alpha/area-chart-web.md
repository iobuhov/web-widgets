# AreaChart

## Purpose

The Area Chart widget renders a filled scatter (area) chart on a Mendix page using plotly.js as the underlying rendering library. It visualizes one or more data series as lines with filled regions below them, enabling comparison of cumulative or proportional quantities over a continuous axis. The widget supports both static series (one datasource per series) and dynamic series (multiple series grouped from a single datasource). It is suited for trend visualization, volume comparison, and stacked data analysis in Mendix applications, including offline apps.

## User Scenarios

### [P1] Render a static data series as an area chart
**Given** a developer configures one or more series with `dataSet = "static"`, a datasource, and X/Y attribute mappings  
**When** the widget loads and the datasource resolves  
**Then** the chart renders a filled area trace per series, with fill color applied below (or to the previous series for stacked appearance)

#### Edge Cases
- The widget renders nothing (empty Fragment) while data is loading — there is no loading spinner.
- Null X or Y values are preserved as plotly `null`, resulting in gaps in the line (unless overridden by the default `connectgaps: true` which connects across null points).
- Color expressions are resolved from the first item in the datasource; if the expression is unavailable, no color override is applied.

### [P2] Render dynamic series grouped by attribute
**Given** `dataSet = "dynamic"` and a `groupByAttribute` is configured  
**When** the datasource resolves  
**Then** each distinct value of `groupByAttribute` produces a separate area series, each rendered with its own fill and line color

#### Edge Cases
- If any item's `groupByAttribute` value is in `Loading` state, the entire dynamic series returns no data until all values resolve.
- A group whose name expression is empty for all items is labeled `"(empty)"`.

### [P3] Aggregate Y values for the same X
**Given** `aggregationType` is set to a non-`"none"` value  
**When** multiple data points share the same X value  
**Then** their Y values are aggregated client-side (count/sum/avg/min/max/median/mode/first/last) before rendering

#### Edge Cases
- Null Y values are excluded from aggregation (not treated as zero).
- Aggregation happens entirely in the browser after all data is fetched.
- `"mode"` returns the most frequent value; ties are resolved by insertion order.

### [P4] Click action on a data point
**Given** a `staticOnClickAction` or `dynamicOnClickAction` is configured for a series  
**When** the user clicks a data point on the chart  
**Then** the configured Mendix action is executed with the clicked item as context

#### Edge Cases
- Click action references are excluded from React.memo re-render comparison, so changing a click action alone does not cause the chart to remount.

### [P5] Advanced: custom layout and series options
**Given** `enableAdvancedOptions` is `true` and a JSON string is provided in `customLayout` or `customSeriesOptions`  
**When** the widget renders  
**Then** the JSON is deep-merged over the default plotly layout and series options, with widget-specific options taking precedence over defaults and theme folder options taking precedence over both

#### Edge Cases
- If `enableThemeConfig` is also `true`, a JSON config file from the Mendix theme folder is loaded asynchronously and further overrides modeler-specified options.
- Advanced options are always visible in Studio Pro; they are hidden on the desktop/Studio platform.

## Functional Requirements

- FR-001: The widget MUST render a filled scatter trace per series using `fill: "tonexty"` (fills to the previous trace, or to the x-axis for the first series).
- FR-002: Both X and Y axes MUST be non-zoomable by default (`fixedrange: true`); users cannot zoom into the chart.
- FR-003: Both axes MUST have `zeroline: true` and grid color `#d7d7d7`.
- FR-004: The plotly toolbar MUST be hidden by default (`displayModeBar: false`).
- FR-005: Double-click MUST NOT reset zoom (`doubleClick: false`).
- FR-006: Data gaps (null X/Y) MUST be connected by default (`connectgaps: true`).
- FR-007: The widget MUST render nothing (empty Fragment) while data is loading; there is no loading indicator.
- FR-008: The widget MUST support three line styles: `"line"` (lines only), `"lineWithMarkers"` (lines with markers), and `"custom"` (no mode override — defers to `customSeriesOptions`).
- FR-009: The widget MUST support offline Mendix applications (`offlineCapable="true"`).
- FR-010: The widget MUST re-render correctly when its container is resized (e.g., inside a popup dialog), implemented via a resize observer dispatch.
- FR-011: When `aggregationType` is not `"none"`, the widget MUST aggregate Y values client-side before passing data to plotly. X values are used as string keys for grouping.
- FR-012: `Big` values (Decimal, Long, Integer Mendix attribute types) MUST be converted to JavaScript `number` via `.toNumber()` before passing to plotly.
- FR-013: When `hoverinfo` is not configured, `hoverinfo: "none"` MUST be set. When custom hover text is provided and at least one item has non-empty text, `hoverinfo: "text"` MUST be used.
- FR-014: The widget MUST support a playground slot for runtime chart editing via the chart-playground widget, controlled by `showPlaygroundSlot`.

## Props Reference

| Name | Type | Default | Caption | Description |
|------|------|---------|---------|-------------|
| `series` | List | — | Series | List of data series configurations. |
| `series.dataSet` | Enum (`static` \| `dynamic`) | — | Dataset | `"static"`: one datasource, one series. `"dynamic"`: one datasource, multiple series grouped by `groupByAttribute`. |
| `series.staticDataSource` | Datasource | — | Data source | Mendix datasource for a static series. |
| `series.dynamicDataSource` | Datasource | — | Data source | Mendix datasource for a dynamic series. |
| `series.groupByAttribute` | ListAttribute | — | Group by | Attribute used to split a dynamic datasource into separate series. Supports String, Boolean, DateTime, Decimal, Enum, HashString, Integer, Long. |
| `series.xAttribute` | ListAttribute | — | X axis | Attribute supplying X values. Supports String, Date, Big (Decimal/Long/Integer). |
| `series.yAttribute` | ListAttribute | — | Y axis | Attribute supplying Y values. Supports String, Date, Big. |
| `series.aggregationType` | Enum | `"none"` | Aggregation | Aggregation function applied to Y values with the same X key: none, count, sum, avg, min, max, median, mode, first, last. |
| `series.interpolation` | Enum (`linear` \| `spline`) | — | Line shape | `"linear"`: straight segments. `"spline"`: curved/smooth segments. |
| `series.lineStyle` | Enum (`line` \| `lineWithMarkers` \| `custom`) | — | Line style | Controls plotly trace mode. |
| `series.staticLineColor` | Expression (String) | — | Line color | CSS color expression for the line. Resolved from the first datasource item. |
| `series.dynamicLineColor` | Expression (String) | — | Line color | Dynamic color expression per group. |
| `series.staticMarkerColor` | Expression (String) | — | Marker color | Marker color for static series. Only relevant when `lineStyle = "lineWithMarkers"`. |
| `series.dynamicMarkerColor` | Expression (String) | — | Marker color | Marker color per group for dynamic series. Only relevant when `lineStyle = "lineWithMarkers"`. |
| `series.staticFillColor` | Expression (String) | — | Fill color | Fill color for the area below the line, static series. |
| `series.dynamicFillColor` | Expression (String) | — | Fill color | Fill color per group, dynamic series. |
| `series.staticOnClickAction` | Action | — | On click | Action executed when a static series data point is clicked. |
| `series.dynamicOnClickAction` | Action | — | On click | Action executed when a dynamic series data point is clicked. |
| `series.tooltipHoverText` | Expression (String) | — | Tooltip text | Custom hover text per data point. |
| `series.customSeriesOptions` | String | `""` | Custom options | JSON string deep-merged into the plotly trace config. Requires `enableAdvancedOptions`. |
| `xAxisLabel` | String | — | X axis label | Label displayed on the X axis. |
| `yAxisLabel` | String | — | Y axis label | Label displayed on the Y axis. |
| `showLegend` | Boolean | — | Show legend | Controls plotly legend visibility. |
| `gridLines` | Boolean | — | Grid lines | Controls grid line visibility on both axes. |
| `widthUnit` | Enum (`percentage` \| `pixels`) | `"percentage"` | Width unit | Unit for the `width` property. |
| `width` | Integer | `100` | Width | Chart width in the selected unit. |
| `heightUnit` | Enum (`percentageOfWidth` \| `pixels` \| `percentageOfParent`) | `"percentageOfWidth"` | Height unit | Unit for the `height` property. |
| `height` | Integer | `75` | Height | Chart height in the selected unit. |
| `enableAdvancedOptions` | Boolean | `false` | Enable advanced options | Unlocks `customLayout`, `customConfigurations`, `enableThemeConfig`, and per-series `customSeriesOptions`. |
| `customLayout` | String | — | Custom layout | JSON string deep-merged into the plotly layout config. |
| `customConfigurations` | String | — | Custom config | JSON string deep-merged into the plotly configuration options. |
| `enableThemeConfig` | Boolean | `false` | Enable theme config | Loads a JSON config from the Mendix theme folder to override chart options. |
| `showPlaygroundSlot` | Boolean | `false` | Show playground | Reveals the playground widget slot for runtime chart option editing. |
| `playground` | Widgets | — | Playground | Drop zone for the chart-playground widget. |

## Changelog

- **v6.2.0 (2025-06-03):** Fixed aggregate function being removed on plotly 3.0 upgrade (regression fix).
- **v6.0.0 (2025-02-28):** Upgraded plotly.js to version 3.0 (major dependency bump).
- **v5.0.1 (2024-10-15):** Fixed widget not auto-resizing inside a popup dialog (resize observer fix in ChartWidget).

## Open Questions

> Could not be determined from source code alone — requires human review

- [ ] The `editorPreview.tsx` contains a copy-paste error: the alt text for the chart image reads "Bubble chart" instead of "Area chart". Should this be corrected?
- [ ] Changelog versions 4.x and 5.0.0 are absent — are these entries intentionally omitted or missing from the changelog?
- [ ] `connectgaps: true` is a fixed default. Is there a planned or desired opt-out mechanism for showing explicit gaps in the area line?
- [ ] The first render cycle with available data produces an empty chart before the data-loading `useEffect` fires (lazy state initialization). Is this flash acceptable for production use?
