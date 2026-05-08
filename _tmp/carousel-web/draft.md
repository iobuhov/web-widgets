# Draft: carousel-web

Extracted from `packages/pluggableWidgets/carousel-web/` on 2026-05-08.

---

## src/Carousel.xml

**1. Purpose:** Declares the widget metadata and all configurable properties for the Mendix platform. This is the authoritative source for which props exist and their default values.

**2. Logic described:** Defines two property groups — "Data Source" (datasource list + content drop zone) and "Display" (showPagination, navigation, autoplay, delay, loop, animation) — plus an "Events" group (onClickAction). All properties and captions are declared here.

**3. Documentable behavior:** The widget requires an entity context (`needsEntityContext="true"`), supports offline (`offlineCapable="true"`), and is a pluggable widget. Default values: `showPagination=true`, `navigation=true`, `autoplay=false`, `delay=1000ms`, `loop=true`, `animation=true`. The `delay` property controls the auto-cycle interval in milliseconds. The `content` drop zone is bound to `dataSource`, making it a list-rendered slot widget.

**4. User-facing:** Yes. All properties here appear in the Studio Pro properties panel.

**5. New learnings:** `autoplay` defaults to `false` — despite `animation` and `loop` defaulting to `true`, auto-play is explicitly off by default. The `delay` default is 1000ms (1 second). The widget is categorized under "Display" in both studioProCategory and studioCategory. `offlineCapable=true` means the widget works in offline/PWA Mendix apps as long as the datasource data is available offline. `needsEntityContext=true` means it must be placed inside a data view or similar context.

---

## typings/CarouselProps.d.ts

**1. Purpose:** Auto-generated TypeScript type definitions from Carousel.xml. Provides compile-time type safety for both the container component (runtime) and the preview component (Studio Pro editor).

**2. Logic described:** `CarouselContainerProps` maps each XML property to its TypeScript type: `dataSource?: ListValue`, `content?: ListWidgetValue`, `showPagination: boolean`, `navigation: boolean`, `autoplay: boolean`, `delay: number`, `loop: boolean`, `animation: boolean`, `onClickAction?: ActionValue`. Preview props have `delay: number | null` (can be null in design mode) and separate `className`/`class` fields.

**3. Documentable behavior:** `dataSource` and `content` are optional (the widget can render with no data). `onClickAction` is optional — click handling is not required. `delay` is always a number at runtime but can be null in editor preview mode. `onClickAction` is a plain `ActionValue` (not `ListActionValue`), meaning the same action fires regardless of which slide is clicked — no per-item action binding is possible. The `@deprecated className` on PreviewProps was deprecated in v9.18.0 in favor of `class`.

**4. User-facing:** No. Internal TypeScript contract only.

**5. New learnings:** The distinction between `CarouselContainerProps` (runtime) and `CarouselPreviewProps` (editor) surfaces a behavioral nuance: `delay` is nullable in preview mode. The `ActionValue` vs `ListActionValue` distinction confirms there is only one global click action for the entire carousel, not per-slide actions.

---

## src/Carousel.tsx

**1. Purpose:** The main container/entry-point component that bridges Mendix data layer (ListValue, ActionValue) to the pure UI Carousel component.

**2. Logic described:** Checks `dataSource.status === ValueStatus.Available`; if not ready, renders a loading spinner (`loading-circle.svg`). When available, maps `dataSource.items` to `{id, content}` pairs using `content.get(item)` and passes them to `CarouselComponent`. Calls `executeAction(props.onClickAction)` via `useCallback` for click events. Uses `useId()` to generate a stable unique ID for ARIA references.

**3. Documentable behavior:** While data source is loading, a spinning SVG is shown (16×16px, `aria-hidden`). The widget gracefully handles no items (empty array). All slides share the same click handler — clicking anywhere on the carousel fires the single configured `onClickAction`. `executeAction` from `@mendix/widget-plugin-platform` is used rather than calling `onClickAction.execute()` directly.

**4. User-facing:** No. Internal orchestration layer. The user sees only the rendered Carousel.

**5. New learnings:** The `useId()` hook ensures the generated ID is stable across renders and unique per component instance, critical for ARIA `aria-controls` references in pagination bullets. The loading spinner uses `aria-hidden` so screen readers skip it entirely.

---

## src/components/Carousel.tsx

