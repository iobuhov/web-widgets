# Tooltip

## Purpose

The Tooltip widget displays contextual information in a floating overlay positioned relative to a trigger element. It supports text content (dynamic expression) and custom content (arbitrary Mendix widgets in a dropzone). Positioning is handled by `@floating-ui/react`, which provides automatic side-switching (`flip`), viewport sliding (`shift`), and safe hover polygon management (`safePolygon`). The widget supports three open modes — hover, click, and hoverFocus — enabling both mouse-driven and keyboard-accessible usage. It is categorized under "Display" and requires entity context.

## User Scenarios

### [P1] Hover tooltip with text
**Given** a Tooltip in "hover" mode with `renderMethod = "text"` and a `textMessage` configured  
**When** the user moves the mouse over the trigger element  
**Then** the tooltip appears after 25ms, positioned on the configured side with an optional directional arrow  
**When** the user moves the mouse away  
**Then** the tooltip closes immediately (0ms close delay); the `safePolygon` prevents accidental closure while the mouse travels across the gap between trigger and tooltip  

#### Edge Cases
- Tab focus alone does NOT open the tooltip in hover mode — `hoverFocus` mode is required for keyboard users.
- If both `textMessage` and `htmlMessage` are absent, the tooltip does not render even when the trigger is interacted with.
- When there is insufficient viewport space on the configured side, `flip()` middleware automatically moves the tooltip to the opposite side.

### [P2] Click tooltip with custom content
**Given** a Tooltip in "click" mode with `renderMethod = "custom"` and a `htmlMessage` dropzone containing widgets  
**When** the user clicks the trigger element  
**Then** the tooltip opens, rendering the configured widget content  
**When** the user clicks the trigger again  
**Then** the tooltip closes (toggle behavior)  
**When** the user clicks outside the tooltip  
**Then** the tooltip closes (via `useDismiss`)  

### [P3] Keyboard-accessible tooltip (hoverFocus mode)
**Given** a Tooltip in "hoverFocus" mode  
**When** the user navigates to the trigger via Tab key  
**Then** the tooltip opens (keyboard focus triggers it)  
**When** the user tabs away  
**Then** the tooltip closes on blur  

## Functional Requirements

- FR-001: System MUST support three open modes: `"hover"` (mouse hover only), `"click"` (click toggle), `"hoverFocus"` (hover + keyboard focus).
- FR-002: In hover mode, Tab focus alone MUST NOT open the tooltip — only mouse hover activates it. `hoverFocus` mode is required for keyboard accessibility.
- FR-003: Hover open delay MUST be 25ms; close delay MUST be 0ms (immediate close). `safePolygon()` MUST be applied to prevent accidental closure while the mouse moves across the gap.
- FR-004: Click mode MUST toggle the tooltip: first click opens, second click closes.
- FR-005: `useDismiss` MUST be active for all modes — clicking outside the tooltip closes it.
- FR-006: Positioning MUST use `@floating-ui/react` with `"fixed"` strategy and the middleware pipeline: `offset(8)` → `flip()` → `shift()` → `arrow()`.
- FR-007: The floating element MUST use `role="tooltip"` (applied via `useRole` hook).
- FR-008: System MUST support four tooltip positions: left, right, top (default), bottom; and three arrow alignments: start, end, none (centered, despite the enum key "none").
- FR-009: The arrow MUST be an 8×8px CSS pseudo-element (`::before`) positioned on the side facing the trigger; alignment along the side (start/end/center) is handled by floating-ui pixel positioning.
- FR-010: The tooltip MUST NOT render when both `textMessage` and `htmlMessage` are empty — prevents blank tooltip bubbles.
- FR-011: Studio Pro MUST hide `htmlMessage` when `renderMethod = "text"` and `textMessage` when `renderMethod = "custom"`.
- FR-012: Studio Pro MUST show a validation error if the active content property is empty (text is empty, or `htmlMessage.widgetCount === 0` for custom mode).
- FR-013: The trigger element MUST be wrapped in a `fit-content` width container to prevent layout stretching.
- FR-014: The widget MUST require entity context and support offline use.

## Props Reference

| Name | Type | Default | Caption | Description |
|------|------|---------|---------|-------------|
| `trigger` | `ReactNode` (widget slot) | — | Trigger | The element that activates the tooltip; any Mendix widget(s) can be placed here |
| `renderMethod` | `"text"` \| `"custom"` | — | Render method | `"text"` for a dynamic text expression; `"custom"` for a widget content dropzone |
| `textMessage` | `DynamicValue<string>` (optional) | — | Text | Tooltip text content; only shown in "text" render mode; may be undefined during data loading |
| `htmlMessage` | `ReactNode` (widget slot, optional) | — | Custom content | Widget content for the tooltip body; only shown in "custom" render mode |
| `tooltipPosition` | `"left"` \| `"right"` \| `"top"` \| `"bottom"` | `"top"` | Position | Which side of the trigger the tooltip appears on; auto-flipped when space is insufficient |
| `arrowPosition` | `"start"` \| `"none"` \| `"end"` | `"none"` | Arrow position | Arrow alignment along the tooltip edge: start, centered ("none"), or end |
| `openOn` | `"hover"` \| `"click"` \| `"hoverFocus"` | `"hover"` | Open on | Interaction mode that triggers the tooltip |

System properties supported: Name, CSS Class, Style, TabIndex.

## Visual Defaults

| Property | Value |
|----------|-------|
| Background | white |
| Text color | `#24276c` (dark blue) |
| Font | bold, 14px |
| Border-radius | 3px |
| Box-shadow | `0 0 5px rgba(0, 0, 0, 0.3)` |
| Z-index | 50 |
| Gap from trigger | 8px |

## Changelog

- **v1.5.1 (2026-02-10)**: License and open-source dependency README added.
- **v1.5.0**: Fixed scrollbar appearance issue.
- **v1.4.2**: Fixed hover flicker — added `safePolygon()` middleware.
- **v1.4.1**: Fixed layout container positioning; added `fit-content` width on trigger container.
- **v1.4.0**: Added `flip()` and `shift()` middleware for viewport overflow handling.
- **v1.3.2**: Added Escape key support for closing the tooltip.
- **v1.3.1**: Fixed positioning in Datagrid headers; fixed disabled input hover.
- **v1.0.0 (2021-12-10)**: Initial release.

## Open Questions

> Could not be determined from source code alone — requires human review
- [ ] Is the Escape key behavior (from v1.3.2) applied via `useDismiss` or a separate key handler? The changelog mentions it but the source code path for Escape handling was not confirmed.
- [ ] Can the tooltip z-index (50) conflict with Mendix platform overlays (modals, dropdowns)? A higher z-index might be needed in some layouts.
