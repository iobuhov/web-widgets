# Draft: barcode-generator-web

Widget package: `@mendix/barcode-generator-web` v1.0.0  
Source: `packages/pluggableWidgets/barcode-generator-web/`

---

## src/BarcodeGenerator.tsx

**1. Purpose of this file?**
Root container component that is the registered Mendix pluggable widget entry point. It reads `codeValue` from props and decides whether to show an empty-state message or delegate rendering to the appropriate code renderer.

**2. What kind of logic is described in this file?**
Calls `barcodeConfig(props)` to produce a normalized config object. If `config.codeValue` is falsy (empty or unavailable), renders a `<span>` with `props.emptyMessage?.value` or the hardcoded fallback "No barcode value provided". Otherwise renders `<QRCodeRenderer>` when `config.type === "qrcode"` and `<BarcodeRenderer>` for all other formats. Applies `showAsCard` class modifier.

**3. What part of behavior can be documented from this file?**
When the dynamic expression bound to `codeValue` resolves to an empty string or is unavailable, the widget renders only the empty message — no SVG is rendered. The outer `<div>` always carries the base class `barcode-generator` plus the developer's custom `class`, and adds `barcode-generator--as-card` when `showAsCard` is true.

**4. Is it user-facing?**
Yes — the outer container and empty message are directly visible to end users.

**5. What new did you learn from this file?**
The QR/barcode routing is decided from `config.type`, which is set by `barcodeConfig()` based on the selected `codeFormat`. The widget itself never renders both types simultaneously — it is strictly one renderer per render.

---

## src/BarcodeGenerator.xml

**1. Purpose of this file?**
Widget descriptor for the Mendix pluggable widget framework. Declares widget identity, all configurable properties, groupings, and system properties.

**2. What kind of logic is described in this file?**
Declares `codeValue` (expression returning String, required), `codeFormat` (enumeration: CODE128 | QRCode | Custom, default CODE128), `emptyMessage` (textTemplate, optional), `allowDownload` (boolean), download-related properties (caption, aria-label, file name, button position), advanced barcode settings (customCodeFormat with 10 options, enableEan128, enableFlat, lastChar, enableMod43), EAN addon settings (addonFormat, addonValue, addonSpacing), advanced QR settings (qrLevel L/M/Q/H, qrSize), QR overlay settings (src, position, dimensions, opacity, excavate), display settings (displayValue, showAsCard, codeWidth, codeHeight, codeMargin, qrMargin, qrTitle, showTitle, qrOverlay), and logLevel.

**3. What part of behavior can be documented from this file?**
`needsEntityContext="true"` means the widget must be placed inside a data widget (data view, list view, etc.) — it requires an entity context to bind `codeValue`. `offlineCapable="true"` means the widget works in offline-enabled apps. Default `codeFormat` is CODE128 (barcode), not QR code. The `helpUrl` is commented out (not yet linked to docs). All download sub-properties are grouped under the main `allowDownload` boolean and are in the same property group.

**4. Is it user-facing?**
Not directly. Defines what developers configure in Studio/Studio Pro.

**5. What new did you learn from this file?**
Unlike the badge-button widget, this widget has `needsEntityContext="true"` — it cannot be placed outside a data context. The `codeFormat` enumeration uses three modes: `CODE128` (maps to the standard barcode path), `QRCode` (QR path), and `Custom` (exposes the `customCodeFormat` dropdown with 10 specific barcode types). This three-tier design allows simplified "barcode or QR" selection for most users while still supporting expert customization.

---

## typings/BarcodeGeneratorProps.d.ts

**1. Purpose of this file?**
Auto-generated TypeScript type definitions from `BarcodeGenerator.xml`. Provides compile-time types for runtime container props and Studio Pro design-time preview props.

**2. What kind of logic is described in this file?**
Defines `BarcodeGeneratorContainerProps`: `codeValue` as `DynamicValue<string>`, overlay image as `DynamicValue<WebImage>`, `qrOverlayOpacity` as `Big` (arbitrary-precision decimal). Defines enumerations for all enum props. `BarcodeGeneratorPreviewProps`: numeric props become `number | null` in preview (nullable), string expressions become plain `string`, and `qrOverlaySrc` is a discriminated union of static (imageUrl) or dynamic (entity) image sources.

