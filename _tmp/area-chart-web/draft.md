# Draft: area-chart-web

Extracted from `packages/pluggableWidgets/area-chart-web/` and local dependency `packages/shared/charts/`.

---

## src/AreaChart.tsx

**Purpose:** Root React component of the Area Chart widget. It is the runtime entry point rendered in the Mendix client.

**Logic:** Uses `usePlotChartDataSeries` to transform the configured series into Plotly trace objects, then renders `ChartWidget` with fixed axis and config options. A `mapSeries` callback converts each series configuration into a Plotly scatter-with-fill trace. Line/marker/fill colors are resolved by calling `getExpressionValue` on the relevant expression attribute.

**Behavioral documentation:** The chart always uses Plotly type `scatter` with `fill: "tonexty"`, which draws the area between consecutive traces. `mode` is set to `"lines"` or `"lines+markers"` based on `lineStyle`. The x/y axes have `fixedrange: true` (no zoom). Both axes display gridlines in `#d7d7d7` and a zero-line in the same color. The component is wrapped in `memo` with `containerPropsEqual` to avoid unnecessary re-renders when on-click actions change but data does not.

**User-facing:** Yes. This is the widget rendered to end-users in a Mendix page.

**New learnings:** The fill color, line color, and marker color are all evaluated per-data-series using the first available item from the data source (via `getExpressionValue`). Colors are optional — when not configured, Plotly applies its defaults.

---

## src/AreaChart.xml

**Purpose:** Mendix widget definition file. It declares all configurable properties, their types, captions, default values, and groupings visible in Studio Pro.

**Logic:** Defines the `com.mendix.widget.web.areachart.AreaChart` pluggable widget. Properties are organized into tabs: General (Data source, Visibility, Common), Dimensions, and Advanced. The `series` property is a list of objects, each configurable with data set type, data source, axis attributes, aggregation, tooltip, interpolation, line style, and color expressions.

**Behavioral documentation:** The `dataSet` enum switches between `static` (single series) and `dynamic` (multiple series grouped by an attribute). When `dataSet` is `dynamic`, a `groupByAttribute` drives how data source items are partitioned into individual traces. The `aggregationType` default is `none`. `lineStyle` defaults to `line`; the `lineWithMarkers` value enables a separate marker color property. Width defaults to 100% and height to 75% of width.

**User-facing:** No (design-time only). This file drives the Studio Pro property panel UI.

**New learnings:** The widget is marked `offlineCapable="true"`, meaning it can run in offline-first Mendix apps. It belongs to both `studioProCategory: Charts` and `studioCategory: Charts`.

---

## typings/AreaChartProps.d.ts

**Purpose:** Auto-generated TypeScript interface declarations derived from `AreaChart.xml`. Provides strongly-typed props for runtime and preview components.

**Logic:** Declares `SeriesType` (runtime), `SeriesPreviewType` (Studio Pro preview), `AreaChartContainerProps` (full runtime widget props), and `AreaChartPreviewProps` (Studio Pro preview props). Enum types are declared for `DataSetEnum`, `AggregationTypeEnum`, `InterpolationEnum`, `LineStyleEnum`, `GridLinesEnum`, `WidthUnitEnum`, `HeightUnitEnum`.

**Behavioral documentation:** `AggregationTypeEnum` supports 10 values: `none`, `count`, `sum`, `avg`, `min`, `max`, `median`, `mode`, `first`, `last`. `InterpolationEnum` has `linear` and `spline`. `LineStyleEnum` includes `line`, `lineWithMarkers`, and `custom`. The `series` prop in `AreaChartContainerProps` is an array of `SeriesType`, making multi-series charts first-class. The `AreaChartPreviewProps.className` is deprecated since 9.18.0 in favor of `class`.

**User-facing:** No (TypeScript compilation artifact, not visible to end-users).

**New learnings:** The distinction between `static*` and `dynamic*` fields in `SeriesType` is fundamental — color and data-source props are duplicated under both prefixes. The runtime component uses whichever set is active based on `dataSet`.

---

## src/AreaChart.editorConfig.ts

**Purpose:** Provides Studio Pro design-time behavior: property visibility rules, structure preview rendering, validation, and caption generation.

**Logic:** `getProperties` hides irrelevant properties per series based on `dataSet` and `lineStyle`. It also conditionally hides advanced options (`customLayout`, `customConfigurations`, `enableThemeConfig`, `customSeriesOptions`) unless `enableAdvancedOptions` is true. On web platform, groups are transformed into tabs. `getPreview` returns an SVG-based structure preview with optional legend. `check` validates that X and Y axis attributes are set whenever a data source is configured. `getCustomCaption` generates a human-readable caption for the widget in page explorer.

