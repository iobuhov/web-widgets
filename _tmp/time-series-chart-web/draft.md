# time-series-chart-web — Draft Spec

Widget: `time-series-chart-web`
Package: `packages/pluggableWidgets/time-series-chart-web/`
Agent: worker
Date: 2026-05-09

---

## src/TimeSeries.tsx

**1. What is the purpose of this file?**
The root Mendix pluggable widget entry component. Processes the `lines` array through `usePlotChartDataSeries`, constructs Plotly layout options, and delegates rendering to `ChartWidget` from `@mendix/shared-charts`.

**2. What kind of logic is described in this file?**
- `createTimeSeriesChartLayoutOptions(showRangeSlider, yAxisRangeMode)` produces the Plotly layout object:
  - `xaxis.type: "date"` — forces datetime x-axis.
  - `xaxis.rangeslider.visible: showRangeSlider` — optional range slider below chart.
  - `yaxis.fixedrange: true` — y-axis is NOT zoomable/pannable.
  - `yaxis.rangemode: yAxisRangeMode || "tozero"` — controls y-axis floor.
  - Grid colors use `#d7d7d7` for both gridlines and zero lines.
- `timeSeriesChartSeriesOptions = { type: "scatter", hoverinfo: "none" }` — Plotly `scatter` type with custom hover disabled.
- Per-series transform (inside `usePlotChartDataSeries` callback):
  - `mode`: `lineStyle === "line"` → `"lines"`, else `"lines+markers"` (both `"lineWithMarkers"` and `"custom"` map to this).
  - `fill`: `enableFillArea ? "tonexty" : "none"` — fills to the **previous trace** (not to x-axis).
  - `fillcolor`: `line.fillColor?.value` — optional color override.
  - `line.shape`: `line.interpolation` — `"linear"` or `"spline"`.
  - `line.color`: `line.lineColor?.value`.
  - `marker.color`: `line.markerColor?.value`.
- `memo` with custom comparator: props equality check uses `flatEqual` for top-level props and `traceEqual` for each item in the `lines` array — prevents unnecessary re-renders from reference changes.

**3. What part of behavior can be documented from this file?**
- The x-axis is always typed as `"date"` — this widget is exclusively for time-based data.
- Y-axis zoom is disabled (`fixedrange: true`); only x-axis panning/zooming is allowed.
- Fill area uses `"tonexty"` (fill to previous trace): the first series fills to the zero baseline; subsequent series fill to the line below them, creating a stacked visual.
- `lineStyle: "custom"` is an XML enum value but maps to `"lines+markers"` in the Plotly data — no additional customization is applied in this file (custom styling via `customSeriesOptions` string is handled by `ChartWidget`/`usePlotChartDataSeries`).
- Default hover tooltip (`hoverinfo: "none"`) is disabled — custom tooltips from `tooltipHoverText` are handled by the shared-charts layer.
- `configOptions: { responsive: true }` — chart resizes with its container.

**4. Is it user-facing?**
No — internal Mendix adapter.

**5. What new did you learn from this file?**
`fill: "tonexty"` is subtle: it fills to the *previous* series trace in the data array, not to the x-axis. The XML description says "Fill area between data point and x-axis," but that only holds for the first series. For subsequent series stacked on top, the fill goes from that series's line down to the previous series's line — creating overlapping/stacked shading.

---

## src/TimeSeries.xml

**1. What is the purpose of this file?**
Mendix widget descriptor defining the widget's property groups. Supports multiple configurable series (lines) and comprehensive chart-level options across three tabs: General, Dimensions, Advanced.

**2. What kind of logic is described in this file?**
`lines` is an object list — each entry represents one time-series line with:
- **Data source group**:
  - `dataSet`: `"static"` (single series datasource) | `"dynamic"` (grouped datasource).
  - `staticDataSource` / `dynamicDataSource`: list data sources (mutually exclusive).
  - `groupByAttribute` (dynamic only): any scalar attribute type; splits data into separate lines per group.
  - `staticXAttribute` / `dynamicXAttribute`: **only `DateTime` type accepted**.
  - `staticYAttribute` / `dynamicYAttribute`: String | Enum | DateTime | Decimal | Integer | Long | AutoNumber.
  - `aggregationType`: none | count | sum | avg | min | max | median | mode | first | last.
  - `staticTooltipHoverText` / `dynamicTooltipHoverText`: custom tooltip text template.
