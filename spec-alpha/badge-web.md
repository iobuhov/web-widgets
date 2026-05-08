# Badge

## Purpose

The Badge widget renders a styled inline display element — either a pill-shaped badge or a rectangular label — bound to a dynamic Mendix text expression. It is intended for use cases where a short status indicator, count, or category label must be shown contextually on a page, with optional click interactivity. When a click action is configured and executable, the element becomes keyboard- and mouse-interactive, following the ARIA button pattern. When no action is configured, it renders as a purely visual display element.

## User Scenarios

### [P1] Display a reactive badge value bound to an attribute

**Given** the widget is placed on a Mendix page with `type` set to `"badge"` and `value` bound to a string or numeric attribute  
**When** the page loads and the attribute has a value  
**Then** the badge renders as a pill-shaped `<span>` displaying the current attribute value; when the attribute changes, the display updates immediately without a page reload

#### Edge Cases

- When `value` is undefined, null, or in a loading state, the badge renders an empty string (no error, no `isAvailable` guard required).
- Supported attribute types (e2e-confirmed): string, integer, long, decimal, and enum. Enum values render as their Mendix caption text, not the raw key.
- Long integers (e.g. 123456789012345678) render without truncation.

---

### [P2] Use label variant for rectangular display

**Given** the widget is configured with `type` set to `"label"`  
**When** the page renders  
**Then** the element renders with the `.label` CSS class (rectangular, slightly rounded corners) instead of the `.badge` CSS class (pill-shaped); all other behavior — reactivity, click action, data type support — is identical to the badge variant

#### Edge Cases

- Both badge and label variants can be bound to the same entity attribute and update independently when that attribute changes.
- The visual distinction (borderRadius 22 for badge, 8 for label in Studio Pro structure preview) is a design hint only — enforced by Atlas theme CSS, not by any runtime prop validation.

---

### [P3] Trigger a Mendix action on click

**Given** `onClick` is configured to invoke a nanoflow and the action is currently executable (`canExecute = true`)  
**When** the user clicks the badge (or presses Enter or Space while the badge has keyboard focus)  
**Then** the nanoflow is invoked and receives the widget's data context; both mouse and keyboard triggers produce identical behavior

#### Edge Cases

- When `onClick` is configured but `canExecute = false` (e.g. restricted by access rules), the badge renders as non-interactive: no handlers, no `widget-badge-clickable` class, no `role="button"`, and no `tabIndex`.
- When `onClick` is not configured at all, the badge renders as a pure display element with no interactive affordances.
- There is no visual disabled state when `canExecute` transitions to `false` — the change is reflected by removing interactivity silently, with no CSS disabled indicator applied by default.

---

## Functional Requirements

- FR-001: The system MUST render a `<span>` element with the base CSS class `widget-badge` always present.
- FR-002: The system MUST apply the CSS class `badge` when `type` is `"badge"` and the class `label` when `type` is `"label"`.
- FR-003: The system MUST add the CSS class `widget-badge-clickable` when an `onClick` handler is present; this class MUST NOT be added when only `onKeyDown` is present.
- FR-004: The system MUST set `role="button"` on the span when EITHER `onClick` OR `onKeyDown` is defined — either handler alone is sufficient.
- FR-005: The system MUST set `tabIndex` to `0` by default when the badge is clickable and no explicit `tabIndex` is configured; when the badge is not clickable, `tabIndex` MUST NOT be set.
- FR-006: The system MUST render `value` using optional-chaining with an empty-string fallback (`props.value?.value ?? ""`); values in a loading or unavailable state MUST render as an empty string without error.
- FR-007: The badge MUST become interactive (handlers and `role="button"` applied) only when `onClick` is both configured AND currently executable (`canExecute = true`).
- FR-008: The system MUST execute the configured `onClick` action on both mouse click and keyboard Enter/Space events, producing identical behavior for both input methods.
- FR-009: Users MUST be able to configure `value` as a Mendix `textTemplate` supporting dynamic attribute references, static text, and i18n tokens.
- FR-010: The widget MUST support attribute types: string, integer, long, decimal, and enum (enum rendered as its Mendix caption).
- FR-011: The widget MUST function in offline-enabled Mendix applications (`offlineCapable = true`).
- FR-012: The widget MUST NOT render with `role="button"`, `widget-badge-clickable`, or `tabIndex` when `onClick` is not configured.

## Props Reference

| Name | Type | Default | Caption | Description |
|------|------|---------|---------|-------------|
| `type` | enum (`"badge"` \| `"label"`) | `"badge"` | Type | Display variant. `"badge"` renders as a pill-shaped span (CSS class `.badge`); `"label"` renders as a slightly rounded rectangle (CSS class `.label`). Required — always has a value at runtime. |
| `value` | textTemplate | `"Badge"` | Value | Text displayed inside the badge. Supports dynamic expressions, attribute references, and i18n tokens. Renders empty string when undefined, null, or unavailable — no `isAvailable` guard is applied. |
| `onClick` | action | _(none)_ | On click | Mendix action executed when the badge is clicked or activated via keyboard (Enter/Space). Optional. When not configured, badge is a non-interactive display element. When configured, badge becomes interactive only if `canExecute = true`. |
| `tabIndex` | number | _(system)_ | Tab index | Keyboard tab order override. When not set and badge is clickable, defaults to `0`. When badge is not clickable, tab index is not applied. |

## Changelog

### [3.2.3] - 2026-02-09
- Added: License file and open-source dependency readme.

### [3.2.2] - 2024-08-28
- Changed: `onClick` action changed from required to optional (`required="false"`), eliminating Studio Pro warnings when no click action is configured.

### [3.2.1] - 2023-09-27
- Changed: Removed redundant code for improved load time.

### [3.2.0] - 2023-06-05
- Changed: Page explorer caption now shows the badge value for quick identification.
- Changed: Updated light and dark icons and tiles.
- Changed: Structure preview colors updated for dark and light mode support.

## Open Questions

> Could not be determined from source code alone — requires human review

- [ ] Are microflow, open page, open modal popup, and close page action types supported for `onClick` in addition to nanoflows? Only nanoflow is e2e-confirmed; all action types are declared in the XML schema but runtime behavior for non-nanoflow actions is untested.
- [ ] Is the absence of a visual disabled state (when `canExecute` transitions to `false`) intentional? The widget silently removes interactivity with no CSS indication — Atlas theme consumers may need custom styling to communicate a disabled badge state.
- [ ] Is the `?? ""` empty-string fallback for `value` (rather than the `isAvailable` guard used in badge-button-web) an intentional API contract, or an implementation divergence that may be unified in a future release?
