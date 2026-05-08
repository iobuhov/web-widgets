# Custom Chart (custom-chart-web)

## Purpose

The Custom Chart widget renders any Plotly.js chart type by accepting raw Plotly `data`, `layout`, and `config` JSON strings. It is the low-level escape hatch for chart visualizations not covered by the structured chart widgets (area, bar, column, etc.). The widget supports merging static JSON with dynamic string attributes, writing click event data to a Mendix attribute, and optional Chart Playground integration. It is offline-capable and does not require an entity context.

## User Scenarios

### [P1] Render a chart from static Plotly JSON
**Given** a Custom Chart with `dataStatic` and `layoutStatic` configured as raw Plotly JSON strings  
**When** the page is loaded  
**Then** Plotly renders the chart using the provided data and layout; the Plotly toolbar (zoom, pan, etc.) is hidden by default  

#### Edge Cases
- If the JSON is invalid, the chart renders empty (parse error is logged to the browser console)
- `sampleData` and `sampleLayout` are used as fallbacks when `dataAttribute` / `layoutAttribute` are not configured

### [P2] Override static data with a dynamic attribute
**Given** a Custom Chart with both `dataStatic` and `dataAttribute` configured  
**When** the page is loaded  
**Then** static and attribute traces are merged by index: attribute trace 0 overrides static trace 0; attribute trace 1 overrides static trace 1; etc.  

#### Edge Cases
- **v6.3.0 breaking change**: prior to v6.3.0, traces were appended as separate elements; from v6.3.0 onward they are merged by index
- If both traces at the same index have their own data keys (e.g., both have `x` and `y`), they are added as independent traces rather than merged
- This pattern allows static JSON to define trace styling/type while the dynamic attribute provides data points
- Array values in layout/config are replaced (not concatenated) during merge

### [P3] Capture click event data
**Given** a Custom Chart with `eventDataAttribute` and/or `onClick` configured  
**When** the user clicks a data point on the chart  
**Then** Plotly click event data for the first clicked point is serialized as JSON and written to `eventDataAttribute`; the `onClick` action fires after the attribute is set  

#### Edge Cases
- Click data includes: `curveNumber`, `pointNumber`, `x`, `y`, `z`, `lat`, `lon`, `location`, `pointNumbers`
- Only the first clicked point (`data.points[0]`) is captured
- The `onClick` action uses the attribute value that was just set — actions relying on `eventDataAttribute` in a nanoflow will see the new value
- The click listener is attached once at chart initialization and cannot be updated without recreating the chart

### [P4] Responsive chart dimensions
**Given** a Custom Chart with `heightUnit=percentageOfWidth`  
**When** the chart container resizes  
**Then** the chart height adjusts proportionally to the container width  

#### Edge Cases
- `percentageOfWidth` (Auto): uses `paddingBottom` CSS trick for ratio-preserving height
- `pixels`: fixed pixel height regardless of container size
- `percentageOfParent`: height relative to parent element
- `percentageOfView`: height relative to the viewport (unique to this widget)
- Min and max height constraints can be applied with the same 4 unit options (plus "none" to disable)
- Font size scales with chart width: `max(12 * (width / 1000), 8)` px — narrower charts get smaller text down to 8px minimum

### [P5] Edit chart JSON via Chart Playground
**Given** a Custom Chart with `showPlaygroundSlot=true` and a Chart Playground widget placed in the playground slot  
**When** the developer opens the playground in the browser  
**Then** the playground allows editing the chart's layout, traces, and configuration as JSON for the current preview session  

#### Edge Cases
- Playground changes are preview-only; they are not persisted to the Mendix model
- The playground integrates with the chart's MobX store via `PlaygroundContext.Provider`
- The chart updates reactively (via MobX `autorun`) when playground edits are applied

## Functional Requirements

- FR-001: The widget MUST accept raw Plotly JSON strings for `data`, `layout`, and `config` via both static string properties and dynamic Mendix string attributes.
- FR-002: When both static and attribute values are provided, the widget MUST merge them by index: attribute trace values override static trace values at the same array index (v6.3.0+ behavior).
- FR-003: When both same-index traces have data keys (any of: `x`, `y`, `z`, `values`, `labels`, `lat`, `lon`, `r`, `theta`, `open`, `high`, `low`, `close`), they MUST be added as independent traces rather than merged.
- FR-004: Array values in layout and config MUST be replaced (not concatenated) during merge.
- FR-005: JSON parse errors in data/layout/config MUST NOT crash the widget; the chart MUST render empty and log the error to the browser console.
- FR-006: The Plotly toolbar (`displayModeBar`) MUST be hidden by default; it can be re-enabled via `configurationOptions`.
- FR-007: The widget MUST use `Plotly.react()` for chart updates (not `Plotly.newPlot`) to preserve zoom state and minimize re-render cost.
- FR-008: Chart updates MUST be debounced by 100ms to prevent excessive re-renders during rapid prop changes.
- FR-009: On click, the widget MUST serialize the first clicked point's event data (`curveNumber`, `pointNumber`, `x`, `y`, `z`, `lat`, `lon`, `location`, `pointNumbers`) to JSON and write it to `eventDataAttribute` before executing `onClick`.
- FR-010: Font sizes in the chart layout MUST scale proportionally with chart width using the formula `max(baseFontSize * (width / 1000), minFontSize)`.
- FR-011: The chart's rendered dimensions MUST always override any width/height values in the user-provided layout JSON.
- FR-012: When the widget is unmounted (e.g., hidden by conditional visibility), `Plotly.purge()` MUST be called to release DOM resources.
- FR-013: The widget MUST support `heightUnit=percentageOfView` (viewport-relative height), which is not available on other chart widgets.
- FR-014: The widget MUST be offline-capable (`offlineCapable=true`) and MUST NOT require an entity context.
- FR-015: When `showPlaygroundSlot=true`, the widget MUST provide a `PlaygroundContext.Provider` for the Chart Playground widget.

