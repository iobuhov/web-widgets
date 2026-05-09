# skiplink-web — Draft Spec

Widget: `skiplink-web`
Package: `packages/pluggableWidgets/skiplink-web/`
Agent: worker
Date: 2026-05-09

---

## src/SkipLink.tsx

**1. What is the purpose of this file?**
The root Mendix pluggable widget entry component. It renders an accessibility "skip link" — a visually hidden anchor that keyboard users can activate to jump past navigation to the main content.

**2. What kind of logic is described in this file?**
- **DOM insertion via portal**: On mount, a new `<div>` is created and inserted as the first child of `document.getElementById("root")` using `root.insertBefore(link, root.firstElementChild)`. The skip link `<a>` is portaled into this `<div>`.
- **Click handler**: `handleClick` prevents default anchor navigation. Then:
  - If `mainContentId !== ""`: uses `document.getElementById(mainContentId)` to locate the target.
  - If empty: uses `document.getElementsByTagName("main")[0]`.
  - If the target lacks a `tabindex`, temporarily adds `tabindex="-1"` to make it programmatically focusable.
  - Calls `main.focus()`.
  - Removes the temporary `tabindex` on first `blur` event (`{ once: true }`).
- `linkRoot` is created once via `useState` lazy initializer — stable across renders.

**3. What part of behavior can be documented from this file?**
- The skip link is always inserted as the first child of `#root`, making it the first focusable element on the page via Tab key.
- If `#root` is not found, a console error is logged and the `<div>` is not inserted.
- If `mainContentId` element is not found, a console error is logged and focus is not moved.
- If no `mainContentId` and no `<main>` element found, console error is logged.
- The `tabindex="-1"` cleanup prevents the target from lingering in the tab order after focus has moved away.
- The `<a>` has `href={#mainContentId}` for progressive enhancement — browsers without JS can still navigate via the href.

**4. Is it user-facing?**
Yes — the skip link is a user-facing accessibility feature, visible when focused via keyboard.

**5. What new did you learn from this file?**
The skip link is portaled into a dedicated `<div>` inserted as the first DOM child of `#root` — not React's render root. This is a deliberate architectural choice to ensure the link is literally the first interactive element in the DOM, which is a WCAG requirement for skip links. React's portal mechanism is used to achieve this while keeping the widget's React lifecycle intact.

---

## src/SkipLink.xml

**1. What is the purpose of this file?**
Mendix widget descriptor defining widget identity, category ("Accessibility"), and properties.

**2. What kind of logic is described in this file?**
Two properties in the "General" group:
- `linkText` (string, required, default "Skip to main content") — text displayed in the skip link.
- `mainContentId` (string, optional, no default) — ID of the main content element to jump to; empty means auto-detect `<main>` tag.

Widget flags: `offlineCapable="true"`, `supportedPlatform="Web"` (web-only), `needsEntityContext` not set (not required).

**3. What part of behavior can be documented from this file?**
- `needsEntityContext` is absent (defaults to false) — the widget can be placed anywhere on the page without an entity context.
- No event properties — no `onClick` action, no `onChange`, no Mendix action hooks.
- `studioProCategory="Accessibility"` and `studioCategory="Accessibility"` — dedicated accessibility toolbox category.
- `linkText` has a default value ("Skip to main content") — the widget works out of the box with no configuration.
- `helpUrl` is empty — no documentation URL provided.

**4. Is it user-facing?**
No — Studio Pro configuration descriptor.

**5. What new did you learn from this file?**
The empty `helpUrl` tag and "Unreleased" CHANGELOG status indicate this widget is newly created and may not yet be published to the Mendix Marketplace. It is the only widget in this set categorized under "Accessibility".

---

## typings/SkipLinkProps.d.ts

**1. What is the purpose of this file?**
Auto-generated TypeScript types for the widget's container and preview props.

**2. What kind of logic is described in this file?**
- `SkipLinkContainerProps` (runtime): `linkText: string`, `mainContentId: string` (always a string — empty string when not configured, not undefined/null).
- `SkipLinkPreviewProps` (Studio Pro preview): same fields plus standard preview fields.

**3. What part of behavior can be documented from this file?**
- `mainContentId` is typed as `string` (not optional) at runtime — the XML `required="false"` with no default results in an empty string when not set (not `undefined`). The component guards with `props.mainContentId !== ""`.

**4. Is it user-facing?**
No — TypeScript types only.

**5. What new did you learn from this file?**
The `tabIndex?: number` in container props (standard system property) shows the widget supports Mendix tab order configuration, which maps to the `<a>` element's `tabIndex`. Since skip links should be first in tab order, this is typically left at the default (0).

