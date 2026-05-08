# Draft: accordion-web

Source path: `packages/pluggableWidgets/accordion-web/`

---

## src/Accordion.tsx

**Purpose:** Root entry point for the Accordion pluggable widget. Bridges Mendix platform props to the internal React component tree.

**Logic:** Reads `AccordionContainerProps` from the Mendix runtime, translates the `groups` array into internal format, resolves dynamic icon props (single animated icon vs separate expand/collapse icons), waits for all group loading states to resolve before rendering, and generates a stable per-mount UUID for accessible element IDs.

**Documentable behavior:** The widget renders `null` until all group data is loaded (no partial renders during data fetch). Each group's `initiallyCollapsed` state can be static ("collapsed"/"expanded") or driven by a Mendix expression ("dynamic"). The `collapsed` attribute is an optional two-way binding — when set, the Mendix attribute governs expanded state and the `onToggleCollapsed` action fires on change.

**User-facing:** Yes — this is the runtime component users interact with.

**New learnings:** The loading guard checks both `initialCollapsedState === "dynamic"` (expression-driven) and the `collapsed` attribute separately. Groups that are missing any required data (visible, headerText, initiallyCollapsed, or collapsed attribute) are treated as incomplete and the whole widget defers rendering. UUID is sequential per page (stored on `window`), not cryptographically random.

---

## src/Accordion.xml

**Purpose:** Mendix widget definition file. Declares all configurable properties, their types, defaults, and Studio Pro groupings.

**Logic:** Defines the schema for widget configuration: a repeatable `groups` list object with per-group state, content, header, and visibility settings; plus top-level behavior (collapsible, expand mode, animation) and visualization (icon position, type, animation) properties. The `advancedMode` boolean gates advanced properties in Studio (web only).

**Documentable behavior:**
- Each group has a `headerRenderMode` (text or custom widgets), `initialCollapsedState` (expanded/collapsed/dynamic), optional two-way `collapsed` Boolean attribute, and `loadContent` (always/whenExpanded).
- `collapsible: false` disables all collapse/expand behavior.
- `expandBehavior` controls single vs. multiple expanded groups simultaneously.
- `showIcon` accepts "right", "left", or "no" positions.
- Icon can be a single animated icon or separate expand/collapse icons (mutually exclusive).
- `loadContent: whenExpanded` defers widget rendering until the group is first expanded — trades slower expand for faster initial page load.

**User-facing:** Indirectly — this file defines what Studio Pro exposes to page designers.

**New learnings:** The widget requires entity context (`needsEntityContext="true"`) and supports offline (`offlineCapable="true"`). The `onToggleCollapsed` action explicitly notes that "Start as" properties can prevent execution on initial state changes — meaning the action only fires on user interaction, not on initial render.

---

## src/Accordion.editorConfig.ts

**Purpose:** Controls which properties are visible/hidden in the Studio Pro property panel depending on current configuration state, and validates configuration correctness.

**Logic:** `getProperties` dynamically hides irrelevant properties: hides `headerContent` when mode is "text", hides `headerText`/`headerHeading` when mode is "custom", hides state-related group props when not in advanced mode or when `collapsible` is false, hides all icon/animation props when `collapsible` is false, and hides icon sub-options based on `showIcon` and `animateIcon` values. `getPreview` generates a structure-mode preview with a title header and per-group preview including header and content drop zones. `check` validates that single-expanded mode does not have more than one group configured to start as expanded.

**Documentable behavior:**
- Validation error: "Expanded groups" set to "Single" while multiple groups start as "Expanded" is an error, not a warning.
- When `animateIcon: true`, separate expand/collapse icons are hidden (only the animated icon is used). When `animateIcon: false`, the single `icon` field is hidden.
- Advanced mode is only surfaced on web; Studio Pro always shows all fields.
- Structure preview shows chevron SVG for icon position and drop zones for group content.

**User-facing:** No — only visible inside Studio Pro property panel.

