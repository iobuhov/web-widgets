# Draft: rich-text-web

Widget package: `packages/pluggableWidgets/rich-text-web`

---

## src/RichText.xml

**1. What is the purpose of this file?**
Widget descriptor XML defining the widget's identity, all configurable props, and Studio Pro categorization. Generates `RichTextProps.d.ts`.

**2. What kind of logic is described in this file?**
Declares 40+ properties in multiple groups: Data (string attribute, image datasource, optional default image upload action); Toolbar (preset: basic/standard/full/custom; position: auto/top/bottom/hide; custom toolbar groups; advanced custom button list); General (read-only style: text/bordered/panel; height/min-height/max-height with px/percentage/viewport units; enable-status-bar; status-bar content type: word-count/character-text/character-html); Events (onFocus, onBlur, onChange with type: always/onLeave; onLoad); Custom Fonts (list of name+CSS declaration pairs); Link validation toggle. System properties: Name, Editability, Visibility.

**3. What part of behavior can be documented from this file?**
- Toolbar presets: basic, standard, full, custom (two custom levels: group toggles and explicit button list with separators).
- Toolbar position "auto" enables sticky toolbar that attaches to the viewport top when scrolling past the editor.
- Three read-only display styles: `text` (plain rich text, no border), `bordered`, `panel` (read panel style).
- Status bar can show word count, character count (text only), or character count (including HTML).
- Widget accepts an optional image datasource (ListValue) and optional default image upload action — enabling entity-based image management.
- `onChange` can fire on every change (`always`) or only when the editor loses focus (`onLeave`).
- Custom fonts are provided as name/CSS-declaration pairs and merged with the default font list.
- Widget is NOT offline capable (no `offlineCapable="true"` attribute).

**4. Is it user-facing?**
Defines the developer-facing configuration interface in Studio/Studio Pro.

**5. What new did you learn from this file?**
The widget has two levels of custom toolbar customization: a "basic" custom mode using boolean group flags, and an "advanced" custom mode with an explicit per-button list where separator items create new toolbar groups. This two-tier custom system allows lightweight configuration for most cases and surgical control for advanced users.

---

## src/RichText.tsx

**1. What is the purpose of this file?**
The root Mendix pluggable widget entry component. It handles Dojo runtime compatibility, wraps the editor with validation alerts, and manages a loading state during initial render.

**2. What kind of logic is described in this file?**
Uses a `MutationObserver` to detect when the widget element is moved from Mendix's incubator div into the actual page content — a Dojo runtime artifact. Only after this transition does the widget fully mount the editor. Shows a Mendix progress indicator (`.mx-progress`) during this loading phase. Wraps `EditorWrapper` with `ValidationAlert` for the bound string attribute.

**3. What part of behavior can be documented from this file?**
- There is a brief loading phase at widget initialization caused by Mendix's Dojo runtime incubator pattern — users may see a spinner before the editor appears.
- Validation alerts from the string attribute appear below the editor.
- The widget applies a read-only CSS class to the container when `readOnlyStyle="text"` or the attribute is read-only.

**4. Is it user-facing?**
Yes — the loading spinner and validation alerts are user-visible.

**5. What new did you learn from this file?**
The Dojo incubator pattern is a Mendix-specific rendering artifact where widget DOM nodes are temporarily attached to an off-screen div before being inserted into the page. The `MutationObserver` in this widget is specifically a workaround for this platform behavior.

---

## src/EditorWrapper.tsx

**1. What is the purpose of this file?**
The main orchestration layer for the editor — manages editor state, toolbar/status bar visibility, event debouncing, word/character counting, and integration of Mendix action callbacks.

**2. What kind of logic is described in this file?**
Debounces `onChange` by 200 ms using `useDebounce`. Computes word count (split on whitespace) and character count (text-only or including HTML tags). Manages fullscreen state via context (EditorProvider). Propagates `defaultFontFamily`/`defaultFontSize` to the Quill theme. Controls toolbar visibility (hidden when read-only unless style is `"text"`). Controls status bar visibility via `enableStatusBar` prop.

