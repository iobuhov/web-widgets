# Draft: tree-node-web

Widget package: `packages/pluggableWidgets/tree-node-web`

---

## src/TreeNode.xml

**1. What is the purpose of this file?**
Widget descriptor XML defining the widget's identity, all configurable props, and Studio Pro categorization. Generates `TreeNodeProps.d.ts`.

**2. What kind of logic is described in this file?**
Declares a "General" property group with: `datasource` (list, entity context required); `headerType` ("text"|"custom"); `openNodeOn` ("headerClick"|"iconClick"); `headerContent` (widget slot for custom header); `headerCaption` (expression string for text header); `hasChildren` (boolean expression, per-item); `startExpanded` (boolean, default false); `children` (widget slot for body content); `animate` (boolean). A "Visualization" group with an "Icon" subgroup: `showIcon` ("left"|"right"|"no", default "left"); `expandedIcon`/`collapsedIcon` (dynamic WebIcon); `animateIcon` (boolean). Widget is `needsEntityContext="true"`, `offlineCapable="true"`, categorized as "List view" or "Tree" type.

**3. What part of behavior can be documented from this file?**
- `hasChildren` is a per-item boolean expression (not a static property) — each node can dynamically declare whether it has children.
- Default state is collapsed (`startExpanded=false`).
- `openNodeOn` distinguishes between clicking the entire header vs clicking only the icon.
- Both `headerContent` and `children` are widget slots (dropzones), not data attributes.
- Widget is offline capable despite lazy-loading children support.

**4. Is it user-facing?**
Defines the developer-facing configuration interface in Studio/Studio Pro.

**5. What new did you learn from this file?**
`hasChildren` changed from a static boolean to a per-item expression in v3.8.0. This enables scenarios where some nodes in the same datasource have children and others don't, with the determination made dynamically per row. The `"none"` value for `arrowPosition` and `"no"` for `showIcon` follow the same naming pattern — enum keys represent absence, not a literal "none" value shown to users.

---

## src/TreeNode.tsx

**1. What is the purpose of this file?**
HOC container that bridges Mendix datasource binding to the presentational `TreeNode` component. Maps `ObjectItem` datasource items to `TreeNodeItem` objects.

**2. What kind of logic is described in this file?**
Waits for `datasource.status === ValueStatus.Available` before rendering (prevents state resets during loading). Maps items via `mapDataSourceItemToTreeNodeItem()`: extracts `id`, `headerContent` (text or custom widget), `bodyContent` (children widget), `isUserDefinedLeafNode` (from `hasChildren?.get(item).value === false`). Resolves `expandedIcon` and `collapsedIcon` from `DynamicValue<WebIcon>` (checks `.status` before extraction). Combines animation flags: `animateIcon && animateTreeNodeContent`. Returns `InfoTreeNodeItem { Message: "No data available" }` when datasource is empty.

**3. What part of behavior can be documented from this file?**
- `isUserDefinedLeafNode` is `true` when `hasChildren.get(item).value === false` — only an explicit `false` marks a leaf; `undefined`/`true` do not.
- When datasource is loading, the previous render is preserved — no flicker on data reload.
- Icon animations require both `animateIcon` AND `animateTreeNodeContent` to be true.
- An empty datasource renders an informational item ("No data available"), not an empty list.

**4. Is it user-facing?**
No — internal Mendix-to-component adapter.

**5. What new did you learn from this file?**
The leaf node determination uses a strict `=== false` check on the `hasChildren` expression, not just a falsy check. This means `undefined` (expression not loaded) and `null` are treated as "unknown — assume children exist" — the widget errs on the side of showing the expand affordance rather than hiding it. Only an explicit `false` from the expression suppresses the expand indicator.

---

## src/components/TreeNode.tsx

**1. What is the purpose of this file?**
Root presentational component that renders the full tree structure as a `<ul>` and maps items to `TreeNodeBranch` components. Manages root-level ARIA context.

**2. What kind of logic is described in this file?**
Defines the `TreeNodeState` enum:
- `COLLAPSED_WITH_JS`: body DOM is not rendered at all (most memory-efficient)
- `COLLAPSED_WITH_CSS`: body DOM rendered but hidden (`aria-hidden`, display none)
- `EXPANDED`: body visible and interactive
- `LOADING`: spinner shown, waiting for lazy-loaded children to mount

