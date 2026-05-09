# Draft: time-series-chart-web

Widget package: `packages/pluggableWidgets/time-series-chart-web`

---

## src/TimeSeries.xml

**1. What is the purpose of this file?**
Widget descriptor XML defining the widget's identity, all configurable props, and Studio Pro categorization. Generates `TimeSeriesProps.d.ts`.

**2. What kind of logic is described in this file?**
Declares: `lines` (object list of series configurations, each with): `dataSet` (static/dynamic), X-axis DateTime attribute, Y-axis attribute (numeric or string types), optional group-by attribute (dynamic mode), `aggregationType` (none/count/sum/avg/min/max/median/mode/first/last), `lineStyle` (line/lineWithMarkers), `interpolation` (linear/spline), `lineColor`, `markerColor`, `fillArea`, `fillColor`, `name` (series label), `tooltipHoverText`, `onClick` action. Layout properties: `xAxisLabel`, `yAxisLabel`, `showLegend`, `showRangeSlider`, `gridLines`, `yAxisRangeMode` (tozero/normal/nonnegative). Dimension properties: width/height with unit options. Advanced: `enableAdvancedOptions`, `customSeriesOptions`, `customLayoutOptions`.

**3. What part of behavior can be documented from this file?**
- X-axis MUST be a DateTime attribute — the widget is specifically for time series (not generic line charts).
- Y-axis accepts numeric types (Integer, Long, Decimal) and String.
- `aggregationType="none"` uses raw data points; other types aggregate multiple Y values per X timestamp.
- 9 aggregation functions: count, sum, avg, min, max, median, mode, first, last.
- Dynamic data source groups multiple series by a `groupByAttribute` — each unique group value becomes a separate series.
- `fillArea` fills the area between this series and the one below it (`tonexty` Plotly mode).
- Range slider is a Plotly-provided interactive control for zooming into time ranges.
- Advanced custom JSON options allow overriding Plotly layout and series configurations.
- Widget is `needsEntityContext="true"`, `offlineCapable="true"`.

**4. Is it user-facing?**
Defines the developer-facing configuration interface in Studio/Studio Pro.

**5. What new did you learn from this file?**
The `fillArea` property fills between adjacent series using Plotly's `tonexty` mode — meaning series ordering matters for fill visualization. The first series is the baseline; each subsequent series can fill the area up to the previous one.

---

## src/TimeSeries.tsx

**1. What is the purpose of this file?**
The root React component that transforms Mendix data binding and configuration props into Plotly.js chart options, then delegates to the shared `ChartWidget`.

**2. What kind of logic is described in this file?**
Uses `usePlotChartDataSeries` from `@mendix/shared-charts` to convert each line configuration to a Plotly trace. Maps `lineStyle`: `"line"` → `mode: "lines"`, `"lineWithMarkers"` → `mode: "lines+markers"`. Maps `interpolation`: `"linear"` → `line.shape: "linear"`, `"spline"` → `line.shape: "spline"`. Sets `fill: "tonexty"` when `fillArea=true`. Constructs layout options: X-axis always `type: "date"`, Y-axis `rangemode` from `yAxisRangeMode`, range slider `visible: showRangeSlider`. Uses `memo` with a custom equality function that compares `lines` array element-by-element using `traceEqual`.

**3. What part of behavior can be documented from this file?**
- X-axis is always `type: "date"` — enforced at the component level, not just the data layer.
- Y-axis `rangemode` options: `"tozero"` (default, starts at 0), `"normal"` (auto-range), `"nonnegative"` (no negative values).
- Range slider appears below the chart when enabled.
- Grid lines color is hardcoded to `#d7d7d7`.
- The `memo` wrapper prevents re-renders when `lines` array elements haven't materially changed (using `traceEqual` for deep comparison of trace data).

**4. Is it user-facing?**
Yes — produces the visible chart.

**5. What new did you learn from this file?**
The custom `memo` equality function treats the `lines` array differently: element-wise comparison via `traceEqual`, while all other props use `defaultEqual`. This is a performance optimization specific to the data series — chart re-renders are expensive, so series equality is checked carefully before allowing a re-render.

---

## src/TimeSeries.editorConfig.ts

**1. What is the purpose of this file?**
Provides `getProperties()` (conditional visibility), `check()` (validation), and `getCustomCaption()` (page explorer label) for Studio Pro.