**New learnings:** The `headerHeading` controls both the rendered HTML tag and the font size in the structure preview (h1=13px down to h6=8px), with corresponding icon padding adjustments (18px down to 13px). Conditional property hiding is fully dynamic per-group by index.

---

## src/Accordion.editorPreview.tsx

**Purpose:** Provides a live preview of the Accordion widget inside the Mendix Studio canvas (design/preview mode).

**Logic:** Renders a full `Accordion` component using preview props, mapping preview-specific icon types to web icon format, synthesizing group data from preview prop shapes, and injecting a `previewMode` flag that suppresses collapse animation and initial-state logic. Falls back to a placeholder group when no groups are configured.

**Documentable behavior:** In preview mode, collapsed state is not applied — all groups render as if expanded. Dynamic class expressions are extracted from single-quote-wrapped expression strings. `getPreviewCss` injects the production SCSS at design time, so the canvas preview reflects actual widget styling.

**User-facing:** No — Studio canvas only.

**New learnings:** Dynamic class in preview mode is stored as a stringified expression (e.g., `'myClass'`), and the preview strips the surrounding quotes via `.slice(1, -1)`. The `previewMode` prop disables initial state computation in the Accordion component.

---

## typings/AccordionProps.d.ts

**Purpose:** Auto-generated TypeScript type definitions derived from `Accordion.xml`. Provides compile-time types for both runtime container props and Studio preview props.

**Logic:** Declares enums for `HeaderRenderModeEnum`, `HeaderHeadingEnum`, `LoadContentEnum`, `InitialCollapsedStateEnum`, `ExpandBehaviorEnum`, and `ShowIconEnum`. Defines `GroupsType` for runtime (using `DynamicValue`, `EditableValue`) and `GroupsPreviewType` for Studio preview (using plain strings). `AccordionContainerProps` is the runtime interface; `AccordionPreviewProps` is the preview interface.

**Documentable behavior:** `collapsed` in `GroupsType` is `EditableValue<boolean>` (two-way binding, optional). `visible` and `initiallyCollapsed` are `DynamicValue<boolean>` (expression-driven, one-way). `headerText` is `DynamicValue<string>`. Icons are `DynamicValue<WebIcon>` (optional). The preview `icon` types union supports glyph, image, and icon class formats.

**User-facing:** No — compile-time only.

**New learnings:** `GroupsType` has no `onToggleCollapsed` — that is wired internally by the Mendix runtime via the `onChange` attribute in the XML. The distinction between `DynamicValue` (expression, read-only) and `EditableValue` (attribute, read-write) is central to understanding which properties support two-way binding.

---

## CHANGELOG.md

**Purpose:** Tracks version history and notable changes for the accordion-web widget.

**Logic:** Semantic versioning from 1.0.0 (2021-06-29) to 2.3.5 (2026-02-09).

**Documentable behavior:**
- v2.3.4 (2025-02-13): Fixed initial collapsed state not updating in some cases.
- v2.3.2 (2024-07-19): Fixed nested mode sync between displayed state and actual collapsed state.
- v2.2.0 (2023-01-30): Added "Load content" property (always vs. whenExpanded).
- v2.1.0 (2021-12-23): Added dark mode to structure preview; advanced mode hidden from Studio Pro (always shows all props).
- v1.1.0 (2021-07-22): Added header render mode (text/custom), initial collapsed state control, collapsed attribute binding, structure mode preview.
- v1.0.0 (2021-06-29): Initial release — groups with composable content, conditional visibility, dynamic classes, collapsible behavior, icon customization, animations.

**User-facing:** No — developer/operator reference.

**New learnings:** The `onToggleCollapsed` action was changed to `required: false` in v2.3.3 to avoid spurious Studio Pro warnings. The initial collapsed state bug fix in v2.3.4 is relevant to the "Start as" / collapsed attribute interaction.

---

## src/components/AccordionGroup.tsx

