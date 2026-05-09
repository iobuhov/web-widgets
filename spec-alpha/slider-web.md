# Slider

## Purpose

The Slider widget is a numeric input control that allows users to select a value within a defined range by dragging a handle or using keyboard arrow keys. It binds to an Integer, Long, or Decimal entity attribute and supports static, dynamic (entity attribute), or expression-based min, max, and step values. The widget is built on `@rc-component/slider` and renders a horizontal (default) or vertical track with optional tick marks and a drag-time tooltip. It is categorized under "Input elements," requires entity context, and supports offline use.

## User Scenarios

### [P1] User selects a value by dragging
**Given** a Slider bound to a numeric attribute with min=0, max=100, step=1  
**When** the user drags the slider handle  
**Then** the entity attribute is updated immediately on each drag step; the configured `onChange` action fires 500ms after the last drag movement (debounced, not per-step)  

#### Edge Cases
- When the entity attribute has no value (`undefined`), the handle renders at the minimum position (`left: 0%`).
- The attribute is never `undefined` after the user interacts — the first drag sets a value.
- If the component unmounts during a drag, the pending debounced `onChange` action is cancelled.

### [P2] Slider with dynamic min/max
**Given** a Slider with `minValueType = "dynamic"` and `maxValueType = "dynamic"`, both bound to entity attributes  
**When** the page loads and the attribute values are still fetching  
**Then** the slider renders disabled (greyed out, `cursor: not-allowed`) until both values are loaded  
**When** the values load  
**Then** the slider becomes interactive with the resolved min/max range  

#### Edge Cases
- Once min/max load, the `useNumber` latch ensures the slider never reverts to the loading state on subsequent re-renders.
- If `min >= max` at runtime (dynamic values), marks are suppressed (no crash), but the rc-slider track may render in a visually broken state. There is no runtime guard beyond mark suppression.

### [P3] Tooltip during drag
**Given** a Slider with `showTooltip = true` (default) and `tooltipType = "value"` (default)  
**When** the user starts dragging  
**Then** the tooltip appears immediately showing the current numeric value  
**When** the user stops dragging  
**Then** the tooltip disappears immediately (no delay)  

**When** `tooltipAlwaysVisible = true`  
**Then** the tooltip is always visible regardless of drag state  

### [P4] Keyboard navigation
**Given** a focused Slider  
**When** the user presses ArrowUp or ArrowRight  
**Then** the value increases by one step  
**When** the user presses ArrowLeft or ArrowDown  
**Then** the value decreases by one step  

## Functional Requirements

- FR-001: System MUST bind to Integer, Long, or Decimal entity attributes; the value is stored as `Big` internally and converted to `number` for rc-slider.
- FR-002: Min, max, and step MUST each independently support three source modes: static (design-time decimal), dynamic (entity attribute), and expression (Mendix expression returning Decimal).
- FR-003: The attribute MUST be updated on every drag step immediately (`valueAttribute.setValue(new Big(value))`); the `onChange` action MUST be debounced 500ms and fire once per drag gesture.
- FR-004: When any of min, max, step, or the attribute value is loading, the slider MUST render in a disabled state as a placeholder.
- FR-005: When `valueAttribute` has no value (undefined), the handle MUST render at the minimum position.
- FR-006: Marks MUST be generated at `(noOfMarkers + 1)` evenly spaced positions from min to max, formatted to `decimalPlaces` decimal places. When `noOfMarkers = 0`, no marks are rendered.
- FR-007: Mark positions are arithmetic (independent of step) — marks may appear at positions the slider handle cannot reach when `(max - min)` is not evenly divisible by step. This is intentional behavior.
- FR-008: The tooltip visibility MUST be fully controlled by `visible={tooltipAlwaysVisible || dragging}`. When `tooltipAlwaysVisible = false`, the tooltip appears on drag start and disappears immediately on drag end (`mouseLeaveDelay = 0`).
- FR-009: The tooltip MUST be anchored to the slider's container div (not `document.body`) to prevent misposition in scrollable containers.
- FR-010: The slider MUST render with `role="slider"` and `aria-valuenow` on the handle element, linked to the Mendix Label system property via `ariaLabelledByForHandle`.
- FR-011: Advanced properties (marks, decimal places, orientation, tooltip details) MUST be hidden in web Studio Pro unless `advanced = true`. Desktop Studio Pro users see all properties unconditionally.
- FR-012: Studio Pro MUST validate: tooltip text non-empty when `tooltipType = "customText"`, `min < max` for static values, step > 0, `decimalPlaces >= 0`, and required attributes present for dynamic/expression modes.
- FR-013: The widget MUST require entity context and support offline use.

