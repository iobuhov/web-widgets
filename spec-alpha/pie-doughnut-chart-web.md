# Pie / Doughnut Chart

## Purpose

The Pie/Doughnut Chart widget renders a proportional chart from a Mendix data source using Plotly.js. A single widget covers both pie format (solid disc) and doughnut format (ring with a hole) — the distinction is controlled by a single `holeRadius` prop. Each slice represents one data source item, with configurable names, numeric values, colors, sorting, optional hover text, and Mendix selection/action integration.

## User Scenarios

### [P1] Render a pie chart from a data source
**Given** a page with a Pie/Doughnut Chart widget bound to a data source entity, with `seriesName` and `seriesValueAttribute` configured  
**When** the data source is available  
**Then** the chart renders one slice per item, sized proportionally to its value, labeled with the configured name, in insertion order by default

#### Edge Cases
- While the data source is loading, no chart content is rendered.
- Items with undefined `seriesValueAttribute` default to value `0` and occupy no visible slice area but are included in the trace.
- Items with undefined `seriesName` default to `null`; Plotly omits these from the legend.

### [P2] Switch between pie and doughnut format
**Given** a widget with `holeRadius` set to a positive integer (1–100)  
**When** the chart renders  
**Then** the chart displays as a doughnut with a hole proportional to the `holeRadius` percentage (e.g., `holeRadius=40` → `hole=0.4` passed to Plotly)

#### Edge Cases
- `holeRadius=0` renders a solid pie (default). Any positive value renders a doughnut.
- The Studio structure preview dynamically switches between pie and doughnut SVG images based on `holeRadius > 0`.
- The design-mode canvas preview always uses light-mode SVGs regardless of Studio theme.

### [P3] Sort slices by an attribute
**Given** `seriesSortAttribute` is configured  
**When** the chart renders  
**Then** slices are ordered by the sort attribute in the configured direction (ascending or descending). Labels, values, and colors arrays are all sorted in the same order.

#### Edge Cases
- Without `seriesSortAttribute`, slices appear in data source insertion order. Plotly's internal value-based sorting is disabled (`sort: false`).
- Sort attribute types supported: String, Boolean, DateTime, Decimal, Enum, HashString, Integer, Long.
- After descending sort, the order is exactly the reverse of ascending order.

### [P4] Click a slice to trigger an action or selection
**Given** `onClickAction` and/or `seriesItemSelection` (Single) are configured  
**When** the user clicks a slice  
**Then** the configured Mendix action fires for the clicked item AND/OR the Mendix selection is updated to the clicked item

#### Edge Cases
- The cursor changes to pointer on all slices when `isPieClickable` is true (either `onClickAction` or `seriesItemSelection.type === "Single"` is set).
- Both action and selection can coexist and are independent — clicking one slice fires the action and updates the selection simultaneously.
- Only single-item selection mode is supported (`SelectionSingleValue`); multi-select is not supported.

### [P5] Display hover tooltips
**Given** `tooltipHoverText` is configured with a text template  
**When** the user hovers over a slice  
**Then** the hover tooltip shows the configured text for that item

#### Edge Cases
- `hoverinfo` mode switches to `"text"` only when at least one item has non-empty hover text; otherwise it is `"none"` (no hover info for any slice).
- All items in the chart share the same `hoverinfo` mode — partial hover text is not possible per item.

## Functional Requirements

