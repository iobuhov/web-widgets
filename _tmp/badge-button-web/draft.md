# Draft: badge-button-web

Widget package: `@mendix/badge-button-web` v3.3.0  
Source: `packages/pluggableWidgets/badge-button-web/`

---

## src/BadgeButton.tsx

**1. Purpose of this file?**
Container component that bridges Mendix platform props to the presentational `BadgeButton` UI component. It is the entry point registered in the Mendix pluggable widget runtime.

**2. What kind of logic is described in this file?**
Wraps the `onClickEvent` action in a `useCallback` handler that calls `executeAction`. Applies `isAvailable` checks before passing `label.value` and `value.value` to the child component, defaulting to empty string when unavailable.

**3. What part of behavior can be documented from this file?**
The button only shows label and value text when the underlying `DynamicValue` is in `"available"` status with a truthy value; loading/error states produce an empty string. Clicking the button fires `executeAction(props.onClickEvent)`, which respects `canExecute` and `isExecuting` guards. If `onClickEvent` is not configured, clicking is a silent no-op. The `onClick` callback is always created (via `useCallback`), so the button element always has a click handler.

**4. Is it user-facing?**
Yes — the widget renders a button visible to end users (via the child component).

**5. What new did you learn from this file?**
`isAvailable` checks both `status === "available"` AND value truthiness, so an empty string `""` from a dynamic attribute is treated as "not available" and renders as empty. This means an intentionally empty label or badge value renders exactly the same as a loading/unavailable state — both produce "".

---

## src/components/BadgeButton.tsx

**1. Purpose of this file?**
Pure presentational React component that renders the button HTML. This is the single source of truth for the widget's DOM structure and CSS class logic.

**2. What kind of logic is described in this file?**
Uses `classNames` to build the CSS class string. Conditionally adds `btn-primary` only when the provided `className` does NOT already contain one of: `btn-primary`, `btn-secondary`, `btn-success`, `btn-warning`, `btn-danger` (detected via regex). Renders a `<button>` with two `<span>` children.

**3. What part of behavior can be documented from this file?**
Renders a `<button>` with two `<span>` children: `.widget-badge-button-text` (label) and `.badge` (value). The base classes are always `widget-badge-button btn`. Default button color is `btn-primary` (Bootstrap primary/blue). The badge `<span>` is always in the DOM even when `value` is empty — no conditional rendering. No disabled state, loading state, or extra ARIA attributes beyond the native `<button>`.

**4. Is it user-facing?**
Yes. This file produces the rendered HTML and CSS classes that end users see and interact with.

**5. What new did you learn from this file?**
The custom-style detection uses a regex (`/btn-(primary|secondary|success|warning|danger)/`), so only exact Bootstrap 4/5 color variants suppress `btn-primary`. A class like `btn-custom-color` would NOT suppress it. Fixed in v3.3.0 (previously custom button styles were not applied correctly). The `onClick` prop is optional — if not provided, the button renders without a click handler (no runtime error).

---

## src/BadgeButton.xml

**1. Purpose of this file?**
Widget descriptor for the Mendix pluggable widget framework. Declares widget identity, Studio category placement, help URL, and all configurable properties.

**2. What kind of logic is described in this file?**
Declares three developer-configurable properties: `label` (textTemplate, optional), `value` (textTemplate, optional), `onClickEvent` (action, optional). Also registers system properties: Visibility, Name, TabIndex. `offlineCapable="true"` means the widget works in offline-enabled apps. `pluginWidget="true"` marks it as the modern Mendix pluggable widget format. No `needsEntityContext` — widget can be placed without a data context.

**3. What part of behavior can be documented from this file?**
Both `label` and `value` are `textTemplate` — they support dynamic expressions, attribute references, and i18n tokens. Default label is "Button" (English) / "Knop" (Dutch); no default for badge value. Widget appears under "Buttons" category in both Studio and Studio Pro. All three configurable properties are optional — a badge button with nothing configured is valid. No conditional visibility controls are built in (beyond the system Visibility property).

