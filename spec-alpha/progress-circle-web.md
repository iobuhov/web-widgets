# Progress Circle

## Purpose

The Progress Circle widget renders an animated SVG circular arc on a Mendix page to represent proportional progress between a minimum and maximum value. It accepts current, minimum, and maximum values from three configurable sources — static design-time integers, dynamic entity attributes, or computed Mendix expressions — and animates the arc fill as a percentage. The widget supports four label modes (percentage, text, custom widget, or none) centered inside the circle, an optional click action, and validates value ranges at both design time (static) and runtime (dynamic/expression). The widget is offline capable and requires no server-side calls for rendering.

## User Scenarios

### [P1] Display progress from a dynamic entity attribute
**Given** a page with a Progress Circle widget configured with `type === "dynamic"` and `dynamicCurrentValue`, `dynamicMinValue`, `dynamicMaxValue` bound to entity attributes  
**When** the page loads and entity context is available  
**Then** the circle arc fills to `(currentValue - minValue) / (maxValue - minValue) × 100%` and animates from 0 to the computed percentage over 800 ms

#### Edge Cases
- If dynamic attributes are unavailable (no entity context), the widget defaults to currentValue = 0, minValue = 0, maxValue = 100 (showing 0% fill).
- When the underlying attribute value changes, the circle re-animates from its current position to the new percentage.
- The widget does NOT write back to dynamic attributes — they are read-only for this widget.

### [P2] Show a label inside the circle
**Given** `showLabel` is true and `labelType` is configured  
**When** the circle renders  
**Then** the label appears centered inside the SVG circle: a percentage string (e.g., "67%"), a configurable text template string, or custom Mendix widget content placed in the center container

#### Edge Cases
- When `showLabel` is false, no label is rendered.
- When `labelType === "custom"`, the Studio Pro structure preview returns `null` (cannot preview arbitrary widget content); the design canvas also shows nothing.
- Percentage label is always a rounded integer (e.g., 67, not 67.3).

### [P3] Click the circle to trigger an action
**Given** `onClick` is configured with a Mendix action  
**When** the user clicks anywhere on the circle element  
**Then** the configured Mendix action executes

#### Edge Cases
- When `onClick` is not configured, no click handler is attached; the cursor does not change.
- The `"widget-progress-circle-clickable"` CSS class is applied only when `onClick` is provided (enables `cursor: pointer` via CSS).

### [P4] Display an error for invalid value ranges
**Given** a data source provides values where `currentValue < minValue`, `currentValue > maxValue`, or `maxValue < minValue`  
**When** the circle would render  
**Then** a danger-styled error alert is rendered in place of the circle — the circle is NOT shown alongside the error

#### Edge Cases
- The error replaces (not supplements) the circle: when an error condition is detected, only the alert is rendered.
- Static mode validates range constraints at design time in Studio Pro; dynamic and expression modes validate at runtime only.
- Out-of-range values produce the error alert; the `calculatePercentage` utility clamps values before any percentage label is displayed.

### [P5] Use static values for a design-time-configured circle
**Given** `type === "static"` and `staticCurrentValue`, `staticMinValue`, `staticMaxValue` are configured in Studio Pro  
**When** the page loads  
**Then** the circle immediately renders at the configured percentage with no loading delay

#### Edge Cases
- Static mode accepts only Integer type; Decimal and Long must use dynamic or expression mode.
- The IDE validates that `staticCurrentValue` is within `[staticMinValue, staticMaxValue]` at design time.

## Functional Requirements