**3. What part of behavior can be documented from this file?**
`codeValue` is `DynamicValue<string>` — it has loading/available/unavailable states. `qrOverlayOpacity` uses `Big` (big.js), so opacity values support arbitrary decimal precision at the type level. Preview props use `number | null` for all integer/decimal fields (nullable because Studio Pro may have blank fields), while runtime props use `number` (always provided by the Mendix runtime).

**4. Is it user-facing?**
No. Type declarations only.

**5. What new did you learn from this file?**
The `qrOverlaySrc` preview type is a discriminated union: `{ type: "static"; imageUrl: string }` or `{ type: "dynamic"; entity: string }`. This is the standard Mendix pattern for image properties in preview contexts — static images are inlined as URLs, dynamic images reference entity names. The runtime type is `DynamicValue<WebImage>` which resolves to `{ uri: string }` when available.

---

## src/config/Barcode.config.ts

**1. Purpose of this file?**
Factory function that normalizes raw Mendix container props into a typed, renderer-ready config object (`BarcodeConfig`). Acts as the single translation layer between Mendix platform types and the rendering components.

**2. What kind of logic is described in this file?**
`barcodeConfig()` reads `codeValue?.value` (defaulting to `""`), resolves the active format (if `codeFormat === "Custom"`, uses `customCodeFormat`; otherwise uses `codeFormat`), builds a `downloadButton` config only when `allowDownload` is true, then branches to produce either a `QRCodeTypeConfig` or a `BarcodeTypeConfig`. For QR, `qrTitle` is resolved via `status === "available"` check, falling back to `"QR Code"`. The overlay config is only set when `qrOverlaySrc` is available. `qrOverlayX/Y === 0` maps to `undefined` (used by qrcode.react to mean "use default/center").

**3. What part of behavior can be documented from this file?**
The `format` union (`CodeFormatEnum | CustomCodeFormatEnum`) collapses `codeFormat + customCodeFormat` into a single canonical format string. The `codeMargin` vs `qrMargin` are separate properties — the factory picks the right one based on format. Custom filename: if `downloadFileName` ends in `.png`, it is used as-is; otherwise `.png` is appended; if empty/blank, `undefined` is returned (letting the download function generate a timestamped name).

**4. Is it user-facing?**
No. Configuration logic only.

**5. What new did you learn from this file?**
`qrOverlayX === 0` and `qrOverlayY === 0` are both converted to `undefined` in the overlay config. This is a deliberate mapping: when position is zero (the default), the qrcode.react library interprets `undefined` as "use its own default" (typically centered), while a non-zero value positions the overlay at exact pixel coordinates. This means setting X=0, Y=0 with `qrOverlayCenter=false` still results in centered overlay behavior from the library's perspective.

---

## src/config/validation.ts

**1. Purpose of this file?**
Runtime and design-time validation functions for barcode values and EAN addon values. Each format has specific character set and length constraints.

**2. What kind of logic is described in this file?**
`validateBarcodeValue(format, value)` returns `{valid: true}` or `{valid: false, message}`. If `value` is empty, always valid (dynamic binding will provide at runtime). Per-format rules: EAN-13 (12 or 13 numeric digits), EAN-8 (7 or 8 numeric digits), UPC (11 or 12 numeric digits), ITF-14 (exactly 14 numeric digits), CODE39 (A-Z, digits, special chars, max 43), MSI (numeric, max 30), Pharmacode (numeric, max 7), Codabar (digits + A-D start/stop + specials, max 20), CODE93 (no control chars, max 47), QRCode (max 1200 chars), CODE128 (no control chars, max 80). `validateAddonValue()` checks EAN-5 (exactly 5 digits) and EAN-2 (exactly 2 digits).

**3. What part of behavior can be documented from this file?**
Validation runs both at design-time (in `editorConfig.ts` check function) and at runtime (in `useRenderBarcode` hook). Design-time validation only applies to static literal values in Studio Pro — dynamic attribute bindings get a warning hint instead of an error. Runtime validation sets the `error` state, which causes the renderer to show an "Unable to generate barcode" alert.

