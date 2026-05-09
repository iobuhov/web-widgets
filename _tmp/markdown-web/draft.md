# Draft: markdown-web

Widget package: `packages/pluggableWidgets/markdown-web`

---

## src/Markdown.tsx

**1. What is the purpose of this file?**
This is the root React component for the Markdown Viewer widget. It renders a Mendix string attribute as parsed HTML using the `markdown-it` library. It is the sole runtime component — the widget has no sub-components.

**2. What kind of logic is described in this file?**
A module-level singleton `MarkdownIt` parser is created once with `"default"` preset, `typographer: true` (smart quotes, dashes), and `linkify: true` (auto-links URLs). A `useEffect` hook watches `stringAttribute.value` and `stringAttribute.status`; when either changes, it calls `mdParser.render()` and writes the HTML string directly to `previewRef.current.innerHTML`. While the attribute is unavailable, a loading spinner (`<div className="mx-progress">`) is rendered instead.

**3. What part of behavior can be documented from this file?**
- The widget renders Markdown content using `innerHTML` injection, not React virtual DOM — this means the rendered HTML is not sanitized by React's built-in escaping. The Markdown-it library itself controls what HTML is output.
- Markdown-it is configured with `typographer: true` — this transforms straight quotes to typographic quotes, double dashes to em-dashes, and triple dots to ellipsis characters.
- `linkify: true` — bare URLs in the Markdown source are automatically converted to `<a>` links.
- When `stringAttribute.status` is not "available" (loading or unavailable), the widget renders a Mendix progress spinner (`.mx-progress`) instead of content — there is no empty state; the transition is directly from spinner to content.
- Re-render is triggered by changes to either `stringAttribute.value` OR `stringAttribute.status`, ensuring the view updates when data loads.

**4. Is it user-facing?**
Yes — this produces the visible Markdown output rendered as HTML in the browser.

**5. What new did you learn from this file?**
The `MarkdownIt` parser instance is created at module scope (outside the component function), meaning it is shared as a singleton across all Markdown widget instances on a page. This is an optimization that avoids recreating the parser on each render, but it also means all instances share the same parser configuration.

---

## typings/MarkdownProps.d.ts

**1. What is the purpose of this file?**
Auto-generated TypeScript typings from `Markdown.xml`. Defines `MarkdownContainerProps` (runtime props) and `MarkdownPreviewProps` (design-mode preview props).

**2. What kind of logic is described in this file?**
No logic. `MarkdownContainerProps` has: `name`, `tabIndex?`, `id`, and `stringAttribute: EditableValue<string>`. `MarkdownPreviewProps` has: `readOnly`, `renderMode`, `translate`, and `stringAttribute: string`.

**3. What part of behavior can be documented from this file?**
- The widget accepts exactly one data prop: `stringAttribute` — a Mendix `EditableValue<string>`. Despite the type allowing editable values, the widget is a viewer only (renders HTML from the value, does not provide any editing UI).
- The XML description notes "recommendation is to use an unlimited string data type" — meaning the attribute should be configured as an unlimited String in the Mendix domain model to avoid truncation of long Markdown content.
- The widget requires an entity context (`needsEntityContext="true"` in XML).

**4. Is it user-facing?**
No — internal type declarations only.

**5. What new did you learn from this file?**
`stringAttribute` is typed as `EditableValue<string>` — meaning the widget is compatible with editable attributes. However, there is no write/edit functionality in the component code; the widget only reads the value. This is an API affordance that is not utilized for editing.

---

## src/Markdown.xml

**1. What is the purpose of this file?**
Widget descriptor XML defining the widget's identity, props schema, and Studio Pro categorization. Generates `MarkdownProps.d.ts`.

**2. What kind of logic is described in this file?**
Declares one property: `stringAttribute` of type `attribute` accepting `String` attribute types. Also declares system properties for Label and Visibility (conditional visibility). The widget is: `needsEntityContext="true"`, `pluginWidget="true"`, `offlineCapable="true"`, categorized in "Display" for both Studio and Studio Pro.

**3. What part of behavior can be documented from this file?**
- The widget supports **conditional visibility** natively via Mendix system property.
- The widget is **offline capable** — it can function in offline-first Mendix apps.
- Only a single `String` attribute type is accepted (no Long String type distinction in the schema — the recommendation to use unlimited string is documentation-level only, not schema-enforced).
- The widget is display-only from the schema perspective: no action properties, no write-back mechanism.
- Widget display name in Studio is "Markdown viewer".

