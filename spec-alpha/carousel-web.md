# Carousel (carousel-web)

## Purpose

The Carousel widget displays a collection of content slides within a Mendix pluggable widget, enabling users to browse a list data source one item at a time. It is used when the page must present multiple pieces of content in a compact space with optional automated cycling. The widget is offline-capable and requires an entity context (data view or list view ancestor).

## User Scenarios

### [P1] Manual slide navigation
**Given** a Carousel placed inside a data view with a list data source containing multiple items  
**When** the user clicks the right navigation arrow  
**Then** the carousel advances to the next slide with or without a fade animation, depending on the `animation` prop  

#### Edge Cases
- When `loop=false` and the user is on the first slide, the left arrow is disabled (`.swiper-button-disabled`)
- When `loop=false` and the user is on the last slide, the right arrow is disabled
- When `loop=true`, advancing past the last slide wraps back to the first slide

### [P2] Keyboard navigation
**Given** a Carousel rendered in the browser  
**When** the user focuses the carousel and presses the left or right arrow key  
**Then** the carousel moves to the previous or next slide respectively  

#### Edge Cases
- Keyboard navigation is always active; it cannot be disabled via widget props

### [P3] Pagination dot navigation
**Given** a Carousel with `showPagination=true`  
**When** the user clicks a pagination bullet  
**Then** the carousel navigates directly to the corresponding slide  

#### Edge Cases
- Each bullet carries `aria-label="Go to slide N"` and `aria-controls` referencing the target slide's DOM ID
- Non-active slides are marked `aria-hidden="true"`

### [P4] Autoplay cycling
**Given** a Carousel with `autoplay=true` and `delay=N` ms  
**When** the carousel is rendered and no user interaction has occurred  
**Then** the carousel automatically advances to the next slide every N milliseconds  

#### Edge Cases
- When `loop=false`, autoplay stops at the last slide (`stopOnLastSlide: true`) and does not wrap
- When `autoplay=false` (the default), no automatic advancement occurs
- The `delay` property is only visible in Studio Pro when `autoplay=true`

### [P5] Click action
**Given** a Carousel with an `onClickAction` configured  
**When** the user clicks anywhere on the carousel  
**Then** the configured action executes  

#### Edge Cases
- There is a single global `onClickAction` for the entire carousel; per-slide actions are not supported
- Child widget events (e.g., Image widget lightbox) execute independently of `onClickAction`
- `onClickAction` is optional; if not configured, no action fires on click

### [P6] Loading state
**Given** a Carousel whose data source has not yet resolved  
**When** the widget is rendered  
**Then** a 16×16 px spinning SVG placeholder is shown (`aria-hidden`)  

#### Edge Cases
- The spinner is hidden from screen readers via `aria-hidden`
- Once the data source resolves, the spinner is replaced by the slides

## Functional Requirements

- FR-001: The widget MUST render each data source item as a separate slide using the configured content drop zone.
- FR-002: The widget MUST display a spinning loader while the data source status is not `Available`.
- FR-003: The widget MUST support fade animation between slides when `animation=true`; transitions MUST use cross-fade.
- FR-004: The widget MUST render slides as `<ul>`/`<li>` semantic HTML elements.
- FR-005: The widget MUST always enable keyboard navigation (left/right arrow keys); this behavior is not configurable.
- FR-006: When `showPagination=true`, the widget MUST render clickable pagination bullets with ARIA labels (`aria-label="Go to slide N"`) and `aria-controls` pointing to the corresponding slide ID.
- FR-007: Non-active slides MUST be marked `aria-hidden="true"`.
- FR-008: A `<span>` with `aria-live="assertive"` MUST always be rendered for screen-reader slide-change announcements.
- FR-009: When `autoplay=true`, the widget MUST auto-advance slides at the configured `delay` interval.
- FR-010: When `autoplay=true` and `loop=false`, autoplay MUST stop at the last slide and MUST NOT wrap.
- FR-011: Navigation arrow color MUST follow the host application's `--brand-primary` CSS variable.
- FR-012: Navigation arrows MUST display a `2px solid` focus ring for keyboard accessibility.
- FR-013: When `onClickAction` is configured, a single global action MUST execute on any carousel click; per-slide action binding is not supported.
- FR-014: The widget MUST be usable in offline/PWA Mendix applications (`offlineCapable=true`) provided the data source is available offline.
- FR-015: The widget MUST require an entity context (`needsEntityContext=true`).

## Props Reference

| Name | Type | Default | Caption | Description |
|------|------|---------|---------|-------------|
| `dataSource` | `ListValue` | — | Data source | The list entity that provides the slides. |
| `content` | `ListWidgetValue` | — | Content | Drop zone rendered inside each slide. Bound to `dataSource`. |
| `showPagination` | `boolean` | `true` | Show pagination | Displays clickable bullet dots below the slides. |
| `navigation` | `boolean` | `true` | Show navigation | Displays previous/next arrow buttons. |
| `autoplay` | `boolean` | `false` | Auto play | Automatically advances slides at the configured delay. |
| `delay` | `number` | `1000` | After (ms) | Milliseconds between auto-advances. Only effective when `autoplay=true`. Hidden in Studio Pro when `autoplay=false`. |
| `loop` | `boolean` | `true` | Loop | Wraps from the last slide back to the first. When `false`, autoplay stops at the last slide. |
| `animation` | `boolean` | `true` | Animate | Enables fade cross-transition between slides. |
| `onClickAction` | `ActionValue` | — | On click | Action executed when the user clicks anywhere on the carousel. Single global action; no per-slide binding. |

## Changelog

**2.3.2** (2026-04-13) — Dependency security updates.

**2.3.1** — Added focus indicator and background to navigation buttons for keyboard accessibility. Changed autoplay default from `true` to `false` (now opt-in).

**2.3.0** — Updated Swiper from v9 to v11.

## Open Questions

> Could not be determined from source code alone — requires human review

- [ ] Is there a documented maximum number of slides before performance degrades?
- [ ] Can `onClickAction` be scoped to individual slides in a future version, or is the single-action design intentional by product decision?
- [ ] What is the intended behavior when the `delay` is set to 0 ms?