- FR-001: The widget MUST support three value source modes: static (design-time integers), dynamic (Mendix entity attributes of type Integer, Decimal, or Long), and expression (computed Mendix expressions returning Decimal).
- FR-002: In dynamic and expression modes, the widget MUST use defaults of currentValue = 0, minValue = 0, maxValue = 100 when attribute/expression values are unavailable.
- FR-003: The displayed default currentValue in the IDE is 50 (50% fill), making the widget visually non-empty in structure previews and during slow data loads.
- FR-004: Percentage MUST be calculated as `Math.round(Math.abs((clampedValue - minValue) / (maxValue - minValue)) * 100)`, returning an integer 0–100.
- FR-005: The circle arc MUST animate to the new percentage over 800 ms on both initial mount and subsequent value changes.
- FR-006: When value range is invalid (`currentValue < minValue`, `currentValue > maxValue`, or `maxValue < minValue`), the widget MUST render a danger-styled error alert and NOT render the circle.
- FR-007: The SVG circle MUST be centered at (50, 50) in a 100×100 viewBox. The radius MUST be adjusted so strokes never overflow the viewBox boundary: `radius = 50 - max(strokeWidth, trailWidth) / 2`.
- FR-008: The SVG scales to `width: 100%` of its container via inline style.
- FR-009: On unmount, `Circle.destroy()` MUST be called to release DOM elements allocated by the SVG library.
- FR-010: Label MUST render as the center text of the SVG shape. When `labelType === "custom"`, the ReactNode content is injected via `innerHTML` into the center text container.
- FR-011: The widget MUST be offline capable.
- FR-012: `onClick` MUST be optional (`required="false"` in widget XML).
- FR-013: The IDE MUST hide irrelevant value property groups based on the selected `type`: only one mode's properties are visible at a time.
- FR-014: Page explorer caption MUST display as `[type, currentValue]` (e.g., `[dynamic, 0]`).

## Props Reference

| Name | Type | Default | Caption | Description |
|------|------|---------|---------|-------------|
| `type` | `"static" \| "dynamic" \| "expression"` | `"static"` | Value type | Determines which set of value props is used. IDE hides irrelevant groups based on this. |
| `staticCurrentValue` | `number` | `50` | Progress value | Static integer for current progress (static mode only). |
| `staticMinValue` | `number` | `0` | Minimum value | Static integer minimum (static mode only). |
| `staticMaxValue` | `number` | `100` | Maximum value | Static integer maximum (static mode only). |
| `dynamicCurrentValue` | `EditableValue<Big>?` | — | Progress attribute | Entity attribute for current value (dynamic mode). Integer, Decimal, or Long. |
| `dynamicMinValue` | `EditableValue<Big>?` | — | Minimum attribute | Entity attribute for minimum value (dynamic mode). |
| `dynamicMaxValue` | `EditableValue<Big>?` | — | Maximum attribute | Entity attribute for maximum value (dynamic mode). |
| `expressionCurrentValue` | `DynamicValue<Big>?` | — | Progress expression | Computed expression for current value (expression mode). Decimal. |
| `expressionMinValue` | `DynamicValue<Big>?` | — | Minimum expression | Computed expression for minimum value (expression mode). |
| `expressionMaxValue` | `DynamicValue<Big>?` | — | Maximum expression | Computed expression for maximum value (expression mode). |
| `showLabel` | `boolean` | `false` | Show label | Enables label display inside the circle center. |
| `labelType` | `"text" \| "percentage" \| "custom"` | `"percentage"` | Label type | Selects label format. Active only when `showLabel` is true. |
| `labelText` | `DynamicValue<string>?` | — | Label text | Text string for the center label (text mode). |
| `customLabel` | `ReactNode?` | — | Custom label | Widget slot for custom ReactNode content at the center (custom mode). |
| `onClick` | `ActionValue?` | — | On click | Optional Mendix action triggered by clicking the circle. |

## Changelog

**v3.3.3 (2026-02-10):** License documentation update only.

**v3.3.2 (2024-08-28):** Disabled the "action required" Studio Pro warning for the widget.

**v3.2.0 (2023-06-05):** Page explorer caption updated to `[type, currentValue]` format; dark mode structure preview icons added.

## Open Questions
> Could not be determined from source code alone — requires human review
- [ ] Is the 800 ms animation duration configurable via widget props or always fixed at the default?
- [ ] Can the circle's stroke color or width be customized via widget props, or only via custom CSS?
- [ ] Does the `Math.abs()` in `calculatePercentage` produce any observable behavior for negative minValue configurations (e.g., min=-50, max=50)?