**4. Is it user-facing?**
Defines the developer-facing configuration interface in Studio/Studio Pro.

**5. What new did you learn from this file?**
The widget is marked `offlineCapable="true"`, meaning it can render Markdown content in Mendix offline apps. Since it only reads a string attribute and renders client-side via `markdown-it`, no server round-trip is needed at runtime, making offline operation fully feasible.

---

## src/Markdown.editorConfig.ts

**1. What is the purpose of this file?**
Provides the `getPreview` function for Studio's "structure preview" mode (XRay/structure canvas view), and a `getCustomCaption` helper.

**2. What kind of logic is described in this file?**
`getPreview` returns a structured preview specification: a bordered, rounded container with the attribute name shown as `[attributeName]` in the data text color. The background changes based on `readOnly` state (`containerDisabled` vs `container`). `getCustomCaption` formats the preview label as `[{stringAttribute}]` or `[No attribute selected]` when no attribute is configured.

**3. What part of behavior can be documented from this file?**
- In structure/XRay preview mode in Studio Pro, the widget shows a placeholder that displays the bound attribute name in square brackets (e.g., `[myStringAttr]`).
- The preview is dark-mode aware — uses `structurePreviewPalette` which adapts colors based on the `isDarkMode` parameter.
- When no attribute is configured, the placeholder shows `[No attribute selected]`.
- No specific properties are hidden or validated in this file — there is no `getProperties` or `check` function for this widget.

**4. Is it user-facing?**
Yes — visible to developers in Studio Pro's structure preview / XRay mode.

**5. What new did you learn from this file?**
This widget has no `getProperties` or `check` export — meaning there are no conditional prop visibility rules and no validation errors beyond what Mendix enforces by default (required attribute must be set). The editor config is minimal: only a structure preview.

---

## src/Markdown.editorPreview.tsx

**1. What is the purpose of this file?**
Provides the live React-based preview component for Studio's design-mode canvas.

**2. What kind of logic is described in this file?**
Renders a `<div className="widget-markdown preview">` containing a `<p>` tag that shows `[attributeName]` or `[No attribute selected]`. Does not render actual Markdown content in design mode — shows only the attribute reference as a placeholder.

**3. What part of behavior can be documented from this file?**
- In design mode, the widget does not render any actual Markdown output — it shows only the attribute binding reference as text.
- The preview container uses both `widget-markdown` and `preview` classes, triggering the `min-height: max-content` style from `Markdown.scss`.
- The design-mode preview imports `Markdown.scss`, so table/image styles are visible in the design canvas even for the placeholder view.

**4. Is it user-facing?**
Yes — visible to developers in Studio design canvas.

**5. What new did you learn from this file?**
Design mode does not execute `markdown-it` rendering — the preview is entirely static placeholder text. This is consistent with the widget's `innerHTML` injection approach, which is only safe and applicable at runtime with real attribute data.

---

## src/ui/Markdown.scss

**1. What is the purpose of this file?**
CSS styles for the Markdown widget container. Defines layout behavior and styles for HTML elements generated by Markdown rendering (tables, images, horizontal rules).

**2. What kind of logic is described in this file?**
- `.widget-markdown`: `display: flex; flex-direction: column; flex: 1; width: 100%; justify-content: flex-start` — the widget occupies full width and grows vertically.
- Table cells (`th`, `td`): 1px solid `#ccc` border, 8px padding, left-aligned text.
- Table headers (`th`): `#f2f2f2` background.
- Images (`img`): `max-width: 35%` — constrains images to at most 35% of the container width.
- Horizontal rules (`hr`): `width: 100%`.
- `.widget-markdown.preview`: `min-height: max-content` to ensure the design-mode preview has visible height.

**3. What part of behavior can be documented from this file?**
- Markdown-rendered tables have a built-in visual style: bordered cells with header row background.
- Images embedded in Markdown content are constrained to 35% of container width — large images will not overflow the container but also cannot be displayed at full width.
- The widget takes up full available width and grows to fit its content vertically.
- HR elements (`---` in Markdown) span the full container width.

**4. Is it user-facing?**
Yes — these styles directly affect the visual appearance of rendered Markdown content.

