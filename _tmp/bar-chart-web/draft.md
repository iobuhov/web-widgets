# Draft: bar-chart-web

Widget package: `packages/pluggableWidgets/bar-chart-web`  
Local dependencies explored: `packages/shared/charts` (`@mendix/shared-charts`), `packages/shared/widget-plugin-platform`

Note: `@mendix/shared-charts` internals (ChartWidget, usePlotChartDataSeries, containerPropsEqual, configs, equality, ChartPreview, preview-utils) are documented in detail in `_tmp/area-chart-web/draft.md`. This draft references those findings and focuses on what is specific to bar-chart-web.

---

## `src/BarChart.tsx`

1. **Purpose:** Root React component of the bar-chart-web widget. Renders a horizontal bar chart using Plotly.js, supporting both grouped and stacked display modes.

2. **Logic:** A `React.memo` component using `containerPropsEqual`. The `layoutOptions` object is computed with `useMemo` so that `barmode` changes (group/stack) cause a re-render without recreating the full options object unnecessarily. The `mapSeries` callback is inlined into `usePlotChartDataSeries` with `useCallback`. Each series trace is type `"bar"` with `orientation: "h"` (horizontal). Bar color is resolved via an optional `barColorExpression` per series; if absent, Plotly uses its auto-color sequence. The `seriesOptions` constant sets `type: "bar"` globally so the default is applied even if the mapper omits it.

3. **Behavior:** Bars are always horizontal (`orientation: "h"`). Y-axis has `rangemode: "tozero"`, ensuring the axis always starts at zero even when all values are positive. Both axes are non-zoomable (`fixedrange: true`). Grid color is `#d7d7d7`. The `barmode` layout property (`"group"` or `"stack"`) is applied dynamically via `useMemo`, meaning changing the barmode prop triggers a Plotly re-layout. No interpolation or line-style options exist — bars have no such styling.

4. **User-facing:** Yes — renders the visible bar chart in the Mendix page.

5. **New learnings:** Unlike area-chart-web (which uses a `SeriesMapper` type annotation), bar-chart-web inlines the mapper directly without the explicit `SeriesMapper<SeriesType>` type annotation, relying on inference. The widget CSS class is `widget-bar-chart` (not shared with other chart types). `barmode` is part of `layoutOptions`, not `seriesOptions` — it must be recalculated per render when `barmode` changes, hence the `useMemo`.

---

## `src/BarChart.editorConfig.ts`

1. **Purpose:** Controls property panel visibility, structure/x-ray preview rendering, validation, and page explorer caption in Mendix Studio Pro.

2. **Logic:**
   - `getProperties`: Hides static or dynamic sub-properties per series based on `dataSet`. For static, hides: `dynamicDataSource`, `dynamicXAttribute`, `dynamicYAttribute`, `dynamicName`, `dynamicTooltipHoverText`, `groupByAttribute`, `dynamicBarColor`. For dynamic, hides the static counterparts. Advanced options (`customLayout`, `customConfigurations`, `enableThemeConfig`, `customSeriesOptions`) are hidden when `enableAdvancedOptions` is false on web. Desktop hides `enableAdvancedOptions` entirely. Web calls `transformGroupsIntoTabs`.
   - `getPreview`: Selects SVG image based on both `values.barmode` (group vs stack) AND `isDarkMode`, giving four distinct preview images. Chart image is 375px; legend is 85px.
   - `check`: Same pattern as area-chart-web — validates X and Y attributes are set when datasource is configured.
   - `getCustomCaption`: Same pattern — first series datasource caption with bracket stripping and "and N more" suffix.

3. **Behavior:** The structure preview is barmode-aware — switching between group and stack in the property panel immediately changes the preview SVG image. No marker-style conditional hiding exists (bar chart has no line style option).

4. **User-facing:** No — Studio Pro design-time only.

5. **New learnings:** Bar-chart-web has no `lineStyle`-equivalent property, so the property panel does not have conditional marker visibility logic (unlike area-chart-web). The `barmode` selection directly affects the preview SVG, providing meaningful visual feedback before the chart is published.

---

## `src/BarChart.editorPreview.tsx`

1. **Purpose:** Live canvas preview in Mendix Studio. Switches between grouped and stacked bar chart SVG images based on `barmode`.

