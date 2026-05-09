# Switch

## Purpose

The Switch widget is a boolean toggle input control for Mendix web applications. It binds to a single Boolean entity attribute and flips its value on click or Space key press. The widget renders in either an iOS-styled (default) or Android-styled appearance, using a hidden `<input type="checkbox">` for form semantics combined with a styled `<div role="switch">` for the interactive element, following the W3C ARIA switch pattern. It is categorized under "Input elements" and requires an entity context.

## User Scenarios

### [P1] User toggles a boolean attribute
**Given** a Switch widget bound to a Boolean attribute in a Mendix data form  
**When** the user clicks the switch  
**Then** the attribute value is set to `!currentValue` (toggled), and the configured `onChange` action executes after the attribute is updated  

#### Edge Cases
- When the attribute is read-only, both click and keyboard interactions are silently blocked (no error or feedback).
- The `onChange` action is optional — the widget functions as a toggle without any action configured.

### [P2] Keyboard activation
**Given** the switch is focused via Tab navigation  
**When** the user presses Space  
**Then** the attribute is toggled and the `onChange` action executes (same as click)  

#### Edge Cases
- Only Space activates the toggle. Enter, Tab, and all other keys are explicitly ignored (per the W3C ARIA switch specification, which reserves Space for toggle activation).
- The switch is announced by screen readers as `role="switch"` with `aria-checked` reflecting the current boolean value.

### [P3] Read-only switch
**Given** a Switch widget whose bound attribute is read-only (e.g., due to a conditional editability rule)  
**When** the user clicks or presses Space  
**Then** no attribute change occurs and no action executes; `aria-disabled="true"` is set on the interactive element and the `.disabled` CSS class is applied  

## Functional Requirements

- FR-001: System MUST bind to a Boolean entity attribute; no other attribute types are supported.
- FR-002: System MUST toggle the attribute value to `!currentValue` on click or Space key press.
- FR-003: System MUST execute the optional `onChange` action after `setValue` completes; the action MUST NOT execute when the attribute is read-only.
- FR-004: System MUST render `role="switch"` on the interactive `<div>` element with `aria-checked` reflecting the current boolean value.
- FR-005: System MUST render `aria-disabled="true"` and CSS class `.disabled` when the attribute is read-only.
- FR-006: System MUST use a hidden `<input type="checkbox" aria-hidden="true">` alongside the styled div for native form semantics without double screen-reader announcement.
- FR-007: System MUST ignore all key events except Space for the toggle action.
- FR-008: System MUST display validation messages from the Mendix attribute validation system as a styled alert below the switch.
- FR-009: The widget MUST require entity context (`needsEntityContext="true"`) and support offline mode.

## Props Reference

| Name | Type | Default | Caption | Description |
|------|------|---------|---------|-------------|
| `booleanAttribute` | `EditableValue<boolean>` | — | Attribute | The Boolean entity attribute to read and write; required |
| `action` | `ActionValue` (optional) | — | On change | Mendix action executed after the attribute is toggled; fires after `setValue`, not before |

System properties supported: Name, Editability, Visibility, Label, TabIndex.

## Visual Behavior

Two platform-specific CSS styles are available:

| Style | Checked color | Unchecked appearance | Active (press) effect |
|-------|--------------|----------------------|-----------------------|
| iOS (default) | `rgb(100, 189, 99)` (green) | White background, gray border | Knob widens to 70% (tactile press simulation) |
| Android | `#6fbeb5` (teal) | Gray background | No knob widening |

The Android style is activated by adding the `.android` CSS class to the container. Both styles use CSS transitions for smooth animation. Disabled state applies 50% opacity (iOS) or light gray colors (Android).

## Changelog

- **v4.3.0 (2025-10)**: Clicking the associated label now toggles the switch (previously only clicking the switch itself worked).
- **v4.2.2 (2024-08)**: The `onChange` action is no longer required.
- **v3.0.0 (2021-04)**: Replaced `<input>` as the interactive element with `<div role="switch">` for ARIA compliance; added `aria-readonly` support.

## Open Questions

> Could not be determined from source code alone — requires human review
- [ ] Can the Android style (`.android` class) be applied via Mendix's CSS Class system property, or does it require custom styling?
- [ ] Is there a way to configure the checked-state color independently per widget instance, or is it always the SCSS variable default?