Renders `<ul role="tree">` at level 0, `<ul role="group">` at deeper levels. Uses `useTreeNodeRef()` and `useInformParentContextOfChildNodes()` to detect parent-child nesting. Maps items to `TreeNodeBranch` components, memoizing the `renderHeaderIcon` callback.

**3. What part of behavior can be documented from this file?**
- Root `<ul>` has `role="tree"`; nested `<ul>` has `role="group"` — required by ARIA tree pattern.
- Null or empty item arrays render nothing.
- `renderHeaderIcon` callback is memoized on `[collapsedIcon, expandedIcon, showCustomIcon, animateIcon]`.
- The `COLLAPSED_WITH_JS` vs `COLLAPSED_WITH_CSS` distinction affects both memory (no DOM) and animation (DOM alive for transition).

**4. Is it user-facing?**
No — internal component.

**5. What new did you learn from this file?**
The transition from `COLLAPSED_WITH_JS` to `EXPANDED` goes through `LOADING` for the first expand (lazy load). Subsequent collapses use `COLLAPSED_WITH_CSS` (DOM kept alive), so re-expanding is instant without a loading state. This means the loading spinner only appears on the very first expand of a node, not on subsequent collapses and re-expansions.

---

## src/components/TreeNodeBranch.tsx

**1. What is the purpose of this file?**
Renders a single tree node with header, expand/collapse icon, and collapsible body. Manages node state machine transitions, lazy-loading detection, and leaf-node communication.

**2. What kind of logic is described in this file?**
State machine transitions: `COLLAPSED_WITH_JS` → click → `LOADING`; `LOADING` → children found → `EXPANDED`; `EXPANDED` → click → `COLLAPSED_WITH_CSS`; `COLLAPSED_WITH_CSS` → click → `EXPANDED`. `toggleTreeNodeContent()` captures current height, changes state, and triggers `useAnimatedTreeNodeContentHeight()`. `informParentOfChildNodes(numberOfNodes)` updates `isActualLeafNode` when child count changes between 0 and positive. Lazy loading detected by `hasNestedTreeNode()` checking if body's last child has class `"widget-tree-node"`.

DOM structure: `<li class="widget-tree-node-branch" role="treeitem"> → <span class="widget-tree-node-branch-header"> → [headerContent + optional icon] + optional body`. Body: `<div class="widget-tree-node-body" aria-hidden={state !== EXPANDED}>`. Icon placed with class `widget-tree-node-branch-header-reversed` when `iconPlacement === "left"`.

**3. What part of behavior can be documented from this file?**
- `isActualLeafNode = isUserDefinedLeafNode || !children` — node with no body content is also a leaf.
- Body conditionally renders when `(!isActualLeafNode && state !== COLLAPSED_WITH_JS) || isAnimating` — keeps body alive during animation.
- `aria-hidden` on body responds to `state !== EXPANDED`, hiding collapsed content from screen readers.
- `tabIndex={0}` on `<li>` — all tree items are keyboard focusable.
- Header clickable only when `openNodeOn === "headerClick"` AND node is not a leaf.
- Icon clickable only when `openNodeOn === "iconClick"` AND node is not a leaf.

**4. Is it user-facing?**
No — internal component, but drives all user-visible node behavior.

**5. What new did you learn from this file?**
The `isAnimating` flag in the body render condition is an architectural choice: even when state is `COLLAPSED_WITH_CSS`, the body DOM must exist during the closing animation or the height transition has nothing to animate. Without this flag, the DOM would be removed before the CSS transition completes, causing an instant disappear rather than a smooth collapse.

---

## src/components/TreeNodeBranchContext.ts

**1. What is the purpose of this file?**
React context for parent-child communication between nested TreeNode instances, carrying nesting level and a callback for child nodes to report their count to parent nodes.

**2. What kind of logic is described in this file?**
`TreeNodeBranchContextProps`: `{ level: number; informParentOfChildNodes: (n: number | undefined) => void }`. Context initialized with `level: 0`. Each nesting level increments: `level: currentContextLevel + 1`. `useInformParentContextOfChildNodes()` hook: fires `informParentOfChildNodes()` when `level > 0` and `identifyParentIsTreeNode()` returns true. The `identifyParentIsTreeNode()` check examines if the immediate parent's className includes `"widget-tree-node-body"`.

