# Popup Menu

## Purpose

The Popup Menu widget displays a floating contextual menu anchored to an arbitrary trigger widget. It supports both basic items (styled text entries with optional Mendix actions) and advanced/custom items (arbitrary Mendix widget content). The menu can be triggered by click or hover, positioned in eight directions relative to its trigger, and controlled programmatically via a Mendix expression. It is suited for contextual navigation, action menus, and dropdown-style interactions.

## User Scenarios

### [P1] Open a basic menu on click
**Given** a Popup Menu widget configured with basic items and a click trigger  
**When** the user clicks the trigger widget  
**Then** a floating menu appears below the trigger (default position), listing the configured items as styled list entries

#### Edge Cases
- The menu is unmounted (not hidden via CSS) when closed; it is remounted on each open.
- Click propagation is stopped at the trigger element — parent elements do not receive the click event.
- The trigger element receives `data-state="open"` or `data-state="closed"` for CSS state styling.

### [P2] Close the menu after item selection
**Given** a menu configured with `clickCloseOn === "onClickAnywhere"` (default)  
**When** the user clicks any menu item  
**Then** the menu closes immediately (synchronously), then the configured item action executes

#### Edge Cases
- When `clickCloseOn === "onClickOutside"`, clicking an item does not close the menu — the user must click outside or press Escape.
- Escape key and clicking outside always close the menu regardless of `clickCloseOn` setting (handled by floating-ui `useDismiss`).

### [P3] Hover-triggered menu with safe polygon
**Given** a Popup Menu configured with `trigger === "onhover"`  
**When** the user hovers over the trigger widget  
**Then** the menu appears; moving the cursor diagonally toward the menu does not prematurely close it (floating-ui `safePolygon` creates a safe zone between trigger and menu)

#### Edge Cases
- When `hoverCloseOn === "onHoverLeave"` (default), the menu closes when the cursor leaves the menu area.
- When `hoverCloseOn === "onClickOutside"`, the menu remains open until the user clicks outside.

### [P4] Programmatic open/close via expression
**Given** `menuToggle` is bound to a Mendix boolean expression  
**When** the expression value changes to `true`  
**Then** the menu opens; when it changes to `false`, the menu closes

#### Edge Cases
- `menuToggle` changes are propagated to local open/close state via `useEffect`.
- This feature is only available in Studio (web); it is hidden in Studio Pro.

### [P5] Advanced mode with custom widget content
**Given** `advancedMode` is enabled and custom items are configured with widget content  
**When** the menu is open  
**Then** each item renders the configured widget content inside a `<li>` element; clicking the `<li>` fires the item-level action regardless of what widget is inside

#### Edge Cases
- In basic mode, `customItems` are hidden and unused; in advanced mode, `basicItems` are hidden.
- Advanced mode items have no style class, caption, or visibility from the basic item model.
- In Studio design mode, basic mode preview shows only the trigger; advanced mode renders custom widget content stacked below the trigger (not floating).

### [P6] Position in all eight directions
**Given** a menu with `position` set to top, bottom, left, or right  
**When** the menu opens  
**Then** it appears on the specified side of the trigger; if there is insufficient viewport space, floating-ui `flip()` moves it to the opposite side, then `shift()` shifts it along the axis to remain within the viewport

#### Edge Cases
- All eight positional configurations (top-left, left, top, top-right, right, bottom-right, bottom-left, bottom) are e2e-confirmed.
- The menu has a 5px gap from its trigger (`offset(5)` middleware).
- `clippingStrategy: "fixed"` must be used when the menu must escape a parent element with `overflow: hidden`.

## Functional Requirements

