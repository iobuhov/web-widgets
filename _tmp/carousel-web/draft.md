# carousel-web — Source Extraction Draft

Widget: `@mendix/carousel-web` v2.3.2  
Package path: `com.mendix.widget.custom`  
Marketplace app #47784, min Mendix 9.6.0

---

## src/Carousel.tsx

**1. Purpose:** Container component — bridges Mendix props to the presentational `Carousel` component. Handles datasource loading state.

**2. Logic:** Renders a loading spinner (`widget-carousel-loading-spinner`) when `props.dataSource?.status !== ValueStatus.Available`. When data is ready, maps each datasource item to `{ id: item.id, content: props.content?.get(item) }`. Uses React's `useId()` to produce a stable unique `id` prop passed into the Swiper component. Single `onClickAction` is wrapped in `useCallback(() => executeAction(props.onClickAction))` and passed as `onClick` — all slides share the same click handler.

**3. Behavioral documentation:** The widget shows a spinner placeholder during data load, then renders the carousel with all items. There is no per-item click action; clicking anywhere on the carousel fires the single configured action.

**4. User-facing?** Yes — container component consumed by Mendix Studio Pro and the runtime.

**5. New learnings:** `onClickAction` is a single `ActionValue` (not `ListActionValue`), meaning the same action fires regardless of which slide was clicked. The loading spinner uses `aria-hidden` so screen readers skip it.

---

## src/Carousel.xml

**1. Purpose:** Widget descriptor — declares properties exposed in Mendix Studio Pro.

**2. Logic:** `needsEntityContext="true"`, `offlineCapable="true"`. Properties:
- `dataSource` — list datasource
- `content` — widgets type (one widget template per item, `ListWidgetValue`)
- `showPagination` — boolean, default `true`
- `navigation` — boolean, default `true`
- `autoplay` — boolean, default `false`
- `delay` — integer (ms), default `1000`
- `loop` — boolean, default `true`
- `animation` — boolean, default `true`
- `onClickAction` — optional action (not per-item)

**3. Behavioral documentation:** Default configuration shows a looping, animated carousel with pagination dots and prev/next navigation buttons. Autoplay is off by default. The `delay` property only matters when `autoplay=true`.

**4. User-facing?** Yes — determines the property panel in Studio Pro.

**5. New learnings:** `offlineCapable=true` means the widget works in offline/PWA Mendix apps as long as the datasource data is available offline. `needsEntityContext=true` means it must be placed inside a data view or similar context.

---

## typings/CarouselProps.d.ts

**1. Purpose:** Auto-generated TypeScript types for widget props (runtime and preview).

**2. Logic:** `content` is typed as `ListWidgetValue` — a per-item widget renderer. `onClickAction` is `ActionValue | undefined`. `dataSource` is `ListValue`. Boolean props match XML defaults. `delay` is `number`.

**3. Behavioral documentation:** Confirms `onClickAction` is a plain `ActionValue` (not `ListActionValue`), meaning no per-item action binding. `content` uses `ListWidgetValue` which exposes a `.get(item)` method to obtain the rendered widget for each list item.

**4. User-facing?** No — TypeScript types only, not rendered.

**5. New learnings:** The distinction between `ActionValue` and `ListActionValue` matters here: this widget cannot bind different actions per slide, only one global action for the entire carousel.

---

## CHANGELOG.md

**1. Purpose:** Release history for the carousel-web widget.

**2. Logic:** Notable versions:
- v2.3.2 (2026-04-13): latest
- v2.3.1: Added focus indicator for navigation buttons; removed autoplay default (now explicitly off)
- v2.3.0: Upgraded Swiper v9 → v11
- v2.2.0: DOM structure changed from `div` to `ul`/`li` for accessibility; added ARIA attributes, keyboard navigation, focus styles
- v2.0.0: Added autoplay, animation, pagination, navigation, loop, delay; converted to pluggable widget

**3. Behavioral documentation:** The widget underwent significant accessibility hardening in v2.2.0 (semantic list markup, ARIA). Swiper library was major-version upgraded in v2.3.0. Autoplay was previously on by default and was changed to off in v2.3.1.

**4. User-facing?** No — developer/maintainer documentation.

**5. New learnings:** The `autoplay` default changed between releases — important for upgrade notes. `package.json` shows `swiper: ^12.1.2` which is a newer major version than the CHANGELOG's "v11" upgrade note, indicating another upgrade happened since v2.3.0 without a changelog entry.

