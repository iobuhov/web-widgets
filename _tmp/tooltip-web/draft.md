# Draft: tooltip-web

Widget package: `packages/pluggableWidgets/tooltip-web`

---

## src/Tooltip.xml

**1. What is the purpose of this file?**
Widget descriptor XML defining the widget's identity, all configurable props, and Studio Pro categorization. Generates `TooltipProps.d.ts`.

**2. What kind of logic is described in this file?**
Declares: `trigger` (widget placeholder for the element that triggers the tooltip); `renderMethod` ("text" | "custom"); `htmlMessage` (widget placeholder for custom content, shown only in custom mode); `textMessage` (text template for text mode); `tooltipPosition` (left/right/top/bottom — default "top"); `arrowPosition` (start/none/end — default "none", where "none" means centered); `openOn` (click/hover/hoverFocus — default "hover"). Widget is `needsEntityContext="true"`, `offlineCapable="true"`, categorized under "Display".

**3. What part of behavior can be documented from this file?**
- Arrow position "none" means centered — confusing name, but the XML description clarifies it.
- Default trigger mode is "hover" — tooltips show on hover unless configured otherwise.
- Both `trigger` and `htmlMessage` are widget placeholders (dropzones), not data attributes.
- The "hoverFocus" mode is an explicit third option, separate from both hover and click.
- Widget is offline capable despite using floating-ui for positioning.

**4. Is it user-facing?**
Defines the developer-facing configuration interface in Studio/Studio Pro.

**5. What new did you learn from this file?**
The `arrowPosition` enum uses "none" as the value for centered arrow position — a naming convention mismatch with the user-visible label "center". The XML description explains it, but the code must compare against `"none"` to detect center alignment.

---

## src/Tooltip.tsx

**1. What is the purpose of this file?**
The Mendix widget container entry point. Bridges `TooltipContainerProps` to the presentational `Tooltip` component by unwrapping Mendix-specific types and translating the two-property position format into floating-ui's single `Placement` string.

**2. What kind of logic is described in this file?**
Extracts `textMessage.value` from `DynamicValue<string>`. Calls `translatePosition(tooltipPosition, arrowPosition)` to merge the two enum props into a floating-ui `Placement` (e.g., "top-start", "left"). Passes `className`, `name`, `style`, `tabIndex` directly through. Imports global SCSS.

**3. What part of behavior can be documented from this file?**
- The container holds zero state — it only adapts Mendix props.
- `textMessage` is a `DynamicValue<string>`, so `.value` may be `undefined` before loading.
- `tooltipPosition` and `arrowPosition` are separate Mendix props combined into one floating-ui string.

**4. Is it user-facing?**
No — internal Mendix-to-component adapter.

**5. What new did you learn from this file?**
The position translation from two XML props into a single `Placement` is done in the container, not the presentational component. This keeps floating-ui knowledge entirely out of the container (just one utility call) and keeps the presentational component's prop API aligned with floating-ui's native types.

---

## src/components/Tooltip.tsx

**1. What is the purpose of this file?**
The core presentational/business logic component. Manages `showTooltip` state, integrates with `useFloatingUI`, conditionally renders tooltip content, and routes to text or custom content based on `renderMethod`.

**2. What kind of logic is described in this file?**
Uses `useState` for `showTooltip` and a `ref` for the arrow DOM element. Calls `useFloatingUI` with position, state setter, arrow ref, and openOn mode. Conditionally renders tooltip content only when `showTooltip && (textMessage || htmlMessage)`. Routes rendering: `renderMethod === "text"` shows the text string; `renderMethod === "custom"` shows `htmlMessage` React nodes. Applies `widget-tooltip-${position}` CSS class. Renders arrow inside the tooltip content div with `widget-tooltip-arrow-${staticSide}` class. Preview mode passes `preview={true}` to disable event handlers.