2. **Logic:** Selects `BarChartGrouped` or `BarChartStacked` light-mode SVG based on `props.barmode === "group"`. Passes the selected image to `ChartPreview.PlotImage` and a fixed legend to `ChartPreview.PlotLegend`. All three SVGs are light-mode only.

3. **Behavior:** The canvas preview updates when barmode is toggled — developers get immediate visual confirmation of the grouped vs stacked layout choice. Like area-chart-web, the alt text is `"Bubble chart"` — a copy-paste artifact from another chart widget.

4. **User-facing:** No — Studio design canvas only.

5. **New learnings:** The preview correctly switches images based on `barmode`, making it one of the more informative chart previews in the suite. The legend image (`BarChartLegend`) is the same for both modes — there's only one legend variant.

---

## `src/BarChart.xml`

1. **Purpose:** Declarative property schema for the widget, consumed by Mendix Studio Pro to generate the property panel.

2. **Logic:** Per-series properties: `dataSet`, datasources (static/dynamic), X/Y attributes, `groupByAttribute`, `aggregationType` (9 options), tooltip text, `barColor` (expression), click actions, `customSeriesOptions`. Top-level: `barmode` (`"group"` default, `"stack"`), axis labels, `showLegend` (default true), `gridLines` (default none), dimensions (100% width, 75% of width height), `enableAdvancedOptions` (default false), `enableThemeConfig` (default false), custom layout/config JSON strings.

3. **Behavior:** Widget ID is `com.mendix.widget.web.barchart.BarChart`. Offline capable (`offlineCapable="true"`). `barColor` accepts `ListExpressionValue<string>` — a per-bar color expression, not a per-series constant. Unlike area-chart-web, there are no interpolation, line style, fill color, or marker color properties — bar charts only have bar color.

4. **User-facing:** No — build-time schema.

5. **New learnings:** The absence of fill color, line color, and marker color properties (compared to area-chart-web) reflects that bar charts have a simpler styling model: only one color per bar (the bar fill color, mapped as `marker.color` in Plotly).

---

## `typings/BarChartProps.d.ts`

1. **Purpose:** Auto-generated TypeScript types from `BarChart.xml`. Adds `BarmodeEnum` and bar-specific color types compared to other chart widgets.

2. **Logic:** `SeriesType` has `staticBarColor?: ListExpressionValue<string>` and `dynamicBarColor?: ListExpressionValue<string>` — single color per series (no fill/line/marker split). `BarChartContainerProps` includes `barmode: BarmodeEnum` (required, not optional). No `InterpolationEnum` or `LineStyleEnum`.

3. **Behavior:** `barmode` is a required prop at runtime — it always has a value (defaulted to `"group"` by the XML). The `SeriesType` interface is leaner than area-chart-web's: fewer optional color properties, no interpolation, no line style.

4. **User-facing:** No — compile-time only.

5. **New learnings:** `BarmodeEnum` is `"group" | "stack"`. There is no `"overlay"` mode (a valid Plotly barmode) — the widget intentionally limits to group and stack. Custom Plotly configuration could override this via `customLayout`.

---

## `src/__tests__/BarChart.spec.tsx`

1. **Purpose:** Jest tests verifying the Plotly data shapes passed to `ChartWidget` for bar chart configurations.

2. **Logic:** Three tests: (1) `type: "bar"` is set on all series data, (2) `staticBarColor` expressions are resolved and applied as `marker.color`, (3) aggregation with `"none"` and `"avg"` types both produce numeric x/y arrays and exactly two series in output. Uses `setupBasicBarSeries` helper that wraps `setupBasicSeries` from shared-charts and adds `staticBarColor`.

3. **Behavior:** The bar color test verifies that `undefined` barColor produces `marker: { color: undefined }` — the Plotly default palette takes over. The test for `color: "red"` confirms the expression value is resolved and applied directly. The aggregation test validates that `data.length === 2` regardless of aggregation type.

4. **User-facing:** No — test-time only.

5. **New learnings:** Unlike the area-chart test which asserts `fill: "tonexty"`, the bar chart test asserts `type: "bar"` on the data. There is no test for `orientation: "h"` (horizontal orientation) — this is untested behavior in the widget.

---

## `CHANGELOG.md`

1. **Purpose:** Documents the version history of bar-chart-web, recording behavioral fixes, dependency upgrades, and UX improvements across all releases from v3.1.0 through v6.2.1.

