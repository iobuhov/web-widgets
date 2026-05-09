# tree-node-web — Draft Spec

Widget: `tree-node-web`
Package: `packages/pluggableWidgets/tree-node-web/`
Agent: worker
Date: 2026-05-09

---

## src/TreeNode.tsx

**1. What is the purpose of this file?**
The root Mendix pluggable widget entry component. Transforms datasource items to `TreeNodeItem` objects, guards against loading-state resets, and passes data to the `TreeNode` presentation component.

**2. What kind of logic is described in this file?**
- `useEffect` only runs when `datasource.status === ValueStatus.Available` — prevents tree collapse/reset while data is reloading.
- When `datasource.items` is non-empty: maps items via `mapDataSourceItemToTreeNodeItem`.
- When `datasource.items` is empty/undefined: sets `InfoTreeNodeItem { Message: "No data available" }`.
- `mapDataSourceItemToTreeNodeItem`:
  - `headerContent`: `headerType === "text"` → `headerCaption?.get(item).value` (string); else → `headerContent?.get(item)` (ReactNode widget slot).
  - `bodyContent`: `children?.get(item)` — per-item widget slot for nested tree nodes.
  - `isUserDefinedLeafNode`: `hasChildren?.get(item).value === false` — if expression returns false, the node is treated as a leaf.
- `showCustomIcon = Boolean(expandedIcon) || Boolean(collapsedIcon)` — uses custom icons only if either is configured.
- `animateIcon = props.animate && props.animateIcon` — both global and per-icon animate flags must be true.
- Icons are only used when `ValueStatus.Available` — undefined otherwise (loading state for icon values).

**3. What part of behavior can be documented from this file?**
- Datasource state is monitored with `useEffect` on `[datasource.status, datasource.items]` — the tree state is NOT reset when the datasource is still loading (`status !== Available`). This preserves expand/collapse state across data refreshes.
- `hasChildren?.get(item).value === false` (explicit false) → leaf node. `undefined` or `true` → has children. This is intentional: if the expression is null/unavailable, the node assumes it has children.
- `children` is optional — if undefined, body content is null.

**4. Is it user-facing?**
No — internal Mendix adapter.

**5. What new did you learn from this file?**
The datasource state guard in `useEffect` is critical: it prevents the tree from flashing "No data available" or resetting expand/collapse state during Mendix data refresh cycles. The items state (`useState([])`) is initialized with `[]` and only updated when the datasource is fully available — the tree renders empty during loading but retains its previous state if data reloads.

---

## src/TreeNode.xml

**1. What is the purpose of this file?**
Mendix widget descriptor. Defines a Data Containers category widget for displaying hierarchical tree structures, with configurable data source, header, icons, and animation.

**2. What kind of logic is described in this file?**
General group:
- `advancedMode`: boolean (default: false) — gates visualization properties in Studio (not Studio Pro).
- `datasource`: list data source (required).
- `headerType`: text | custom (default: text) — header render method.
- `openNodeOn`: headerClick (default) | iconClick — which element triggers expand/collapse.
- `headerContent`: widget slot per datasource item (for custom header).
- `headerCaption`: text template per datasource item (for text header).
- `hasChildren`: Boolean **expression** per datasource item — whether node has children.
- `startExpanded`: boolean (default: false) — initial state.
- `children`: widget slot per datasource item — for nested tree node content.
- `animate`: boolean (default: true) — enables all animations.

Visualization group (Icon sub-group):
- `showIcon`: left (default) | right | no — icon position.
- `expandedIcon`: optional Mendix icon — custom icon when expanded.
- `collapsedIcon`: optional Mendix icon — custom icon when collapsed.
- `animateIcon`: boolean (default: true) — animates the chevron/icon on expand/collapse.

No system properties explicitly listed (framework provides name, class, style, tabIndex).

