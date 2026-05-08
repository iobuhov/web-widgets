# BarcodeGenerator

## Purpose

The BarcodeGenerator widget renders barcodes and QR codes in a Mendix web application from a dynamically bound string value. It supports three top-level format families — CODE128 (default linear barcode), QRCode, and Custom (10 specific barcode sub-formats) — and offers optional download, EAN addon support, QR image overlay, card presentation, and configurable error handling. The widget is intended for display contexts such as product pages, order confirmations, and asset tracking where machine-readable codes must be rendered alongside application data.

## User Scenarios

### [P1] Generate and display a linear barcode from a dynamic attribute

**Given** the widget is placed on a page with `codeFormat = "CODE128"` and `codeValue` bound to an entity attribute containing "ABC-12345"  
**When** the page loads and the attribute value is available  
**Then** a CODE128 barcode SVG is rendered with the encoded value; when the attribute value changes, the barcode updates automatically

#### Edge Cases

- When `codeValue` is empty or the attribute is in loading/unavailable state, the widget renders a `<span>` with the configured empty message (default: "No barcode value provided") instead of a barcode.
- When the value fails format-specific validation (e.g. non-numeric input for EAN-13), the widget renders nothing or shows an error message depending on `logLevel` configuration.
- When `logLevel = "None"`, rendering errors are completely silent — no UI indicator and no console output.

---

### [P2] Generate a QR code with an embedded image overlay

**Given** `codeFormat = "QRCode"`, `codeValue` is bound, `qrOverlay = true`, and `qrOverlaySrc` points to an available image  
**When** the page renders  
**Then** the QR code SVG includes the overlay image at the configured position and size; when `qrOverlayCenter = true`, the image is centered regardless of X/Y values

#### Edge Cases

- When `qrOverlaySrc` status is not `"available"` at render time, the overlay is omitted — no error, no placeholder.
- When `qrOverlayCenter = true`, X and Y position values are ignored even if set to non-zero.
- X=0 and Y=0 values are treated as "undefined" position by the config layer when `qrOverlayCenter` logic is active.
- Dynamic (entity-based) overlay images are not previewable in Studio Pro — they show a grey placeholder in the design canvas.
- Downloaded PNGs embed overlay images as base64; if the image fetch fails (CORS or network error), the download may reference an unresolvable external URL.

---

### [P3] Download the rendered barcode or QR code as a PNG

**Given** `allowDownload = true` and a barcode or QR code is rendered  
**When** the user clicks the download button  
**Then** the browser downloads a PNG file at 2× resolution with a white background; if `downloadFileName` is set, the file is named accordingly with a `.png` extension enforced

#### Edge Cases

- The download format is always PNG — SVG download is not supported.
- If no custom filename is configured, the file name is auto-generated as `{type}_{hash}_{timestamp}.png` using browser-local time.
- Custom filenames without a `.png` extension have it appended automatically.
- Download relies on browser canvas and DOM; it is not supported in non-DOM environments.

---

### [P4] Configure a specific barcode sub-format via Custom mode

**Given** `codeFormat = "Custom"` and `customCodeFormat` is set to a specific format (e.g. `"EAN-13"`)  
**When** the page renders  
**Then** the barcode is encoded using the selected format's rules; format-specific advanced options (e.g. `enableFlat` for EAN-13, `enableMod43` for CODE39) become configurable

#### Edge Cases

- `enableFlat` is only available for EAN-13 and EAN-8 in Custom mode, and only when no EAN addon is configured.
- `lastChar` is only available for Custom EAN-13 without flat or addon settings.
- `enableMod43` is only available for CODE39.
- EAN addons (EAN-2, EAN-5) are only available for EAN-13, EAN-8, and UPC formats.

---

## Functional Requirements