**Behavioral documentation:** Key visibility constraints: (1) When `dataSet = static`, all `dynamic*` properties are hidden and vice versa. (2) Marker color properties are hidden unless `lineStyle = lineWithMarkers`. (3) All advanced options are hidden unless `enableAdvancedOptions = true` (web only). Validation errors reference specific property paths like `series/{index}/staticXAttribute`. The playground slot property is hidden if `showPlaygroundSlot = false`.

**User-facing:** No (Studio Pro design-time only).

**New learnings:** The `check` function returns a `Problem[]` from `@mendix/pluggable-widgets-tools`. Validation errors will block the user from publishing the app until resolved. The caption uses the first series' data source caption and appends "+ N more" for multi-series.

---

## src/AreaChart.editorPreview.tsx

**Purpose:** Renders the widget's visual appearance in Studio Pro's page canvas (design mode, x-ray mode, structure mode).

**Logic:** Delegates to `ChartPreview` from `@mendix/shared-charts/preview`, passing SVG images for the chart body and legend. The `alt` text in `PlotImage` incorrectly says "Bubble chart" (copy-paste artifact, not a behavioral issue).

**Behavioral documentation:** In design mode, the preview shows a static SVG of an area chart shape. If `showLegend` is true, a legend SVG is displayed beside the chart image. The playground slot dropzone is rendered above the chart if `showPlaygroundSlot` is true.

**User-facing:** No (Studio Pro canvas preview only).

**New learnings:** Light/dark mode SVG variants are selected automatically by the `isDarkMode` flag passed by Studio Pro to `getPreview` in `editorConfig.ts`. The preview component uses a fixed 300×232px area.

---

## src/__tests__/AreaChart.spec.tsx

**Purpose:** Unit test suite verifying the AreaChart component produces correct Plotly trace structures.

**Logic:** Mocks `ChartWidget` and inspects its call arguments. Six test cases: fill type (`tonexty`), mode based on `lineStyle`, line shape based on `interpolation`, line color, marker color, and area fill color. One additional test verifies aggregation passes data arrays of the right shape.

**Behavioral documentation:** The test confirms `fill: "tonexty"` is always set regardless of configuration, establishing it as a fixed behavioral constant. The `lineWithMarkers` style maps to `mode: "lines+markers"` and `line` maps to `mode: "lines"`. Color undefined when not configured (Plotly defaults apply). Tests use `setupBasicSeries` from shared-charts for fixture construction.

**User-facing:** No (test file).

**New learnings:** The test uses `@testing-library/react` and `happy-dom` as the jest environment. `setupBasicSeries` is a shared utility that constructs a minimal valid `SeriesType` object — useful for understanding the minimum required props.

---

## @mendix/shared-charts — hooks/usePlotChartDataSeries.ts

**Purpose:** Central data-loading hook for all Mendix chart widgets. Transforms Mendix `ListValue` data sources into Plotly-compatible trace objects.

**Logic:** Iterates over the `series` array; delegates to `loadStaticSeries` or `loadDynamicSeries` based on `dataSet`. Static series extracts x/y values directly from the data source. Dynamic series groups items by `groupByAttribute` and creates one trace per group. Aggregation is applied if `aggregationType !== "none"`. Click actions are bound per-item. Returns `null` while data is loading.

**Behavioral documentation:** The hook uses `useState` + `useEffect` for async data loading. Returning `null` triggers the `ChartWidget` to render a `Fragment` (empty). Groups in dynamic mode are built by comparing attribute values — Date comparison uses `.getTime()`, Big.js uses `.eq()`. Series with no loaded items return `null` and are filtered out. The `customSeriesOptions` JSON string is attached to each trace for later deep-merge in `ChartView`.

**User-facing:** No (internal hook). Its output directly controls what the end-user sees.

**New learnings:** `mapperHelpers.getExpressionValue` returns the value from the first available item in the data source. This means expression-based colors are evaluated from a representative item, not per-point. Null x/y values are passed as `null` (Plotly renders gaps by default, but `connectgaps: true` in the default series options fills them).

---

## @mendix/shared-charts — utils/aggregations.ts

**Purpose:** Implements all 10 aggregation functions for the chart widgets. Groups data points by x-value and reduces multiple y-values to a single one.

**Logic:** `aggregateDataPoints` groups x/y pairs using a string key derived from the x value. For each group, `computeAggregate` computes the result. Null y-values are skipped. Hover text is also aggregated — when multiple values exist, the aggregated numeric value is used as hover text.