**4. Is it user-facing?**
Not directly, but defines what developers configure in Studio/Studio Pro.

**5. What new did you learn from this file?**
The `onClickEvent` is optional with no required attribute, meaning clicking a badge button with no action configured is silently ignored. The help URL points to `docs.mendix.com/appstore/widgets/badge-button`. There are no advanced-mode properties or validation (`check`) functions — the widget has no configuration that can be invalid.

---

## typings/BadgeButtonProps.d.ts

**1. Purpose of this file?**
Auto-generated TypeScript type definitions derived from `BadgeButton.xml`. Provides compile-time types for both the runtime container props and the Studio Pro design-time preview props.

**2. What kind of logic is described in this file?**
Defines `BadgeButtonContainerProps` (runtime): `label` and `value` as `DynamicValue<string>`, `onClickEvent` as `ActionValue`. Defines `BadgeButtonPreviewProps` (design-time): all props as plain strings; includes `renderMode` union ("design" | "xray" | "structure") and `translate` function.

**3. What part of behavior can be documented from this file?**
`DynamicValue<string>` can have status `"loading"`, `"available"`, or `"unavailable"` — the widget guards against all non-available states. `style` is `CSSProperties | undefined` (optional inline style). `tabIndex` is optional. Preview props use plain strings (Studio Pro resolves expressions before passing them). `onClickEvent` in preview is `{} | null`.

**4. Is it user-facing?**
No. Type declarations only.

**5. What new did you learn from this file?**
`className` in `BadgeButtonPreviewProps` is deprecated since Mendix 9.18.0 — `class` should be used instead. `renderMode` confirms the widget supports three Studio Pro design canvas modes: normal design, x-ray (structural overlay), and structure (abstract block diagram). The `translate` callback in preview props enables i18n of caption text in the design-time preview.

---

## src/BadgeButton.editorConfig.ts

**1. Purpose of this file?**
Defines the structure mode (abstract block diagram) preview appearance in Mendix Studio Pro. Also provides a custom caption displayed in the page explorer.

