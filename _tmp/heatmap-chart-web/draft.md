# Draft: heatmap-chart-web

Widget: `heatmap-chart-web`  
Task: EX-031  
Agent: worker  
Date: 2026-05-09

---

## src/HeatMap.xml

**1. What is the purpose of this file?**
This is the Mendix widget descriptor. It declares the widget identity (`com.mendix.widget.web.heatmap.HeatMap`), its properties schema, and Studio/StudioPro metadata. It is the authoritative source for all configurable props and their types.

**2. What kind of logic is described in this file?**
Declarative configuration only — no runtime logic. Defines property groups, types (datasource, attribute, enumeration, boolean, string, object list, action, textTemplate, widgets), defaults, and captions.

**3. What part of behavior can be documented from this file?**
- `seriesDataSource` (datasource, required, isList=true) is the root data source. All axis/value attributes and events are scoped to it.
- `seriesValueAttribute` accepts Decimal, Integer, Long — numeric only; this is the "heat" value placed at each (x, y) cell.
- `seriesItemSelection` supports None or Single selection types.
- X axis (`horizontalAxisAttribute`) accepts String or Enum; Y axis (`verticalAxisAttribute`) accepts String or Enum.
- Sort attributes for both axes accept Decimal, Long, Integer, String, AutoNumber, DateTime — a broader set than the display attributes.
- `horizontalSortOrder` and `verticalSortOrder` default to ascending.
- `showScale` (boolean, default false) controls colorbar visibility.
- `gridLines` enum: none (default), horizontal, vertical, both.
- `scaleColors` is a list of objects with `valuePercentage` (integer, 0–100) and `colour` (CSS string). Requires at least two entries (0% and 100%) to take effect; otherwise defaults are used.
- `smoothColor` (boolean, default false) enables gradual gradient between data points.
- `showValues` (boolean, default false) overlays the numeric Z values as annotations on each cell.
- `valuesColor` (string, optional) overrides annotation font color.
- `onClickAction` is scoped to `seriesDataSource`, enabling per-object actions.
- `tooltipHoverText` (textTemplate, scoped to datasource) provides custom hover text per cell.
- Dimension props: `widthUnit` (percentage/pixels, default percentage, 100), `heightUnit` (percentageOfWidth/pixels/percentageOfParent, default percentageOfWidth, 75).
- Advanced props (hidden unless `enableAdvancedOptions`=true): `customLayout`, `customConfigurations`, `customSeriesOptions`.
- `enableThemeConfig` loads configuration from the Mendix theme folder.
- Widget is `offlineCapable="true"`.

**4. Is it user-facing?**
Not directly — this file configures the design-time experience in Mendix Studio(Pro). End users don't see it at runtime.

**5. What new did you learn from this file?**
The widget is categorized under "Charts" in both studioProCategory and studioCategory. The value attribute is restricted to numeric types (Decimal/Integer/Long), meaning non-numeric heat values are not supported. Axis labels (x/y) accept textTemplate for dynamic label content. The playground slot system is exposed for advanced customization.

---

## src/HeatMap.tsx

**1. What is the purpose of this file?**
The main React container component for the heatmap widget. It composes `useHeatMapDataSeries` (data hook), `createHeatMapAnnotation` (annotation utility), and `ChartWidget` (shared chart renderer) into the complete widget output.

**2. What kind of logic is described in this file?**
Orchestration: wires props to the data hook, builds Plotly layout options (including cell-value annotations when `showValues` is true), and renders `ChartWidget` with merged layout/config/series options. Uses `useMemo` to avoid recomputing annotations unless `heatmapChartData`, `showValues`, or `valuesColor` change.

**3. What part of behavior can be documented from this file?**
- Fixed Plotly layout defaults: font color #555 size 12, left margin 80px, legend font Open Sans 14pt.
- Both axes have `fixedrange: true` (no zoom/pan on axes), no tick marks shown.
- Y-axis title has standoff=5.
- Series type is always "heatmap"; `xgap` and `ygap` are both 1 (small gap between cells).
- `connectgaps: false` — missing data points (null z-values) are not interpolated.
- `hoverinfo: "none"` at series level (tooltip is handled separately via `tooltipHoverText`).
- Colorbar is positioned at the top-right (`y:1, yanchor:"top", ypad:0, xpad:5`).
- `showLegend` prop of ChartWidget is driven by `showScale` — the scale/colorbar and legend toggle together.
- When `showValues=true`, annotations are generated for every cell: iterates `z` matrix, uses x/y axis values and the formatted value string (`.toLocaleString()`), applies `valuesColor` or falls back to #555.
- Empty dataset with `showValues=true` is handled safely (z array is empty, no annotation loop).
- Grid lines mode, custom layout/config/series options, theme config, and playground slot are all forwarded to `ChartWidget`.

