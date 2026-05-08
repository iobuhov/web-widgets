# Draft: accordion-web

Extracted by worker on 2026-05-08. Covers all source files and local workspace dependencies.

---

## src/Accordion.tsx

**Purpose:** Main widget entry point. Bridges Mendix pluggable widget props to the internal Accordion component.

**Logic:** Receives `AccordionContainerProps` from the Mendix runtime. Checks if any group data is still loading (dynamic initial state or controlled `collapsed` attribute), returning `null` until ready. Translates raw Mendix group props into the internal `AccordionGroups` format via `translateGroups`. Constructs the icon generator via `useIconGenerator`. Renders `<AccordionComponent>` only when all data is available.

**Behavioral constraints from this file:**
- The widget returns `null` (renders nothing) if any group with `initialCollapsedState === "dynamic"` has `initiallyCollapsed.status === "loading"`, or if any group's `collapsed` attribute is still loading.
- When `collapsible` is false, `singleExpandedGroup` is explicitly set to `undefined` (single-expand behavior is disabled entirely).
- `expandBehavior === "singleExpanded"` is only applied when `collapsible` is `true`.
- `someGroupMissingData` also returns `undefined` (and thus `null` render) if `visible`, `headerText`, `initiallyCollapsed`, or `collapsed` values are not yet resolved.

**User-facing:** Yes — this is the top-level component rendered in the Mendix page.

**New findings:** The widget uses `useRef(generateUUID())` for the accordion ID, ensuring a stable, globally-unique ID per mount that avoids duplication across widget instances (changelog v2.0.1 fix). The `onToggleCompletion` callback is wired to `group.collapsed.setValue`, enabling two-way binding between the accordion's collapse state and a Mendix entity attribute.

---

## src/Accordion.xml

**Purpose:** Widget descriptor file that declares all configurable properties exposed to Mendix Studio/Studio Pro.

**Logic:** Defines the widget metadata (id, name, category, offlineCapable) and its full property tree. Groups are defined as an `isList` object property. Each group carries header, content, visibility, state, and load-content settings. Top-level props handle collapsibility, expand behavior, animation, and icon configuration.

**Behavioral constraints from this file:**
- `offlineCapable="true"` means the widget functions in offline Mendix apps.
- `loadContent` per group accepts `"always"` (default) or `"whenExpanded"` — the latter defers widget rendering and data fetching until the group is first expanded.
- `initialCollapsedState` has three values: `"expanded"`, `"collapsed"` (default), `"dynamic"` (driven by expression).
- `collapsed` is a Boolean attribute linked to `onToggleCollapsed` action for two-way state binding.
- `advancedMode` (boolean, default `false`) gates visibility of advanced properties in Studio.
- Icon properties (`icon`, `expandIcon`, `collapseIcon`, `animateIcon`, `showIcon`) are under the "Visualization" group; `showIcon` controls position (right/left/no).

**User-facing:** Yes — every property here is configurable by the Mendix developer.

**New findings:** The widget is explicitly categorized under "Structure" in both Studio and Studio Pro toolboxes. The `helpUrl` points to the official Mendix docs for this widget.

---

## typings/AccordionProps.d.ts

**Purpose:** Auto-generated TypeScript type definitions derived from `Accordion.xml`. Provides compile-time types for all widget props used throughout the codebase.

**Logic:** Exports enum types for `HeaderRenderModeEnum`, `HeaderHeadingEnum`, `LoadContentEnum`, `InitialCollapsedStateEnum`, `ExpandBehaviorEnum`, `ShowIconEnum`. Exports `GroupsType` (runtime group object), `AccordionContainerProps` (full widget props at runtime), `GroupsPreviewType` (group props in editor preview), and `AccordionPreviewProps` (full props in editor preview).

**Behavioral constraints from this file:**
- `collapsed` in `GroupsType` is `EditableValue<boolean>` — optional, allowing groups that are not bound to a Mendix attribute.
- `icon`, `expandIcon`, `collapseIcon` in `AccordionContainerProps` are `DynamicValue<WebIcon> | undefined` — optional icon configurations.
- `AccordionPreviewProps` includes a `renderMode: "design" | "xray" | "structure"` field that is used to distinguish Studio Pro preview contexts.
- The `className` field in `AccordionPreviewProps` is deprecated since v9.18.0 in favor of `class`.

**User-facing:** Internal — developers interact via generated type safety.