**Behavioral documentation:** Aggregation happens client-side after data is fetched from the Mendix data source. `count` returns the number of items per x-value (ignoring y). `mode` returns the most frequent value; on ties, the first encountered wins. `median` uses standard sorted-midpoint formula. Null x-values are keyed as empty string `""` — multiple null-x rows all aggregate together under that key.

**User-facing:** No (internal utility). Results are visible to end-users through the rendered chart.

**New learnings:** The aggregation type `none` returns the original data unchanged. All other types perform a full group-by-x operation, which means x-values in the output are always strings (converted via `toString()` or `toISOString()`), which could affect sorting/rendering for date and numeric axes.

---

## @mendix/shared-charts — utils/configs.ts

**Purpose:** Manages Plotly layout, config, and series option merging, including theme folder configuration loading.

**Logic:** Exports `defaultConfigs` with baseline Plotly layout, configuration, and series options. `getModelerLayoutOptions`, `getModelerConfigOptions`, `getModelerSeriesOptions` merge defaults with widget-specific overrides using `deepmerge`. `getCustomLayoutOptions` maps Mendix props (`showLegend`, axis labels, gridlines mode) to Plotly layout fields. `useThemeFolderConfigs` asynchronously fetches a JSON file from the theme folder if `enableThemeConfig` is true.

**Behavioral documentation:** Default config disables the mode bar (`displayModeBar: false`) and double-click zoom (`doubleClick: false`). Default series options set `connectgaps: true` (line connects over null values) and `hoverinfo: "none"` (tooltip suppressed unless hover text is provided). Grid lines are mapped: `"both"` → both axes show grid; `"horizontal"` → y-axis only; `"vertical"` → x-axis only. Theme folder config can override layout, configuration, and per-chart-type series options.

**User-facing:** No (configuration utility). Settings affect the visual appearance end-users see.

**New learnings:** The theme folder config supports chart-type-specific series overrides under a `charts` key (e.g., `charts.AreaChart`). Warnings are logged to the console if the theme config file has structural errors.

---

## @mendix/shared-charts — utils/equality.ts

**Purpose:** Custom equality functions used with `React.memo` to avoid unnecessary re-renders of the Area Chart container.

**Logic:** `containerPropsEqual` uses `flatEqual` to compare all props except `series`, which is compared element-by-element using `traceEqual`. `traceEqual` skips comparison of `staticOnClickAction` and `dynamicOnClickAction` (always returns `true` for those keys), preventing re-renders when action references change.

**Behavioral documentation:** Action props are intentionally excluded from equality checks because Mendix re-creates `ListActionValue` objects on every render cycle even when the underlying action hasn't changed. Without this exclusion, every page state change would cause the chart to fully re-render. This is a performance-critical behavioral constraint.

**User-facing:** No (internal memoization utility). Prevents visible re-render flicker for end-users.

**New learnings:** The `flatEqual` utility from `@mendix/widget-plugin-platform` performs a shallow key-by-key comparison with an override callback. `containerPropsEqual` is passed directly to `memo(AreaChart, containerPropsEqual)` in `AreaChart.tsx`.

---

## @mendix/shared-charts — components/ChartWidget.tsx

**Purpose:** Layout and dimension wrapper that sits between the specific chart component (AreaChart) and the core Chart rendering logic.

**Logic:** Computes initial layout/config/series options by merging defaults with modeler settings and theme folder overrides. Attaches a resize observer to the container div. Renders `Chart` with computed options. Returns an empty Fragment if `data.length === 0`.

**Behavioral documentation:** The container div uses CSS dimensions computed from `getDimensions` (width/height with their respective unit types). The resize observer dispatches resize events so Plotly can reflow. When `data` is empty (all series still loading), the widget renders nothing — no loading spinner, no placeholder.

**User-facing:** No (internal wrapper). The div it renders is the visible chart container.

**New learnings:** Options are memoized with `useMemo` keyed on all relevant layout inputs. This means grid-lines, axis labels, legend visibility, and theme configs are re-computed only when those specific props change.

---

## @mendix/shared-charts — components/Chart.tsx

**Purpose:** Bridges the chart widget with the optional playground slot by providing a `PlaygroundContext`.

**Logic:** Calls `useChartController` which applies the playground's edits (modified layout/config/series) on top of the base props. Renders `ChartView` with the (possibly modified) props and provides the playground context to the playground slot children.

**Behavioral documentation:** When playground is not active (`playground` prop is null/undefined), `useChartController` returns the original props unchanged. The playground slot renders as a React subtree consuming the `PlaygroundContext`, enabling live JSON editing of Plotly options in development mode.