**3. What part of behavior can be documented from this file?**
- `needsEntityContext="true"`, `offlineCapable="true"`.
- Category: Data containers.
- `hasChildren` is an **expression** (not a boolean attribute) — allows conditional logic like `$currentObject/ChildCount > 0`.
- `children` and `headerContent` are `dataSource="datasource"` — per-item widget slots.
- `expandedIcon` and `collapsedIcon` are top-level (not per-item) — same custom icons for all nodes.
- `openNodeOn` is hidden when `headerType === "text"` (only relevant for custom headers where header content may be clickable itself).

**4. Is it user-facing?**
No — Studio Pro configuration descriptor.

**5. What new did you learn from this file?**
`hasChildren` was changed from a boolean attribute to an expression in v3.8.0. This allows dynamic calculation (e.g., `$currentObject/ChildCount > 0`) rather than just binding to a stored Boolean attribute — enabling more flexible tree configurations without denormalized data.

---

## typings/TreeNodeProps.d.ts

**1. What is the purpose of this file?**
Auto-generated TypeScript types from the XML descriptor.

**2. What kind of logic is described in this file?**
- `HeaderTypeEnum = "text" | "custom"`.
- `OpenNodeOnEnum = "headerClick" | "iconClick"`.
- `ShowIconEnum = "left" | "right" | "no"`.
- `TreeNodeContainerProps`:
  - `hasChildren: ListExpressionValue<boolean>` — per-item expression (not attribute).
  - `headerContent?: ListWidgetValue` — optional per-item widget slot.
  - `children?: ListWidgetValue` — optional per-item widget slot.
  - `expandedIcon?: DynamicValue<WebIcon>` — widget-level (not per-item) icon.
  - `collapsedIcon?: DynamicValue<WebIcon>` — widget-level icon.
  - `animateIcon: boolean` — non-optional boolean.
- Container props: `name`, `class`, `style`, `tabIndex` — CSS customization supported.

**3. What part of behavior can be documented from this file?**
- `expandedIcon` and `collapsedIcon` are `DynamicValue<WebIcon>` — single icon for all nodes (not per-item).
- `hasChildren` is `ListExpressionValue<boolean>` — evaluates per datasource item.
- `headerContent` and `children` are `ListWidgetValue` (optional) — per-item widget slots.

**4. Is it user-facing?**
No — TypeScript types only.

**5. What new did you learn from this file?**
`DynamicValue<WebIcon>` for icons (not `ListExpressionValue`) means the same expanded/collapsed icon applies to every tree node in the widget. There's no per-node custom icon — it's a widget-level configuration. This is consistent with the XML which has `expandedIcon` at the widget level, not inside the per-item properties.

---

## src/components/TreeNode.tsx

**1. What is the purpose of this file?**
The presentation component for the tree container. Renders the `<ul>` tree root, maps items to `<TreeNodeBranch>` components, tracks nesting level via context, and signals its parent when it's a nested tree.

**2. What kind of logic is described in this file?**
- `TreeNodeState` const enum: `COLLAPSED_WITH_JS | COLLAPSED_WITH_CSS | EXPANDED | LOADING`.
- Root renders `<ul class="widget-tree-node" role={level === 0 ? "tree" : "group"} data-focusindex={tabIndex || 0}>`.
- Null return: when `items === null` or empty array.
- `InfoTreeNodeItem` (no data): renders a "No data available" message (handled in the array check).
- `useContext(TreeNodeBranchContext)` — gets current nesting level (`level`).
- `useTreeNodeRef`: returns `[treeNodeElement, updateTreeNodeElement]` — `useState+callback` ref that triggers re-renders when DOM is ready.
- `useInformParentContextOfChildNodes(items.length, isInsideAnotherTreeNode)` — notifies parent branch about how many child nodes this nested tree has.
- `isInsideAnotherTreeNode()`: checks if `treeNodeElement.parentElement.className.includes("widget-tree-node-body")` — detects if this tree node is a child of another tree node's body.
- Per item: renders `<TreeNodeBranch key={id} ...>` with body content as children.

**3. What part of behavior can be documented from this file?**
- Root level (`level === 0`): `role="tree"` — ARIA tree widget root.
- Nested level (`level > 0`): `role="group"` — ARIA tree group.
- `data-focusindex` on `<ul>`: used by focus management (not standard ARIA, custom attribute for focus tracking).
- Empty array or null: renders nothing — the tree disappears when no data is available.
- `useTreeNodeRef` uses `useState` (not `useRef`) to force a re-render when the DOM element is attached — needed so `useInformParentContextOfChildNodes` can check DOM structure.