## Props Reference

| Name | Type | Default | Caption | Description |
|------|------|---------|---------|-------------|
| `dataStatic` | `string` | `""` | Data (static) | Raw Plotly `data` JSON string defining traces. Merged with `dataAttribute` by index at runtime. |
| `dataAttribute` | `EditableValue<string>` | — | Data (attribute) | Mendix string attribute providing Plotly `data` JSON. Overrides `dataStatic` by index. |
| `sampleData` | `string` | `""` | Sample data | Fallback data JSON used in preview or when `dataAttribute` is not configured. |
| `layoutStatic` | `string` | `""` | Layout (static) | Raw Plotly `layout` JSON string. Merged with `layoutAttribute`. |
| `layoutAttribute` | `EditableValue<string>` | — | Layout (attribute) | Mendix string attribute providing Plotly `layout` JSON. Overrides `layoutStatic`. |
| `sampleLayout` | `string` | `""` | Sample layout | Fallback layout JSON used in preview or when `layoutAttribute` is not configured. |
| `configurationOptions` | `string` | `""` | Configuration options | Raw Plotly `config` JSON. Merged with default `{ displayModeBar: false }`. |
| `onClick` | `ActionValue` | — | On click | Optional action executed after a data point click. Fires after `eventDataAttribute` is updated. |
| `eventDataAttribute` | `EditableValue<string>` | — | Event data attribute | String attribute that receives serialized JSON of the clicked point's event data. |
| `showPlaygroundSlot` | `boolean` | `false` | Show playground | Enables the Chart Playground widget slot. |
| `widthUnit` | `"percentage"` \| `"pixels"` | `"percentage"` | Width unit | Unit for chart width. |
| `width` | `number` | `100` | Width | Width value. |
| `heightUnit` | `"percentageOfWidth"` \| `"pixels"` \| `"percentageOfParent"` \| `"percentageOfView"` | `"percentageOfWidth"` | Height unit | Auto (ratio), fixed pixels, parent-relative, or viewport-relative. |
| `height` | `number` | `75` | Height | Height value. |
| `minHeightUnit` | `"none"` \| `"percentageOfWidth"` \| `"pixels"` \| `"percentageOfParent"` \| `"percentageOfView"` | `"none"` | Min height unit | Unit for minimum height constraint. `none` disables the constraint. |
| `minHeight` | `number` | — | Min height | Minimum height value. |
| `maxHeightUnit` | `"none"` \| `"percentageOfWidth"` \| `"pixels"` \| `"percentageOfParent"` \| `"percentageOfView"` | `"none"` | Max height unit | Unit for maximum height constraint. `none` disables the constraint. |
| `maxHeight` | `number` | — | Max height | Maximum height value. |
| `overflowY` | `"auto"` \| `"scroll"` \| `"hidden"` | `"auto"` | Vertical overflow | Controls vertical overflow behavior of the chart container. |

## Changelog

**6.3.0** (2026-02-17) — **Breaking change**: data merge strategy changed from "append as separate traces" to "merge by index" (attribute overrides static at same index). Fixed re-draw after conditional visibility change.

**1.2.3** — Click event data now returns extended properties (curveNumber, pointNumber, x/y/z coordinates) beyond bounding box.

**1.2.2** — Fixed `eventDataAttribute.setValue()` not working.

**1.2.1** — Fixed layout and data parsing/merging.

**1.0.0** (2025-02-28) — Initial release.

## Open Questions

> Could not be determined from source code alone — requires human review

- [ ] The `onClick` listener is attached once at chart creation and cannot be updated without recreating the chart — if the `onClick` action changes dynamically, does the old action fire? Is this an acknowledged limitation?
- [ ] Is there a documented maximum JSON payload size for `dataAttribute` / `layoutAttribute` before Mendix string attribute limits are exceeded?
- [ ] Can `displayModeBar` be re-enabled via `configurationOptions`? If so, does Plotly toolbar interaction (zoom, pan, download) work correctly with the MobX-based update cycle?
- [ ] The `devMode` property was commented out before release — was it removed permanently or is it planned for a future version?