**3. What part of behavior can be documented from this file?**
- `onChange` is debounced at 200 ms — rapidly-typed characters do not produce one Mendix attribute write per keystroke.
- Word count splits text content on whitespace — multiple spaces count as a single word boundary.
- Character count has two modes: text-only (strips HTML tags before counting) or HTML-including (counts raw HTML character length).
- Toolbar is hidden when editor is in read-only mode UNLESS `readOnlyStyle="text"` (which shows the content as plain rich text, no toolbar needed).
- The status bar is fully suppressed when `enableStatusBar=false`, regardless of content type.

**4. Is it user-facing?**
Yes — word/character count in the status bar and the toolbar are directly visible.

**5. What new did you learn from this file?**
The `defaultFontFamily` and `defaultFontSize` are propagated to the Quill theme at EditorWrapper level — meaning the "default font" shown when no explicit font is selected is driven by widget configuration, not hardcoded in the theme. This enables the widget to match the surrounding page font.

---

## src/Editor.tsx

**1. What is the purpose of this file?**
The core Quill editor integration component. Manages the Quill instance lifecycle, handles external value updates, coordinates modal dialogs, and registers custom handlers.

**2. What kind of logic is described in this file?**
Quill is used as an uncontrolled component via `useRef` — React does not manage Quill's internal state. External value updates (from Mendix attribute changes) are applied only when the editor is not focused (prevents focus loss mid-edit). Content is set using `updateContents` (delta-based) rather than `setContents`, avoiding full re-initialization. Custom toolbar handler bindings: link, image, video, indent, view-code. Image upload delegates to the `MxUploader` module (file-based or entity-based). `TABLE-BETTER` module is configured with column/row/merge/delete menu items. Resize module is applied to images and videos.

**3. What part of behavior can be documented from this file?**
- External data updates only apply when the editor is NOT focused — a user actively typing will not have their cursor or selection disrupted by attribute value changes.
- The editor uses Quill's `updateContents` API (not `setContents`) for incremental updates.
- Link, image, and video insertion trigger dedicated modal dialogs.
- The "view code" toolbar button opens a CodeMirror-based HTML code viewer.
- Image/video resizing is enabled via the resize module — users can drag handles to resize embedded media.
- Tables support column/row operations (insert/delete) and cell merging via `TABLE-BETTER` context menu.

**4. Is it user-facing?**
Yes — this is the editing interface.

**5. What new did you learn from this file?**
The focus check before applying external value updates is a critical UX detail. Without it, a long-running Mendix data refresh could overwrite a user's in-progress edits. The 200 ms debounce on `onChange` (in EditorWrapper) combined with the focus guard (in Editor) forms a two-layer protection against data race conditions.

---

## src/MxQuill.ts

**1. What is the purpose of this file?**
An extension of the base Quill class that overrides HTML output generation and `setContents` behavior for widget-specific requirements.

**2. What kind of logic is described in this file?**
`getHTML()` override: generates HTML with correct `list-style-type` CSS for custom list types. List indentation cascades style: level 0 → `ordered`, level 1 → `lower-alpha`, level 2 → `lower-roman` (for ordered lists). `setContents` override: internally calls `updateContents` to prevent full re-initialization. `customFonts` and `validateURL` are registered as Quill-level configuration.

**3. What part of behavior can be documented from this file?**
- Ordered list indentation automatically changes style: top-level → numeric, first indent → lower-alpha, second indent → lower-roman.
- Custom fonts registered at the Quill level (not just CSS) are validated before application.
- Link URL validation is optional — controlled by the `validateURL` flag from widget configuration.
- The `setContents`→`updateContents` override ensures delta operations are always incremental.

**4. Is it user-facing?**
No — internal Quill customization. Affects HTML output.

**5. What new did you learn from this file?**
The multi-level ordered list style cascade (numeric → lower-alpha → lower-roman) is a business logic decision hardcoded in `MxQuill`. This means Mendix's rich text widget always applies this indentation convention, regardless of the source content's explicit list style.

