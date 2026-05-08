# Draft: badge-web

Extracted by worker on 2026-05-08. Covers all source files and local workspace dependencies.

---

## src/Badge.tsx

**Purpose:** Main widget entry point. Bridges Mendix runtime props to the internal Badge display component, adding keyboard and click interactivity.

**Logic:** Reads `BadgeContainerProps` from the Mendix runtime. Creates memoized `onClick` (calls `executeAction`) and `onKeyDown` (triggers `onClick` on Enter or Space key) callbacks. Computes `clickable` based on `props.onClick?.canExecute`. Passes `onClick`, `onKeyDown`, and `tabIndex` only when `clickable` is true — when no action or action is disabled, the badge renders as a non-interactive display element.

**Behavioral constraints from this file:**
- Badge value uses optional chaining with fallback: `props.value?.value ?? ""` — if `value` is undefined or its value is null/undefined, renders empty string (not `isAvailable` guarded like badge-button-web).
- `tabIndex` defaults to `0` when clickable and not explicitly set — ensuring keyboard focus when an action is configured.
- `clickable` is `props.onClick && props.onClick.canExecute` — the badge becomes interactive only when an action is configured AND that action is currently executable.
- Keyboard support: Enter and Space trigger click — consistent with ARIA button pattern.
- `executeAction` is called in both the mouse and keyboard handlers — ensuring identical behavior for both input methods.

**User-facing:** Yes — the badge is visible to end users and may be interactive.

**New findings:** Unlike badge-button-web, the badge-web widget uses `?? ""` instead of `isAvailable` guard — this means even a loading `DynamicValue` will display empty string rather than requiring the value to be available. The clickability check directly reads `canExecute` rather than calling `isAvailable`.

---

## src/Badge.xml

**Purpose:** Widget descriptor declaring all configurable properties for Mendix Studio/Studio Pro.

**Logic:** Declares `type` (required enumeration: `"badge"` or `"label"`, default `"badge"`), `value` (optional textTemplate with default "Badge"/"Badge"), `onClick` (optional action), plus system properties for Visibility, Name, TabIndex.

**Behavioral constraints from this file:**
- `type` is required (unlike most optional props in other widgets) — always has a value.
- Both `"badge"` and `"label"` types use the same `value` text; only the visual rendering differs (pill vs. rectangular).
- `onClick` is optional — badge renders as a non-clickable display element when not configured.
- `offlineCapable="true"` — works in offline Mendix apps.
- `pluginWidget="true"` — modern pluggable widget API.
- Categorized under "Display" in both Studio and Studio Pro toolboxes.
- `helpUrl` points to official Mendix docs.

**User-facing:** Indirectly — every property is configurable by the Mendix developer.

**New findings:** v3.2.2 changed `onClick` action to `required="false"` (from previously required) to avoid unnecessary Studio Pro warnings. This explains why clicking without a configured action is now silent.

---

## typings/BadgeProps.d.ts

**Purpose:** Auto-generated TypeScript type definitions from `Badge.xml`.

**Logic:** Exports `TypeEnum = "badge" | "label"`, `BadgeContainerProps` (runtime), `BadgePreviewProps` (preview). `value` is `DynamicValue<string> | undefined`; `onClick` is `ActionValue | undefined`.

**Behavioral constraints from this file:**
- `type` is a required string enum — never undefined at runtime.
- `tabIndex` is optional `number | undefined` — when not set, widget uses its own default (0 when clickable, undefined when not).
- Preview props: `value` is a plain string, `onClick` is `{} | null`.

**User-facing:** Internal TypeScript safety only.

**New findings:** The preview props expose `onClick` as `{} | null` (not an action type) — preview cannot actually execute the action, it just knows whether one is configured.

---

## src/components/Badge.tsx

**Purpose:** Presentational component rendering the badge UI as a `<span>` element.

**Logic:** Renders a `<span>` with classes `widget-badge`, `{type}` (either `"badge"` or `"label"`), `{className}`, and conditionally `widget-badge-clickable` when `onClick` is present. Sets `role="button"` when either `onClick` or `onKeyDown` is passed (making it an interactive element in the accessibility tree). `tabIndex` is passed through directly.