- **Appearance group**:
  - `interpolation`: linear | spline (curved).
  - `lineStyle`: line | lineWithMarkers | custom.
  - `lineColor`, `markerColor`, `fillColor`: optional text templates (CSS color values).
  - `enableFillArea`: boolean, default `true`.
- **Events group**: `staticOnClickAction` / `dynamicOnClickAction` (optional).
- **Advanced group**: `customSeriesOptions` (multiline string for raw Plotly trace options).

Chart-level (top-level) settings:
- `enableAdvancedOptions`: boolean toggle that gates advanced properties.
- `showPlaygroundSlot` + `playground`: widget slot for developer experimentation.
- `xAxisLabel`, `yAxisLabel`: optional axis label text.
- `showLegend`: boolean, default `true`.
- `showRangeSlider`: boolean, default `true`.
- `gridLines`: none | horizontal | vertical | both.
- Dimensions: `widthUnit` (percentage|pixels, default percentage/100), `heightUnit` (percentageOfWidth|pixels|percentageOfParent, default percentageOfWidth/75).
- Advanced (gated by `enableAdvancedOptions`): `customLayout`, `customConfigurations`, `enableThemeConfig`, `yAxisRangeMode` (normal/tozero/nonnegative, default tozero).

System properties: Name, Visibility, TabIndex.

**3. What part of behavior can be documented from this file?**
- `needsEntityContext="true"`, `offlineCapable="true"`.
- X-axis attribute is restricted to **DateTime only** — enforced at widget type level.
- Y-axis attribute supports a wide variety of types (including String, Enum, DateTime) — non-numeric values are presumably aggregated or used as labels.
- `aggregationType` with 10 options handles multiple Y values at the same X timestamp.
- `yAxisRangeMode` is hidden unless `enableAdvancedOptions` is on — the default `"tozero"` behavior is invisible to basic users.
- `showRangeSlider` default is `true` — range slider is shown by default.
- `enableFillArea` default is `true` — fill area is on by default.

**4. Is it user-facing?**
No — Studio Pro configuration descriptor.

**5. What new did you learn from this file?**
Dynamic series mode with `groupByAttribute` allows one datasource to produce multiple lines: all data points sharing the same `groupByAttribute` value are grouped into one line. This is analogous to a "group by" SQL clause — e.g., grouping sensor readings by `sensorId` attribute to get one line per sensor from a single entity/datasource.

---

## typings/TimeSeriesProps.d.ts

**1. What is the purpose of this file?**
Auto-generated TypeScript types from the XML descriptor.

**2. What kind of logic is described in this file?**
- `LinesType` (runtime per-series object):
  - `groupByAttribute?: ListAttributeValue<string | boolean | Date | Big>` — all scalar types.
  - `lineColor?: DynamicValue<string>` — per-series (not per-data-point) dynamic expression.
  - `enableFillArea: boolean` — non-optional boolean.
  - `customSeriesOptions: string` — always present (empty string if not set).
- `TimeSeriesContainerProps` includes: `name`, `class`, `style?: CSSProperties`, `tabIndex?: number` — this widget supports CSS class/style customization (unlike switch-web).
- `YAxisRangeModeEnum = "normal" | "tozero" | "nonnegative"` — the XML enum key `"normal"` maps to the label "Auto".
- `LinesPreviewType.fillColor: string` (not optional in preview) vs `LinesType.fillColor?: DynamicValue<string>` (optional in runtime).

**3. What part of behavior can be documented from this file?**
- `lineColor`, `markerColor`, `fillColor` are `DynamicValue<string>` (not `ListExpressionValue`) — they are per-series expressions, not per-data-point. A single color applies to the whole series.
- `aggregationType: AggregationTypeEnum` is non-optional in `LinesType` — always has a value (default "none" from XML).
- Container props have `class` and `style` — CSS class customization is supported.

**4. Is it user-facing?**
No — TypeScript types only.