**3. What part of behavior can be documented from this file?**
- Context level 0 = root tree; levels 1+ = nested.
- Only communicates up if parent is a direct TreeNode body — prevents false positives when other widgets wrap a TreeNode.
- The `informParentOfChildNodes` callback lets parent detect when lazy-loaded children appear/disappear.

**4. Is it user-facing?**
No — internal context mechanism.

**5. What new did you learn from this file?**
The `identifyParentIsTreeNode()` guard is specifically designed to handle the case where a `<TreeNode>` is wrapped inside a third-party widget. Without this check, a wrapping widget's container would appear as the "parent," causing incorrect parent-child communication. The className check (`"widget-tree-node-body"`) ensures only genuine TreeNode parent-child relationships trigger the callback.

---

## src/components/hooks/useAnimatedHeight.tsx

**1. What is the purpose of this file?**
Custom hook that orchestrates smooth expand/collapse height animation using CSS transitions and inline style manipulation.

**2. What kind of logic is described in this file?**
Three-step animation sequence:
1. `captureElementHeight()`: reads `getBoundingClientRect().height` and stores in a ref.
2. `animateTreeNodeContent()`: reads new height, sets inline `height` to old value, then after a 1ms `setTimeout` sets inline `height` to new value (triggering CSS transition), sets `isAnimating = true`.
3. `cleanupAnimation()`: called on `transitionEnd`, sets `isAnimating = false`, removes inline `height` style (reverts to auto).

Returns `{ isAnimating, captureElementHeight, animateTreeNodeContent, cleanupAnimation }`.

**3. What part of behavior can be documented from this file?**
- The 1ms timeout is required to force the browser to paint the "old height" frame before transitioning to "new height" — without it, both changes batch into one frame and no animation occurs.
- Animation only fires when old height ≠ new height.
- After animation completes, inline height is removed so content can resize naturally (not locked to a pixel value).
- Relies on external CSS to define `transition: height <duration>` on `.widget-tree-node-body`.

**4. Is it user-facing?**
No — internal animation hook.

**5. What new did you learn from this file?**
This is the classic CSS height animation trick: `height: auto` cannot be directly animated in CSS, so JavaScript captures numeric heights and transitions between them. The 1ms setTimeout is a minimal "browser paint flush" — the smallest delay that reliably separates the two style mutations into distinct render frames. The `getBoundingClientRect()` call before the timeout triggers a layout flush, ensuring the old height is computed correctly before the animation begins.

---

## src/components/hooks/lazyLoading.ts

**1. What is the purpose of this file?**
Detects whether a TreeNode's body element contains a lazily-loaded child TreeNode widget.

**2. What kind of logic is described in this file?**
`elementHasNestedTreeNode(element)`: checks if `element.lastElementChild?.className.includes("widget-tree-node")`. Returns true by default if element is null or class check fails. `useTreeNodeLazyLoading()`: wraps the check in `useCallback` with no dependencies, returns `{ hasNestedTreeNode }`.

**3. What part of behavior can be documented from this file?**
- Only checks the last child element — assumes the child TreeNode renders last in the body slot.
- Defaults to `true` (children assumed to exist) as a safe fallback — prevents premature leaf-node declaration.
- Called during `LOADING` state to determine whether to transition to `EXPANDED`.

**4. Is it user-facing?**
No — internal lazy-loading detection.

**5. What new did you learn from this file?**
The "default to true" safety fallback means: when uncertain, assume children exist and keep the node in `LOADING` state. This prevents a node from incorrectly becoming a leaf while children are still mounting. The trade-off is that if children genuinely don't mount (configuration error), the node stays in `LOADING` forever — but this is safer than falsely declaring a leaf and hiding children.

---

## src/components/hooks/TreeNodeAccessibility.tsx

**1. What is the purpose of this file?**
Implements WAI-ARIA compliant keyboard navigation for the tree widget: arrow keys, Home/End, Enter/Space, with full support for nested tree traversal.

