# Draft: area-chart-web

Extracted by worker on 2026-05-08. Covers all source files and local workspace dependencies.

---

## src/AreaChart.tsx

**Purpose:** Main widget entry point. Translates Mendix datasource props into a plotly.js area (filled scatter) chart via the shared `ChartWidget` infrastructure.

**Logic:** A `React.memo` component using `containerPropsEqual` for shallow equality to prevent unnecessary re-renders. Uses `usePlotChartDataSeries` to asynchronously load and map Mendix data series. The `mapSeries` callback converts each series into a plotly scatter trace with `fill: "tonexty"` (fill to previous trace, or to baseline for the first trace). Line color, marker color, and fill color are all resolved via expression values per series. Returns `<ChartWidget>` with fixed layout options (both axes have `fixedrange: true`, `zeroline: true`, and grid color `#d7d7d7`).

**Behavioral constraints from this file:**
- Both axes are non-zoomable (`fixedrange: true`) by default — users cannot zoom into the chart.
- `fill: "tonexty"` means each series fills to the series below it (stacked appearance), or to the x-axis for the first series.
- Line mode is `"lines"` when `lineStyle === "line"`, and `"lines+markers"` when `lineStyle === "lineWithMarkers"`. The `"custom"` lineStyle passes no mode, deferring to custom series options.
- Color expressions are resolved from the first matching item in the datasource (via `getExpressionValue`).
- The widget is wrapped in `React.memo` with custom comparator (`containerPropsEqual`) which ignores click action changes to avoid re-renders on every Mendix action re-creation.

**User-facing:** Yes — the main chart UI rendered in the Mendix page.

**New findings:** The CSS class applied is `"widget-line-chart"` (shared with line chart widget), not an area-chart-specific class. The `responsive: true` config option makes the chart resize automatically with its container.

---

## src/AreaChart.xml

**Purpose:** Widget descriptor declaring all configurable properties for the area chart widget in Mendix Studio/Studio Pro.

**Logic:** Defines a `series` list property where each series has: `dataSet` (static/dynamic), data source, X/Y attribute mappings, `groupByAttribute` (for dynamic series), `aggregationType`, `interpolation`, `lineStyle`, color expressions (line, marker, fill), click action, tooltip hover text, and `customSeriesOptions`. Top-level props include axis labels, legend, grid lines, dimensions (width/height with units), playground slot, advanced options toggle, and custom layout/config JSON strings.

**Behavioral constraints from this file:**
- `offlineCapable="true"` — works in offline Mendix apps.
- `dataSet: "static"` = one series per configuration object, one Mendix data source. `dataSet: "dynamic"` = multiple series from one data source, separated by `groupByAttribute`.
- `groupByAttribute` supports String, Boolean, DateTime, Decimal, Enum, HashString, Integer, Long.
- `aggregationType` defaults to `"none"`. When set, Y values for the same X value are combined before rendering.
- `interpolation`: `"linear"` (straight line segments) or `"spline"` (curved).
- `lineStyle`: `"line"`, `"lineWithMarkers"`, or `"custom"`. Marker color only shown when `lineStyle === "lineWithMarkers"`.
- `widthUnit`: `"percentage"` (default 100%) or `"pixels"`. `heightUnit`: `"percentageOfWidth"` (default 75%), `"pixels"`, or `"percentageOfParent"`.
- `enableAdvancedOptions` gates `customLayout`, `customConfigurations`, `enableThemeConfig`, and per-series `customSeriesOptions` (web platform only).
- `enableThemeConfig` enables loading a JSON config file from the Mendix theme folder.

**User-facing:** Indirectly — every property here is configurable by the Mendix developer in Studio Pro.

**New findings:** The `playground` and `showPlaygroundSlot` properties enable integration with the chart-playground widget, which allows runtime editing of chart options. The widget is categorized under "Charts" in both Studio and Studio Pro toolboxes.

---

## typings/AreaChartProps.d.ts

**Purpose:** Auto-generated TypeScript types for all widget props at runtime and in editor preview.

**Logic:** Exports `SeriesType` (runtime series props), `AreaChartContainerProps` (all widget runtime props), `SeriesPreviewType` and `AreaChartPreviewProps` (editor preview shapes). X/Y attributes at runtime use `ListAttributeValue<string | Date | Big>`. Group-by attribute supports `string | boolean | Date | Big`.