**4. Is it user-facing?**
Yes — renders the tree container.

**5. What new did you learn from this file?**
The `TreeNodeState` enum is `const enum` — TypeScript inlines the values at compile time (no runtime object). The four states represent a careful state machine: two collapsed states distinguish between "never opened" (JS-collapsed, body not in DOM) and "previously opened" (CSS-collapsed, body in DOM but hidden). This distinction enables lazy loading on first open while supporting animation on subsequent opens.

---

## src/components/TreeNodeBranch.tsx

**1. What is the purpose of this file?**
Individual tree node item — renders the header, manages expand/collapse state machine, handles click and keyboard events, animates height transitions, and renders body content with lazy loading detection.

**2. What kind of logic is described in this file?**
State machine (`TreeNodeState`):
- Initial state: `startExpanded ? EXPANDED : COLLAPSED_WITH_JS`.
- Toggle transitions:
  - `COLLAPSED_WITH_JS` → `LOADING` (first open; lazy-loads children into DOM).
  - `LOADING` → stays LOADING (no toggle while loading).
  - `COLLAPSED_WITH_CSS` → `EXPANDED`.
  - `EXPANDED` → `COLLAPSED_WITH_CSS`.
- `LOADING` → `EXPANDED` automatically when `informParentOfChildNodes` fires (children loaded).
- `LOADING` → `EXPANDED` also when no nested tree node is found after render.

`isActualLeafNode` state:
- Initial: `isUserDefinedLeafNode || !children`.
- Updated by `informParentOfChildNodes` callback from nested tree: if nested tree has 0 items → `isActualLeafNode = true`; if > 0 → `false`.
- Updated by `useEffect` on `[children, isUserDefinedLeafNode]` changes.

DOM structure:
- `<li class="widget-tree-node-branch" role="treeitem" aria-expanded={isExpanded} tabIndex={0}>`:
  - `<span class="widget-tree-node-branch-header [clickable] [reversed]" id="{id}TreeNodeBranchHeader">`:
    - `<span class="widget-tree-node-branch-header-value">{headerContent}</span>`.
    - (if non-leaf and icon visible) `<span class="widget-tree-node-branch-header-icon-container [clickable]">{icon}</span>`.
  - (conditional) Context.Provider for child level+1:
    - `<div class="widget-tree-node-body [hidden] [loading]" aria-hidden={!expanded} id="{id}TreeNodeBranchBody">`:
      - `{children}` (nested tree nodes).

Body rendering condition: `(!isActualLeafNode && state !== COLLAPSED_WITH_JS) || isAnimating`.
- Body not rendered until first expand (COLLAPSED_WITH_JS prevents DOM insertion).
- Body stays in DOM once rendered (supports CSS animation and preserves nested state).
- `isAnimating` keeps body in DOM during collapse animation.

`eventTargetIsNotCurrentBranch`: event guard that returns `true` if the click/keydown came from a nested branch (not this branch). Checks: target is not the `<li>`, not within the first child (header span), not the last child (body div). Prevents nested tree nodes from triggering parent toggle.

**3. What part of behavior can be documented from this file?**
- First open triggers `LOADING` state — a spinner is shown in the icon while Mendix renders child data.
- After first open, node stays in DOM (CSS-collapsed) — preserves nested expand state, no re-mounting.
- Icon click vs header click is controlled by `openNodeOn`: appropriate `onClick` assigned to either the header span or the icon span.
- `aria-expanded` on `<li>` reflects only `EXPANDED` state (LOADING is not expanded).
- `aria-hidden` on body: `true` for all non-EXPANDED states — screen readers don't see collapsed content.
- `header-reversed` class when `iconPlacement === "left"` — visual reversal of icon/text order.
- `header-clickable` class added only for non-leaf nodes.

**4. Is it user-facing?**
Yes — renders each individual tree node item.