**4. Is it user-facing?**
Partially. Design-time errors appear in Studio Pro as configuration warnings. Runtime errors render an alert visible to end users (when `logLevel !== "None"`).

**5. What new did you learn from this file?**
EAN-13 allows 12 OR 13 digits: 12 is the data-only form and the library auto-computes the checksum digit; 13 includes the pre-computed checksum. The same pattern applies to EAN-8 (7 or 8 digits) and UPC (11 or 12 digits). This means the widget is flexible about whether the developer provides checksum digits.

---

## src/components/Barcode.tsx

**1. Purpose of this file?**
Presentational component for non-QR barcodes. Renders an `<svg>` element (populated by the `useRenderBarcode` hook via JsBarcode), handles error display, and optionally renders a download button.

**2. What kind of logic is described in this file?**
Uses `useRenderBarcode(config)` to get `{ ref, error }`. If `error` is true, renders a div with an error alert (`role="alert"`, `alert-danger` class) — only visible when `config.logLevel !== "None"`. Otherwise renders a `<div class="barcode-renderer">` containing the download button (at top or bottom based on `buttonPosition`) and an `<svg ref={ref} />`.

**3. What part of behavior can be documented from this file?**
When barcode generation fails (invalid value or JsBarcode throws), the SVG is not rendered — only the error alert appears (if logLevel allows it). The download button placement (top/bottom) is driven by `downloadButton.buttonPosition`. The SVG element is always present in the DOM on success; JsBarcode mutates it in-place via the ref.

**4. Is it user-facing?**
Yes — renders the visible barcode SVG and error messages.

**5. What new did you learn from this file?**
When `logLevel === "None"`, errors are completely silent — no visual feedback, no console output. The error div still renders (empty) when `logLevel !== "None"` and error is true, but displays nothing when `logLevel === "None"` because the `alert-danger` content is behind the `logLevel !== "None"` guard.

---

## src/components/QRCode.tsx

**1. Purpose of this file?**
Presentational component for QR codes. Wraps `QRCodeSVG` from the `qrcode.react` library in a forward-ref wrapper, renders it with optional title, optional image overlay, and optional download button.

**2. What kind of logic is described in this file?**
`QRCodeSVGWrapper` is a `forwardRef` component wrapping `QRCodeSVG` to expose the underlying SVG ref for download operations. `QRCodeRenderer` destructures the config and renders: optional `<h3>` title (when `showTitle` is true), download button (top/bottom), and the `QRCodeSVGWrapper` with `imageSettings={overlay}` for the optional overlay image. The download button uses the shared `downloadCode` utility.

**3. What part of behavior can be documented from this file?**
QR code title is rendered as `<h3>` with class `qrcode-renderer-title` — semantic heading, not just visual text. The `imageSettings` prop passed to QRCodeSVG is the full overlay config object (src, x, y, height, width, opacity, excavate) or `undefined` if no overlay is configured. When `qrOverlay` is enabled but `qrOverlaySrc` is unavailable (loading/not set), `overlay` is `undefined` and no image is applied.

**4. Is it user-facing?**
Yes — renders the visible QR code SVG, optional title, and download button.

**5. What new did you learn from this file?**
The `forwardRef` wrapper exists specifically to allow the download functionality to access the underlying SVG DOM element via `ref`. The `QRCodeSVG` component from `qrcode.react` does not expose a ref by default (or the API changed), requiring this wrapper pattern. The `displayName = "QRCodeSVGWrapper"` is set for React DevTools clarity.

---

## src/components/DownloadButton.tsx

**1. Purpose of this file?**
Simple presentational button component that renders the download trigger UI element — a `<button>` with a download icon SVG and a caption.

**2. What kind of logic is described in this file?**
Renders a `<button type="button" class="barcode-generator-download-button">` with an optional `aria-label`, a `<DownloadIcon>` SVG, and an optional text caption. The `onClick` handler is passed through directly.

**3. What part of behavior can be documented from this file?**
The button always has `type="button"` (no form submission). When `ariaLabel` is not provided, the button has no `aria-label` attribute — the download icon alone provides no accessible label. Caption text is optional — the button can be icon-only.