**5. What new did you learn from this file?**
`lineColor`, `markerColor`, and `fillColor` are `DynamicValue<string>` (widget-level expressions), not `ListExpressionValue<string>` (per-item expressions bound to the datasource). This means color values can be set from widget parameters or context entity attributes, but NOT from individual data point attributes in the list datasource. Color is uniform per series.

---

## src/TimeSeries.editorConfig.ts

**1. What is the purpose of this file?**
Studio Pro property visibility, validation (`check`), structure preview, and custom caption for the Time Series chart widget.

**2. What kind of logic is described in this file?**
`getProperties` rules:
- Per series (`lines[index]`):
  - `dataSet === "static"` → hides: `dynamicDataSource`, `dynamicXAttribute`, `dynamicYAttribute`, `dynamicName`, `dynamicTooltipHoverText`, `groupByAttribute`.
  - `dataSet === "dynamic"` → hides: `staticDataSource`, `staticXAttribute`, `staticYAttribute`, `staticName`, `staticTooltipHoverText`.
  - `lineStyle !== "lineWithMarkers"` → hides `markerColor`.
  - `!enableFillArea` → hides `fillColor`.
  - `!enableAdvancedOptions && platform === "web"` → hides per-series `customSeriesOptions`.
- Chart-level:
  - `!showPlaygroundSlot` → hides `playground`.
  - `!enableAdvancedOptions && platform === "web"` → hides `customLayout`, `customConfigurations`, `enableThemeConfig`, `yAxisRangeMode`.
  - `platform === "web"` → `transformGroupsIntoTabs(defaultProperties)` — converts property groups to tabs in the Studio Pro panel.
  - `platform !== "web"` (desktop/mobile) → hides `enableAdvancedOptions`.

`getPreview`: Returns a `RowLayout` with:
- Chart SVG (375px): switches between range-slider and no-range-slider variant based on `showRangeSlider`.
- Legend SVG (85px): included only if `showLegend` is true.
- Filler container (grow: 1) to fill remaining space.
- All 6 SVGs: `TimeSeries.light/dark.svg`, `TimeSeries-range.light/dark.svg`, `TimeSeries-legend.light/dark.svg`.
- Wrapped with `withPlaygroundSlot(values, chart)` — adds playground slot if `showPlaygroundSlot`.

`check`: Studio Pro validation errors if data source is set but X or Y attribute is missing (per series, for both static and dynamic modes).

`getCustomCaption`: First series datasource caption, plus "and N more" if multiple series; "Time series" if no series configured.

**3. What part of behavior can be documented from this file?**
- The `markerColor` property is hidden unless `lineStyle === "lineWithMarkers"` — markers are only relevant for that style.
- `fillColor` is hidden when `enableFillArea` is false — cleaner Studio Pro UX.
- `yAxisRangeMode` and other advanced options are hidden from basic users unless `enableAdvancedOptions` is explicitly turned on.
- Studio Pro shows a validation error if a datasource is configured but axis attributes are missing — preventing misconfigured charts.
- The preview SVG adapts to the `showRangeSlider` setting — users see an accurate representation.

**4. Is it user-facing?**
No — Studio Pro only.

**5. What new did you learn from this file?**
`withPlaygroundSlot` is a shared-charts utility that conditionally adds the playground widget slot to the structure preview. This pattern allows developers to embed a Plotly-config playground widget inside the chart for live experimentation in design mode — without affecting production rendering.

---

## src/TimeSeries.editorPreview.tsx

**1. What is the purpose of this file?**
Live React preview rendering in Studio Pro design mode using `ChartPreview` from `@mendix/shared-charts/preview`.

**2. What kind of logic is described in this file?**
- Renders `<ChartPreview>` with:
  - `image`: `<ChartPreview.PlotImage src={showRangeSlider ? TimeSeriesRange : TimeSeries} alt="Bubble chart" />` — note the alt text says "Bubble chart" (copy-paste artifact from another widget).
  - `legend`: `<ChartPreview.PlotLegend src={TimeSeriesLegend} alt="Legend" />` — always rendered (legend is always shown in design preview regardless of `showLegend` prop).
- All props spread onto `ChartPreview` — includes `showLegend`, `showRangeSlider`, dimensions, etc.

