# DocumentViewer

## Purpose

The Document Viewer widget enables Mendix developers to render file attachments inline within a page, without requiring the user to download and open them externally. It automatically detects the document's format via an HTTP HEAD request and selects the appropriate renderer for PDF, DOCX, XLSX, image, and plain-text formats. The widget provides a unified toolbar shell with zoom and download controls across all supported formats, and a configurable dimension system supporting fixed, percentage, and auto-height modes. It is intended for use in pages where document preview is a primary user interaction, such as document management portals or record detail views.

## User Scenarios

### [P1] View a PDF document with page navigation
**Given** a Mendix file attribute is bound to the widget's `file` property and the file is a PDF  
**When** the page loads and the widget resolves the content-type via HEAD request  
**Then** the PDF is rendered inline with a toolbar showing: page back/forward buttons, a direct page number input, zoom in/out/fit-to-width buttons, and a download button

#### Edge Cases
- If `pdfjsWorkerUrl` status is "unavailable" (e.g., bound to a null attribute), the ErrorViewer is shown immediately with no PDF rendered.
- If `pdfjsWorkerUrl` is not configured, the widget falls back to the unpkg CDN for the worker script.
- Entering a non-numeric value in the page number input resets it to the current page.
- Changing the bound file resets the current page to 1.

### [P2] View a DOCX or XLSX file inline
**Given** a Mendix file attribute is bound to the widget's `file` property and the file is a DOCX or XLSX  
**When** the content-type is resolved  
**Then** the document is rendered using the corresponding renderer (docx-preview or xlsx library) with zoom and download controls

#### Edge Cases
- DOCX content ignores the document's declared page width and renders at full container width (`ignoreWidth: true`).
- DOCX page breaks from the original document are preserved.
- For XLSX files, only the first sheet (index 0) is rendered; additional sheets are not accessible.
- Files served as `application/octet-stream` are disambiguated by file extension (`.docx` → DocxViewer, `.xlsx` → ExcelViewer).

### [P3] View an image or text-based file
**Given** a Mendix file attribute is bound to the widget's `file` property and the file is an image or text-based format  
**When** the content-type is resolved  
**Then** the file is rendered — images natively via `<img>`, text formats (plain text, CSV, JSON, XML, HTML, etc.) as preformatted content in a `<pre>` element

#### Edge Cases
- Supported image types: jpg, jpe, jpeg, png, gif, bmp, tif, tiff, webp — matched via the `image/*` content-type regex.
- Broken images show the browser's default broken image icon; no error state is set.
- Text content is displayed as raw preformatted text with no syntax highlighting or parsing, regardless of format (CSV, JSON, XML).
- `.html` files are routed to TextViewer (not treated as HTML documents) based on extension tiebreaking.

### [P4] Loading and error states
**Given** a file is bound to the widget  
**When** the file's `DynamicValue` status is not yet "available", or the HEAD request fails, or the content-type is unsupported  
**Then** the widget displays a loading skeleton animation or an error message, with a download button always available

#### Edge Cases
- The widget displays a CSS-animated loading skeleton until `file.status` becomes "available".
- Once available, if the content-type is unsupported, an error message is shown — there is no retry mechanism; the error persists until the file prop changes.
- The download button is always present in the error/loading state, allowing the user to retrieve the file even when it cannot be displayed.

### [P5] Download the displayed document
**Given** a document is rendered in the viewer  
**When** the user clicks the download button  
**Then** the file is opened in a named browser tab (`mendix_file`) using the Mendix platform's `?target=window` convention

#### Edge Cases
- Repeated clicks reuse the same browser tab rather than opening new ones.
- If the file URI is falsy, the download action is a no-op.

## Functional Requirements

