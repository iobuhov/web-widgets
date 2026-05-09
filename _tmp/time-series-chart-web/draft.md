# Draft: time-series-chart-web

Extracted by worker on 2026-05-09. Covers all source files and local workspace dependencies.

---

## src/TimeSeries.tsx

**Purpose:** Main widget component. Transforms `lines` configuration into Plotly scatter traces and renders them via the shared `ChartWidget`.

**Logic:** Wrapped in `memo` with a custom equality comparator using `flatEqual`/`traceEqual` (deep comparison that avoids unnecessary re-renders). `usePlotChartDataSeries` converts each configured line into a Plotly data series (`PlotChartSeries`). `createTimeSeriesChartLayoutOptions` builds Plotly layout with `xaxis.type: "date"`, range slider visibility, y-axis rangemode, and static visual defaults (font color, grid color). Per-line trace options set `type: "scatter"`, `hoverinfo: "none"`, mode (`lines` or `lines+markers`), fill, interpolation (`line.shape`), and colors.

**Behavioral constraints from this file:**
- X-axis type is hardcoded as `"date"` — this widget only supports DateTime x-axis values, which matches the XML constraint that `staticXAttribute`/`dynamicXAttribute` only accept DateTime attributes.
- `hoverinfo: "none"` is set globally on series — custom hover text from `tooltipHoverText` is handled by the shared ChartWidget's click/hover mechanism, not by Plotly's built-in hover.
- `yaxis.fixedrange: true` — the y-axis cannot be zoomed or panned (fixed range). Only the x-axis is interactive via the range slider.
- `fill: "tonexty"` when `enableFillArea: true` — fills to the previous trace ("tonexty"), not to the x-axis ("tozeroy"). This means fill area behavior depends on series stacking order.
- `yaxis.rangemode: yAxisRangeMode || "tozero"` — defaults to "tozero" when no rangemode is configured.
- Series order is significant: "the first line (from the top) is drawn lowest and other lines are drawn on top of it" (from XML description).
- `configOptions.responsive: true` — chart resizes with container.

**User-facing:** Yes — renders the visible chart.

**New findings:** The `fill: "tonexty"` means that when multiple series are configured, the fill area is between adjacent series, not between each series and the x-axis. The first series fills to itself (no previous series, so fills to zero). The description warns developers about series order influencing fill area appearance.

---

## src/TimeSeries.xml

**Purpose:** Widget descriptor declaring all configurable properties across General, Dimensions, and Advanced tabs.

**Logic:** `lines` is an object list — each line entry configures: `dataSet` (static/dynamic), data source, x/y attributes (DateTime x; String/Enum/DateTime/Decimal/Integer/Long/AutoNumber y), `groupByAttribute` (for dynamic mode, all attribute types), `aggregationType` (10 options), `staticName`/`dynamicName`, tooltip hover text, interpolation (linear/spline), `lineStyle` (line/lineWithMarkers/custom), colors (lineColor, markerColor, fillColor as textTemplates), `enableFillArea` (boolean, default true), per-series `customSeriesOptions` (JSON string), and `staticOnClickAction`/`dynamicOnClickAction`. Chart-level: `showLegend` (default true), `showRangeSlider` (default true), `gridLines` (none/horizontal/vertical/both, default none), `xAxisLabel`, `yAxisLabel`, `yAxisRangeMode` (tozero/normal/nonnegative, default tozero), `enableAdvancedOptions` (boolean, default false), `showPlaygroundSlot` + `playground` slot, `enableThemeConfig`, `customLayout`/`customConfigurations` (JSON strings). Dimensions: width (default 100 percentage), height (default 75 percentageOfWidth).