**4. Is it user-facing?**
Yes — the download button is visible to end users when `allowDownload` is true.

**5. What new did you learn from this file?**
An icon-only download button (no caption, no aria-label) would not be accessible. The widget provides `downloadButtonAriaLabel` as a configurable textTemplate to ensure screen reader users can identify the button purpose — but it is optional and defaults to "Download code as file" (the XML default), so accessibility is only guaranteed if the developer retains the default or sets a meaningful label.

---

## src/hooks/useRenderBarcode.ts

**1. Purpose of this file?**
React hook that manages the JsBarcode rendering lifecycle for non-QR barcodes — validates the value, renders to the SVG ref, and tracks error state.

**2. What kind of logic is described in this file?**
Returns `{ ref: RefObject<SVGSVGElement>, error: boolean }`. In a `useEffect` keyed on all rendering parameters: resets error, runs `validateBarcodeValue(format, value)`, runs `validateAddonValue(addonFormat, addonValue)` if addon is set, calls `renderBarcode(ref, options)` inside try/catch. On any validation failure or thrown error, calls `printError(message, logLevel)` and sets `error = true`. When `value` becomes empty, clears error without re-rendering.

**3. What part of behavior can be documented from this file?**
Both validation errors and JsBarcode runtime errors are treated identically from the user perspective — both produce the same error alert. The hook re-runs whenever any rendering parameter changes (value, format, dimensions, advanced options), so barcode updates reactively as the bound attribute changes. Error state persists until either a valid render succeeds or the value clears.

**4. Is it user-facing?**
No. Hook logic only, but controls whether the user sees a barcode or error.

**5. What new did you learn from this file?**
The `printError()` call respects `logLevel` — "None" suppresses all console output and UI errors; "Info" shows UI error alert only; "Debug" adds detailed console information. This three-level log system means the widget can fail completely silently in production (`logLevel="None"`) while still providing detailed debug output during development (`logLevel="Debug"`).

---

## src/utils/barcodeRenderer-utils.ts

**1. Purpose of this file?**
Wrapper around the `jsbarcode` library providing two rendering modes: standard single barcode and composite barcode-with-addon (EAN-5/EAN-2). Exports the `renderBarcode` orchestration function.

**2. What kind of logic is described in this file?**
`renderBarcode()` dispatches on `addonFormat`: for EAN5 or EAN2, calls `createBarcodeWithAddon()`; otherwise calls `createStandardBarcode()`. `createBarcodeWithAddon()` uses JsBarcode's chaining API: `JsBarcode(svgRef)[mainFormat](value, opts).blank(spacing)[addonFormat](addonValue, opts).render()`. `createStandardBarcode()` uses the direct single-call API: `JsBarcode(svgRef, value, options)`.

**3. What part of behavior can be documented from this file?**
EAN addons are rendered via JsBarcode's multi-barcode chaining API (not a separate barcode instance). The `blank()` call inserts pixel spacing between the main barcode and addon. Both main and addon share the same `displayValue` setting — the addon text below mirrors the main barcode's text-display preference. Standard barcodes use a simpler single-call API.

**4. Is it user-facing?**
No. Internal rendering utility.

**5. What new did you learn from this file?**
The addon rendering uses `[mainFormat]` dynamic property access on the BarcodeService object — this means the format string (e.g. `"EAN13"`) is used directly as a method name. This is fragile if the format name in the enum doesn't exactly match the JsBarcode method name. Currently: `EAN13`, `EAN8`, `EAN5`, `EAN2` match exactly. The `addonValue` is asserted non-null (`addonValue!`) in the EAN5/EAN2 case — this is safe because `createBarcodeWithAddon` is only called when an addon value exists.

---

## src/utils/download-code.ts

**1. Purpose of this file?**
Async orchestration function for downloading the rendered barcode/QR code as a PNG file. Bridges the rendering ref to the download utilities.

**2. What kind of logic is described in this file?**
`downloadCode(ref, config, fileName)`: gets `svgElement` from ref, calls `prepareSvgForDownload()` to clone and set namespaces, processes QR overlay images to base64 (for cross-origin compatibility), converts SVG to PNG via `convertSvgToPng()` at 2x scale, generates a filename if not provided (format+value hash+timestamp), then triggers a browser download via `downloadBlob()`.

