# Rich Text

## Purpose

The Rich Text widget provides a full-featured WYSIWYG editor built on Quill V2 within a Mendix page. It binds a writable Mendix string attribute, supports four toolbar presets (basic, standard, full, custom), sticky and positional toolbar modes, three read-only display styles, a status bar with word/character count, embedded image upload (inline base64 or entity-based), video embedding, table management, custom fonts, fullscreen mode, and keyboard accessibility shortcuts. The widget is NOT offline capable. It requires entity context and renders directly in the document DOM (not in an iframe), enabling seamless CSS integration with the Mendix Atlas design system.

## User Scenarios

### [P1] Edit rich text content bound to a Mendix attribute
**Given** a page with a Rich Text widget bound to a string attribute  
**When** the user types in the editor  
**Then** content is written to the attribute 200 ms after the last keystroke (debounced), using Quill's delta-based `updateContents` API (not full re-initialization)

#### Edge Cases
- External attribute value changes (from Mendix data refresh) are applied only when the editor is NOT focused — an actively typing user's cursor and selection are never disrupted.
- Clicking a toolbar button or opening a link/image/video dialog does NOT trigger `onBlur` or `onChange` (on-leave mode) — focus is considered "internal" to the editor.
- When `onChangeType === "onLeave"`, `onChange` fires only when (a) the editor genuinely loses focus AND (b) content has changed since the last focus entry.
- A brief spinner (`.mx-progress`) is shown during the initial loading phase caused by Mendix's Dojo runtime incubator pattern before the editor fully mounts.

### [P2] Configure toolbar preset and position
**Given** `toolbarPreset` is configured as basic, standard, full, or custom  
**When** the editor renders  
**Then** only toolbar buttons up to the configured preset level are visible; the toolbar appears in the configured position

#### Edge Cases
- Preset levels are additive: basic → standard → full reveals progressively more buttons. Increasing the preset never removes buttons visible at a lower preset.
  - basic (preset 1): bold, italic, underline, strikethrough, ordered/unordered list, link, undo/redo.
  - standard (preset 2): adds superscript, subscript, font/size dropdowns, text/background color.
  - full (preset 3): adds indent/outdent, code block, block quote, image, video, table, fullscreen, view-code (HTML source editor).
- The `viewCode` button (CodeMirror-based HTML source editor) is a full-preset-only feature and exposes raw HTML to the user.
- Toolbar position `"auto"` enables sticky behavior: an `IntersectionObserver` sentinel detects when the user scrolls past the editor top and applies `"container-stuck"` class, docking the toolbar to the viewport top via `position: sticky; top: 0`.
- Toolbar position `"hide"` removes the toolbar entirely; position `"top"` or `"bottom"` places it statically.
- Toolbar is hidden in read-only mode unless `readOnlyStyle === "text"`.
- Tab key moves between toolbar groups; Arrow keys navigate within groups.

### [P3] Display content in read-only mode
**Given** `readOnly` is true (or the bound attribute is read-only)  
**When** the editor renders  
**Then** the content is displayed according to `readOnlyStyle` without an interactive editor

#### Edge Cases
- `readOnlyStyle === "text"`: plain rich text rendering, no Quill editor chrome (no border, no toolbar).
- `readOnlyStyle === "bordered"`: retains a border around the content.
- `readOnlyStyle === "panel"`: panel-style display.
- The read-only CSS class is applied to the widget container element.

### [P4] Insert and manage images
**Given** the user clicks the image toolbar button (full preset)  
**When** the image dialog opens  
**Then** the user can insert an image via URL, file upload, or entity image gallery (when a datasource is configured)

#### Edge Cases
- Default image upload (no datasource configured): the selected file is converted to a base64 data URL and embedded inline in the rich text HTML. Large images stored this way will significantly increase the attribute value size.
- Entity-based upload (datasource configured): uploads are delegated to the configured Mendix upload action; the image is stored as a `data-src` attribute containing the entity GUID, not a public URL. Rendering in other contexts requires resolving GUIDs to actual URLs.
- Image dimensions (width/height) are configurable per image via the dialog; aspect ratio lock is available.
- Images can be resized after insertion by dragging resize handles.
- Alt text is configurable per image.

