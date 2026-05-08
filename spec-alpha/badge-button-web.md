# BadgeButton

## Purpose

The BadgeButton widget renders a Bootstrap-styled clickable button with an embedded badge overlay. It is intended for use cases where a count, status label, or short text indicator must be visually attached to a call-to-action button — for example, a "Messages" button displaying an unread count. Both the button caption and the badge value are bound to dynamic Mendix expressions, so they update reactively when the underlying data changes without a page reload.

## User Scenarios

### [P1] Display a reactive badge counter on a button

**Given** the widget is placed on a Mendix page and `label` is set to "Messages" and `value` is bound to an integer attribute representing unread message count  
**When** the page loads and the attribute has value 5  
**Then** the button renders as "Messages" with a badge showing "5"; when the attribute changes to 6 the badge updates immediately without a page reload

#### Edge Cases

- When `value` is an empty string or the attribute is in a loading/unavailable state, the badge span remains in the DOM but renders empty.
- When `label` is unset or empty, the button caption is empty; the widget renders without error.
- When `value` is a long integer (e.g. 2,147,483,647), Mendix locale formatting applies and the badge displays "2,147,483,647".

---

### [P2] Trigger a Mendix action on button click

**Given** `onClickEvent` is configured to call a microflow  
**When** the user clicks the button  
**Then** the microflow is invoked and receives the widget's data context (including the badge value)

#### Edge Cases

- When the action is currently executing (`isExecuting = true`), a second click is silently ignored — no double-trigger.
- When the action is not executable (`canExecute = false`, e.g. due to access restrictions), clicking is silently ignored.
- When `onClickEvent` is not configured, clicking the button is a silent no-op.
- Rapid sequential clicks will each trigger the action if the previous execution completes before the next click arrives — no debounce is applied.

---

### [P3] Apply a custom Bootstrap button color

**Given** a custom CSS class containing one of `btn-primary`, `btn-secondary`, `btn-success`, `btn-warning`, or `btn-danger` is set on the widget  
**When** the widget renders  
**Then** only the provided button color class is applied; the default `btn-primary` class is NOT added

#### Edge Cases

- When a class like `btn-custom-color` is provided (does not match the known Bootstrap variants), `btn-primary` is still applied in addition to the custom class.
- When no class is set, the default `btn btn-primary` styling applies.

---

## Functional Requirements

- FR-001: The system MUST render a `<button>` element with base CSS classes `widget-badge-button` and `btn` always present.
- FR-002: The system MUST render a `<span class="widget-badge-button-text">` child element containing the button label text.
- FR-003: The system MUST render a `<span class="badge">` child element containing the badge value text; this span MUST always be present in the DOM regardless of whether `value` is empty.
- FR-004: The system MUST apply the CSS class `btn-primary` by default when none of `btn-primary`, `btn-secondary`, `btn-success`, `btn-warning`, `btn-danger` is present in the configured class name.
- FR-005: The system MUST NOT add `btn-primary` when the configured class name already contains one of the five Bootstrap color variant classes listed in FR-004.
- FR-006: Users MUST be able to configure `label` and `value` as Mendix `textTemplate` expressions supporting dynamic attribute references and i18n tokens.
- FR-007: The system MUST execute the configured `onClickEvent` action when the button is clicked, subject to `canExecute` and `isExecuting` guards.
- FR-008: The system MUST NOT execute a concurrent action when `isExecuting` is `true`.
- FR-009: The widget MUST be usable without any data source context (`needsEntityContext` is false).
- FR-010: The widget MUST function in offline-enabled Mendix applications (`offlineCapable = true`).
- FR-011: The widget MUST NOT support Mendix's modern React client (`MODERN_CLIENT`); it targets the classic Dojo-based web client only.
- FR-012: When `label` or `value` has status other than `"available"`, or has an empty string value, the rendered text for that field MUST be an empty string.

## Props Reference

| Name | Type | Default | Caption | Description |
|------|------|---------|---------|-------------|
| `label` | textTemplate | "Button" | Label | Button caption text. Supports dynamic expressions and i18n tokens. Renders empty when unavailable or empty string. |
| `value` | textTemplate | _(none)_ | Badge value | Text displayed inside the badge span. Supports dynamic expressions and i18n tokens. Renders empty when unavailable or empty string. |
| `onClickEvent` | action | _(none)_ | On click | Mendix action executed when the button is clicked. Supports microflows, nanoflows, page navigation, modal popup, and close page. Optional — if not set, clicking is a no-op. |

## Changelog

### [3.3.0] - 2026-03-13
- Fixed: Custom button styles were not being applied correctly (regex-based detection of Bootstrap color variants now correctly suppresses `btn-primary`).

### [3.2.2] - 2026-02-09
- Added: License file and readme documenting open source dependencies.

### [3.2.0] - 2023-06-05
- Changed: Updated colors in the structure mode preview for dark and light modes.
- Changed: Page explorer caption now displays the label value.
- Changed: Updated light and dark icons and tiles.

## Open Questions

> Could not be determined from source code alone — requires human review

- [ ] Is the badge span intentionally always rendered (even when empty), or should it be conditionally hidden via CSS when `value` is empty? The current behavior keeps an empty `<span class="badge">` in the DOM which may affect layout.
- [ ] Is the conflation of "attribute unavailable" and "empty string value" in `isAvailable` an intentional design decision? A Mendix developer who deliberately sets `value = ""` gets the same rendered output as a loading state.
- [ ] What is the intended compatibility timeline for the modern Mendix React client? The widget is explicitly excluded via `MODERN_CLIENT` skip flags in e2e tests.