**3. What part of behavior can be documented from this file?**
Downloads always produce PNG files (not SVG). The 2x scale factor (`convertSvgToPng(clonedSvg, 2)`) means the downloaded PNG is twice the pixel dimensions of the rendered SVG, improving print/display quality. Custom filename is normalized to end in `.png` (handled in `barcodeConfig`). The auto-generated filename format is `{type}_{valueHash}_{timestamp}.png`.

**4. Is it user-facing?**
Yes — triggers a browser download, which is directly user-visible. Errors are caught and logged to `console.error` but are not shown to the user (no UI error feedback on download failure).

**5. What new did you learn from this file?**
For QR codes with overlay images, `processQRImages()` fetches the overlay image and converts it to base64 before serializing the SVG. This is required because external URLs in SVG `<image>` elements are blocked during canvas rendering (CORS/taint restrictions). The download fails silently on download error but logs to console — download failures are invisible to the end user.

---

## src/utils/download-utils.ts

**1. Purpose of this file?**
Low-level SVG/PNG download utilities: cloning SVG, converting to PNG via canvas, converting overlay images to base64, and triggering browser download via anchor click.

**2. What kind of logic is described in this file?**
`prepareSvgForDownload()`: clones SVG and sets `xmlns` + `xmlns:xlink` namespace attributes (required for proper serialization). `convertSvgToPng()`: serializes SVG to string, creates Blob URL, loads into `<img>`, draws onto `<canvas>` at 2x scale with white background fill, converts canvas to PNG Blob. `convertImageToBase64()`: fetches external image, reads as DataURL via FileReader. `downloadBlob()`: creates `<a>` element with `download` attribute, appends to body, clicks, removes. `generateFileName()`: `{prefix}_{hash}_{timestamp}.png` using DJB2-like hash + YYYYMMDD_HHMMSS timestamp.

**3. What part of behavior can be documented from this file?**
Downloaded PNG always has a white background — the canvas fill `ctx.fillStyle = "#ffffff"` ensures transparent SVG areas become white in the PNG. The canvas approach means the download captures the SVG at the point of click, not a live screenshot — if the SVG animates or changes, the download captures the current frame. The DJB2-like hash of the barcode value is truncated to 10 base36 characters in the filename.

**4. Is it user-facing?**
No. Download utilities only, but produce the downloaded file visible to the user.

**5. What new did you learn from this file?**
`downloadBlob` uses `ownerDocument` (passed from the component) rather than the global `document` — this correctly handles cases where the widget renders in an iframe or shadow DOM context (common in Mendix). The `isExternalUrl()` check uses a simple `http://` / `https://` prefix test — data URIs or relative URLs are treated as local (no base64 conversion needed).

---

## src/BarcodeGenerator.editorConfig.ts

**1. Purpose of this file?**
Defines design-time property visibility rules (`getProperties`) and Studio Pro validation (`check`) for the barcode generator widget. Determines which properties are shown/hidden based on the current configuration state.

**2. What kind of logic is described in this file?**
`getProperties()`: hides QR-only properties when format is not QRCode and vice versa; hides overlay sub-properties unless `qrOverlay` is true; hides `enableEan128` for non-CODE128 formats; hides `enableFlat` unless Custom + EAN13/EAN8 + no addons; hides `lastChar` unless Custom + EAN13 + not flat + no addons; hides EAN addon props unless Custom + EAN13/EAN8/UPC; hides `enableMod43` unless Custom + CODE39; hides custom format selector unless `codeFormat === "Custom"`; hides download sub-props unless `allowDownload` is true; hides X/Y position fields when `qrOverlayCenter` is true. `check()`: validates `codeWidth >= 1`, `codeHeight >= 20`, `qrSize >= 50`, and static barcode/addon values via `validateBarcodeValue` / `validateAddonValue`. `getPreview()` returns `null` (uses widget icon, no structure preview).