**1. Purpose:** The pure React UI component implementing the carousel, built on the Swiper library (v11). Contains all visual and interaction logic.

**2. Logic described:** Uses Swiper with modules: A11y, Navigation, Pagination, EffectFade, Autoplay, Keyboard. Configures: `slidesPerView: 1`, `centeredSlides: true`. Animation is implemented via `effect: "fade"` with `crossFade: true` when `animation=true`. Autoplay uses `{ delay, stopOnLastSlide: true }` — stops when the last slide is reached in non-loop mode. Pagination renders clickable bullet dots with ARIA roles (`role="button"`, `aria-controls`, `aria-label`). Keyboard navigation is always enabled. Tracks `activeIndex` via `useState` using `swiper.realIndex`, hiding non-active slides with `aria-hidden`.

**3. Documentable behavior:** Slides are rendered as `<ul>` / `<li>` elements (`wrapperTag="ul"`, `SwiperSlide tag="li"`). Navigation arrows use `.swiper-button-prev` / `.swiper-button-next` CSS classes. When `loop=false` and autoplay is on, playback stops at the last slide (`stopOnLastSlide: true`). Non-active slides are marked `aria-hidden`. Keyboard navigation is unconditionally enabled — it cannot be disabled via widget props. Slide IDs follow the pattern `carousel-slide-{id}-{itemId}`.

**4. User-facing:** Yes. This is the rendered component users interact with in the browser.

**5. New learnings:** The `autoplay.stopOnLastSlide: true` behavior is significant: in non-loop mode, auto-advance stops at the last slide rather than wrapping. A `swiper-notification` span with `aria-live="assertive"` is always rendered for screen-reader announcements. The ARIA pagination bullets reference slide IDs via `aria-controls`, providing a proper accessible navigation structure.

---

## src/Carousel.editorConfig.ts

**1. Purpose:** Configures how the widget appears and behaves in the Studio Pro editor: which properties are shown/hidden and how they're presented.

**2. Logic described:** `getProperties` hides the `delay` property when `autoplay=false` (conditional property visibility). On web platform, groups are transformed into tabs via `transformGroupsIntoTabs`. `getPreview` returns a structure preview showing a "Carousel" header, a drop zone for content, and optional pagination dots — using blue/grey/empty dot SVGs.

**3. Documentable behavior:** The `delay` property is only visible in Studio Pro when autoplay is enabled — a hard UI constraint. On web, property groups become tabs. The editor preview shows 5 pagination bullets (1 active blue + 2 grey + 2 empty) as a static approximation regardless of actual item count.

**4. User-facing:** No. Studio Pro editor only. Not part of runtime behavior.

**5. New learnings:** The conditional `delay` visibility directly enforces the semantics: delay is meaningless without autoplay. The preview structure uses actual SVG assets embedded as data URIs for the pagination dot visuals.

---

## src/Carousel.editorPreview.tsx

**1. Purpose:** Renders a live preview of the Carousel inside the Studio Pro canvas using the actual CarouselComponent with mock data.

**2. Logic described:** Creates 3 mock items (`"1"`, `"2"`, `"3"`) each rendering via `props.content.renderer` with caption `"Carousel item content {N}"`. Sets `loop=false` in preview mode. Uses `generateUUID()` for a stable ID.

**3. Documentable behavior:** The editor preview always shows 3 placeholder slides. Loop is hardcoded to `false` in preview — prevents Swiper from cloning slides for the infinite loop effect, which would produce confusing duplicate placeholders. The content drop zone renders inside each slide using the platform's `renderer` API.

**4. User-facing:** Studio Pro canvas only. Gives developers a visual approximation of the widget.

**5. New learnings:** `loop=false` being hardcoded in the preview is a deliberate choice to avoid Swiper's DOM duplication behavior in a static preview context.

---

## src/ui/Carousel.scss

**1. Purpose:** Defines all visual styles for the carousel widget, including Swiper CSS variable overrides and custom widget-specific styles.

**2. Logic described:** Sets Swiper CSS variables: `--swiper-navigation-size: 30px`, `--swiper-navigation-color: var(--brand-primary)`. Navigation buttons get `border-radius: 50%` (circular), `var(--gray-200)` background, padding, and a focus outline (`2px solid #264ae5`). Slides use flex centering. Images use `object-fit: cover`. Loading spinner is 16×16px with CSS `spin` keyframe animation (2s linear infinite).

