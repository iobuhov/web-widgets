# Heatmap Chart

## Purpose

The Heatmap Chart widget renders a two-dimensional color-encoded grid in a Mendix application, where each cell's color encodes a numeric value (the "heat") at a given (x, y) coordinate pair. It is used to visualize magnitude distributions across two categorical or enumeration axes — for example, density, intensity, or frequency data organized by row and column categories. The widget supports custom colorscales, optional numeric cell annotations, per-cell click actions, tooltip hover text, and configurable sort order for both axes.

## User Scenarios

### [P1] Display a color-coded heatmap grid
**Given** the widget is placed on a Mendix page with `seriesDataSource` configured and items available  
**When** the page renders  
**Then** a Plotly heatmap is displayed with each cell color-encoded according to the `seriesValueAttribute` value; axes are labeled from the `horizontalAxisAttribute` and `verticalAxisAttribute` values; gaps between cells are rendered (xgap=1, ygap=1); both axes have fixed range (no zoom or pan)

#### Edge Cases
- Missing (x, y) combinations produce `null` z-values, which render as blank/empty cells — no interpolation occurs (`connectgaps: false`).
- If `seriesDataSource` has no items, the chart renders with an empty z-matrix and no cells.
- Axis attribute values are formatted using `.toLocaleString()`, so locale-specific formatting applies to axis labels.

### [P2] Configure a custom colorscale
**Given** `showScale` is `true` and `scaleColors` contains at least two entries (one at 0% and one at 100%)  
**When** the chart renders  
**Then** the colorbar is displayed at the top-right of the chart and cell colors follow the user-defined gradient; `scaleColors` entries are sorted by `valuePercentage` ascending before mapping to Plotly colorscale format

#### Edge Cases
- If fewer than two `scaleColors` entries are provided, the default colorscale is used: `[[0, "#17347B"], [0.5, "#0595DB"], [1, "#76CA02"]]` (dark blue → light blue → green).
- When `smoothColor` is `true` (requires `showScale` and `enableAdvancedOptions`), Plotly's `"best"` smoothing is applied between data points, producing a gradient effect. When `false`, `zsmooth` is `false` (discrete cells).

### [P3] Overlay numeric cell value annotations
**Given** `showValues` is `true` (requires `showScale` and `enableAdvancedOptions`)  
**When** the chart renders  
**Then** each cell displays its numeric z-value as a text annotation formatted with `.toLocaleString()`; annotation text color is set by `valuesColor` or defaults to `#555`

#### Edge Cases
- An empty dataset with `showValues=true` produces zero annotations without error.
- Annotations are positioned in data coordinates (`xref: "x"`, `yref: "y"`), use Open Sans 14pt font, and have no arrow (`showarrow: false`).
- `valuesColor` is only visible in Studio Pro when `showScale`, `enableAdvancedOptions`, and `showValues` are all `true`.

### [P4] Sort axis values
**Given** `horizontalSortOrder` or `verticalSortOrder` (or both) are configured  
**When** the chart renders  
**Then** axis values are sorted by the corresponding sort attribute (which may differ from the display attribute); vertical sort is applied before horizontal sort; each axis independently supports ascending or descending order

#### Edge Cases
- Sort attributes accept a broader set of types than display attributes: Decimal, Long, Integer, String, AutoNumber, DateTime.
- Unique axis values are deduplicated using a Set, so sort order before deduplication determines the final axis order (JavaScript insertion-order Set semantics).
- Horizontal descending sort reverses the column order; vertical descending sort reverses the row order. Both can be combined independently.

### [P5] Click to trigger an action or set selection
**Given** `onClickAction` is configured and `seriesItemSelection` supports Single selection  
**When** the user clicks a cell  
**Then** the `onClickAction` is executed for the corresponding datasource ObjectItem; if `seriesItemSelection` is Single, the clicked item becomes the selected item

#### Edge Cases
- If the direct ObjectItem reference passed by Plotly is null or undefined, the widget resolves the item by matching x/y/z coordinates in the current dataset.
- Selection state is managed separately from the datasource items array (`dataSourceItems` is always `[]` in the series data; selection is tracked via the item map ref).