**Behavioral constraints from this file:**
- Color expressions (`staticLineColor`, `dynamicLineColor`, etc.) are `ListExpressionValue<string> | undefined` — optional, returning CSS color strings.
- `customSeriesOptions` is always a `string` (required in runtime, empty string if not set) — but the XML marks it as `required="false"`, so it defaults to `""`.
- `playground` is `ReactNode | undefined` (optional widget slot).
- `AreaChartPreviewProps` includes `renderMode: "design" | "xray" | "structure"` and a `translate` function for i18n.

**User-facing:** Internal — TypeScript compile-time safety only.

**New findings:** The runtime props use `Big` from `big.js` for Decimal/Long/Integer Mendix attribute types, ensuring numeric precision.

---

## src/AreaChart.editorConfig.ts

**Purpose:** Controls Studio Pro property panel visibility, validates configuration, generates structure preview, and provides a custom caption for page explorer.

**Logic:** `getProperties` hides static-only or dynamic-only sub-properties per series based on `dataSet`, hides marker color props when `lineStyle !== "lineWithMarkers"`, and hides playground slot when `showPlaygroundSlot` is false. In advanced mode, shows `customSeriesOptions`, `customLayout`, `customConfigurations`, and `enableThemeConfig`. `getPreview` renders an SVG image preview with optional legend (light/dark mode variants). `check` validates that when a datasource is set, both X and Y attributes are configured. `getCustomCaption` shows the first series datasource name in the page explorer.

**Behavioral constraints from this file:**
- Validation errors (not warnings): missing X attribute or Y attribute when datasource is configured. One error per series, per missing attribute.
- `checkSlot` from shared-charts: error if playground has widgets but `showPlaygroundSlot === false`.
- Structure preview: 375px wide chart SVG + optional 85px legend SVG, wrapped in `withPlaygroundSlot` if slot is shown.
- Advanced mode is hidden on desktop platform; all advanced fields are always visible in Studio Pro.
- `getCustomCaption` returns the first datasource caption with brackets stripped, plus "and N more" when multiple series exist.

**User-facing:** Editor-only — affects Studio Pro panel UX and page explorer label.

**New findings:** The structure preview uses actual chart SVG assets (light/dark variants) rather than generated shapes, giving a realistic visual in structure mode.

---

## src/AreaChart.editorPreview.tsx

**Purpose:** Renders the area chart widget preview inside Mendix Studio's design canvas using static SVG images.

**Logic:** Renders `<ChartPreview>` with `ChartPreview.PlotImage` (the area chart SVG) and `ChartPreview.PlotLegend` (the legend SVG). The preview always uses the light-mode SVGs, regardless of Studio's theme.

**Behavioral constraints from this file:**
- The preview is purely static — no live data, no interactivity.
- The legend SVG is always shown in the preview regardless of `showLegend` prop (the `ChartPreview` component handles legend visibility).
- The alt text for the chart image is "Bubble chart" (a copy-paste error from another chart widget; cosmetic, has no runtime impact).

**User-facing:** Editor canvas only — not visible at runtime.

**New findings:** The preview reuses `ChartPreview` from shared-charts to ensure a consistent appearance across all chart widget previews.

---

## packages/shared/charts/src/components/ChartWidget.tsx

**Purpose:** Shared container that applies dimensions, theme configuration, and merged layout/config/series options before rendering the plotly chart.

**Logic:** Receives widget-level props and chart-type-specific options. Uses `useThemeFolderConfigs` to optionally load JSON config from the Mendix theme folder. Merges default configs with modeler-specified options and theme folder overrides via deep merge. Dispatches resize events via `useDispatchResizeObserver`. Returns `<Fragment>` when `data.length === 0` (data still loading). Renders a `<div class="widget-chart {className}">` with inline dimensions from `getDimensions`.

**Behavioral constraints from this file:**
- Returns an empty Fragment (renders nothing) while data is loading — there is no loading spinner.
- `data.length` is used as the `key` for `<Chart>` — when the number of series changes, the Chart remounts completely.
- Dimensions are computed inline; the chart div responds to the `widthUnit`/`heightUnit` configuration.
- Theme folder config is fetched asynchronously once on mount (only when `enableThemeConfig === true`).