**3. What part of behavior can be documented from this file?**
- Design preview always shows the legend (ignores `showLegend` value) — `ChartPreview.PlotLegend` is always passed.
- Image choice (`TimeSeries` vs `TimeSeriesRange`) follows the `showRangeSlider` prop — preview accurately reflects the range slider setting.

**4. Is it user-facing?**
No — Studio Pro design mode preview only.

**5. What new did you learn from this file?**
The `alt="Bubble chart"` text on the image is a copy-paste artifact from another chart widget — this is a minor bug in the code (incorrect alt text). The legend is always shown in preview even when `showLegend` is false — `ChartPreview` likely handles the `showLegend` prop internally, or the design preview always shows legend as a visual affordance.

---

## src/__test__/TimeSeries.spec.tsx

**1. What is the purpose of this file?**
Unit tests for the `TimeSeries` container component — mocks `ChartWidget` and asserts props passed to it.

**2. What kind of logic is described in this file?**
- `seriesOptions.type === "scatter"` — Plotly type is always "scatter".
- `lineStyle: "lineWithMarkers"` → `mode: "lines+markers"`; `lineStyle: "line"` → `mode: "lines"`.
- `interpolation: "linear"` → `line.shape: "linear"`; `interpolation: "spline"` → `line.shape: "spline"`.
- `lineColor: dynamic("red")` → `data[0].line.color === "red"`; undefined → `data[1].line.color === undefined`.
- `markerColor: dynamic("blue")` → `data[1].marker.color === "blue"`.
- `aggregationType: "none"` with 2 items → `x: [Date("2022-01-01"), Date("2022-01-02")]` (raw Date objects), `y: [3, 6]`.
- `aggregationType: "avg"` with same 2 items → `x: [Date("2022-01-01").toISOString(), Date("2022-01-02").toISOString()]` (ISO strings), `y: [3, 6]`.
- `fillColor: dynamic("red")` → `fillcolor: "red"`.
- `showRangeSlider: true` → `layoutOptions.xaxis.rangeslider.visible === true`.
- `yAxisRangeMode: "nonnegative"` → `layoutOptions.yaxis.rangemode === "nonnegative"`.

**3. What part of behavior can be documented from this file?**
- Confirmed: Plotly chart type is `"scatter"` (not `"line"`) — Plotly uses scatter with mode=lines for line charts.
- Key finding: `aggregationType: "none"` produces raw `Date` objects for x-values; `aggregationType: "avg"` produces ISO string x-values — the aggregation pipeline converts dates to ISO strings as part of grouping.
- Default test setup uses `aggregationType: "avg"` in `setupBasicSeries` — most tests exercise the aggregation path.

**4. Is it user-facing?**
No — test file.

**5. What new did you learn from this file?**
The x-axis data type differs based on aggregation: `aggregationType: "none"` passes raw `Date` objects to Plotly, while any aggregation function converts them to ISO strings. This is because the aggregation pipeline uses string-keyed grouping internally. Plotly accepts both formats for its `"date"` axis type.

---

## e2e/TimeSeriesChart.spec.js

**1. What is the purpose of this file?**
Playwright E2E tests that verify visual rendering through screenshot comparison only — no behavioral interaction tests.

**2. What kind of logic is described in this file?**
All tests are `toHaveScreenshot()` comparisons with `threshold: 0.5` (50% pixel difference tolerance):
- Multiple series rendering.
- Without range slider.
- Without fill area (line-only).
- With custom fill area color.
- Y-range non-negative mode.
- Y-range auto mode.
- Uses `scrollIntoViewIfNeeded()` before screenshot to ensure the chart is fully visible.
- Session logout after each test (`window.mx.session.logout()`) to avoid Mendix license session limit.

**3. What part of behavior can be documented from this file?**
- E2E tests are visual regression only — no click/interaction/data assertions.
- 6 test scenarios cover the major visual variations of the chart.
- The 50% screenshot threshold is very lenient — allows for minor rendering differences across environments.

**4. Is it user-facing?**
No — test file.