### [P6] Custom hover tooltip
**Given** `tooltipHoverText` (a textTemplate) is configured  
**When** the user hovers over a cell  
**Then** the custom hover text for that cell's datasource item is displayed; Plotly `hoverinfo` is set to `"text"` and a hovertext matrix matching the z-matrix is built from the template values

#### Edge Cases
- When `tooltipHoverText` is not configured, `hoverinfo` is `"none"` at the series level (no tooltip shown).
- The hovertext matrix is built with the same x/y indexing as the z-matrix (`[yIndex][xIndex]`).

## Functional Requirements

- FR-001: The widget MUST render a Plotly chart of type `"heatmap"` with a single data series sourced from `seriesDataSource`.
- FR-002: The z-matrix MUST be indexed as `z[yIndex][xIndex]` — rows correspond to y-axis values, columns to x-axis values. Missing (x, y) pairs MUST produce `null` (not `0`).
- FR-003: Both axes MUST have `fixedrange: true` — users MUST NOT be able to zoom or pan the axes.
- FR-004: Cell gaps MUST be `xgap=1` and `ygap=1` (1px gap between all cells).
- FR-005: The widget MUST NOT interpolate missing data cells (`connectgaps: false`).
- FR-006: When fewer than 2 `scaleColors` entries are provided, the widget MUST use the default colorscale: `[[0, "#17347B"], [0.5, "#0595DB"], [1, "#76CA02"]]`.
- FR-007: The colorbar MUST be positioned at `y:1, yanchor:"top", ypad:0, xpad:5` (top-right) when `showScale` is `true`. The colorbar outline color is `#9ba492`.
- FR-008: When `showValues` is `true`, the widget MUST generate one annotation per cell using the z-value formatted via `.toLocaleString()`. An empty dataset MUST produce zero annotations without error.
- FR-009: Vertical sort MUST be applied before horizontal sort when both are configured. Sort attribute values of Big/Decimal type MUST be converted via `.toNumber()` before sorting.
- FR-010: The click handler MUST resolve the ObjectItem by direct reference first; if the reference is null, it MUST fall back to matching by x/y/z value coordinates in the current dataset.
- FR-011: The widget MUST support auto-resize inside Mendix popup dialogs (fixed in v5.0.1).
- FR-012: The widget MUST be `offlineCapable="true"` and require Mendix version 9.6.0 or later.
- FR-013: Advanced options (`customLayout`, `customConfigurations`, `customSeriesOptions`, `enableThemeConfig`) MUST be hidden in Studio Pro unless `enableAdvancedOptions` is `true` (on web). On Studio (desktop), advanced options are always accessible.
- FR-014: The `playground` slot MUST be hidden unless `showPlaygroundSlot` is `true`.

## Props Reference