## Props Reference

### General

| Name | Type | Default | Caption | Description |
|------|------|---------|---------|-------------|
| `valueAttribute` | `EditableValue<Big>` | — | Value | The Integer, Long, or Decimal attribute to read and write |
| `advanced` | boolean | `false` | Advanced options | Unlocks advanced properties in Mendix Studio (web); always visible in Studio Pro (desktop) |
| `minValueType` | `"static"` \| `"dynamic"` \| `"expression"` | — | Minimum type | Source mode for the minimum value |
| `staticMinimumValue` | `Big` (optional) | — | Minimum value | Static design-time minimum |
| `minAttribute` | `EditableValue<Big>` (optional) | — | Minimum attribute | Dynamic minimum bound to an entity attribute |
| `expressionMinimumValue` | `DynamicValue<Big>` (optional) | — | Minimum expression | Minimum computed from a Mendix expression |
| *(same pattern for max and step)* | | | | |
| `showTooltip` | boolean | `true` | Show tooltip | Shows a tooltip on the handle during drag (or always if `tooltipAlwaysVisible`) |
| `tooltipType` | `"value"` \| `"customText"` | `"value"` | Tooltip type | `"value"` shows the numeric value; `"customText"` shows a text template expression |
| `tooltip` | `DynamicValue<string>` (optional) | — | Tooltip text | Custom tooltip text template; only shown when `tooltipType = "customText"` |
| `tooltipAlwaysVisible` | boolean | `false` | Always visible | Keeps the tooltip visible at all times |

### Track

| Name | Type | Default | Caption | Description |
|------|------|---------|---------|-------------|
| `noOfMarkers` | integer | `2` | Number of markers | Number of mark intervals; `noOfMarkers = 2` produces 3 labels (min, mid, max); 0 hides marks |
| `decimalPlaces` | integer | `0` | Decimal places | Number of decimal places shown in mark labels |
| `orientation` | `"horizontal"` \| `"vertical"` | `"horizontal"` | Orientation | Layout orientation |
| `heightUnit` | — | — | Height unit | Only relevant for vertical orientation |
| `height` | — | — | Height | Only relevant for vertical orientation |

### Events

| Name | Type | Default | Caption | Description |
|------|------|---------|---------|-------------|
| `onChange` | `ActionValue` (optional) | — | On change | Mendix action executed 500ms after the last drag step |

System properties supported: Name, Editability, Visibility, Label, TabIndex.

## Changelog

- **v3.0.2 (2026-02-19)**: Fixed accessibility — slider handle now correctly linked to Mendix Label system property via `ariaLabelledByForHandle` (contributed by @DiljohnSingh).
- **v3.0.0 (2026-01-06)**: Updated `@rc-component/slider` for React 19 support.
- **v2.1.4 (2024-08-28)**: Made the `onChange` action optional.
- **v2.1.3 (2024-07-15)**: Fixed initial value being reset to minimum on page load (the `useNumber` latch mechanism).
- **v2.1.2 (2024-06-25)**: Fixed tooltip position when the page is scrolled (`getTooltipContainer` anchored to slider div).

## Open Questions

> Could not be determined from source code alone — requires human review
- [ ] When `min >= max` with dynamic values at runtime, is there any user-visible error or fallback beyond the visually broken track? Should a validation message be shown?
- [ ] The Studio Pro design preview shows mark labels with ≥2 decimal places (due to `decimalPlaces ?? 2` fallback), while the runtime default is 0 decimal places. Is this cosmetic discrepancy acceptable or should the preview fallback be changed to `?? 0`?