**5. What new did you learn from this file?**
All interaction with chart data and behavior is tested only via visual screenshots — no data assertions (x/y values, click handlers). This is typical for Plotly-based charts where the rendered output is complex SVG/canvas; interacting with specific data points in E2E is fragile. Screenshot baseline files (`.png` snapshots) are committed to the repo.

---

## CHANGELOG.md

**1. What is the purpose of this file?**
Version history from v3.1.0 onward.

**2. What kind of logic is described in this file?**
- **v6.2.1 (2025-07-15)**: Updated shared charts dependency.
- **v6.2.0 (2025-06-03)**: Fixed aggregate being removed on Plotly 3.0.
- **v6.0.0 (2025-02-28)**: Updated Plotly.js library to version 3.0.
- **v5.1.0 (2024-10-28)**: Changed bundling to make Plotly scannable by package scanners (security/compliance).
- **v5.0.1 (2024-10-15)**: Fixed auto-resize inside popup dialog.
- **v3.1.2 (2023-09-27)**: Removed redundant code to improve widget load time.
- **v3.1.0 (2023-06-06)**: Updated page explorer caption to display datasource; updated icons/tiles.
- Version gaps: v3.x → v5.x → v6.x (v4.x not in this CHANGELOG).

**3. What part of behavior can be documented from this file?**
- v6.0.0 Plotly 3.0 migration was a breaking change (major version bump).
- v6.2.0 fix for aggregate: Plotly 3.0 removed or changed the aggregation API — had to be patched.
- v5.0.1: popup dialog resize fix means the chart didn't correctly detect size changes within a Mendix popup.

**4. Is it user-facing?**
No — developer changelog.

**5. What new did you learn from this file?**
The Plotly 3.0 upgrade broke the aggregation feature (v6.2.0 fix) — confirming that aggregation is implemented using Plotly's built-in transform mechanism (not in application code), since a Plotly version change broke it. Plotly transforms (including aggregations) were reorganized in Plotly 3.0.

---

## Summary of Key Findings

- **Purpose**: Time series line chart showing data changes over time. Supports multiple configurable series, static or dynamic (grouped) datasources, area fill, range slider, and optional on-click actions.
- **Library**: Uses Plotly.js (`plotly.js-dist-min ^3.0.0`) via `@mendix/shared-charts`. Plotly type is `"scatter"` with `mode: "lines"` or `"lines+markers"`.
- **X-axis**: Always `type: "date"` — only `DateTime` attributes accepted. Data points passed as raw `Date` objects (no aggregation) or ISO strings (with aggregation).
- **Y-axis**: `fixedrange: true` — y-axis is not zoomable. `rangemode` defaults to `"tozero"` (can be changed to `"normal"` (Auto) or `"nonnegative"`).
- **Fill area**: `fill: "tonexty"` — fills to the previous trace, not the x-axis. For the first series this is effectively fill-to-zero; for subsequent series it creates stacked shading.
- **Line modes**: `"line"` → `mode: "lines"`; `"lineWithMarkers"` and `"custom"` → `mode: "lines+markers"`.
- **Colors**: `lineColor`, `markerColor`, `fillColor` are per-series `DynamicValue<string>` expressions — uniform per series, not per data point.
- **Aggregation**: 10 aggregation functions (none, count, sum, avg, min, max, median, mode, first, last). Implemented via Plotly transforms. Changes x-values from Date objects to ISO strings.
- **Dynamic series**: `groupByAttribute` groups data from one datasource into multiple lines by attribute value.
- **Range slider**: On by default (`showRangeSlider: true`). Styled with `#d7d7d7` border.
- **CSS customization**: Widget supports `class` and `style` props — CSS class customization is available.
- **Advanced options**: `customLayout`, `customConfigurations`, `customSeriesOptions`, `enableThemeConfig` are all gated behind `enableAdvancedOptions` toggle.
- **Playground slot**: Optional `playground` widget slot for developer experimentation with Plotly config.
- **offlineCapable**: `true`.
- **Testing**: Unit tests mock `ChartWidget` and assert data/layout props; E2E tests are screenshot-only with 50% threshold.
- **Bug**: `alt="Bubble chart"` in `editorPreview.tsx` — incorrect alt text (copy-paste from bubble chart widget).