- FR-001: The menu MUST use `@floating-ui/react` for positioning, with `offset(5)`, `flip()`, and `shift()` middleware active.
- FR-002: The menu MUST be unmounted (return `null`) when closed — not hidden via CSS `display: none`.
- FR-003: When `trigger === "onhover"`, `safePolygon()` MUST be applied as the `handleClose` for `useHover` to prevent premature dismissal during diagonal cursor movement.
- FR-004: Escape key and outside clicks MUST always dismiss the menu via `useDismiss`, regardless of `clickCloseOn`/`hoverCloseOn` settings.
- FR-005: The trigger wrapper MUST receive `data-state="open"` or `data-state="closed"` attribute reflecting the current menu state.
- FR-006: Click propagation MUST be stopped at the trigger element (`e.stopPropagation()`).
- FR-007: Menu items with `visible: { value: false }` MUST be removed from the DOM entirely, not hidden via CSS.
- FR-008: Divider items MUST render as `<li className="popupmenu-basic-divider">` with no caption, action, visibility, or style class.
- FR-009: The `popupmenu-menu` element MUST be a sibling of the trigger (not a descendant) to prevent z-index layering issues behind modal popups.
- FR-010: `FloatingFocusManager` MUST wrap the menu content to trap keyboard focus within the open menu.
- FR-011: The widget is NOT offline capable — Mendix server connectivity is required for item actions.
- FR-012: When `clickCloseOn === "onClickAnywhere"`, the menu MUST close synchronously before executing the item action.

## Props Reference

| Name | Type | Default | Caption | Description |
|------|------|---------|---------|-------------|
| `menuTrigger` | `ReactNode` | — | Trigger (required) | Widget slot: any Mendix widget placed here becomes the menu trigger. |
| `basicItems` | `BasicItemsType[]` | — | Basic items | Used when `advancedMode` is false. Each item: `itemType` (item/divider), `caption`, `visible`, `action`, `styleClass`. |
| `customItems` | `CustomItemsType[]` | — | Custom items | Used when `advancedMode` is true. Each item: `content` (widget slot), `visible`, `action`. |
| `advancedMode` | `boolean` | `false` | Enable advanced options | Switches between basic and custom item modes. |
| `trigger` | `"onclick" \| "onhover"` | `"onclick"` | Trigger | Interaction that opens the menu. |
| `clickCloseOn` | `"onClickAnywhere" \| "onClickOutside"` | `"onClickAnywhere"` | Close on (click) | When to close an onclick-triggered menu after an item is clicked. |
| `hoverCloseOn` | `"onClickOutside" \| "onHoverLeave"` | `"onHoverLeave"` | Close on (hover) | When to close an onhover-triggered menu. |
| `position` | `"left" \| "right" \| "top" \| "bottom"` | `"bottom"` | Position | Preferred side of the trigger to display the menu. May flip/shift if insufficient space. |
| `clippingStrategy` | `"absolute" \| "fixed"` | `"absolute"` | Clipping strategy | Controls floating-ui CSS positioning strategy. Use `"fixed"` when the menu must escape an `overflow: hidden` parent. |
| `menuToggle` | `boolean` | `false` | Open/close (Studio only) | Mendix boolean expression for programmatic control of menu visibility. Hidden in Studio Pro. |

### BasicItemsType

| Name | Type | Caption | Description |
|------|------|---------|-------------|
| `itemType` | `"item" \| "divider"` | Type | `"divider"` renders a separator with no other props active. |
| `caption` | `DynamicValue<string>?` | Caption | Display label for the item. Required for non-divider items (validated in Studio). |
| `visible` | `DynamicValue<boolean>?` | Visible | When `value: false`, the item is removed from the DOM. |
| `action` | `ActionValue?` | On click | Mendix action executed when the item is clicked. |
| `styleClass` | `StyleClassEnum` | Style | Visual style: defaultStyle, inverseStyle, primaryStyle, infoStyle, successStyle, warningStyle, dangerStyle. |

## Changelog

**v4.1.0 (2026-04-13):** Fixed advanced mode not showing custom widget content in Studio design mode.

**v4.0.0 (2024-11-13):** Moved styling to Atlas core (requires atlas-core 3.15.0+). Added `clippingStrategy` option. Fixed popup not overflowing parent elements.

**v3.6.0 (2024-08-20):** Added configurable "Close on" setting for hover trigger. Breaking: hover default close behavior changed to "hover leave" (previously "outside click").

## Open Questions
> Could not be determined from source code alone — requires human review
- [ ] Is there a maximum number of items (basic or custom) before performance or accessibility degrades?
- [ ] Does `safePolygon()` have any known limitations with non-rectangular trigger shapes?
- [ ] Is `menuToggle` planned for Studio Pro in a future release?