- FR-001: The system MUST render a barcode SVG when `codeValue` is a non-empty, valid string for the configured format.
- FR-002: The system MUST render a plain `<span>` with the empty message text when `codeValue` is empty or unavailable — no barcode SVG is rendered.
- FR-003: The system MUST display an error alert with `role="alert"` when rendering fails AND `logLevel` is `"Info"` or `"Debug"`.
- FR-004: The system MUST suppress all error output (UI and console) when `logLevel = "None"`.
- FR-005: The system MUST log rendering errors to `console.error` only when `logLevel = "Debug"`.
- FR-006: The system MUST support CODE128 (default), QRCode, and Custom format families; the Custom family MUST support 10 sub-formats: EAN-13, EAN-8, UPC, ITF-14, CODE39, MSI, Pharmacode, Codabar, CODE93, CODE128.
- FR-007: The system MUST validate barcode values against format-specific character set and length constraints at runtime; validation failure MUST produce the same error state as a library rendering exception.
- FR-008: The system MUST support EAN-2 and EAN-5 addons for EAN-13, EAN-8, and UPC formats; the addon MUST be rendered in the same SVG element as the main barcode.
- FR-009: The system MUST render a download button when `allowDownload = true`, positioned above or below the barcode/QR code per `buttonPosition`.
- FR-010: Downloads MUST produce PNG files at 2× scale with a white background; the download file extension MUST always be `.png`.
- FR-011: The system MUST support QR image overlay with configurable position, size, opacity, and excavation via the `qrOverlay` property group.
- FR-012: The widget MUST require an entity context (`needsEntityContext = true`) for data binding.
- FR-013: The widget MUST be usable in offline-enabled Mendix applications (`offlineCapable = true`).
- FR-014: The widget MUST apply a card visual treatment (border, background, padding via Mendix theme variables) when `showAsCard = true`.
- FR-015: Studio Pro design-time validation MUST block publish when `codeWidth < 1`, `codeHeight < 20`, or `qrSize < 50`.
- FR-016: The system MUST accept EAN-13 input as either 12 digits (data only) or 13 digits (data + checksum); the barcode library handles checksum calculation.

## Props Reference

| Name | Type | Default | Caption | Description |
|------|------|---------|---------|-------------|
| `codeValue` | textTemplate | _(none)_ | Value | The data to encode. Supports dynamic expressions. Widget renders empty state when unavailable. |
| `codeFormat` | enum | `"CODE128"` | Format | Top-level format family: `CODE128`, `QRCode`, or `Custom`. |
| `customCodeFormat` | enum | — | Custom format | Sub-format when `codeFormat = "Custom"`. One of: EAN-13, EAN-8, UPC, ITF-14, CODE39, MSI, Pharmacode, Codabar, CODE93, CODE128. Visible only in Custom mode. |
| `emptyMessage` | textTemplate | "No barcode value provided" | Empty message | Text shown when `codeValue` is empty or unavailable. |
| `showAsCard` | boolean | false | Show as card | Wraps the widget in a card container styled with Mendix theme variables. |
| `allowDownload` | boolean | false | Allow download | Renders a download button. Enables download sub-properties. |
| `downloadCaption` | textTemplate | — | Download button caption | Label text on the download button. |
| `downloadAriaLabel` | textTemplate | — | Download button aria-label | Accessible label for the download button. |
| `downloadFileName` | textTemplate | — | File name | Custom base name for the downloaded PNG. `.png` extension is enforced. Auto-generated when empty. |
| `buttonPosition` | enum | — | Button position | Positions download button `"top"` or `"bottom"` relative to the barcode/QR code. |
| `codeWidth` | integer | — | Barcode width | Bar width multiplier for linear barcodes. Minimum: 1. |
| `codeHeight` | integer | — | Barcode height | Height in pixels of linear barcodes. Minimum: 20. |
| `codeMargin` | integer | — | Barcode margin | Margin in pixels around linear barcodes. |
| `displayValue` | boolean | — | Display value | Show human-readable text below linear barcodes. |
| `enableFlat` | boolean | — | Enable flat | Removes bar separators on EAN-13/EAN-8 (Custom mode only, no addons). |
| `lastChar` | integer | — | Last character | Sets the last character for Custom EAN-13 (no flat, no addons). |
| `enableMod43` | boolean | — | Enable mod43 | Enables modulo-43 checksum for CODE39. |
| `enableEan128` | boolean | — | Enable EAN-128 | Enables EAN-128 mode for CODE128. |
| `addonFormat` | enum | `"None"` | Addon format | EAN-2 or EAN-5 addon for EAN-13, EAN-8, UPC. |
| `addonValue` | textTemplate | — | Addon value | Value for the EAN addon. EAN-5 requires exactly 5 numeric digits; EAN-2 requires exactly 2. |
| `addonSpacing` | integer | 20 | Addon spacing | Pixel gap between main barcode and addon barcode. |
| `qrSize` | integer | 128 | QR code size | QR code canvas size in pixels. Minimum: 50. Clamped to 200px in Studio Pro preview. |
| `qrMargin` | integer | — | QR code margin | Margin around the QR code in QR module units (grid cells), not pixels. |
| `qrLevel` | enum | — | Error correction level | QR error correction: `"L"`, `"M"`, `"Q"`, or `"H"`. |
| `showTitle` | boolean | — | Show title | Renders an `<h3>` title element above the QR code. |
| `qrTitle` | textTemplate | — | QR code title | Title text. Rendered as `<h3>` when `showTitle = true`. |
| `logLevel` | enum | — | Log level | `"None"` (silent), `"Info"` (UI error alert), `"Debug"` (UI + console.error). |
| `qrOverlay` | boolean | false | Show overlay | Enables image overlay on QR code. Reveals overlay sub-properties. |
| `qrOverlaySrc` | DynamicValue\<WebImage\> | — | Overlay image | Image to overlay on the QR code. Rendered only when status is `"available"`. |
| `qrOverlayCenter` | boolean | — | Center overlay | Centers the overlay image; hides X/Y position controls. |
| `qrOverlayX` | decimal | — | Overlay X position | Horizontal offset when not centered. Value 0 is treated as undefined. |
| `qrOverlayY` | decimal | — | Overlay Y position | Vertical offset when not centered. Value 0 is treated as undefined. |
| `qrOverlayWidth` | decimal | — | Overlay width | Overlay image width in pixels. |
| `qrOverlayHeight` | decimal | — | Overlay height | Overlay image height in pixels. |
| `qrOverlayOpacity` | decimal | — | Overlay opacity | Overlay image opacity (0–1, arbitrary-precision decimal). |
| `qrOverlayExcavate` | boolean | — | Excavate background | Clears the QR modules behind the overlay image (handled by qrcode.react library). |

