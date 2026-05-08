# Color Picker (color-picker-web)

## Purpose

The Color Picker widget allows users to select a color value and persist it to a Mendix string attribute. It supports three display modes (popover button, text input with popup, and always-visible inline picker), eleven visual picker types via the `react-color` library, and three color output formats (hex, RGB, RGBA). The widget is offline-capable and requires an entity context.

## User Scenarios

### [P1] Open and select a color via popover mode
**Given** a Color Picker in popover mode on a Mendix page  
**When** the user clicks the color swatch button  
**Then** a color picker overlay opens; when the user selects a color, the string attribute is updated in real time; clicking outside the picker (or the cover overlay) closes it  

#### Edge Cases
- Inside a Mendix modal dialog, the picker popover switches to `position: fixed` to avoid being clipped by the modal scroll container
- If the widget is disabled (attribute is read-only), the button has a `disabled` CSS class; interaction is blocked by a semi-transparent overlay (`z-index: 50`)
- The `onClickAction` is optional; if not configured no action fires

### [P2] Type a color value in input mode
**Given** a Color Picker in input mode  
**When** the user types a color string in the text field (e.g., `#4caf50`, `rgb(42,94,210)`)  
**Then** the string attribute is updated; pressing `ArrowDown` opens the color picker dropdown below the input  

#### Edge Cases
- RGB values must have no spaces: `rgb(42,94,210)` is valid; `rgb(42, 94, 210)` fails validation
- RGBA alpha must be between 0 and 1 (e.g., `0.49`); no spaces allowed
- HEX accepts 3-digit (`#0d0`) or 6-digit (`#d0d0d0`) formats, with or without the `#` prefix
- If the entered value does not match the configured format, an error alert is shown below the picker; the `:colors:` token in `invalidFormatMessage` is replaced with an example of the correct format

### [P3] Use the inline picker
**Given** a Color Picker in inline mode  
**When** the page is rendered  
**Then** the full color picker is always visible on the page without requiring a click to open  

#### Edge Cases
- In inline mode, `invalidFormatMessage` is hidden (users cannot type invalid values via inline pickers)
- When disabled in inline mode, an overlay div blocks all interaction

### [P4] Select from preset colors (defaultColors)
**Given** a Color Picker with `advanced=true` and `defaultColors` configured  
**When** the user opens a picker type that supports custom swatches (block, sketch, circle, compact, twitter)  
**Then** the configured preset colors appear as clickable swatches in the picker  

#### Edge Cases
- `defaultColors` is ignored for picker types that do not support it: hue, chrome, github, material, swatches, slider
- Validation of `defaultColors` entries runs against the configured `format`; invalid entries log errors

### [P5] Color attribute update with onChange action
**Given** a Color Picker with an `onChange` action configured  
**When** the user selects a color  
**Then** the string attribute is updated immediately (on every change during drag); the `onChange` action fires after the user finishes selecting with a 500 ms debounce  

#### Edge Cases
- The attribute update is synchronous and real-time; the action is debounced
- If the user starts a new selection while the debounce is pending, the previous debounced action call is cancelled
- Known issue (unreleased fix): `onChange` action may not trigger in some edge cases

## Functional Requirements

- FR-001: The widget MUST bind to a Mendix string attribute and update it synchronously on every color change via `EditableValue.setValue()`.
- FR-002: The widget MUST disable all interaction (button disabled class + inline overlay) when `colorAttribute.readOnly` is `true`.
- FR-003: The `onChange` action MUST be debounced 500 ms and MUST cancel any pending call when a new color selection begins.
- FR-004: The widget MUST validate the stored color value against the configured `format` on every value change and display an error alert when the format is invalid.
- FR-005: For invalid format errors, the `:colors:` token in `invalidFormatMessage` MUST be replaced with a format-specific example string.
- FR-006: RGB format validation MUST reject values with spaces (e.g., `rgb(42, 94, 210)` is invalid).
- FR-007: The widget MUST support 11 picker types: block, chrome, circle, compact, github, hue, material, sketch, slider, swatches, twitter.
- FR-008: For picker types github, block, and twitter, the picker triangle decoration MUST be hidden (`triangle: "hide"`).
- FR-009: The `advanced` prop MUST gate the type, defaultColors, and format properties; when `advanced=false`, these properties MUST be hidden in Studio Pro.
- FR-010: `defaultColors` MUST only be visible and applied for picker types: block, sketch, circle, compact, twitter.
- FR-011: `invalidFormatMessage` MUST be hidden in Studio Pro when `mode=inline`.
- FR-012: In popover and input modes, a fixed-position full-viewport cover div MUST be rendered while the picker is open to enable "click outside to close" behavior.
- FR-013: Inside a Mendix modal dialog, the popover MUST switch to `position: fixed` to prevent clipping by the modal scroll container.
- FR-014: Pressing `ArrowDown` in the text input (input mode) MUST open the color picker dropdown.
- FR-015: The widget MUST be compatible with both the Mendix classic (Dojo) and React clients.
- FR-016: `onChange` action MUST be optional; the widget MUST NOT require it to be configured.

## Props Reference

| Name | Type | Default | Caption | Description |
|------|------|---------|---------|-------------|
| `colorAttribute` | `EditableValue<string>` | — | Color attribute | String attribute storing the color value. Supports hex, rgb, rgba formats. Named color strings (e.g., "red") are not supported. |
| `advanced` | `boolean` | `false` | Advanced options | When true, exposes the type, defaultColors, and format properties in Studio Pro. |
| `mode` | `"popover"` \| `"input"` \| `"inline"` | `"popover"` | Display mode | popover: color swatch button opens picker overlay. input: text input with button opens picker. inline: picker always visible. |
| `type` | `TypeEnum` | `"sketch"` | Picker type | One of 11 visual styles: block, chrome, circle, compact, github, hue, material, sketch, slider, swatches, twitter. |
| `format` | `"hex"` \| `"rgb"` \| `"rgba"` | `"hex"` | Color format | Output format for the stored color string. |
| `defaultColors` | `DefaultColorsType[]` | — | Default colors | List of preset color strings shown as swatches. Supported by: block, sketch, circle, compact, twitter. |
| `invalidFormatMessage` | `DynamicValue<string>` | — | Invalid format message | Message shown when the color attribute contains an invalid format. Use `:colors:` token as a placeholder for a format example. Hidden in inline mode. |
| `onChange` | `ActionValue` | — | On change | Optional action executed 500 ms after the user finishes selecting a color. |

## Changelog

**2.1.5** (2026-03-06) — Fixed context-switch bug: in "listen to data view" setup, stale attribute values from the previous context are no longer applied when the context changes.

**2.1.3** (2025-03-21) — Fixed Color Picker not working in the Mendix React client.

**2.1.2** — `onChange` action changed from required to optional.

**2.0.0** — Converted from legacy to pluggable widget.

## Open Questions

> Could not be determined from source code alone — requires human review

- [ ] The HEX format e2e test is marked `test.fixme` — what is the specific failure, and is it a widget bug or a test environment issue?
- [ ] The unreleased changelog entry notes "`onChange` not triggering in some cases" — which specific scenarios are affected?
- [ ] Does the `material-picker` hardcoded width of 130px cause layout regressions in any standard Mendix layout containers?
- [ ] The color swatch button (popover/input mode) has no `aria-label` — is this an acknowledged accessibility gap with a planned fix?