---

## src/SkipLink.editorConfig.ts

**1. What is the purpose of this file?**
Provides Studio Pro property validation (`check`) and structure preview (`getPreview`). No conditional property hiding is needed.

**2. What kind of logic is described in this file?**
- `getProperties`: no-op (returns `defaultValues` unchanged) — no conditional visibility rules needed for this simple widget.
- `check`: validates that `linkText` is not empty; emits a "Link text is required" error if it is.
- `getPreview`: renders a two-section structure preview:
  - Header row: "SkipLink" label in `topbarStandard` background with `text.secondary` color.
  - Content row: link text (or "Skip to main content" as fallback) in `text.primary`, bold, fontSize 14.
  - Both rows have borders. Supports dark mode via `structurePreviewPalette`.

**3. What part of behavior can be documented from this file?**
- Studio Pro will show a validation error if `linkText` is cleared (since it has a default, this requires active deletion).
- The structure preview shows the actual configured link text.
- Dark mode support in structure preview via the palette API.

**4. Is it user-facing?**
No — Studio Pro only.

**5. What new did you learn from this file?**
The validation check (`check`) for a property that has a default value (`"Skip to main content"`) suggests the developer anticipated that users might clear the default. The check ensures the field is never accidentally left blank.

---

## src/SkipLink.editorPreview.tsx

**1. What is the purpose of this file?**
Renders a static anchor element in Studio Pro's live design mode preview.

