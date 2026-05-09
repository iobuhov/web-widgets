# Draft: switch-web

Widget package: `packages/pluggableWidgets/switch-web`

---

## src/Switch.xml

**1. What is the purpose of this file?**
Widget descriptor XML defining the widget's identity, props schema, and Studio Pro categorization. Generates `SwitchProps.d.ts`.

**2. What kind of logic is described in this file?**
Declares: `booleanAttribute` (Boolean attribute type only — required); `action` (action, optional). System properties: Name, Editability, Visibility, Label, TabIndex. Widget is `needsEntityContext="true"`, `pluginWidget="true"`, `offlineCapable="true"`, categorized under "Input elements".

**3. What part of behavior can be documented from this file?**
- Accepts only Boolean attribute type — cannot be used with integer or string attributes.
- The `action` (onChange) is optional — the widget functions as a toggle without any action.
- Widget requires entity context — must be placed in a data container (form, list, etc.).
- Widget is offline capable.
- Categorized under "Input elements" (not "Display") — it is a read-write input control.

**4. Is it user-facing?**
Defines the developer-facing configuration interface in Studio/Studio Pro.

**5. What new did you learn from this file?**
The only property is `booleanAttribute` — the widget is intentionally minimal. Unlike progress-bar-web which has 3 value modes, the switch always binds to a single boolean entity attribute with no alternative data sourcing.

---

## src/Switch.tsx

**1. What is the purpose of this file?**
The root Mendix pluggable widget entry component. Handles the boolean attribute read/write and routes the onChange action to the internal `Switch` component.

**2. What kind of logic is described in this file?**
Reads `booleanAttribute.value` (boolean) and `booleanAttribute.readOnly`. Click handler: calls `booleanAttribute.setValue(!currentValue)` then `executeAction(props.action)`. Keyboard handler: fires only on Space key; same behavior as click. Guards both handlers with `!booleanAttribute.readOnly` check. Passes `booleanAttribute.validation` to the presentational component.

**3. What part of behavior can be documented from this file?**
- Toggle: the new value is always `!currentValue` — the switch flips the boolean on every interaction.
- The Space key triggers toggle (keyboard accessibility); other keys are ignored.
- Read-only state blocks both click and keyboard interactions.
- `executeAction(props.action)` fires after `setValue` — the action runs after the attribute is updated.
- Validation messages from Mendix's attribute validation are passed down for display.

**4. Is it user-facing?**
No — internal Mendix-to-component adapter.

**5. What new did you learn from this file?**
The keyboard handler explicitly checks for Space key only (not Enter). This is the ARIA switch pattern — ARIA switches respond to Space, not Enter. Enter is typically reserved for buttons (default form submission behavior).

---

## src/components/Switch.tsx

**1. What is the purpose of this file?**
The presentational component rendering the visual toggle switch UI with full accessibility attributes.

**2. What kind of logic is described in this file?**
Renders: a hidden `<input type="checkbox" className="sr-only">` (for form semantics); a `<div className="widget-switch-btn-wrapper" role="switch" aria-checked aria-disabled aria-labelledby tabIndex>` (the interactive element); a `<div className="widget-switch-btn">` (the toggle knob). Conditional CSS classes: `checked`/`un-checked`, `disabled`. Validation alert rendered below when `validationFeedback` is present.

