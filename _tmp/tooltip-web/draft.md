# tooltip-web — Draft Spec

Widget: `tooltip-web`
Package: `packages/pluggableWidgets/tooltip-web/`
Agent: worker
Date: 2026-05-09

---

## src/Tooltip.tsx

**1. What is the purpose of this file?**
The root Mendix pluggable widget entry component. Maps XML props to the presentation component, calling `translatePosition` to combine `tooltipPosition` + `arrowPosition` into a single Floating UI `Placement` string.

**2. What kind of logic is described in this file?**
- `translatePosition(props.tooltipPosition, props.arrowPosition)` → Floating UI `Placement` (e.g., `"top-start"`, `"bottom"`, `"right-end"`).
- `textMessage?.value` — unwraps the `DynamicValue<string>` to a plain string; undefined if unset.
- Passes all props directly to `<DisplayTooltip>` with no additional logic.

**3. What part of behavior can be documented from this file?**
- `textMessage` is extracted as `.value` — uses the evaluated string value; if the attribute or expression is unavailable, it is `undefined`.
- No conditional rendering, loading states, or editability logic here — the tooltip is purely display-only (no user data input).

**4. Is it user-facing?**
No — internal Mendix adapter.

**5. What new did you learn from this file?**
The position is a two-part combination: `tooltipPosition` (side: top/right/bottom/left) + `arrowPosition` (alignment: start/center/end). This maps cleanly to Floating UI's placement notation: `"top-start"`, `"top"` (center, no suffix), `"top-end"`.

---

## src/Tooltip.xml

**1. What is the purpose of this file?**
Mendix widget descriptor. Defines the tooltip as a Display-category widget with a widget trigger slot, two content render modes, positioning, and trigger mode options.

**2. What kind of logic is described in this file?**
Properties (all in a single "General" group):
- `trigger`: widgets slot — **required**. The UI element the user interacts with to show the tooltip.
- `renderMethod`: text | custom (default: text). Selects between plain text or widget-based tooltip content.
- `htmlMessage`: widgets slot, optional. Used when `renderMethod === "custom"`.
- `textMessage`: text template, optional. Used when `renderMethod === "text"`.
- `tooltipPosition`: left | right | top (default) | bottom. Which side of the trigger the tooltip appears.
- `arrowPosition`: start | none (default, labeled "Center") | end. Where along the tooltip the arrow points.
- `openOn`: click | hover (default) | hoverFocus. What interaction triggers the tooltip.

No system properties explicitly listed (no `<systemProperty>` elements), but the framework implicitly provides `name`, `class`, `style`, `tabIndex`.

**3. What part of behavior can be documented from this file?**
- `needsEntityContext="true"`, `offlineCapable="true"`.
- Category: Display (both studioProCategory and studioCategory).
- `trigger` is required — a tooltip without a trigger is invalid.
- `htmlMessage` and `textMessage` are both optional at the XML level — the `check` function in editorConfig enforces that one is present based on `renderMethod`.
- The `arrowPosition: "none"` key is labeled "Center" in Studio Pro — the enum key `"none"` means "no alignment modifier" (i.e., center-aligned).
- Default position is `"top"`, default open trigger is `"hover"`.

**4. Is it user-facing?**
No — Studio Pro configuration descriptor.

**5. What new did you learn from this file?**
The `arrowPosition` enum uses `"none"` as the key for center alignment. This is because Floating UI's center placement has no suffix (`"top"` not `"top-center"`), so the "none" key cleanly represents "no alignment modifier" while the Studio Pro label says "Center" for clarity.

---

## typings/TooltipProps.d.ts

**1. What is the purpose of this file?**
Auto-generated TypeScript types from the XML descriptor.

**2. What kind of logic is described in this file?**
- `RenderMethodEnum = "text" | "custom"`.
- `TooltipPositionEnum = "left" | "right" | "top" | "bottom"`.
- `ArrowPositionEnum = "start" | "none" | "end"`.
- `OpenOnEnum = "click" | "hover" | "hoverFocus"`.
- `TooltipContainerProps`: `name`, `class`, `style?: CSSProperties`, `tabIndex?: number`, `trigger: ReactNode`, `renderMethod`, `htmlMessage?: ReactNode`, `textMessage?: DynamicValue<string>`, `tooltipPosition`, `arrowPosition`, `openOn`.
- `TooltipPreviewProps.trigger` and `htmlMessage` are `{ widgetCount: number; renderer: ComponentType<...> }` — widget slot preview objects.