**Purpose:** Renders a single accordion group section — its header button, content region, expand/collapse animation, keyboard navigation, and accessibility attributes.

**Logic:** Manages local `renderCollapsed` state (distinct from prop `collapsed` to decouple animation timing). Animates via CSS class toggling (`widget-accordion-group-collapsing`, `widget-accordion-group-expanding`) and inline height transitions. Uses `useDebouncedResizeObserver` to keep content wrapper height in sync when group content resizes while expanded. Handles keyboard events: Enter/Space toggle, Home/End and ArrowUp/ArrowDown move focus between group headers.

**Documentable behavior:**
- Header button has `role="button"`, `aria-expanded`, `aria-disabled`, and `aria-controls` pointing to the content wrapper `id`.
- Content wrapper has `role="region"` and `aria-labelledby` pointing to the header button `id`.
- `loadContent: "whenExpanded"` defers content rendering until first expansion — `renderContent.current` tracks whether content has ever been shown.
- Animation: collapse sets explicit height then clears it after 50ms delay (triggering CSS transition); expand sets height from `getBoundingClientRect`. `onTransitionEnd` fires `completeTransitioning` to finalize state.
- `onToggleCompletion` callback fires after the visual transition completes, not immediately on click.
- When not collapsible, `tabIndex`, `onClick`, `onKeyDown` are all removed from the header button.

**User-facing:** Yes — this is the primary interactive element.

**New learnings:** There is a defensive sync check in the animation effect: if `renderCollapsed` and prop `collapsed` drift (e.g., transition end not firing), the component self-corrects by calling `completeTransitioning`. The 50ms delay before setting height ensures the browser has rendered the DOM before measuring height.

---

## src/components/Accordion.tsx

**Purpose:** Container component managing the collapsed state array for all accordion groups, handling single vs. multiple expand modes, and dispatching keyboard focus changes.

**Logic:** Uses `useReducer` with `getCollapsedAccordionGroupsReducer` to manage the `boolean[]` collapsed state array. Initializes state from `initiallyCollapsed` values of each group, enforcing single-expanded invariant at init. Monitors `props.groups` for changes to `initiallyCollapsed` values (triggering full state reset) and for external `collapsed` attribute changes (syncing to reducer). Focus navigation queries `.widget-accordion-group-header-button` elements within the container.

**Documentable behavior:**
- In `singleExpanded` mode, when initializing, all groups before the last `initiallyCollapsed: false` group are set to collapsed — ensuring only the last configured "start expanded" group is actually expanded.
- External `collapsed` attribute changes (from Mendix) are applied per-group without resetting other groups.
- If `initiallyCollapsed` values change (e.g., navigating to different entity), the full accordion state resets to the new initial values.
- Keyboard focus wraps within the accordion container using direct DOM queries; there is no wrapping (ArrowDown on last does nothing, ArrowUp on first does nothing).

**User-facing:** Indirectly — manages state; renders `AccordionGroup` elements.

**New learnings:** The `singleExpandedGroup` prop is frozen at mount time (ref, never updated) since the XML-defined expand behavior cannot change at runtime. The component key for each group is `${group.initiallyCollapsed}_${index}`, meaning when initial collapsed state changes the group component re-mounts.

---

## src/components/Header.tsx

**Purpose:** Renders the text header of an accordion group as a semantic HTML heading element.

**Logic:** Maps `HeaderHeadingEnum` values to HTML elements `h1`–`h6`. Defaults to `h1` and switches per enum value. Wraps children in the selected heading tag.

**Documentable behavior:** The heading level is purely presentational/semantic — it doesn't affect accordion behavior. Only used when `headerRenderMode === "text"`.

**User-facing:** Yes — renders visible heading text in the accordion header.

**New learnings:** The component defaults to `h1` when no switch case matches (only `headingTwo`–`headingSix` are in the switch; `headingOne` falls through to default `h1`). Heading level affects HTML semantics/accessibility (screen reader document outline).

---

## src/components/Icon.tsx

