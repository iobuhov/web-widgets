# Draft: column-chart-web

Extracted from `packages/pluggableWidgets/column-chart-web/` on 2026-05-08.

---

## src/ColumnChart.xml

**1. Purpose:** Declares widget metadata and all configurable properties. Authoritative source for props, types, defaults, and enumerations.

**2. Logic described:** The widget has three property tabs: General (series list, advancedOptions, showPlaygroundSlot, playground, xAxisLabel, yAxisLabel, showLegend, gridLines, barmode, Visibility, Name, TabIndex), Dimensions (widthUnit/width/heightUnit/height), and Advanced (enableThemeConfig, customLayout, customConfigurations). The `series` is a list of objects, each with: dataSet enum (static/dynamic), data source, groupByAttribute (dynamic only), staticName/dynamicName, X/Y attributes, aggregationType (10 options), tooltipHoverText, barColor expression, onClickAction, customSeriesOptions. `groupByAttribute` accepts String, Boolean, DateTime, Decimal, Enum, HashString, Integer, Long.

**3. Documentable behavior:** Default dimensions: 100% width, 75% height (percentageOfWidth). Default barmode: "group" (grouped). Default showLegend: true. Default gridLines: none. `advancedOptions=false` by default — hides customLayout/customConfigurations/enableThemeConfig/customSeriesOptions. Series ordering matters: "The order influences how columns overlay one another: the first column is drawn lowest and other columns are drawn on top of it." Both X and Y attributes accept: String, DateTime, Decimal, Integer, Long, AutoNumber.

**4. User-facing:** Yes. Properties panel in Studio Pro.

**5. New learnings:** The widget is not `needsEntityContext` — it does not require a surrounding data view. It does NOT have `offlineCapable` attribute set, meaning offline use is not supported. The `aggregationType` has 10 options including statistical modes (median, mode, first, last). The `barColor` is a `ListExpressionValue<string>` (expression per data item, not a fixed color), enabling per-item color variations. The `playground` slot (via `showPlaygroundSlot`) is a charts-specific debug feature.

---

## typings/ColumnChartProps.d.ts

**1. Purpose:** Auto-generated TypeScript types from ColumnChart.xml.

**2. Logic described:** `SeriesType` has dual properties for static/dynamic modes. `groupByAttribute` is `ListAttributeValue<string | boolean | Date | Big>`. X/Y attributes are `ListAttributeValue<string | Date | Big>`. `staticBarColor`/`dynamicBarColor` are `ListExpressionValue<string>`. `staticOnClickAction`/`dynamicOnClickAction` are `ListActionValue` (per-item actions). `aggregationType` has 10 values. `BarmodeEnum` is "group" | "stack". `WidthUnitEnum` is "percentage" | "pixels". `HeightUnitEnum` has a third option: "percentageOfParent".