---

## src/components/Carousel.tsx

**1. Purpose:** Presentational component — wraps the Swiper library to render the actual carousel UI.

**2. Logic:**
- Swiper modules used: `A11y`, `Navigation`, `Pagination`, `EffectFade`, `Autoplay`, `Keyboard`
- `slidesPerView: 1`, `centeredSlides: true` always
- `animation=true` enables `effect: "fade"` with `crossFade: true`
- `autoplay` configures `{ delay, stopOnLastSlide: true }` — autoplay halts at the last slide rather than looping
- `loop` is passed directly to Swiper
- Pagination bullets are rendered with `role="button"`, `aria-controls` pointing to the slide ID, and `aria-label="Go to slide N"`
- `wrapperTag="ul"` makes Swiper render slides in a `<ul>`; each `SwiperSlide` uses `tag="li"` — semantic list structure
- `aria-hidden={index !== activeIndex}` is set per-slide to hide non-active slides from screen readers
- `keyboard: { enabled: true }` — arrow key navigation
- `a11y: { enabled: true, slideRole: "listitem" }` — Swiper's built-in accessibility
- Tracks `activeIndex` via `onActiveIndexChange` / `onSwiper` callbacks using `swiper.realIndex`
- `onClick` on Swiper propagates the single shared action handler

**3. Behavioral documentation:** Carousel renders one slide at a time centered. Fade animation crossfades between slides. Autoplay stops on the last slide. Non-active slides are hidden from assistive technology. Pagination dots have keyboard-accessible role=button with aria-controls referencing slide IDs. Prev/Next buttons are Swiper-native with keyboard focus support.

**4. User-facing?** Yes — this is the rendered UI component.

**5. New learnings:** `stopOnLastSlide: true` means autoplay + `loop: false` creates a self-terminating slideshow. Slide IDs are deterministic: `carousel-slide-{widgetId}-{itemGUID}`. The `a11y` module and explicit `aria-hidden` are combined — double accessibility layering, ensuring both Swiper's built-in a11y and custom attribute management work in concert.

---

## src/Carousel.editorConfig.ts

**1. Purpose:** Studio Pro editor configuration — controls property visibility and Studio Pro UI layout.

**2. Logic:**
- `getProperties()`: hides the `delay` property when `autoplay=false` (no delay needed if autoplay is off)
- `getPreview()`: builds a `StructurePreviewProps` tree showing a labeled header ("Carousel"), a `DropZone` for the `content` property, and optionally pagination dots (3 dots: 1 blue + 4 grey) if `showPagination=true`. Dots are rendered as inline SVG images (`dot_blue.svg`, `dot_grey.svg`, `dot_empty.svg`).

**3. Behavioral documentation:** In Studio Pro, the carousel renders a preview with a "Carousel" label, a content drop zone, and conditional pagination indicator. The `delay` property is hidden from the property panel when autoplay is off.

**4. User-facing?** No — Studio Pro editor only.

**5. New learnings:** The editor preview uses SVG dot assets to approximate the pagination indicator visually. The `transformGroupsIntoTabs` call only applies on `platform === "web"`, meaning the web editor groups properties into tabs.

---

## src/Carousel.editorPreview.tsx

**1. Purpose:** Renders a live React preview of the carousel inside Studio Pro's canvas.

**2. Logic:** Renders the `Carousel` component directly with 3 hardcoded placeholder items (`"1"`, `"2"`, `"3"` as GUIDs). `loop=false` is hardcoded to avoid infinite looping during static preview. Uses `props.content.renderer` with a caption (`"Carousel item content {N}"`) for each placeholder. Passes `navigation` and `showPagination` from actual props.

**3. Behavioral documentation:** Studio Pro canvas shows a working Swiper carousel with 3 placeholder slides. Navigation arrows and pagination dots reflect the actual configured values. Loop is disabled in preview.

**4. User-facing?** No — Studio Pro canvas preview only.

**5. New learnings:** `loop=false` in preview prevents Swiper from duplicating slides (which it does internally for looping), keeping the preview stable. `generateUUID()` from `@mendix/widget-plugin-platform` provides a stable ID for the preview instance.

---

## src/components/__tests__/Carousel.spec.tsx

**1. Purpose:** Unit tests for the presentational `Carousel` component.