**3. Documentable behavior:** Navigation button colors use Atlas UI CSS variables (`--brand-primary`, `--gray-200`, `--spacing-small`, `--focus-outline`), falling back to hardcoded values. Focus ring: `2px solid #264ae5` at `2px` offset — added in v2.3.1 for keyboard accessibility. Navigation arrow color matches the host application's brand color via `--brand-primary`.

**4. User-facing:** Yes. Directly controls the visual appearance.

**5. New learnings:** The focus indicator on navigation buttons was explicitly added in v2.3.1. The `--brand-primary` CSS variable integration means carousel arrows automatically adopt the host app's theme color without any configuration.

---

## src/components/__tests__/Carousel.spec.tsx

**1. Purpose:** Unit tests (Jest/React Testing Library) for the `Carousel` component verifying snapshot rendering under four configurations.

**2. Logic described:** Mocks Swiper modules and `swiper/react` to avoid DOM/CSS issues in test environments. Tests: (1) full render with all features, (2) without pagination, (3) without navigation, (4) minimal setup. Default test props: `pagination=true`, `animation=true`, `autoplay=true`, `delay=3000`, `loop=true`, `navigation=true`.

**3. Documentable behavior:** Confirms ARIA structure: pagination bullets with `aria-label="Go to slide N"` and `aria-controls` referencing slide IDs; navigation with `aria-label="Previous slide"` / `"Next slide"` and `role="button"`; non-active slides with `aria-hidden="true"`. A `swiper-notification` span with `aria-live="assertive"` is always rendered. The `wrapperTag="ul"` / `SwiperSlide tag="li"` combination produces a proper semantic `<ul>` list.

**4. User-facing:** No. Test infrastructure only.

**5. New learnings:** The mock Swiper reveals the complete expected DOM structure that the real Swiper library produces, making it a useful reference for understanding the ARIA accessibility tree.

---

## e2e/Carousel.spec.js

**1. Purpose:** End-to-end Playwright tests running against a live Mendix application verifying navigation arrow behavior and image interaction.

**2. Logic described:** Tests: (1) left arrow is disabled on first item (`.swiper-button-disabled` class), (2) left arrow becomes visible after navigating right, (3) right arrow is disabled on last item, (4) clicking an image opens a lightbox (`.mx-image-viewer-lightbox`). The accessibility violation test is skipped.

**3. Documentable behavior:** When `loop=false` (the e2e test app configuration), the first item disables the left arrow and the last item disables the right arrow. Images inside carousel slides can trigger the Mendix image viewer lightbox independently of the carousel's `onClickAction`. There is no `test.skip(MODERN_CLIENT)` guard — the carousel widget does NOT have a Mendix modern React client incompatibility.

**4. User-facing:** No. Test infrastructure. However, the behaviors tested are directly user-facing: arrow disable states and lightbox interaction.

**5. New learnings:** The e2e tests confirm `loop=false` is the reference test app configuration. The lightbox behavior confirms that child widget events (Image click → lightbox) are independent of the carousel's own click action. The carousel is compatible with the Mendix modern React client (no client guard).

---

## CHANGELOG.md

**1. Purpose:** Documents all notable changes per semantic version since the widget was first published.

**2. Logic described:** v2.0.0: pluggable widget rewrite with Swiper, adding autoplay/animation/pagination/navigation/loop/delay. v2.2.0: updated Swiper to v9.4, changed DOM from `<div>` to `<ul>/<li>` for accessibility. v2.3.0: updated Swiper v9 → v11. v2.3.1: added focus indicator + background to navigation buttons, removed autoplay default. v2.3.2: dependency security updates.

**3. Documentable behavior:** The `<ul>/<li>` DOM structure was introduced in v2.2.0 for semantic accessibility. Autoplay was explicitly changed to off-by-default in v2.3.1 (opt-in). Focus ring on navigation buttons added in v2.3.1. Latest version is 2.3.2 (2026-04-13).

**4. User-facing:** No. Developer/maintainer documentation.

**5. New learnings:** Autoplay being made opt-in in v2.3.1 reflects a UX decision — autoplay can be disruptive and is better as a conscious choice. The Swiper v9→v11 upgrade (v2.3.0) was the most significant recent behavioral change.