**3. Documentable behavior:** `ListActionValue` for click actions means each data point can trigger its own action (unlike carousel's `ActionValue`). `ListExpressionValue<string>` for bar color means color is computed per data item — the expression has access to each data source item's attributes. `Big` from `big.js` is used for Decimal values, preserving precision.

**4. User-facing:** No. Internal TypeScript contract.

**5. New learnings:** `heightUnit` has a third option "percentageOfParent" not present in barChart (based on prior drafts). The distinction between `ListAttributeValue` (reads attribute) and `ListExpressionValue` (evaluates expression) is important: bar color can be a complex expression like `if ValueOfAttribute > 100 then 'red' else 'green'`.

---

## src/ColumnChart.tsx

**1. Purpose:** Main container component. Builds chart configuration from Mendix props and renders the shared `ChartWidget`.

**2. Logic described:** Uses `usePlotChartDataSeries` from `@mendix/shared-charts/main` to transform Mendix data series to Plotly format. The callback extracts `staticBarColor` or `dynamicBarColor` expression value and applies it to `marker.color`. `columnChartLayoutOptions` configures axes: xaxis has `fixedrange: true`, `zeroline: false`; yaxis has `fixedrange: true`, `zeroline: true`, `rangemode: "tozero"` (always starts at 0). `columnChartSeriesOptions` sets Plotly type `"bar"` with `orientation: "v"` (vertical). Layout merges `barmode` from props. Component is wrapped in `memo` with `containerPropsEqual` for performance.

**3. Documentable behavior:** The Y axis always starts at zero (`rangemode: "tozero"`). Both axes are not user-zoomable (`fixedrange: true`). Column color is applied at the `marker.color` level per Plotly's API. When `barColor` expression is undefined, `marker.color` is left undefined (Plotly uses its default color cycle). The chart uses `@mendix/shared-charts` as a shared layer (also used by bar-chart, area-chart, etc.).

**4. User-facing:** No. Internal orchestration. Users see the rendered chart.

**5. New learnings:** `orientation: "v"` (vertical) is what makes this a "column" chart vs a "bar" chart (which uses `orientation: "h"`). Both use Plotly's `type: "bar"` — the orientation flag is the only difference. `usePlotChartDataSeries` handles aggregation, grouping, and data transformation from Mendix's ListValue to Plotly trace format.

---

## src/ColumnChart.editorConfig.ts

**1. Purpose:** Controls Studio Pro property visibility, structure preview rendering, property validation (check function), and caption display.

**2. Logic described:** `getProperties`: hides playground slot if `showPlaygroundSlot=false`; hides static props when `dataSet=dynamic` and vice versa; hides `customSeriesOptions` and chart-level advanced props when `advancedOptions=false`; transforms groups to tabs on web. `getPreview`: selects between grouped/stacked SVG assets based on `barmode`, with light/dark mode variants; shows legend image when `showLegend=true`. `check`: validates that X and Y attributes are set when a data source is configured (produces Problems). `getCustomCaption`: returns the first series datasource caption (or "Column chart" if no series).

**3. Documentable behavior:** When `dataSet=static`, the dynamic properties (dynamicDataSource, groupByAttribute, etc.) are completely hidden. This is critical — switching between static/dynamic modes completely changes the available configuration. X and Y attribute validation runs in Studio Pro, preventing publishing without them configured. The chart preview shows different SVG images for grouped vs stacked layout, matching the actual rendered output.

**4. User-facing:** No. Studio Pro editor only.

**5. New learnings:** The `check` function generates Studio Pro validation errors (shown in the error list) for missing X/Y attributes. These are soft constraints that prevent app publishing. `checkSlot` from `@mendix/shared-charts/preview` handles playground slot validation (ensuring playground widgets are not added unless the slot is enabled).

---

## src/ColumnChart.editorPreview.tsx

**1. Purpose:** Live preview in Studio Pro canvas using `ChartPreview` from shared-charts.

**2. Logic described:** Selects between `ColumnChartGrouped` and `ColumnChartStacked` SVG based on `barmode`. Renders via `ChartPreview` (from shared-charts) with `PlotImage` and `PlotLegend` sub-components.

**3. Documentable behavior:** The preview always uses light-mode SVG assets. Preview switches dynamically between grouped and stacked visuals when the user changes `barmode` in Studio Pro. The legend image is always shown in preview (regardless of `showLegend` setting).

**4. User-facing:** Studio Pro canvas only.

**5. New learnings:** `ChartPreview` is shared infrastructure from `@mendix/shared-charts/preview` — all chart widgets use it for consistent preview rendering. This means the column chart preview is visually consistent with bar, area, and other charts.

---

## src/__tests__/ColumnChart.spec.tsx

**1. Purpose:** Unit tests verifying the ColumnChart component passes correct options to ChartWidget.

**2. Logic described:** Mocks `ChartWidget` from `@mendix/shared-charts/main` and `react-plotly.js`. Tests: (1) series options have `type: "bar"` and `orientation: "v"`; (2) bar color is applied to `marker.color` per series (undefined when not set); (3) aggregation type is passed through correctly (both "none" and "avg" produce x/y arrays); (4) barmode is passed to layoutOptions.

**3. Documentable behavior:** Confirms the Plotly type is "bar" with vertical orientation for columns. Confirms that when `staticBarColor` is set to a list expression returning "red", `marker.color === "red"`. When `staticBarColor` is undefined, `marker.color` is `undefined` (not a fallback color — Plotly handles defaults). Barmode "stack" is passed as `layoutOptions.barmode`.

**4. User-facing:** No. Test infrastructure.

**5. New learnings:** The test uses `setupBasicSeries` from `@mendix/shared-charts/main` as a helper — this is shared test infrastructure. The test confirms that the `aggregationType` flow doesn't crash and produces x/y arrays (but doesn't test the specific aggregation math, which lives in shared-charts).

---

## e2e/ColumnChart.spec.js

**1. Purpose:** End-to-end Playwright screenshot-based tests verifying visual rendering of column charts.

**2. Logic described:** Four screenshot tests: (1) default color chart, (2) custom color chart, (3) grouped format chart, (4) stacked format chart. Each navigates to `/` and clicks an action button before testing. Each compares against a baseline PNG screenshot using Playwright's `toHaveScreenshot`. A 1-second wait is added before screenshots to ensure full render.

**3. Documentable behavior:** Screenshot tests confirm visual output for all four combinations of color/format. The `wait 1 second` before screenshots reveals that chart rendering (Plotly) may not be synchronous and needs a settle time. No `test.skip(MODERN_CLIENT)` guard — column chart works in both Mendix clients. Custom color expressions visibly affect bar color in the rendered chart.

**4. User-facing:** No. Test infrastructure. Visual output is user-facing.

**5. New learnings:** The action button click before each test (`.mx-name-actionButton1`) suggests the test page requires a user action to populate data — likely loading data into the Mendix context. The 1-second wait for Plotly rendering is a known limitation documented in the test comment.

---

## CHANGELOG.md

**1. Purpose:** Release history from v3.1.0 to v6.2.1 (latest as of 2025-07-15).

**2. Logic described:** v6.2.1: updated shared charts dependency. v6.2.0: fixed aggregate on Plotly 3.0. v6.0.0: updated Plotly.js to v3.0. v5.1.0: changed bundling for package scanner compatibility. v5.0.1: fixed resize inside popup dialog. v3.1.3: fixed entity attribute access in column color expression editor. v3.1.2: removed redundant code. v3.1.0: updated page explorer caption to display datasource; updated icons.

**3. Documentable behavior:** Plotly v3.0 was a breaking change (v6.0.0 adoption) requiring an aggregate fix in v6.2.0. The popup dialog resize fix (v5.0.1) is an important behavioral note: the chart now auto-resizes when inside a modal dialog. The column color expression editor bug (v3.1.3) confirms that entity attributes must be accessible from the `staticBarColor` expression — this is a documented behavioral requirement.

**4. User-facing:** No. Developer documentation.

**5. New learnings:** The widget went from v3.x to v5.x to v6.x with version jumps, suggesting shared version bumps across the charts widget family. The bundling change in v5.1.0 for "package scanners" suggests the Plotly library was being bundled in a way that prevented security scanning tools from auditing its dependencies.
