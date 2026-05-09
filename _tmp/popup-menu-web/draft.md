# Draft: popup-menu-web

Widget package: `packages/pluggableWidgets/popup-menu-web`

---

## src/PopupMenu.tsx

**1. What is the purpose of this file?**
The root Mendix widget entry point. A thin wrapper that delegates all logic to `PopupMenu` from `./components/PopupMenu`.

**2. What kind of logic is described in this file?**
No logic — single-line passthrough: `return <PopupMenuComponent {...props} />`.

**3. What part of behavior can be documented from this file?**
The widget's behavior is fully contained in `src/components/PopupMenu.tsx`, not in the root file. The root file's only role is to satisfy the Mendix pluggable widget export convention (default export).

**4. Is it user-facing?**
Not directly — all user-facing output comes from the delegated component.

**5. What new did you learn from this file?**
No new behavioral information. Architecture is a standard Mendix pluggable widget pattern with a thin root file.

---

## typings/PopupMenuProps.d.ts

**1. What is the purpose of this file?**
Auto-generated TypeScript typings from `PopupMenu.xml`. Defines all prop types for both runtime (`PopupMenuContainerProps`) and design-mode preview (`PopupMenuPreviewProps`).

**2. What kind of logic is described in this file?**
Enums: `ItemTypeEnum` ("item" | "divider"), `StyleClassEnum` (7 values: defaultStyle/inverseStyle/primaryStyle/infoStyle/successStyle/warningStyle/dangerStyle), `TriggerEnum` ("onclick" | "onhover"), `ClickCloseOnEnum` ("onClickAnywhere" | "onClickOutside"), `HoverCloseOnEnum` ("onClickOutside" | "onHoverLeave"), `PositionEnum` (left/right/top/bottom), `ClippingStrategyEnum` ("absolute" | "fixed").

**3. What part of behavior can be documented from this file?**
- `menuTrigger: ReactNode` — any widget can serve as the popup trigger (a widget slot, not a fixed button type).
- `basicItems: BasicItemsType[]` — each item has `itemType` (item or divider), optional `caption` (DynamicValue<string>), optional `visible` (DynamicValue<boolean>), optional `action`, and `styleClass`.
- `customItems: CustomItemsType[]` — each item has `content: ReactNode` (arbitrary widgets), optional `visible`, optional `action`.
- `menuToggle: boolean` — programmatic control of open/closed state from Mendix expressions.
- `clippingStrategy` — "absolute" or "fixed", controls floating-ui's CSS positioning strategy for overflow handling.
- Two separate close behaviors: `clickCloseOn` (for onclick trigger) and `hoverCloseOn` (for onhover trigger).

**4. Is it user-facing?**
No — internal type declarations.

**5. What new did you learn from this file?**
`menuToggle: boolean` enables programmatic open/close control — a Mendix expression can control popup visibility, making the widget reactive to application state rather than only to user interaction.

---

## src/PopupMenu.xml

**1. What is the purpose of this file?**
Widget descriptor XML defining all props, categories, and defaults. Generates `PopupMenuProps.d.ts`.

**2. What kind of logic is described in this file?**
Declares: required `menuTrigger` widget slot, `basicItems`/`customItems` lists, `trigger` enum (onclick default), `clickCloseOn`/"onClickAnywhere" default, `hoverCloseOn`/"onHoverLeave" default, `position`/bottom default, `clippingStrategy`/absolute default, `menuToggle` boolean/false default, `advancedMode` boolean/false default. Widget is in "Menus & navigation" category.

**3. What part of behavior can be documented from this file?**
- Widget is NOT `offlineCapable` — it requires server connection for actions.
- Default trigger is "onclick"; default position is "bottom"; default clipping strategy is "absolute".
- Default `clickCloseOn` is "onClickAnywhere" — by default, clicking anywhere closes an onclick-triggered menu.
- Default `hoverCloseOn` is "onHoverLeave" — by default, hover menus close when the cursor leaves the menu area.
- `advancedMode` (default false) determines whether `basicItems` or `customItems` is used.
- `menuToggle` defaults to false — not open by default.

**4. Is it user-facing?**
Defines the developer-facing configuration interface in Studio/Studio Pro.

**5. What new did you learn from this file?**
The widget is NOT offline capable — this is notable since many display widgets are. This is expected because popup menu actions (microflows, etc.) require server connectivity.

---

## src/components/PopupMenu.tsx