## Format Validation Constraints

| Format | Input Constraint |
|--------|-----------------|
| EAN-13 | 12 or 13 numeric digits (checksum optional) |
| EAN-8 | 7 or 8 numeric digits (checksum optional) |
| UPC | 11 or 12 numeric digits |
| ITF-14 | Exactly 14 numeric digits |
| CODE39 | Uppercase A–Z, 0–9, `- . $ / + %`, space; max 43 characters |
| MSI | Numeric only; max 30 digits |
| Pharmacode | Numeric only; max 7 digits |
| Codabar | Digits 0–9, A–D (start/stop), `- $ : / . +`; max 20 characters |
| CODE93 | No control characters; max 47 characters |
| CODE128 | No control characters; max 80 characters recommended |
| QRCode | Any string; max 1200 characters recommended |

## Changelog

### [1.0.0] - 2026-04-17
- Initial release: QR code and multi-format barcode generation from dynamic string values; configurable QR properties; download functionality; comprehensive barcode type configuration and styling.

## Open Questions

> Could not be determined from source code alone — requires human review

- [ ] Is the E2E test suite intentionally left as a placeholder (`e2e/BarcodeGenerator.spec.js` has no barcode-specific assertions)? What is the testing strategy for runtime barcode rendering?
- [ ] Is QR overlay excavation behavior (pixel removal) guaranteed to match across QR error correction levels when the overlay covers a significant portion of the QR code?
- [ ] Can `qrOverlayX = 0, qrOverlayY = 0` be configured for an explicit top-left position? The current config layer treats both zeros as "undefined" (equivalent to "no position"), which makes top-left positioning impossible when `qrOverlayCenter` logic is active.
- [ ] What is the `logLevel` default value? The XML declares three options but the default is not explicitly captured in the draft findings.
- [ ] Are there plans to add Mendix modern React client compatibility? No MODERN_CLIENT skip flags appear in e2e tests, but formal compatibility testing evidence is absent.
