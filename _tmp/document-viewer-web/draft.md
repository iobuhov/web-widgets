# Draft: document-viewer-web

Widget: `@mendix/document-viewer-web` v1.2.0  
Task: EX-024  
Author: worker  
Date: 2026-05-09

---

## src/DocumentViewer.tsx

**Purpose:** Root widget component exported as the Mendix pluggable widget entry point. Composes dimension styling with dynamic renderer selection.

**Logic:** Calls `useRendererSelector` to get the current renderer component and its props, then calls `constructWrapperStyle` to build inline CSS dimensions. Renders a `div.widget-document-viewer.form-control` wrapper with the selected renderer inside.

**Behavior documentable:** The widget applies Mendix `class`, SCSS `widget-document-viewer`, and `form-control` CSS classes to the outer container. The renderer is rendered only if `CurrentRenderer` is truthy.

**User-facing:** Yes — this is the top-level rendered element visible to the end user.

**New learned:** The widget starts with `ErrorViewer` as the default component (set in `useRendererSelector`) and dynamically swaps to the correct renderer after the HEAD fetch resolves. The outer div carries `form-control` making it visually consistent with Mendix form input styling.

---

## src/DocumentViewer.xml

**Purpose:** Mendix widget definition file that declares the widget's identity, supported properties, and Studio Pro metadata.

**Logic:** Declares widget ID `com.mendix.widget.web.documentviewer.DocumentViewer`, marks it as offline-capable and a plugin widget. Defines four property groups: Data source (file), Dimensions (widthUnit, width, heightUnit, height, minHeightUnit, minHeight, maxHeightUnit, maxHeight, overflowY), Advanced (pdfjsWorkerUrl), and Common (Name, TabIndex, Visibility system properties).

**Behavior documentable:** The `file` property is of type `file` and is required. `pdfjsWorkerUrl` is an optional `textTemplate`, allowing dynamic expressions. Width defaults to 100, height to 250, minHeight to 250, maxHeight to 250. Widget appears in the "Display" category in both Studio Pro and Studio.

**User-facing:** Indirectly — defines the configuration interface presented to developers in Studio Pro.

**New learned:** The widget is marked `offlineCapable="true"`, meaning it can function without a server connection. The `pdfjsWorkerUrl` description explicitly mentions self-hosting as an option.

---

## typings/DocumentViewerProps.d.ts

**Purpose:** Auto-generated TypeScript type definitions derived from `DocumentViewer.xml`. Defines all prop types for the container and preview components.

**Logic:** Exports enum types (`WidthUnitEnum`, `HeightUnitEnum`, `MinHeightUnitEnum`, `MaxHeightUnitEnum`, `OverflowYEnum`) and two interfaces: `DocumentViewerContainerProps` (runtime) and `DocumentViewerPreviewProps` (Studio Pro design-time). `file` is typed as `DynamicValue<FileValue>` from `mendix`; `pdfjsWorkerUrl` as `DynamicValue<string> | undefined`.

**Behavior documentable:** `pdfjsWorkerUrl` is optional (`?`) at the container level, matching its `required="false"` in XML. Preview props use nullable numbers (`number | null`) for dimension values. The `className` prop in preview is deprecated since Mendix 9.18.0.

**User-facing:** No — internal typing only.

**New learned:** The preview interface includes `renderMode: "design" | "xray" | "structure"` and `translate` function, confirming Studio Pro design-mode rendering support.

---

## src/utils/useRendererSelector.ts

**Purpose:** Core hook that determines which renderer component to use based on the document's HTTP content-type and file extension.

**Logic:** Issues a HEAD fetch to `file.value.uri` using an AbortController. Parses the `content-type` response header (splits on `;` to strip charset). Iterates `DocumentRenderers` array, matching via exact string or regex. If multiple renderers match, uses the file extension (`file.value.name.split(".").pop()`) to disambiguate. Sets the matched renderer component or shows error for unsupported types. A second `useEffect` watches `documentStatus` to fall back to `ErrorViewer` on error.

