# Rating (Star Rating)

## Purpose

The Rating widget (published as "Star Rating" on the Mendix Marketplace) renders a row of interactive star icons on a Mendix page, enabling users to select a rating value by clicking a star. The selected value is stored in a writable Mendix entity attribute (Integer, Long, or Decimal). The widget supports configurable star counts, custom filled/empty icons, an optional stretch-bounce animation on selection, full keyboard navigation using ARIA radiogroup semantics, and a toggle-to-clear interaction (clicking the currently selected star sets the value to 0). The widget is offline capable and requires entity context.

## User Scenarios

### [P1] Select a rating by clicking a star
**Given** a page with a Rating widget bound to `rateAttribute`  
**When** the user clicks the third star  
**Then** `rateAttribute.setValue(new Big(3))` is called, the `onChange` action executes, and the first three stars render as filled

#### Edge Cases
- Clicking the currently selected star calls `onChange(0)` — clearing the rating to 0 (toggle-to-clear behavior).
- Stars are 1-indexed: clicking the star at position N sets the value to N (not N-1).
- When `rateAttribute.readOnly` is true, no interaction is processed and the widget renders as disabled.
- If the stored value exceeds `maximumStars`, the displayed value is clamped to `maximumStars` — no extra stars appear.

### [P2] Keyboard navigation between stars
**Given** the widget has keyboard focus  
**When** the user presses arrow keys  
**Then** focus moves between stars: ArrowRight/ArrowUp increases focus, ArrowLeft/ArrowDown decreases focus; pressing Space or Enter selects the focused star

#### Edge Cases
- The widget uses the roving tabIndex pattern: only the currently selected star has `tabIndex="0"`; all others have `tabIndex="-1"`.
- Focus moves between stars with arrow keys but does NOT change the stored value until Space/Enter is pressed.
- Disabled state blocks all pointer and keyboard interactions.

### [P3] Display a hover animation when hovering over stars
**Given** `animation` is true and the widget is not disabled  
**When** the user hovers over a star  
**Then** a stretch-bounce overlay animation plays on the hovered star (scales 1 → 1.5 → 0.9 → 1.2 → 1 over 500 ms)

#### Edge Cases
- When `animation` is false, no hover overlay is rendered.
- When the widget is disabled, hover animation is also suppressed.
- The animation runs on a `.rating-item-hover` overlay element, not on the base icon — the overlay is separate from the permanently rendered icon so state transitions are smooth.
- The hover overlay element has `animation: none` applied explicitly to prevent CSS conflicts with the selection animation class.

### [P4] Use custom icons for filled and empty states
**Given** `icon` and/or `emptyIcon` are configured with Mendix image or glyph properties  
**When** the widget renders  
**Then** the configured icons are used in place of the default SVG star; the `animate` CSS class is applied to custom icons as well

#### Edge Cases
- Custom icons are used only when Mendix `isAvailable()` returns true; unavailable or unconfigured icons fall back to the built-in SVG star.
- Both `icon` and `emptyIcon` are `DynamicValue<WebIcon>` — they can be expression-driven and change at runtime.
- Three icon source types are supported: Mendix glyph (CSS class), image (URL), and sprite icon.
- An empty `<div>` placeholder renders when `IconInternal` has no value, preventing layout shifts.

### [P5] Show a design canvas preview in Studio Pro
**Given** the widget is placed on a page in Studio Pro design canvas  
**When** the canvas renders  
**Then** the actual `Rating` React component renders with `value = maximumStars - 1` (all-but-last star filled), using the configured icon and star count

#### Edge Cases
- `maximumStars = null` in preview props defaults to 5.
- Preview renders the real React component — not a placeholder image.
- The structure preview (not the canvas) shows filled/empty star SVGs assembled from data URIs, capped at 50 stars to prevent Studio Pro performance issues.
- The IDE rejects `maximumStars = 0` at design time.

## Functional Requirements