---

## src/customPluginRegisters.ts

**1. What is the purpose of this file?**
Central registration point for all custom Quill formats, modules, blots, and themes used by the widget.

**2. What kind of logic is described in this file?**
Registers via `Quill.register()`: custom formats (list item, video, image, softbreak, button, block); style attributors (direction, alignment, indent, whitespace); modules (MxUploader, scroll, quill-resize-module, table-better, keyboard bindings); MendixTheme. All registrations happen once at module load time.

**3. What part of behavior can be documented from this file?**
- The `softbreak` format enables Shift+Enter to insert a line break (`<br>`) within a paragraph, rather than starting a new paragraph.
- The `whitespace` attributor preserves CSS `white-space` property in HTML output.
- `MxUploader` replaces Quill's default image uploader module entirely.
- `table-better` replaces Quill's default table module.
- All registrations are global — they apply to ALL Quill instances on the page (module-level side effects).

**4. Is it user-facing?**
No — plugin infrastructure. Affects all editor behaviors.

**5. What new did you learn from this file?**
Quill's `register()` is a global singleton operation — registering a custom module affects ALL Quill instances on a page, not just the one in this widget. If a page has multiple rich-text widgets, all share the same registered modules. This is an architectural constraint of Quill's design.

---

## src/Toolbar.tsx

**1. What is the purpose of this file?**
The toolbar React component that renders formatted groups of toolbar buttons based on the configured preset and custom configuration.

**2. What kind of logic is described in this file?**
Maps toolbar content (arrays of button specs) through `TOOLBAR_MAPPING` to React components. Filters button visibility by preset level (basic=1, standard=2, full=3, custom=4). Keyboard navigation: Tab moves between groups, Arrow keys move within groups, Enter activates buttons. Custom fonts are appended to the default font list and sorted alphabetically.

**3. What part of behavior can be documented from this file?**
- Toolbar buttons are organized into groups; Tab key moves between groups, Arrow keys navigate within groups.
- Preset level controls which buttons are visible: basic shows minimal set; full shows all available buttons.
- Custom fonts (from widget config) are merged with the default font list alphabetically.
- The toolbar re-renders when fullscreen state changes (to update button active states).

**4. Is it user-facing?**
Yes — the toolbar is the primary formatting interface.

**5. What new did you learn from this file?**
The preset system is additive: a button with `presetValue=2` appears in standard AND full (any preset >= 2). This means increasing the toolbar preset always reveals more buttons, never removes ones from lower presets.

---

## src/presets.ts

**1. What is the purpose of this file?**
Generates the toolbar button configuration arrays from widget props, for each toolbar preset mode.

**2. What kind of logic is described in this file?**
For `basic`/`standard`/`full` presets: returns the hardcoded `DEFAULT_TOOLBAR` constant. For custom "basic mode": maps `TOOLBAR_GROUP` boolean flags from props to toolbar group arrays. For custom "advanced mode": parses the `advancedConfig` array from props — items marked as separators create new toolbar group boundaries.

**3. What part of behavior can be documented from this file?**
- Basic, standard, and full presets all use the same `DEFAULT_TOOLBAR` — the preset level is only used to filter button visibility, not to define different toolbar layouts.
- Custom "basic" mode provides boolean group toggles (e.g., "show history buttons", "show font style buttons").
- Custom "advanced" mode provides an explicit per-button list with separator items to group buttons.
- Separators in the advanced config create new toolbar groups (affecting Tab navigation boundaries).

**4. Is it user-facing?**
No — determines toolbar structure.

**5. What new did you learn from this file?**
The preset modes (basic/standard/full) share the same DEFAULT_TOOLBAR array — the distinction is purely in the filter applied at render time. This means a "basic" toolbar can show all buttons if a developer misconfigures `presetValue` on a button.

---

## src/toolbar/constants.ts

**1. What is the purpose of this file?**
Defines all toolbar button types, their icons/values, preset levels, and the `TOOLBAR_GROUP` enumeration for the basic custom mode.

