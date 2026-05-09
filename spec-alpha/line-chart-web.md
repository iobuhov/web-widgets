# LineChart

## Purpose

The LineChart widget renders a Plotly.js line chart within a Mendix page, visualizing one or more data series as connected line traces. It supports static series (one fixed datasource per series) and dynamic series (multiple series derived from a single datasource partitioned by a `groupByAttribute`). The widget is suited for trending continuous data over a numeric or temporal X axis and integrates with the shared Mendix chart infrastructure for data loading, aggregation, and responsive sizing.

## User Scenarios

### [P1] Render a basic line chart

**Given** a developer has configured one or more static series with X and Y attribute mappings pointing to a Mendix datasource  
**When** the page loads and all datasource items are available  
**Then** a Plotly scatter trace is rendered for each series with `mode: "lines"`, connecting data points in order, inside a responsive container that resizes with its parent

#### Edge Cases
- The chart renders nothing (returns `null`) while any configured datasource is still loading (`status: "loading"`).
- Null attribute values are passed as `null` in the x/y arrays; Plotly renders these as gaps in the line.

### [P2] Render a line chart with data point markers

**Given** a series has `lineStyle` set to `"lineWithMarkers"`  
**When** the chart renders  
**Then** each data point is shown as a visible dot in addition to the connecting line (`mode: "lines+markers"`)

#### Edge Cases
- Marker color is independently configurable via `staticMarkerColor` (a per-item expression). If undefined, Plotly uses its default color palette.
- The marker color property is only exposed in Studio Pro when `lineStyle === "lineWithMarkers"`.

### [P3] Render dynamic (grouped) series

**Given** a series has `dataSet: "dynamic"` with a `groupByAttribute` configured  
**When** the datasource items are loaded  
**Then** one Plotly trace is produced per unique value of `groupByAttribute`, each with its own name derived from the `dynamicName` expression

#### Edge Cases
- If `dynamicName` evaluates to `null` or is unavailable for a group, the series name defaults to `"(empty)"`.
- Supported `groupByAttribute` types: String, Boolean, DateTime, Decimal, Enum, HashString, Integer, Long.

### [P4] Aggregate data before rendering

**Given** a series has an `aggregationType` other than `"none"` (e.g., `"avg"`, `"sum"`)  
**When** data points are processed  
**Then** points are aggregated by X value before being passed to Plotly, producing one Y value per unique X

#### Edge Cases
- `aggregationType: "none"` passes all data points to Plotly without modification.
- Aggregation was non-functional in v6.0.0–v6.1.x (Plotly 3.0 regression); fixed in v6.2.0.

### [P5] Tooltip and click interaction

**Given** a series has hover text configured  
**When** a user hovers over a data point  
**Then** the hover tooltip shows the configured text (`hoverinfo: "text"`)

**Given** a series has a click action configured  
**When** a user clicks a data point  
**Then** the Mendix action bound to that item is executed

#### Edge Cases
- If no item in the series has a non-empty hover text value, `hoverinfo` falls back to `"none"` (no tooltip).

## Functional Requirements

- FR-001: The widget MUST render a Plotly scatter trace with `type: "scatter"` for each configured series.
- FR-002: `lineStyle: "line"` MUST produce Plotly `mode: "lines"`. `lineStyle: "lineWithMarkers"` MUST produce `mode: "lines+markers"`.
- FR-003: `lineStyle: "custom"` MUST pass through with no `mode` set, delegating full control to `customSeriesOptions`.
- FR-004: `interpolation: "linear"` MUST produce `line.shape: "linear"`. `interpolation: "spline"` MUST produce `line.shape: "spline"`.
- FR-005: Both chart axes MUST be non-zoomable (`fixedrange: true`).
- FR-006: Both axes MUST display a zero-line (`zeroline: true`). Grid lines MUST use color `#d7d7d7`.
- FR-007: The chart MUST be `responsive: true`, resizing automatically with its container.
- FR-008: The widget MUST render `null` (nothing) while any datasource series is in loading state.
- FR-009: For `dataSet: "static"`, one Plotly trace MUST be produced per configured series entry.
- FR-010: For `dataSet: "dynamic"`, one Plotly trace MUST be produced per unique `groupByAttribute` value in the datasource.
- FR-011: `aggregationType` MUST be applied before Plotly rendering; `"none"` skips aggregation.
- FR-012: Null attribute values MUST be passed as `null` in x/y arrays, producing visible gaps in the line.
- FR-013: `Big` (big.js) numeric values MUST be converted to `number` via `.toNumber()` before being passed to Plotly.
- FR-014: The widget MUST apply CSS class `"widget-line-chart"` plus the user-configured `class` prop.
- FR-015: The widget MUST auto-resize correctly when placed inside a Mendix popup dialog.
- FR-016: `customLayout` and `customConfigurations` (JSON strings) MUST be applied as Plotly layout and config overrides when `enableAdvancedOptions` is `true`.
- FR-017: Per-series `customSeriesOptions` (JSON string) MUST be applied to the Plotly trace when `enableAdvancedOptions` is `true`.
- FR-018: Advanced options (`customLayout`, `customConfigurations`, `enableThemeConfig`, `customSeriesOptions`) MUST be hidden in Studio Pro when `enableAdvancedOptions` is `false`.
- FR-019: Validation MUST block configuration (error in Studio Pro) when a datasource is set on a series but X or Y attribute is missing.
- FR-020: The page explorer caption MUST show the first series datasource name (with "and N more" suffix when multiple series exist), falling back to `"Line chart"` when no series are defined.
- FR-021: The widget MUST support integration with the chart-playground widget via a `playground` slot when `showPlaygroundSlot` is `true`.