**5. What new did you learn from this file?**
The 35% max-width constraint on images is a hardcoded design decision — images in Markdown content will always be limited to 35% of the widget container width regardless of the source image size or container dimensions.

---

## src/__tests__/Markdown.spec.tsx

**1. What is the purpose of this file?**
Unit tests for the `Markdown` component using Jest and Testing Library. Tests two states: normal rendering (attribute available) and loading/unavailable state.

**2. What kind of logic is described in this file?**
Two test cases:
1. Renders `.widget-markdown` element when `stringAttribute` is available with value "Markdown viewer default value" — snapshot test.
2. Renders `.mx-progress` (loading spinner) when `stringAttribute` is unavailable — snapshot test.

Uses `EditableValueBuilder` from `@mendix/widget-plugin-test-utils` to construct test prop values.

**3. What part of behavior can be documented from this file?**
- When `stringAttribute` is unavailable, the widget renders a Mendix progress spinner (`.mx-progress`), not an empty container or error state.
- When the attribute is available, the `.widget-markdown` container is present in the DOM.
- The test uses the `isUnavailable()` builder, confirming the widget handles the Mendix `ValueStatus.Unavailable` state explicitly.

**4. Is it user-facing?**
No — test file only.

**5. What new did you learn from this file?**
The test confirms that the distinction between "unavailable" and "loading" attribute states is handled uniformly — both result in the `.mx-progress` spinner. The widget does not distinguish between these two non-available states in its rendering.

---

## e2e/Markdown.spec.js

**1. What is the purpose of this file?**
End-to-end Playwright tests for the Markdown Viewer widget. Tests five content scenarios via screenshot baselines: basic Markdown, tables, inline HTML, lists, and conditional visibility.

**2. What kind of logic is described in this file?**
All five tests are screenshot baseline comparisons against pre-committed PNG snapshots. Each test navigates to a tab on the default page, locates a specific named Markdown widget, and compares its screenshot. The conditional visibility test additionally clicks a "Yes" radio button to make the widget visible before screenshotting.

**3. What part of behavior can be documented from this file?**
- **Basic Markdown** rendering is e2e-confirmed (headings, paragraphs, bold, italic, etc.).
- **Tables** render correctly (the `markdownTables.png` baseline confirms the bordered table style from `Markdown.scss`).
- **Inline HTML** rendering is tested separately (`markdownInlineHTML.png`) — the test name "do not render inline HTML" suggests inline HTML from the Markdown source is stripped or escaped (the `MarkdownIt` "default" preset does not enable raw HTML by default).
- **Lists** render correctly (ordered and unordered, from `markdownLists.png` baseline).
- **Conditional visibility** is e2e-confirmed to work — the widget respects Mendix visibility conditions and renders correctly when made visible.
- Session logout is performed after each test due to Mendix's 5-session license limit.

**4. Is it user-facing?**
The tested behaviors (rendered Markdown, table styles, visibility) are user-facing.

**5. What new did you learn from this file?**
The test titled "do not render inline HTML" confirms that inline HTML in Markdown source is **not** rendered as HTML — consistent with `MarkdownIt`'s default preset which disables raw HTML output. HTML tags in the Markdown input are displayed as escaped text, not executed as HTML. This is a security-relevant behavioral constraint.

---

## CHANGELOG.md

**1. What is the purpose of this file?**
Documents version history for the markdown-web widget. Three versions: 1.0.0 (initial release), 1.0.1 (added license and dependency docs), 1.0.2 (dependency security updates).

**2. What kind of logic is described in this file?**
No logic — version history only.

**3. What part of behavior can be documented from this file?**
- v1.0.0 (2024-08-16): Initial release of the Markdown Viewer widget.
- v1.0.1 (2026-02-10): Added license file and open source dependency documentation.
- v1.0.2 (2026-04-13): Updated dependencies for security vulnerability mitigation.

No functional behavior changes have been made since the initial release.

**4. Is it user-facing?**
The changelog is publicly visible on the Mendix Marketplace.

**5. What new did you learn from this file?**
The widget is very new (first released August 2024) and has had no functional changes — only maintenance updates. All core behavior (Markdown rendering, `markdown-it` with typographer and linkify, inline HTML suppression) has been stable since v1.0.0.