**2. What kind of logic is described in this file?**
`TOOLBAR_MAPPING`: 31+ button/dropdown/component entries. Each entry specifies: `type` (button/dropdown/component), toolbar `format` name, `icon` (SVG), `value` (Quill format value), and `presetValue` (minimum preset level). `TOOLBAR_GROUP`: groups buttons into logical categories (history, fontStyle, colorStyle, list, indentGroup, linkGroup, table, mediaGroup, codeGroup, viewCode, fullscreen). `DEFAULT_TOOLBAR`: ordered array of toolbar group arrays.

**3. What part of behavior can be documented from this file?**
- `presetValue=1` (basic): bold, italic, underline, strikethrough, ordered/unordered list, link, undo/redo.
- `presetValue=2` (standard): adds superscript, subscript, font/size dropdowns, text/background color.
- `presetValue=3` (full): adds indent/outdent, code block, block quote, image, video, table, fullscreen, view-code.
- The `viewCode` button (HTML source viewer) is a full preset (3) only feature.
- Fullscreen is a full preset (3) feature.

**4. Is it user-facing?**
No — button configuration definitions. Determines what appears in the toolbar.

**5. What new did you learn from this file?**
The `viewCode` button opens a CodeMirror-based HTML source editor — users on the "full" toolbar can inspect and directly edit the raw HTML of the rich text content. This is a power-user feature with XSS implications if the content is later rendered without sanitization.

---

## src/keyboard.ts

**1. What is the purpose of this file?**
Defines custom Quill keyboard bindings for accessibility navigation and special input behaviors.

**2. What kind of logic is described in this file?**
Bindings: Alt+F10 → focus toolbar; Alt+F11 → focus status bar; Shift+Tab → move focus backward; Tab → indent (in blockquote/list) or focus next (elsewhere); Shift+Enter → insert softbreak (`<br>`); Enter → standard line break with format preservation. Table-related bindings from the `table-better` module.

**3. What part of behavior can be documented from this file?**
- Alt+F10 moves keyboard focus to the toolbar (accessibility pattern from ARIA).
- Alt+F11 moves keyboard focus to the status bar.
- Shift+Enter inserts a `<br>` (line break within paragraph) rather than creating a new block element.
- Tab indents in blockquotes and lists; in regular paragraphs Tab moves focus out of the editor.
- All shortcuts are registered at the Quill keyboard module level, not at the DOM event level.

**4. Is it user-facing?**
Yes — keyboard shortcuts directly affect user interaction.

**5. What new did you learn from this file?**
The Tab key behavior is context-sensitive: it indents within list/blockquote contexts (common text editor convention) but moves focus OUT of the editor in regular text (accessibility requirement). This dual behavior requires a conditional binding that checks the current format context.

---

## src/useActionEvents.ts

**1. What is the purpose of this file?**
Custom hook that manages focus/blur event handling, distinguishing internal focus transfers (toolbar, modals) from genuine editor blur events.

**2. What kind of logic is described in this file?**
`onFocus` handler: fires `props.onFocus?.execute()` when focus enters the editor for the first time (not from internal element transitions). `onBlur` handler: checks if the new focus target (`relatedTarget`) is inside the editor container — if so, the blur is internal (toolbar click, modal open) and `onBlur`/`onChange` are NOT fired. Tracks `editorText` at focus time for `onChangeType="onLeave"` — fires `onChange` only if text actually changed.

**3. What part of behavior can be documented from this file?**
- Clicking a toolbar button does NOT trigger the `onBlur` action — focus stays "within" the editor from the Mendix action perspective.
- Opening a link/image/video dialog does NOT trigger `onBlur`.
- `onChangeType="onLeave"`: onChange fires only when (a) the editor genuinely loses focus AND (b) the text content has changed since last focus. If the user clicks in and out without editing, onChange does NOT fire.
- `onFocus` fires only on the first focus entry, not on every internal focus transfer.

**4. Is it user-facing?**
No — internal event filtering. Affects when Mendix action events fire.