**Behavior documentable:** Renderer selection is content-type-driven, not extension-driven, with extension as tiebreaker. If `file.status` is not "available" or URI is missing, no fetch is attempted and the component stays as `ErrorViewer`. AbortController cleanup cancels in-flight requests on unmount or file change. The hook re-runs when `file`, `file.status`, or `file.value.uri` changes.

**User-facing:** No — internal hook.

**New learned:** Renderer matching supports regex patterns (`contentType.match(type)`), enabling the `image/*` wildcard used by `ImageViewer`. The initial state is `DocumentStatus.loading` and `ErrorViewer` — the component is visible in a "loading/error" state until the HEAD response completes.

---

## src/utils/dimension.ts

**Purpose:** Constructs the inline `CSSProperties` style object for the widget wrapper based on dimension configuration props.

**Logic:** Maps `widthUnit` to `fit-content`, `{n}px`, or `{n}%`. For `heightUnit === "percentageOfWidth"` (Auto mode), sets `height: "auto"` and conditionally applies `minHeight` and `maxHeight` (with `overflowY` tied to maxHeight). For other height units, sets a fixed `height` value using `getHeightScale`. Height units map: `pixels` → `px`, `percentageOfView` → `vh`, `percentageOfParent` → `%`.

**Behavior documentable:** `overflowY` is only applied to the style when `heightUnit` is "percentageOfWidth" AND `maxHeightUnit !== "none"`. In fixed-height modes, overflow is not set. `minHeight` is only applied in auto-height mode when `minHeightUnit !== "none"`.

**User-facing:** No — internal utility.

**New learned:** The `percentageOfWidth` height unit is labeled "Auto" in the UI — it produces `height: auto` with optional min/max constraints. This is the only mode that supports overflow control.

---

## src/utils/helpers.ts

**Purpose:** Provides a `downloadFile` utility that opens the document in a new browser tab/window with a Mendix-specific query parameter.

**Logic:** Constructs a URL object from the file URI, appends `?target=window`, then calls `window.open(url, "mendix_file")`. The named window target `"mendix_file"` reuses the same tab across multiple download calls.

**Behavior documentable:** Appending `target=window` is a Mendix platform convention to trigger file download/viewing rather than inline navigation. If `fileUrl` is falsy, the function returns early without action.

**User-facing:** Yes — triggers file download visible to the user.

**New learned:** The download uses a named window target `"mendix_file"`, meaning repeated clicks will reuse the same browser tab rather than opening new ones each time.

---

## src/utils/useZoomScale.ts

**Purpose:** Manages zoom state with in/out/reset controls, enforcing minimum and maximum zoom bounds.

**Logic:** Maintains `zoomLevel` state starting at `1`. `zoomIn` multiplies by `1.2` capped at `10`; `zoomOut` multiplies by `0.8` floored at `0.3`; `reset` sets back to `1`. Returns `{ zoomLevel, zoomIn, zoomOut, reset }`.

**Behavior documentable:** Zoom range is `0.3x` to `10x`. Each zoom step is multiplicative (1.2× in, 0.8× out), so it takes approximately 14 clicks to go from 1x to 10x. Zoom is applied via CSS custom property `--current-zoom-scale` on a wrapper div, using `transform: scale()`.

**User-facing:** Yes — zoom level directly affects document display scale.

**New learned:** The reset function sets zoom to exactly `1` regardless of current level. The zoom is applied via CSS `transform: scale()` starting from top-left (`transform-origin: 0 0`), so zoomed content expands right and downward within the scrollable content area.

---

## src/components/documentRenderer.ts

**Purpose:** Defines the TypeScript contract (interfaces and enums) that all renderer components must satisfy.

**Logic:** Declares `DocumentStatus` const enum (`available`, `error`, `loading`). `DocumentStatusEvent` holds status + optional message. `DocumentRendererProps` extends `DocumentViewerContainerProps` with `setDocumentStatus` and `documentStatus`. `DocRendererElement` is a `FC<DocumentRendererProps>` with two static arrays: `contentTypes: string[]` and `fileTypes: string[]`.

