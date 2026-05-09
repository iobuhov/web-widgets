# Draft: pie-doughnut-chart-web

Widget package: `packages/pluggableWidgets/pie-doughnut-chart-web`

---

## src/PieChart.tsx

**1. What is the purpose of this file?**
Root React component for the Pie/Doughnut chart widget. It wires Mendix props to the shared `ChartWidget` component from `@mendix/shared-charts`, passing Plotly configuration options and resolved chart data.

**2. What kind of logic is described in this file?**
- Calls `usePieChartDataSeries` hook with all data-related props to get the Plotly-compatible data array.
- Determines whether the chart is "clickable" (`isPieClickable`) — true if either `onClickAction` is configured or `seriesItemSelection.type === "Single"`.
- Passes fixed chart options: `type: "pie"`, `hoverinfo: "none"`, `sort: false` (maintains insertion order, overridden by explicit sort props), `responsive: true`, no grid lines (`gridLinesMode: "none"`), font color white (#FFF) size 12 for data labels.
- The `xAxisLabel` and `yAxisLabel` are always `undefined` — pie charts have no axes.

**3. What part of behavior can be documented from this file?**
- The chart is a Plotly `"pie"` type trace — the same component renders both pie (holeRadius=0) and doughnut (holeRadius>0) formats.
- When clickable (`isPieClickable`), the CSS class `widget-pie-chart-selectable` is added, which sets cursor to pointer on pie slices.
- Pie chart hover info is set to `"none"` at the series level — hover behavior is controlled separately via `hoverinfo` in the data hook (can be `"text"` if `tooltipHoverText` is configured).
- Grid lines are always disabled (`gridLinesMode: "none"`) — pie charts have no axes or grid.
- Legend font: Open Sans, 14px, color #555.
- The `playground` slot allows injecting arbitrary widgets for developer customization (used in Mendix Charts Playground).

**4. Is it user-facing?**
Yes — this produces the visible pie/doughnut chart.

**5. What new did you learn from this file?**
The same `PieChart` component covers both pie and doughnut chart formats — the distinction is entirely driven by the `holeRadius` prop (0 = pie, >0 = doughnut). No separate widget exists for each format; they share all behavior.

---

## typings/PieChartProps.d.ts

**1. What is the purpose of this file?**
Auto-generated TypeScript typings from `PieChart.xml`. Defines `PieChartContainerProps` (runtime) and `PieChartPreviewProps` (design-mode preview).

**2. What kind of logic is described in this file?**
No runtime logic. Key props: `seriesDataSource: ListValue` (required), `seriesName: ListExpressionValue<string>`, `seriesValueAttribute: ListAttributeValue<Big>` (numeric, Decimal/Integer/Long), `seriesColorAttribute?: ListExpressionValue<string>`, `seriesItemSelection?: SelectionSingleValue`, `holeRadius: number` (0–100), `onClickAction?: ListActionValue`, `customLayout/customConfigurations/customSeriesOptions: string` (JSON for Plotly overrides).

**3. What part of behavior can be documented from this file?**
- `seriesValueAttribute` is typed as `ListAttributeValue<Big>` — only Decimal, Integer, and Long attribute types are supported (enforced in XML schema).
- `seriesItemSelection` is `SelectionSingleValue` — only single-item selection mode is supported (not multi-select).
- `holeRadius` is a plain `number` (integer in XML, 0–100 percent) — no union type or enum; validation of range must be done by the user.
- Both `customLayout` and `customConfigurations` accept multiline strings (raw JSON passed to Plotly).
- `className` in preview props is deprecated since Mendix 9.18.0; use `class` instead.

**4. Is it user-facing?**
No — internal type declarations.

**5. What new did you learn from this file?**
`seriesItemSelection` is typed as `SelectionSingleValue` (from Mendix framework), which provides a `setSelection(item)` method. This enables the chart to participate in Mendix's selection framework — selecting a slice can update context selections used by other widgets on the page.

---

## src/PieChart.xml

**1. What is the purpose of this file?**
Widget descriptor XML defining all props, their types, captions, defaults. Generates `PieChartProps.d.ts`.

**2. What kind of logic is described in this file?**
Declares: required `seriesDataSource`, `seriesName` (text template), `seriesValueAttribute` (Decimal/Integer/Long), optional `seriesSortAttribute` (String/Boolean/DateTime/Decimal/Enum/HashString/Integer/Long), `seriesSortOrder` (asc default), optional `seriesColorAttribute` (expression → String), `seriesItemSelection` (None/Single), `holeRadius` (integer, default 0), `showLegend` (boolean, default true), `tooltipHoverText` (optional text template), `onClickAction` (optional list action), dimension props, and advanced JSON overrides.

**3. What part of behavior can be documented from this file?**
- Widget name in Studio is "Pie chart"; description is "Renders a pie or doughnut chart" — confirming the dual-mode nature.
- `seriesSortAttribute` supports a wide range of attribute types for sorting: String, Boolean, DateTime, Decimal, Enum, HashString, Integer, Long.
- `holeRadius` default is 0 (pie mode). Doughnut mode requires setting this to a positive integer percentage.
- `showLegend` defaults to `true`.
- The widget is marked `offlineCapable="true"`.
- `enableAdvancedOptions` (boolean, default false) gates access to `customLayout`, `customConfigurations`, `customSeriesOptions`, and `enableThemeConfig` in Studio (web).
- `showPlaygroundSlot` controls visibility of the `playground` widget slot.
- Categorized under "Charts" in both Studio and Studio Pro.

**4. Is it user-facing?**
Defines the developer-facing configuration interface.

**5. What new did you learn from this file?**
`seriesSortAttribute` accepts Enum and HashString attribute types in addition to numeric types — the sort comparison (`compareAttrValuesAsc`) must handle all these types generically. This is a broader sort capability than many widgets provide.

---

## src/hooks/data.ts

**1. What is the purpose of this file?**
The `usePieChartDataSeries` hook converts Mendix data source items into a Plotly-compatible data array for the `ChartWidget`. It handles per-item data extraction (names, values, colors, hover text), optional sorting, and click/selection event wiring.

**2. What kind of logic is described in this file?**
- Reads data source items when `status === ValueStatus.Available`; stores in `LocalPieChartData[]` state.
- Per item: extracts `itemName` (from `seriesName` expression), `itemValue` (from `seriesValueAttribute`, converted to number), `itemColor` (from `seriesColorAttribute` expression), `itemSortValue` (from `seriesSortAttribute`), `itemHoverText` (from `tooltipHoverText`).
- If `seriesSortAttribute` is provided, sorts items ascending by default (via `compareAttrValuesAsc`); reverses for descending.
- `hoverinfo` is set to `"text"` if any item has non-empty `itemHoverText`; otherwise `"none"`.
- `onClick` callback: fires `onClickAction` for the clicked item AND calls `seriesItemSelection.setSelection(item)` if selection is configured. Both can coexist.
- Sorting is applied twice: once to the `LocalPieChartData[]` array (for labels/values/colors), and once to the raw `ObjectItem[]` (for `dataSourceItems` used by the shared chart for click handling).
- `holeRadius` is divided by 100 before being passed to Plotly (e.g., 40 → 0.4).

**3. What part of behavior can be documented from this file?**
- When `seriesSortAttribute` is not configured, the chart renders slices in data source insertion order. `sort: false` in `seriesOptions` (from PieChart.tsx) prevents Plotly from auto-sorting by value.
- `seriesValueAttribute` values are converted to JavaScript numbers via `.toNumber()` — undefined values default to 0.
- `seriesName` values default to `null` (not empty string) when undefined — Plotly will omit these from the legend.
- All items in a single series share the same `hoverinfo` mode — if any item has hover text, the mode switches to `"text"` for all items.
- Clicking a slice can simultaneously fire an action AND update a Mendix selection — these are independent behaviors controlled by separate props.

**4. Is it user-facing?**
Indirectly — drives the data visualized on the chart.

**5. What new did you learn from this file?**
The `dataSourceItems` in the returned Plotly data array must be sorted in the same order as the visual data (labels/values/colors) so that click events correctly map the clicked Plotly slice index to the right Mendix `ObjectItem`. Both arrays go through the same sort algorithm.

---

## src/PieChart.editorConfig.ts

**1. What is the purpose of this file?**
Provides `getProperties` (prop visibility rules), `getPreview` (structure preview image), `check` (validation), and `getCustomCaption` (Studio page explorer label) for the Studio/Studio Pro editor.

**2. What kind of logic is described in this file?**
- `getProperties`: On web, hides `customLayout`, `customConfigurations`, `customSeriesOptions`, `enableThemeConfig` unless `enableAdvancedOptions` is true. On desktop (Studio Pro), hides `enableAdvancedOptions` (all options always visible). Calls `transformGroupsIntoTabs` for web layout. Hides `playground` prop if `showPlaygroundSlot` is false.
- `getPreview`: Selects one of four SVG images based on `holeRadius > 0` (pie vs doughnut) and dark/light mode. Shows legend image alongside chart if `showLegend` is true. Wraps in `withPlaygroundSlot` from shared charts.
- `check`: Only calls `checkSlot(values)` — validates playground slot configuration. No other validation.
- `getCustomCaption`: Shows data source caption (e.g., entity name) or "Pie chart" as the widget label in Studio page explorer.

**3. What part of behavior can be documented from this file?**
- The structure preview distinguishes between pie and doughnut chart appearance based on `holeRadius > 0` — a non-zero hole radius shows the doughnut SVG preview.
- Advanced options (`customLayout`, `customSeriesOptions`, `customConfigurations`, `enableThemeConfig`) are hidden from Studio users by default; only unlocked by "Enable advanced options" toggle.
- In Studio Pro, all options are always visible (no gating).
- The `withPlaygroundSlot` wrapper adds a playground widget slot area to the structure preview when configured.

**4. Is it user-facing?**
Yes — visible to developers configuring the widget in Studio/Studio Pro.

**5. What new did you learn from this file?**
The `holeRadius` preview switching happens at `holeRadius > 0` (not `holeRadius !== 0`), so a holeRadius of 0 shows the pie SVG and any positive value shows the doughnut SVG — consistent with the runtime behavior where `hole: holeRadius/100` of 0 renders as a pie.

---

## src/PieChart.editorPreview.tsx

**1. What is the purpose of this file?**
Provides the React-based design-mode preview for Studio canvas. Uses the shared `ChartPreview` component.

**2. What kind of logic is described in this file?**
Selects between `DoughnutChart` and `PieChart` light SVG images based on `holeRadius > 0`. Renders `ChartPreview` with `PlotImage` (the chart SVG) and `PlotLegend` (the legend SVG).

**3. What part of behavior can be documented from this file?**
- Design-mode preview dynamically switches between pie and doughnut SVG based on `holeRadius` — giving developers visual feedback on chart format selection.
- The alt text `"Bubble chart"` is a copy-paste error in the code (the actual chart is pie/doughnut). This does not affect runtime behavior.
- Preview uses light-mode SVGs only (not dark-mode adaptive), regardless of Studio's theme.

**4. Is it user-facing?**
Yes — visible to developers in Studio design canvas.

**5. What new did you learn from this file?**
The design-mode preview uses light SVGs only (imports `PieChart.light.svg` and `DoughnutChart.light.svg`), unlike the `editorConfig.ts` structure preview which properly selects between dark and light SVGs based on `isDarkMode`. The live design preview is always light-themed.

---

## src/ui/PieChart.scss

**1. What is the purpose of this file?**
Provides CSS for the pie chart widget — specifically the cursor style for interactive (selectable/clickable) slices.

**2. What kind of logic is described in this file?**
One rule: `.widget-pie-chart.widget-pie-chart-selectable .pielayer .slice { cursor: pointer; }` — sets pointer cursor on Plotly pie slices when the chart has either `onClickAction` or single-item selection configured.

**3. What part of behavior can be documented from this file?**
- Pointer cursor on pie slices is applied via the `.widget-pie-chart-selectable` class, which is added by the component when `isPieClickable` is true.
- Without this class, the cursor remains the default (not pointer) even if Plotly's internal state might suggest interactivity.
- `.pielayer .slice` is a Plotly-internal CSS class — this SCSS directly targets Plotly's DOM structure.

**4. Is it user-facing?**
Yes — affects the visual cursor feedback for clickable pie slices.

**5. What new did you learn from this file?**
The clickable cursor indicator uses Plotly's internal `.pielayer .slice` DOM class — this couples the widget's styling to Plotly's internal DOM structure and could break if Plotly changes its class naming in a future version.

---

## src/__tests__/PieChart.spec.tsx

**1. What is the purpose of this file?**
Unit tests for the `PieChart` component verifying data wiring to `ChartWidget`. Mocks `ChartWidget` to capture the props passed to it.

**2. What kind of logic is described in this file?**
Five test groups:
1. Confirms `seriesOptions.type === "pie"` — the chart type is always "pie" regardless of hole radius.
2. Confirms `holeRadius: 40` → `data[0].hole === 0.4` (percent to fraction conversion).
3. Confirms `marker.colors` are correctly mapped from `seriesColorAttribute`.
4. Confirms `labels` are mapped from `seriesName` expressions.
5. Confirms `values` are mapped from `seriesValueAttribute`.
6. Sorting tests: ascending order reverses two items (sort value 20 before 15, then ascending puts 15 first → value 2 before value 1). Descending reverses that.

**3. What part of behavior can be documented from this file?**
- `holeRadius` is confirmed to be divided by 100 before passing to Plotly: 40 → 0.4.
- Sorting in ascending order reorders data: items are sorted by `seriesSortAttribute`, and all parallel arrays (values, labels, colors) maintain aligned order after sort.
- In descending sort, the final array is the reverse of ascending — confirmed by the test swapping expected values/labels/colors.
- The `ChartWidget` receives exactly one data series (array length 1) — pie/doughnut charts always have a single Plotly trace.

**4. Is it user-facing?**
No — test file only.

**5. What new did you learn from this file?**
The sort logic is applied to a parallel array of `LocalPieChartData` objects (not to the Plotly arrays directly). After sorting, the values, labels, and colors arrays are derived from the sorted `LocalPieChartData` array in order — so they remain correctly aligned even after sort reversal.

---

## e2e/PieChart.spec.js

**1. What is the purpose of this file?**
End-to-end Playwright tests for the Pie/Doughnut chart widget. Tests pie color rendering and both chart formats (pie and doughnut) via screenshot baselines.

**2. What kind of logic is described in this file?**
Three tests with retry(3) on the color test:
1. Custom slice color — screenshot baseline for `containerSliceColor`.
2. Pie format — screenshot baseline for `containerPieFormat`.
3. Doughnut format — screenshot baseline for `containerDoughnutFormat`.

All tests use `.scrollIntoViewIfNeeded()` before screenshotting and a threshold of 0.5.

**3. What part of behavior can be documented from this file?**
- Custom `seriesColorAttribute` coloring is e2e-confirmed to affect slice colors visually.
- Pie format (holeRadius=0) and doughnut format (holeRadius>0) are separately verified as distinct visual states.
- The color test has `retry: 3` — indicating visual rendering can be slightly flaky (possibly due to animation or rendering timing).
- All screenshot comparisons use `threshold: 0.5` — a relatively high tolerance, accommodating minor rendering differences.

**4. Is it user-facing?**
The tested behaviors (custom colors, pie vs doughnut format) are user-facing.

**5. What new did you learn from this file?**
The e2e tests do not test click interactions or selection — only visual rendering and format switching. Click and selection behavior is only covered at the unit level (`PieChart.spec.tsx`).

---

## CHANGELOG.md

**1. What is the purpose of this file?**
Version history for pie-doughnut-chart-web from v3.1.0 to v6.2.1.

**2. What kind of logic is described in this file?**
No logic — version history entries.

**3. What part of behavior can be documented from this file?**
Key behavioral changes:
- v6.0.0 (2025-02-28): Updated Plotly.js to version 3.0 — a major dependency upgrade.
- v5.2.0 (2025-01-21): Added "listen to widget" functionality for pie chart selection — enabling the chart to receive selection state from other widgets.
- v5.1.1 (2024-12-12): Fixed onClick when multiple points are added to the same item — a specific bug in the click handler.
- v5.0.2 (2024-10-15): Fixed auto-resize inside popup dialogs — a placement/layout constraint fix.
- v5.0.1 (2024-09-19): Fixed onClick not executing properly — the click action was silently failing before this fix.
- v3.1.2 (2023-11-21): Fixed entity attribute access in "slice color" expression editor (broke after v4.0 and restored) — `seriesColorAttribute` expression can reference entity attributes.
- v3.1.0 (2023-06-06): Updated page explorer caption to display datasource.

**4. Is it user-facing?**
The changelog is publicly visible on the Mendix Marketplace.

**5. What new did you learn from this file?**
v3.1.2 documents that the `seriesColorAttribute` expression editor lost access to entity attributes in v4.0 and had it restored in v3.1.2 — this was a regression that affected users configuring color from data source entity attributes. Users upgrading from v3.x to v4.0 would have experienced this breakage.