**5. What new did you learn from this file?**
The `relatedTarget` check in the blur handler is a DOM API technique for detecting focus transfers within a container. This is the correct (non-hacky) way to implement "editor blur" semantics in a complex UI with toolbars and dialogs — without it, every toolbar button click would incorrectly fire the Mendix `onBlur` action.

---

## src/dialogs/

**1. What is the purpose of these files?**
Modal dialog components for link editing (`LinkDialog.tsx`), image insertion (`ImageDialog.tsx`), video embedding (`VideoDialog.tsx`), and HTML source editing (`ViewCodeDialog.tsx`), plus shared dialog scaffolding (`Dialog.tsx`, `DialogContent.tsx`, `DialogFooter.tsx`, `DialogHeader.tsx`).

**2. What kind of logic is described in these files?**
`Dialog.tsx`: uses `floating-ui` for overlay positioning and focus trapping. `LinkDialog`: fields for URL, display text, title, and link target (new tab vs same window). `ImageDialog`: tabs for URL/file upload/entity image; fields for alt text, width, height with aspect ratio lock; entity image gallery when datasource is configured. `VideoDialog`: tabs for URL vs embed code; auto-detects YouTube/Vimeo/DailyMotion/Google Maps URL patterns and extracts embed dimensions. `ViewCodeDialog`: CodeMirror editor with HTML beautification via `js-beautify`.

**3. What part of behavior can be documented from this file?**
- Link dialog: link target (`_blank` = new tab, `_self` = same tab) is configurable per link.
- Image dialog has three source modes: direct URL, file upload (base64 conversion), entity image (from datasource).
- When an image datasource is configured, the image dialog shows a gallery of available images.
- Video URLs auto-populate dimensions: YouTube → 560×314, Vimeo → 425×350, DailyMotion → 480×270.
- The ViewCode dialog preserves tabs in HTML output by converting them to em-spaces (workaround for CodeMirror tab-handling).
- All dialogs use floating-ui for positioning, preventing overflow clipping.

**4. Is it user-facing?**
Yes — all dialogs are user-visible editing UIs.

**5. What new did you learn from this file?**
The `ViewCodeDialog` (HTML source editor) uses CodeMirror and `js-beautify` to format the raw HTML before presenting it to the user. This means the HTML the user sees in the code view may differ slightly in whitespace from what Quill produces internally — formatting is applied purely for readability.

---

## src/uploader.ts

**1. What is the purpose of this file?**
Custom Quill uploader module replacing the default Quill image upload behavior.

**2. What kind of logic is described in this file?**
`MxUploader` extends Quill's `Uploader`. When `setEntityUpload=false` (default): reads selected files, converts to base64 data URLs, inserts as inline images. When `setEntityUpload=true`: delegates the upload to an external Mendix widget via `ACTION_DISPATCHER` instead of handling locally. MIME type filtering is applied to the file input `accept` attribute.

**3. What part of behavior can be documented from this file?**
- Default image upload embeds images as base64 data URLs — no server upload, images stored inline in the attribute value.
- When an image datasource is configured with a default upload action, uploads are delegated to that action (entity-based upload).
- File type filtering (MIME types) is enforced at the file picker level via the `accept` attribute.
- `ACTION_DISPATCHER` is a shared event bus pattern for cross-component communication.

**4. Is it user-facing?**
No — internal upload mechanism. Affects how images are stored.

**5. What new did you learn from this file?**
The default base64 upload stores images INLINE in the rich text string attribute value. For large images or many images, this can make the attribute value extremely large. The entity-based upload alternative is essential for production use cases with significant image content.

---

## src/formats/video.ts, image.ts, link.ts

**1. What is the purpose of these files?**
Custom Quill format implementations for video (`<iframe>`), image (`<img>`), and link (`<a>`) elements, extending Quill's default handling.

