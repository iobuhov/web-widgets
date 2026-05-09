# Progress Bar

## Purpose

The Progress Bar widget renders a Bootstrap-style horizontal progress bar on a Mendix page. It accepts current, minimum, and maximum values from three configurable sources — static design-time constants, dynamic entity attributes, or computed Mendix expressions — and displays the proportional fill as a percentage. The widget supports optional text, percentage, or custom widget labels, an optional click action, and validates value ranges at both design time (static) and runtime (dynamic).

## User Scenarios

### [P1] Display progress from a dynamic entity attribute
**Given** a page with a Progress Bar widget configured with `type === "dynamic"` and `dynamicCurrentValue`, `dynamicMinValue`, `dynamicMaxValue` bound to entity attributes  
**When** the page loads and the entity context is available  
**Then** the bar fills to `(currentValue - minValue) / (maxValue - minValue) × 100%`, rounded to the nearest integer

#### Edge Cases
- If dynamic attributes are unavailable (no entity context), the bar defaults to 0% for currentValue, 0 for minValue, and 100 for maxValue.
- The widget does NOT write back to `dynamicCurrentValue` — attributes are read-only for this widget.
- Values outside the [min, max] range are clamped: below min → 0%, above max → 100%. An error alert is also rendered below the bar.

### [P2] Show a percentage or text label inside the bar
**Given** `showLabel` is true and `labelType` is configured  
**When** the chart renders  
**Then** the label appears inside the colored bar fill: a percentage string (e.g., "67%"), a configurable text template string, or custom Mendix widget content

#### Edge Cases
- When `showLabel` is false (default), no label is rendered at all.
- In "small size" mode (Atlas UI `progress-bar-small` CSS class): string labels (text/percentage) are shown as a tooltip on hover; custom widget labels are silently ignored.
- The "small size" behavior is driven by the CSS class applied via Atlas UI, not by a widget prop.

### [P3] Click the bar to trigger an action
**Given** `onClick` is configured with a Mendix action  
**When** the user clicks anywhere on the `.progress` container (not just the colored fill)  
**Then** the configured action executes (microflow, nanoflow, open page, open popup, or blocking popup)

#### Edge Cases
- When `onClick` is not configured, the click handler is not passed to the display component; the bar has no pointer cursor and is not interactive.
- The `.widget-progress-bar-clickable` CSS class is added only when `onClick` is provided.
- All five standard Mendix action types are confirmed functional via e2e tests.

### [P4] Display an error for invalid value ranges
**Given** a dynamic data source provides values where `currentValue < minValue`, `currentValue > maxValue`, or `maxValue < minValue`  
**When** the bar renders  
**Then** a danger-styled error alert appears below the bar; the bar itself still renders with a clamped 0% or 100% fill

#### Edge Cases
- Runtime errors are visible to end users, not only in Studio validation.
- Static mode validates range constraints at design time in Studio; dynamic/expression modes do not validate ranges at design time.
- `maxValue < 1` triggers the `.widget-progress-bar-alert` CSS class on the `.progress` element, visually signaling a configuration issue.

### [P5] Use in Mendix data grid "listen to" pattern
**Given** a Progress Bar widget in dynamic mode within the same context as a data grid  
**When** the user selects a data grid row  
**Then** the bar updates to reflect the selected row's attribute values

#### Edge Cases
- This pattern is e2e-confirmed; the widget responds to Mendix context selection changes.
- The widget works correctly in all common Mendix layout containers: group box, list view, template grid, and tab container.

## Functional Requirements