**2. What kind of logic is described in this file?**
Key handler map (via `useKeyboardHandler`):
- `Enter` / `Space` → `toggleTreeNodeContent` (expand/collapse)
- `Home` → focus first visible treeitem
- `End` → focus last visible treeitem
- `ArrowUp` → focus previous sibling (or exit to parent with `VERTICAL` traversal)
- `ArrowDown` → focus next sibling (or enter children with `VERTICAL` traversal)
- `ArrowRight` → if collapsed: expand; if expanded: focus first child
- `ArrowLeft` → if expanded: collapse; if collapsed or leaf: focus parent

Visible nodes are computed by querying all `.widget-tree-node[role="tree"]` in `document.body` and filtering out nodes inside `aria-hidden="true"` bodies. `FocusTargetChange` enum: `FIRST`, `LAST`, `PREVIOUS`, `NEXT`.

**3. What part of behavior can be documented from this file?**
- ArrowRight has two behaviors: expands if collapsed, OR moves focus to first child if already expanded.
- ArrowLeft has two behaviors: collapses if expanded, OR moves focus to parent if collapsed/leaf.
- Home/End are NOT circular — clamped to first/last, no wrapping.
- Collapsed sections (inside `aria-hidden="true"`) are invisible to keyboard navigation.
- Focus scope is the entire document tree, not local to a container — consistent focus across multiple trees on the same page.

**4. Is it user-facing?**
Yes — keyboard navigation is a core user interaction for accessibility.

**5. What new did you learn from this file?**
The ArrowRight/ArrowLeft context-aware behavior is the ARIA tree pattern: "right" doesn't just expand, it also navigates into already-expanded children. "Left" doesn't just collapse, it also navigates out of collapsed/leaf nodes to their parent. This matches the W3C ARIA Authoring Practices Guide tree navigation specification — a tree is navigated like a file explorer, not a standard list.

---

## src/components/hooks/useKeyboardHandler.tsx

**1. What is the purpose of this file?**
Generic keyboard event dispatch hook that maps `event.key` values to named handler functions and handles common event concerns.

**2. What kind of logic is described in this file?**
Maps key strings to handler names: `" "` (space) → `"Space"`, plus `"Enter"`, `"Home"`, `"End"`, `"ArrowUp"`, `"ArrowDown"`, `"ArrowLeft"`, `"ArrowRight"`. Returns a memoized handler that: (1) checks `event.currentTarget === event.target` (not bubbled); (2) looks up handler name from key map; (3) calls matching handler; (4) calls `preventDefault()` and `stopPropagation()`.

**3. What part of behavior can be documented from this file?**
- Only handles events that originate directly on the element (not bubbled from children).
- Space bar (` `) is mapped to `"Space"` — needed because `" "` is the actual `event.key` value for space.
- `preventDefault()` suppresses browser defaults (scrolling on arrow keys, form submission on Enter).
- `stopPropagation()` prevents key events from triggering outer widgets.

**4. Is it user-facing?**
No — internal keyboard event utility.

**5. What new did you learn from this file?**
The `currentTarget === target` guard is essential for a widget like TreeNode where nested nodes are children of other nodes. Without this guard, pressing ArrowDown on a child node would bubble up and trigger the parent's ArrowDown handler too, causing double focus movement. The guard ensures each node only handles events fired directly on it.

---

## typings/TreeNodeProps.d.ts

**1. What is the purpose of this file?**
Auto-generated TypeScript typings from `TreeNode.xml`. Defines `TreeNodeContainerProps`, `TreeNodePreviewProps`, and all enum types.

**2. What kind of logic is described in this file?**
Enums: `HeaderTypeEnum` ("text"|"custom"), `OpenNodeOnEnum` ("headerClick"|"iconClick"), `ShowIconEnum` ("left"|"right"|"no"). `TreeNodeContainerProps`: `datasource: ListValue`, `hasChildren: ListExpressionValue<boolean>`, `headerType`, `openNodeOn`, `headerCaption?: ListExpressionValue<string>`, `headerContent?: ListWidgetValue`, `startExpanded: boolean`, `expandedIcon?: DynamicValue<WebIcon>`, `collapsedIcon?: DynamicValue<WebIcon>`, `animateIcon: boolean`, `animate: boolean`, `showIcon: ShowIconEnum`, plus system props. `TreeNodePreviewProps`: `trigger` and `htmlMessage` as `{ widgetCount: number; renderer: ComponentType }` objects.

