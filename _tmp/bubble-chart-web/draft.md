# Draft: bubble-chart-web

Widget package: `@mendix/bubble-chart-web` v6.2.1  
Source: `packages/pluggableWidgets/bubble-chart-web/`

---

## src/BubbleChart.tsx

**1. Purpose of this file?**
Container component and Mendix widget entry point for the bubble chart. Maps props to Plotly scatter chart configuration with "markers" mode, adds bubble-specific size calculation, and delegates rendering to the shared `ChartWidget`.

**2. What kind of logic is described in this file?**
Uses `usePlotChartDataSeries` from `@mendix/shared-charts/main` to transform `props.lines` into Plotly trace data. For each line, the `SeriesMapper` callback: (1) extracts bubble sizes from the size attribute (handling both `Big` and plain number values); (2) calls `calculateSizeRef()` to compute Plotly `sizemode`/`sizeref` values; (3) optionally resolves per-item marker color from `staticMarkerColor` / `dynamicMarkerColor` expressions; (4) returns `{ type: "scatter", mode: "markers", marker: { color, symbol: ["circle"], size, ...markerOptions } }`. The component is wrapped in `memo` with a custom equality function using `flatEqual` / `traceEqual` to prevent unnecessary re-renders.

**3. What part of behavior can be documented from this file?**
Bubbles are always circles (`symbol: ["circle"]`). Marker color is optional — when not configured, `color` is `undefined` (uses Plotly default). Axis ranges are fixed (`fixedrange: true`) — users cannot zoom in/out on either axis. Both axes show a zero line and grid colored `#d7d7d7`. The chart is responsive (`configOptions.responsive: true`). When `chartBubbles` is `null` (data not yet loaded), an empty array is passed to `ChartWidget` which renders nothing.

**4. Is it user-facing?**
Yes — the bubble chart visualization is directly user-visible.

**5. What new did you learn from this file?**
The custom memo equality function compares each prop with `defaultEqual`, but compares the `lines` array with `flatEqual(traceEqual)` — a deep chart-aware equality check for trace data. This prevents the chart from re-rendering when the Mendix data source provides a new list with the same content. The `Big` type for bubble sizes is explicitly converted: `Big(value).toNumber()` — this handles both Decimal Mendix attributes and already-numeric values.

---

## src/BubbleChart.xml

**1. Purpose of this file?**
Widget descriptor for the bubble chart. Declares identity, all configurable properties in a `lines` object list (each line represents a data series), chart-level settings, and system properties.

**2. What kind of logic is described in this file?**
Per-series (in `lines` list): `dataSet` (static/dynamic), data sources (staticDataSource / dynamicDataSource with groupBy), X/Y axis attributes (accepting String, Enum, DateTime, Decimal, Integer, Long, AutoNumber), `aggregationType` (none/count/sum/avg/min/max/median/mode/first/last), `staticSizeAttribute`/`dynamicSizeAttribute` (Decimal/Long/Integer only), `autosize` (boolean, default true), `sizeref` (integer, default 10), tooltip hover text, marker color expression (String return type), click action, and `customSeriesOptions` (multiline string). Chart-level: `enableAdvancedOptions`, `showPlaygroundSlot`, `playground` (widget slot), `xAxisLabel`, `yAxisLabel`, `showLegend` (default true), `gridLines` (none/horizontal/vertical/both). Dimensions: widthUnit/width/heightUnit/height. Advanced: `enableThemeConfig`, `customLayout`, `customConfigurations`.

**3. What part of behavior can be documented from this file?**
`offlineCapable="true"`. No `needsEntityContext` — widget can be placed anywhere. Bubble size accepts only numeric types (Decimal/Long/Integer) — string or date attributes cannot be bubble size. Multiple series (multiple `lines` entries) can be configured, with the first line drawn lowest. Aggregation is per-series, not chart-wide.

**4. Is it user-facing?**
Not directly. Developer configuration in Studio/Studio Pro.

**5. What new did you learn from this file?**
The `playground` property is of type `widgets` — it accepts child widgets to be displayed in the "playground" slot (a developer/debug UI for testing chart configurations). The `showPlaygroundSlot` boolean controls whether this slot is shown in Studio Pro configuration. The `customSeriesOptions` property is a multiline string accepting raw Plotly trace JSON options, giving developers direct access to Plotly's configuration surface.

---

## typings/BubbleChartProps.d.ts

**1. Purpose of this file?**
Auto-generated TypeScript types from `BubbleChart.xml`. Defines the `LinesType` interface (per-series config) and `BubbleChartContainerProps` (full widget props).