**2. Logic:** Mocks Swiper CSS imports and all Swiper modules (`A11y`, `Navigation`, `Pagination`, etc.). Provides a detailed mock of `Swiper` and `SwiperSlide` that replicates the expected DOM structure including navigation buttons, pagination bullets, and accessibility attributes. 4 snapshot tests: full config, no pagination, no navigation, minimal (no pagination + no navigation). Uses `Math.random` mock for deterministic snapshots.

**3. Behavioral documentation:** Tests verify the rendered HTML snapshot for each display variant. No behavior/interaction tests — only structural snapshots. The mock swiper simulates `onSwiper` and `onActiveIndexChange` callbacks with `realIndex: 0` to initialize state.

**4. User-facing?** No — test file.

**5. New learnings:** The detailed Swiper mock replicates real Swiper DOM output (classes like `swiper-button-prev`, `swiper-pagination-bullet`, `aria-live="assertive"` notification span). This makes snapshots representative of actual rendered output. `Keyboard` module is conspicuously absent from the mock imports — it was missing from the mock's `swiper/modules` mock list (the real component uses it but the test mock doesn't include it).

---

## e2e/Carousel.spec.js

**1. Purpose:** Playwright end-to-end tests for the carousel widget in a real Mendix test application.

**2. Logic:** 4 active tests + 1 skipped:
1. Left arrow is disabled (class `swiper-button-disabled`) on first item
2. Left arrow becomes visible/enabled after clicking the right arrow
3. Right arrow is disabled on the last image item
4. Clicking an image opens a lightbox (`mx-image-viewer-lightbox`)
5. SKIPPED: accessibility violations check (using `carousel.accessibility.snapshot()` + `toHaveNoViolations`)

Session cleanup: `window.mx.session.logout()` after each test to avoid Mendix license limit (5 sessions).

**3. Behavioral documentation:** The e2e suite tests navigation state (disabled arrows at boundaries), forward navigation, and image lightbox integration. The accessibility test exists as a skeleton but is skipped — likely due to API unavailability in the test framework at the time.

**4. User-facing?** No — test file.

**5. New learnings:** The `loop` is assumed to be `false` in the test app (otherwise the left arrow would never be disabled on first item). The e2e tests target a specific test project (branch `carousel-web` of `mendix/testProjects`). Image lightbox behavior is tested through an `mx-name-Image01` image widget placed inside carousel content — confirming the widget's content slot accepts image widgets that retain their own click behavior. The skipped accessibility test references `carousel.accessibility` which is a non-standard Playwright API — likely a custom accessor that wasn't implemented.

---

## Summary

**Widget:** carousel-web v2.3.2 (`@mendix/carousel-web`)

**Core functionality:** A Mendix pluggable widget that renders a content carousel using the Swiper library (currently ^12.1.2). Accepts a list datasource and a per-item widget template (`ListWidgetValue`), rendering one slide at a time with optional fade animation, autoplay, loop, navigation arrows, and pagination dots.

**Architecture:**
- `src/Carousel.tsx` — Mendix container; handles datasource loading state, maps items, single shared click action
- `src/components/Carousel.tsx` — Pure presentational Swiper wrapper; all Swiper config lives here

**Key behaviors:**
- `needsEntityContext=true`, `offlineCapable=true`
- Single `onClickAction` (not per-item) fires on any slide click
- `animation=true` → Swiper fade effect with crossFade
- `autoplay=true` + `loop=false` → autoplay stops at last slide (`stopOnLastSlide: true`)
- Pagination bullets: `role="button"`, `aria-controls` → slide ID, clickable
- Slides rendered as `<ul>/<li>` semantic list; non-active slides get `aria-hidden="true"`
- Keyboard navigation enabled via Swiper Keyboard module
- Loading spinner shown while datasource not yet Available

**Accessibility:** Strong — ARIA roles (listitem), keyboard navigation, aria-hidden on inactive slides, labeled navigation buttons, aria-controls on pagination bullets. Introduced in v2.2.0.

**Swiper version discrepancy:** CHANGELOG says "upgrade Swiper v9→v11" in v2.3.0, but `package.json` shows `swiper: ^12.1.2` — another major upgrade occurred without a changelog entry.

**Tests:**
- Unit: 4 snapshot tests (with/without pagination, with/without navigation)
- E2E: 4 Playwright tests (arrow disabled states at boundaries, lightbox on image click); 1 skipped accessibility test

**Editor support:** Studio Pro property panel hides `delay` when `autoplay=false`. Canvas preview renders 3 placeholder slides with `loop=false`. Properties shown as tabs in web platform.
