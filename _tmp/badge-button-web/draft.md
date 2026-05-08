# Draft: badge-button-web

Extracted by worker on 2026-05-08. Covers all source files and local workspace dependencies.

---

## src/BadgeButton.tsx

**Purpose:** Main widget entry point. Bridges Mendix runtime props to the internal BadgeButton presentational component.

**Logic:** Reads `BadgeButtonContainerProps` from the Mendix runtime. Uses `isAvailable` to guard access to `label` and `value` before reading their `.value` — if either is unavailable (loading or error), an empty string is passed instead. Creates a memoized `onClick` callback via `useCallback` that calls `executeAction` on the configured action.

**Behavioral constraints from this file:**
- If `label` is undefined or not available (`status !== "available"`), the button displays empty string as label — no null render.
- If `value` is undefined or not available, the badge displays empty string — the badge element is always rendered, just empty.
- Click is always callable (the `onClick` callback is always created), but `executeAction` internally guards against disabled or in-progress actions.
- `onClickEvent` is optional — if not configured, clicking the button does nothing (executeAction silently exits).

**User-facing:** Yes — the widget renders a button visible to end users.

**New findings:** The `isAvailable` guard uses both `status === "available"` AND truthy `value` — so even an available empty string label would show as "" (which is falsy — this means an explicitly empty label also results in ""). This is a subtle edge case.

---

## src/BadgeButton.xml

**Purpose:** Widget descriptor declaring all configurable properties for Studio/Studio Pro.

**Logic:** Declares `label` (textTemplate, optional, default "Button"/"Knop"), `value` (textTemplate, optional, the badge content), `onClickEvent` (action, optional), plus system properties for Visibility, Name, TabIndex.

**Behavioral constraints from this file:**
- Both `label` and `value` are optional `textTemplate` properties — they support dynamic Mendix expressions, not just static text.
- `onClickEvent` is optional — no click behavior is required.
- `offlineCapable="true"` — works in offline Mendix apps.
- No `needsEntityContext` attribute — the widget can be used without an entity context object.
- `pluginWidget="true"` — uses the modern pluggable widget API.
- Default label translation: "Button" (en_US), "Knop" (nl_NL). No default for the badge value.

**User-facing:** Indirectly — every property here is configurable by the Mendix developer.

**New findings:** The widget is categorized as "Buttons" in both Studio and Studio Pro. It has a helpUrl pointing to official Mendix docs. There are no advanced mode properties or conditional visibility controls built in.

---

## typings/BadgeButtonProps.d.ts

**Purpose:** Auto-generated TypeScript type definitions from `BadgeButton.xml`.

**Logic:** Exports `BadgeButtonContainerProps` (runtime props) and `BadgeButtonPreviewProps` (editor preview props). Both `label` and `value` are `DynamicValue<string> | undefined` at runtime; `onClickEvent` is `ActionValue | undefined`.

**Behavioral constraints from this file:**
- `DynamicValue<string>` can have status `"loading"`, `"available"`, or `"unavailable"` — the widget must guard against all non-available states.
- `style` is `CSSProperties | undefined` — optional inline style.
- `tabIndex` is optional.
- Preview props: `label` and `value` are plain `string` (already resolved); `onClickEvent` is `{} | null`.

**User-facing:** Internal — TypeScript compile-time safety only.

**New findings:** The preview props use plain strings (not DynamicValue) since Studio Pro resolves expressions to their string representation for preview purposes.

---

## src/components/BadgeButton.tsx

**Purpose:** Presentational component rendering the badge button UI. Stateless, accepts resolved string values.

**Logic:** Renders a `<button>` element with Bootstrap-compatible classes. The button has class `widget-badge-button btn` plus any passed `className`. If the `className` does not already contain a Bootstrap button variant class (`btn-primary`, `btn-secondary`, `btn-success`, `btn-warning`, `btn-danger`), `btn-primary` is added as the default. Inside the button: a `<span class="widget-badge-button-text">` for the label, and a `<span class="badge">` for the badge value.

**Behavioral constraints from this file:**
- Default button style is `btn-primary` (blue) unless an explicit Bootstrap variant is found in `className`.
- If `className` contains one of `btn-primary|btn-secondary|btn-success|btn-warning|btn-danger`, the default `btn-primary` class is NOT added (mutual exclusion via regex).
- The badge `<span>` is always rendered even if `value` is empty — the badge element is always in the DOM.
- No disabled state, no loading state, no aria attributes beyond what the native `<button>` provides.
- The `onClick` prop is optional — if not provided, the button renders without a click handler.

**User-facing:** Yes — the directly rendered HTML button.

**New findings:** The v3.3.0 fix ("custom button styles not applied correctly") relates to the regex match for Bootstrap variants — the bug was likely in the conditional class logic. The fix ensures that custom classes that include Bootstrap variant names are respected instead of being overridden.

---

## src/BadgeButton.editorConfig.ts

