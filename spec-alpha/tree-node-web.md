# TreeNode

## Purpose

The TreeNode widget renders a hierarchical, collapsible tree structure from a Mendix datasource. Each node has a header (text or custom widget content) and an optionally lazy-loaded body (widget slot) that appears when the node is expanded. The widget implements the full W3C ARIA tree pattern (`role="tree"`, `role="treeitem"`, `role="group"`) with keyboard navigation (arrow keys, Home/End, Enter/Space) validated to WCAG 2.1 AA compliance. Nesting is achieved by placing TreeNode widgets inside other TreeNode body slots. The widget requires entity context and supports offline use.

## User Scenarios

### [P1] Expand and collapse a node
**Given** a TreeNode configured with `headerType = "text"` and `startExpanded = false`  
**When** the user clicks the header  
**Then** the node body slides open (with animation if enabled) and the expand icon rotates  
**When** the user clicks the header again  
**Then** the node body slides closed  

#### Edge Cases
- When `openNodeOn = "iconClick"`, only clicking the expand/collapse icon triggers the state change — clicking other header content does not.
- When `openNodeOn = "headerClick"` (default for text headers), the entire header area is clickable.
- Leaf nodes (those with `hasChildren` expression evaluating to `false`, or nodes with no body content) render no expand icon and are not expandable.

### [P2] Keyboard navigation (ARIA tree pattern)
**Given** a tree with multiple nodes and levels  
**When** the user navigates with keyboard  
**Then** the following behaviors apply:

| Key | Behavior |
|-----|----------|
| ArrowRight | If collapsed: expand node. If expanded: move focus to first child. |
| ArrowLeft | If expanded: collapse node. If collapsed or leaf: move focus to parent. |
| ArrowDown | Move focus to next visible sibling or first child of the node below. |
| ArrowUp | Move focus to previous visible sibling or exit to parent. |
| Home | Move focus to first visible tree item (not circular). |
| End | Move focus to last visible tree item (not circular). |
| Enter / Space | Toggle expand/collapse. |

#### Edge Cases
- Collapsed sections (inside `aria-hidden="true"` bodies) are invisible to keyboard navigation.
- `preventDefault()` suppresses browser defaults (e.g., page scroll on arrow keys).

### [P3] Lazy-loaded children
**Given** a TreeNode body containing another TreeNode widget (nested)  
**When** the user expands the node for the first time  
**Then** the body enters a `LOADING` state (spinner shown) until the nested TreeNode mounts; subsequent collapses and re-expansions are instant (body DOM is kept alive after first load)  

## Functional Requirements

- FR-001: System MUST render `<ul role="tree">` at the root level and `<ul role="group">` at nested levels; each node `<li>` MUST have `role="treeitem"` and `tabIndex={0}`.
- FR-002: System MUST support four node states: `COLLAPSED_WITH_JS` (body DOM not rendered), `LOADING` (spinner), `EXPANDED` (body visible), `COLLAPSED_WITH_CSS` (body hidden in DOM for CSS animations).
- FR-003: On first expand, the node MUST enter `LOADING` state and transition to `EXPANDED` only after the nested TreeNode body mounts. On subsequent collapses, the body DOM MUST be kept alive (`COLLAPSED_WITH_CSS`) so re-expansion is instant.
- FR-004: A node MUST be considered a leaf when `hasChildren` expression explicitly returns `false` OR when the node has no configured body content. Only an explicit `false` suppresses the expand indicator — `undefined` is treated as "unknown, assume children exist."
- FR-005: `aria-hidden` MUST be applied to collapsed body divs to hide content from screen readers.
- FR-006: The widget MUST support ARIA keyboard navigation as specified in the User Scenarios (FR-001 to FR-006 key behaviors).
- FR-007: Height animation MUST use a JavaScript-driven CSS transition: capture old height via `getBoundingClientRect()`, set inline height to old value, then update to new value after a 1ms `setTimeout` to force a separate render frame, then clean up inline height on `transitionEnd`.
- FR-008: Icon animation and content animation MUST be independently controlled by `animateIcon` and `animate` props; both must be true for animated icon rotation during animated expand/collapse.
- FR-009: When `headerType = "text"`, the `openNodeOn` option MUST be hidden (text headers are always fully clickable).
- FR-010: When `datasource.status` is loading, the previous render MUST be preserved to prevent flicker.
- FR-011: An empty datasource MUST render an informational item "No data available" rather than an empty tree.
- FR-012: The widget MUST require entity context and support offline use.

## Props Reference

| Name | Type | Default | Caption | Description |
|------|------|---------|---------|-------------|
| `datasource` | `ListValue` | — | Data source | The Mendix entity datasource providing tree node items |
| `headerType` | `"text"` \| `"custom"` | — | Header type | `"text"` uses `headerCaption` expression; `"custom"` uses `headerContent` widget slot |
| `headerCaption` | `ListExpressionValue<string>` (optional) | — | Header caption | Per-item text expression for the node header (text mode only) |
| `headerContent` | `ListWidgetValue` (optional) | — | Header content | Per-item widget slot for the node header (custom mode only) |
| `openNodeOn` | `"headerClick"` \| `"iconClick"` | — | Open node on | Whether clicking the full header or only the icon toggles the node; only available in custom header mode |
| `hasChildren` | `ListExpressionValue<boolean>` | — | Has children | Per-item boolean expression; `false` = leaf node (no expand icon); `undefined` = assume children exist |
| `startExpanded` | boolean | `false` | Start expanded | Whether nodes are expanded on initial render |
| `children` | `ListWidgetValue` (optional) | — | Children | Per-item widget slot rendered as the node body when expanded |
| `animate` | boolean | — | Animate | Enable height animation for expand/collapse |
| `showIcon` | `"left"` \| `"right"` \| `"no"` | `"left"` | Show icon | Position of the expand/collapse icon, or no icon |
| `expandedIcon` | `DynamicValue<WebIcon>` (optional) | — | Expanded icon | Custom icon shown when node is expanded (defaults to chevron) |
| `collapsedIcon` | `DynamicValue<WebIcon>` (optional) | — | Collapsed icon | Custom icon shown when node is collapsed (defaults to chevron) |
| `animateIcon` | boolean | — | Animate icon | Enable icon rotation animation during expand/collapse |

System properties supported: Name, CSS Class, Style, TabIndex.

## Changelog

- **v3.8.0 (2026-01-16)**: Changed `hasChildren` from a static boolean to a per-item expression (`ListExpressionValue<boolean>`), enabling dynamic leaf detection per row; fixed lazy-loading spinner persisting indefinitely.
- **v1.2.1 (2024-11-13)**: Fixed collapse state reset during data reload (the `ValueStatus.Available` guard).
- **v1.2.0 (2024-05-15)**: Fixed nested empty TreeNode breaking parent behavior.
- **v1.1.3 (2023-08-22)**: Added `openNodeOn` property; changed HTML structure (breaking for CSS customizations).
- **v1.0.0 (2021-09-28)**: Initial release.

## Open Questions

> Could not be determined from source code alone — requires human review
- [ ] Is `v3.8.0`'s `hasChildren` change backwards-compatible? Existing configurations that used static `hasChildren = true` may need to migrate to an expression.
- [ ] What is the default `animate` value — is it on or off by default? The XML declares it as a boolean but the default value is not explicit in the reviewed source.
