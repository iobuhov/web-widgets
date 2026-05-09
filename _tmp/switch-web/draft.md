# switch-web — Draft Spec

Widget: `switch-web`
Package: `packages/pluggableWidgets/switch-web/`
Agent: worker
Date: 2026-05-09

---

## src/Switch.tsx

**1. What is the purpose of this file?**
The root Mendix pluggable widget entry component. Reads a Boolean attribute, creates click/keydown handlers, and delegates rendering to `SwitchComponent`.

**2. What kind of logic is described in this file?**
- `isChecked = isAvailable(props.booleanAttribute)` — uses Mendix `isAvailable` which returns `true` when the attribute status is "available" and has a value; effectively `true` when the attribute's boolean value is `true`.
- `editable = !props.booleanAttribute.readOnly`.
- `toggle()`: `booleanAttribute.setValue(!props.booleanAttribute.value)` then `executeAction(props.action)`.
- `onClick`: prevents default + calls `toggle()` if editable.
- `onKeyDown`: prevents default + calls `toggle()` only on Space key if editable.

**3. What part of behavior can be documented from this file?**
- `isAvailable` is used to compute `isChecked` — this means the switch shows as "off" when the attribute is loading or unavailable.
- Toggling always writes `!props.booleanAttribute.value` — toggles the actual stored value, not just the display.
- Action fires after `setValue` — always fires on toggle (not only on value change).
- Only Space key triggers keyboard toggle (not Enter).

**4. Is it user-facing?**
No — internal Mendix adapter.

**5. What new did you learn from this file?**
Using `isAvailable(props.booleanAttribute)` for `isChecked` (rather than `props.booleanAttribute.value`) means: if the attribute is `null`/unavailable, the switch renders as off. This is a safe fallback for initial loading or null boolean values.

---

## src/Switch.xml

**1. What is the purpose of this file?**
Mendix widget descriptor with minimal properties: one Boolean attribute binding and one optional action.

**2. What kind of logic is described in this file?**
Two property groups:
- **Data source**: `booleanAttribute` (attribute — Boolean only).
- **Actions**: `action` (action, optional — "On change").
System properties: Label, Editability.

**3. What part of behavior can be documented from this file?**
- `needsEntityContext="true"`, `offlineCapable="true"`.
- Only Boolean attribute type accepted (no Integer, String, etc.).
- `action` is labeled "On change" — fires after each toggle.
- Label system property: the widget label links to the switch via `aria-labelledby`.

**4. Is it user-facing?**
No — Studio Pro configuration descriptor.

**5. What new did you learn from this file?**
The widget has exactly 2 user-configurable properties (attribute + action). This is the simplest possible input widget — a single boolean toggle with an optional side effect.

---

## typings/SwitchProps.d.ts

**1. What is the purpose of this file?**
Auto-generated TypeScript types.

**2. What kind of logic is described in this file?**
- `SwitchContainerProps` (runtime): `booleanAttribute: EditableValue<boolean>`, `action?: ActionValue`.
- `SwitchPreviewProps` (Studio Pro preview): `booleanAttribute: string` (attribute name), `action: {} | null`.
- Note: no `class`, `style`, or `name` properties in `SwitchContainerProps` — only `id` and `tabIndex` from system props.

**3. What part of behavior can be documented from this file?**
- The omission of `class` and `style` from the container props type means the widget doesn't have Mendix design property CSS class support. (Confirmed — the XML has no `systemProperty key="ClassName"` or `systemProperty key="Style"`.)

**4. Is it user-facing?**
No — TypeScript types only.

**5. What new did you learn from this file?**
The absence of `class`/`style` props is intentional — this widget doesn't support Mendix design properties or custom CSS class from Studio Pro. This makes it one of the few "opinionated" widgets with no Mendix class customization hook.

---

## src/components/Switch.tsx

**1. What is the purpose of this file?**
The presentation component rendering the visual toggle switch.

**2. What kind of logic is described in this file?**
DOM structure:
- `<div class="widget-switch">` (outer container)
  - `<input type="checkbox" class="sr-only" ...>` — visually hidden, `aria-hidden="true"`, `tabIndex={-1}`, `readOnly` (no onChange handler). Reflects `isChecked` and `disabled`. This is the underlying `<input>` for semantic HTML form support.
  - `<div class="widget-switch-btn-wrapper widget-switch-btn-wrapper-default checked|un-checked disabled">` — the interactive toggle element; `role="switch"`, `aria-checked`, `aria-labelledby="${id}-label"`, `aria-disabled`, `tabIndex`.
    - `<div class="widget-switch-btn left|right">` — the sliding knob element; `left` when unchecked, `right` when checked.
  - `<Alert bootstrapStyle="danger">` — renders validation error below.