**3. What part of behavior can be documented from this file?**
- Tooltip does not render at all when both `textMessage` and `htmlMessage` are absent.
- The arrow element is a child of the tooltip content div, not a sibling.
- `staticSide` (computed in `useFloatingUI`) determines which CSS arrow variant class is applied.
- Trigger element is wrapped in a `widget-tooltip-trigger` div to constrain its width to `fit-content`.
- `blurFocusEvents` from the hook are applied to the trigger only in `hoverFocus` mode.

**4. Is it user-facing?**
Yes — produces the visible tooltip.

**5. What new did you learn from this file?**
The tooltip open condition checks both `textMessage` and `htmlMessage` — meaning the tooltip stays hidden if both are empty, even when the trigger is interacted with. This prevents an empty tooltip bubble from appearing during misconfiguration or data loading.

---

## src/utils/useFloatingUI.ts

**1. What is the purpose of this file?**
Custom React hook encapsulating all floating-ui integration: positioning strategy, middleware pipeline, interaction hooks for all three open modes, and arrow position computation.

**2. What kind of logic is described in this file?**
Strategy: `"fixed"` positioning. Middleware (in order): `offset(8)` → `flip()` → `shift()` → `arrow({ element: arrowRef })`. `autoUpdate` watches for scroll/resize/DOM changes and recalculates. Interactions: `useHover` (enabled for hover/hoverFocus) with `openDelay: 25ms`, `closeDelay: 0ms`, `restMs: 25`, and `handleClose: safePolygon()`; `useFocus` (hoverFocus only); `useClick` (click only, toggle mode); `useDismiss` (always); `useRole` (always, role="tooltip"). Arrow math: `staticSide` maps placement side to its opposite (top→bottom, right→left etc.); `arrowStyles` inlines the `left`/`top` pixel values from `middlewareData.arrow` with an `alignmentOffset` of 8px for "start"-aligned vertical placements.

**3. What part of behavior can be documented from this file?**
- 8px gap between trigger and tooltip (from `offset(8)` middleware).
- `safePolygon()` creates an invisible polygon between trigger and tooltip, preventing the tooltip from closing while the mouse moves across the gap.
- `flip()` automatically moves the tooltip to the opposite side when there is insufficient viewport space.
- `shift()` slides the tooltip along its axis to stay within the viewport.
- Hover open delay: 25ms (prevents accidental triggers). Close delay: 0ms (instant).
- `useDismiss` closes the tooltip on outside click for all modes.
- `useRole` applies `role="tooltip"` ARIA attribute to the floating element.

**4. Is it user-facing?**
No — internal hook, but drives all user-visible tooltip positioning behavior.

**5. What new did you learn from this file?**
The `alignmentOffset` calculation (8px for "start"-aligned vertical placements) handles the edge case where the arrow is aligned to the start of the tooltip — the arrow's pixel position from floating-ui needs to be adjusted by the arrow's own width to account for the alignment. Without this, the arrow would appear misaligned when using "top-start" or "bottom-start" placements.

---

## src/utils/index.ts

**1. What is the purpose of this file?**
Exports `translatePosition`, a pure utility that converts Mendix's two-enum position model into floating-ui's single `Placement` string.

**2. What kind of logic is described in this file?**
`translatePosition(tooltipPosition, arrowPosition)`: returns `${tooltipPosition}` if `arrowPosition === "none"`, otherwise `${tooltipPosition}-${arrowPosition}`. Casts result to floating-ui `Placement` type.

**3. What part of behavior can be documented from this file?**
- (top, none) → "top"; (left, start) → "left-start"; (bottom, end) → "bottom-end".
- Used in both the container component and the editor preview component.

**4. Is it user-facing?**
No — internal utility.

**5. What new did you learn from this file?**
This utility is shared between container and preview, ensuring consistent position translation in both runtime and Studio design-mode contexts. It's a thin bridge between Mendix's separate-enum convention and floating-ui's compound placement strings.

---

## src/Tooltip.editorConfig.ts

**1. What is the purpose of this file?**
Provides `getProperties()` (conditional property visibility), `check()` (validation), and `getPreview()` (structure preview layout) for Studio Pro.