**Behavioral constraints from this file:**
- `role="button"` is set when ANY of `onClick` or `onKeyDown` is defined — ensures screen readers announce it as interactive when a click action is configured.
- The CSS type class (`"badge"` or `"label"`) is applied directly as a class name — relies on the Mendix Atlas theme to define `.badge` and `.label` styles.
- `widget-badge-clickable` class is only added when `onClick` is defined (not `onKeyDown`) — cursor and hover styling keyed to the click handler.
- No disabled state rendered differently — if the action becomes non-executable (`canExecute: false`), the parent widget strips the handlers, making it appear non-interactive (but with no visual disabled feedback by default).

**User-facing:** Yes — the rendered HTML `<span>` is visible to end users.

**New findings:** The badge renders with the type as a CSS class (`badge` or `label`), not as a custom attribute, which means Bootstrap/Atlas CSS selectors like `.badge` apply directly, integrating the widget into the Atlas theme design system.

---

## src/Badge.editorConfig.ts

**Purpose:** Provides structure-mode preview and custom page explorer caption for Studio Pro.

**Logic:** `getPreview` renders a badge in the structure-mode preview using the structure preview API: a `buttonInfo` palette background with rounded corners (borderRadius: 22 for badge type, 8 for label type), containing the badge value text in primary color. `getCustomCaption` returns the value or "Badge" as fallback.

**Behavioral constraints from this file:**
- Structure preview uses `palette.background.buttonInfo` (adapts to dark/light mode), with primary text color.
- `borderRadius` differs by type: `22` (pill-shaped) for badge, `8` (slightly rounded rect) for label — matching the visual distinction between the two types.
- Padding adapts: `padding: values.value ? 8 : 18` — larger when value is empty (to show a visible empty placeholder).
- No validation (`check`) function — no invalid configurations possible.
- `getCustomCaption` uses `values?.value || "Badge"` — shows the badge value in the page explorer for quick identification.

**User-facing:** Editor-only — affects Studio Pro structure view and page explorer.

**New findings:** The borderRadius difference (22 vs 8) in the structure preview matches the visual intent: badges are pill-shaped, labels are rectangular. This is a design hint communicated through the preview, not enforced by the CSS class.

---

## src/Badge.editorPreview.tsx

**Purpose:** Renders the badge's live preview on the Mendix Studio design canvas.

**Logic:** Extracts `class`, `style`, `type`, `value` from preview props. Renders the actual `Badge` component with parsed style. Falls back to `""` when `value` is falsy.

**Behavioral constraints from this file:**
- No `onClick`/`onKeyDown` passed in preview — badge always renders as non-interactive in Studio canvas.
- `parseStyle` converts the string style representation to `CSSProperties`.
- Preview exactly matches the runtime badge appearance (same component, same classes).

**User-facing:** Editor canvas only.

**New findings:** Since `onClick` is not passed, the badge never gets `role="button"` or `widget-badge-clickable` in the Studio preview — it always appears as a display element regardless of whether a click action is configured.

---

## packages/shared/widget-plugin-platform/src/framework/execute-action.ts

*(Already documented in badge-button-web draft — same file. Key constraints: guards against undefined, canExecute=false, and isExecuting=true before calling execute.)*

**User-facing:** Indirectly — controls when the badge's onClick action executes.

---

## CHANGELOG.md

**Summary of relevant versions:**

- **v3.2.3 (2026-02-09):** Added license file and open-source dependency readme.
- **v3.2.2 (2024-08-28):** Changed `onClick` action to `required="false"` — removes Studio Pro warning when no action is configured.
- **v3.2.1 (2023-09-27):** Removed redundant code for load time improvement.
- **v3.2.0 (2023-06-05):** Updated page explorer caption to show value; updated icons and tiles; changed structure preview colors for dark/light mode.
- **v3.1.0 (2021-12-23):** Added dark mode to structure preview; dark icons.
- **v3.0.0 (2021-09-28):** Added toolbox category and tile image.

**Findings:** The v3.2.2 change (action required → false) aligns with the `onClick?: ActionValue` optional type in typings. Before this change, Studio Pro would warn if `onClick` wasn't set, which was friction for the common use case of a display-only badge.