**Behavioral constraints from this file:**
- X-attribute must be DateTime — enforced by `attributeType name="DateTime"` constraint.
- Y-attribute accepts String, Enum, DateTime, Decimal, Integer, Long, AutoNumber — broad type support.
- `groupByAttribute` for dynamic mode accepts String, Boolean, DateTime, Decimal, Enum, HashString, Integer, Long — determines how dynamic data is partitioned into separate lines.
- Colors (lineColor, markerColor, fillColor) are textTemplates — can be dynamic expressions, enabling data-driven colors.
- `aggregationType` default is "none" — no aggregation by default; 9 other options available.
- `enableFillArea` default is true — fill is on by default.
- `showRangeSlider` default is true — range slider shown by default.
- `offlineCapable="true"`, `needsEntityContext="true"`, `pluginWidget="true"`.
- Categorized under "Charts" in both Studio Pro and Studio.

**User-facing:** Indirectly — developer configuration.

**New findings:** The `lineStyle: "custom"` option exists but its behavior is not specified in the XML — it would require `customSeriesOptions` JSON to define the Plotly trace properties for the line. Two on-click action properties exist: one for static data source items, one for dynamic — both are `ListActionValue`, meaning the action has access to the clicked data item's context.

---

## src/TimeSeries.editorConfig.ts

**Purpose:** Studio Pro property visibility rules, validation checks, structure preview, and custom caption.

**Logic:** `getProperties` handles per-line property hiding (static vs. dynamic data source properties, markerColor hidden unless lineStyle is "lineWithMarkers", fillColor hidden unless enableFillArea is true, customSeriesOptions hidden unless enableAdvancedOptions on web). Chart-level: advanced properties (customLayout, customConfigurations, enableThemeConfig, yAxisRangeMode) hidden unless enableAdvancedOptions. On web: `transformGroupsIntoTabs`; on desktop: hides `enableAdvancedOptions` toggle. `check` validates X and Y attributes are set when a data source is configured. `getPreview` selects among 6 SVG assets (dark/light × series/range × chart/legend) based on `showRangeSlider` and `isDarkMode`. `withPlaygroundSlot` wraps the chart preview when playground is enabled. `getCustomCaption` returns first series datasource caption + "and N more" for multi-series.

**Behavioral constraints from this file:**
- X and Y attribute validation: Studio Pro shows an error if a data source is configured without axis attributes — prevents misconfigured deployments.
- `lineStyle: "lineWithMarkers"` is the only mode that exposes `markerColor` — for "line" and "custom" modes, markerColor is hidden.
- Preview SVG differs based on `showRangeSlider` — with range slider, a different SVG is shown in the editor.
- `enableAdvancedOptions` gates: customLayout, customConfigurations, enableThemeConfig, yAxisRangeMode (chart-level) and customSeriesOptions (per-series) on the web platform.
- On desktop (Studio), the advanced toggle is hidden and all properties are always visible.

**User-facing:** Studio Pro editor only.

**New findings:** Six distinct SVG preview assets exist: TimeSeries.dark/light.svg (no range slider), TimeSeries-range.dark/light.svg (with range slider), TimeSeries-legend.dark/light.svg (legend panel). The chart preview changes shape in the editor based on the range slider toggle — giving developers a realistic preview of how the chart will look.

---

## src/TimeSeries.editorPreview.tsx

**Purpose:** Live React preview rendered inside Studio Pro page editor.

**Logic:** Uses `ChartPreview` from `@mendix/shared-charts/preview`. Selects between `TimeSeries.light.svg` and `TimeSeries-range.light.svg` based on `showRangeSlider`. Passes the SVG as `ChartPreview.PlotImage` and the legend SVG as `ChartPreview.PlotLegend`. Light-mode SVGs only — no dark mode awareness in the live preview.

**Behavioral constraints from this file:**
- The live preview always uses light-mode SVGs — dark mode is only handled in `getPreview` (structure preview), not in the React `preview` function.
- The chart preview changes appearance based on `showRangeSlider` — the range slider preview SVG is taller (more space for the slider control).

**User-facing:** Studio Pro design mode only.

**New findings:** The `alt` text says "Bubble chart" for the time series image — this appears to be a copy-paste error from another chart widget. The displayed SVG is correct (TimeSeries), but the alt text is incorrect.

---

## typings/TimeSeriesProps.d.ts