| Name | Type | Default | Caption | Description |
|------|------|---------|---------|-------------|
| `seriesDataSource` | Datasource (list, required) | — | Data source | Root datasource for all chart data. All axis and value attributes are scoped to this source. |
| `seriesValueAttribute` | Attribute (Decimal \| Integer \| Long) | — | Value attribute | Numeric heat value placed at each cell. Non-numeric types are not supported. |
| `horizontalAxisAttribute` | Attribute (String \| Enum) | — | Horizontal axis | X-axis category attribute. Accepts String or Enum types. |
| `verticalAxisAttribute` | Attribute (String \| Enum) | — | Vertical axis | Y-axis category attribute. Accepts String or Enum types. |
| `horizontalSortAttribute` | Attribute (Decimal \| Long \| Integer \| String \| AutoNumber \| DateTime) | — | Horizontal sort | Sort attribute for x-axis order. May differ from `horizontalAxisAttribute`. |
| `verticalSortAttribute` | Attribute (Decimal \| Long \| Integer \| String \| AutoNumber \| DateTime) | — | Vertical sort | Sort attribute for y-axis order. May differ from `verticalAxisAttribute`. Applied before horizontal sort. |
| `horizontalSortOrder` | Enum (`asc` \| `desc`) | `"asc"` | Horizontal sort order | Sort direction for x-axis values. |
| `verticalSortOrder` | Enum (`asc` \| `desc`) | `"asc"` | Vertical sort order | Sort direction for y-axis values. |
| `seriesItemSelection` | Enum (`None` \| `Single`) | `"None"` | Selection | Enables single-item selection on cell click. |
| `onClickAction` | Action | — | On click | Action executed when a cell is clicked. Scoped to `seriesDataSource`. |
| `tooltipHoverText` | TextTemplate | — | Tooltip | Custom hover text per cell, resolved from the datasource item. When absent, no tooltip is shown. |
| `showScale` | Boolean | `false` | Show scale | Displays the colorbar (legend) and unlocks advanced color options. |
| `scaleColors` | List | — | Scale colors | Custom colorscale gradient stops. Requires at least 2 entries (0% and 100%); otherwise defaults apply. |
| `scaleColors.valuePercentage` | Integer (0–100) | — | Value percentage | Position of this color stop in the gradient. |
| `scaleColors.colour` | String (CSS color) | — | Colour | CSS color value for this stop. |
| `smoothColor` | Boolean | `false` | Smooth color | Enables gradient interpolation between data points (`zsmooth: "best"`). Requires `showScale` and `enableAdvancedOptions`. |
| `showValues` | Boolean | `false` | Show values | Overlays the numeric z-value as text annotation on each cell. Requires `showScale` and `enableAdvancedOptions`. |
| `valuesColor` | String (CSS color) | — | Values color | Annotation text color override. Defaults to `#555`. Requires `showScale`, `enableAdvancedOptions`, and `showValues`. |
| `gridLines` | Enum (`none` \| `horizontal` \| `vertical` \| `both`) | `"none"` | Grid lines | Controls grid line visibility on the chart axes. |
| `widthUnit` | Enum (`percentage` \| `pixels`) | `"percentage"` | Width unit | Unit for widget width. |
| `width` | Integer | `100` | Width | Widget width value in the selected unit. |
| `heightUnit` | Enum (`percentageOfWidth` \| `pixels` \| `percentageOfParent`) | `"percentageOfWidth"` | Height unit | Unit for widget height. |
| `height` | Integer | `75` | Height | Widget height value in the selected unit. |
| `enableAdvancedOptions` | Boolean | `false` | Enable advanced options | Unlocks advanced customization properties in Studio Pro. Always accessible in Studio (desktop). |
| `customLayout` | String | — | Custom layout | JSON string merged with the Plotly layout object. |
| `customConfigurations` | String | — | Custom configurations | JSON string merged with the Plotly config object. |
| `customSeriesOptions` | String | — | Custom series options | JSON string merged with the Plotly series data object. |
| `enableThemeConfig` | Boolean | — | Enable theme config | Loads additional Plotly configuration from the Mendix theme folder. |
| `showPlaygroundSlot` | Boolean | `false` | Show playground | Exposes the playground widget slot for advanced configuration. |

## Changelog

- **v6.3.0 (current):** Current release. Uses Plotly 3.x (`plotly.js-dist-min ^3.0.0`).
- **v6.2.1 (2025-07-15):** Fixed on-click events — datasource was not correctly added and selection listening was broken.
- **v6.0.0 (2025-02-28):** Upgraded Plotly.js to version 3.0 (breaking semver change).

## Open Questions

> Could not be determined from source code alone — requires human review

- [ ] Is there a defined maximum number of unique x or y axis values? No limit is declared in the XML or data hook; very large axis sets may cause rendering performance issues.
- [ ] The `alt` text for the structure preview image in `HeatMap.editorPreview.tsx` reads "Bubble chart" — this appears to be a copy-paste artifact. Confirm whether this should be corrected to "Heatmap chart".
- [ ] `smoothColor` and `showValues` are gated behind both `showScale` and `enableAdvancedOptions`. Is this intentional (i.e., these are considered advanced behaviors that should not appear without the colorscale), or an unintended constraint that may be relaxed in a future release?
- [ ] The e2e tests cover custom color, ascending sort, and descending sort, but do not test click interaction or tooltip hover. Is click/selection behavior covered by manual testing only?