- FR-001: The widget MUST support three value source modes: static (design-time integer constants), dynamic (Mendix entity attributes), and expression (computed Mendix expressions).
- FR-002: In dynamic and expression modes, default values MUST be: currentValue = 50, minValue = 0, maxValue = 100.
- FR-003: Percentage MUST be calculated as `Math.round(((currentValue - minValue) / (maxValue - minValue)) * 100)` with `Math.abs()` applied to the result. Values below minValue clamp to 0; values above maxValue clamp to 100.
- FR-004: The bar fill width MUST be set via inline `style.width` as a percentage string (e.g., `"67%"`).
- FR-005: When value range is invalid (currentValue < minValue, currentValue > maxValue, or maxValue < minValue), the widget MUST render both a clamped bar AND a danger-styled error alert.
- FR-006: The click target MUST be the `.progress` container element (not the inner `.progress-bar` fill div).
- FR-007: When `labelType === "text"` or `"percentage"` and the CSS class includes `progress-bar-small`, the label MUST be set as the `title` attribute (tooltip). When `labelType === "custom"`, no `title` attribute is set and the label is silently omitted in small mode.
- FR-008: Labels MUST render inside the `.progress-bar` fill div — the label is positioned within the colored portion of the bar.
- FR-009: Default Atlas UI CSS classes MUST be applied: `progress-bar-medium` (size) and `progress-bar-primary` (color), in addition to any custom `class` prop.
- FR-010: The widget MUST be offline capable.
- FR-011: `onClick` MUST be marked `required="false"` in the widget XML to avoid unnecessary Studio Pro warnings.

## Props Reference

| Name | Type | Default | Caption | Description |
|------|------|---------|---------|-------------|
| `type` | `"static" \| "dynamic" \| "expression"` | `"static"` | Value type | Determines which set of value props is used. |
| `staticCurrentValue` | `number` | `50` | Progress value | Static integer value for current progress (static mode). |
| `staticMinValue` | `number` | `0` | Minimum value | Static integer minimum (static mode). |
| `staticMaxValue` | `number` | `100` | Maximum value | Static integer maximum (static mode). |
| `dynamicCurrentValue` | `EditableValue<Big>?` | — | Progress attribute | Entity attribute for current value (dynamic mode). Decimal/Integer/Long. Read-only in widget. |
| `dynamicMinValue` | `EditableValue<Big>?` | — | Minimum attribute | Entity attribute for minimum value (dynamic mode). |
| `dynamicMaxValue` | `EditableValue<Big>?` | — | Maximum attribute | Entity attribute for maximum value (dynamic mode). |
| `expressionCurrentValue` | `DynamicValue<Big>?` | — | Progress expression | Computed expression for current value (expression mode). |
| `expressionMinValue` | `DynamicValue<Big>?` | — | Minimum expression | Computed expression for minimum value (expression mode). |
| `expressionMaxValue` | `DynamicValue<Big>?` | — | Maximum expression | Computed expression for maximum value (expression mode). |
| `showLabel` | `boolean` | `false` | Show label | Enables label display inside the bar. |
| `labelType` | `"text" \| "percentage" \| "custom"` | `"text"` | Label type | Selects label format. Active only when `showLabel` is true. |
| `labelText` | `DynamicValue<string>?` | — | Label text | Text template for label content (text mode). |
| `customLabel` | `ReactNode?` | — | Custom label | Widget slot for custom label content (custom mode). |
| `onClick` | `ActionValue?` | — | On click | Optional Mendix action triggered by clicking the bar container. |

## Changelog

**v3.2.2 (2024-08-28):** `onClick` made `required="false"` to remove unnecessary Studio Pro warnings.

**v3.2.0 (2023-06-05):** Page explorer caption updated to `[type, currentValue]` format for easier identification.

**v3.1.0 (2021-12-23):** Dark mode support added for structure preview.

## Open Questions
> Could not be determined from source code alone — requires human review
- [ ] Is the `Math.abs()` call in `calculatePercentage` intentional behavior or a defensive workaround? Could it mask misconfigured negative ranges?
- [ ] Is there a plan to add `indeterminate` mode (animated bar with no defined value)?
- [ ] Should the error alert shown for invalid runtime values be configurable (e.g., suppressible for cases where brief out-of-range states are expected during data loading)?