**Purpose:** Renders the chevron/toggle icon in the accordion group header.

**Logic:** If no icon data is provided and not loading, renders a built-in inline SVG chevron. If icon data is provided, delegates to `IconInternal` from `@mendix/widget-plugin-component-kit` which handles glyph, image, and icon-class types. Applies `widget-accordion-group-header-button-icon` CSS class and optionally adds `widget-accordion-group-header-button-icon-animate` for CSS rotation animation.

**Documentable behavior:** The animate class triggers a CSS `transform: rotate(-180deg)` transition defined in `accordion-main.scss`. When `loading` is true and no icon data exists, renders nothing (prevents flash of default chevron while custom icon loads). The default chevron is a downward-pointing path that rotates 180° when expanded.

**User-facing:** Yes — visible icon in collapsed/expanded state.

**New learnings:** The SVG is `aria-hidden` — icon is purely decorative (the `aria-expanded` attribute on the header button conveys state to screen readers). The `animate` prop controls whether the icon rotates on toggle vs. swapping between two distinct icons.

---

## src/utils/reducers.ts

**Purpose:** State reducer for the accordion groups' collapsed boolean array.

**Logic:** Exports `getCollapsedAccordionGroupsReducer` which returns a reducer function configured for "single" or "multiple" expand mode. Actions: `expand` (set one index to false), `collapse` (set one index to true), `reset` (replace entire state). In "single" mode, `expand` sets all other groups to `true` (collapsed) atomically.

**Documentable behavior:** In single-expand mode, expanding group N automatically collapses all other groups in one state update (no intermediate state). "Multiple" mode expands independently. `reset` allows full replacement for initial-state changes.

**User-facing:** No — internal state management only.

**New learnings:** The reducer is created once via `useRef` and never recreated — a deliberate optimization since `singleExpandedGroup` is treated as immutable after mount. Returning a new array even for single-element changes ensures React detects the state change.

---

## src/utils/iconGenerator.tsx

**Purpose:** Hook that generates the correct icon element based on animation and state settings.

**Logic:** `useIconGenerator` returns a memoized callback `(collapsed: boolean) => ReactElement`. When `animateIcon` is true, always renders the single `icon` (CSS handles rotation). When false, renders `expandIcon` when `collapsed === true` and `collapseIcon` when `collapsed === false`.

**Documentable behavior:** This is where the two icon configuration modes are implemented: animated single icon (rotates via CSS) vs. static swap (two distinct icons). The hook is used by both the runtime (`Accordion.tsx`) and preview (`Accordion.editorPreview.tsx`).

**User-facing:** No — produces ReactElements rendered by AccordionGroup.

**New learnings:** The `animateIcon` flag in the XML maps exactly to this branch: `true` = single animated icon, `false` = two static icons. The `collapsed` parameter received at call time means this function is called per-render of each group.

---

## src/utils/resizeObserver.ts

**Purpose:** Keeps the accordion content wrapper's explicit height in sync when content dimensions change while the group is expanded (supporting CSS height transitions).

**Logic:** `useDebouncedResizeObserver` attaches a `ResizeObserver` to `contentRef`. When the observed content resizes and the group is not collapsed, it updates `contentWrapperRef.current.style.height` to match the new content height. The observer callback is debounced at 32ms via the local `debounce` utility.

**Documentable behavior:** This is needed because the CSS animation requires an explicit pixel height on the wrapper (not `auto`). Without this observer, if content inside an expanded group changes size (e.g., dynamic content loads), the wrapper height would become stale and clip or leave extra space. The 32ms debounce prevents thrashing during rapid resize events.

**User-facing:** No — internal animation support.

**New learnings:** The ResizeObserver is disconnected on component unmount via the `useEffect` cleanup. The 32ms debounce interval is below typical 60fps frame time (16ms) but above typical 30fps (33ms), balancing responsiveness with CPU usage.

---

## src/ui/accordion-main.scss

