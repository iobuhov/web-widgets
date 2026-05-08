# BarChart

## Purpose

The Bar Chart widget renders a horizontal bar chart on a Mendix page using Plotly.js as the underlying rendering engine. It visualizes one or more data series as horizontal bars, supporting grouped and stacked display modes. The widget supports both static series (one datasource per series) and dynamic series (multiple series grouped from a single datasource), with optional client-side aggregation of Y values and per-bar color expressions. It is suited for category comparison, distribution visualization, and ranked data display in Mendix applications, including offline apps.

## User Scenarios

### [P1] Render a static data series as a grouped bar chart

**Given** a developer configures one or more series with `dataSet = "static"`, a datasource, X and Y attribute mappings, and `barmode` is `"group"` (default)  
**When** the widget loads and the datasource resolves  
**Then** the chart renders horizontal bars per category per series, grouped side-by-side when multiple series share the same X value; the Y-axis always starts at zero

#### Edge Cases

- The widget renders nothing (empty Fragment) while data is loading — there is no loading spinner.
- Null X or Y values produce gaps; `null` is passed to Plotly without aggregation unless an aggregation type is set.
- The Y-axis `rangemode` is always `"tozero"` — even when all bar values are positive, the axis starts at zero.

---

### [P2] Render stacked bars from multiple series

**Given** `barmode` is set to `"stack"` and multiple series are configured  
**When** the widget renders  
**Then** bars for each category are stacked vertically rather than placed side-by-side; the total height of the stack equals the sum of all series values for that category

#### Edge Cases

- Switching `barmode` between `"group"` and `"stack"` in the property panel immediately updates the structure-mode preview SVG in Studio Pro and the canvas preview in Studio — developers get real-time visual confirmation before publishing.
- There is no `"overlay"` barmode option, even though Plotly.js supports it. Custom layout JSON (`customLayout`) could override `barmode` to `"overlay"` as an unsupported workaround.

---

### [P3] Color bars using a per-bar expression

**Given** `barColor` (static or dynamic) is configured as an expression referencing an entity attribute (e.g., `$currentObject/StatusColor`)  
**When** the widget renders  
**Then** each bar is rendered with the color resolved from its corresponding entity object; bars without a resolved color fall back to Plotly's automatic color sequence

#### Edge Cases

- The `barColor` expression must evaluate to a valid CSS color string per bar object; invalid values are passed to Plotly without sanitization.
- Entity attribute references in the `barColor` expression are fully supported (restored in v3.1.3 after a regression in v4.0).

---

### [P4] Aggregate Y values for the same X category

**Given** `aggregationType` is set to a non-`"none"` value  
**When** multiple data points share the same X value  
**Then** their Y values are aggregated client-side (count/sum/avg/min/max/median/mode/first/last) before rendering

#### Edge Cases

- Null Y values are excluded from aggregation (not treated as zero).
- Aggregation is performed entirely in the browser after all data is fetched.
- Aggregation was broken by the Plotly 3.0 upgrade (v6.0.0) and restored in v6.2.0. Deployments on v6.0.x without upgrading to v6.2.0+ have non-functional aggregation.

---

### [P5] Trigger a click action on a bar

**Given** a `staticOnClickAction` or `dynamicOnClickAction` is configured for a series  
**When** the user clicks a bar  
**Then** the configured Mendix action is executed with the clicked item as context

#### Edge Cases

- Click action references are excluded from React.memo re-render comparison (`containerPropsEqual`), so changing a click action alone does not cause the chart to remount.

---

## Functional Requirements

- FR-001: The widget MUST render all bar traces with `orientation: "h"` (horizontal bars). There is no vertical orientation option.
- FR-002: The Y-axis MUST always have `rangemode: "tozero"`, ensuring the axis starts at zero regardless of data values.
- FR-003: Both X and Y axes MUST be non-zoomable (`fixedrange: true`); users cannot zoom the chart.
- FR-004: Grid color MUST be `#d7d7d7` when grid lines are enabled.
- FR-005: The Plotly toolbar MUST be hidden by default (`displayModeBar: false`).
- FR-006: The widget MUST support `barmode` values `"group"` and `"stack"` only. `"overlay"` is not an exposed option.
- FR-007: The `barmode` layout property MUST be applied per-render via `useMemo` so that changing `barmode` triggers a Plotly re-layout without recreating the full layout options object.
- FR-008: The `barColor` prop MUST accept a `ListExpressionValue<string>` — a per-bar color expression, not a static per-series color. Entity attribute references (e.g., `$currentObject/StatusColor`) MUST be accepted in the expression editor.
- FR-009: The widget MUST render nothing (empty Fragment) while data is loading. There is no loading indicator.
- FR-010: The widget MUST support offline Mendix applications (`offlineCapable="true"`).
- FR-011: The widget MUST auto-resize when its container is resized (e.g., inside a popup/modal dialog), implemented via a resize observer in `ChartWidget`.
- FR-012: When `aggregationType` is not `"none"`, Y values MUST be aggregated client-side using the selected function before passing data to Plotly. X values are used as string keys for grouping.
- FR-013: `Big` values (Decimal, Long, Integer Mendix attribute types) MUST be converted to JavaScript `number` via `.toNumber()` before passing to Plotly.
- FR-014: The structure-mode preview and Studio canvas preview MUST switch SVG images based on `barmode` — grouped vs. stacked images are distinct and update in real time as the property is changed.
- FR-015: The widget MUST support a playground slot for runtime chart editing via the chart-playground widget, controlled by `showPlaygroundSlot`.