**5. What new did you learn from this file?**
The two collapsed states (`COLLAPSED_WITH_JS` vs `COLLAPSED_WITH_CSS`) are a deliberate lazy-loading pattern: the body div doesn't exist in the DOM until the first expand, which prevents Mendix from pre-loading child data for all nodes on mount. Only when the user expands a node does the `children` widget slot render — triggering the datasource load for that subtree. After first load, the body stays in DOM to avoid re-fetching.

---

## src/components/hooks/lazyLoading.ts

**1. What is the purpose of this file?**
Detects whether a nested tree node has been rendered inside a branch's body div — used to transition from LOADING to EXPANDED state.

**2. What kind of logic is described in this file?**
- `elementHasNestedTreeNode(element)`: `element?.lastElementChild?.className.includes("widget-tree-node") ?? true`.
- `useTreeNodeLazyLoading(ref)`: wraps in a stable `useCallback` returning `hasNestedTreeNode()`.
- Default return `true` when `element` is null — assumes children exist while DOM is initializing.

**3. What part of behavior can be documented from this file?**
- Lazy loading detection: checks if the last child of the body div has class `"widget-tree-node"` — this is what Mendix renders when `children` widget slot contains another Tree Node widget.
- Returns `true` by default (no element yet) — safe assumption during initial render.

**4. Is it user-facing?**
No — internal hook.

**5. What new did you learn from this file?**
The lazy loading detection works by checking if the **last** child element of the body div has the tree node CSS class. The assumption is that a nested `<TreeNode>` widget renders a `<ul class="widget-tree-node">` as the final element in the body. If the body doesn't have this class (no nested tree node), the `LOADING` state transitions to `EXPANDED` directly — meaning the `hasChildren` expression returned `true` but no tree node was placed in the `children` slot.

---

## src/components/hooks/useAnimatedHeight.tsx

**1. What is the purpose of this file?**
Provides height animation for tree node body expand/collapse transitions.

**2. What kind of logic is described in this file?**
- `captureElementHeight()`: records current body height in a `useRef` before state change.
- `animateTreeNodeContent()`:
  1. Reads new height after state change.
  2. If heights differ: sets `isAnimating = true`, sets body `style.height` to old height.
  3. After 1ms `setTimeout`: sets `style.height` to new height — CSS transition animates the change.
  4. Returns cleanup function (clears timeout).
- `cleanupAnimation()`: called on `transitionend` event — removes `height` style, sets `isAnimating = false`.
- `isAnimating`: true while height is transitioning — keeps body in DOM during collapse animation.

**3. What part of behavior can be documented from this file?**
- Animation works via JavaScript-driven CSS transition: explicit `height` pixels set, then changed 1ms later to trigger the CSS `transition: height`.
- If heights are the same (no change), animation is skipped.
- `isAnimating` prevents body from being removed from DOM during the collapse animation (since body render condition checks `|| isAnimating`).
- `animateTreeNodeContent` is called in `useLayoutEffect` (synchronous after DOM paint) — reads new height immediately after render.

**4. Is it user-facing?**
No — animation hook.

**5. What new did you learn from this file?**
The 1ms `setTimeout` trick is a classic CSS animation technique: setting an element's height in the same frame as its DOM attachment doesn't trigger a transition (no "from" state). The 1ms delay ensures the browser paints the initial height first, then the transition animates to the final height. This is needed because body elements go from non-existent (COLLAPSED_WITH_JS) to visible — they have no prior height to transition from.

---

## src/components/hooks/TreeNodeAccessibility.tsx

**1. What is the purpose of this file?**
Implements WAI-ARIA tree keyboard navigation and focus management.

**2. What kind of logic is described in this file?**
`useTreeNodeFocusChangeHandler()` — returns a `TreeNodeFocusChangeHandler` callback:
- Gets all visible `<li.widget-tree-node-branch>` elements within the current tree scope (`.widget-tree-node[role=tree]` ancestor).
- "Visible" = not inside any `[aria-hidden=true]` body — excludes collapsed subtrees.
- `FocusTargetChange.FIRST/LAST`: focuses first/last visible item.
- `FocusTargetChange.PREVIOUS/NEXT` (HORIZONTAL): focuses adjacent sibling in flat visible list.
- `FocusTargetChange.PREVIOUS` (VERTICAL): finds parent branch (li whose last child contains target) and focuses it.
- `FocusTargetChange.NEXT` (VERTICAL): finds first visible child item inside target's last element child.

