# Draft: line-chart-web

Extracted by worker on 2026-05-09. Covers all source files and local workspace dependencies.

---

## src/LineChart.tsx

**Purpose:** Main widget entry point. Translates Mendix datasource props into a Plotly.js scatter chart (used as a line chart) via the shared `ChartWidget` infrastructure.

**Logic:** A `React.memo` component with a custom shallow comparator (`flatEqual` with `traceEqual` for the `lines` array). Uses `usePlotChartDataSeries` to asynchronously load and map Mendix data series. The `mapSerie` callback converts each line series into a Plotly scatter trace: `type: "scatter"`, mode is `"lines"` for `lineStyle === "line"` or `"lines+markers"` for `lineStyle === "lineWithMarkers"`. Line shape (interpolation) and color expressions are resolved per series. Renders `<ChartWidget>` with fixed layout options (both axes `zeroline: true`, `fixedrange: true`, grid color `#d7d7d7`).

**Behavioral constraints from this file:**
- Both axes are non-zoomable (`fixedrange: true`) — users cannot zoom the chart.
- Line style `"line"` renders with mode `"lines"` only; `"lineWithMarkers"` renders with mode `"lines+markers"` (data point dots visible).
- Line color and marker color are resolved via expression values using `getExpressionValue` (first matching item wins). If undefined, Plotly picks its default color.
- `responsive: true` config option makes the chart resize automatically with its container.
- The widget is memoized with a deep-flat comparator: `lines` array is compared using `traceEqual` (trace-level equality), all other props use `Object.is` via `defaultEqual`.
- CSS class applied: `"widget-line-chart"` plus the user-configured `props.class`.

**User-facing:** Yes — the main chart UI rendered in the Mendix page.

**New findings:** The widget uses `type: "scatter"` (not `"line"`) because Plotly.js implements line charts as scatter traces with `mode: "lines"`. The `"custom"` lineStyle is declared in the XML but not handled explicitly in the component mapper — it passes through without setting `mode`, allowing `customSeriesOptions` to control it entirely.

---

## src/LineChart.xml

**Purpose:** Widget descriptor declaring all configurable properties for the line chart in Mendix Studio/Studio Pro.

**Logic:** Defines a `lines` list property (series) where each series has: `dataSet` (static/dynamic), data source, X/Y attribute mappings, `groupByAttribute` (for dynamic series), `aggregationType`, `interpolation`, `lineStyle`, color expressions (line, marker), click action, tooltip hover text, and `customSeriesOptions`. Top-level props include axis labels, legend toggle, grid lines, dimensions, playground slot, advanced options toggle, and custom layout/config JSON strings.

**Behavioral constraints from this file:**
- `offlineCapable="true"` — widget works in offline Mendix apps.
- `dataSet: "static"` = one series per config object (one data source). `dataSet: "dynamic"` = multiple series from one data source divided by `groupByAttribute`.
- `groupByAttribute` supports String, Boolean, DateTime, Decimal, Enum, HashString, Integer, Long attribute types.
- `aggregationType` defaults to `"none"`. Supported values: none, count, sum, avg, min, max, median, mode, first, last.
- `interpolation`: `"linear"` (straight segments) or `"spline"` (curved/smooth). Default: `"linear"`.
- `lineStyle`: `"line"`, `"lineWithMarkers"`, or `"custom"`. Marker color property is only shown in Studio Pro when `lineStyle === "lineWithMarkers"`.
- `widthUnit`: `"percentage"` (default 100%) or `"pixels"`. `heightUnit`: `"percentageOfWidth"` (default 75%), `"pixels"`, or `"percentageOfParent"`.
- `enableAdvancedOptions` gates `customLayout`, `customConfigurations`, `enableThemeConfig`, and per-series `customSeriesOptions` on the web platform.
- Series order in the list affects rendering layer order: the first series is drawn lowest; subsequent series are drawn on top.

**User-facing:** Indirectly — all properties are configurable by the Mendix developer in Studio Pro.

**New findings:** The widget is categorized under "Charts" in both Studio and Studio Pro toolboxes (`studioProCategory` and `studioCategory`). The `playground` and `showPlaygroundSlot` properties enable integration with the chart-playground widget for runtime chart option editing.

---

## typings/LineChartProps.d.ts

**Purpose:** Auto-generated TypeScript types for all widget props at runtime and in editor preview. Generated from `LineChart.xml`.

**Logic:** Exports `LinesType` (runtime per-series props), `LineChartContainerProps` (all runtime widget props), `LinesPreviewType` and `LineChartPreviewProps` (editor preview shapes). X/Y attributes use `ListAttributeValue<string | Date | Big>` at runtime. Color expressions are `ListExpressionValue<string> | undefined`.

**Behavioral constraints from this file:**
- `aggregationType` is required in `LinesType` (no `undefined`) — it always has a value (default `"none"` from XML).
- `customSeriesOptions` is always a `string` in `LinesType` — defaults to `""` even though XML has `required="false"`.
- `playground` is `ReactNode | undefined` — optional widget slot.
- `LineChartPreviewProps` includes `renderMode: "design" | "xray" | "structure"` and a `translate` function for Studio i18n.
- Numeric attribute values (`Decimal`, `Integer`, `Long`) use `Big` from `big.js` to preserve precision.