**2. What kind of logic is described in this file?**
`getProperties()`: hides static/dynamic-specific properties based on `dataSet` selection per line; hides `markerColor` when `lineStyle≠"lineWithMarkers"`; hides `fillColor` when `fillArea=false`; hides advanced properties unless `enableAdvancedOptions=true`. `check()`: for each line, validates that X attribute and Y attribute are set (and `groupByAttribute` for dynamic mode). `getCustomCaption()`: returns the datasource caption + "and N more" when multiple series are configured.

**3. What part of behavior can be documented from this file?**
- Studio Pro only shows `markerColor` when "Line with markers" style is selected.
- Fill color property is only shown when fill area is enabled.
- Advanced Plotly customization options are hidden by default — must be explicitly unlocked via `enableAdvancedOptions`.
- Validation checks are per-line: error messages include the line index in the property path (e.g., `lines/0/staticXAttribute`).
- Page explorer caption shows the first datasource name + count of additional series.

**4. Is it user-facing?**
Yes — visible to developers configuring the widget in Studio Pro.

**5. What new did you learn from this file?**
The `getCustomCaption` generating "Data and N more" uses the series count to give context in the page explorer. For a chart with 3 series, the caption would be something like "[MyData] and 2 more" — this is helpful when multiple time-series charts are on the same page.

---

## src/TimeSeries.editorPreview.tsx

**1. What is the purpose of this file?**
Provides the structure preview (SVG-based) and design canvas preview for the widget in Studio Pro.

**2. What kind of logic is described in this file?**
`getPreview()`: Returns a `RowLayout` with a chart SVG image (selected based on `showRangeSlider` and `isDarkMode`) plus optional legend SVG. Uses `withPlaygroundSlot()` wrapper from `@mendix/shared-charts` to add a playground widget slot for advanced customization testing. `getProperties()` and `check()` delegate to `editorConfig.ts`. Runtime preview component: renders `ChartPreview` with the appropriate SVG based on `showRangeSlider`.

**3. What part of behavior can be documented from this file?**
- Structure preview has two variants: with and without the range slider.
- Dark mode is supported — dark SVG variants are used when `isDarkMode=true`.
- The playground slot (from `withPlaygroundSlot`) allows attaching a testing widget for custom Plotly options.
- Legend image is shown conditionally based on `showLegend`.

**4. Is it user-facing?**
Yes — visible to developers in Studio Pro structure preview.

**5. What new did you learn from this file?**
The `withPlaygroundSlot` wrapper from `@mendix/shared-charts` adds a playground widget slot to ALL charts in the web-widgets repository that use it. This is a shared mechanism for testing advanced Plotly customization options without modifying the widget code.

---

## typings/TimeSeriesProps.d.ts

**1. What is the purpose of this file?**
Auto-generated TypeScript typings from `TimeSeries.xml`. Defines `TimeSeriesContainerProps` (runtime) and `TimeSeriesPreviewProps` (Studio design-mode), plus the `LinesType` interface for each series configuration.

**2. What kind of logic is described in this file?**
Enumerations: `DataSetEnum` ("static"|"dynamic"), `AggregationTypeEnum` (9 values), `InterpolationEnum`, `LineStyleEnum`, `GridLinesEnum`, `WidthUnitEnum`, `HeightUnitEnum`, `YAxisRangeModeEnum`. `LinesType`: datasource (`ListValue`), X attribute (`ListAttributeValue<Date>`), Y attribute (`ListAttributeValue<Big>|ListAttributeValue<string>`), color properties (`ListExpressionValue<string>`), onClick (`ListActionValue`). `TimeSeriesContainerProps`: `lines: LinesType[]`, layout options.

**3. What part of behavior can be documented from this file?**
- X-axis attribute is typed as `ListAttributeValue<Date>` — enforces DateTime type at the TypeScript level.
- Y-axis attribute is typed as a union: `ListAttributeValue<Big>` (numeric) OR `ListAttributeValue<string>`.
- `lineColor` and `fillColor` are `ListExpressionValue<string>` — colors can be per-row expressions (e.g., conditional coloring).
- `onClick` is `ListActionValue` — can be a per-item action (different action per data point).
- `lines` is an array — supports multiple series in a single widget instance.

**4. Is it user-facing?**
No — internal type declarations.