**Behavior documentable:** All renderer components implement `DocRendererElement` — a React functional component with additional static properties. Renderers communicate errors back to the parent via `setDocumentStatus`, which triggers fallback to `ErrorViewer`.

**User-facing:** No — internal contract only.

**New learned:** The `contentTypes` array on `DocRendererElement` enables the renderer registry pattern in `useRendererSelector`, where a single array is iterated to find matching renderers. This makes adding new format support as simple as implementing the interface and adding to `index.ts`.

---

## src/components/BaseViewer.tsx

**Purpose:** Provides the shared toolbar shell (filename display + custom controls area + content area) used by all renderers. Exports both a minimal `BaseViewer` and a `BaseControlViewer` with built-in zoom and download controls.

**Logic:** `BaseViewer` renders a two-row layout: a controls bar (filename left, custom controls right) and a content area. `BaseControlViewer` wraps `BaseViewer`, adds zoom buttons (out/in/fit-to-width) and a download button to `CustomControl`, and wraps children in a `zoom-container` div with the `--current-zoom-scale` CSS custom property.

**Behavior documentable:** The FitToWidth reset button is incorrectly disabled when `zoomLevel >= 10` (should be disabled at `zoomLevel === 1` when already at default). Download calls `downloadFile(file.value?.uri)`. Zoom buttons are disabled at bounds (0.3 min, 10 max). The zoom container uses `transform-origin: 0 0` so zoomed content grows right/downward.

**User-facing:** Yes — toolbar and content container are directly visible.

**New learned:** The `BaseControlViewer` accepts an optional `CustomControl` prop that is rendered before the download and zoom buttons, allowing PDFViewer to inject its pagination control into the toolbar.

---

## src/components/PDFViewer.tsx

**Purpose:** Renders PDF documents with page navigation, zoom controls, and download. Uses `react-pdf` (backed by `pdfjs-dist`) for rendering.

**Logic:** Worker URL is resolved from `pdfjsWorkerUrl` prop: if available and non-empty uses that value; if available but empty falls back to unpkg CDN; if "unavailable" sets error status. Tracks `currentPage`, `numberOfPages`, `zoomLevel`, `pageInputValue`, and `pdfUrl`. Page input accepts only numeric characters and validates on blur/Enter/submit. File change resets page to 1. Renders only when `pdfUrl` is set AND `pdfjsWorkerUrl.status === "available"` (via `If` component).

**Behavior documentable:** The widget will NOT render the PDF until `pdfjsWorkerUrl` status is "available" — even if a valid URL is provided later. If `pdfjsWorkerUrl` status is "unavailable", an error is shown immediately. Default worker is hosted on unpkg CDN. CMap files are served from `/widgets/com/mendix/shared/pdfjs/cmaps/`, standard fonts from `/widgets/com/mendix/shared/pdfjs/standard_fonts/`. Page input resets to current page on invalid input.

**User-facing:** Yes — full PDF rendering with pagination and zoom.

**New learned:** The PDF rendering is gated on `pdfjsWorkerUrl.status === "available"` using the `If` component from `widget-plugin-component-kit`. This means if the worker URL expression evaluates to "unavailable" (e.g., a null attribute), no PDF renders and an error message is shown.

---

## src/components/DocxViewer.tsx

**Purpose:** Renders DOCX (Word) documents inline using the `docx-preview` library.

**Logic:** Fetches the document binary via GET and passes the `ArrayBuffer` to `parseAsync` then `renderDocument`. Renders into a `ref`-attached `div.docx-viewer-container`. Configuration: `className: "docx-viewer-content"`, `ignoreWidth: true`, `ignoreLastRenderedPageBreak: false`, `inWrapper: false`. Uses `BaseControlViewer` for toolbar with zoom/download.

**Behavior documentable:** Document width is ignored (`ignoreWidth: true`), so DOCX renders at full container width regardless of page width specified in the document. Page breaks from the original document are preserved (`ignoreLastRenderedPageBreak: false`). Errors at parse or render time set `DocumentStatus.error`.