**2. What kind of logic is described in this file?**
`getProperties()`: hides `htmlMessage` when `renderMethod === "text"`, hides `textMessage` when `renderMethod === "custom"`. `check()`: requires `textMessage` (non-empty) for text mode; requires `htmlMessage.widgetCount > 0` for custom mode. `getPreview()`: builds a structured layout with a title bar ("Tooltip"), a centered message area (text or dropzone), and a trigger dropzone below; supports dark/light mode palette.

**3. What part of behavior can be documented from this file?**
- Studio Pro hides the irrelevant property based on render method — developers only see what they need.
- Validation error messages reference the specific property path (e.g., "textMessage", "htmlMessage").
- Custom mode validation uses `widgetCount === 0` to detect an empty dropzone.
- Structure preview shows three regions: title bar, tooltip message, trigger area.
- Dark mode preview text: `#DEDEDE`; placeholder text: `#A4A4A4`.
- Light mode preview text: `#000000`; placeholder text: `#6B707B`.

**4. Is it user-facing?**
Yes — controls Studio Pro property panel and structure preview.

**5. What new did you learn from this file?**
The `widgetCount` property on preview props is a Mendix mechanism to check if a widget dropzone is populated, enabling validation that prevents publishing a tooltip with no content. This is a pattern used across Mendix pluggable widgets that include dropzones.

---

## src/Tooltip.editorPreview.tsx

**1. What is the purpose of this file?**
Renders the tooltip widget's interactive preview on the Studio Pro design canvas, using the presentational component with `preview={true}` to disable event handlers.

**2. What kind of logic is described in this file?**
Calls `parseStyle()` on props.style. Renders `<Tooltip>` with `preview={true}`, passing `htmlMessage` through `props.htmlMessage.renderer` and `trigger` through `props.trigger.renderer` (Mendix framework wrappers for dropzone UI). Imports SCSS via `getPreviewCss()` which returns compiled styles as a string for Studio injection.

**3. What part of behavior can be documented from this file?**
- `preview={true}` disables all tooltip interactivity in the Studio canvas.
- The renderer wrappers show Mendix's dropzone editing UI (drag-and-drop placeholder).
- `getPreviewCss()` is Mendix's mechanism for injecting widget CSS into Studio Pro's preview rendering context.

**4. Is it user-facing?**
Yes — visible to developers in Studio Pro.

**5. What new did you learn from this file?**
The `preview` prop is the single flag that suppresses all interaction handlers in the presentational component. Rather than duplicating the presentational component for preview, a single boolean switches the component between interactive and inert modes — a clean design that avoids divergence between preview and runtime rendering.

---

## src/ui/Tooltip.scss

**1. What is the purpose of this file?**
Defines all visual styling: tooltip container colors, typography, shadow, z-index, and four arrow position variants.

**2. What kind of logic is described in this file?**
`.widget-tooltip-content`: background white, color `#24276c` (dark blue), font bold 14px, padding 6px, border-radius 3px, box-shadow `0 0 5px rgba(0, 0, 0, 0.3)`, z-index 50, display inline-block, word-break with break-spaces. Arrow base: `position: absolute`, 8×8px, shadow `1px -1px 1px`. Arrow uses `::before` pseudo-element rotated 45° for the diamond/triangle shape. Four arrow variants by `staticSide`: `.widget-tooltip-arrow-left` (left: -8px), `.widget-tooltip-arrow-bottom` (bottom: -3px), `.widget-tooltip-arrow-top` (top: -4px), `.widget-tooltip-arrow-right` (right: 1px). Each variant applies a directionally-appropriate box-shadow to simulate consistent top-left lighting. `.widget-tooltip-trigger`: width `fit-content`.

**3. What part of behavior can be documented from this file?**
- Default tooltip background: white; text color: `#24276c`.
- Font: bold, 14px.
- Border-radius: 3px; box-shadow: `0 0 5px rgba(0, 0, 0, 0.3)`.
- Z-index: 50.
- Arrow: 8×8px, CSS pseudo-element rotated 45°, inherits parent background color.
- Arrow shadows differ per direction to preserve visual consistency (light appears from top-left).
- Trigger container width is forced to `fit-content` (prevents stretching in layout containers — added in v1.4.1).
- Only 4 CSS arrow variants (one per side), not 8 — alignment is handled purely by floating-ui's pixel offset.