**2. What kind of logic is described in these files?**
`CustomVideo`: embeds video as `<iframe>` with configurable width/height; supports multiple URL platforms via `videoUrlPattern.ts`. `CustomImage`: adds `alt`, `width`, `height`, and `data-src` attribute support to Quill's default image blot; `data-src` stores entity GUID references for entity-based images. `CustomLink`: adds `title` and `target` attribute support; applies `linkify` URL protocol injection (adds `https://` if no protocol specified); validates URLs against a configurable URL pattern.

**3. What part of behavior can be documented from this file?**
- Video embeds use `<iframe>` elements with platform-specific src URL transformation.
- Entity images are stored in the HTML via `data-src` attribute containing the entity GUID — the actual image URL is resolved at render time.
- Links without a protocol prefix get `https://` prepended automatically when saved.
- Link URL validation uses a configurable pattern (optional — can be disabled via `validateURL=false`).
- Images support explicit width/height attributes — set via the image dialog.

**4. Is it user-facing?**
No — internal Quill format implementations. Affects the HTML structure of saved content.

**5. What new did you learn from this file?**
Entity image GUIDs are stored in `data-src` rather than `src` in the saved HTML. This means the widget's saved HTML is not self-contained — rendering the HTML in any other context requires resolving entity GUIDs back to image URLs. The widget provides a mechanism to do this, but the HTML saved to the Mendix attribute contains Mendix-platform-specific data attributes.

---

## src/themes/mxTheme.ts

**1. What is the purpose of this file?**
Mendix-branded Quill theme that extends the default Quill "snow" theme to match the Mendix Atlas design system.