**User-facing:** Yes — renders the outermost chart container div visible in the page.

**New findings:** The resize observer dispatch ensures the plotly chart re-renders correctly when the container size changes (e.g., inside a responsive layout or popup dialog — v5.0.1 fix).

---

## packages/shared/charts/src/hooks/usePlotChartDataSeries.ts

**Purpose:** Core data-loading hook that converts Mendix datasource items into plotly-ready data series arrays.

**Logic:** Uses `useEffect` to recompute series on `[series, mapSerie]` changes, storing results in state. For static series: extracts data points from `staticDataSource`, applies aggregation if configured, then calls `mapSerie` for chart-specific trace props. For dynamic series: groups items by `groupByAttribute` value (supports String, Boolean, DateTime, Big), extracts data points per group, applies aggregation, calls `mapSerie` per group. Null X/Y values are preserved as plotly `null` (gaps in the line). Tooltip hover text is only populated when at least one item has non-empty text.

**Behavioral constraints from this file:**
- Returns `null` (not an empty array) when all series are empty — `ChartWidget` uses `data.length === 0` to detect loading.
- Grouping: if `groupByAttributeValue.status === "loading"` for any item, the entire dynamic series returns `null` (not partial data).
- The `dynamicName` for a group is taken from the first item where the name is non-empty; if empty it stays `"(empty)"`.
- `getExpressionValue` (mapper helper): returns the first available value from all items. If any item's attribute is not "available", returns `undefined`.
- `Big` values (Decimal/Long) are converted to `number` via `.toNumber()` before passing to plotly.
- Click actions are bound to `executeAction(listAction.get(item))` on click — per-item action execution.
- `hoverinfo: "none"` when no hover text is configured; `hoverinfo: "text"` when custom hover text is set.

**User-facing:** Indirectly — determines what data is rendered in the chart.

**New findings:** The hook is lazy (state initialized to `null`), so on first render with data already available, there is one render cycle with no data before the effect fires and populates the series.

---

## packages/shared/charts/src/utils/aggregations.ts

**Purpose:** Aggregates multiple Y values for the same X key into a single data point.

**Logic:** Groups data points by X value (stringified), computes the specified aggregate (count/sum/avg/min/max/median/mode/first/last) over Y values per group. Null Y values are skipped. Aggregated hover text falls back to the first item's text when only one value, or the numeric result when aggregated from multiple.

**Behavioral constraints from this file:**
- Only Y values are aggregated; X values become the string keys (dates are ISO strings, numbers become string representations).
- Null Y values are completely excluded from aggregation (no imputation).
- `"mode"` returns the first-seen most-frequent value. Ties are broken by insertion order.
- When `aggregationType === "none"` (default), this function is bypassed entirely.
- Empty arrays return `NaN` from `computeAggregate`.

**User-facing:** Indirectly — affects chart data when aggregation is configured.

**New findings:** Aggregation happens on the client side after data loading — the Mendix datasource fetches all raw rows and the widget aggregates in-browser.

---

## packages/shared/charts/src/utils/configs.ts

**Purpose:** Defines default plotly layout/config/series options and provides deep-merge utilities for combining modeler, widget-specific, and theme folder options.