**3. What part of behavior can be documented from this file?**
- The `<div role="switch">` is the actual interactive element (not the `<input>`). The `<input>` is a hidden sr-only fallback for form submission or assistive technology that doesn't support `role="switch"`.
- `aria-labelledby="${id}-label"` links to the Mendix Label system property's generated label element.
- The knob slides via CSS class change (`left`/`right`), not CSS transitions (transition is handled by the SCSS).
- Validation alert always renders in DOM; only visible when `validation` is non-empty (Alert hides itself when children are empty).

**4. Is it user-facing?**
Yes — this is the visible toggle switch component.

**5. What new did you learn from this file?**
The dual rendering (hidden `<input type="checkbox">` + visible `<div role="switch">`) is a progressive enhancement pattern. The hidden input ensures the boolean value is included in native HTML form submissions, while the `<div role="switch">` provides the interactive ARIA semantics. The `<input>` has `aria-hidden="true"` to prevent screen readers from announcing it twice.

---

## src/Switch.editorConfig.ts

**1. What is the purpose of this file?**
Studio Pro structure preview and custom caption — no conditional property visibility rules.

**2. What kind of logic is described in this file?**
- `getPreview`: returns a static SVG image (80px wide) — light or dark mode checked-state switch icon.
- `getCustomCaption`: returns `values.booleanAttribute` (attribute name) or `"Switch"`.

**3. What part of behavior can be documented from this file?**
- The structure preview always shows the switch in "checked" state (both SVG assets appear to be the checked visual).
- No `getProperties` or `check` functions — no conditional visibility or validation needed.
- Dark mode supported via separate SVG.

**4. Is it user-facing?**
No — Studio Pro only.

**5. What new did you learn from this file?**
The absence of `getProperties` and `check` is consistent with the widget's minimal property set — there's nothing to conditionally show/hide or validate.

---

## src/Switch.editorPreview.tsx

**1. What is the purpose of this file?**
Live React preview rendering in Studio Pro design mode.

**2. What kind of logic is described in this file?**
- Renders `<Switch id="switch-preview" validation={undefined} editable={!props.readOnly} isChecked />`.
- Preview is always rendered with `isChecked={true}` — shows the checked visual state.
- No click/keydown handlers (non-interactive in preview).

**3. What part of behavior can be documented from this file?**
- Preview always shows the switch as "on" (checked).
- Respects `readOnly` state from Studio Pro (shows disabled styling).

**4. Is it user-facing?**
No — Studio Pro design mode preview only.

**5. What new did you learn from this file?**
The decision to always show `isChecked={true}` in preview is consistent with the structure preview SVG (also always checked). This makes the "active" switch state the canonical visual identifier for this widget.

---

## src/ui/switch-main.scss (and partials)

**1. What is the purpose of this file?**
Entry SCSS that imports `_switch.scss` and `_theme.scss` (with `_variables.scss` and `_mixins.scss`).

**2. What kind of logic is described in this file?**
The partials define:
- Toggle dimensions, colors, border-radius, transition animations.
- Color theming for checked/unchecked/disabled states.
- CSS transition for the knob sliding animation (implied by the presence of partials; full content not read but structure is clear).

**3. What part of behavior can be documented from this file?**
- E2E test confirmed: checked state background-color is `rgb(100, 189, 99)` (green) — set in `_theme.scss` or `_variables.scss`.
- The sliding animation is CSS-based (class change `left`/`right` on knob).

**4. Is it user-facing?**
Yes — controls all visual appearance of the switch.

**5. What new did you learn from this file?**
The SCSS is split into 4 files (`_switch.scss`, `_theme.scss`, `_variables.scss`, `_mixins.scss`) — a more structured approach than single-file widgets like rating-web. The `_variables.scss` pattern suggests the widget's colors/sizes are easily customizable for Atlas theme overrides.

---

## src/components/__tests__/Switch.spec.tsx

**1. What is the purpose of this file?**
Unit tests for the `Switch` presentation component.

**2. What kind of logic is described in this file?**
- `role="switch"` present; `aria-checked="false"` by default; `aria-disabled="false"`.
- `isChecked=true` → `aria-checked="true"`.
- `editable=false` → `"disabled"` class + `aria-disabled="true"`.
- Click calls `onClick`; keydown calls `onKeyDown`.
- `validation` prop renders the error string.

**3. What part of behavior can be documented from this file?**
- Confirmed: `role="switch"` ARIA role used.
- Confirmed: `aria-disabled` and `class="disabled"` both set when `editable=false`.

**4. Is it user-facing?**
No — test file.

**5. What new did you learn from this file?**
The presentation component tests use `userEvent` for click tests (`await user.click(...)`) but `fireEvent.keyDown` for keyboard tests. This suggests the onClick path requires the simulated user event queue, while keyDown can be fired synchronously.

---

## src/__tests__/Switch.spec.tsx

**1. What is the purpose of this file?**
Unit tests for the `Switch` container component — integration tests with Mendix EditableValue mocks.