**New findings:** The preview props expose icon fields as discriminated union types (`glyph | image | icon`), whereas runtime props use the Mendix `WebIcon` abstraction. This is the contract boundary between Studio Pro preview and runtime rendering.

---

## src/Accordion.editorConfig.ts

**Purpose:** Configures how the widget appears and behaves in the Mendix Studio Pro property panel and structure-mode preview.

**Logic:** `getProperties` dynamically hides/shows properties based on current values — e.g., hiding `headerContent` when `headerRenderMode === "text"`, hiding `headerText`/`headerHeading` when in custom mode, hiding collapse-related props when `collapsible` is false, hiding icon props when `showIcon === "no"`. On web platform, calls `transformGroupsIntoTabs` to render each group as a tab in the property panel. `check` validates that when `expandBehavior === "singleExpanded"`, at most one group has `initialCollapsedState === "expanded"`. `getPreview` renders a structure-mode preview of the accordion using the structure preview API.

**Behavioral constraints from this file:**
- When `animateIcon` is true, `expandIcon` and `collapseIcon` are hidden (only the single animated `icon` is used). When false, `icon` is hidden and `expandIcon`/`collapseIcon` are shown.
- When `!collapsible`, all of `expandBehavior`, `animate`, `showIcon`, `icon`, `expandIcon`, `collapseIcon`, `animateIcon` are hidden.
- In simple web mode (`!advancedMode`), `animate`, `showIcon`, and all icon props are hidden from Studio users.
- The studio validation error fires when `expandBehavior === "singleExpanded"` AND more than one group starts as expanded — this is an error, not a warning.
- `advancedMode` property itself is hidden on the desktop platform.

**User-facing:** Editor-only — affects property panel UX and structure preview; not visible to end users at runtime.

**New findings:** Icon size in the structure preview is hardcoded to 14px width. Header text font sizes in the preview are mapped by heading level (h1→13px, h2→12px, …, h6→8px), used purely for visual fidelity in the structure view.

---

## src/Accordion.editorPreview.tsx

**Purpose:** Implements the live preview of the accordion inside Mendix Studio Pro's design/xray mode.

**Logic:** Exports `getPreviewCss` (returns SCSS for the preview), `PreviewComponent` (the React component), and `preview` (entry point called by Studio Pro). Maps `AccordionPreviewProps` to `AccordionGroups`, renders the real `Accordion` component with `previewMode` flag. Falls back to a single placeholder group if no groups are configured.

**Behavioral constraints from this file:**
- In preview mode, the `Accordion` component receives `previewMode={true}`, which disables initial collapse calculation (all groups appear expanded).
- `dynamicClassName` in preview is derived by stripping surrounding single quotes from the expression string (`group.dynamicClass.slice(1, -1)`).
- Uses `mapPreviewIconToWebIcon` (from `widget-plugin-platform`) to convert the icon union type to the runtime `WebIcon` type.
- If no groups exist, a placeholder group with fixed header "[No groups configured]" is shown.

**User-facing:** Editor-only — renders inside Studio Pro, not at runtime.

**New findings:** The preview reuses the exact same `Accordion` and `useIconGenerator` as the runtime, ensuring visual fidelity in Studio Pro.

---

## src/components/Accordion.tsx

**Purpose:** Core presentational container component. Manages the array of collapsed states for all groups and renders them.

**Logic:** Uses `useReducer` with a reducer strategy (`single` or `multiple` expand mode) to maintain `accordionGroupCollapsedState[]`. `reducerInitialState` computes initial collapse states from group `initiallyCollapsed` values; in single-expand mode, only the last non-collapsed group stays expanded, forcing all preceding ones collapsed. A `useMemo` on `props.groups` detects external changes: if `initiallyCollapsed` values changed, resets state; if a `collapsed` attribute value changed, dispatches to sync it. Keyboard navigation is handled in `AccordionGroupWrapper.focusAccordionGroupHeaderElement` by querying all header buttons within the container.

**Behavioral constraints from this file:**
- In `singleExpanded` mode at initialization, if multiple groups have `initiallyCollapsed = false`, only the **last** one stays expanded; all preceding are forced collapsed.
- The `collapsed` external attribute takes precedence over internal state during normal operation (synced via `useMemo`). However, `initiallyCollapsed` changes trigger a full reset.
- Keyboard focus targets: Home → first header, End → last header, ArrowUp → previous header, ArrowDown → next header.
- The `AccordionGroupWrapper` key is `${group.initiallyCollapsed}_${index}`, so when `initiallyCollapsed` changes, React remounts the group — this ensures clean state but is a notable re-mount behavior.
- In `previewMode`, `reducerInitialState` sets all groups to `collapsed = false` (all shown expanded).