**2. What kind of logic is described in this file?**
Exports `getPreview` which builds a `StructurePreviewProps` tree: a row layout with one button container and one trailing empty container. The button container has `backgroundColor: buttonInfo` and `borderRadius: 4`. Inside it, label text (#FFF, bold, fontSize 8) and a badge container (white background, `borderRadius: 16`, padding adapts to whether value is set). Also exports `getCustomCaption` which returns `values.label` or "Badge button" as fallback.

**3. What part of behavior can be documented from this file?**
In structure mode the badge renders as a pill shape (`borderRadius: 16`) on a white background inside a blue button. The whole button uses the theme's `buttonInfo` color (blue: #146FF4 light, #579BF9 dark). Adapts to dark/light mode via `structurePreviewPalette`. Badge padding: `padding: 4` when value present, `padding: 8` when empty — maintaining visual balance. Page explorer caption falls back to "Badge button" (not empty) when no label is configured.

**4. Is it user-facing?**
No. Design-time only (Mendix Studio Pro structure canvas and page explorer).

**5. What new did you learn from this file?**
`getCustomCaption` returns the label directly (matching visible button text, not the badge value) — this was added in v3.2.0. The structure preview renders a compact fixed-size layout (not reflective of actual runtime dimensions). No `check` validation function is exported — indicating no prop combination is invalid.

---

## src/BadgeButton.editorPreview.tsx

**1. Purpose of this file?**
Renders the live preview of the widget in Mendix Studio Pro's design canvas (non-structure modes).

**2. What kind of logic is described in this file?**
Imports `parseStyle` to convert the style string prop (a raw CSS string from Studio) to a `CSSProperties` object. Renders the actual `BadgeButton` presentational component with all preview props forwarded. No `onClick` is passed — preview is non-interactive.

**3. What part of behavior can be documented from this file?**
The design canvas preview mirrors the runtime widget appearance exactly, using the same component (same Bootstrap classes, same DOM structure). Style strings from the designer are parsed at render time. Empty or malformed style strings produce an empty style object — no crash, no visible error.

**4. Is it user-facing?**
No. Designer-facing only.

**5. What new did you learn from this file?**
Since the preview reuses the exact same `BadgeButton` component as runtime, Bootstrap button classes (`btn`, `btn-primary`, etc.) are applied in the Studio design canvas — giving accurate visual preview including the default blue color. `parseStyle` silently swallows parse errors; malformed inline styles set in Studio Pro render with no inline styles rather than throwing.

---

## src/components/__tests__/BadgeButton.spec.tsx

**1. Purpose of this file?**
Unit tests for the `BadgeButton` presentational component covering rendering, click behavior, and the CSS class fallback logic.

**2. What kind of logic is described in this file?**
Tests: (1) structure check — button has correct CSS classes and children; (2) click event fires the `onClick` callback; (3) custom classes are applied; (4) btn-primary is NOT added when btn-success is present; (5) btn-primary IS added for non-btn-* custom classes; (6) btn-primary IS added when className is undefined.

**3. What part of behavior can be documented from this file?**
The behavioral contract for CSS classes is formally tested: `widget-badge-button btn` are always present; `btn-primary` is the default; any of `btn-{primary,secondary,success,warning,danger}` suppresses the default. When a known button style class is provided, it completely replaces the default — no duplication. Rendered DOM: `<span class="widget-badge-button-text">` for label and `<span class="badge">` for value.

**4. Is it user-facing?**
No. Tests only.

**5. What new did you learn from this file?**
`expect(button.className).toEqual("widget-badge-button btn btn-success")` (without `btn-primary`) explicitly confirms no class duplication. The tests use `@testing-library/react` and `userEvent` — click simulation is realistic (fires all browser events). No tests for empty/undefined label or value rendering.

---

## e2e/render.spec.js

**1. Purpose of this file?**
End-to-end tests verifying that the badge button renders correctly with dynamic data and responds to value updates in a real Mendix runtime.

**2. What kind of logic is described in this file?**
Tests widget visibility, initial content ("Button" label, "New" badge), live value update (typing into a linked textbox changes the badge text), and screenshot baseline comparison.

**3. What part of behavior can be documented from this file?**
The badge value is reactive — when the underlying Mendix attribute changes, the badge text updates without page reload. The default label is "Button" and a common badge value used in tests is "New". A visual regression screenshot baseline exists for the main page.

**4. Is it user-facing?**
Tests only, but validates user-facing behavior.

**5. What new did you learn from this file?**
The widget supports live data binding for the badge value — changes to a bound attribute are immediately reflected in the rendered badge without any reload. Sessions are explicitly logged out after each test to work around Mendix's 5-session license limit in test environments.

---

## e2e/onClick.spec.js

**1. Purpose of this file?**
E2E tests covering all supported action types for the `onClickEvent` property.

**2. What kind of logic is described in this file?**
Tests four action types: call microflow (shows dialog with context data), call nanoflow (shows dialog), open page (navigates), open modal popup page (shows modal). Also tests close page action.

**3. What part of behavior can be documented from this file?**
The widget supports the full Mendix action spectrum: microflows, nanoflows, page navigation (full-page and modal popup), and close-page. The microflow test confirms contextual data passes to the microflow — dialog says "Microflow Successfully Called With badge New", meaning the widget's data context (including the badge value) is accessible to the configured microflow.

**4. Is it user-facing?**
Tests only, but validates user-facing click behavior.

**5. What new did you learn from this file?**
The microflow receives badge context data — the widget's data context is accessible from the action, not just a static trigger. All action types work without any special widget configuration beyond setting `onClickEvent`.

---

## e2e/dataTypes.spec.js

**1. Purpose of this file?**
E2E tests verifying that the badge value renders correctly for different Mendix attribute data types.

**2. What kind of logic is described in this file?**
Tests four data types for the `value` prop: string ("New"), integer (10), long (2,147,483,647), decimal (2.5). Each is verified against expected rendered text.

**3. What part of behavior can be documented from this file?**
The badge value renders as a Mendix-formatted string for string, integer, long (with locale comma separators), and decimal types. Mendix's locale-specific number formatting applies — the long value uses comma separators in the test output.

**4. Is it user-facing?**
Tests only, but confirms user-visible rendering for different data types.

**5. What new did you learn from this file?**
Because `value` is `textTemplate`, numeric values are rendered as Mendix-formatted strings — locale and display format settings in the Mendix model apply. The badge does not do its own number formatting; formatting is delegated to the Mendix runtime.

---

## e2e/differentViews.spec.js

**1. Purpose of this file?**
E2E tests for rendering in various Mendix page structures: data grids (listen mode), list views, template grids, and tab containers.

**2. What kind of logic is described in this file?**
Tests correct rendering in: data grid listen mode (shows data from selected row), list view (each row has its own badge button), template grid (multiple rows), and tab container (visible in both default and secondary tabs). Tests are skipped when `MODERN_CLIENT=true`.

**3. What part of behavior can be documented from this file?**
The widget works in all classic Mendix page structures. In data grid listen mode, the badge updates based on the selected grid row. The `MODERN_CLIENT` skip flag (`process.env.MODERN_CLIENT === true`) explicitly marks this widget as incompatible with Mendix's newer React-based client.

**4. Is it user-facing?**
Tests only, but validates user-facing compatibility across page structures.

**5. What new did you learn from this file?**
The widget does NOT support Mendix's modern React client (`MODERN_CLIENT=true`). It targets the classic Dojo-based web client only. This is a significant compatibility constraint for widget placement decisions.

---

## widget-plugin-platform/src/framework/execute-action.ts (local dependency)

**1. Purpose of this file?**
Utility that safely executes a Mendix `ActionValue` without double-triggering or executing disabled actions.

**2. What kind of logic is described in this file?**
Guards `action.execute()` behind `action && action.canExecute && !action.isExecuting`. If action is undefined, the function is a no-op. No error handling — if `execute()` throws, it propagates up.

**3. What part of behavior can be documented from this file?**
Clicking when the action is currently executing (e.g. slow microflow) will not re-trigger it. Clicking when action is not executable (e.g. access rights restriction) is silently ignored. The guard does not debounce or throttle — rapid clicks will fire the action multiple times if each completes before the next click.

**4. Is it user-facing?**
No. Internal utility.

**5. What new did you learn from this file?**
The `isExecuting` guard only prevents overlapping executions, not rapid sequential ones. If a user clicks rapidly and the first action completes quickly, the second click fires a new action. This is intentional — the widget does not add debounce on top.

---

## widget-plugin-platform/src/framework/is-available.ts (local dependency)

**1. Purpose of this file?**
Utility to check if a `DynamicValue` or `EditableValue` is ready and has a non-empty value.

**2. What kind of logic is described in this file?**
Returns `true` only when `property.status === "available"` AND `property.value` is truthy. Single expression: `property && property.status === "available" && property.value`.

**3. What part of behavior can be documented from this file?**
Returns `false` for: undefined/null property, status "loading", status "unavailable", or a falsy value (empty string `""`, `false`, `0`, `null`). An available `DynamicValue<string>` with `value = ""` returns `false`. In the badge button, this means `isAvailable(props.label)` and `isAvailable(props.value)` both return `false` for empty-string values — the container falls back to the `""` default in both the "not available" and "available but empty" cases.

**4. Is it user-facing?**
No. Internal utility.

**5. What new did you learn from this file?**
The utility conflates "no data loaded" with "empty string data" — a Mendix developer who deliberately sets an empty badge value gets the same result as one who left it unbound. In practice this is fine since an empty badge still renders (the `<span class="badge">` is always in the DOM from the presentational component).

---

## widget-plugin-platform/src/preview/structure-preview-api.ts (local dependency)

**1. Purpose of this file?**
Provides TypeScript types and builder functions for constructing structure preview descriptors, plus the shared color palette for dark/light mode theming.

**2. What kind of logic is described in this file?**
Exports builder functions (`container`, `rowLayout`, `text`, `dropzone`, `selectable`, `datasource`, `image`, `svgImage`) and the `structurePreviewPalette` with colors: `buttonInfo` = #146FF4 (light) / #579BF9 (dark). Also exports `colorWithAlpha` for alpha variants.

**3. What part of behavior can be documented from this file?**
The badge button's structure preview uses `palette.background.buttonInfo` as the button background — consistent with Mendix's standard button appearance. Dark mode uses a lighter blue (#579BF9) vs light mode (#146FF4). The palette is shared across all widgets using this package, ensuring visual consistency in structure mode.

**4. Is it user-facing?**
No. Design-time only.

**5. What new did you learn from this file?**
Changes to the shared palette affect all widgets using this platform package. The `StructurePreviewProps` type union (`ImageProps | ContainerProps | RowLayoutProps | TextProps | DropZoneProps | SelectableProps | DatasourceProps`) defines all possible preview node types — `BadgeButton` uses only `Container`, `RowLayout`, and `Text`.

---

## widget-plugin-platform/src/preview/parse-style.ts (local dependency)

**1. Purpose of this file?**
Converts a CSS string (e.g., `"color: red; font-size: 12px"`) to a React `CSSProperties` object for use as an inline style in preview rendering.

**2. What kind of logic is described in this file?**
Splits on `;`, then `:`, trims whitespace, converts kebab-case property names to camelCase using a regex replace. Wraps in try/catch returning `{}` on failure.

**3. What part of behavior can be documented from this file?**
Style strings set on the widget in Studio Pro are parsed at preview render time. Malformed or empty strings silently produce an empty object — no visible error in the designer. `font-size` → `fontSize`, `background-color` → `backgroundColor`, etc.

**4. Is it user-facing?**
No. Design-time utility only.

**5. What new did you learn from this file?**
The parser splits on the first `:` via `line.split(":")` producing a 2-element array via `pair[0]`/`pair[1]` — but `String.split(":")` without a limit returns ALL segments, so `background: url(data:image/png)` would break (pair[1] = "url(data", pair[2] ignored). This is a known limitation of the simple split approach for complex CSS values.

---

## Summary of Key Findings

- **Widget identity**: Bootstrap-styled button with an embedded badge/counter span. Both label and value accept dynamic Mendix expressions via `textTemplate`.
- **Props**: `label` (button caption, textTemplate, default "Button"), `value` (badge text, textTemplate), `onClickEvent` (action). All optional.
- **CSS class behavior**: `btn-primary` is default; suppressed when any of `btn-{primary,secondary,success,warning,danger}` is present in `className`. Fixed in v3.3.0.
- **DOM structure**: Always `<button class="widget-badge-button btn ..."><span class="widget-badge-button-text">label</span><span class="badge">value</span></button>` — badge span is always in DOM.
- **Reactivity**: Value and label are reactive to underlying attribute changes via Mendix `DynamicValue`.
- **Action types**: Supports microflow, nanoflow, open page, open popup, close page.
- **Data types**: Value renders as Mendix-formatted string for string, integer, long, decimal — locale formatting applies.
- **Compatibility**: Does NOT support Mendix modern React client. Works in list views, template grids, data grids (listen), and tab containers in the classic Dojo client.
- **Offline**: `offlineCapable=true` — usable in offline Mendix apps.
- **Empty value behavior**: Empty string values from dynamic attributes render as empty (treated as "unavailable" by `isAvailable`). Badge `<span>` always renders regardless.
- **No entity context required**: Widget can be placed anywhere in a page without a surrounding data source.