### [P5] Embed video content
**Given** the user clicks the video toolbar button (full preset) and enters a video URL  
**When** the video dialog resolves the URL  
**Then** the video is embedded as an `<iframe>` with platform-specific dimensions auto-populated: YouTube → 560×314, Vimeo → 425×350, DailyMotion → 480×270

#### Edge Cases
- An embed code tab is also available for direct iframe embed code input.
- Google Maps URLs are also recognized and auto-dimensioned.
- Video embeds store as `<iframe>` elements with the transformed embed URL.
- Inserted videos can be resized by dragging resize handles (same module as images).

### [P6] Use keyboard shortcuts for accessibility
**Given** the editor has focus  
**When** the user presses keyboard shortcuts  
**Then** the following actions are available: Alt+F10 → focus toolbar; Alt+F11 → focus status bar; Shift+Enter → insert line break (`<br>`) within current paragraph; Tab → indents in blockquote/list, or moves focus out of editor in regular text

#### Edge Cases
- Tab's behavior is context-sensitive: indents within list/blockquote, exits editor in regular paragraphs (accessibility requirement).
- Shift+Enter inserts a `<br>` (softbreak), not a new block element.
- All shortcuts are registered at the Quill keyboard module level.

### [P7] Fullscreen editing mode
**Given** the user clicks the fullscreen button (full preset toolbar)  
**When** fullscreen is activated  
**Then** the editor covers the entire viewport using `position: fixed; inset: 0; z-index: 1000`

#### Edge Cases
- Fullscreen mode may conflict with Mendix modal dialogs that also use high z-index values — the editor will appear above any Mendix UI elements already open.

## Functional Requirements

- FR-001: The widget MUST bind one writable Mendix string attribute for the rich text content.
- FR-002: Content writes MUST be debounced at 200 ms — the attribute MUST NOT be written on every keystroke.
- FR-003: External attribute value updates MUST be applied only when the editor is NOT focused, using `updateContents` (not `setContents`) for incremental updates.
- FR-004: Clicking toolbar buttons, opening link/image/video dialogs, and using keyboard shortcuts that transfer focus within the editor MUST NOT trigger `onBlur` or `onChange` (on-leave) actions. The `relatedTarget` DOM check on blur MUST determine whether the focus target is inside the editor container.
- FR-005: When `onChangeType === "onLeave"`, `onChange` MUST fire only when the editor genuinely loses focus AND content has changed since the last focus entry.
- FR-006: Toolbar preset levels MUST be additive (basic ⊂ standard ⊂ full). Each button MUST have a `presetValue` (1/2/3) that is the minimum preset required to display it.
- FR-007: Toolbar position `"auto"` MUST use an `IntersectionObserver` sentinel to apply `"container-stuck"` CSS class and dock the toolbar via `position: sticky; top: 0`.
- FR-008: Ordered list indentation MUST cascade styles: level 0 → numeric, level 1 → lower-alpha, level 2 → lower-roman.
- FR-009: Default image upload MUST convert files to base64 data URLs and embed inline. Entity image upload MUST delegate to the configured Mendix upload action and store images as `data-src` attributes containing entity GUIDs.
- FR-010: Video URLs for YouTube, Vimeo, DailyMotion, and Google Maps MUST be auto-detected and auto-dimensioned.
- FR-011: Link URLs without a protocol prefix MUST have `https://` prepended automatically when saved.
- FR-012: Link URL validation MUST be optional; the `validateURL` flag controls whether URL pattern validation is applied.
- FR-013: The `softbreak` format MUST be registered to enable Shift+Enter → `<br>` within a paragraph.
- FR-014: Alt+F10 MUST move keyboard focus to the toolbar; Alt+F11 MUST move focus to the status bar.
- FR-015: The status bar MUST support three content types: word count, character count (text-only, HTML stripped), and character count (HTML-inclusive).
- FR-016: Fullscreen mode MUST use `position: fixed; inset: 0; z-index: 1000; background: white`.
- FR-017: Custom fonts MUST be provided as name/CSS-declaration pairs; they MUST be merged with the default font list alphabetically.
- FR-018: All Quill modules registered via `Quill.register()` MUST be treated as global — they affect ALL Quill instances on the page, not only the widget instance.
- FR-019: Validation alerts for the string attribute MUST be rendered below the editor.
- FR-020: The widget is NOT offline capable.