**3. What part of behavior can be documented from this file?**
The property visibility rules define strict format-compatibility constraints: `enableFlat` only works for EAN-13/EAN-8 without addons; `lastChar` only works for EAN-13 without flat/addons; MOD43 check digit only applies to CODE39. These constraints are enforced at design-time (by hiding the properties) but NOT at runtime (the hook passes all values to JsBarcode regardless). Design-time validation uses `stripQuotes()` and `isDynamicExpression()` to skip attribute-bound values and only validate static literals.

**4. Is it user-facing?**
No. Design-time only.

**5. What new did you learn from this file?**
`isDynamicExpression(value)` checks if a value starts with `$`, contains `/`, or is empty — this catches `$currentObject/Attribute`, `$module/Microflow`, etc. Static string literals like `'1234567890128'` or `"ABC-123"` are validated; dynamic expressions get a warning hint instead of a hard error. The `check` function validates `qrSize >= 50` against the `codeHeight` property key (a copy-paste bug in the error property reference: `property: "codeHeight"` should be `property: "qrSize"`).

---

## src/BarcodeGenerator.editorPreview.tsx

**1. Purpose of this file?**
Renders the live preview of the widget in Mendix Studio Pro's design canvas and structure canvas. Shows either a QR code preview or a barcode preview image based on the selected format.

**2. What kind of logic is described in this file?**
`preview()` function (exported): uses `parseStyle` to convert style strings, determines `isQrCode` based on `codeFormat`, renders a `PreviewDownloadButton` (an `<a>` element, non-functional) if `allowDownload` is true, and delegates to `<QRCodePreview>` or `<BarcodePreview>` components from `src/components/preview/`. `getPreviewCss()` exports the widget SCSS for Studio Pro injection.

**3. What part of behavior can be documented from this file?**
The preview download button is an `<a>` element with `role="button"` (not a real `<button>`) — it is non-functional in the design canvas, purely visual. The `showAsCard` CSS modifier is applied in preview, giving accurate visual representation of the card style.

**4. Is it user-facing?**
No. Design-time preview only.

**5. What new did you learn from this file?**
The preview components (`BarcodePreview`, `QRCodePreview`) are separate from the runtime components — they use static SVG assets from `src/assets/barcodes/` rather than dynamically generating barcodes. This means the Studio Pro canvas always shows a representative static barcode image, not the actual encoded value. `getPreviewCss()` is needed to inject the widget stylesheet into the Studio Pro design canvas for accurate visual preview.

---

## src/__tests__/BarcodeGenerator.spec.tsx

**1. Purpose of this file?**
Comprehensive unit tests covering all widget configurations: format selection, QR rendering, overlay, download button, display options, advanced barcode options, EAN addons, error handling, accessibility, and integration scenarios.

**2. What kind of logic is described in this file?**
`jsbarcode` and `qrcode.react` are fully mocked. Tests cover: empty/loading state shows fallback message; `showAsCard` adds CSS class; all 10 barcode formats call JsBarcode with correct format string; QR code receives correct props (size, margin, level); title appears as `<h3>`; overlay is applied when `qrOverlaySrc` is available; download button position (top/bottom); aria-label and caption; `displayValue`, `codeWidth`, `codeHeight`, `codeMargin` passed to JsBarcode; `enableEan128`, `enableFlat`, `enableMod43`, `lastChar`; EAN-5 and EAN-2 addon chaining; error display with `role="alert"`; keyboard accessibility of download button.

**3. What part of behavior can be documented from this file?**
The behavioral contract for error rendering: JsBarcode errors produce an alert with class `alert-danger` and `role="alert"`. A valid value after error clears the error state. The EAN addon chaining test formally verifies the JsBarcode chain: `instance.EAN13(value, opts).blank(spacing).EAN5(addonValue, opts).render()`. Accessibility: download button has `aria-label` support; error messages use ARIA roles; QR title is a heading; container is focusable via `tabIndex`.

**4. Is it user-facing?**
No. Tests only.

**5. What new did you learn from this file?**
The `dynamic()` helper from `@mendix/widget-plugin-test-utils` creates mock `DynamicValue<T>` objects — `dynamic("value")` creates an available value, `dynamic("", true)` creates a loading state, `dynamic<string>()` (no args) creates unavailable. The test confirms that when `qrOverlaySrc` is unavailable (dynamic()), the QR code still renders but `data-image` attribute is `"false"` — the overlay is gracefully absent.