**User-facing:** Internal — TypeScript compile-time types only.

**New findings:** The preview props use plain `string` for attribute and expression fields (no Mendix runtime types), reflecting that Studio Pro only needs captions for display.

---

## src/LineChart.editorConfig.ts

**Purpose:** Controls Studio Pro property panel visibility, validates configuration, generates structure preview, and provides a custom page explorer caption.

**Logic:** `getProperties` hides static-only or dynamic-only sub-properties per series based on `dataSet`, hides marker color properties when `lineStyle !== "lineWithMarkers"`, hides playground property when `showPlaygroundSlot === false`. Advanced mode gates `customSeriesOptions` per series, and top-level `customLayout`, `customConfigurations`, `enableThemeConfig`. `getPreview` renders an SVG image (light/dark variants) with optional legend. `check` validates that X and Y attributes are set when a datasource is configured. `getCustomCaption` returns the first series datasource caption.

**Behavioral constraints from this file:**
- Validation errors (blocking): missing X attribute or Y attribute when datasource is set — one error per series per missing attribute.
- `checkSlot` error: playground slot widgets are configured but `showPlaygroundSlot === false`.
- Structure preview: 375px wide chart SVG + optional 85px legend SVG side-by-side; extra space filled with a flex-grow container.
- Advanced options are hidden on the desktop platform (`platform === "desktop"` hides `enableAdvancedOptions`); on web, advanced fields are tab-grouped via `transformGroupsIntoTabs`.
- `getCustomCaption` returns `"Line chart"` when no series are defined, or the first series datasource name (brackets stripped), with "and N more" suffix when multiple series exist.

**User-facing:** Editor-only — affects Studio Pro panel UX and page explorer label.

**New findings:** The preview uses light and dark SVG chart assets stored in `src/assets/`. Dark mode is detected via the `isDarkMode` parameter passed to `getPreview`.

---

## src/LineChart.editorPreview.tsx

**Purpose:** Renders the line chart widget preview on the Mendix Studio design canvas using static SVG images.

**Logic:** Renders `<ChartPreview>` with `ChartPreview.PlotImage` (the line chart SVG) and `ChartPreview.PlotLegend` (the legend SVG). Always uses light-mode assets. The preview wraps in the playground slot if configured.

**Behavioral constraints from this file:**
- The preview is purely static — no live data, no runtime chart rendering.
- The alt text reads "Bubble chart" (copy-paste artifact from another widget; cosmetic only, no runtime impact).
- The legend is always included in the preview component, regardless of the `showLegend` prop at design time.

**User-facing:** Studio design canvas only — not visible at runtime.

**New findings:** Uses `ChartPreview` from `@mendix/shared-charts/preview` to ensure consistent appearance with other chart widget previews in Studio.

---

## hooks/usePlotChartDataSeries (shared-charts)

**Purpose:** React hook (workspace package `@mendix/shared-charts`) that loads Mendix datasource items and maps them to Plotly trace objects asynchronously.

**Logic:** Uses `useEffect` + `useState` to process the `series` array. For `dataSet: "static"`, loads items directly from `staticDataSource`; for `dataSet: "dynamic"`, groups items by `groupByAttribute` and produces one trace per group. Applies `aggregationType` when set (delegates to `aggregateDataPoints`). Returns `null` while loading (widget renders nothing). The `mapSerie` callback (provided by the widget) converts data points into Plotly-specific trace properties.

**Behavioral constraints from this file:**
- Returns `null` if any series is still loading (any datasource item or group-by attribute value has `status: "loading"`).
- Aggregation is applied after data extraction, before Plotly rendering. `"none"` skips aggregation.
- Dynamic series produce one Plotly trace per unique `groupByAttribute` value. Series name comes from `dynamicName` expression evaluated per group.
- `getExpressionValue` helper returns the value from the first item in `dataSourceItems` that has `status === "available"`, or `undefined` — used for line/marker color per-series.
- Null attribute values are pushed as `null` into x/y arrays (Plotly handles gaps).
- Hover text uses `hoverinfo: "text"` when any text is non-empty; falls back to `hoverinfo: "none"`.
- Click actions are bound per-item via `executeAction(listAction.get(item))`.

**User-facing:** Internal hook — drives data loading for the chart.

**New findings:** Dynamic series name defaults to `"(empty)"` when the expression value is unavailable or null; the name is updated for the first valid value found. `Big` values are converted to `number` via `.toNumber()` before being passed to Plotly.

---

## e2e/LineChart.spec.js

**Purpose:** End-to-end (Playwright) tests confirming the line chart widget renders correctly in a running Mendix app.

**Logic:** Tests are grouped by feature: line style (basic, with markers, colored markers), interpolation (linear, curved), grid lines (horizontal, vertical, both), legend (with/without), axis labels (X only, Y only, X+Y), and dimensions (6 combinations of pixel/percentage width/height). Each test navigates to the app, locates the widget container by CSS class, scrolls it into view, and compares a screenshot to a stored baseline PNG.