**3. What part of behavior can be documented from this file?**
- `htmlMessage` is optional in runtime (`ReactNode?`) — no content renders if not provided.
- `textMessage` is `DynamicValue<string>` (not `EditableValue`) — read-only expression or text template.
- Container props have `class` and `style` — CSS class customization is supported.

**4. Is it user-facing?**
No — TypeScript types only.

**5. What new did you learn from this file?**
`htmlMessage` is typed as `ReactNode` (not a widget slot object) in the runtime props — the Mendix framework renders widget slot contents to ReactNode before passing them to the component. This is different from the preview props where it's a `renderer` object.

---

## src/components/Tooltip.tsx

**1. What is the purpose of this file?**
The presentation component — renders the trigger wrapper, floating tooltip content, and directional arrow using `@floating-ui/react`.

**2. What kind of logic is described in this file?**
DOM structure:
- `<div class="widget-tooltip widget-tooltip-{position} {props.class}">` — outer container, position CSS class.
  - `<div class="widget-tooltip-trigger" ref={refs.setReference} {...getReferenceProps(blurFocusEvents?)}>` — the trigger area; non-interactive in `preview` mode (no props attached).
    - `{trigger}` — the widget slot content.
  - (conditional) `<div class="widget-tooltip-content" ref={refs.setFloating} style={floatingStyles} {...getFloatingProps()}>` — the floating tooltip panel; only rendered when `showTooltip && (textMessage || htmlMessage)`.
    - `renderMethod === "text"` → renders `{textMessage}` (string)
    - `renderMethod === "custom"` → renders `{htmlMessage}` (ReactNode)
    - `<div class="widget-tooltip-arrow-{staticSide}" ref={setArrowElement} style={arrowStyles} />` — arrow indicator.

State:
- `showTooltip`: `useState(preview ?? false)` — `true` in preview mode (always visible), `false` in runtime (controlled by floating UI).
- `arrowElement`: DOM ref for the arrow div (passed to Floating UI `arrow` middleware).

`hoverFocus` trigger mode: `blurFocusEvents` (`{onFocus, onBlur}`) are spread onto the trigger div — focus/blur on the trigger open/close the tooltip directly.

**3. What part of behavior can be documented from this file?**
- Tooltip position CSS class is added to the outer wrapper: `"widget-tooltip-top"`, `"widget-tooltip-right"`, etc.
- In preview mode (`preview={true}`): tooltip is always shown (no interactions) — used by `editorPreview.tsx`.
- Tooltip content is not rendered at all (null) when `showTooltip` is false — no hidden content in DOM.
- Arrow class is `"widget-tooltip-arrow-{staticSide}"` where `staticSide` is the opposite of the placement side (e.g., `"bottom"` for `"top"` placement).
- `hoverFocus` mode uses separate focus/blur events on the trigger in addition to floating UI's hover handling.

**4. Is it user-facing?**
Yes — this is the visible tooltip component.

**5. What new did you learn from this file?**
`hoverFocus` is implemented by spreading `blurFocusEvents` (`onFocus`/`onBlur`) directly onto the trigger div, not by using floating UI's built-in focus interaction. This is intentional: floating UI's `useFocus` provides focus-on-the-reference behavior, but the `blurFocusEvents` also need to close on blur (not just on outside click/dismiss). The two systems work in parallel for `hoverFocus` mode.

---

## src/utils/useFloatingUI.ts

**1. What is the purpose of this file?**
Custom hook that encapsulates all `@floating-ui/react` logic — positioning, middleware, interactions, and arrow calculation.