---

## e2e/BarcodeGenerator.spec.js

**1. Purpose of this file?**
End-to-end test file for the barcode generator widget using Playwright. Currently contains only a placeholder test.

**2. What kind of logic is described in this file?**
Describes a single placeholder test ("renders barcode generator widget") that only checks `page.locator("body")` is visible. Includes session logout cleanup in `afterEach` (Mendix's 5-session license limit). The test file includes commented-out TODO notes with example test structure for actual barcode verification.

**3. What part of behavior can be documented from this file?**
The widget has not yet been tested in a real Mendix runtime environment via e2e tests. The placeholder structure shows the expected test pattern: locate `.mx-name-barcodeGenerator`, fill a text input, check for canvas/SVG visibility. The session logout cleanup pattern (`window.mx.session.logout()`) is consistent with other widgets in this repo.

**4. Is it user-facing?**
Tests only — no user-facing behavior captured yet.

**5. What new did you learn from this file?**
This widget has no completed e2e tests as of v1.0.0. All behavioral verification is done through unit tests. The commented-out example shows the intended test approach (using `.mx-name-barcodeGenerator` locator and a linked text input to set the code value), indicating a standard Mendix test app structure is planned but not yet implemented.

---

## CHANGELOG.md

**1. Purpose of this file?**
Version history for the barcode-generator-web widget.

**2. What kind of logic is described in this file?**
Single released version: v1.0.0 (2026-04-17). Features: QR code generator, configurable QR properties, download functionality, comprehensive configuration for various barcode types.

**3. What part of behavior can be documented from this file?**
This is the initial release. The changelog title says "QR Code Generator widget" in the initial release note — suggesting the widget began as a QR-only widget and was expanded to include full barcode support before release.

**4. Is it user-facing?**
No. Developer-facing only.

**5. What new did you learn from this file?**
The "Initial release" framing under v1.0.0 with "Initial release of QR Code Generator widget" title suggests the project started as a QR code widget and grew to include all barcode formats. This aligns with the `codeFormat` default being `CODE128` (barcode) in the XML while the initial changelog entry emphasizes QR.

---

## Summary of Key Findings

- **Widget identity**: Generates barcodes and QR codes from string input. Supports 10+ barcode formats + QR code. Rendered as SVG in the browser. Version 1.0.0, initial release April 2026.
- **Context requirement**: `needsEntityContext="true"` — must be placed inside a data widget (data view, list view, etc.). Unlike badge-button which works anywhere.
- **Format selection**: Three modes: `CODE128` (standard barcode, default), `QRCode`, and `Custom` (10 specific formats: CODE128, EAN-13, EAN-8, UPC, CODE39, ITF-14, MSI, Pharmacode, Codabar, CODE93).
- **Props**: `codeValue` (expression, required), `codeFormat` (enum, required), `emptyMessage` (textTemplate), `allowDownload` (boolean), display settings (width, height, margin, showAsCard, displayValue), advanced barcode options (EAN-128, flat, MOD43, lastChar), EAN addons (EAN-5/EAN-2 with spacing), QR settings (size, error correction level L/M/Q/H, margin, title, showTitle), QR overlay (image, position, dimensions, opacity, excavate).
- **Empty state**: When `codeValue` is empty/unavailable, renders a configurable empty message instead of the barcode.
- **Error handling**: Invalid barcode values show an alert (controlled by `logLevel`). `logLevel=None` silences all errors. Three levels: None, Info (UI alert only), Debug (UI alert + console details).
- **Download**: Optional PNG download button (position: top/bottom). Downloads at 2x pixel density. Auto-generated filename includes value hash + timestamp. Custom filename supported.
- **QR overlay**: Optional image overlay on QR codes. Supports centering or exact pixel positioning. `excavate=true` removes QR dots behind the image. Image converted to base64 on download for cross-origin compatibility.
- **Validation**: Per-format value validation at both design-time (Studio Pro check) and runtime. Character set and length constraints vary by format.
- **Offline capable**: `offlineCapable=true`.
- **E2E tests**: No completed e2e tests in v1.0.0 — placeholder only.