## Props Reference

| Name | Type | Default | Caption | Description |
|------|------|---------|---------|-------------|
| `series` | List | — | Series | List of data series configurations. |
| `series.dataSet` | Enum (`static` \| `dynamic`) | — | Dataset | `"static"`: one datasource, one trace. `"dynamic"`: one datasource, multiple traces grouped by `groupByAttribute`. |
| `series.staticDataSource` | Datasource | — | Data source | Mendix datasource for a static series. |
| `series.dynamicDataSource` | Datasource | — | Data source | Mendix datasource for a dynamic series. |
| `series.groupByAttribute` | ListAttribute | — | Group by | Attribute used to split a dynamic datasource into separate bar traces. |
| `series.staticXAttribute` | ListAttribute | — | X axis | Attribute supplying X (category) values for a static series. |
| `series.staticYAttribute` | ListAttribute | — | Y axis | Attribute supplying Y (value) values for a static series. |
| `series.dynamicXAttribute` | ListAttribute | — | X axis | X attribute for a dynamic series. |
| `series.dynamicYAttribute` | ListAttribute | — | Y axis | Y attribute for a dynamic series. |
| `series.aggregationType` | Enum | `"none"` | Aggregation | Client-side aggregation: none, count, sum, avg, min, max, median, mode, first, last. |
| `series.staticBarColor` | ListExpression (String) | — | Bar color | Per-bar color expression for static series. Resolved per datasource object. Accepts entity attribute references. |
| `series.dynamicBarColor` | ListExpression (String) | — | Bar color | Per-bar color expression for dynamic series. |
| `series.staticOnClickAction` | Action | — | On click | Action executed when a static series bar is clicked. |
| `series.dynamicOnClickAction` | Action | — | On click | Action executed when a dynamic series bar is clicked. |
| `series.staticTooltipHoverText` | ListExpression (String) | — | Tooltip text | Custom hover text per bar (static series). |
| `series.dynamicTooltipHoverText` | ListExpression (String) | — | Tooltip text | Custom hover text per bar (dynamic series). |
| `series.customSeriesOptions` | String | `""` | Custom options | JSON string deep-merged into the Plotly trace configuration. Requires `enableAdvancedOptions`. |
| `barmode` | Enum (`"group"` \| `"stack"`) | `"group"` | Bar mode | `"group"`: bars placed side-by-side per category. `"stack"`: bars stacked for each category. |
| `xAxisLabel` | String | — | X axis label | Label displayed on the X axis. |
| `yAxisLabel` | String | — | Y axis label | Label displayed on the Y axis. |
| `showLegend` | Boolean | `true` | Show legend | Controls Plotly legend visibility. |
| `gridLines` | Enum | `"none"` | Grid lines | Controls grid line display on chart axes. |
| `widthUnit` | Enum | `"percentage"` | Width unit | Unit for the `width` property. |
| `width` | Integer | `100` | Width | Chart width in the selected unit. |
| `heightUnit` | Enum | `"percentageOfWidth"` | Height unit | Unit for the `height` property. |
| `height` | Integer | `75` | Height | Chart height in the selected unit. |
| `enableAdvancedOptions` | Boolean | `false` | Enable advanced options | Unlocks `customLayout`, `customConfigurations`, `enableThemeConfig`, and per-series `customSeriesOptions`. |
| `customLayout` | String | — | Custom layout | JSON string deep-merged into the Plotly layout configuration. |
| `customConfigurations` | String | — | Custom config | JSON string deep-merged into the Plotly configuration options. |
| `enableThemeConfig` | Boolean | `false` | Enable theme config | Loads a JSON config from the Mendix theme folder to further override chart options. |
| `showPlaygroundSlot` | Boolean | `false` | Show playground | Reveals the playground widget slot for runtime chart option editing. |
| `playground` | Widgets | — | Playground | Drop zone for the chart-playground widget. |

## Changelog

### [6.2.1] - 2025-07-15
- Changed: Updated shared-charts dependency (maintenance release).

### [6.2.0] - 2025-06-03
- Fixed: Aggregation (`aggregationType`) broken by Plotly 3.0 upgrade in v6.0.0 — restored in this release. Deployments on v6.0.x must upgrade to v6.2.0+ to recover aggregation functionality.

### [6.0.0] - 2025-02-28
- Changed: Upgraded Plotly.js dependency to version 3.0 (major version bump). Note: this introduced a regression in aggregation, fixed in v6.2.0.

## Open Questions

> Could not be determined from source code alone — requires human review

- [ ] Is horizontal-only orientation (`orientation: "h"`) an intentional design constraint, or is vertical bar chart support planned? There is no `orientation` prop exposed to developers.
- [ ] Is the `"overlay"` Plotly barmode intentionally excluded? It is a valid Plotly option but is not available through the widget property panel; developers can only access it via `customLayout` JSON override.
- [ ] The `editorPreview.tsx` alt text reads `"Bubble chart"` for all bar chart preview images — is this a copy-paste artifact that should be corrected?
- [ ] The horizontal orientation (`orientation: "h"`) is not covered by any unit test in `BarChart.spec.tsx`. Should a test be added to verify this invariant?
- [ ] Deployments on bar-chart-web v6.0.0 through v6.0.x (before v6.2.0) have non-functional aggregation. Is there a migration notice or upgrade advisory for affected users?