**2. What kind of logic is described in this file?**
`useFloating` configuration:
- `strategy: "fixed"` — positions floating element relative to the viewport, avoids scroll offset issues.
- `placement: position` — user-configured side + alignment.
- Middleware pipeline:
  1. `offset(8)` — 8px gap between trigger and tooltip.
  2. `flip({ fallbackPlacements: ["top", "right", "bottom", "left"] })` — repositions to opposite side if no space.
  3. `shift()` — keeps tooltip within viewport bounds by sliding along its axis.
  4. `arrow({ element: arrowElement })` — computes arrow position within the tooltip.
- `whileElementsMounted: autoUpdate` — keeps position updated on scroll/resize.

Interactions:
- `useHover`: enabled for `"hover"` and `"hoverFocus"`. `move: false` (no movement events), open delay 25ms, `restMs: 25`, `safePolygon()` close handler (prevents flicker when moving between trigger and tooltip).
- `useFocus`: enabled for `"hoverFocus"` only.
- `useClick`: enabled for `"click"` mode, `toggle: showTooltip` (click again to close).
- `useDismiss`: `outsidePress: true` — click outside closes the tooltip for all modes.
- `useRole`: ARIA `role="tooltip"` on the floating element.

Arrow position calculation:
- `staticSide`: opposite of current placement side (`"top"` → `"bottom"`, etc.).
- `alignmentOffset`: for `"start"`-aligned placements on top/bottom sides, shifts arrow by `arrowElement.offsetWidth` to align with the start edge.
- `arrowStyles`: `{ left: arrowX - alignmentOffset, top: arrowY }` from `middlewareData.arrow`.

Returns: `{ arrowStyles, blurFocusEvents, floatingStyles, getFloatingProps, getReferenceProps, refs, staticSide }`.

**3. What part of behavior can be documented from this file?**
- Hover has a 25ms open delay (`restMs: 25`) — prevents accidental tooltip triggers on quick mouse passes.
- `safePolygon()` close handler: after hover, moving the mouse toward the tooltip doesn't close it prematurely — a triangular "safe zone" between trigger and tooltip is maintained.
- `flip` fallback order: top → right → bottom → left. If configured position has no space, it tries all four sides.
- Outside press dismisses for all trigger modes — click-opened tooltips also close on outside click.
- `"fixed"` strategy: tooltip position is relative to viewport, not page — avoids clipping inside overflow-hidden containers.

**4. Is it user-facing?**
No — internal hook.

**5. What new did you learn from this file?**
The hover delay (`delay: { open: 25, close: 0 }` + `restMs: 25`) is carefully tuned: the 25ms rest time prevents triggers from accidental hover-through, while the 0ms close delay means the tooltip disappears immediately on unhover. The `safePolygon()` handler ensures users can move from the trigger to the tooltip content without the tooltip closing (needed when the tooltip contains interactive widgets like links).

---

## src/utils/index.ts

**1. What is the purpose of this file?**
Exports the `translatePosition` utility function.

**2. What kind of logic is described in this file?**
`translatePosition(tooltipPosition, arrowPosition)`: concatenates `tooltipPosition` with `"-" + arrowPosition` (unless `arrowPosition === "none"` — in that case no suffix). Returns a Floating UI `Placement` string.
- `("top", "none")` → `"top"`
- `("top", "start")` → `"top-start"`
- `("right", "end")` → `"right-end"`

**3. What part of behavior can be documented from this file?**
- Arrow position "Center" (XML `"none"`) maps to the Floating UI default placement (no alignment modifier).
- The function is used in both the runtime component and the editor preview.

**4. Is it user-facing?**
No — utility function.

**5. What new did you learn from this file?**
The mapping is straightforward: Floating UI's 12 placement values (4 sides × 3 alignments) are exposed to Studio Pro users as two separate dropdowns (position + arrow position), then merged here into a single string. The XML `"none"` key for center alignment avoids adding a `"-center"` suffix that Floating UI doesn't use.

---

## src/Tooltip.editorConfig.ts

**1. What is the purpose of this file?**
Studio Pro property visibility, validation, and structure preview for the Tooltip widget.

**2. What kind of logic is described in this file?**
`getProperties`:
- `renderMethod === "text"` → hides `htmlMessage` property.
- `renderMethod === "custom"` → hides `textMessage` property.

`check` validation:
- `renderMethod === "text" && !textMessage` → error on `textMessage`: "For render method Text, a Tooltip message is required".
- `renderMethod === "custom" && htmlMessage.widgetCount === 0` → error on `htmlMessage`: "For render method custom, a Content is required".