**4. Is it user-facing?**
Yes — this is the runtime component rendered in the browser. All visual output (chart, colorbar, annotations, labels) originates here.

**5. What new did you learn from this file?**
The annotation z-values are formatted with `.toLocaleString()`, so they will appear locale-formatted (e.g., commas for thousands). The colorbar `outlinecolor` is hardcoded to #9ba492. `connectgaps: false` is a significant behavioral constraint — gaps in data create visible blank/null cells rather than interpolated values.

---

## src/hooks/data.ts

**1. What is the purpose of this file?**
Provides `useHeatMapDataSeries`, the central data-processing hook. Transforms raw Mendix datasource items into a Plotly-ready heatmap data structure: unique x/y axis arrays and a z-value matrix.

**2. What kind of logic is described in this file?**
Data transformation, sorting, deduplication, matrix construction, click handling, colorscale processing. Uses `useState`+`useEffect` for datasource subscription, `useRef` for ObjectItem lookup map, `useMemo` for derived chart data, and `useCallback` for click handler stability.

**3. What part of behavior can be documented from this file?**
- Data is read once the datasource status is `Available` with items present.
- `seriesValueAttribute` values (Big/Decimal) are converted via `.toNumber()`.
- Axis attribute values: Big → number, string → string, Date → Date (via `formatValueAttribute`). This means numeric axis values are rendered as numbers, not strings.
- Axis display values are stringified with `.toLocaleString()` in the final output, so locale-specific formatting applies.
- Sorting is independent for each axis: vertical sort is applied first, then horizontal sort. Both support asc/desc. Sort attributes can differ from display attributes.
- `getUniqueValues` uses a Set — order preservation follows insertion order (JavaScript Set semantics), so sort order before deduplication determines axis order.
- The z-matrix is built as `z[yIndex][xIndex]` — row = y-axis value, column = x-axis value. Missing combinations produce `null` (not 0).
- Click handler resolves item by: first checks if the passed ObjectItem is non-null; if null/undefined, finds the matching data point by x/y/z coordinates in `heatmapChartData`. Executes `onClickAction` and sets Single selection if configured.
- `colorscale` defaults to `[[0, "#17347B"], [0.5, "#0595DB"], [1, "#76CA02"]]` (dark blue → light blue → green) when fewer than 2 `scaleColors` entries are provided.
- Custom colorscale: sorted by `valuePercentage` ascending, mapped to `[percentage/100, color]` pairs.
- `zsmooth`: `"best"` when `smoothColor=true`, `false` otherwise.
- `hoverinfo`: `"text"` + `hovertext` matrix when `tooltipHoverText` is set; `"none"` otherwise.
- `dataSourceItems` is always set to `[]` in returned data (selection handled separately via `seriesItemSelection`).

**4. Is it user-facing?**
Indirectly — it is internal logic, but its output directly determines what the user sees in the chart.

**5. What new did you learn from this file?**
The click resolution fallback (matching by x/y/z value when item reference is null) means clicking a cell can still trigger actions even when the direct item reference is lost. The `objectMap` ref maintains a stable id→ObjectItem mapping across re-renders. The vertical sort is applied before horizontal sort when both are configured, which is a subtle ordering constraint.

---

## src/utils/annotation.ts

**1. What is the purpose of this file?**
Provides `createHeatMapAnnotation`, a pure factory function that generates a Plotly `Annotations` object for displaying a text value at a specific (x, y) cell position on the heatmap.

**2. What kind of logic is described in this file?**
Simple data transformation: accepts cell coordinates (x, y as strings), text value, and optional color, returns a Plotly annotation config object. No side effects.

**3. What part of behavior can be documented from this file?**
- Annotations use `xref: "x"` and `yref: "y"` — positioned in data coordinates, not pixel coordinates.
- Font: Open Sans, size 14, color defaults to #555 (overridable via `colorValue`).
- `showarrow: false` — annotations are text-only, no pointer arrow.
- All parameters are optional (x, y, text, colorValue), so partial annotations are possible without errors.

**4. Is it user-facing?**
Indirectly — annotations appear on the rendered chart when `showValues=true`, so end users see the output.