**User-facing:** Yes — rendered DOCX content is directly visible.

**New learned:** The DOCX renderer uses a dummy `styleContainer` div (not attached to the DOM) to prevent docx-preview from injecting global styles into the page, limiting style bleed into the Mendix application shell.

---

## src/components/ExcelViewer.tsx

**Purpose:** Renders XLSX (Excel) spreadsheets as HTML tables using the `xlsx` library.

**Logic:** Fetches the file binary via GET, passes to `xlsx.read()`, accesses the first sheet (`wb.SheetNames[0]`), converts to HTML via `utils.sheet_to_html()`, and sets the HTML string as component state rendered via `dangerouslySetInnerHTML`.

**Behavior documentable:** Only the FIRST sheet of a workbook is rendered — multi-sheet workbooks show only sheet index 0. The rendered content is an HTML table with no additional interactivity. Uses `BaseControlViewer` for zoom/download toolbar.

**User-facing:** Yes — rendered spreadsheet table is directly visible.

**New learned:** `dangerouslySetInnerHTML` is used for the rendered HTML table. This is safe because the HTML is generated by the xlsx library from binary data, not from user text input.

---

## src/components/ImageViewer.tsx

**Purpose:** Renders image files inline using a native `<img>` element.

**Logic:** Directly renders `<img src={file.value?.uri} alt="Image" className="image-viewer-content">` inside `BaseControlViewer`. No fetch needed — the browser loads the image natively. Supports `image/*` content type via regex match.

**Behavior documentable:** Supported file extensions: jpg, jpe, jpeg, png, gif, bmp, tif, tiff, webp. Alt text is fixed as "Image" (not configurable). Uses zoom/download toolbar from `BaseControlViewer`.

**User-facing:** Yes — image is directly visible.

**New learned:** ImageViewer requires no async loading logic because the browser's native image loading handles it. Unlike other viewers, it does not call `setDocumentStatus` for errors — broken images show the browser's default broken image icon.

---

## src/components/TextViewer.tsx

**Purpose:** Renders plain text, CSV, JSON, and other text-based formats inside a `<pre>` element.

**Logic:** Fetches file content via GET as text, stores in state, renders in `<pre className="text-content">`. Error handling sets `DocumentStatus.error`. Uses `BaseControlViewer` for toolbar.

**Behavior documentable:** Content is displayed as preformatted text (monospace, preserving whitespace). Supported MIME types: `text/plain`, `text/csv`, `application/json`. Supported extensions: txt, csv, json, text, log, xml, html, htm, css, js, jsx, ts, tsx, svg. No syntax highlighting.

**User-facing:** Yes — raw text content is directly visible.

**New learned:** Despite supporting `text/csv` and structured formats like JSON and XML, content is shown as raw preformatted text without parsing or formatting. HTML files match both TextViewer (`text/html` extension) and PDFViewer (`text/html` content type) — the extension tiebreaker in `useRendererSelector` would route `.html` files to TextViewer (html is in TextViewer.fileTypes) but NOT to PDFViewer (whose fileTypes is only `["pdf"]`).

---

## src/components/ErrorViewer.tsx

**Purpose:** Fallback renderer shown during loading, for unsupported file types, or when a specific renderer encounters an error.

**Logic:** Renders `BaseViewer` (not `BaseControlViewer` — no zoom) with only a Download button in the toolbar. Shows a loading skeleton animation while `file.status !== "available"`, or the error message string from `documentStatus.message` once available.

**Behavior documentable:** The loading skeleton is a CSS-animated gradient (1s linear alternate). There is no retry mechanism — errors are terminal until the file prop changes. The download button allows users to save the file even if it can't be displayed.

**User-facing:** Yes — shown during loading and on error.

**New learned:** ErrorViewer is the initial renderer before content-type detection completes. It renders the loading skeleton during this phase. Once `file.status` becomes "available" (but after an error), it switches to showing the error message text from `documentStatus.message`.