`getPreview` structure: 3-part vertical layout:
1. Header bar (`topbarStandard` background, border): label text "Tooltip".
2. Content area (bordered): for `"text"` mode — centered text showing `textMessage` (dark, bold, 14px) or placeholder grey text "Place your tooltip message"; for `"custom"` mode — a `DropZone` for the `htmlMessage` widget slot.
3. Trigger drop zone: labeled "Place widget(s) here".

`centerLayout` helper: creates a 3-column row (grow:99, grow:1, grow:99) to horizontally center the text prop — centering trick using spacer containers.

**3. What part of behavior can be documented from this file?**
- Studio Pro validates that content is always present for the selected render method — a tooltip with neither text nor custom content shows an error.
- `htmlMessage.widgetCount === 0` (not `!htmlMessage`) is the check — the widget slot always exists but must have at least one widget inside it.
- The structure preview shows the tooltip content above the trigger — reversed from actual rendering (trigger is the visible element, tooltip floats above/around it).
- Text preview uses 14px font and `#000000`/`#DEDEDE` (dark/light) for configured text; `#6B707B`/`#A4A4A4` for placeholder.

**4. Is it user-facing?**
No — Studio Pro only.

**5. What new did you learn from this file?**
The `centerLayout` helper centers content using flex-grow spacers (grow:99 / grow:1 / grow:99) — a technique used because the Studio Pro structure preview API doesn't support CSS flexbox centering directly. The 99:1:99 ratio effectively centers the 1-unit content between two ~50% spacers.

---

## src/Tooltip.editorPreview.tsx

**1. What is the purpose of this file?**
Live React preview in Studio Pro design mode — renders the `Tooltip` component with `preview={true}` to show the tooltip open.

**2. What kind of logic is described in this file?**
- Renders `<Tooltip preview={true} ...>` — the `preview` prop causes the tooltip content to be visible by default.
- `htmlMessage` rendered via `props.htmlMessage.renderer` (widget slot preview renderer).
- `trigger` rendered via `props.trigger.renderer`.
- `getPreviewCss()` exports the SCSS (makes styles apply in design mode).
- Uses `parseStyle(props.style)` to convert style string to `CSSProperties` object.

**3. What part of behavior can be documented from this file?**
- Design mode always shows the tooltip open (floating content visible), using the same `Tooltip` component with `preview={true}`.
- The `preview` flag bypasses all floating UI event handlers — no position updates, no hover/click logic.
- Widget slots in both `trigger` and `htmlMessage` are rendered via Mendix's preview renderer system.

**4. Is it user-facing?**
No — Studio Pro design mode preview only.

**5. What new did you learn from this file?**
The `getPreviewCss()` export (line 32) is the mechanism for injecting widget SCSS into Studio Pro's live preview — without it, the tooltip would render unstyled in design mode. This is a separate code path from the `import "./ui/Tooltip.scss"` in the entry component (which bundles CSS for the runtime widget).

---

## src/ui/Tooltip.scss

**1. What is the purpose of this file?**
All visual styles for the tooltip widget.

**2. What kind of logic is described in this file?**
- `.widget-tooltip-trigger`: `width: fit-content` — trigger wrapper doesn't stretch to fill parent width (prevents entire row from becoming the hover area).
- `.widget-tooltip-content`:
  - Color: `#24276c` (dark navy blue) text, white background.
  - Typography: `font-weight: bold`, `font-size: 14px`.
  - Shape: `border-radius: 3px`, `padding: 6px`.
  - Elevation: `box-shadow: 0 0 5px rgba(0,0,0,0.3)`.
  - Stacking: `z-index: 50`.
  - Text wrapping: `white-space: break-spaces`, `word-break: break-word`.
- Arrow (`.widget-tooltip-arrow-*`):
  - 8px × 8px div with `background: inherit` (matches tooltip background).
  - The actual arrow visual is the `::before` pseudo-element: `visibility: visible`, `transform: rotate(45deg)`, 8×8px rotated square.
  - The arrow div itself is `visibility: hidden` — only `::before` is visible.
  - Positioned per side: `left: -8px` for left-side arrow, `bottom: -3px` for bottom arrow, `top: -4px` for top arrow.