`useTreeNodeBranchKeyboardHandler()` — maps keys to actions:
- `Enter`, `Space`: toggle expand/collapse.
- `Home`: focus first item.
- `End`: focus last item.
- `ArrowUp`: previous sibling (HORIZONTAL).
- `ArrowDown`: next sibling (HORIZONTAL).
- `ArrowRight`: if collapsed → expand; if expanded or leaf → focus first child (VERTICAL).
- `ArrowLeft`: if collapsed or leaf → focus parent (VERTICAL); if expanded → collapse.

Guard: `eventTargetIsNotCurrentBranch` — only processes keyboard events originating from this branch's `<li>`.

**3. What part of behavior can be documented from this file?**
- Full WAI-ARIA tree keyboard pattern implemented.
- ArrowRight on a leaf node: moves to first child (same as expanded) — consistent behavior since a leaf can't be expanded.
- ArrowLeft on a collapsed node: moves to parent (skips expand/collapse — nothing to collapse).
- Visible branches exclude items within `aria-hidden=true` bodies — collapsed subtrees are skipped in navigation.
- Focus scope is limited to the root `<ul role="tree">` ancestor — multiple tree widgets on page are independent.

**4. Is it user-facing?**
No — keyboard navigation hook.

**5. What new did you learn from this file?**
The tree focus navigation uses DOM queries at focus-change time (not cached at render time). This means the visible branch list is always fresh — expanding a node immediately makes its children available for navigation without any state sync. The `aria-hidden=true` filter ensures collapsed subtrees are invisible to keyboard navigation, matching the ARIA spec for tree widgets.

---

## src/components/hooks/useKeyboardHandler.tsx

**1. What is the purpose of this file?**
Generic keyboard event handler factory — maps W3C key attribute values to named handler functions.

**2. What kind of logic is described in this file?**
- `keyValueToHandlerNameMap`: maps `" "` (space) → `"Space"`, others map to their own name.
- `itCameFromCurrentTarget(event)`: `event.currentTarget === event.target` — only handles events directly on the registered element (not bubbled from children).
- `useKeyboardHandler(keyHandlers)`: returns a `KeyboardEventHandler` that: checks if it came from current target, maps key to handler name, calls `stopPropagation()` + `preventDefault()`, then calls the handler.

**3. What part of behavior can be documented from this file?**
- Space bar maps to `"Space"` (not `" "`) for readability in handler definitions.
- `stopPropagation()` on handled keys: prevents keyboard events from bubbling to parent tree items.
- `preventDefault()` on handled keys: prevents browser default (e.g., Space scrolling page, arrow keys scrolling).
- Only handles events from the exact current target — child form widgets (text inputs, etc.) can use the same keys without triggering tree navigation.

**4. Is it user-facing?**
No — utility hook.

**5. What new did you learn from this file?**
The `itCameFromCurrentTarget` guard is what fixed the v1.1.3 issue (keyboard input in child form widgets was broken). Without this check, typing Space in a `<TextBox>` child would trigger the tree node toggle. With the check, only keyboard events directly on the `<li>` element are handled by the tree — child widget keyboard events bubble up but are ignored.

---

## src/components/TreeNodeBranchContext.ts

**1. What is the purpose of this file?**
React context for tracking tree nesting level and propagating child node count from nested tree components to parent branch components.

**2. What kind of logic is described in this file?**
- `TreeNodeBranchContext`: default `{ level: 0, informParentOfChildNodes: () => null }`.
- `useInformParentContextOfChildNodes(numberOfNodes, identifyParentIsTreeNode)`:
  - `useEffect` fires when `numberOfNodes` or `identifyParentIsTreeNode` changes.
  - Only fires when `level > 0` AND `identifyParentIsTreeNode()` returns true.
  - Calls `informParentOfChildNodes(numberOfNodes)` — tells the parent `TreeNodeBranch` how many items this nested tree has.