**1. What is the purpose of this file?**
The main orchestration component. Manages popup open/close state, integrates the `usePopup` floating-ui hook, provides `PopupContext` to children (`PopupTrigger` and `Menu`), and handles item click behavior.

**2. What kind of logic is described in this file?**
- Initializes visibility state from `props.menuToggle`.
- A `useEffect` synchronizes visibility state with `props.menuToggle` changes (allows external programmatic toggle).
- `handleOnClickItem`: closes the menu if `clickCloseOn === "onClickAnywhere"`, then executes the item action via `executeAction`.
- Wraps children in `PopupContext.Provider` so `PopupTrigger` and `Menu` can access floating-ui refs and interaction props.
- DOM: `<div className="popupmenu">` wrapping `<PopupTrigger>{menuTrigger}</PopupTrigger>` and `<Menu {...props} onItemClick={handleOnClickItem} />`.

**3. What part of behavior can be documented from this file?**
- `menuToggle` prop changes are propagated to local state via `useEffect` — external programmatic open/close works.
- When `clickCloseOn === "onClickAnywhere"` (default), clicking any item immediately closes the popup before executing the action.
- When `clickCloseOn === "onClickOutside"`, items can be clicked without closing the menu unless the user clicks outside.
- The popup root DOM element uses CSS class `popupmenu` plus any additional custom `class` from props.
- `dismissal` (clicking outside or pressing Escape) is handled by floating-ui's `useDismiss`, not custom code.

**4. Is it user-facing?**
Yes — produces the container DOM structure and manages popup visibility.

**5. What new did you learn from this file?**
The "onClickAnywhere" close behavior closes the popup synchronously BEFORE the action executes (`setVisibility(false)` then `executeAction(itemAction)`). This means visual feedback (popup disappears) is immediate, even if the action takes time.

---

## src/components/Menu.tsx

**1. What is the purpose of this file?**
Renders the floating popup menu content (the list of items). Uses `FloatingFocusManager` from `@floating-ui/react` for focus trapping and management. Renders nothing when `floatingContext.open` is false.

**2. What kind of logic is described in this file?**
- Returns `null` when `floatingContext.open` is false — menu is completely unmounted when closed.
- `FloatingFocusManager` wraps the menu to manage keyboard focus (tab/arrow navigation).
- DOM: `<div className="widget-popupmenu-root"><ul className="popupmenu-menu">...</ul></div>`.
- In basic mode: items are `<li className="popupmenu-basic-item [style-class]">caption</li>` or `<li className="popupmenu-basic-divider">` for dividers.
- In advanced mode: items are `<li className="popupmenu-custom-item">{content}</li>`.
- `checkVisibility`: item is hidden if `visible?.value === false`. If `visible` property is absent entirely, item is shown.
- Click handlers call `e.preventDefault()` and `e.stopPropagation()` before invoking `handleOnClickItem`.
- Style class is derived by removing "Style" suffix: "inverseStyle" → "popupmenu-basic-item-inverse".
- `floatingStyles` (position CSS from floating-ui) is merged with `props.style` on the `<ul>` element.

**3. What part of behavior can be documented from this file?**
- The menu is **unmounted** (not hidden) when closed — there is no `display:none` toggle; the component returns `null`.
- `visible: DynamicValue<boolean>` with `value: false` hides the item; `visible` being undefined shows the item.
- Divider items (`itemType: "divider"`) display no caption, action, or style class.
- Basic item style classes map to CSS: defaultStyle → no extra class, inverseStyle → `.popupmenu-basic-item-inverse`, primaryStyle → `.popupmenu-basic-item-primary`, etc.
- Keyboard accessibility: `FloatingFocusManager` traps focus within the menu when open, enabling keyboard navigation.
- The `<ul>` element gets `role`, `aria-*`, and interaction props from floating-ui via `getFloatingProps`.

**4. Is it user-facing?**
Yes — renders the visible menu list.

**5. What new did you learn from this file?**
Custom items' `action` is also executed when the `<li>` is clicked — even though custom items contain arbitrary widget content. This means click events bubble to the `<li>` regardless of what's inside it, and the item-level action always fires on click.

---

## src/components/PopupTrigger.tsx

**1. What is the purpose of this file?**
Wraps the `menuTrigger` widget slot in a `<div className="popupmenu-trigger">` that serves as the floating-ui reference element (anchor for positioning) and handles trigger interactions (click/hover) via floating-ui's `getReferenceProps`.