## Props Reference

| Name | Type | Default | Caption | Description |
|------|------|---------|---------|-------------|
| `lines` | List (series) | — | Lines | One or more data series to render. Each series configures its own datasource, attributes, style, and actions. |
| `lines[].dataSet` | Enum (`static` \| `dynamic`) | `static` | Data set | `static`: one series from one datasource. `dynamic`: multiple series from one datasource split by `groupByAttribute`. |
| `lines[].staticDataSource` | Datasource | — | Data source | Datasource for a static series. |
| `lines[].dynamicDataSource` | Datasource | — | Data source | Datasource for a dynamic series. |
| `lines[].groupByAttribute` | Attribute | — | Group by attribute | Attribute used to partition dynamic series into separate traces. |
| `lines[].xAttribute` | Attribute | — | X axis attribute | Data attribute mapped to the X axis. Required when datasource is set. |
| `lines[].yAttribute` | Attribute | — | Y axis attribute | Data attribute mapped to the Y axis. Required when datasource is set. |
| `lines[].aggregationType` | Enum (`none` \| `count` \| `sum` \| `avg` \| `min` \| `max` \| `median` \| `mode` \| `first` \| `last`) | `none` | Aggregation | Aggregation function applied to Y values grouped by X. |
| `lines[].interpolation` | Enum (`linear` \| `spline`) | `linear` | Interpolation | Controls line segment shape: straight segments or curved/smooth spline. |
| `lines[].lineStyle` | Enum (`line` \| `lineWithMarkers` \| `custom`) | `line` | Line style | `line`: line only. `lineWithMarkers`: line with data point dots. `custom`: fully custom via `customSeriesOptions`. |
| `lines[].staticLineColor` | Expression (string, per-item) | — | Line color | Expression resolving to a CSS color string for the trace line. |
| `lines[].staticMarkerColor` | Expression (string, per-item) | — | Marker color | Expression resolving to a CSS color string for the data point markers. Only shown when `lineStyle === "lineWithMarkers"`. |
| `lines[].onClickAction` | Action | — | On click action | Mendix action executed when a user clicks a data point. |
| `lines[].tooltipHoverText` | Expression (string, per-item) | — | Tooltip | Per-item hover text shown in the Plotly tooltip. |
| `lines[].customSeriesOptions` | String | `""` | Custom series options | JSON string merged into the Plotly trace object. Only effective when `enableAdvancedOptions` is `true`. |
| `xAxisLabel` | String | — | X axis label | Label displayed on the X axis. |
| `yAxisLabel` | String | — | Y axis label | Label displayed on the Y axis. |
| `showLegend` | Boolean | `true` | Show legend | Toggle the Plotly legend. |
| `gridLines` | Enum (`none` \| `horizontal` \| `vertical` \| `both`) | `none` | Grid lines | Which grid lines to display. |
| `widthUnit` | Enum (`percentage` \| `pixels`) | `percentage` | Width unit | Unit for the chart container width. |
| `width` | Integer | `100` | Width | Chart container width value. |
| `heightUnit` | Enum (`percentageOfWidth` \| `pixels` \| `percentageOfParent`) | `percentageOfWidth` | Height unit | Unit for chart container height. |
| `height` | Integer | `75` | Height | Chart container height value. |
| `enableAdvancedOptions` | Boolean | `false` | Enable advanced options | Unlocks `customLayout`, `customConfigurations`, `enableThemeConfig`, and per-series `customSeriesOptions`. Web platform only. |
| `customLayout` | String | — | Custom layout | JSON string merged into the Plotly layout object. Requires `enableAdvancedOptions`. |
| `customConfigurations` | String | — | Custom configurations | JSON string merged into the Plotly config object. Requires `enableAdvancedOptions`. |
| `showPlaygroundSlot` | Boolean | `false` | Show playground | Exposes a widget slot for the chart-playground widget. |
| `playground` | Widget slot | — | Playground | Slot for chart-playground integration. Only visible when `showPlaygroundSlot` is `true`. |

## Changelog

- **v6.2.1** (2025-07-15): Updated shared charts dependency (no behavioral change).
- **v6.2.0** (2025-06-03): Fixed aggregation broken by the Plotly 3.0 upgrade in v6.0.0. Aggregation (count, sum, avg, etc.) was non-functional on v6.0.0–v6.1.x.
- **v6.0.0** (2025-02-28): Upgraded Plotly.js to version 3.0 (introduced aggregation regression fixed in v6.2.0).

## Open Questions

> Could not be determined from source code alone — requires human review

- [ ] The `lineStyle: "custom"` mode sets no Plotly `mode` value and delegates entirely to `customSeriesOptions`. Confirm whether the intended use is fully documented or if there are additional platform-level constraints on what `customSeriesOptions` may override.
- [ ] `editorPreview.tsx` contains an alt text value of `"Bubble chart"` — an apparent copy-paste artifact. Confirm whether this is harmless (cosmetic only, design canvas) or should be corrected.
- [ ] Advanced options are described as disabled on the desktop platform (`platform === "desktop"`). Confirm the exact set of Mendix client targets on which `enableAdvancedOptions` is available.