---

## src/components/index.ts

**Purpose:** Defines the ordered registry of renderer components used by `useRendererSelector`.

**Logic:** Exports `DocumentRenderers = [DocxViewer, ExcelViewer, PDFViewer, TextViewer, ImageViewer]`. Order matters for disambiguation when multiple renderers could match.

**Behavior documentable:** `application/octet-stream` is registered in both DocxViewer and ExcelViewer contentTypes. When a file is served as `application/octet-stream`, both could match; the file extension tiebreaker (docx vs. xlsx) resolves the correct renderer. DocxViewer appears before ExcelViewer in the array.

**User-facing:** No — internal configuration.

**New learned:** The registry order is significant: DocxViewer is first, so for ambiguous content types it is preferred before extension tiebreaking. This means a file with `application/octet-stream` and extension `.xlsx` will correctly fall to ExcelViewer via tiebreaking, while a `.docx` extension selects DocxViewer.

---

## src/DocumentViewer.editorConfig.ts

**Purpose:** Controls which properties are shown or hidden in the Studio Pro property panel based on the current dimension configuration.

**Logic:** Hides `width` when `widthUnit === "contentFit"`. Hides `height` when `heightUnit === "percentageOfWidth"` (Auto). Hides `minHeight/minHeightUnit/maxHeight/maxHeightUnit/overflowY` when height is NOT auto. Hides `minHeight` when `minHeightUnit === "none"`. Hides `maxHeight` and `overflowY` when `maxHeightUnit === "none"`.

**Behavior documentable:** Min/max height and overflow controls are only accessible when the widget is in auto-height mode. Entering a fixed pixel height mode removes the min/max/overflow UI entirely.

**User-facing:** No — developer/design-time configuration only.

**New learned:** The `getPreview` function uses the Studio Pro structure preview API to render a labeled box in design mode. The caption shows the configured file path or "No document selected" when no file is configured, supporting quick visual identification in Studio Pro layouts.

---

## src/DocumentViewer.editorPreview.tsx

**Purpose:** Renders a static preview of the widget in Studio Pro design view, showing the control bar and document name.

**Logic:** Calls `constructWrapperStyle` with preview props (using null-coalesced defaults). Renders the same outer `div.widget-document-viewer` wrapper as runtime, but uses `BaseControlViewer` with a synthetic file object `{ status: "available", value: { uri: file, name: file } }`.

**Behavior documentable:** In design preview, the file prop (a string path/name) is shown as both the URI and the document name in the toolbar. The zoom and download buttons are rendered but non-functional in design mode.

**User-facing:** No — Studio Pro design-time only.

**New learned:** The preview reuses `BaseControlViewer` directly from runtime components, meaning the design-time preview accurately reflects the toolbar layout with zoom and download buttons visible.

---

## src/ui/documentViewer.scss

**Purpose:** Main stylesheet defining the visual layout of the viewer shell, content area, pagination, zoom container, and loading skeleton.

**Logic:** Uses SCSS variables for skeleton animation colors. Defines `widget-document-viewer` block with controls bar (flex row, gray background, rounded top), content area (margin/padding/border/overflow:auto), pagination (flex row), zoom-container (CSS `transform: scale()` via custom property), loading skeleton (animated gradient). Defines page-input sizing (5ch width, centered text).

**Behavior documentable:** The content area has `overflow: auto` allowing internal scrolling when content exceeds the container. The zoom container uses `--current-zoom-scale` CSS custom property for scaling, with `transform-origin: 0 0`. The loading animation is a horizontal gradient sweep over 1s.

**User-facing:** Yes — directly affects visual appearance.

**New learned:** The page number input is sized to `5ch` (5 character widths), accommodating 5-digit page counts. The inner content has its own border, giving a "panel within panel" visual.

---

## src/ui/documentViewerIcons.scss

**Purpose:** Loads and registers the custom `DocViewer` icon font, and applies icon characters to button elements within the controls bar.