**3. What part of behavior can be documented from this file?**
- `hasChildren` is typed as `ListExpressionValue<boolean>` — per-item expression since v3.8.0.
- `headerCaption` is `ListExpressionValue<string>` — can be a per-row expression, not just a literal.
- Icons are `DynamicValue<WebIcon>` (optional) — widget works without custom icons (defaults to chevron).
- Both `animateIcon` and `animate` are separate booleans — icon animation and content animation are independently controlled.

**4. Is it user-facing?**
No — internal type declarations.

**5. What new did you learn from this file?**
The existence of two separate animation flags (`animateIcon` and `animate`) confirms that the container combines them with `&&` — you need both enabled to get animated icon rotation during animated content expand/collapse. A user who enables content animation but disables icon animation gets a sliding body with a non-rotating icon.

---

## src/TreeNode.editorConfig.ts

**1. What is the purpose of this file?**
Provides `getProperties()` (conditional property visibility) and `getPreview()` (structure preview layout) for Studio Pro. Controls which configuration options appear based on current selections.

**2. What kind of logic is described in this file?**
`getProperties()`: hides icon-related properties when `showIcon === "no"`; hides `headerContent` and `openNodeOn` when `headerType === "text"` (text header needs no custom content); hides `headerCaption` when `headerType === "custom"`; hides `startExpanded` and `children` when `!hasChildren`; uses `advancedMode` toggle to show/hide advanced properties (icon animation, content animation) in Studio (not Pro). `getPreview()`: renders structure preview with dark/light palette, chevron icon, header, and nested rows.

**3. What part of behavior can be documented from this file?**
- Advanced properties (icon type, animations) are hidden in Studio by default unless `advancedMode` is enabled.
- Studio Pro always shows all properties regardless of `advancedMode`.
- When `headerType="text"`, the `openNodeOn` property is hidden — text headers are always fully clickable.
- Structure preview shows the expand/collapse visual structure with icon and header.

**4. Is it user-facing?**
Yes — controls the Studio Pro property panel experience.

**5. What new did you learn from this file?**
When `headerType="text"`, the `openNodeOn` property is hidden. This implies that text headers always use `headerClick` mode — the entire header is clickable. The `openNodeOn` distinction (header vs icon) only makes sense when there's custom `headerContent` that itself might contain interactive elements — hiding the option for text headers prevents confusion.

---

## src/TreeNode.editorPreview.tsx

**1. What is the purpose of this file?**
Renders the TreeNode widget preview in Mendix Studio design canvas. Always shows nodes as expanded with animation disabled.

**2. What kind of logic is described in this file?**
Maps `TreeNodePreviewProps` to `TreeNodeProps`. Forces `startExpanded=true` and `animateTreeNodeContent=false` for preview. Renders a single sample item with `headerContent` via `props.headerContent.renderer` or `headerCaption` text. Uses renderer captions: "Place header contents here" and "Place other tree nodes here".

**3. What part of behavior can be documented from this file?**
- Preview always starts expanded — shows full structure in Studio.
- No animation in preview — avoids confusing designers with animation on canvas load.
- Shows one sample node with placeholder dropzones.

**4. Is it user-facing?**
Yes — visible to developers in Studio Pro.

**5. What new did you learn from this file?**
The "Place other tree nodes here" placeholder text in the children dropzone confirms the widget is specifically designed for recursive/nested TreeNode configurations. The widget's primary use case is nesting TreeNode widgets inside other TreeNode body slots to create hierarchical trees.

---

## src/components/__tests__/TreeNode.spec.tsx

**1. What is the purpose of this file?**
Comprehensive Jest unit tests for the TreeNode component, covering rendering, interaction, keyboard navigation, lazy loading, nested structures, and leaf node behavior.

**2. What kind of logic is described in this file?**
Tests: collapsed/expanded DOM structure; header click to expand/collapse; Space/Enter key toggle; ArrowUp/ArrowDown for sibling navigation; ArrowRight (expand then enter children); ArrowLeft (collapse then exit to parent); Home/End (first/last node); LOADING state for first expand; detecting nested TreeNodes (`hasNestedTreeNode`); parent-child communication (`informParentOfChildNodes`); wrapping in random widgets (isolation test); leaf node behavior (no icon, not clickable); icon animation CSS class; render performance (spy for minimal re-renders). Uses `jest.useFakeTimers()` for the 1ms animation timeout.