**4. Is it user-facing?**
Yes — all visible colors, dimensions, shadows, and animations are defined here.

**5. What new did you learn from this file?**
There are exactly 4 arrow CSS classes (top/bottom/left/right), not 8. Arrow alignment within a side (start/center/end) is handled entirely by floating-ui's pixel positioning (the `arrowStyles` inline style). CSS only needs to know which side the arrow is on, not its alignment along that side. This keeps the CSS simple and puts the alignment logic where it belongs — in the hook.

---

## typings/TooltipProps.d.ts

**1. What is the purpose of this file?**
Auto-generated TypeScript typings from `Tooltip.xml`. Defines `TooltipContainerProps`, `TooltipPreviewProps`, and all enum types.

**2. What kind of logic is described in this file?**
Enums: `RenderMethodEnum` ("text"|"custom"), `TooltipPositionEnum` (4 directions), `ArrowPositionEnum` ("start"|"none"|"end"), `OpenOnEnum` ("click"|"hover"|"hoverFocus"). `TooltipContainerProps`: `trigger: ReactNode`, `renderMethod`, `htmlMessage: ReactNode`, `textMessage: DynamicValue<string>`, `tooltipPosition`, `arrowPosition`, `openOn`, plus system props (name, class, style, tabIndex). `TooltipPreviewProps`: `trigger` and `htmlMessage` are objects with `widgetCount: number` and `renderer: (...) => ReactNode` — enabling dropzone validation and rendering in preview. `textMessage` is plain `string` in preview.

**3. What part of behavior can be documented from this file?**
- `textMessage` is `DynamicValue<string>` at runtime (may be `undefined` during loading).
- `htmlMessage` is a plain `ReactNode` at runtime — the framework renders it before passing it in.
- In preview, both `trigger` and `htmlMessage` use the `{ widgetCount, renderer }` shape for Studio Pro integration.
- `ArrowPositionEnum` uses `"none"` (not `"center"`) to represent centered alignment.

**4. Is it user-facing?**
No — internal type declarations.

**5. What new did you learn from this file?**
In preview mode, `htmlMessage` is not a `ReactNode` but a `{ widgetCount: number; renderer: (...) => ReactNode }` object. This shape is unique to Mendix's editor preview context — the `widgetCount` allows validation logic to check for empty dropzones, while `renderer` provides the dropzone UI rendering function.

---

## src/components/__tests__/Tooltip.spec.tsx

**1. What is the purpose of this file?**
Unit tests for the presentational `Tooltip` component, validating all three open modes, both render methods, and dismiss behavior using Jest and React Testing Library with fake timers.

**2. What kind of logic is described in this file?**
Setup uses `jest.useFakeTimers()`. Tests: (1) snapshot; (2) hover mode — hover opens after 100ms advance, unhover closes, Tab focus does NOT open; (3) click mode — hover does NOT open, focus does NOT open, click opens, second click closes; (4) hoverFocus mode — hover opens, Tab focus opens, blur closes; (5) text rendering — tooltip shows textMessage content; (6) custom rendering — htmlMessage React element rendered; (7) dismiss — click outside closes tooltip.

**3. What part of behavior can be documented from this file?**
- Tests advance timers by 100ms to ensure the 25ms hover open delay passes.
- In hover mode, Tab focus alone does NOT open the tooltip — `hoverFocus` mode is required for keyboard accessibility.
- Click mode has toggle behavior: first click opens, second click closes.
- `role="tooltip"` is the accessibility query target in tests.
- `useDismiss` behavior is verified: clicking `document.body` closes the tooltip.

**4. Is it user-facing?**
No — test file only.

**5. What new did you learn from this file?**
The explicit test confirming Tab focus doesn't open in hover mode is important for accessibility: hover-only tooltips are NOT keyboard accessible. Users relying solely on keyboard navigation cannot access tooltip content unless `hoverFocus` mode is selected. This is an intentional design choice documented by the test, not an oversight.