**3. What part of behavior can be documented from this file?**
- Tooltip text is dark navy (`#24276c`), bold, 14px — distinct, readable style.
- Arrow is a rotated square pseudo-element, not an SVG — purely CSS.
- `fit-content` on trigger prevents the hover zone from expanding beyond the trigger content's natural width.
- `z-index: 50` — tooltip floats above most page content.
- `word-break: break-word` — long words wrap within the tooltip.

**4. Is it user-facing?**
Yes — controls all visual appearance of the tooltip.

**5. What new did you learn from this file?**
The arrow `visibility: hidden` on the actual element + `visibility: visible` on `::before` is a CSS trick: the actual div provides a box model reference for Floating UI's arrow positioning calculations, but visually only the pseudo-element (the rotated square) appears. This separates the layout space from the visual rendering.

---

## src/components/__tests__/Tooltip.spec.tsx

**1. What is the purpose of this file?**
Unit tests for the `Tooltip` presentation component — tests all three trigger modes with `userEvent`.

**2. What kind of logic is described in this file?**
- `openOn: "hover"`: hover → tooltip appears (after 100ms timer advance); unhover → tooltip disappears; focus (Tab) → no tooltip; blur → no tooltip.
- `openOn: "click"`: hover → no tooltip; focus → no tooltip; click → tooltip appears; second click → tooltip disappears (toggle); outside click → tooltip disappears.
- `openOn: "hoverFocus"`: hover → tooltip (after 100ms); unhover → tooltip gone; Tab focus → tooltip appears; Tab blur → tooltip disappears.
- `renderMethod: "text"`: tooltip has `textContent === textMessage`.
- `renderMethod: "custom"`: `htmlMessage` content is rendered.
- Confirmed: `role="tooltip"` on floating element.

**3. What part of behavior can be documented from this file?**
- Hover: requires timer advancement (25ms delay) — `jest.advanceTimersByTime(100)` used.
- Click mode: toggle behavior confirmed (click opens, click again closes).
- `hoverFocus`: focus alone (without hover) opens tooltip; blur closes it.
- `hoverFocus`: hover also opens tooltip (both mechanisms active).
- Outside click closes tooltip even in `"click"` mode.

**4. Is it user-facing?**
No — test file.

**5. What new did you learn from this file?**
In `openOn: "hover"` mode, Tab/focus does NOT open the tooltip — focus is only active in `"hoverFocus"` mode. This confirms the two focus handling systems are separate: `useFocus` (hoverFocus mode only) vs. `blurFocusEvents` (hoverFocus additional behavior). Also: `useFakeTimers` is needed because the 25ms hover delay is timer-based.

---

## e2e/Tooltip.spec.js

**1. What is the purpose of this file?**
Playwright E2E tests verifying tooltip visual rendering and positioning via screenshots (10% threshold).

**2. What kind of logic is described in this file?**
Arrow position tests (focus-triggered, `/p/arrow` page):
- Arrow start, center, end — 3 screenshot comparisons.

Position tests (focus-triggered, `/p/position` page):
- Top, left, right, bottom positions.
- Flip test: tooltip configured as "left" but near screen edge — verifies `flip` middleware repositions it to the other side.

Custom render test: navigates to custom tab, focuses custom-content button.
Click test (`/p/click` page): clicks trigger, takes screenshot.

All screenshots use `threshold: 0.1` (10% pixel difference tolerance) — stricter than time-series-chart tests.

**3. What part of behavior can be documented from this file?**
- E2E uses `.focus()` to trigger `hoverFocus` tooltips — confirming focus opens tooltips in production.
- Flip behavior confirmed: "left" tooltip near screen edge flips to opposite side (right side).
- Custom render mode shows widget slot content in the tooltip.
- Click-triggered tooltip confirmed to open on `.click()`.

**4. Is it user-facing?**
No — test file.

**5. What new did you learn from this file?**
The flip test (`.mx-name-actionButtonFlip`) is a critical behavioral test: it verifies that `@floating-ui/react`'s `flip` middleware works in a real Mendix app. The tooltip configured as "left" but placed near the left screen edge will appear on the right instead — this is automatic and requires no user configuration.