**2. What kind of logic is described in this file?**
`LinesType`: `staticSizeAttribute` and `dynamicSizeAttribute` are `ListAttributeValue<Big>` — Mendix numeric attributes are exposed as `Big` (big.js arbitrary-precision) values. `staticMarkerColor` and `dynamicMarkerColor` are `ListExpressionValue<string>`. Click actions are `ListActionValue` (per-item actions). `customSeriesOptions` is `string` (always present, empty string when not configured). `BubbleChartContainerProps`: `lines` is `LinesType[]`. Preview props use nullable numbers (`number | null`) for `sizeref`.

**3. What part of behavior can be documented from this file?**
`staticSizeAttribute` is typed as `ListAttributeValue<Big>` — all three numeric types (Decimal/Long/Integer) are mapped to `Big` in the Mendix framework. The bubble chart container explicitly converts `Big` values to JavaScript `number` via `Big(value).toNumber()`. `customSeriesOptions` is always a string (never undefined) — empty string means "no custom options."

**4. Is it user-facing?**
No. Type declarations only.

**5. What new did you learn from this file?**
`aggregationType` in `LinesType` is required (not optional) with type `AggregationTypeEnum` — it always has a value, defaulting to `"none"`. The `playground` prop in the container is `ReactNode` (can be any React content). In preview props, it is `{ widgetCount: number; renderer: ComponentType<...> }` — a Studio Pro-specific type for widget slot preview rendering.

---

## src/utils/index.ts

**1. Purpose of this file?**
Utility functions for computing Plotly bubble size reference values from widget configuration and chart dimensions.

**2. What kind of logic is described in this file?**
`getMarkerSizeReference(props, markerSize, dimensions)`: when `autosize=true`, computes a `sizeref` that scales the largest bubble to `sizeref%` of the average of chart width and height — `maxSize / (avgDimension / (sizeref/100))`. When `autosize=false`, uses `sizeref` directly as a percentage: `1 / (sizeref / 100)`. `calculateSizeRef(series, marker, dimensions)`: calls `getMarkerSizeReference` and returns `{ sizemode: "diameter", sizeref }`.

**3. What part of behavior can be documented from this file?**
Sizemode is always `"diameter"` — Plotly interprets `size` values as bubble diameter (not area). The auto-scale algorithm normalizes the largest bubble to occupy `sizeref%` of the average chart dimension. When `sizeref=10`, the largest bubble occupies roughly 10% of the chart's average dimension. When `autosize=false` and `sizeref=10`, the computed sizeref is `1 / 0.1 = 10` — each data-unit of size is 10x larger visually.

**4. Is it user-facing?**
No. Internal calculation utility.

**5. What new did you learn from this file?**
The auto-scale formula `Math.round(sizeref * 1000) / 1000` rounds to 3 decimal places to avoid floating-point noise in the Plotly sizeref value. The `sizeref` widget prop is interpreted as a percentage (0–100 scale), not a direct Plotly sizeref — Plotly's sizeref is the inverse percentage. This means a higher `sizeref` widget value = smaller Plotly sizeref = larger bubbles.

---

## src/BubbleChart.editorConfig.ts (shared-charts local dep)

**1. Purpose of this file?**
Design-time property visibility rules, structure preview, check validation, and page explorer caption for the bubble chart.

**2. What kind of logic is described in this file?**
`getProperties()`: hides `playground` widget slot when `showPlaygroundSlot=false`; hides static/dynamic paired properties based on `dataSet` setting per line; hides `customSeriesOptions` and chart-level advanced options when `enableAdvancedOptions=false` (web platform); hides `sizeref` when `autosize=true`; transforms groups into tabs on web. `getPreview()`: renders a `RowLayout` containing a static chart image (375px) and optionally a legend image (85px), wrapped with `withPlaygroundSlot`. `check()`: validates that when a data source is configured for a line, X attribute, Y attribute, and size attribute are all also configured (errors for each missing attribute). `getCustomCaption()`: shows the first data source name + "and N more" count.

**3. What part of behavior can be documented from this file?**
When a data source is set for a line but X, Y, or size attribute is missing, Studio Pro reports a configuration error (blocking publish). This makes all three data attributes required if a data source is configured. The `check()` function uses `checkSlot(values)` from `@mendix/shared-charts/preview` to validate the playground slot configuration.

**4. Is it user-facing?**
No. Design-time only.