2. **Logic:** Eight versioned entries are documented:
   - **v6.2.1 (2025-07-15):** Updated shared charts dependency (maintenance).
   - **v6.2.0 (2025-06-03):** Fixed aggregation being removed on Plotly 3.0 — a regression introduced by the Plotly 3.0 upgrade in v6.0.0.
   - **v6.0.0 (2025-02-28):** Upgraded Plotly.js dependency to version 3.0 (major version bump).
   - **v5.1.0 (2024-10-28):** Changed bundling to make Plotly scannable by package scanners.
   - **v5.0.1 (2024-10-15):** Fixed the widget not auto-resizing inside popup dialogs.
   - **v3.1.3 (2023-11-21):** Fixed barColor expression editor not accepting entity attribute references — a regression introduced in v4.0 that prevented developers from using entity attributes in the barColor expression. The fix restored the behavior present before v4.0.
   - **v3.1.2 (2023-09-27):** Removed redundant code to improve widget load time.
   - **v3.1.0 (2023-06-06):** Updated page explorer caption to display datasource; updated light and dark icons and tiles.

3. **Behavioral constraints from this file:**
   - **Popup dialog auto-resize (v5.0.1):** The widget correctly auto-resizes when placed inside a popup/modal dialog. Before v5.0.1, the chart dimensions were fixed on mount and did not react to dialog open/resize events — this is a confirmed behavioral placement constraint for popup usage.
   - **barColor expression and entity attributes (v3.1.3):** The `barColor` expression editor supports entity attribute references. This capability was broken in versions between v4.0 and v3.1.2 (inclusive of those v3.x patch releases that preceded the fix). As of v3.1.3, developers can reference entity attributes (e.g., `$currentObject/StatusColor`) in the barColor expression — this is a confirmed behavioral capability.
   - **Aggregation + Plotly 3.0 compatibility (v6.0.0/v6.2.0):** Aggregation (`aggregationType` prop) was broken by the Plotly 3.0 upgrade in v6.0.0 and was restored in v6.2.0. The Plotly dependency version is 3.0 as of v6.0.0.

4. **User-facing:** No — the CHANGELOG is developer-facing documentation, not visible to end users. The fixes it describes (popup resize, color expression, aggregation) do affect the user-facing runtime behavior.

5. **New findings:**
   - The CHANGELOG reveals a non-linear versioning pattern: v3.1.x patch releases (v3.1.3) were published after the v4.0 major line already existed, indicating the v3.1.x branch was a concurrent maintenance track. This explains the gap between v3.1.3 (2023) and v5.0.1 (2024) in the log.
   - The Plotly 3.0 upgrade (v6.0.0, Feb 2025) was a breaking change for aggregation — a fact not visible from source code alone. Any deployment on v6.0.0 or v6.0.x without upgrading to v6.2.0+ will have broken aggregation.
   - The bundling change in v5.1.0 (making Plotly scannable) is a deployment/security constraint: environments using package scanners (e.g., for license compliance or vulnerability auditing) required this change for Plotly to be properly analyzed.

---

## `@mendix/shared-charts` — shared components (reference)

The following shared-charts exports are used by bar-chart-web in the same way as area-chart-web (see area-chart-web draft for full details):

- **`ChartWidget`**: Container rendering Plotly with merged layout/config/series options. `type="BarChart"` enables theme folder config loading.
- **`usePlotChartDataSeries`**: Bridges Mendix `ListValue` datasources to Plotly trace arrays; returns `null` while loading.
- **`containerPropsEqual`**: Memoization comparator that skips `staticOnClickAction`/`dynamicOnClickAction` to prevent spurious re-renders.
- **`getModelerLayoutOptions`/`getModelerConfigOptions`/`getModelerSeriesOptions`**: Deep-merge utilities; bar chart adds `barmode` to layout options via `useMemo`.
- **`ChartPreview`**: Static image preview component; bar-chart-web passes barmode-selected SVGs.
- **`checkSlot`/`withPlaygroundSlot`**: Playground validation and structure preview wrapping.

Bar-chart-specific behavior in shared-charts: The `type: "bar"` set in `barChartSeriesOptions` is deep-merged as a default for all series. The `barmode` layout option is applied per-render since it varies with the `barmode` prop.