**5. What new did you learn from this file?**
Both `lineColor` and `fillColor` are `ListExpressionValue<string>` — they can vary per data point, not just per series. This means individual data points within a series could have different colors if the expression evaluates differently per item. However, Plotly's line chart typically uses a single color per series, so this capability may not be fully utilized at the rendering layer.

---

## src/__tests__/TimeSeries.spec.tsx

**1. What is the purpose of this file?**
Unit tests for the `TimeSeries` component using Jest, verifying correct mapping of widget props to Plotly chart options.

**2. What kind of logic is described in this file?**
Tests: line style mapping (`"line"` → `mode: "lines"`, `"lineWithMarkers"` → `mode: "lines+markers"`); interpolation mapping (`"linear"` → `shape: "linear"`, `"spline"` → `shape: "spline"`); color properties (line, marker, fill); aggregation type application; range slider visibility; Y-axis range mode. Uses `ListAttributeValueBuilder` to mock date/numeric data. Mocks `ChartWidget` to capture Plotly options without rendering.

**3. What part of behavior can be documented from this file?**
- `aggregationType="none"` returns raw Date objects for X values.
- `aggregationType="avg"` returns ISO string dates for X values (aggregation converts dates to strings).
- Color values from expressions are passed through to Plotly traces when available.
- When no color is set, Plotly's default color scheme is used.
- `showRangeSlider=true` sets `xaxis.rangeslider.visible: true` in layout.

**4. Is it user-facing?**
No — test file only.

**5. What new did you learn from this file?**
The date format change with aggregation is important: aggregated X values become ISO strings, not Date objects. This means Plotly receives different types depending on whether aggregation is enabled — both are valid for Plotly's date axis, but the format change is a behavioral detail that could affect custom axis formatting.

---

## e2e/TimeSeriesChart.spec.js

**1. What is the purpose of this file?**
End-to-end Playwright tests for the Time Series Chart widget using visual regression (screenshot comparisons).

**2. What kind of logic is described in this file?**
Tests: multiple series rendering; range slider visible/hidden; fill area enabled/disabled; custom fill colors; Y-axis range modes (non-negative, auto). Each test: locates the test container by `mx-name-*` class, waits for visibility, takes a screenshot comparison with 50% threshold. Session logout after each test.

**3. What part of behavior can be documented from this file?**
- Multiple series render as separate lines on the same chart.
- Range slider renders below the X-axis as an interactive time-range control.
- Fill area renders the shaded region between lines.
- Y-axis non-negative mode prevents the axis from going below 0 even if data is 0.
- Screenshots use 50% threshold tolerance (allowing minor rendering differences between environments).

**4. Is it user-facing?**
The tested behaviors (chart rendering, range slider, fill areas) are user-facing.

**5. What new did you learn from this file?**
The 50% screenshot threshold is unusually high — typical visual regression tests use 1-5%. This may reflect intentional tolerance for Plotly's anti-aliased rendering which can differ slightly between browser versions or operating systems. The test primarily catches major visual regressions, not pixel-perfect comparisons.

---

## CHANGELOG.md

**1. What is the purpose of this file?**
Documents version history for the time-series-chart-web widget.

**2. What kind of logic is described in this file?**
Key versions: 6.3.0 (current, unreleased), 6.2.1 (fix: aggregation issue with Plotly 3.0), 6.0.0 (2025-02-28, Plotly.js updated to v3.0 — **breaking**), 5.1.0 (Plotly bundling change for package scanners), 5.0.1 (fix: auto-resize in popup dialogs), earlier versions: redundant code removal, icon/tile updates.

**3. What part of behavior can be documented from this file?**
- v6.0.0 upgraded Plotly.js from 2.x to 3.0 — a breaking change for any users with custom Plotly options that relied on v2.x APIs.
- v6.2.1 fixed an aggregation regression introduced by the Plotly 3.0 upgrade.
- v5.0.1 fixed chart resizing inside Mendix popup dialogs (popups don't fire standard resize events).
- v5.1.0 changed Plotly bundling to make it scannable by open-source dependency scanners.

**4. Is it user-facing?**
The changelog is publicly visible on the Mendix Marketplace.

**5. What new did you learn from this file?**
The Plotly 3.0 upgrade (v6.0.0) broke aggregation in v6.2.1 — suggesting that Plotly's internal data processing APIs changed between major versions. This is a reminder that custom Plotly options (via `customSeriesOptions`/`customLayoutOptions`) may need updating when Plotly major versions are released.