**2. What kind of logic is described in this file?**
- Reads `getReferenceProps`, `open`, and `refs` from `PopupContext` (via `usePopupContext`).
- Merges refs: `refs.setReference` (floating-ui anchor), `propRef` (forwarded ref), and `childrenRef` (children's own ref) via `useMergeRefs`.
- Sets `data-state="open"` or `data-state="closed"` on the trigger div for CSS state styling.
- Click handler calls `e.stopPropagation()` before passing to floating-ui's `getReferenceProps` — prevents click from bubbling to parent elements.

**3. What part of behavior can be documented from this file?**
- The trigger element always gets `data-state` attribute (`"open"` or `"closed"`) — CSS can style the trigger differently based on menu state (e.g., highlight the active trigger button).
- Click propagation is stopped at the trigger level — parent elements do not receive the trigger click event.
- Any widget placed in the `menuTrigger` slot becomes the visual trigger; the wrapper div handles all interaction mechanics.

**4. Is it user-facing?**
Yes — the trigger wrapper is part of the DOM and affects click/hover behavior.

**5. What new did you learn from this file?**
The `data-state` attribute on `.popupmenu-trigger` enables CSS-driven open/closed state visualization — a developer can style the trigger button differently when the menu is open vs. closed using `[data-state="open"]` selectors.

---

## src/hooks/usePopup.ts

**1. What is the purpose of this file?**
Core floating-ui integration hook. Configures positioning, auto-update, and interaction behaviors (click/hover/dismiss) for the popup. Returns refs, styles, and interaction prop getters used by `PopupTrigger` and `Menu`.

**2. What kind of logic is described in this file?**
- `useFloating` with: `middleware: [offset(5), flip(), shift()]`, `strategy: clippingStrategy` (absolute or fixed), `whileElementsMounted: autoUpdate` (repositions when viewport or trigger element changes).
- `useDismiss`: closes on outside click or Escape key.
- `useRole`: adds ARIA role to the floating element.
- `useClick`: enabled only when `trigger === "onclick"`.
- `useHover`: enabled only when `trigger === "onhover"`, with `handleClose: safePolygon()` — prevents accidental close when moving the cursor from trigger to menu.
- Default placement is `"bottom"`.

**3. What part of behavior can be documented from this file?**
- The menu has a 5px offset from its trigger element (`offset(5)` middleware).
- `flip()`: if the menu doesn't fit in the specified position, floating-ui flips it to the opposite side.
- `shift()`: if the menu still overflows after flip, shifts it along the axis to fit within the viewport.
- `autoUpdate`: the menu position is continuously recalculated while both the trigger and menu are mounted — handles scroll and resize.
- `safePolygon()` for hover: creates a virtual safe area between trigger and menu so moving the mouse diagonally doesn't prematurely dismiss the menu.
- Escape key and clicking outside are both handled by `useDismiss` — these close behaviors are always active regardless of `clickCloseOn`/`hoverCloseOn` settings.
- `clippingStrategy === "fixed"` is needed when the popup must escape a parent with `overflow: hidden` — this was the v4.0.0 fix for popup menus being cut off.

**4. Is it user-facing?**
Indirectly — drives all positioning and interaction behavior.

**5. What new did you learn from this file?**
`safePolygon()` is the mechanism behind smooth hover menu behavior — it generates a triangular safe zone between the trigger and the floating menu, so the popup doesn't close when moving the cursor from trigger corner to menu corner.

---

## src/hooks/usePopupContext.ts

**1. What is the purpose of this file?**
A helper hook that reads `PopupContext` and throws an error if the context is null (i.e., the hook is used outside a `PopupContext.Provider`).

**2. What kind of logic is described in this file?**
Calls `useContext(PopupContext)`, checks for null (which would indicate a programming error), and returns the non-null value.

**3. What part of behavior can be documented from this file?**
Both `Menu` and `PopupTrigger` read popup state via this hook — they cannot function without being descendants of `PopupMenu` (which provides `PopupContext`). Attempting to use either component outside the context throws a descriptive error at runtime.

**4. Is it user-facing?**
No — internal hook.

**5. What new did you learn from this file?**
The architecture enforces context dependency explicitly — `Menu` and `PopupTrigger` are not standalone; they are tightly coupled to the `PopupMenu` component tree via context.

---

## src/components/PopupContext.tsx

**1. What is the purpose of this file?**
Defines and exports the `PopupContext` React context with default value `null`. Used to share `UsePopupReturn` state (floating-ui refs, open state, interaction props) from `PopupMenu` down to `Menu` and `PopupTrigger`.

**2. What kind of logic is described in this file?**
`createContext<UsePopupReturn | null>(null)` — no logic beyond context creation.

**3. What part of behavior can be documented from this file?**
Context shares: `context` (floating-ui context object), `floatingStyles` (CSS positioning), `refs` (setReference/setFloating refs), `getFloatingProps`, `getReferenceProps`, `open` (boolean), and `modal`.

**4. Is it user-facing?**
No — internal plumbing.

**5. What new did you learn from this file?**
The context pattern allows `Menu` and `PopupTrigger` to be independent sibling components under `PopupMenu` — neither is a child of the other, yet both share floating-ui state through context rather than prop drilling.

---

## src/utils/attrValue.ts

**1. What is the purpose of this file?**
Test utilities for creating Mendix value mocks: `dynamicValue<T>()` creates a `DynamicValue` with `ValueStatus.Available` and `actionValue()` creates a mock `ActionValue`.

**2. What kind of logic is described in this file?**
Pure test helpers — no runtime logic. Used in unit tests to build prop values without needing a real Mendix runtime.

**3. What part of behavior can be documented from this file?**
No behavioral information beyond test infrastructure. These helpers confirm that the widget props use standard Mendix types (`DynamicValue`, `ActionValue`).

**4. Is it user-facing?**
No — test utilities only.

**5. What new did you learn from this file?**
Nothing new about behavior. Standard Mendix widget test utility pattern.

---

## src/PopupMenu.editorConfig.ts

**1. What is the purpose of this file?**
Provides `getProperties` (prop visibility), `check` (validation), and `getPreview` (structure preview) for Studio/Studio Pro editor.

**2. What kind of logic is described in this file?**
- `getProperties`: In basic mode, hides `customItems` and all sub-props for divider items (caption, action, styleClass, visible all hidden for dividers). In advanced mode, hides `basicItems`. `hoverCloseOn` is hidden when trigger is not "onhover"; `clickCloseOn` hidden when trigger is not "onclick". `menuToggle` is hidden on desktop (Studio Pro).
- `check`: In basic mode, validates at least one item exists and all "item" type entries have a caption. In advanced mode, validates at least one custom item exists.
- `getPreview`: Returns `null` in advanced mode (no preview for custom items). In basic mode, shows a styled container with the trigger drop zone and each basic item rendered as a selectable row.

**3. What part of behavior can be documented from this file?**
- Divider items in Studio have all their properties hidden — no caption, no action, no visibility, no style class. A divider is purely a visual separator.
- `menuToggle` prop is not exposed in Studio Pro (only in Studio) — programmatic toggle is a web-only Studio feature.
- The structure preview is clickable (items are `Selectable`) — clicking an item in Studio's structure view selects it for editing.
- In advanced mode, structure preview returns null — advanced mode items are not previewable in structure mode.

**4. Is it user-facing?**
Yes — visible to developers in Studio/Studio Pro.

**5. What new did you learn from this file?**
The `styleClass` property description in the editor is dynamically updated to show the corresponding CSS class name (e.g., `"popupmenu-basic-item-inverse"`). This gives developers the CSS class reference directly in the editor description field, without needing to consult documentation.

---

## src/PopupMenu.editorPreview.tsx

**1. What is the purpose of this file?**
Provides the React-based design-mode canvas preview for Studio.

**2. What kind of logic is described in this file?**
Renders the trigger widget via `props.menuTrigger` renderer, then in advanced mode renders custom item content in a `popupmenu-menu` container. In basic mode, no menu items are shown in design view. Uses `parseStyle` for style prop handling.

**3. What part of behavior can be documented from this file?**
- In design mode, basic items are not rendered in the preview — only the trigger is visible.
- In advanced mode, custom widget content is rendered below the trigger in the design canvas.
- This was fixed in v4.1.0 (previously advanced mode didn't show options content in Design mode).

**4. Is it user-facing?**
Yes — visible to developers in Studio design canvas.

**5. What new did you learn from this file?**
Advanced mode design-mode preview renders custom item content directly (not floating) — the preview is static and does not replicate the floating popup behavior. It shows trigger and items in a stacked layout.

---

## src/__tests__/Menu.spec.tsx

**1. What is the purpose of this file?**
Unit tests for the `Menu` component covering: basic rendering, action triggering, visibility filtering, all style classes, and custom item mode.

**2. What kind of logic is described in this file?**
Tests confirm:
- 1 basic item renders (excluding the divider) when `advancedMode: false`.
- Clicking a basic item calls `onItemClick`.
- A hidden item (`visible: { value: false }`) is not rendered.
- Each of 6 non-default style classes renders the correct CSS class.
- In advanced mode: 1 custom item renders.
- Hidden custom item is not rendered.
- Clicking a custom item calls `onItemClick`.

**3. What part of behavior can be documented from this file?**
- Dividers are excluded from the count of `popupmenu-basic-item` elements — they render as `popupmenu-basic-divider`.
- A hidden visible attribute (`value: false`) removes the item from DOM entirely — not hidden via CSS.
- All 6 non-default style classes are e2e-confirmed to produce distinct CSS class names.
- Custom items fire `onItemClick` on click — even custom content items have click-triggered actions.

**4. Is it user-facing?**
No — test file only.

**5. What new did you learn from this file?**
The `MenuWithContext` test helper wraps `Menu` with a `PopupContext.Provider` open state — confirming that the `Menu` component checks `floatingContext.open` before rendering. The context must be set to open=true for menu items to appear in tests.

---

## e2e/PopupMenu.spec.js

**1. What is the purpose of this file?**
End-to-end Playwright tests for the popup menu widget. Tests all 8 position configurations (top-left, left, top, top-right, right, bottom-right, bottom-left, bottom) for both basic and custom item modes, plus hover trigger and item click behavior.

**2. What kind of logic is described in this file?**
Screenshot baseline tests for each position/mode combination. Hover test: triggers hover on a button, then hovers the visible menu text, confirms text "Gooooooo" is visible. Click test: opens menu via button click, clicks first `.popupmenu-basic-item`, verifies modal dialog with "hello" text appears.

**3. What part of behavior can be documented from this file?**
- All 8 positions (combinations of top/bottom/left/right and their diagonals via floating-ui flip/shift) are e2e-confirmed.
- Both basic and custom item modes are e2e-confirmed visually.
- Hover trigger (`onhover`) is e2e-confirmed — moving hover from trigger to menu text does not close the menu (safePolygon behavior confirmed).
- Clicking a basic item fires a Mendix action that opens a modal dialog — action execution is e2e-confirmed.
- Custom item click also fires an action opening a modal with "hello" text — confirmed from the custom items test.
- Screenshot threshold is 0.1 (tight baseline — minor differences will fail).

**4. Is it user-facing?**
The tested behaviors (positioning, hover, click actions) are user-facing.

**5. What new did you learn from this file?**
The e2e test tests all 8 positional configurations explicitly — confirming floating-ui's flip/shift middleware produces correct positioning in all directions. The diagonal positions (top-left, top-right, bottom-left, bottom-right) are provided by floating-ui's placement system where available.

---

## CHANGELOG.md

**1. What is the purpose of this file?**
Version history from v3.0.0 to v4.1.0.

**2. What kind of logic is described in this file?**
No logic — version history.

**3. What part of behavior can be documented from this file?**
Key behavioral changes:
- Unreleased: Added "Close on" option for Click trigger — choosing between close on click anywhere vs. outside click only.
- v4.1.0 (2026-04-13): Fixed Advanced options mode not showing content in Design mode.
- v4.0.2 (2025-02-20): Fixed user click interaction triggering overlaid elements.
- v4.0.1 (2024-12-13): "Area to open or close the menu" (menuTrigger) became required.
- v4.0.0 (2024-11-13): Moved styling to Atlas core (breaking — requires atlas-core 3.15.0+); added clipping strategy option; fixed popup not overflowing parent.
- v3.6.1 (2024-09-18): Fixed popup getting cut off by overflow parent.
- v3.6.0 (2024-08-20): Added configurable "Close on" setting for hover trigger. Breaking: hover default close behavior changed to "hover leave".
- v3.5.0 (2023-06-28): Changed DOM from `div` to `ul`/`li` for accessibility; moved `popupmenu-menu` to trigger sibling level (previously nested — caused z-index issues behind modal popups).
- v3.1.0 (2021-11-18): Added `visible` property on popup items.
- v3.0.0 (2021-09-28): Renamed "custom visualization" to "enable advanced options".

**4. Is it user-facing?**
Publicly visible on Mendix Marketplace.

**5. What new did you learn from this file?**
v3.5.0's DOM restructuring (moving `popupmenu-menu` to sibling of trigger button) was necessary to fix z-index layering — when the menu was inside the trigger's DOM tree, it could appear behind modal popups. This architectural change means custom CSS that previously targeted the menu as a descendant of the trigger may need updating.