**User-facing:** No (internal component). Its visible output is the rendered chart via `ChartView`.

**New learnings:** The `key={data.length}` on `Chart` in `ChartWidget` causes a full remount whenever the number of traces changes, resetting all playground-modified state.

---

## @mendix/shared-charts — components/ChartView.tsx

**Purpose:** The final rendering layer that creates a `react-plotly.js` Plot component from processed trace data.

**Logic:** Merges `layoutOptions` with `customLayout` JSON, `configOptions` with `customConfig` JSON, and applies `seriesOptions` as base options for each trace via deep-merge. The click handler resolves the clicked `ObjectItem` from the trace's `dataSourceItems` array and calls the bound `onClick`. Dispatches a window resize event once on first data ready to fix an initial sizing bug in Plotly.

**Behavioral documentation:** `customLayout` and `customConfig` are JSON strings that deep-merge on top of programmatic options, giving developers full override capability. Array merging in `createPlotlyData` uses a custom strategy: it concatenates non-undefined values from both arrays. The `dataSourceItems` array is deleted before passing to Plotly (prevents circular reference issues). `onClick` handler resolves `pointIndices?.at(-1)` as the item index when aggregation is used (aggregated points span multiple original items; the last index is taken).

**User-facing:** No (internal). Produces the Plotly chart SVG visible to end-users.

**New learnings:** The `PREVENT_DEFAULT_INLINE_STYLES_BY_PASSING_EMPTY_OBJ` pattern prevents `react-plotly.js` from setting an inline `style` attribute, letting CSS fully control the chart container. This is a deliberate override of the library's default behavior.

---

## @mendix/shared-charts — components/ChartPreview.tsx

**Purpose:** Renders the chart widget's static preview inside Studio Pro's page canvas.

**Logic:** Renders a fixed-size 300×232px `div` containing the chart image SVG and optionally a legend SVG. If `showPlaygroundSlot` is true, renders a playground dropzone above the chart. Light/dark images are selected externally (in `editorConfig.ts`) and passed as `image`/`legend` props.

**Behavioral documentation:** Legend is shown only when `showLegend` is true, and when visible the container expands to 385px wide to accommodate both chart (300px) and legend (85px). The playground dropzone has fixed height of 58px.

**User-facing:** No (Studio Pro design-time only).

**New learnings:** `ChartPreview` exposes `PlotImage` and `PlotLegend` as static sub-components for callers to construct the image nodes with correct sizing. The `editorPreview.tsx` uses `ChartPreview.PlotImage` and `ChartPreview.PlotLegend` sub-components.

---

## @mendix/shared-charts — utils/preview-utils.ts (checkSlot / withPlaygroundSlot)

**Purpose:** Provides validation and structure-preview construction helpers for the playground slot.

**Logic:** `checkSlot` returns a validation error if widgets exist in the playground slot but `showPlaygroundSlot` is false — a common misconfiguration. `withPlaygroundSlot` wraps the chart's structure preview in a container with a dropzone row when the playground is enabled.

**Behavioral documentation:** This validation prevents silent misconfiguration where a developer adds content to the playground slot but doesn't enable it, causing the content to be hidden at runtime.

**User-facing:** No (design-time validation only).

**New learnings:** The structure preview API (`container`, `rowLayout`, `dropzone`) from `@mendix/widget-plugin-platform/preview/structure-preview-api` is used to build visual design-mode representations that show widget structure without actually rendering the runtime component.

---

## CHANGELOG.md

**Purpose:** Version history tracking changes to the widget.

**Behavioral documentation:**
- **6.2.1 (2025-07-15):** Updated shared charts dependency (no user-facing change).
- **6.2.0 (2025-06-03):** Fixed aggregation removal bug introduced by plotly 3.0 upgrade.
- **6.0.0 (2025-02-28):** Upgraded plotly.js to version 3.0 (breaking change in major version, likely API surface changes).
- **5.1.0 (2024-10-28):** Changed bundling for plotly to enable package scanner compatibility.
- **5.0.1 (2024-10-15):** Fixed auto-resize issue when widget is inside a popup dialog.
- **3.1.3 (2023-11-21):** Restored ability to use entity attributes in color expression editors (regression fix).
- **3.1.0 (2023-06-06):** Updated page explorer caption to show datasource; updated icons/tiles.

**User-facing:** No (developer changelog). Documents user-visible behavioral changes.

**New learnings:** The jump from 3.1.x directly to 5.0.x (skipping 4.x) in the changelog suggests some version history was omitted. The 6.2.0 fix for aggregation is important context: plotly 3.0 introduced a change that broke the aggregation pipeline, which was patched in 6.2.0.