- FR-001: The widget MUST bind one writable Mendix entity attribute (`rateAttribute`) of type Integer, Long, or Decimal.
- FR-002: On star click, the widget MUST call `rateAttribute.setValue(new Big(value))` then `onChange?.execute()`.
- FR-003: Clicking the currently-selected star MUST call `onChange(0)`, clearing the rating. The Rating component passes 0 to its callback; the container writes `new Big(0)` to the attribute.
- FR-004: When `rateAttribute.readOnly` is true, the widget MUST render the `Rating` component with `disabled={true}` and MUST NOT call `setValue` or `onChange.execute` on any interaction.
- FR-005: Displayed value MUST be clamped to `maximumStars` — a stored value greater than the maximum renders as `maximumStars` filled stars.
- FR-006: The widget MUST render exactly `maximumStars` star items in the DOM.
- FR-007: Stars MUST use the roving tabIndex pattern: the current value's star has `tabIndex="0"`, all others have `tabIndex="-1"`.
- FR-008: The following keyboard bindings MUST be implemented: ArrowLeft/ArrowDown move focus left/down; ArrowRight/ArrowUp move focus right/up; Space and Enter select the focused star.
- FR-009: The role attribute MUST be `"radiogroup"` on the container and `"radio"` on each star item.
- FR-010: Hover animation (`.rating-item-hover` overlay with `stretch-bounce` keyframe) MUST render only when `animation === true` AND the widget is not disabled.
- FR-011: Default star color MUST be `#ccc` (empty) and `#ffa611` (filled); default star size MUST be 24×24 px using `currentColor` SVG paths.
- FR-012: Focus outline MUST appear only for keyboard navigation (`:focus-visible`), not on mouse click.
- FR-013: Disabled state MUST apply `opacity: 0.65` and `cursor: not-allowed`.
- FR-014: The widget MUST be offline capable.
- FR-015: The `animate` CSS class MUST be applied at the `Icon` component level, enabling animation for both default SVG stars and custom icon types.
- FR-016: Structure preview star count MUST be capped at 50 to prevent Studio Pro performance issues with large `maximumStars` values.

## Props Reference

| Name | Type | Default | Caption | Description |
|------|------|---------|---------|-------------|
| `rateAttribute` | `EditableValue<Big>` | — | Rating attribute | Required writable attribute storing the selected rating value. Integer, Long, or Decimal. |
| `emptyIcon` | `DynamicValue<WebIcon>?` | — | Empty icon | Custom icon for unselected stars. Falls back to built-in outline SVG star when unavailable. |
| `icon` | `DynamicValue<WebIcon>?` | — | Filled icon | Custom icon for selected stars. Falls back to built-in filled SVG star when unavailable. |
| `maximumStars` | `number` | `5` | Maximum stars | Number of star items rendered. Must be greater than 0. |
| `animation` | `boolean` | `true` | Animation | Enables the stretch-bounce hover animation on stars. |
| `onChange` | `ActionValue?` | — | On change | Optional Mendix action fired after a rating is selected. |

## Changelog

**v3.2.2 (2026-02-10):** License documentation update only.

**v3.2.1 (2023-09-27):** Bundle size reduction.

**v3.1.2 (2023-05-23):** Replaced glyphicons with internal SVG star icons; widget no longer depends on Mendix built-in icon font.

**v3.1.1 (2022-04-01):** Content Security Policy (CSP) compliance fix.

**v2.0.0 (2021-05-10):** Major refactor introducing ARIA accessibility (radiogroup/radio roles), keyboard navigation (arrow keys, Space, Enter), the stretch-bounce animation, and custom icon support.

## Open Questions
> Could not be determined from source code alone — requires human review
- [ ] Is there a maximum value for `maximumStars` at runtime (beyond the 50-star design-time cap)?
- [ ] Does the Decimal type for `rateAttribute` enable fractional display (e.g., half-star rendering), or is display always whole-star increments?
- [ ] Is the `onChange` action documented as firing before or after `setValue`? The source confirms `setValue` then `execute()` order, but this has implications for microflow triggers that read the attribute immediately.