**3. What part of behavior can be documented from this file?**
- `informParentOfChildNodes` is the callback used by `TreeNodeBranch` to receive child count updates from nested tree widgets.
- Triggered whenever `numberOfNodes` changes — handles empty tree case (0 items → parent branch becomes leaf node).
- `level > 0` guard: root-level tree nodes don't report up (no parent branch).
- `identifyParentIsTreeNode()` guard: prevents false reports from unrelated context inheritance.

**4. Is it user-facing?**
No — context/communication mechanism.

**5. What new did you learn from this file?**
`informParentOfChildNodes` is how nested tree nodes "auto-discover" whether they're leaf nodes: when a child `TreeNode` widget renders with 0 items, it reports 0 upward → the parent `TreeNodeBranch` sets `isActualLeafNode = true` → icon disappears, node becomes non-expandable. This is fully automatic without any `hasChildren` configuration needed for the empty case.

---

## src/components/HeaderIcon.tsx

**1. What is the purpose of this file?**
Renders the expand/collapse icon for each tree node header based on state and configuration.

**2. What kind of logic is described in this file?**
- `LOADING` state → spinning loading circle SVG (`aria-hidden`).
- `showCustomIcon` is true → `<CustomHeaderIcon icon={isExpanded ? expandedIcon : collapsedIcon} />`.
- Default → `<ChevronIcon className={...}>` with conditional CSS classes:
  - `"widget-tree-node-branch-header-icon-animated"`: when `animateIcon` is true.
  - `"widget-tree-node-branch-header-icon-collapsed-left"`: collapsed + `iconPlacement === "left"`.
  - `"widget-tree-node-branch-header-icon-collapsed-right"`: collapsed + `iconPlacement === "right"`.

**3. What part of behavior can be documented from this file?**
- Loading spinner is `aria-hidden` — not announced to screen readers.
- Custom icon switches between `expandedIcon` and `collapsedIcon` based on state.
- Chevron animation: CSS transforms rotate the chevron via the `animated` class.
- Separate collapsed-left and collapsed-right classes — allows CSS to rotate differently based on icon position.

**4. Is it user-facing?**
Yes — the expand/collapse icon is visible to users.

**5. What new did you learn from this file?**
The LOADING state shows a spinner icon instead of the expand/collapse icon — this is user-visible feedback that child data is being loaded. The loading circle is an SVG (not a Mendix icon), always displayed during the `LOADING` state regardless of `showCustomIcon` or `animateIcon` settings.

---

## src/TreeNode.editorConfig.ts

**1. What is the purpose of this file?**
Studio Pro property visibility, validation, structure preview, and custom caption for the Tree Node widget.

**2. What kind of logic is described in this file?**
`getProperties`:
- `showIcon === "no"` → hides `expandedIcon`, `collapsedIcon`.
- `headerType === "text"` → hides `headerContent`, `openNodeOn`.
- `headerType === "custom"` → hides `headerCaption`.
- `!hasChildren` → hides `startExpanded`, `children`.
- `platform === "web"` → `transformGroupsIntoTabs`; if `!advancedMode` hides `showIcon`, `expandedIcon`, `collapsedIcon`, `animate`, `animateIcon`.
- `platform !== "web"` → hides `advancedMode` (all visualization options visible in Studio Pro).

`getPreview`: 3-part structure:
1. Header bar: "Tree node" label with `topbarData` background.
2. Node header row: optional chevron + text/drop zone for header; chevron shown only when `showIcon !== "no"` AND `hasChildren`.
3. Body row: drop zone for `children` (only when `hasChildren` is set).

`check` validation: error if `openNodeOn === "iconClick"` AND `showIcon === "no"` — can't open by clicking icon if icon is hidden.
`getCustomCaption`: datasource caption or `"Tree node"`.