**User-facing:** Yes — renders the accordion container `<div>` with `widget-accordion` class.

**New findings:** The accordion is ARIA-compliant: header buttons have `aria-expanded`, `aria-disabled`, `aria-controls` pointing to the content region, which has `role="region"` and `aria-labelledby` linking back to the header button.

---

## src/components/AccordionGroup.tsx

**Purpose:** Renders a single accordion group with animated expand/collapse behavior and keyboard support.

**Logic:** Manages local `renderCollapsed` state and animates transitions using CSS class toggling and explicit `height` manipulation via JS. Uses a `useEffect` to detect prop-to-state divergence and trigger CSS animation. On expand: adds `widget-accordion-group-expanding`, sets height from content's `getBoundingClientRect`; on collapse: first sets height then clears it after 50ms (triggering CSS transition to 0). A `ResizeObserver` (debounced) adjusts content wrapper height when the content size changes while expanded. The `completeTransitioning` callback (called on `transitionend`) removes transitioning classes and settles final state.

**Behavioral constraints from this file:**
- When `animateContent` is false or the group is not visible, state transitions happen immediately without animation.
- Content is only rendered in the DOM when `renderContent.current` is true: always when `loadContent === "always"`, or lazily once first expanded (and remains in DOM thereafter).
- `onToggleCompletion` (Mendix attribute setter) is called **after** the visual transition completes (`renderCollapsed` change), not immediately on click.
- Keyboard handlers: Enter, Space → toggle; Home → first; End → last; ArrowUp → previous; ArrowDown → next.
- When collapsed, the CSS class `widget-accordion-group-collapsed` hides content via `display: none`.
- Resize observer debounce is 32ms; it keeps content wrapper height in sync when content inside changes size while the group is expanded.

**User-facing:** Yes — each group is a `<section>` with `<header>` and content region.

**New findings:** The `completeTransitioning` function has a defensive check for state/class-list de-sync: if `renderCollapsed` and the presence of `widget-accordion-group-expanding/collapsing` disagree, it corrects state. This guards against missed `transitionend` events.

---

## src/components/Header.tsx

**Purpose:** Renders the text header of an accordion group as the appropriate semantic HTML heading element.

**Logic:** Accepts a `heading` enum (`headingOne`…`headingSix`) and renders children inside `<h1>`…`<h6>` accordingly. Used only when `headerRenderMode === "text"`.

**Behavioral constraints from this file:**
- The heading level is purely semantic/accessibility — the visual font size is controlled by CSS, not by the heading level choice itself (the SCSS sets `font-size: 18px` uniformly for all heading elements inside the header button).
- `headingOne` is the default HTML element (`h1`) if no match — but `headingThree` is the default prop value in the XML.

**User-facing:** Yes — rendered as visible header text inside the collapsible header button.

**New findings:** The component is a thin semantic wrapper, enabling correct document outline and screen reader heading navigation without imposing visual differentiation.

---

## src/components/Icon.tsx

**Purpose:** Renders the expand/collapse indicator icon inside the accordion group header.

**Logic:** If no `data` (no custom icon configured) and not loading, renders an inline SVG chevron (downward-pointing). If `data` is provided, renders via `IconInternal` from the widget-plugin-component-kit. Applies `widget-accordion-group-header-button-icon-animate` class when `animate` is true.

**Behavioral constraints from this file:**
- While an icon is loading (`loading === true`) and no `data` is present, renders `null` (no placeholder).
- The default chevron SVG is always rendered facing downward; rotation to indicate expanded/collapsed state is achieved via CSS (`rotate(-180deg)` when expanded, `transform: none` when collapsed or collapsing).
- The `animate` prop controls whether the CSS transition is applied.

**User-facing:** Yes — the icon is visible in the header button when `showIcon !== "no"`.

**New findings:** The default icon is always the same SVG; the direction semantics (up vs. down) are entirely CSS-driven. Custom icons (glyph, image, icon type) are delegated to `IconInternal`.

---

## src/utils/reducers.ts

**Purpose:** Provides the state reducer for the array of group collapsed states.

**Logic:** `getCollapsedAccordionGroupsReducer` returns a reducer function parameterized by `expandMode` ("single" or "multiple"). Actions: `"expand"` in single mode collapses all other groups (sets all to `true` except `action.index`); `"expand"` in multiple mode just sets `state[index] = false`; `"collapse"` sets `state[index] = true`; `"reset"` replaces the entire state array with `action.values`.