---

## CHANGELOG.md

**1. What is the purpose of this file?**
Version history from v1.0.0 (initial release).

**2. What kind of logic is described in this file?**
- **v1.5.1 (2026-02-10)**: Added license file and open source dependency readme.
- **v1.5.0 (2026-02-03)**: Fixed scrollbars appearing in some cases.
- **v1.4.2 (2024-10-31)**: Fixed tooltip content flickering on hover.
- **v1.4.1 (2024-10-09)**: Fixed placement inside layout containers (width of trigger container).
- **v1.4.0 (2024-08-14)**: Fixed tooltip overflowing off screen.
- **v1.3.4 (2024-05-23)**: Fixed content forced to fit-content width.
- **v1.3.3 (2023-09-27)**: Removed redundant code for load time improvement.
- **v1.3.2 (2023-08-10)**: Fixed tooltip not closing on Escape key press.
- **v1.3.1 (2023-08-03)**: Fixed display near datagrid table header; fixed unexpectedly wide trigger element; fixed hover not closing on disabled input.
- **v1.3.0 (2023-06-06)**: Updated icons/tiles; changed structure preview colors for dark/light mode.
- **v1.2.1 (2022-08-30)**: Fixed positioning issue on specific placements.
- **v1.2.0 (2022-05-10)**: Fixed contained widgets rendering without full width.
- **v1.1.0 (2021-12-23)**: Added dark mode for structure preview; dark icons.
- **v1.0.0 (2021-12-10)**: Initial release.

**3. What part of behavior can be documented from this file?**
- v1.4.2 "flicker on hover" fix — likely the `safePolygon()` handler or hover delay tuning.
- v1.4.1 trigger width fix — led to `width: fit-content` on the trigger div (current SCSS).
- v1.4.0 overflow fix — likely improved `shift` or `flip` middleware configuration.
- v1.3.2 Escape close — `useDismiss` handles Escape key in `@floating-ui/react`.
- v1.3.1 disabled input hover fix — floating UI's handling of pointer events on disabled elements.

**4. Is it user-facing?**
No — developer changelog.

**5. What new did you learn from this file?**
The history reveals this widget has had numerous positioning edge cases. Many fixes relate to `@floating-ui/react` integration challenges: overflow issues, flicker on hover, trigger width, datagrid headers. The current implementation (fixed strategy, safePolygon, fit-content trigger width) is the result of iterating through these real-world edge cases.

---

## Summary of Key Findings

- **Purpose**: Floating tooltip that appears on hover, click, or hover+focus. Supports plain text or custom widget content. The trigger is a widget slot — any Mendix widget can be a tooltip trigger.
- **Library**: `@floating-ui/react` for all positioning and interaction logic. Strategy: `"fixed"` (viewport-relative).
- **Positioning**: 4 sides (top/right/bottom/left) × 3 arrow alignments (start/center/end) = 12 possible placements. Automatic `flip` to opposite side when no space available. `shift` keeps tooltip in viewport.
- **Arrow**: CSS-only — rotated square `::before` pseudo-element, hidden parent div used for Floating UI offset calculation.
- **Trigger modes**:
  - `hover`: opens after 25ms delay; `safePolygon()` prevents close when moving mouse to tooltip. Does NOT open on focus.
  - `click`: toggle — click opens, click again or outside click closes. Does NOT open on hover/focus.
  - `hoverFocus`: opens on hover OR focus; closes on unhover or blur.
- **Render methods**: `text` (plain text template) | `custom` (widget slot).
- **CSS**: Text color `#24276c` (navy), white background, bold 14px, `z-index: 50`, `width: fit-content` trigger.
- **Preview**: In Studio Pro design mode, tooltip is always shown open (`preview={true}` prop bypasses events).
- **Validation**: Studio Pro validates that content is present for the selected render method.
- **offlineCapable**: `true`.
- **Testing**: Unit tests cover all 3 trigger modes + render methods; E2E tests use focus/click triggers with screenshot comparisons (0.1 threshold). Flip behavior tested E2E.