**3. What part of behavior can be documented from this file?**
- `openNodeOn` is hidden when `headerType === "text"` — text headers are always fully clickable; icon-only click only matters for custom headers where the header content itself may be interactive.
- `startExpanded` and `children` hidden when `hasChildren` is not configured — prevents orphaned properties.
- Studio Pro shows validation error when icon click is set but icon is hidden — prevents unusable configuration.
- Visualization options (icons, animation) hidden from basic users in Studio app.

**4. Is it user-facing?**
No — Studio Pro only.

**5. What new did you learn from this file?**
The `openNodeOn` property is hidden for text headers but shown for custom headers. This makes sense: with a text header, clicking anywhere in the header row is unambiguously a toggle. With a custom header containing interactive widgets (buttons, links), users may want to restrict toggling to the icon only — avoiding conflicts with the header content's own click handlers.

---

## src/TreeNode.editorPreview.tsx

**1. What is the purpose of this file?**
Live React preview in Studio Pro design mode — renders one expanded tree node using the actual `TreeNode` component.

**2. What kind of logic is described in this file?**
- Renders `<TreeNode>` with a single item (id="1"):
  - `headerContent`: text via `renderTextTemplateWithFallback` or custom via renderer.
  - `bodyContent`: always rendered via `children.renderer`.
  - `isUserDefinedLeafNode = !props.hasChildren` — if no `hasChildren` expression, treats as leaf.
- `startExpanded={true}` — always expanded in design mode.
- `animateIcon={false}`, `animateTreeNodeContent={false}` — no animations in preview.
- `openNodeOn="headerClick"` always — icon-only click not used in preview.
- `mapPreviewIconToWebIcon(props.expandedIcon/collapsedIcon)` — converts preview icon format to runtime `WebIcon`.

**3. What part of behavior can be documented from this file?**
- Preview always shows one node expanded — design mode shows the header + body layout.
- `renderTextTemplateWithFallback`: shows `"[No header caption configured]"` placeholder if template is empty/whitespace.
- Both header and body drop zones are always rendered — Studio Pro users can drag widgets into either slot.

**4. Is it user-facing?**
No — Studio Pro design mode preview only.

**5. What new did you learn from this file?**
`renderTextTemplateWithFallback` trims the text before checking emptiness — a template with only whitespace (e.g., just spaces) is treated as "not configured" and shows the placeholder. This prevents invisible/empty headers in design mode when users haven't typed a caption yet.

---

## e2e/TreeNode.spec.js

**1. What is the purpose of this file?**
Playwright E2E tests for expand/collapse behavior and WCAG 2.1 AA accessibility compliance.

**2. What kind of logic is described in this file?**
Expand tests:
- Single node expand: click first header → screenshot comparison.
- Multiple nodes expand: click 2nd, then 1st → screenshot.

Collapse tests:
- Single collapse: click to expand, click again to collapse.
- Multiple collapse: complex sequence with index-shifting (opening a node shifts subsequent header indices).

Accessibility audit:
- `AxeBuilder` with `withTags(["wcag21aa"])` on `.mx-name-treeNode1` — no violations expected.
- Excludes navigation tree (`.mx-name-navigationTree3`).

**3. What part of behavior can be documented from this file?**
- WCAG 2.1 AA compliance tested automatically — confirms ARIA roles (tree, treeitem, group) are correctly implemented.
- Header click toggles expand/collapse (headerClick mode tested here).
- Node index shifting after expand: opening a node with N children shifts all subsequent headers by N positions.

**4. Is it user-facing?**
No — test file.

**5. What new did you learn from this file?**
The E2E test is the only place accessibility compliance is formally verified — using `@axe-core/playwright`. The tree widget claims full WCAG 2.1 AA compliance (`wcag21aa` tag), validated against a real Mendix app. This level of accessibility testing (not just screenshot comparison) is more thorough than other widgets covered so far.

---

## CHANGELOG.md

**1. What is the purpose of this file?**
Version history from v1.0.0 (initial release) to v3.8.0.