**Behavioral constraints from this file:**
- Three confirmed line styles: basic line, line with markers, colored line with colored markers.
- Two confirmed interpolation modes: linear (straight segments), curved (spline).
- Three grid line modes confirmed: horizontal-only, vertical-only, both. (None is the default and tested implicitly via other tests.)
- Legend can be shown or hidden — both states confirmed with baselines.
- Axis labels can be X-only, Y-only, or X+Y simultaneously.
- Six dimension combinations confirmed: pixels×pixels, pixels×percentageOfWidth, pixels×percentageOfParent, percentage×pixels, percentage×percentageOfParent, percentage×percentageOfWidth.
- Session cleanup is enforced after each test (`window.mx.session.logout()`) to stay within the 5-session Mendix license limit.

**User-facing:** QA/testing only — validates visual output in real browser against Mendix app.

**New findings:** No `test.skip` guards for client mode — the widget is supported in both classic and modern Mendix React clients (unlike some other widgets in this repo). Screenshot baselines are stored as `lineChart*.png` files in `e2e/LineChart.spec.js-snapshots/`.

---

## src/__tests__/LineChart.spec.tsx

**Purpose:** Unit tests for the `LineChart` component using Jest + React Testing Library, with `ChartWidget` mocked.

**Logic:** Tests focus on the `mapSerie` callback behavior: correct Plotly trace type (`scatter`), mode (based on `lineStyle`), line shape (based on `interpolation`), line color, marker color, and aggregation behavior. Uses a `setupBasicSeries` helper to build minimal `LinesType` objects with two data points (x: 1,2; y: 3,6).

**Behavioral constraints from this file:**
- Chart type is always `"scatter"` — confirmed by unit test.
- `lineWithMarkers` → mode `"lines+markers"`; `"line"` → mode `"lines"` — confirmed.
- `interpolation: "linear"` → `line.shape: "linear"`; `"spline"` → `line.shape: "spline"` — confirmed.
- Line color expression (`staticLineColor`) is a `ListExpressionValue<string>`; when provided returns the color string (e.g. `"red"`); when `undefined`, `line.color` is `undefined` (Plotly uses default).
- Marker color expression (`staticMarkerColor`): same behavior — `"blue"` when set, `undefined` otherwise.
- Aggregation: with `aggregationType: "none"`, two points (x: [1,2], y: [3,6]) remain separate; with `aggregationType: "avg"`, they would be aggregated (the test confirms raw data length = 2 for `"none"`).
- Default test setup: `dataSet: "static"`, `aggregationType: "avg"`, `interpolation: "linear"`, `lineStyle: "line"`.

**User-facing:** Internal testing only.

**New findings:** The unit test mocks `ChartWidget` to return `null`, so only the `data` prop passed to `ChartWidget` is inspected. Tests confirm that static series with no line/marker color expressions pass `undefined` to Plotly (no default color injection from the widget level).

---

## CHANGELOG.md

**Purpose:** Version history for the line-chart-web widget following Keep a Changelog / SemVer conventions.

**1. What is the purpose of this file?**
Documents all notable changes per release, from v3.1.0 through v6.2.1 (plus unreleased).

**2. What kind of logic is described in this file?**
Release notes: dependency upgrades, bug fixes, behavioral fixes.

**3. What part of behavior can be documented from this file?**
- v6.2.1 (2025-07-15): updated shared charts dependency (no behavioral change stated).
- v6.2.0 (2025-06-03): fixed aggregation broken by Plotly 3.0 upgrade — aggregation (count, sum, avg, etc.) was non-functional after the Plotly 3.0 update until this fix.
- v6.0.0 (2025-02-28): upgraded Plotly.js to version 3.0 — this is a major dependency version change that broke aggregation (fixed in v6.2.0).
- v5.1.0 (2024-10-28): changed bundling to make Plotly scannable by package scanners (no behavioral change).
- v5.0.1 (2024-10-15): fixed auto-resize inside popup dialogs — widget now resizes correctly when placed inside a Mendix popup (placement constraint fixed).
- v3.1.2 (2023-09-27): removed redundant code to improve widget load time (no behavioral change).
- v3.1.1 (2023-08-16): fixed entity attribute availability in "line color" and "marker color" expression editors — entity attributes were unavailable after a regression in v4.0; this fix restores access to entity attributes in both the line color and marker color expression editors (confirmed for both, not just line color).
- v3.1.0 (2023-06-06): updated page explorer caption to show datasource; updated icons and tiles.

**4. Is it user-facing?**
Indirectly — release notes for developers and operators.

**5. What new did you learn from this file?**
The line color expression editor regression (entity attributes unavailable) was introduced in v4.0 and fixed in v3.1.1. This means versions v4.0.x had broken color expression editing — a significant behavioral constraint. The v6.0.0 Plotly 3.0 upgrade introduced a breaking change (aggregation broken) that took until v6.2.0 to fix; users on v6.0.0 or v6.1.x would have seen aggregation silently produce incorrect or missing data.