**2. What kind of logic is described in this file?**
Extends `SnowTheme` (Quill's default). Overrides tooltip styling to use Mendix CSS custom properties (Atlas variables). Sets `defaultFontFamily` and `defaultFontSize` on the Quill theme at initialization time. Registers `MxTooltip` as the tooltip component.

**3. What part of behavior can be documented from this file?**
- The editor's visual styling uses Mendix Atlas CSS variables — font colors, border colors, and focus states match the Mendix design system.
- Default font family and size set at theme level control what text looks like when no explicit font format is applied.
- The tooltip (shown when hovering over toolbar buttons) uses Mendix-styled positioning.

**4. Is it user-facing?**
Yes — theme affects the visual appearance of the editor.

**5. What new did you learn from this file?**
By extending Quill's `SnowTheme`, the widget inherits Quill's standard toolbar/editor CSS and only overrides the parts needed for Mendix branding. This approach is more maintainable than a full custom theme.

---

## src/StickySentinel.tsx

**1. What is the purpose of this file?**
Implements sticky toolbar behavior — the toolbar docks to the viewport top when the user scrolls past the editor start.

**2. What kind of logic is described in this file?**
Renders a sentinel `<div>` at the top of the editor container. An `IntersectionObserver` watches this sentinel with `threshold: [0, 1]`. When the sentinel is fully hidden (scroll ratio = 0), applies `"container-stuck"` CSS class to the editor container. When fully visible, removes the class.

**3. What part of behavior can be documented from this file?**
- The sticky toolbar is activated by the `IntersectionObserver` detecting when the sentinel leaves the viewport.
- The `"container-stuck"` CSS class triggers the toolbar's sticky positioning via SCSS.
- This only applies when toolbar position is set to `"auto"` — other positions (top/bottom/hide) do not use this sentinel.
- The `IntersectionObserver` approach is more performant than a `scroll` event listener.

**4. Is it user-facing?**
Yes — sticky toolbar is visible to users when scrolling a long page with the editor.

**5. What new did you learn from this file?**
The `IntersectionObserver` sentinel pattern is a standard web platform approach for sticky elements that avoids scroll event listener overhead. The sentinel is a zero-height element placed exactly at the scroll trigger point.

---

## src/ui/RichText.scss

**1. What is the purpose of this file?**
Main stylesheet for the rich text widget — imports Quill/table-better CSS, applies Mendix styling overrides, handles fullscreen mode, and controls toolbar/editor layout.

**2. What kind of logic is described in this file?**
Imports: Quill snow theme CSS, table-better CSS. Widget container: flex column layout. Fullscreen: `position: fixed; inset: 0; z-index: 1000; background: white; height: 100vh; width: 100vw`. Sticky toolbar: `position: sticky; top: 0; z-index: 10` when `.container-stuck`. Read-only styles: removes toolbar, adjusts border/background per `readOnlyStyle`. Editor height: applies min-height, max-height, overflow from props.

**3. What part of behavior can be documented from this file?**
- Fullscreen mode uses `position: fixed; inset: 0` — covers the entire viewport including any scrolled content.
- Sticky toolbar uses `position: sticky; top: 0` — naturally docks to viewport top when scrolling.
- `readOnlyStyle="text"` removes the Quill editor chrome (toolbar, border) to show clean rich text.
- `readOnlyStyle="bordered"` retains the border styling around the content.
- Min/max height and overflow are applied to the editor's content area (not the toolbar).

**4. Is it user-facing?**
Yes — all visual layout is defined here.

**5. What new did you learn from this file?**
Fullscreen mode uses `z-index: 1000` — this may conflict with Mendix's own modal dialogs (which also use high z-index values). The fullscreen widget covers the entire page, including any Mendix navigation or popup dialogs that were already open.

---

## src/components/__tests__/RichText.spec.tsx

**1. What is the purpose of this file?**
Unit tests for the `RichText` widget rendering with different configurations.

**2. What kind of logic is described in this file?**
Snapshot tests for: basic preset; full preset; read-only text style; read-only bordered style; read-only panel style; status bar with word count; status bar with character count (text); status bar with character count (HTML). Uses mocked Quill to avoid DOM dependencies.

**3. What part of behavior can be documented from this file?**
- All five read-only style variants are snapshot-tested.
- All three status bar content types are snapshot-tested.
- Quill is mocked in unit tests — no actual editor initialization occurs.
- Both basic and full toolbar presets produce different rendered structures.

**4. Is it user-facing?**
No — test file only.

**5. What new did you learn from this file?**
The unit tests mock Quill entirely — this means unit tests cover only React component structure and prop routing, not Quill's actual editor behavior. Actual editor functionality (typing, formatting, toolbar actions) would need e2e tests for full coverage.

---

## CHANGELOG.md

**1. What is the purpose of this file?**
Documents version history for the rich-text-web widget.

**2. What kind of logic is described in this file?**
Key versions: 4.12.0 (link URL validation optional), 4.11.x (list style bug fixes, empty line limits), 4.10.x (character count variants, overflow configuration), 4.9.x (keyboard shortcut improvements), 4.8.x (bug fixes), 4.7.0 (custom font support), 4.6.1 (table insertion), 4.5.0 (fullscreen mode), 4.4.0 (image resizing), 4.1.0 (custom toolbar advanced mode), 4.0.0 (migration from TinyMCE to Quill V2 — removes iframe requirement).

**3. What part of behavior can be documented from this file?**
- v4.0.0 was a major breaking change: replaced TinyMCE with Quill V2, eliminating the iframe-based rendering model.
- v4.12.0 made link URL validation optional — previously all URLs were validated against a pattern.
- v4.11.x fixed list style preservation (lower-alpha/lower-roman) and limited empty line proliferation.
- v4.6.1 added table support (`TABLE-BETTER` module).
- v4.5.0 added fullscreen mode.
- v4.4.0 added image resizing.
- v4.7.0 added custom font support (name + CSS declaration pairs).
- Keyboard shortcuts (Alt+F10, Alt+F11, Shift+Enter) were added across multiple versions.

**4. Is it user-facing?**
The changelog is publicly visible on the Mendix Marketplace.

**5. What new did you learn from this file?**
The v4.0.0 migration from TinyMCE to Quill V2 was a complete rewrite of the underlying editor. The "removes iframe requirement" note is significant — TinyMCE used an `<iframe>` sandbox for the editor, which caused CSP issues, styling isolation problems, and accessibility challenges. Quill renders directly in the document DOM, eliminating all of those issues.