**2. What kind of logic is described in this file?**
- `renderMode === "xray"`: wraps in a `<div>` with `position: relative; height: 40` so the absolutely-positioned link is contained within the preview area.
- Other render modes: renders the `<a>` directly using `widget-skip-link-preview` class (which lacks the `transform: translateY(-120%)` that hides the link in production — so it's always visible in preview).
- `getPreviewCss()` returns the SCSS file content for injection.

**3. What part of behavior can be documented from this file?**
- The preview always shows the skip link in its visible state (no CSS transform to hide it).
- In xray mode, the preview is constrained to 40px height.
- The `widget-skip-link-preview` class is defined in the SCSS with the same visual style as `widget-skip-link` but without the `transform: translateY(-120%)` hide rule.

**4. Is it user-facing?**
No — Studio Pro design mode preview only.

**5. What new did you learn from this file?**
The separate `widget-skip-link-preview` CSS class (vs `widget-skip-link`) is specifically to keep the link visible in Studio Pro while hiding it by default in production. This avoids showing the "off-screen" state in the design canvas.

---

## src/ui/SkipLink.scss

**1. What is the purpose of this file?**
Stylesheet for the skip link widget — defines positioning, visual appearance, and the focus-reveal animation.

**2. What kind of logic is described in this file?**
- `.widget-skip-link`: `position: absolute; top: 0; left: 0; z-index: 1000`. By default: `transform: translateY(-120%)` moves it above the viewport; `transition: transform 0.2s` animates reveal. Visual style: white background, `#0078d4` (Microsoft blue) color, `8px 16px` padding, `2px solid #0078d4` border, `border-radius: 4px`, bold font.
- `.widget-skip-link:focus`: `transform: translateY(0)` reveals the link; `outline: none` (focus outline removed since the border already provides visual focus indication).
- `.widget-skip-link-preview`: identical visual styles but without the `transform: translateY(-120%)` — always visible.

**3. What part of behavior can be documented from this file?**
- The skip link is only visible when focused (keyboard Tab reveals it via CSS transition).
- It appears at the top-left of its containing positioned ancestor (the first child `<div>` of `#root`), which places it visually at the top of the page.
- The 200ms CSS transition ensures a smooth reveal.
- `z-index: 1000` ensures it appears above all other page content when visible.
- The focus style removes the browser's default `outline` — the blue border serves as the visible focus indicator.

**4. Is it user-facing?**
Yes — entirely user-facing; defines the visual appearance when the skip link is focused.

**5. What new did you learn from this file?**
The `#0078d4` blue color is Microsoft's Fluent UI / Office brand blue — used here for both text and border. This suggests alignment with the Atlas/Mendix design system or intentional borrowing of accessible high-contrast blue for the link color.

---

## src/__tests__/SkipLink.spec.tsx

**1. What is the purpose of this file?**
Unit tests for the `SkipLink` component — verifying DOM insertion, content, href, and cleanup on unmount.

**2. What kind of logic is described in this file?**
- Sets up a `<div id="root">` in `document.body` before each test.
- Verifies: skip link is inserted into `rootElement` (not the React render root); text matches `linkText`; `href` matches `#mainContentId`; `tabIndex` defaults to 0.
- Tests custom link text, custom `mainContentId`, empty `mainContentId` (href = `#`).
- Tests unmount cleanup: skip link `<a>` is removed from DOM after unmount.

**3. What part of behavior can be documented from this file?**
- Confirmed: the widget inserts the link into `#root`, not wherever React renders the component.
- Confirmed: cleanup on unmount removes the portal and its container `<div>`.
- Confirmed: `tabIndex` defaults to `0` (first in tab order) when not explicitly set.
- `href` is `#` when `mainContentId` is empty — the anchor still works but the href target is invalid.

**4. Is it user-facing?**
No — test file.

**5. What new did you learn from this file?**
The test structure reveals that portal cleanup works correctly on React unmount — the `<div>` container created by `useState` lazy initializer is garbage-collected when the React tree is unmounted. This confirms the portal DOM lifecycle is properly tied to the React component lifecycle.

---

## e2e/SkipLink.spec.js

**1. What is the purpose of this file?**
Playwright end-to-end tests verifying the skip link's keyboard accessibility behavior in a real Mendix app.

**2. What kind of logic is described in this file?**
- **DOM presence test**: verifies `.widget-skip-link` is attached but its `getBoundingClientRect().bottom` is `≤ 0` (above viewport, hidden via CSS transform).
- **Keyboard focus test**: pressing Tab focuses the skip link; verifies `rect.top >= 0` (link is now visible within viewport).
- **Navigation test**: Tab → Enter activates skip link; verifies `<main>` element is focused.
- **Attributes test**: verifies link text "Skip to main content", `href="#"`, class `"widget-skip-link mx-name-skipLink1"`.
- **Visual comparison test**: takes screenshot of focused skip link against baseline.

**3. What part of behavior can be documented from this file?**
- Confirmed: skip link is initially above the viewport (`rect.bottom <= 0`).
- Confirmed: Tab key reveals it (focus → CSS transform animates to `translateY(0)`).
- Confirmed: Enter key activates the link and moves focus to `<main>`.
- Confirmed: default `href` is `"#"` (no mainContentId configured in test app).
- The test app uses `mx-name-skipLink1` as the widget name class.

**4. Is it user-facing?**
No — test file.

**5. What new did you learn from this file?**
The E2E test validates the complete keyboard accessibility flow: hidden → Tab → visible → Enter → main focused. This is exactly the intended WCAG 2.4.1 (Bypass Blocks) accessibility pattern. The `<main>` element focus check confirms the programmatic `main.focus()` call works end-to-end.

---

## CHANGELOG.md

**1. What is the purpose of this file?**
Version history — currently only an "Unreleased" section with "Created skiplink widget."

**2. What kind of logic is described in this file?**
No released versions. Only: "Added: Created skiplink widget."

**3. What part of behavior can be documented from this file?**
- Widget is brand new and not yet published (all changes in "Unreleased").

**4. Is it user-facing?**
No — developer changelog.

**5. What new did you learn from this file?**
This is the only widget in this set with zero released versions — it is still in development/pre-release state. This explains the empty `helpUrl` in the XML and the minimal feature set.

---

## Summary of Key Findings

- **Purpose**: WCAG 2.4.1 (Bypass Blocks) compliance widget — provides keyboard-accessible "skip to content" navigation for screen reader and keyboard users.
- **DOM architecture**: Uses `createPortal` to insert `<a>` as the first child of `#root` — not in React's normal render tree. This ensures it is the first interactive element in DOM order.
- **Focus management**: Programmatically focuses the target element; temporarily adds `tabindex="-1"` if needed and removes it on blur to keep the target out of the tab order otherwise.
- **CSS reveal pattern**: Normally hidden via `transform: translateY(-120%)` with a 200ms transition; revealed on `:focus` via `transform: translateY(0)`.
- **Two target modes**: Explicit `mainContentId` (by element ID) or auto-detect `<main>` tag.
- **No entity context**: `needsEntityContext` not set — can be placed freely without a data source.
- **No Mendix actions**: No configurable onClick/onChange events — purely accessibility-utility widget.
- **Pre-release**: CHANGELOG shows only "Unreleased" — widget not yet published to Marketplace.
- **Accessibility category**: First widget in this set categorized under "Accessibility" in Studio Pro.
- **Web-only**: `supportedPlatform="Web"` — not for native mobile.