**Purpose:** Auto-generated TypeScript types from `TimeSeries.xml`.

**Logic:** Exports: `DataSetEnum`, `AggregationTypeEnum` (10 values), `InterpolationEnum`, `LineStyleEnum`, `GridLinesEnum`, `WidthUnitEnum`, `HeightUnitEnum`, `YAxisRangeModeEnum`. `LinesType` (runtime per-series type): `staticDataSource?: ListValue`, `dynamicDataSource?: ListValue`, `groupByAttribute?: ListAttributeValue<string | boolean | Date | Big>`, `staticXAttribute?: ListAttributeValue<Date>`, `dynamicYAttribute?: ListAttributeValue<string | Date | Big>`, colors as `DynamicValue<string>`, `staticOnClickAction?: ListActionValue`, `dynamicOnClickAction?: ListActionValue`. `TimeSeriesContainerProps`: `lines: LinesType[]`.

**Behavioral constraints from this file:**
- Colors (`lineColor`, `markerColor`, `fillColor`) are `DynamicValue<string>` — they are textTemplates resolved at runtime, not static strings.
- `aggregationType` is required (not optional) in `LinesType` — always has a value (default "none").
- `staticOnClickAction` and `dynamicOnClickAction` are `ListActionValue` — they operate on the list item context.
- `interpolation` and `lineStyle` are required in `LinesType` — always present.
- `customSeriesOptions` is `string` (not optional) — empty string when not configured.

**User-facing:** Internal TypeScript only.

**New findings:** The Y-axis attribute at runtime supports `string | Date | Big` — this is a union that enables non-numeric Y values (String or Enum categories), making the time series chart usable for categorical time data as well as numeric.

---

## e2e/TimeSeriesChart.spec.js

**Purpose:** End-to-end screenshot-based tests for the time series chart widget.

**1. What is the purpose of this file?**
Playwright e2e test suite verifying visual rendering of the chart under different configurations.

**2. What kind of logic is described in this file?**
Tests: multiple series (screenshot), without range slider (screenshot), fill area configurations (without fill area screenshot, custom fill area color screenshot), y-axis range modes (non-negative screenshot, auto screenshot). All tests use screenshot baseline comparison with 50% threshold.

**3. What part of behavior can be documented from this file?**
- Multiple series rendering is e2e-confirmed (screenshot baseline exists).
- Without range slider: confirmed visual variant exists.
- Fill area off: confirmed — chart renders without fill area.
- Custom fill area color: confirmed — a custom color is applied to the fill area.
- Y-axis range "non-negative": confirmed — only positive y-values shown.
- Y-axis range "auto" (normal): confirmed — axis range based on data.

**4. Is it user-facing?**
Internal test file.

**5. What new did you learn from this file?**
All tests are screenshot-only — no interactive or data-assertion tests. This reflects the nature of chart widgets: correctness is primarily visual. No on-click action or tooltip interaction tests are present.

---

## src/__test__/TimeSeries.spec.tsx

**Purpose:** Unit tests for the `TimeSeries` component covering data mapping and layout options.

**1. What is the purpose of this file?**
Jest unit tests that verify how TimeSeries maps props to `ChartWidget` calls.

**2. What kind of logic is described in this file?**
Tests: chart type is "scatter" (confirmed via `seriesOptions.type`), lineStyle mapping (lineWithMarkers → "lines+markers", line → "lines"), interpolation mapping (linear → "linear", spline → "spline"), lineColor (dynamic "red" → `data[0].line.color === "red"`), markerColor (dynamic "blue"), aggregation (none = raw Date values, avg = ISO string dates with averaged Y), fillColor (dynamic "red" → `fillcolor === "red"`), showRangeSlider (→ `layoutOptions.xaxis.rangeslider.visible`), yAxisRangeMode (→ `layoutOptions.yaxis.rangemode`).