**2. What kind of logic is described in this file?**
- **v3.8.0 (2026-01-16)**: `hasChildren` changed from boolean attribute to expression; fixed loading spinner stuck when no children found.
- **v1.2.1 (2024-11-13)**: Fixed collapse state reset on datasource reload (→ the `datasource.status` guard in `useEffect`).
- **v1.2.0 (2024-05-15)**: Fixed nested empty tree node breaking parent behavior (→ `informParentOfChildNodes` with 0 → `isActualLeafNode = true`).
- **v1.1.3 (2023-08-22)**: Fixed keyboard input in child form widgets (`itCameFromCurrentTarget` guard); added `openNodeOn` property; breaking HTML change.
- **v1.1.2 (2023-08-10)**: Fixed Atlas icon display; fixed child not refreshing.
- **v1.0.1 (2021-10-13)**: `advancedMode` available only in Studio (not Studio Pro).
- **v1.0.0 (2021-09-28)**: Initial release.

**3. What part of behavior can be documented from this file?**
- v3.8.0 spinner fix: the loading spinner was getting stuck when `hasChildren = true` but no tree node widget was placed in the body slot — now detects this case and transitions to EXPANDED.
- v1.2.1: the `datasource.status` guard was added after a real regression where datasource reload caused tree collapse.
- v1.2.0: empty nested tree was causing its parent branch to break — fixed by auto-detecting 0-item trees and marking parent as leaf.
- v1.1.3: `openNodeOn` property added as a result of users needing clickable widgets inside the tree header.

**4. Is it user-facing?**
No — developer changelog.

**5. What new did you learn from this file?**
Almost every behavioral mechanism in this widget was added in response to a real user-reported bug. The `datasource.status` guard (v1.2.1), the `informParentOfChildNodes` auto-leaf detection (v1.2.0), the `itCameFromCurrentTarget` keyboard guard (v1.1.3), and the `hasChildren` expression upgrade (v3.8.0) all trace directly to specific changelogs. The code's complexity is justified by the accumulated edge cases of real-world tree widget usage.

---

## Summary of Key Findings

- **Purpose**: Hierarchical tree view widget backed by a Mendix datasource. Each node is a datasource item with a configurable header and optional children widget slot for nested tree nodes.
- **State machine**: Four states per node: `COLLAPSED_WITH_JS` (never opened, body not in DOM), `LOADING` (first open, waiting for child data), `EXPANDED`, `COLLAPSED_WITH_CSS` (previously opened, body hidden via CSS).
- **Lazy loading**: Body `<div>` not rendered until first expand (COLLAPSED_WITH_JS) — prevents pre-loading all subtrees on mount. Detected via DOM class check (`"widget-tree-node"` on last child of body).
- **Parent-child communication**: `TreeNodeBranchContext` carries nesting `level` and `informParentOfChildNodes` callback. Nested empty tree nodes auto-mark their parent as a leaf node.
- **Datasource guard**: `useEffect` only updates items when `datasource.status === Available` — preserves expand/collapse state during data refreshes.
- **`hasChildren` expression**: Per-item Boolean expression (as of v3.8.0). Explicit `false` → leaf node; `undefined`/`true` → expandable.
- **Keyboard navigation**: Full WAI-ARIA tree pattern — Enter/Space toggle, Home/End, ArrowUp/Down (siblings), ArrowLeft/Right (parent/child traversal). Only events from `event.currentTarget === event.target` are handled — child form widget keyboard input is passed through.
- **ARIA**: `role="tree"` on root `<ul>`, `role="group"` on nested `<ul>`, `role="treeitem"` on `<li>`, `aria-expanded`, `aria-hidden` on body.
- **Icon**: Chevron (default), custom Mendix icon (expandedIcon/collapsedIcon), or loading spinner (LOADING state). Icons animate via CSS classes.
- **Animation**: Height animation using capture/set/setTimeout/transitionEnd pattern. `isAnimating` keeps body in DOM during collapse animation.
- **`openNodeOn`**: headerClick (whole header is toggle area) | iconClick (only icon triggers toggle). Only relevant for custom headers with interactive content.
- **CSS customization**: Supports `class` and `style` props.
- **offlineCapable**: `true`.
- **Accessibility**: WCAG 2.1 AA tested with `@axe-core/playwright` in E2E — the only widget in this set with automated accessibility auditing.