## Props Reference

| Name | Type | Default | Caption | Description |
|------|------|---------|---------|-------------|
| `stringAttribute` | `EditableValue<string>` | — | Value | Required writable string attribute containing rich text HTML. |
| `imageDataSourceContext` | `ListValue?` | — | Image datasource | Entity list for image gallery in image dialog. Enables entity-based image management. |
| `defaultImageUploadAction` | `ActionValue?` | — | Default image upload action | Mendix action for entity-based image upload. Required for entity upload mode. |
| `toolbarPreset` | `"basic" \| "standard" \| "full" \| "custom"` | `"basic"` | Toolbar preset | Controls which toolbar buttons are visible by preset level. |
| `toolbarPosition` | `"auto" \| "top" \| "bottom" \| "hide"` | `"auto"` | Toolbar position | `"auto"` enables sticky toolbar; `"hide"` removes it entirely. |
| `readOnlyStyle` | `"text" \| "bordered" \| "panel"` | `"bordered"` | Read-only style | Visual style when editor is in read-only mode. |
| `enableStatusBar` | `boolean` | `true` | Enable status bar | Shows/hides the status bar below the editor. |
| `statusBarType` | `"wordCount" \| "characterText" \| "characterHtml"` | `"wordCount"` | Status bar content | Content type displayed in the status bar. |
| `height` | `number?` | — | Height | Editor content area height value. |
| `heightUnit` | `"px" \| "percentage" \| "vh"` | `"px"` | Height unit | Unit for editor height. |
| `minHeight` | `number?` | — | Minimum height | Minimum height for the editor content area. |
| `maxHeight` | `number?` | — | Maximum height | Maximum height; content scrolls when exceeded. |
| `onFocus` | `ActionValue?` | — | On focus | Action fired when the editor first receives focus (not on internal transfers). |
| `onBlur` | `ActionValue?` | — | On blur | Action fired when the editor genuinely loses focus. |
| `onChange` | `ActionValue?` | — | On change | Action fired after content changes, according to `onChangeType`. |
| `onChangeType` | `"always" \| "onLeave"` | `"always"` | On change type | `"always"` fires on every change (debounced 200 ms); `"onLeave"` fires only on focus loss with changed content. |
| `onLoad` | `ActionValue?` | — | On load | Action fired when the editor has completed initialization. |
| `customFonts` | `{name: string; css: string}[]?` | — | Custom fonts | Additional fonts as name/CSS-declaration pairs, merged alphabetically with defaults. |
| `validateURL` | `boolean` | `true` | Validate URLs | Enables URL pattern validation for link insertion. |

## Changelog

**v4.12.0:** Link URL validation made optional via `validateURL` prop.

**v4.11.x:** Fixed list style preservation (lower-alpha/lower-roman on indented ordered lists) and limited empty line proliferation.

**v4.10.x:** Added character count variants (text-only, HTML-inclusive) to the status bar.

**v4.7.0:** Added custom font support (name + CSS declaration pairs).

**v4.6.1:** Added table support via `TABLE-BETTER` module.

**v4.5.0:** Added fullscreen mode.

**v4.4.0:** Added image and video resizing via drag handles.

**v4.0.0:** Major breaking change — replaced TinyMCE with Quill V2. Eliminated iframe-based rendering; editor now renders directly in the document DOM. Resolves CSP issues, styling isolation problems, and accessibility challenges present in the TinyMCE version.

## Open Questions
> Could not be determined from source code alone — requires human review
- [ ] What is the exact HTML schema saved to the attribute for entity images (is it `<img data-src="{GUID}">` or another structure)?
- [ ] Is there a built-in mechanism in the widget to resolve entity GUIDs back to image URLs at display time, or is this responsibility delegated to the page/developer?
- [ ] Does the 200 ms `onChange` debounce apply to `onChangeType === "always"` only, or is the `onLeave` path also debounced before the focus-loss check?
- [ ] Are there any documented Content Security Policy restrictions that affect the CodeMirror-based `viewCode` dialog?
- [ ] What Mendix runtime version is required for the Dojo incubator `MutationObserver` workaround to be triggered?