**5. What new did you learn from this file?**
Advanced options (customLayout, customConfigurations, enableThemeConfig, customSeriesOptions) are all hidden by default — only shown when `enableAdvancedOptions=true`. The `withPlaygroundSlot()` wrapper in the preview integrates a Plotly configuration playground UI when `showPlaygroundSlot=true`, allowing developers to experiment with chart configurations at design time.

---

## src/BubbleChart.editorPreview.tsx

**1. Purpose of this file?**
Renders the Studio Pro design canvas preview using the shared `ChartPreview` component with static bubble chart and legend images.

**2. What kind of logic is described in this file?**
`preview()` renders `<ChartPreview>` from `@mendix/shared-charts/preview` with a `PlotImage` (light-mode bubble chart SVG) and a `PlotLegend` (light-mode legend SVG).

**3. What part of behavior can be documented from this file?**
The design canvas always shows the light-mode preview image, regardless of the user's dark/light mode setting. The actual dark/light mode SVG switching for the `getPreview()` structure preview (in `editorConfig.ts`) is handled separately. The preview reuses the shared chart preview component for consistent appearance across all chart widgets.

**4. Is it user-facing?**
No. Design-time preview only.

**5. What new did you learn from this file?**
The `BubbleChart.editorPreview.tsx` file uses only light-mode assets (`BubbleChart.light.svg`) for the design canvas preview, while `editorConfig.ts` uses both light and dark assets for the structure preview panel. This means in the design canvas the bubble chart preview is always shown in light mode.

---

## src/__tests__/BubbleChart.spec.tsx

**1. Purpose of this file?**
Unit tests verifying that the bubble chart correctly produces Plotly scatter trace data with the appropriate mode and marker configuration.

**2. What kind of logic is described in this file?**
Three tests: (1) verifies `ChartWidget` receives data with `type: "scatter"` and `mode: "markers"`; (2) verifies `staticMarkerColor` expression value ("red") is passed to the marker `color` field, and `undefined` color for series without a marker color; (3) verifies aggregation rendering (two series with different `aggregationType` values produce two separate data series in the chart).

**3. What part of behavior can be documented from this file?**
The marker color per-series: a series with `staticMarkerColor` set to a list expression returning "red" produces `marker.color = "red"` in the Plotly trace. A series without marker color produces `marker.color = undefined`. The chart renders each `lines` entry as one data series (static case) or multiple series (dynamic case with groupBy).

**4. Is it user-facing?**
No. Tests only.

**5. What new did you learn from this file?**
`ChartWidget` is fully mocked in unit tests — tests do not exercise Plotly rendering. The `setupBasicBubbleSeries` helper adds `autosize: true` and `sizeref: 10` defaults. The `listExpression(() => "red")` test helper from `@mendix/widget-plugin-test-utils` creates a mock `ListExpressionValue<string>` that returns the same value for all items.

---

## packages/shared/charts/src/hooks/usePlotChartDataSeries.ts (local dependency)

**1. Purpose of this file?**
Shared hook that transforms Mendix data series (with static/dynamic data sources, attribute values, groupBy, and aggregation) into Plotly `PlotData` trace arrays.

**2. What kind of logic is described in this file?**
`usePlotChartDataSeries(series, mapSerie)` calls `loadStaticSeries` or `loadDynamicSeries` per element in `series`. Static: reads items from `staticDataSource`, extracts X/Y values (converting `Big` to number), builds tooltip hover arrays, then applies aggregation if configured; calls `mapSerie` to get chart-type-specific props (e.g., marker for bubble chart). Dynamic: groups items by `groupByAttribute` value (handling Date equality via `.getTime()`, Big equality via `.eq()`), creates one series per group. Returns `null` while loading (data source items not yet available).

**3. What part of behavior can be documented from this file?**
X/Y values that are `null` (unavailable attribute) are pushed as `null` into the data arrays — Plotly handles gaps in data. The first non-empty `dynamicName` value per group is used as the group's series name. The `getExpressionValue` helper returns the value from the first item in `dataSourceItems` — for per-item expressions like `staticMarkerColor`, only the first item's value is used as the uniform series color.

**4. Is it user-facing?**
No. Internal shared chart hook.

**5. What new did you learn from this file?**
The `hoverinfo` field is set to `"text"` when any hover text is non-empty, and `"none"` when all hover text values are empty/undefined. This means if even one item has hover text, all items show their hover text (items with undefined hover text show nothing). The `getExpressionValue` function iterates items and returns the FIRST available value — this is by design for expressions like `markerColor` where all items in a series typically return the same color value.

---

## packages/shared/charts/src/components/ChartWidget.tsx (local dependency)