**Logic:** Declares `@font-face` for `DocViewer.woff2`. Maps icon names to Unicode PUA code points: Download (`\e903`), Right (`\e906`), Left (`\e902`), ZoomIn (`\e901`), ZoomOut (`\e900`), FitToWidth (`\e904`). Generates `.icon-{Name}:before { content: "{code}" }` rules.

**Behavior documentable:** Icon buttons use CSS class names like `icon-Download`, `icon-ZoomIn`, etc. — the icon character is injected via CSS `:before` pseudo-element. Button background is transparent; disabled state also transparent with `cursor: not-allowed`.

**User-facing:** Yes — visible toolbar icons.

**New learned:** All icons are rendered from a single custom woff2 font file (`DocViewer.woff2`) bundled with the widget, eliminating any external CDN dependency for icons.

---

## src/components/DocxViewer.scss

**Purpose:** Applies content padding to DOCX-rendered sections within the viewer.

**Logic:** Targets `.widget-document-viewer section.docx-viewer-content` and sets `padding: var(--spacing-largest, 48pt) !important`.

**Behavior documentable:** DOCX content sections get 48pt padding (matching typical document margins) using `!important` to override docx-preview default styles.

**User-facing:** Yes — affects visual margins of rendered DOCX content.

**New learned:** The `!important` is needed to override the docx-preview library's inline padding styles injected during rendering.

---

## e2e/DocumentViewer.spec.js

**Purpose:** E2E test file placeholder — currently empty.

**Logic:** The file contains a single blank line with no tests.

**Behavior documentable:** No e2e behavioral coverage exists for this widget.

**User-facing:** No — test infrastructure only.

**New learned:** No e2e-confirmed behaviors. All behavioral findings in this draft come from source code analysis alone.

---

## CHANGELOG.md

**Purpose:** Records version history and notable changes for the widget.

**Logic:** Follows Keep a Changelog format with Semantic Versioning.

**Behavior documentable:**
- **v1.0.0** (2025-05-05): Initial release with DOCX, PDF, XLSX, TXT, and Image viewing.
- **v1.1.0** (2025-09-11): PDF.js worker bundle moved from CDN to local build for CSP compliance. This is a significant behavioral change — the worker is no longer fetched from an external CDN by default.
- **v1.1.1** (2025-09-26): Added `pdfjsWorkerUrl` advanced property allowing users to specify a self-hosted PDF.js worker. Provides escape hatch from default unpkg CDN fallback.
- **v1.2.0** (2025-10-29): Added manual page number input to PDF viewer — users can type a page number directly instead of only using next/previous buttons.
- **[Unreleased]**: Internal structure change (non-user-facing).

**User-facing:** No — documentation only.

**New learned:** The v1.1.0 CSP compliance change was significant: the worker moved to local build, but v1.1.1 immediately added the ability to override this with a custom URL. The v1.2.0 page number input was a community contribution (`@mx-kshitij`).

---

## package.json

**Purpose:** Package metadata, dependency declarations, and build scripts for the widget.

**Logic:** Declares `@mendix/document-viewer-web` at v1.2.0. Key runtime dependencies: `docx-preview ^0.3.6`, `pdfjs-dist 4.8.69` (pinned), `react-pdf ^9.2.1`, `xlsx` (loaded from SheetJS CDN tarball). Local workspace packages: `@mendix/widget-plugin-component-kit` and `@mendix/widget-plugin-platform`.

**Behavior documentable:** `pdfjs-dist` version is pinned at `4.8.69` — no automatic upgrades. The `xlsx` package is sourced from `https://cdn.sheetjs.com/xlsx-0.20.3/xlsx-0.20.3.tgz` rather than npm registry. Marketplace minimum Mendix version: `9.24.0.2965`. Widget is marked `reactReady: true`.

**User-facing:** No — build configuration only.

**New learned:** The `xlsx` package is fetched from the SheetJS CDN rather than npm, likely due to licensing constraints (SheetJS uses a non-standard license outside npm). The widget mpk package name is `com.mendix.widget.web.DocumentViewer.mpk`.