**Purpose:** Provides structure-mode preview and a custom page explorer caption for Studio Pro.

**Logic:** `getPreview` renders a styled badge button representation using the structure preview API: a rounded button background in `palette.background.buttonInfo` color, with white bold text for the label and white badge circle with the value text. `getCustomCaption` returns the label value or "Badge button" as fallback.

**Behavioral constraints from this file:**
- Structure preview uses `buttonInfo` palette color, ensuring dark/light mode compatibility.
- Badge padding in the structure preview adapts: `padding: values.value ? 4 : 8` — smaller padding when value is present, larger when empty.
- Badge has `borderRadius: 16` (rounded pill shape) and white background.
- No validation (`check`) function — the widget has no configuration that can be invalid.
- The structure preview renders a fixed-size compact layout (non-responsive, not reflective of actual dimensions).

**User-facing:** Editor-only — affects Studio Pro page explorer and structure view.

**New findings:** `getCustomCaption` returns the label directly (not the badge value), making the page explorer caption match the visible button text.

---

## src/BadgeButton.editorPreview.tsx

**Purpose:** Renders the badge button's live preview on the Mendix Studio design canvas.

**Logic:** Parses the `style` string via `parseStyle`, then renders the actual `BadgeButton` component with `className`, `label`, `style`, and `value` from preview props. The preview exactly matches the runtime rendering (same component, same classes, same structure).

**Behavioral constraints from this file:**
- Preview uses the same `BadgeButton` component as runtime — design canvas rendering matches production.
- `parseStyle` converts the string-form style from preview props to a `CSSProperties` object.
- No click handling in preview — `onClick` is not passed.

**User-facing:** Editor canvas only.

**New findings:** Since the preview reuses the exact same component as runtime, the Bootstrap button classes (`btn`, `btn-primary`, etc.) are applied in the Studio design canvas, giving an accurate preview of the button appearance.

---

## packages/shared/widget-plugin-platform/src/framework/execute-action.ts

**Purpose:** Safely executes a Mendix action, respecting `canExecute` and `isExecuting` guards.

**Logic:** Calls `action.execute()` only if `action` is defined, `canExecute` is true, and `isExecuting` is false.

**Behavioral constraints from this file:**
- No-op when `action` is undefined (not configured).
- No-op when `action.canExecute` is false (action is disabled by a Mendix expression or rule).
- No-op when `action.isExecuting` is true (action is already running — prevents double execution).
- No error handling — if `action.execute()` throws, it propagates up.

**User-facing:** Indirectly — ensures click actions are only triggered in valid states.

**New findings:** The guard does not debounce or throttle — rapid clicks will fire the action multiple times if each completes before the next click. The `isExecuting` guard only prevents overlapping executions, not rapid sequential ones.

---

## packages/shared/widget-plugin-platform/src/framework/is-available.ts

**Purpose:** Guards access to `DynamicValue` or `EditableValue` props, returning true only when the value is loaded and non-falsy.

**Logic:** Returns `property && property.status === "available" && property.value`.

**Behavioral constraints from this file:**
- Returns `false` (not available) for: undefined/null property, status "loading", status "unavailable", or a falsy value (empty string `""`, `false`, `0`, `null`).
- A `DynamicValue<string>` with `value = ""` (empty string) returns `false` from `isAvailable` — empty string is treated as unavailable.
- This means an intentionally empty label/badge value will fall back to `""` in the parent widget (the `? props.label.value : ""` pattern produces empty string in both the available-but-empty and not-available cases).

**User-facing:** Indirectly — determines whether label and badge value are displayed.

**New findings:** The `isAvailable` check conflates "no data loaded" with "empty string data", which could cause confusion if a Mendix developer deliberately sets an empty badge value. In practice, empty badge values are fine since the badge `<span>` renders regardless.

---

## CHANGELOG.md

**Summary of relevant versions:**

- **v3.3.0 (2026-03-13):** Fixed an issue where custom button styles were not being applied correctly (the `btn-primary` default class override bug fix).
- **v3.2.2 (2026-02-09):** Added license file and open-source dependency readme.
- **v3.2.1 (2023-09-27):** Removed redundant code for load time improvement.
- **v3.2.0 (2023-06-05):** Updated structure-mode preview colors for dark/light modes; updated page explorer caption to show label; updated icons and tiles.
- **v3.1.0 (2021-12-23):** Added dark mode to structure preview; added dark icons for tile/list view.
- **v3.0.1 (2021-12-03):** Fixed design properties and styles not applied in Design mode.
- **v3.0.0 (2021-09-28):** Added toolbox category and tile image.

**Findings:** The v3.3.0 fix for "custom button styles not applied correctly" is the most relevant behavioral change — it corresponds to the `btn-primary` default class logic in `components/BadgeButton.tsx`. The v3.2.0 update to show the label in the page explorer caption corresponds to the `getCustomCaption` function in `editorConfig.ts`.