**Logic:** `defaultConfigs` sets: font (Open Sans 14px #555), autosize, hover mode "closest" with gray hover labels, fixed margins (60px all sides, 10px pad). Config: no mode bar, no double-click zoom. Series defaults: `connectgaps: true`, `hoverinfo: "none"`, `hoveron: "points"`. `getModelerLayoutOptions` deep-merges default + custom layouts. `useThemeFolderConfigs` asynchronously fetches a JSON file from the theme folder and merges per-chart-type series options.

**Behavioral constraints from this file:**
- `displayModeBar: false` — the plotly toolbar (save as image, zoom, etc.) is hidden by default.
- `doubleClick: false` — double-clicking does not reset the zoom (since axes are already `fixedrange`).
- `connectgaps: true` — null/missing data points are connected rather than shown as gaps.
- Theme folder config must have at least one of `layout`, `configuration`, or `charts` properties to be applied; otherwise a warning is logged.
- Deep merge means widget-specific options override defaults, and theme folder options further override widget-specific options.

**User-facing:** Indirectly — determines plotly chart appearance and interaction behavior.

**New findings:** `connectgaps: true` by default means area charts always show a continuous line even when some data points have null values.

---

## packages/shared/charts/src/utils/equality.ts

**Purpose:** Custom equality function for `React.memo` to prevent re-renders when only click actions change.

**Logic:** `containerPropsEqual` uses `flatEqual` to compare all widget props, treating each series with `traceEqual`. `traceEqual` ignores `staticOnClickAction` and `dynamicOnClickAction` keys (always returns `true` for them), applying `defaultEqual` to all other props.

**Behavioral constraints from this file:**
- Click actions are excluded from re-render triggers — changing a click action alone does not cause a re-render.
- All other prop changes (data, colors, labels, dimensions) trigger a re-render.

**User-facing:** Invisible performance optimization.

**New findings:** This is necessary because Mendix recreates action references on every render; ignoring them prevents the chart from re-rendering unnecessarily when the parent Mendix page re-renders.

---

## packages/shared/charts/src/components/ChartPreview.tsx

**Purpose:** Provides a static preview component for use in Mendix Studio's design canvas across all shared-charts-based widgets.

**Logic:** Renders the playground slot (when `showPlaygroundSlot` is true) above the chart area. The chart area is a fixed-size div (300px or 385px wide, 232px tall) containing the chart image and optional legend image. `PlotImage` renders a 300px-wide `<img>`; `PlotLegend` renders an 85px-wide `<img>`.

**Behavioral constraints from this file:**
- Chart preview is always 232px tall and 300px (no legend) or 385px (with legend) wide — fixed, not responsive.
- Playground slot is hidden via `display: none` when `showPlaygroundSlot` is false (still rendered in DOM).
- No dynamic data is shown — the preview is a static SVG screenshot of a representative chart.

**User-facing:** Editor canvas only.

**New findings:** `ChartPreview.PlotImage` and `ChartPreview.PlotLegend` are static sub-components, enabling each chart widget to supply its own SVG assets without custom logic.

---

## packages/shared/charts/src/utils/preview-utils.ts

**Purpose:** Shared utilities for playground slot validation and structure preview composition.

**Logic:** `checkSlot` returns a validation error if widgets are placed in the playground slot while `showPlaygroundSlot` is false. `withPlaygroundSlot` wraps a structure preview with an additional playground drop zone row when the slot is shown.

**Behavioral constraints from this file:**
- `checkSlot` is a warning-level check surfaced in the Studio Pro property panel — prevents confusion when the slot is hidden but populated.
- Structure preview playground slot is a fixed-size drop zone rendered above the chart SVG.

**User-facing:** Editor-only — affects Studio Pro validation and structure preview.

**New findings:** The playground slot integration is a cross-chart concern handled uniformly by shared-charts utilities, keeping individual chart widgets thin.

---

## CHANGELOG.md

**Summary of relevant versions:**

- **v6.2.1 (2025-07-15):** Updated shared charts dependency.
- **v6.2.0 (2025-06-03):** Fixed aggregate function being removed on plotly 3.0 upgrade (regression fix).
- **v6.0.0 (2025-02-28):** Upgraded plotly.js to version 3.0 (major dependency bump).
- **v5.1.0 (2024-10-28):** Changed bundling to make plotly scannable by package scanners.
- **v5.0.1 (2024-10-15):** Fixed widget not auto-resizing inside a popup dialog (the resize observer fix in ChartWidget).
- **v3.1.3 (2023-11-21):** Fixed entity attributes not selectable in color expression editor after configuring datasource (regression introduced in v4.0).
- **v3.1.2 (2023-09-27):** Removed redundant code for load time improvement.
- **v3.1.0 (2023-06-06):** Updated page explorer caption to show datasource; updated icons and tiles.

**Note:** Versions 4.x and 5.0.0 are absent from the changelog (possible gap in changelog maintenance or entries removed).

**Findings:** The v6.2.0 fix for aggregate with plotly 3.0 corresponds to the `aggregateDataPoints` function in `aggregations.ts` — a regression was introduced when plotly 3.0 changed internal behavior. The popup resize fix (v5.0.1) is implemented via `useDispatchResizeObserver` in `ChartWidget.tsx`.
