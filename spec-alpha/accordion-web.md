# Accordion

## Purpose

The Accordion widget renders a collapsible group container for Mendix pages. Each group has a configurable header and a content area that expands or collapses when activated. It is used to organize page content into sections, reduce visual clutter, and control information density. The widget supports both independently collapsible groups and a single-expanded mode where opening one group automatically closes the others. Two-way attribute binding allows accordion state to be persisted to and driven from a Mendix entity attribute.

## User Scenarios

### [P1] Collapse and expand a group
**Given** the Accordion widget is placed on a page with one or more groups configured  
**When** the user clicks or activates a group header  
**Then** the group content area expands or collapses with a CSS height animation (0.2s ease, 50ms delay), and the icon rotates 180 degrees to indicate the new state

#### Edge Cases
- When `collapsible` is `false`, groups cannot be collapsed; all content is always visible and all collapse/expand controls are disabled.
- When `animate` is `false` (or `advancedMode` is not enabled), state transitions are immediate with no animation.
- When `loadContent` is `"whenExpanded"`, content widgets are not rendered or their data fetched until the group is first expanded; once expanded, they remain in the DOM thereafter.

### [P2] Single-expand mode (accordion behavior)
**Given** `expandBehavior` is set to `"singleExpanded"` and `collapsible` is `true`  
**When** the user expands a group  
**Then** all other groups collapse automatically; only one group can be expanded at a time

#### Edge Cases
- At initialization in single-expand mode, if multiple groups have `initialCollapsedState` set to `"expanded"`, only the last one (by order) remains expanded; all preceding ones are forced collapsed.
- Studio Pro validation prevents saving a configuration where more than one group has `initialCollapsedState = "expanded"` while `expandBehavior = "singleExpanded"` — this is surfaced as an error, not a warning.

### [P3] Two-way state binding via Mendix attribute
**Given** a group's `collapsed` property is bound to a Boolean entity attribute  
**When** the user expands or collapses the group  
**Then** the widget calls `setValue(true|false)` on the attribute after the visual transition completes, persisting the state to the entity

#### Edge Cases
- The widget renders nothing (`null`) until all bound `collapsed` and `initiallyCollapsed` attribute values have resolved from `Loading` state.
- External changes to the bound attribute (e.g., from a microflow) are reflected in the accordion state via internal sync on the next render.

### [P4] Dynamic initial collapse state
**Given** a group's `initialCollapsedState` is `"dynamic"` and bound to a boolean expression  
**When** the widget mounts  
**Then** the group starts in the collapsed/expanded state determined by the expression value

#### Edge Cases
- The widget renders nothing (`null`) if any group with `initialCollapsedState = "dynamic"` has its expression value in `Loading` state.
- Changing `initiallyCollapsed` after mount triggers a full state reset (via `useMemo` detection), which remounts affected group components.

### [P5] Keyboard navigation
**Given** the user moves focus to an accordion group header  
**When** the user presses keyboard keys  
**Then** the following navigation applies: Enter/Space → toggle group; Home → first header; End → last header; ArrowUp → previous header; ArrowDown → next header

#### Edge Cases
- Focus management targets header `<button>` elements by querying within the accordion container.

## Functional Requirements

- FR-001: The widget MUST render a `<section>` per group with an `<header>` containing a `<button>`, and a content region with `role="region"` and `aria-labelledby` pointing to the header button.
- FR-002: Header buttons MUST carry `aria-expanded`, `aria-disabled`, and `aria-controls` attributes reflecting current state.
- FR-003: Collapsed group content MUST be hidden via `display: none` (not opacity or visibility).
- FR-004: The widget MUST support three `initialCollapsedState` values: `"expanded"`, `"collapsed"` (default), and `"dynamic"` (expression-driven).
- FR-005: In `singleExpanded` mode, expanding any group MUST immediately collapse all other groups.
- FR-006: When `loadContent` is `"whenExpanded"`, the content widget tree MUST NOT render until the group is first expanded. Once rendered, it MUST remain in the DOM for the widget lifetime.
- FR-007: When `collapsed` is bound to an `EditableValue<boolean>`, the widget MUST call `setValue` after the transition completes (not on click).
- FR-008: The widget MUST support offline Mendix applications (`offlineCapable="true"`).
- FR-009: The resize observer MUST keep the animated content wrapper height in sync with its contents (32ms debounce) while the group is expanded.
- FR-010: The widget MUST use a globally-unique, page-stable ID (derived from a page-global counter) for ARIA linkage to prevent duplicate `id` attributes across multiple widget instances.