**Purpose:** Defines all visual styles for the accordion widget — layout, colors, animations, hover/focus states, and icon positioning.

**Logic:** Uses SCSS variables for colors (`$background-color`, `$border-color`, `$header-background-color-hover`, `$header-color`, `$header-color-hover`). Defines styles for collapsed state (hides content wrapper via `display: none`), animation states (`-collapsing`, `-expanding` classes use `height` transition over 0.2s), icon rotation (`rotate(-180deg)` for expanded), hover/focus styling (background change + color change to `$header-color-hover`), focus-visible ring, and icon placement (left = `flex-direction: row-reverse`).

**Documentable behavior:**
- Collapsed content is hidden via `display: none` (not just `height: 0`), so it's fully removed from layout.
- Animation uses CSS `transition: height 0.2s ease 50ms` — the 50ms delay matches the 50ms JavaScript `setTimeout` delay before setting height.
- Focus ring uses `box-shadow` (not `outline`) to support border-radius and avoid being clipped by `overflow: hidden`.
- Left icon placement uses `flex-direction: row-reverse` — the header text still appears first in DOM order (accessibility) but visually after the icon.
- Default heading font size is 18px/weight 600 regardless of `h1`–`h6` tag used.

**User-facing:** Yes — all visual presentation.

**New learnings:** The `$header-color-hover` is `#264ae5` (Mendix blue), and hover/focus applies to clickable headers only (`.widget-accordion-group-header-button-clickable`). The preview mode class (`.widget-accordion-preview`) has overrides for nested div selectors since the Studio canvas wraps content differently than runtime.

---

## packages/shared/widget-plugin-platform/src/framework/generate-uuid.ts

**Purpose:** Generates sequential numeric IDs for widget instances to ensure unique accessible element IDs within a page.

**Logic:** Stores a counter on `window['com.mendix.widgets.web.UUID']`, incrementing on each call. Returns a number starting at 1.

**Documentable behavior:** IDs are unique per page load, not globally unique. Multiple accordion instances on the same page get different numeric suffixes. The window-scoped counter is shared across all widget types using this utility.

**User-facing:** No — produces IDs used in HTML attributes (`id`, `aria-controls`, `aria-labelledby`).

**New learnings:** This is a sequential counter, not a UUID. The name is a slight misnomer. The namespaced window key prevents collisions with other JavaScript on the page.

---

## packages/shared/widget-plugin-platform/src/utils/debounce.ts

**Purpose:** Generic debounce utility with abort capability used to throttle ResizeObserver callbacks.

**Logic:** Returns a `[debouncedFn, abortFn]` tuple. Each call to `debouncedFn` cancels any pending timeout and schedules a new one after `waitFor` ms. `abortFn` cancels without calling the function.

**Documentable behavior:** The ResizeObserver callback is debounced at 32ms to prevent excessive height recalculation during rapid content changes.

**User-facing:** No — internal utility.

**New learnings:** The abort function enables clean teardown (though not currently used in the accordion's cleanup path — only the `ResizeObserver.disconnect()` is called on unmount).

---

## packages/shared/widget-plugin-component-kit/src/IconInternal.tsx

**Purpose:** Reusable icon renderer supporting three Mendix `WebIcon` formats: glyph (Bootstrap icon), image (URL), and icon class (custom font icon).

**Logic:** Switches on `icon.type` to render a `<span>` with glyph class, `<img>` with URL, or `<span>` with icon class. Returns fallback or null if no type matches.

**Documentable behavior:** All rendered icons have `aria-hidden` — they're decorative; screen readers use `aria-expanded` on the button. The `className` prop allows accordion-specific classes (`widget-accordion-group-header-button-icon` and `animate` variant) to be injected.

**User-facing:** Indirectly — renders the custom icon configured by the widget user.

**New learnings:** The glyph type uses Bootstrap's `glyphicon` class as a prefix. Image type uses `iconUrl` (not `imageUrl` — that field also exists in the type but is not used for rendering).