**Behavioral constraints from this file:**
- In `"single"` mode, expanding any group instantly collapses all others — the reducer returns a new array with all `true` except the target index.
- The reducer is stable: instantiated once per `Accordion` mount (stored in `useRef`) since `singleExpandedGroup` won't change during the component's lifetime.

**User-facing:** Internal logic; no direct UI surface.

**New findings:** The `"reset"` action enables complete state replacement without remounting, used when `initiallyCollapsed` values change across a re-render.

---

## src/utils/iconGenerator.tsx

**Purpose:** Hook that returns a memoized function to generate the correct icon element for a given collapsed state.

**Logic:** `useIconGenerator` takes `animateIcon` flag and three icon specs (`icon`, `expandIcon`, `collapseIcon`). Returns a callback that, when called with `collapsed: boolean`, renders `<Icon>` with animation when `animateIcon` is true (single animated icon regardless of state), or the appropriate expand/collapse icon without animation.

**Behavioral constraints from this file:**
- When `animateIcon` is true: the same `icon` is always rendered (CSS rotation handles direction); separate `expandIcon`/`collapseIcon` are ignored.
- When `animateIcon` is false: `expandIcon` is shown when collapsed, `collapseIcon` when expanded.
- The callback is memoized via `useCallback` with `[animateIcon, icon, expandIcon, collapseIcon]` as dependencies.

**User-facing:** Indirectly — the generated icon is rendered inside each group's header button.

**New findings:** The icon generation strategy (animate vs. separate icons) directly corresponds to the property hiding logic in `editorConfig.ts` — when `animateIcon` is true, only `icon` is exposed in the editor.

---

## src/utils/resizeObserver.ts

**Purpose:** Provides a debounced ResizeObserver hook to keep the content wrapper height in sync when expanded content changes size.

**Logic:** `CallResizeObserver` directly adjusts `contentWrapperRef` height from `contentRef.getBoundingClientRect().height` on resize events (only when `!renderCollapsed`). `useDebouncedResizeObserver` creates a debounced version of `CallResizeObserver` (32ms debounce) using the local `debounce` utility, then sets up and tears down a `ResizeObserver` on `contentRef`.

**Behavioral constraints from this file:**
- The resize observer is only active while the component is mounted and only adjusts height when the group is not collapsed.
- The 32ms debounce prevents excessive re-layout on rapid resize events.
- Observer teardown on unmount prevents memory leaks.

**User-facing:** Invisible behavior — ensures the CSS height animation remains correct if content inside a group resizes (e.g., dynamic content, lazy-loaded images).

**New findings:** This is needed because the expand animation relies on explicit `height` values (no `height: auto` during transitions). Without this observer, content that grows after initial expansion would be clipped.

---

## src/ui/accordion-main.scss

**Purpose:** Provides all visual styling for the accordion widget.

**Logic:** Defines CSS variables (SCSS vars) for background, border, and hover colors. Styles `.widget-accordion-group` and its descendants. Controls collapsed state (content `display: none`), transitioning states (`widget-accordion-group-collapsing`, `widget-accordion-group-expanding`) with `height` transitions (0.2s ease, 50ms delay). Icon rotation: expanded state applies `rotate(-180deg)` to animate class; collapsed/collapsing states reset to `transform: none`. Header button styles include padding, flex layout, hover/focus/active states with color changes and focus ring.

**Behavioral constraints from this file:**
- Collapsed groups hide content via `display: none` (not `visibility: hidden` or opacity 0), meaning hidden content is fully removed from the layout flow.
- The height transition duration is 0.2s with a 50ms ease delay (matching the JS setTimeout in AccordionGroup).
- Icon animation transition is 0.2s ease-in-out with 50ms delay.
- Focus ring is a 2px solid blue box-shadow (not the browser default outline) — `outline: none` is set explicitly, so focus-visible is the only focus indicator.
- Icon position: right → `margin-left: 16px`; left → flex `row-reverse` with `margin-right: 16px`.
- Preview mode (`.widget-accordion-preview`) applies heading styles to nested `div > :is(h1…h6)` to accommodate how preview renders header content.

**User-facing:** Yes — all visual appearance of the widget.

**New findings:** The 50ms JS `setTimeout` in the expand/collapse animation in `AccordionGroup.tsx` is synchronized with the SCSS `transition-delay: 50ms`, ensuring the height value is set before the CSS transition begins.