- FR-001: The widget MUST issue an HTTP HEAD request to the file URI to determine content-type before rendering; renderer selection is content-type-driven with file extension as a tiebreaker when multiple renderers match.
- FR-002: The widget MUST support the following renderer types: PDF (via react-pdf/pdfjs-dist), DOCX (via docx-preview), XLSX (via xlsx), Images (via native `<img>`), and plain text/CSV/JSON/XML/HTML (via `<pre>`).
- FR-003: The widget MUST display a loading skeleton animation while `file.status` is not "available" or while the HEAD request is in flight.
- FR-004: The widget MUST fall back to the ErrorViewer (with download button) when the content-type is unsupported or a renderer encounters an error.
- FR-005: The widget MUST cancel in-flight HEAD requests via `AbortController` on unmount or when the `file` prop changes.
- FR-006: The widget MUST apply Mendix `form-control` and `widget-document-viewer` CSS classes to the outer container.
- FR-007: The widget MUST support zoom levels between 0.3× and 10× (multiplicative steps: 1.2× in, 0.8× out) applied via CSS `transform: scale()` with `transform-origin: 0 0`.
- FR-008: The PDF renderer MUST NOT render until `pdfjsWorkerUrl.status` is "available"; if status is "unavailable", the ErrorViewer MUST be shown.
- FR-009: The PDF renderer MUST fall back to the unpkg CDN worker URL if `pdfjsWorkerUrl` is configured but evaluates to an empty string.
- FR-010: The PDF renderer MUST support direct page number entry; invalid input MUST reset to the current page.
- FR-011: The XLSX renderer MUST render only the first sheet of a workbook.
- FR-012: The download action MUST open the file URI with `?target=window` appended in a named window (`mendix_file`).
- FR-013: Dimension constraints (`minHeight`, `maxHeight`, `overflowY`) MUST only be applied when `heightUnit` is "percentageOfWidth" (Auto mode); `overflowY` MUST only be set when `maxHeightUnit` is not "none".
- FR-014: The widget MUST support offline Mendix applications (`offlineCapable="true"`).

## Props Reference

| Name | Type | Default | Caption | Description |
|------|------|---------|---------|-------------|
| `file` | File | — | Document | The Mendix file document to display. Required. |
| `widthUnit` | Enum (`contentFit` \| `pixels` \| `percentage`) | `percentage` | Width unit | Determines how the widget width is expressed. `contentFit` hides the `width` input. |
| `width` | Integer | `100` | Width | Width value in the selected unit. Hidden when `widthUnit` is `contentFit`. |
| `heightUnit` | Enum (`pixels` \| `percentageOfView` \| `percentageOfParent` \| `percentageOfWidth`) | `pixels` | Height unit | Determines how the widget height is expressed. `percentageOfWidth` is the "Auto" mode and enables min/max/overflow controls. |
| `height` | Integer | `250` | Height | Height value in the selected unit. Hidden in Auto mode. |
| `minHeightUnit` | Enum (`none` \| `pixels` \| `percentageOfView` \| `percentageOfParent`) | `none` | Min height unit | Unit for minimum height. Only active in Auto mode. `none` disables min height. |
| `minHeight` | Integer | `250` | Min height | Minimum height value. Hidden when `minHeightUnit` is `none`. |
| `maxHeightUnit` | Enum (`none` \| `pixels` \| `percentageOfView` \| `percentageOfParent`) | `none` | Max height unit | Unit for maximum height. Only active in Auto mode. `none` disables max height and overflow. |
| `maxHeight` | Integer | `250` | Max height | Maximum height value. Hidden when `maxHeightUnit` is `none`. |
| `overflowY` | Enum (`scroll` \| `hidden` \| `auto`) | — | Overflow Y | Controls vertical overflow behavior. Applied only in Auto mode when `maxHeightUnit` is not `none`. |
| `pdfjsWorkerUrl` | Text template | — | PDF.js worker URL | URL for the PDF.js worker script. Optional. If empty, defaults to unpkg CDN. If bound to an unavailable expression, PDF rendering is blocked and an error is shown. |

## Changelog

- **v1.2.0 (2025-10-29):** Added manual page number input to PDF viewer — users can type a page number directly instead of using only next/previous buttons.
- **v1.1.1 (2025-09-26):** Added `pdfjsWorkerUrl` advanced property, allowing users to specify a self-hosted PDF.js worker URL as an escape hatch from the default unpkg CDN fallback.
- **v1.1.0 (2025-09-11):** PDF.js worker bundle moved from external CDN to local build for Content Security Policy (CSP) compliance.

## Open Questions

> Could not be determined from source code alone — requires human review

- [ ] The FitToWidth reset button in `BaseControlViewer` is disabled when `zoomLevel >= 10` rather than when `zoomLevel === 1` — is this a known bug scheduled for a fix?
- [ ] What is the expected behavior when `file.value.uri` changes while a HEAD request is already in flight? The `AbortController` cancels the old request, but the resulting state transition is not covered by tests.
- [ ] `ImageViewer` does not call `setDocumentStatus` on image load errors — broken images show the browser's default broken image icon with no widget-level error state. Is this intentional?
- [ ] CMap files are served from `/widgets/com/mendix/shared/pdfjs/cmaps/` and standard fonts from `/widgets/com/mendix/shared/pdfjs/standard_fonts/` — are these paths configurable or assumed to be present in all Mendix deployments?