---

## e2e/Tooltip.spec.js

**1. What is the purpose of this file?**
Playwright end-to-end tests for the Tooltip widget, verifying visual rendering for arrow alignment, tooltip position, flip behavior, custom content, and click mode using screenshot comparisons.

**2. What kind of logic is described in this file?**
Tests across three Mendix pages (`/p/arrow`, `/p/position`, `/p/click`): (1) three arrow alignment variants (start, end, center) — focus buttons, screenshot compare; (2) four position variants (top, left, right, bottom) — focus buttons, screenshot compare; (3) flip behavior — `actionButtonFlip` verifies tooltip switches side when space is constrained; (4) custom content — navigation tree interaction, custom widget renders inside tooltip; (5) click mode — `actionButtonClick.click()` opens tooltip. All screenshots use 10% tolerance. Session logout after each test.

**3. What part of behavior can be documented from this file?**
- Arrow alignment variations (start/center/end) are visually distinct — confirmed by screenshot comparison.
- Flip middleware is tested end-to-end: tooltip moves to opposite side when viewport space is insufficient.
- Custom render mode renders actual widgets inside the tooltip (not just text).
- Focus interaction is used in e2e tests (hoverFocus mode on test buttons), consistent with keyboard accessibility.
- 10% screenshot tolerance allows for minor cross-browser rendering differences.

**4. Is it user-facing?**
The tested behaviors (visual rendering, positioning, interactivity) are user-facing.

**5. What new did you learn from this file?**
The flip test (`tooltipPositionFlipped.png`) provides end-to-end verification that floating-ui's `flip()` middleware functions correctly in a Mendix application context, not just in isolation. This is notable because Mendix's rendering can interfere with overflow detection (scroll containers, z-index stacking), so an e2e flip test adds real confidence.

---

## CHANGELOG.md

**1. What is the purpose of this file?**
Documents version history for the tooltip-web widget from initial release (1.0.0, 2021-12-10) to current version (1.5.1, 2026-02-10).

**2. What kind of logic is described in this file?**
Key versions: v1.5.1 (license/open-source dependency README added), v1.5.0 (fix: scrollbar appearance issue), v1.4.2 (fix: hover flicker → `safePolygon` implementation), v1.4.1 (fix: layout container positioning + trigger width → `fit-content`), v1.4.0 (fix: tooltip overflow off-screen → `flip` + `shift` middleware), v1.3.4 (fix: forced fit-to-content width on tooltip content), v1.3.2 (fix: Escape key not closing tooltip), v1.3.1 (fix: datagrid header positioning, trigger width, disabled input hover), v1.3.0 (dark/light mode icons), v1.2.1 (fix: positioning on specific placement), v1.2.0 (fix: contained widgets losing full width), v1.1.0 (dark mode preview), v1.0.0 (initial release).

**3. What part of behavior can be documented from this file?**
- `safePolygon` was added in v1.4.2 to fix hover flicker — a known UX issue without it.
- `flip` and `shift` middleware were added in v1.4.0 — the widget initially had no viewport overflow handling.
- `fit-content` on both trigger and tooltip content were fixes for layout container interference.
- Escape key support (v1.3.2) was a later addition, not in the initial release.
- Datagrid header and disabled input hover edge cases (v1.3.1) indicate the widget was tested in real Mendix patterns.

**4. Is it user-facing?**
The changelog is publicly visible on the Mendix Marketplace.

**5. What new did you learn from this file?**
The widget went through three distinct maturation phases: (1) initial release (1.0–1.2), (2) positioning robustness (1.3–1.4) adding flip, shift, safePolygon, escape, fit-content, and (3) hardening (1.5+) fixing scrollbar and adding licensing. The `safePolygon` fix (v1.4.2) was critical — without it, moving the mouse from the trigger to the tooltip briefly passes through a gap, triggering hover-off and closing the tooltip before the user can read it.