**5. What new did you learn from this file?**
The annotation font is hardcoded to Open Sans 14 regardless of theme. The default text color (#555) matches the axis font color set in `HeatMap.tsx`, maintaining visual consistency.

---

## src/HeatMap.editorConfig.ts

**1. What is the purpose of this file?**
Studio(Pro) editor configuration: controls property visibility/grouping, generates the structure preview image, supplies the custom caption for the page explorer, and validates the playground slot.

**2. What kind of logic is described in this file?**
Conditional property hiding based on widget state values (`getProperties`), SVG image selection for structure preview (`getPreview`), caption derivation from datasource (`getCustomCaption`), and slot validation (`check`).

**3. What part of behavior can be documented from this file?**
- `playground` slot is hidden unless `showPlaygroundSlot=true`.
- Advanced options (customLayout, customConfigurations, customSeriesOptions, enableThemeConfig) are hidden on web unless `enableAdvancedOptions=true`. On desktop, `enableAdvancedOptions` itself is hidden (advanced options always accessible).
- Scale controls follow a cascade: `scaleColors` visible only when `showScale=true`; `smoothColor` and `showValues` visible only when `showScale AND enableAdvancedOptions`; `valuesColor` visible only when `showScale AND enableAdvancedOptions AND showValues`.
- Groups are transformed into tabs for web (`transformGroupsIntoTabs`); empty groups are removed; Visibility property is moved from position 7 to position 4.
- Structure preview: shows HeatMap SVG (375px wide) with optional legend SVG (57px wide) when `showScale=true`. Supports dark/light mode asset selection.
- `getCustomCaption` returns the datasource property caption or falls back to "Heatmap chart".
- `checkSlot` is the only validation — no other error conditions checked in editor config.

**4. Is it user-facing?**
No — this is design-time only, visible in Mendix Studio(Pro) property panels.

**5. What new did you learn from this file?**
The `valuesColor` property is deeply nested behind three conditions (showScale + enableAdvancedOptions + showValues), making it effectively an advanced-advanced option. The `removeEmptyGroups` function ensures the tab layout stays clean after hiding. Visibility is repositioned in the tab order, suggesting the final tab order in Studio differs from the XML declaration order.

---

## src/HeatMap.editorPreview.tsx

**1. What is the purpose of this file?**
Provides the live React preview component for Mendix Studio(Pro). Renders a `ChartPreview` component with the widget's SVG assets and configures the legend display.

**2. What kind of logic is described in this file?**
Minimal: passes `showScale` as `showLegend` to `ChartPreview`, uses hardcoded light-theme SVG assets.

**3. What part of behavior can be documented from this file?**
- Preview always uses the light-theme assets (`HeatMap.light.svg`, `HeatMap-legend.light.svg`) regardless of Studio theme mode. (Dark mode asset selection is handled in `editorConfig.ts` `getPreview`, not here.)
- The `ChartPreview.PlotLegend` component is used for the colorbar/legend visual in the editor.
- Note: the `alt` text for the main image is "Bubble chart" — a copy-paste artifact, not a functional issue.

**4. Is it user-facing?**
No — this is the design-time preview in Mendix Studio, not the runtime rendering.

**5. What new did you learn from this file?**
There are two separate preview mechanisms: `HeatMap.editorPreview.tsx` (React-based live preview) and `getPreview` in `HeatMap.editorConfig.ts` (structure preview API returning an image tree). Both exist and serve different Studio contexts.

---

## src/__tests__/HeatMap.spec.tsx

**1. What is the purpose of this file?**
Unit tests for the `HeatMap` container component. Verifies data processing, matrix construction, colorscale logic, sorting, and annotation generation.

**2. What kind of logic is described in this file?**
Jest + React Testing Library tests, mocking `ChartWidget` to inspect props passed to it. Tests cover: rendering type, empty-data safety, default/custom colorscale, x/y deduplication, z-matrix shape, annotation count/content, horizontal sort desc, vertical sort desc.

**3. What part of behavior can be documented from this file?**
- Confirmed: `ChartWidget` receives `seriesOptions.type === "heatmap"` and a single data series.
- `showValues=true` with 0 items does not throw (empty data safe-path confirmed).
- Default colorscale is exactly `[[0, "#17347B"], [0.5, "#0595DB"], [1, "#76CA02"]]`.
- Custom scaleColors with 2 entries `[{valuePercentage:0, colour:"red"}, {valuePercentage:100, colour:"blue"}]` produces `[[0, "red"], [1, "blue"]]`.
- For a 3x4 dataset (3 horizontal × 4 vertical): x has 3 unique values, y has 4, z is a 4×3 matrix.
- `showValues=true` generates 12 annotations (one per cell in 3×4), texts match z value indices.
- Horizontal sort desc reverses column order: `[[2,1,0],[5,4,3],[8,7,6],[11,10,9]]`.
- Vertical sort desc reverses row order: `[[11,10,9],[8,7,6],[5,4,3],[2,1,0]]`.

**4. Is it user-facing?**
No — test file only. Not part of the widget bundle.

**5. What new did you learn from this file?**
The test fixture uses a 12-item dataset (3 horizontal × 4 vertical), confirming that axis values are deduced purely from the attribute values of each item — not from separate axis datasources. The z-matrix orientation `[yRow][xCol]` is confirmed. Sort attributes produce independent sort passes, not a composite sort.

---

## e2e/HeatMapChart.spec.js

**1. What is the purpose of this file?**
End-to-end Playwright tests for the heatmap widget in a running Mendix application. Tests visual rendering with screenshot baselines for custom color and sort order configurations.

**2. What kind of logic is described in this file?**
Playwright test scenarios: navigates to the page, locates named containers by CSS class (`mx-name-containerCustomColor`, `mx-name-containerAscending`, `mx-name-containerDescending`), waits for chart and colorbar to be visible, then takes screenshot comparisons (threshold 0.5).

**3. What part of behavior can be documented from this file?**
- Three scenarios e2e-confirmed: custom color rendering, ascending sort order, descending sort order.
- The colorbar (`.colorbar` SVG element) is confirmed visible in all tested configurations — meaning `showScale=true` is required in these test configurations.
- Chart renders as `.mx-react-plotly-chart`.
- Session cleanup is performed after each test (Mendix session logout) due to 5-session license limit.
- No `test.skip(MODERN_CLIENT)` guard — the widget is supported in the Mendix React client.
- Screenshot threshold of 0.5 allows moderate pixel variance (accommodates anti-aliasing differences).

**4. Is it user-facing?**
No — test infrastructure only. Confirms the widget's visual behavior in a real Mendix runtime.

**5. What new did you learn from this file?**
The e2e tests only cover visual snapshot assertions — no interaction testing (click, hover tooltip) is present in e2e. Sorting and color customization are the primary e2e-confirmed behaviors. The widget container is identified by Mendix class `mx-name-{containerName}`, which is the standard Mendix naming convention.

---

## CHANGELOG.md

**1. What is the purpose of this file?**
Tracks version history of behavioral and technical changes to the heatmap-chart-web widget.

**2. What kind of logic is described in this file?**
Version release notes following Keep a Changelog format and Semantic Versioning.

**3. What part of behavior can be documented from this file?**
- **v6.2.1 (2025-07-15):** Fixed on-click events — datasource was not correctly added, and selection listening was broken. This is a confirmed behavioral fix for the `onClickAction` and `seriesItemSelection` features.
- **v6.0.0 (2025-02-28):** Upgraded plotly.js to version 3.0. This is a major dependency version bump (breaking semver).
- **v5.1.0 (2024-10-28):** Changed bundling to make plotly scannable by package scanners (infrastructure change, no behavioral impact).
- **v5.0.1 (2024-10-15):** Fixed auto-resize inside popup dialogs — behavioral placement constraint: the widget now correctly responds to size changes when placed inside a Mendix popup dialog.
- **v3.1.1 (2023-09-27):** Removed redundant code to improve browser load time (optimization, no behavioral change).
- **v3.1.0 (2023-06-06):** Updated page explorer caption to display datasource; updated light/dark icons.

**4. Is it user-facing?**
Indirectly — changelog is for developers/operators. The fixes described have direct user-visible behavioral impact (click, resize).

**5. What new did you learn from this file?**
The v5.0.1 fix for popup dialog resize means the widget had a known layout bug in popup contexts prior to that version. The v6.2.1 click/selection fix means the datasource binding for events was added relatively recently. The plotly 3.0 upgrade (v6.0.0) aligns with the `plotly.js-dist-min: "^3.0.0"` dependency in package.json.

---

## package.json

**1. What is the purpose of this file?**
NPM package manifest. Declares the widget's name, version, description, scripts, and dependency tree.

**2. What kind of logic is described in this file?**
Metadata and dependency declarations only. Build/dev/test/release scripts using Mendix tooling.

**3. What part of behavior can be documented from this file?**
- Current version: **6.3.0**.
- `plotly.js-dist-min: "^3.0.0"` — major version constraint; Plotly 3.x is required (aligns with v6.0.0 changelog upgrade).
- `@mendix/shared-charts: "workspace:*"` — uses the monorepo's shared chart library (not a versioned external package).
- `date-fns: "^2.30.0"` is a dependency (likely used in shared-charts for date formatting).
- `classnames: "^2.5.1"` for class composition.
- Minimum Mendix version: 9.6.0 (from `marketplace.minimumMXVersion`).
- Widget is `"private": true` — not published to npm directly.
- MPK package: `com.mendix.widget.web.HeatMap.mpk`.

**4. Is it user-facing?**
No — build-time configuration only.

**5. What new did you learn from this file?**
The minimum supported Mendix version is 9.6.0. The widget ships as an MPK file with package path `com.mendix.widget.web`. The test project branch is `heatmap-chart-web` in the Mendix testProjects repo.