**3. What part of behavior can be documented from this file?**
- Default: `startExpanded=false`, `openNodeOn="headerClick"`, `animateIcon=false`, `animateTreeNodeContent=false`.
- ArrowRight: if node is collapsed, expands it; if node is already expanded, moves focus to first child.
- ArrowLeft: if node is expanded, collapses it; if collapsed/leaf, moves focus to parent.
- Home/End not circular — clamped.
- `role="treeitem"` on each `<li>`, `role="tree"` on root `<ul>`, `role="group"` on nested `<ul>`.

**4. Is it user-facing?**
No — test file only.

**5. What new did you learn from this file?**
The isolation test (wrapping TreeNode in a random widget) specifically validates the `identifyParentIsTreeNode()` fix: a child TreeNode inside `<RandomWidget><TreeNode /></RandomWidget>` should NOT trigger parent-child communication with TreeNodeBranch. This was a regression scenario where third-party widget wrappers caused false leaf-node detection.

---

## e2e/TreeNode.spec.js

**1. What is the purpose of this file?**
End-to-end Playwright tests for the TreeNode widget, including functional interaction tests and Axe accessibility compliance validation.

**2. What kind of logic is described in this file?**
Tests: expand single and multiple nodes (click header, verify body visibility); collapse single and multiple nodes; Axe WCAG 2.1 AA accessibility violations check (excludes `navigationTree3` which is a different widget). Interacts with `.mx-name-treeNode1` widget, clicks `.widget-tree-node-branch-header-value` headers. Uses 0.1% screenshot tolerance.

**3. What part of behavior can be documented from this file?**
- Header element class: `.widget-tree-node-branch-header-value`.
- Expand/collapse tested with real Mendix app and real datasource data.
- Axe accessibility test validates WCAG 2.1 AA compliance for the full tree structure.
- Screenshot tolerance is 0.1% — much tighter than most other widgets (chart widgets use 10-50%).

**4. Is it user-facing?**
The tested behaviors (expand/collapse, keyboard accessibility) are user-facing.

**5. What new did you learn from this file?**
The 0.1% screenshot tolerance (vs 10-50% for chart widgets) reflects that the tree widget renders pure HTML/CSS without anti-aliasing or GPU-rendered graphics — it should be pixel-perfect across environments. The Axe test is specifically noteworthy: it validates that ARIA `role="tree"`, `role="treeitem"`, `role="group"`, `aria-expanded`, `aria-hidden`, and `tabIndex` attributes are all correctly implemented per WCAG 2.1 AA.

---

## CHANGELOG.md

**1. What is the purpose of this file?**
Documents version history for the tree-node-web widget from initial release (1.0.0, 2021-09-28) to current (3.8.0).

**2. What kind of logic is described in this file?**
Key versions: v3.8.0 (2026-01-16, `hasChildren` changed to expression, fixed lazy-loading spinner issue); v1.2.1 (2024-11-13, fixed collapse state reset during data reload); v1.2.0 (2024-05-15, fixed nested empty TreeNode breaking parent behavior); v1.1.3 (2023-08-22, fixed keyboard input in form widgets + added `openNodeOn` property — breaking HTML change); v1.1.2 (2023-08-10, fixed Atlas icon display, fixed child refresh); v1.0.0 (2021-09-28, initial release).

**3. What part of behavior can be documented from this file?**
- v3.8.0 changed `hasChildren` to a per-item expression — potentially breaking for existing configurations using static boolean.
- v1.2.1 fixed collapse state reset on data reload — confirms the `ValueStatus.Available` guard is a specific fix, not just defensive coding.
- v1.1.3 introduced `openNodeOn` and changed HTML structure (breaking CSS customizations).
- The lazy-loading spinner bug (fixed in v3.8.0) was caused by the old static `hasChildren` not being able to detect that a node has no children, leaving it in `LOADING` indefinitely.

**4. Is it user-facing?**
The changelog is publicly visible on the Mendix Marketplace.

**5. What new did you learn from this file?**
The `hasChildren` change in v3.8.0 solved a real bug: with the old static `hasChildren`, every node appeared to have children (or no children), causing the lazy-loading spinner to either always appear or never appear. The per-item expression allows the widget to know per-row whether to expect children, enabling the `LOADING → EXPANDED` vs `LOADING → no children (leaf)` distinction to work correctly.