**1. Purpose of this file?**
Shared component that assembles Plotly chart configuration from modeler options, theme folder configs, and custom JSON overrides, then renders the chart with proper dimensions.

**2. What kind of logic is described in this file?**
Merges layout options from three sources: modeler config (legend, axis labels, grid lines), widget-specific layout options (fixed from `bubbleChartLayoutOptions`), and theme folder config (optional). Same merge for config and series options. Uses `useDispatchResizeObserver` to notify the Mendix platform when the chart dimensions change. Renders nothing (`<Fragment />`) when `data.length === 0`. Renders a container div with computed dimensions containing the `<Chart>` component.

**3. What part of behavior can be documented from this file?**
When data is empty (still loading), the widget renders nothing — no placeholder or loading indicator. Chart dimensions use the shared `getDimensions()` utility supporting `percentage`, `pixels`, and `percentageOfWidth`/`percentageOfParent` height units. The `key={data.length}` prop on `<Chart>` forces a full remount when the number of data series changes — this resets Plotly's internal state when series are added/removed.

**4. Is it user-facing?**
No. Internal shared component.

**5. What new did you learn from this file?**
`useDispatchResizeObserver` dispatches a resize event when the chart container changes size. This is a Mendix-specific hook that notifies the platform of widget dimension changes (for layout recalculation). The `key={data.length}` approach is a deliberate choice to remount Plotly when series count changes, avoiding Plotly's incremental update behavior which can leave ghost traces.

---

## CHANGELOG.md

**1. Purpose of this file?**
Version history for bubble-chart-web.

**2. What kind of logic is described in this file?**
v6.2.1 (2025-07-15): Updated shared charts dependency. v6.2.0 (2025-06-03): Fixed aggregate removal issue with Plotly 3.0. v6.0.0 (2025-02-28): Updated Plotly.js to v3.0. v5.1.0 (2024-10-28): Changed bundling for package scanner compatibility. v5.0.1 (2024-10-15): Fixed resize inside popup dialog. v3.1.3 (2023-11-21): Fixed marker color expression editor access to entity attributes. v3.1.2 (2023-09-27): Removed redundant code. v3.1.0 (2023-06-06): Updated page explorer caption; updated icons.

**3. What part of behavior can be documented from this file?**
The v6.0.0 Plotly 3.0 upgrade was a major change. v3.1.3 was a significant bug fix: entity attributes in the "marker color" expression editor were broken between v3.1.x and v4.0.x versions; the fix restored attribute access in the marker color expression editor.

**4. Is it user-facing?**
No. Developer-facing.

**5. What new did you learn from this file?**
The version numbers jump from 3.1.x to 5.x — versions 4.x are not represented. Version 5.1.0 changed bundling specifically for "package scanners" — this suggests a security scanning compatibility requirement in enterprise environments.

---

## Summary of Key Findings

- **Widget identity**: Plotly-based scatter chart rendered in "markers" mode with bubble size dimension. v6.2.1, supports Plotly.js v3.0. Offline capable.
- **Architecture**: Thin widget layer over `@mendix/shared-charts` — the bubble chart only adds bubble-specific size calculation and always-circle marker symbols; everything else (data loading, aggregation, groupBy, chart rendering, layout) is in the shared charts package.
- **Data series**: Multiple series via `lines` list. Each series is either Static (single data source) or Dynamic (grouped by attribute — one series per group). Supports aggregation per series (none/count/sum/avg/min/max/median/mode/first/last).
- **Axes**: X and Y accept String, Enum, DateTime, Decimal, Integer, Long, AutoNumber. Axes are fixed-range (no user zoom). Both axes show zero lines.
- **Bubble size**: Dedicated size attribute (Decimal/Long/Integer only). Two sizing modes: `autosize=true` (normalizes largest bubble to `sizeref%` of chart dimension) or `autosize=false` (direct scale factor). Always `sizemode: "diameter"`.
- **Marker color**: Optional per-series expression returning a color string. When omitted, uses Plotly default color.
- **Click actions**: Optional per-item click action (ListActionValue) — fires the action with the clicked data source item as context.
- **Advanced options**: `customSeriesOptions` (per-series JSON), `customLayout` (chart-level JSON), `customConfigurations` (Plotly config JSON), `enableThemeConfig` (theme folder JSON loading). All hidden by default until `enableAdvancedOptions=true`.
- **Playground**: Optional developer slot for interactive chart configuration exploration at design/runtime.
- **No entity context required**: Can be placed anywhere in a page.
- **Design-time validation**: X, Y, and size attributes are all required when a data source is configured for a line (Studio Pro errors).