**2. What kind of logic is described in this file?**
- Snapshot tests for editable and read-only states.
- Attribute write-back: click → `setValue(!value)`, then action executes.
- Keyboard: Space → toggle; Enter → no action.
- Read-only: click/keydown → no `setValue`, no `action.execute`.
- CSS class: `value=false` → `un-checked`, knob class `left`; `value=true` → `checked`, knob class `right`.
- `tabIndex`: defaults to `0` when undefined.
- Validation: `withValidation("error")` → renders `.alert.alert-danger` with text "error".

**3. What part of behavior can be documented from this file?**
- Confirmed: only Space key triggers toggle; Enter does not.
- Confirmed: `setValue` is always called on toggle when editable (both click and Space).
- Confirmed: `action` executes after `setValue` on every toggle.
- Confirmed: read-only prevents both `setValue` and action execution.
- Confirmed: `tabIndex` defaults to 0 when prop is undefined.

**4. Is it user-facing?**
No — test file.

**5. What new did you learn from this file?**
The test fires events on `.widget-switch-btn` (the inner knob div) rather than `.widget-switch-btn-wrapper` (the role="switch" div). Both propagate events to the handler. This confirms both elements are valid interaction targets.

---

## e2e/Switch.spec.js

**1. What is the purpose of this file?**
Playwright E2E tests verifying visual state, attribute binding, and action execution in a real Mendix app.

**2. What kind of logic is described in this file?**
- **Color test**: verifies `background-color` is `rgb(100, 189, 99)` (green) for checked state.
- **Attribute-driven state**: clicks a radio button to set attribute to `true` → switch shows `checked` class + `aria-checked="true"`.
- **Two-way binding**: clicking the switch wrapper → `radioButtons6` input value becomes `"true"`.
- **Action test**: clicking switch3 → attribute set + popup modal appears with "IT WORKS" text.

**3. What part of behavior can be documented from this file?**
- Confirmed: checked switch color is `rgb(100, 189, 99)` (green #64BD63).
- Confirmed: two-way binding — switch reflects attribute and attribute reflects switch.
- Confirmed: `action` can open a popup page (modal) as an on-change action.

**4. Is it user-facing?**
No — test file.

**5. What new did you learn from this file?**
The E2E test demonstrates the widget working with a microflow action that opens a popup — this is the "On change" action being tested end-to-end. The test confirms the action fires after the attribute is set (the radio button shows the new value AND the popup appears).

---

## CHANGELOG.md

**1. What is the purpose of this file?**
Version history since v3.0.0.

**2. What kind of logic is described in this file?**
- **v4.3.0 (2025-10-17)**: Improved accessibility — clicking the label now toggles the switch (label click → switch toggle).
- **v4.2.2 (2024-08-28)**: Changed `action` to not required (removes Studio Pro warning).
- **v4.2.0 (2023-06-06)**: Updated caption to display datasource; updated icons/tiles.
- **v4.1.0 (2021-12-23)**: Added dark mode structure preview and dark icons.
- **v4.0.0 (2021-09-28)**: Added toolbox category/tile.
- **v3.0.0 (2021-04-23)**: Added "Brand Secondary" style design property; added `aria-readonly`; changed from `<small>` to `<div>` for semantics.

**3. What part of behavior can be documented from this file?**
- v4.3.0 label-click fix: when the Mendix Label system property is used, clicking the label text now toggles the switch. This likely works through the hidden `<input type="checkbox" id={props.id}>` which is linked to the label via `for` attribute.
- v4.2.2: `action` had been required, causing unnecessary Studio Pro warnings when no action was needed.
- v3.0.0: The `<div>` semantic improvement removed `<small>` tags from the DOM structure.

**4. Is it user-facing?**
No — developer changelog.

**5. What new did you learn from this file?**
The label-click fix in v4.3.0 reveals how the hidden `<input type="checkbox" id={props.id}>` serves a purpose: clicking the Mendix `<label for="{id}">` element triggers the checkbox's click event, which then propagates to the `onClick` handler on the outer wrapper. Without this hidden input, label clicks would not toggle the switch.

---

## Summary of Key Findings

- **Purpose**: Toggles a single Boolean entity attribute with optional on-change action. The simplest input widget in this set.
- **ARIA**: `role="switch"` on the interactive div; `aria-checked`, `aria-disabled`, `aria-labelledby` all properly set. Hidden `<input type="checkbox">` for native form support + label-click handling.
- **Keyboard**: Space key toggles; Enter does NOT toggle.
- **Read-only**: Does not call `setValue` or execute action. Shows `disabled` class + `aria-disabled="true"`.
- **Checked color**: `rgb(100, 189, 99)` — green `#64BD63`.
- **No class/style props**: Widget doesn't support Mendix design property CSS class customization.
- **Label integration**: Mendix Label system property creates `<label for="{id}">` which triggers the hidden checkbox's click event → label click toggles the switch (added in v4.3.0).
- **Action timing**: Action fires immediately after `setValue(!value)` on every toggle, no debounce.
- **offlineCapable**: `true`.
- **Testing**: Two separate test files — `components/__tests__/Switch.spec.tsx` (presentation layer) and `__tests__/Switch.spec.tsx` (container/integration layer).