**3. What part of behavior can be documented from this file?**
- ARIA role `"switch"` is used (not `"checkbox"`) — per W3C ARIA specification for toggle switches.
- `aria-checked` reflects the boolean state.
- `aria-disabled` reflects the read-only state (inverted: `editable=true` → `aria-disabled=false`).
- `aria-labelledby` links to the associated label (Mendix's system Label property).
- A hidden `<input type="checkbox" aria-hidden="true">` provides form semantics without visual presence.
- CSS classes: `.widget-switch-btn-wrapper` always; `.checked` when true, `.un-checked` when false; `.disabled` when not editable.
- Validation error messages appear as a styled alert below the switch.

**4. Is it user-facing?**
Yes — this is the visible, interactive switch users interact with.

**5. What new did you learn from this file?**
The dual-element approach (hidden checkbox + styled div) is a common accessible toggle pattern: the hidden checkbox provides native form behavior and screen reader compatibility, while the styled div provides the visual representation. The checkbox is `aria-hidden="true"` to prevent double-announcement by screen readers.

---

## src/ui/_mixins.scss

**1. What is the purpose of this file?**
SCSS mixins defining the full visual appearance of iOS and Android platform-styled switches.

**2. What kind of logic is described in this file?**
`ios` mixin: Rounded rect wrapper (50×30 px, 20px border-radius). White background, gray border by default. Circular toggle knob (30px diameter) positioned left (unchecked) or right (checked) via `left: 0` / `left: 50%`. Smooth transitions on `border`, `box-shadow`, `transform`, `background-color`. Active state: knob widens to 70% width. Checked state: green background (`rgb(100, 189, 99)`) with inset shadow. Disabled: 50% opacity. `android` mixin: Smaller (44×20 px). Gray background. White toggle knob (26×26 px) with shadow offset for 3D appearance. Checked: teal background (`#6fbeb5`), darker toggle (`#179588`).

**3. What part of behavior can be documented from this file?**
- iOS style: checked state background is `rgb(100, 189, 99)` (green) — confirmed by e2e tests.
- Android style: checked state background is `#6fbeb5` (teal).
- iOS active state: the knob stretches to 70% width, creating a tactile press feedback effect.
- Both styles use CSS transitions for smooth animation — no JavaScript animation.
- Disabled state uses 50% opacity (iOS) or light gray colors (Android).
- iOS style is applied by default; Android style requires the `.android` CSS class on the container.

**4. Is it user-facing?**
Yes — all visual appearance including colors, dimensions, and animations is defined here.

**5. What new did you learn from this file?**
The iOS active-state knob-widening animation (`width: 70%; border-radius: 45%`) is a pure CSS trick that simulates the physical "push" feel of a real iOS toggle switch. This tactile feedback is specific to the iOS style and absent from the Android style.

---

## src/ui/_theme.scss

**1. What is the purpose of this file?**
Applies brand colors to the switch variants by invoking the color-application parts of the iOS and Android mixins.

**2. What kind of logic is described in this file?**
Applies `bootstrap-style-ios` (with `$default-ios-color`) for the default iOS checked appearance. Applies `bootstrap-style-android` (with `$default-android-color`) for the `.android` class checked appearance, plus sets the toggle button background to the brand color when checked.

**3. What part of behavior can be documented from this file?**
- Default (iOS) checked color: `rgb(100, 189, 99)` from `$default-ios-color`.
- Android checked color: `#6fbeb5` from `$default-android-color`.
- The `bootstrap-style-ios` mixin applies `background-color`, `border-color`, and an inset `box-shadow` — creating the depth effect for the checked state.

**4. Is it user-facing?**
Yes — defines the checked-state colors visible to users.

**5. What new did you learn from this file?**
The "theme" file is specifically about checked-state brand colors — all other appearance (size, shape, unchecked state) is in the mixins. This separation allows the brand color to be customized independently from the structural styles.

---

## src/ui/_variables.scss

**1. What is the purpose of this file?**
SCSS variable definitions for platform-specific brand colors.

**2. What kind of logic is described in this file?**
`$default-ios-color: rgb(100, 189, 99)` and `$default-android-color: #6fbeb5`. These are the only two variable declarations.

**3. What part of behavior can be documented from this file?**
- iOS default toggle color: `rgb(100, 189, 99)` (green).
- Android default toggle color: `#6fbeb5` (teal).
- These values can be overridden by Mendix theme customization (SCSS variable override).

**4. Is it user-facing?**
No — SCSS variable definitions only.

**5. What new did you learn from this file?**
The iOS default green (`rgb(100, 189, 99)`) matches Apple's UISwitch tint color in iOS 7+ — this is intentional visual consistency with native iOS UI conventions.

---

## typings/SwitchProps.d.ts

**1. What is the purpose of this file?**
Auto-generated TypeScript typings from `Switch.xml`. Defines `SwitchContainerProps` (runtime) and `SwitchPreviewProps` (Studio design-mode).

**2. What kind of logic is described in this file?**
`SwitchContainerProps`: `booleanAttribute: EditableValue<boolean>`, `action?: ActionValue`. `SwitchPreviewProps`: `booleanAttribute: string` (attribute name), `action: {} | null`, `renderMode`, `readOnly`.

**3. What part of behavior can be documented from this file?**
- `booleanAttribute` is `EditableValue<boolean>` — two-way binding with the Mendix boolean attribute.
- `action` is typed as `ActionValue | undefined` at runtime — optional action after toggle.
- In preview mode, `booleanAttribute` is a plain `string` (the attribute name, not the value).

**4. Is it user-facing?**
No — internal type declarations.

**5. What new did you learn from this file?**
The minimal props interface (just `booleanAttribute` and `action`) confirms this is the simplest input widget in the set — no dynamic values, no expression types, no datasource. One boolean attribute, one optional action.

---

## src/__tests__/Switch.spec.tsx

**1. What is the purpose of this file?**
Unit tests for the root `Switch` container component using Jest and React Testing Library.

**2. What kind of logic is described in this file?**
Tests: snapshot for editable state; snapshot for read-only state; `setValue` called with `!currentValue` on click; action executed after `setValue`; Space key triggers toggle; other keys do not trigger toggle; read-only state: `setValue` NOT called; read-only state: action NOT executed; `tabIndex` defaults to 0 when not provided.

**3. What part of behavior can be documented from this file?**
- Toggle logic: new value is always `!booleanAttribute.value` — no special handling for null.
- Read-only completely blocks both `setValue` and action execution.
- Only Space key triggers toggle (confirmed by testing other keys).
- Default `tabIndex` is 0 when none is provided by Mendix.
- Tests use `EditableValueBuilder` from `@mendix/widget-plugin-test-utils`.

**4. Is it user-facing?**
No — test file only.

**5. What new did you learn from this file?**
The Space key specificity is tested explicitly — the test confirms that pressing Enter, Tab, or other keys does NOT trigger the toggle. This matches the ARIA `role="switch"` specification: only Space activates a switch control.

---

## src/components/__tests__/Switch.spec.tsx

**1. What is the purpose of this file?**
Unit tests for the presentational `Switch` component in isolation.

**2. What kind of logic is described in this file?**
Tests: `role="switch"` attribute present; `aria-checked` matches `isChecked` prop; `aria-disabled` set to `"true"` when not editable; `.disabled` CSS class applied when not editable; click calls `onClick`; Space key calls `onKeyDown`; validation text renders when provided.

**3. What part of behavior can be documented from this file?**
- `aria-disabled="true"` is applied when editable is false (not when the Mendix attribute is read-only — that is handled at the container level).
- `.disabled` class is added to `.widget-switch-btn-wrapper` (not just the input element).
- Validation text appears as a separate rendered node when `validationFeedback` prop is provided.

**4. Is it user-facing?**
No — test file only.

**5. What new did you learn from this file?**
The presentational component tests confirm a clean separation: the component only knows about `editable: boolean`, not about Mendix's `readOnly` property. The container translates `booleanAttribute.readOnly` to the `editable` prop, maintaining the separation between Mendix integration and pure UI logic.

---

## e2e/Switch.spec.js

**1. What is the purpose of this file?**
End-to-end Playwright tests for the Switch widget in a real Mendix application.

**2. What kind of logic is described in this file?**
Tests: switch background is green (`rgb(100, 189, 99)`) when checked; clicking the switch toggles the attribute value (verified via radio button state changes); external attribute change (radio button) updates switch visual state; clicking switch executes the configured action (modal popup appears); multiple switch instances are tested on the same page. Session logout after each test.

**3. What part of behavior can be documented from this file?**
- Checked state background: `rgb(100, 189, 99)` — confirming the iOS green color from CSS variables.
- The switch element is identified by `.widget-switch-btn-wrapper` CSS class.
- Switch and radio buttons on the same page share the same boolean attribute — clicking either updates both.
- Action execution is e2e-confirmed: a modal popup appears after switching.
- `waitForNetworkIdle()` is used before assertions — confirming data-driven test setup.

**4. Is it user-facing?**
The tested behaviors (visual toggle, attribute sync, action execution) are user-facing.

**5. What new did you learn from this file?**
The e2e test verifies bi-directional sync: clicking the switch updates radio buttons, AND clicking a radio button updates the switch. This confirms that Mendix's reactive attribute system correctly propagates changes in both directions — the switch does not need to manually subscribe to attribute changes.

---

## CHANGELOG.md

**1. What is the purpose of this file?**
Documents version history for the switch-web widget.

**2. What kind of logic is described in this file?**
Key versions: 4.3.0 (2025-10, improved accessibility: clicking label toggles switch), 4.2.2 (2024-08, action no longer required), 4.2.1 (2023-09, code optimization), 4.2.0 (2023-06, editor icons/tiles update), 4.1.0 (2021-12, dark mode preview), 4.0.0 (2021-09, toolbox category), 3.0.0 (2021-04, Brand Secondary style, ARIA improvements, semantic HTML — removed input element, added div-based approach, `aria-readonly` support).

**3. What part of behavior can be documented from this file?**
- v4.3.0 added label-click-toggles-switch — previously only clicking the switch itself worked.
- v4.2.2 made the onChange action optional — it was previously required.
- v3.0.0 removed the `<input>` element as the interactive element and replaced it with a `<div role="switch">` approach, adding `aria-readonly` support.
- The current architecture (hidden input + styled div) was established in v3.0.0.

**4. Is it user-facing?**
The changelog is publicly visible on the Mendix Marketplace.

**5. What new did you learn from this file?**
The v3.0.0 change from `<input>`-based to `<div role="switch">`-based was driven by accessibility and ARIA compliance. The `<input type="checkbox">` element does not support `role="switch"`, so using it as the interactive element was an accessibility violation. The current approach (hidden input for form semantics + div for the interactive switch) is the correct implementation of ARIA switch.