## Props Reference

| Name | Type | Default | Caption | Description |
|------|------|---------|---------|-------------|
| `groups` | List | — | Groups | List of accordion groups. Each group configures a collapsible section. |
| `groups.headerText` | String expression | — | Header text | Text displayed in the group header (when `headerRenderMode` is `"text"`). |
| `groups.headerContent` | Widgets | — | Header content | Custom widget content for the header (when `headerRenderMode` is `"custom"`). |
| `groups.headerRenderMode` | Enum (`text` \| `custom`) | `"text"` | Header render mode | Selects between a text header or a custom-content header. |
| `groups.headerHeading` | Enum (`headingOne`…`headingSix`) | `"headingThree"` | Heading | Semantic heading level for the text header. Visual size is CSS-controlled. |
| `groups.content` | Widgets | — | Content | Drop zone for group body content. |
| `groups.visible` | Boolean expression | `true` | Visible | Controls group visibility. |
| `groups.dynamicClass` | String expression | — | Dynamic class | CSS class appended to the group element at runtime. |
| `groups.loadContent` | Enum (`always` \| `whenExpanded`) | `"always"` | Load content | `"whenExpanded"` defers rendering and data fetch until first expansion. |
| `groups.initialCollapsedState` | Enum (`expanded` \| `collapsed` \| `dynamic`) | `"collapsed"` | Initial state | Starting collapsed state. `"dynamic"` reads from `initiallyCollapsed`. |
| `groups.initiallyCollapsed` | Boolean expression | — | Initially collapsed | Expression providing initial state when `initialCollapsedState` is `"dynamic"`. |
| `groups.collapsed` | EditableValue (Boolean) | — | Collapsed attribute | Two-way binding to a Boolean entity attribute for state persistence. Optional. |
| `groups.onToggleCollapsed` | Action | — | On toggle | Action executed when the group is toggled (linked to `collapsed.setValue`). |
| `collapsible` | Boolean | `true` | Collapsible | When `false`, all groups are always expanded and cannot be collapsed. |
| `expandBehavior` | Enum (`singleExpanded` \| `allExpanded`) | `"allExpanded"` | Expand behavior | `"singleExpanded"`: opening one group closes all others. Only applies when `collapsible` is `true`. |
| `animate` | Boolean | `true` | Animate | Enables height animation on expand/collapse. Hidden unless `advancedMode` is on. |
| `showIcon` | Enum (`left` \| `right` \| `no`) | `"right"` | Show icon | Position of the expand/collapse icon in the header button, or `"no"` to hide it. |
| `icon` | WebIcon | — | Icon | Custom animated icon (used when `animateIcon` is `true`). |
| `expandIcon` | WebIcon | — | Expand icon | Icon shown when group is collapsed (used when `animateIcon` is `false`). |
| `collapseIcon` | WebIcon | — | Collapse icon | Icon shown when group is expanded (used when `animateIcon` is `false`). |
| `animateIcon` | Boolean | `true` | Animate icon | When `true`, uses a single rotated `icon`; when `false`, swaps `expandIcon`/`collapseIcon`. |
| `advancedMode` | Boolean | `false` | Advanced mode | Unlocks advanced properties (`animate`, `showIcon`, icon settings) for Studio Pro users. |

## Changelog

- **v2.3.5 (2026-02-09):** Added license file and open-source dependency readme. No behavioral changes.
- **v2.3.4 (2025-02-13):** Fixed initial collapsed state not always updating the accordion.
- **v2.3.2 (2024-07-19):** Fixed nested accordion display de-sync between collapsed/uncollapsed content and state.

## Open Questions

> Could not be determined from source code alone — requires human review

- [ ] Is there a documented maximum number of groups? No limit is declared in the XML or source.
- [ ] When `loadContent = "whenExpanded"` and a group is later hidden via `visible = false`, is the rendered content preserved in the DOM or unmounted? The draft confirms lazy-load stays in DOM after first expansion, but interaction with `visible` is unconfirmed.
- [ ] The `advancedMode` property is hidden on the desktop platform — what constitutes "desktop platform" in this context? Presumably Studio (web) vs. Studio Pro (desktop), but this is not confirmed by the source.