**3. What part of behavior can be documented from this file?**
- Chart type is always "scatter" — Plotly scatter with "lines" mode renders as a line chart.
- Aggregation "none": x values are `Date` objects; aggregation "avg": x values are ISO string dates (converted during aggregation).
- `lineWithMarkers` → `mode: "lines+markers"`; `line` → `mode: "lines"`.
- `lineColor: dynamic("red")` → `line.color: "red"` (DynamicValue resolved to string).
- `fillColor: dynamic("red")` → `fillcolor: "red"` in the chart data.
- `showRangeSlider: true` → `layoutOptions.xaxis.rangeslider.visible: true`.
- `yAxisRangeMode: "nonnegative"` → `layoutOptions.yaxis.rangemode: "nonnegative"`.
- Test data uses two dates (2022-01-01, 2022-01-02) with Y values 3 and 6.

**4. Is it user-facing?**
Internal test file.

**5. What new did you learn from this file?**
The aggregation function changes the type of x values: "none" leaves them as `Date` objects; any aggregation converts them to ISO string format. This is a subtle behavioral difference between aggregated and non-aggregated time series.

---

## CHANGELOG.md

**1. What versions are documented?**
Seven versions: v3.1.0, v3.1.2, v5.0.1, v5.1.0, v6.0.0, v6.2.0, v6.2.1.

**2. What are the most significant behavioral changes?**
- v5.0.1 (2024-10-15): Fixed auto-resize inside popup dialogs — behavioral placement constraint.
- v6.0.0 (2025-02-28): Upgraded Plotly.js to version 3.0 — major dependency upgrade.
- v6.2.0 (2025-06-03): Fixed aggregation broken by Plotly 3.0 upgrade — aggregation feature was broken and then fixed.

**3. What dependency or integration facts are relevant?**
- Plotly.js 3.0 was adopted in v6.0.0 (2025-02-28).
- `@mendix/shared-charts` is a peer dependency updated in v6.2.1.
- Minimum Mendix version: 9.24.0 (from package.json).

**4. Were any behavioral constraints added or removed?**
- v5.0.1: Auto-resize in popup dialogs was fixed — widget now correctly resizes when placed inside a popup dialog.
- v6.2.0: Aggregation was broken in v6.0.0 (Plotly 3.0 migration) and fixed in v6.2.0 — the aggregation feature returned to working state.
- v5.1.0: Bundling changed for package scanner compatibility — no behavioral change.

**5. What is the latest stable version and when was it released?**
v6.2.1, released 2025-07-15, updating the shared charts dependency.

---

## Summary of Key Findings

- **Purpose**: A Plotly.js-based time series (line) chart that visualizes DateTime x-axis data against numeric or categorical y-axis values.
- **External dependency**: `@mendix/shared-charts` wraps Plotly.js; TimeSeries itself adds only the x-axis "date" type hardcoding and fill area behavior.
- **Data modes**: Static (single series per data source) and dynamic (multiple series via group-by attribute).
- **Aggregation**: 10 aggregation functions; "none" preserves raw Date objects while aggregated data uses ISO string dates.
- **Fill area**: `fill: "tonexty"` — fills to the previous series, not the x-axis. Series order matters for visual stacking.
- **Y-axis**: Fixed range (non-zoomable). Three range modes: tozero (default), auto, non-negative.
- **Range slider**: X-axis range slider enabled by default, can be disabled.
- **Colors**: Line, marker, and fill colors are textTemplates (DynamicValue<string>) — data-driven color support.
- **On-click actions**: Both static and dynamic data sources support per-item click actions (ListActionValue).
- **Custom options**: Per-series `customSeriesOptions` and chart-level `customLayout`/`customConfigurations` for advanced Plotly customization (gated behind `enableAdvancedOptions`).
- **Theme config**: Optional theme folder configuration for brand-consistent chart styling.
- **Popup resize**: Fixed in v5.0.1 — widget auto-resizes in popup dialogs.
- **Plotly 3.0**: Aggregation was broken by Plotly 3.0 upgrade (v6.0.0) and fixed in v6.2.0.
- **Preview**: Studio Pro preview shows different SVGs for range slider on/off; live preview has incorrect alt text ("Bubble chart").