- FR-001: The chart MUST use Plotly `"pie"` trace type for both pie and doughnut formats.
- FR-002: `holeRadius` (0–100 integer) MUST be divided by 100 before being passed to Plotly as the `hole` parameter (e.g., 40 → 0.4).
- FR-003: Plotly's automatic value-based slice sorting MUST be disabled (`sort: false`); insertion order MUST be preserved unless `seriesSortAttribute` is configured.
- FR-004: `seriesValueAttribute` MUST be a Mendix `Decimal`, `Integer`, or `Long` attribute type.
- FR-005: `seriesItemSelection` MUST be `SelectionSingleValue` only; multi-select is not supported.
- FR-006: When `onClickAction` is configured or `seriesItemSelection` is set, the CSS class `widget-pie-chart-selectable` MUST be added to the widget root, triggering pointer cursor on slices.
- FR-007: The `dataSourceItems` array passed to `ChartWidget` MUST be sorted in the same order as the parallel data arrays (labels, values, colors) to ensure click events map to the correct `ObjectItem`.
- FR-008: Grid lines MUST be permanently disabled (`gridLinesMode: "none"`); axes are never shown for pie/doughnut charts.
- FR-009: The widget MUST be offline capable.
- FR-010: Advanced options (`customLayout`, `customSeriesOptions`, `customConfigurations`, `enableThemeConfig`) MUST be hidden in Studio unless `enableAdvancedOptions` is true.

## Props Reference

| Name | Type | Default | Caption | Description |
|------|------|---------|---------|-------------|
| `seriesDataSource` | `ListValue` | — | Data source | Required data source providing the list of items to visualize. |
| `seriesName` | `ListExpressionValue<string>` | — | Slice name | Expression evaluated per item; sets the label for each slice. Null when undefined. |
| `seriesValueAttribute` | `ListAttributeValue<Big>` | — | Value attribute | Numeric attribute (Decimal/Integer/Long) determining slice size. Undefined values default to 0. |
| `seriesColorAttribute` | `ListExpressionValue<string>?` | — | Slice color | Optional expression returning a color string per item (e.g., `"#FF5733"`). |
| `seriesSortAttribute` | `ListAttributeValue<any>?` | — | Sort attribute | Optional attribute for ordering slices. Supports String, Boolean, DateTime, Decimal, Enum, HashString, Integer, Long. |
| `seriesSortOrder` | `"asc" \| "desc"` | `"asc"` | Sort order | Direction of sort when `seriesSortAttribute` is set. |
| `seriesItemSelection` | `SelectionSingleValue?` | — | Selection | Enables Mendix single-item selection when a slice is clicked. |
| `holeRadius` | `number` | `0` | Hole radius (%) | Integer 0–100. `0` = pie, `>0` = doughnut. Value is divided by 100 before passing to Plotly. |
| `showLegend` | `boolean` | `true` | Show legend | Displays the Plotly legend alongside the chart. |
| `tooltipHoverText` | `ListExpressionValue<string>?` | — | Tooltip hover text | Optional per-item hover tooltip text. Activates `hoverinfo: "text"` for all items when any item has text. |
| `onClickAction` | `ListActionValue?` | — | On click | Mendix action executed when a slice is clicked. |
| `customLayout` | `string` | — | Custom layout | Raw Plotly layout JSON override. Requires `enableAdvancedOptions`. |
| `customSeriesOptions` | `string` | — | Custom series options | Raw Plotly series-level JSON override. Requires `enableAdvancedOptions`. |
| `customConfigurations` | `string` | — | Custom configurations | Raw Plotly config JSON override. Requires `enableAdvancedOptions`. |

## Changelog

**v6.0.0 (2025-02-28):** Updated Plotly.js to version 3.0 (major dependency upgrade).

**v5.2.0 (2025-01-21):** Added "listen to widget" functionality enabling the chart to receive selection state from other widgets.

**v5.1.1 (2024-12-12):** Fixed onClick when multiple points are added to the same item.

## Open Questions
> Could not be determined from source code alone — requires human review
- [ ] Is the 0–100 integer range for `holeRadius` enforced by Studio/Studio Pro validation, or is out-of-range input possible at runtime?
- [ ] What Plotly.js version is bundled in v6.0.0+ and what breaking changes may affect `customLayout` / `customSeriesOptions` users upgrading from v5.x?
- [ ] Is multi-select selection planned for a future release?