---

## packages/shared/widget-plugin-platform/src/framework/generate-uuid.ts

**Purpose:** Generates monotonically increasing integer IDs, stored on the `window` object as a global counter.

**Logic:** Reads and increments `window["com.mendix.widgets.web.UUID"]`, initializing it to 1 if absent.

**Behavioral constraints from this file:**
- IDs are integers, not true UUIDs.
- The counter is page-global: all widgets sharing this mechanism get unique IDs across the full page, preventing duplicate `id` attributes (changelog v2.0.1 fix for inconsistent accessibility IDs).

**User-facing:** Indirectly — the generated ID is used as the accordion element's `id` attribute, referenced by ARIA attributes.

**New findings:** The ID is captured once in a `useRef` at mount time, so it remains stable across re-renders.

---

## packages/shared/widget-plugin-platform/src/utils/debounce.ts

**Purpose:** A simple debounce utility that delays function execution until after a quiet period.

**Logic:** Returns a `[debouncedFn, abortFn]` tuple. The debounced function cancels any pending timeout before scheduling a new one. The abort function clears the timeout without invoking the callback.

**Behavioral constraints from this file:**
- Not a leading-edge debounce — the function always executes after the last call (trailing edge).
- The abort function is returned, allowing explicit cancellation (used by `useDebouncedResizeObserver` implicitly via React cleanup).

**User-facing:** Internal utility; no direct UI surface.

**New findings:** The ResizeObserver in accordion uses a 32ms debounce window, chosen to be within one animation frame (~16ms) * 2 for safety.

---

## packages/shared/widget-plugin-component-kit/src/IconInternal.tsx

**Purpose:** Shared utility component for rendering Mendix `WebIcon` objects (glyph, image, or icon font types).

**Logic:** Discriminates on `icon.type`: `"glyph"` renders a `<span>` with glyphicon classes; `"image"` renders an `<img>` with `aria-hidden` and empty alt; `"icon"` renders a `<span>` with the icon class. Returns `fallback` or `null` if no icon.

**Behavioral constraints from this file:**
- All rendered icon elements have `aria-hidden` — icons are decorative and excluded from the accessibility tree.
- Custom `className` is merged onto the icon element via `classNames`.
- The component has a stable `displayName` for debugging.

**User-facing:** Indirectly yes — rendered inside the accordion header button when a custom icon is configured.

**New findings:** This shared component abstracts the three Mendix icon types, allowing the accordion Icon component to be icon-type-agnostic.

---

## CHANGELOG.md

**Summary of relevant versions:**

- **v2.3.5 (2026-02-09):** Added license file and open-source dependency readme.
- **v2.3.4 (2025-02-13):** Fixed initial collapsed state not always updating the accordion.
- **v2.3.3 (2024-08-28):** Changed action required to false to avoid Studio Pro warnings.
- **v2.3.2 (2024-07-19):** Fixed nested accordion display de-sync between collapsed/uncollapsed content and state.
- **v2.3.1 (2023-09-27):** Removed redundant code for browser load time improvement.
- **v2.3.0 (2023-06-05):** Updated light/dark icons and tiles; changed structure-mode preview colors.
- **v2.2.0 (2023-01-30):** Added `loadContent` property per group to control when widgets are rendered/fetched.
- **v2.1.2 (2022-07-14):** Fixed icon rotation (180deg) when animations are off.
- **v2.1.1 (2022-04-01):** Fixed CSP (Content Security Policy) compatibility.
- **v2.1.0 (2021-12-23):** Added dark mode to structure preview; advanced options hidden for Studio (only shown in Studio Pro by default).
- **v2.0.1 (2021-10-06):** Fixed duplicate accessibility IDs (the generate-uuid fix).
- **v2.0.0 (2021-09-28):** Added toolbox category and tile; renamed advanced options property.
- **v1.1.0 (2021-07-22):** Added `headerRenderMode`, initial collapse state properties, two-way attribute binding, structure preview.
- **v1.0.0 (2021-06-29):** Initial release — groups with composable content, conditional visibility, dynamic classes, collapsibility, expand behavior, icon customization, animations.

**Findings:** The `loadContent` feature (v2.2.0) was added specifically to address page load time issues. The nested accordion de-sync fix (v2.3.2) relates to the `completeTransitioning` defensive sync code observed in `AccordionGroup.tsx`. The v2.3.4 fix for initial collapsed state is reflected in the `useMemo` reset logic in `components/Accordion.tsx`.
