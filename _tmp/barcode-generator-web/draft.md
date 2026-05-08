# Draft: barcode-generator-web

Widget path: `packages/pluggableWidgets/barcode-generator-web/`

---

## BarcodeGenerator.xml

1. **What is the purpose of this file?**
   This is the Mendix widget descriptor XML. It defines the widget's identity (`com.mendix.widget.web.barcodegenerator.BarcodeGenerator`), metadata (name, description, Studio category), and the full property schema. The widget is marked `needsEntityContext="true"` and `offlineCapable="true"`, placing it in the "Display" category in Studio Pro.

2. **What kind of logic is described in this file?**
   No runtime logic. Declares property groups: General (data source, download), Advanced (barcode-format-specific options, EAN addons, QR settings, log level), and Display (barcode dimensions, card mode, QR overlay). Contains i18n translations for user-visible text properties (en_US, nl_NL).

3. **What part of behavior can be documented from this file?**
   Three top-level format modes: `CODE128` (Barcode), `QRCode`, and `Custom` (reveals `customCodeFormat` with 10 sub-formats). The download subsystem is opt-in via `allowDownload` boolean with configurable caption, aria-label, custom filename, and button position (top/bottom). EAN addon support (EAN-2, EAN-5) with configurable spacing. QR overlay is a separate opt-in boolean (`qrOverlay`) that shows image-source, centering, position, size, opacity, and excavate properties. Log level has three modes: None (silent), Info (UI error), Debug (console detail).

4. **Is it user-facing?**
   Indirectly — this file drives what Studio Pro shows in the widget property panel. Developers configure the widget here; end-users see the rendered barcode/QR code.

5. **What new did you learn from this file?**
   The widget uses `codeFormat=CODE128` as the default and provides a `Custom` mode that unlocks 10 specific barcode sub-formats. The `enableFlat` and `lastChar` options are scoped to EAN variants only (not broadly applicable). `qrSize` default is 128px; preview is clamped to 200px max. `qrMargin` is in module units (QR grid cells), not pixels.

---

## typings/BarcodeGeneratorProps.d.ts

1. **What is the purpose of this file?**
   Auto-generated TypeScript interface file (from BarcodeGenerator.xml). Provides type definitions for all container props (`BarcodeGeneratorContainerProps`) and preview props (`BarcodeGeneratorPreviewProps`). Acts as the contract between Mendix framework and widget code.

2. **What kind of logic is described in this file?**
   Type declarations only. Enumerations: `CodeFormatEnum` ("CODE128" | "QRCode" | "Custom"), `ButtonPositionEnum`, `CustomCodeFormatEnum` (10 values), `AddonFormatEnum`, `QrLevelEnum` ("L"|"M"|"Q"|"H"), `LogLevelEnum`. Container props use Mendix runtime types (`DynamicValue<string>`, `DynamicValue<WebImage>`, `Big` for decimal opacity). Preview props use plain primitives with nullable numbers.

3. **What part of behavior can be documented from this file?**
   `codeValue` and `addonValue` are `DynamicValue<string>` — they can be loading/unavailable at runtime. `qrOverlaySrc` is `DynamicValue<WebImage>`. `qrOverlayOpacity` is typed as `Big` (arbitrary-precision decimal), converted to number at usage. Preview props have nullable numbers for all integer fields (`addonSpacing: number | null`, `qrSize: number | null`), indicating widget handles nulls gracefully.

4. **Is it user-facing?**
   No — internal type contract.

5. **What new did you learn from this file?**
   `qrOverlaySrc` in preview can be either `{ type: "static"; imageUrl: string }` or `{ type: "dynamic"; entity: string }`. Dynamic entity-based images are not resolvable in preview mode (fall back to placeholder). This is a behavioral constraint: overlay images from Mendix entities won't show in Studio Pro preview.

---

## src/BarcodeGenerator.tsx (main entry)

1. **What is the purpose of this file?**
   The top-level widget component. Receives all Mendix props, calls `barcodeConfig()` to normalize them, then branches on `config.type` to render either `QRCodeRenderer` or `BarcodeRenderer`. Also applies root CSS classes and handles the empty-value fallback.

2. **What kind of logic is described in this file?**
   Minimal orchestration: empty-value guard (renders `<span>` with `emptyMessage` if `codeValue` is falsy), format dispatch, and CSS class composition (`barcode-generator`, `barcode-generator--as-card` when `showAsCard` is true). Imports and applies the SCSS stylesheet.

3. **What part of behavior can be documented from this file?**
   When `codeValue` is empty/unavailable, the widget renders a plain `<span>` with the configured empty message (default: "No barcode value provided"). The root element always has `tabIndex` and `style` applied for keyboard navigation and inline styling support. `showAsCard` adds a card border/background via CSS modifier class.

4. **Is it user-facing?**
   Yes — renders the visible widget root and controls card presentation.

5. **What new did you learn from this file?**
   The fallback message is the raw `emptyMessage?.value` with a hardcoded English default. Localization of the fallback depends on the `emptyMessage` textTemplate being properly configured. If `codeValue` is loading (DynamicValue not yet available), the empty state is shown — not a loading spinner.

---

## src/config/Barcode.config.ts

1. **What is the purpose of this file?**
   Central configuration builder. The `barcodeConfig()` function maps raw Mendix props to a normalized `BarcodeConfig` discriminated union (`BarcodeTypeConfig | QRCodeTypeConfig`). Also defines the `DownloadButtonConfig`, `CodeBaseTypeConfig`, `BarcodeTypeConfig`, and `QRCodeTypeConfig` interfaces.

2. **What kind of logic is described in this file?**
   Format resolution: when `codeFormat === "Custom"`, it uses `customCodeFormat`; otherwise uses `codeFormat` directly. Download button config is built only when `allowDownload` is true. QR margin is separate from barcode margin. Overlay config is only included when `qrOverlaySrc.status === "available"`. When `qrOverlayCenter` is true and X/Y are 0, they are passed as `undefined` (centering takes precedence). Custom filename is sanitized and forced to `.png` extension.

3. **What part of behavior can be documented from this file?**
   **Behavioral constraint**: `margin` for barcode uses `props.codeMargin` while QR uses `props.qrMargin` — these are independent controls. `qrOverlayX` and `qrOverlayY` of 0 are treated as undefined (no manual positioning), which is a constraint: X=0, Y=0 cannot be used to explicitly position at top-left if `qrOverlayCenter` logic is active. Custom filenames without `.png` extension get it appended automatically; auto-generated filenames are handled by `download-utils`.

4. **Is it user-facing?**
   No — internal config normalization.

5. **What new did you learn from this file?**
   The `getFileName` function normalizes download filenames: if the user provides a non-empty value it enforces `.png` extension; if empty, returns `undefined` so `download-utils` generates a name from format type, content hash, and timestamp. This means developers cannot configure a non-PNG download format.

---

## src/config/validation.ts

1. **What is the purpose of this file?**
   Runtime and design-time barcode value validation. `validateBarcodeValue()` checks format-specific constraints (character set, length). `validateAddonValue()` validates EAN-5 (exactly 5 numeric digits) and EAN-2 (exactly 2 numeric digits) addon values.

2. **What kind of logic is described in this file?**
   Format-by-format switch with regex and length constraints. All formats have documented max lengths for "readability" (not fundamental encoding limits). Empty values pass validation (assumed dynamic binding will provide value at runtime).

3. **What part of behavior can be documented from this file?**
   **Per-format constraints:**
   - EAN-13: 12 or 13 numeric digits only
   - EAN-8: 7 or 8 numeric digits only
   - UPC: 11 or 12 numeric digits only
   - ITF-14: exactly 14 numeric digits
   - CODE39: uppercase A-Z, digits, space and `- . $ / + %`, max 43 chars
   - MSI: numeric only, max 30 digits
   - Pharmacode: numeric only, max 7 digits
   - Codabar: digits, A-D (start/stop), `- $ : / . +`, max 20 chars
   - CODE93: no control characters, max 47 chars
   - QRCode: max 1200 chars recommended
   - CODE128: no control characters, max 80 chars

4. **Is it user-facing?**
   Indirectly — validation errors surface as Studio Pro design-time errors or runtime error states in the widget.

5. **What new did you learn from this file?**
   EAN-13 and EAN-8 accept either the data-only length (12/7) OR data+checksum length (13/8) — the checksum digit is optional in the input. This is a behavioral constraint: the barcode library handles checksum calculation when 12/7 digits are provided.

---

## src/components/Barcode.tsx

1. **What is the purpose of this file?**
   The `BarcodeRenderer` component. Renders a linear barcode using JsBarcode into an SVG element. Handles render errors, optional download button positioning (top/bottom), and error display.

2. **What kind of logic is described in this file?**
   Uses `useRenderBarcode` hook to get a ref and error state. Conditionally renders an error alert (`alert-danger` with `role="alert"`) when rendering fails AND `logLevel !== "None"`. Download button rendered above or below SVG based on `buttonPosition`. The SVG element itself is a plain `<svg ref={ref} />` — JsBarcode mutates it in place.

3. **What part of behavior can be documented from this file?**
   **Behavioral constraint**: error message is only shown when `logLevel` is `Info` or `Debug` — with `None`, the widget silently renders nothing on error. Error message text: "Unable to generate barcode. Please check the barcode value and format configuration." Error container has `role="alert"` for screen reader support. Download button only appears when `downloadButton` config is present (i.e., `allowDownload` is true).

4. **Is it user-facing?**
   Yes — renders the barcode SVG and error state visible to end-users.

5. **What new did you learn from this file?**
   The `BarcodeRenderer` has no loading state — it either renders the barcode or shows an error. There's no intermediate "loading" visual. The SVG is rendered at `max-width: 100%` via CSS, so it scales with container width.

---

## src/components/QRCode.tsx

1. **What is the purpose of this file?**
   The `QRCodeRenderer` component. Renders a QR code using the `qrcode.react` library's `QRCodeSVG` via a `forwardRef` wrapper (`QRCodeSVGWrapper`). Supports optional title heading, download button, and image overlay.

2. **What kind of logic is described in this file?**
   Uses `useRef` for the SVG element. Passes `imageSettings` (overlay config) directly to `QRCodeSVG`. Title is rendered as `<h3>` when `showTitle` is true. Download button positioned above/below the QR SVG. The `forwardRef` wrapper exists to bridge React's ref forwarding with `qrcode.react`'s API.

3. **What part of behavior can be documented from this file?**
   QR title appears as an `<h3>` element with class `qrcode-renderer-title` — styled as small, normal-weight text above the code. The image overlay is passed as `imageSettings` to `QRCodeSVG`, which handles rendering internally (excavation, positioning, opacity). When `qrOverlay` is false or `qrOverlaySrc` is unavailable, `imageSettings` is `undefined` and no overlay appears.

4. **Is it user-facing?**
   Yes — renders the QR code, optional title, and optional download button.

5. **What new did you learn from this file?**
   The library used for QR codes is `qrcode.react` (specifically `QRCodeSVG`). The overlay image handling including excavation is entirely delegated to that library via `imageSettings`. This means overlay behavior (including the "excavate background" feature) is library-controlled, not custom code.

---

## src/hooks/useRenderBarcode.ts

1. **What is the purpose of this file?**
   React hook that orchestrates barcode rendering: validates the value+format+addon combination, then calls `renderBarcode()` from `barcodeRenderer-utils`. Returns a ref (for the SVG element) and an `error` boolean.

2. **What kind of logic is described in this file?**
   `useEffect` with full dependency array (value, format, dimensions, all advanced options). Runs validation first (design-time constraints applied at runtime). On addon format present: also validates addon value. On validation failure: sets error state and returns. On render exception from JsBarcode: catches and sets error state. On empty value: clears error state.

3. **What part of behavior can be documented from this file?**
   Error state resets at the start of each render attempt (not sticky from previous values). The dependency array ensures re-render on any prop change. Validation failure AND JsBarcode exceptions both result in the same `error: true` state — the UI cannot distinguish between "invalid value" and "library crash". `printError` is called for developer debugging only when `logLevel === "Debug"`.

4. **Is it user-facing?**
   No — internal hook, but its `error` output drives the error UI in `BarcodeRenderer`.

5. **What new did you learn from this file?**
   Addon validation is only applied when `addonValue` is non-empty AND `addonFormat` is not "None". If `addonValue` is empty string, addon validation is skipped (no error), which means empty addon is effectively treated as "no addon" at runtime.

---

## src/utils/barcodeRenderer-utils.ts

1. **What is the purpose of this file?**
   Low-level barcode rendering utilities. `renderBarcode()` dispatches to either `createBarcodeWithAddon()` or `createStandardBarcode()` based on `addonFormat`. Wraps the JsBarcode library calls.

2. **What kind of logic is described in this file?**
   `createStandardBarcode` calls `JsBarcode(ref.current, value, options)` directly. `createBarcodeWithAddon` uses JsBarcode's chained API: `JsBarcode(ref)[mainFormat](value, opts).blank(spacing)[addonFormat](addonValue, opts).render()`. The `BarcodeService` interface documents the chained method pattern for EAN formats.

3. **What part of behavior can be documented from this file?**
   Addon barcode rendering uses JsBarcode's multi-barcode chain: main barcode → blank spacer → addon barcode → render. Addon barcode always uses `width: 1` (narrow bars) and inherits `displayValue` from main barcode options. Default `addonSpacing` is 20px. Only `EAN5` and `EAN2` trigger addon rendering; all others use standard single-barcode rendering.

4. **Is it user-facing?**
   No — utility layer.

5. **What new did you learn from this file?**
   JsBarcode is the underlying library for all non-QR barcodes (`jsbarcode` package). The chained API pattern (`JsBarcode(el)[format](value, opts).render()`) is used for multi-barcode layouts. This means EAN addons are rendered as part of the same SVG element, not as separate DOM nodes.

---

## src/utils/download-code.ts

1. **What is the purpose of this file?**
   Orchestrates barcode/QR code download. Clones the SVG, processes QR overlay images (converts external URLs to base64), converts SVG to PNG at 2× scale, generates filename, triggers download via `downloadBlob`.

2. **What kind of logic is described in this file?**
   Async function: clone SVG → for QR codes, `processQRImages` (inline external image URLs as base64 for standalone PNG) → `convertSvgToPng` at 2× scale → resolve filename (custom or auto-generated) → `downloadBlob`.

3. **What part of behavior can be documented from this file?**
   Downloads are always PNG format (not SVG). Scale is hardcoded to 2× for better resolution. QR overlay images from external URLs are fetched and embedded as base64 so the downloaded PNG is self-contained. Auto-generated filename pattern: `{type_format}_{hash}_{timestamp}.png`. For QR: `qrcode_{hash}_{timestamp}.png`; for barcodes: `barcode_{format}_{hash}_{timestamp}.png`.

4. **Is it user-facing?**
   Yes — triggers browser download visible to end-user.

5. **What new did you learn from this file?**
   If the overlay image fetch fails (CORS, network error), `convertImageToBase64` falls back to returning the original URL. This means the downloaded PNG might reference an external URL that doesn't work offline. The white background fill is hardcoded: downloaded PNGs always have a white background regardless of page theme.

---

## src/utils/download-utils.ts

1. **What is the purpose of this file?**
   SVG/PNG download helper utilities. Functions: `generateFileName`, `prepareSvgForDownload`, `convertImageToBase64`, `isExternalUrl`, `processQRImages`, `downloadBlob`, `convertSvgToPng`.

2. **What kind of logic is described in this file?**
   `convertSvgToPng`: serializes SVG to string → Blob → Object URL → loads in `<img>` → draws on `<canvas>` scaled by factor → calls `canvas.toBlob("image/png", 1.0)`. `generateFileName`: hash (djb2-like, 32-bit) of codeValue in base36, padded timestamp in `YYYYMMDD_HHMMSS` format. `prepareSvgForDownload`: clones SVG node, sets xmlns attributes for valid standalone SVG.

3. **What part of behavior can be documented from this file?**
   The hash function is a 32-bit integer (djb2-variant) of the encoded value, converted to base36, truncated to 10 chars — not cryptographically unique. Timestamp uses local (browser) time, not UTC. `downloadBlob` creates a temporary `<a>` element, appends to body, clicks, and removes — this is the standard browser download trigger pattern. Canvas PNG quality is set to 1.0 (maximum). Canvas background is always white (`#ffffff` fill before image draw).

4. **Is it user-facing?**
   Indirectly — determines download filename and PNG quality.

5. **What new did you learn from this file?**
   The SVG-to-PNG pipeline relies on the browser's canvas API and `XMLSerializer`. This means download is not supported in environments without DOM/canvas (e.g., SSR, some testing environments). The `ownerDocument` of the SVG element is used for creating the canvas and link — this correctly handles widgets rendered in iframes.

---

## src/utils/helpers.ts

1. **What is the purpose of this file?**
   Single utility function `printError(message, logLevel)` — outputs to `console.error` only when `logLevel === "Debug"`. Silent on `None` or `Info`.

2. **What kind of logic is described in this file?**
   Minimal: a conditional `console.error` call prefixed with `[Barcode Generator]`.

3. **What part of behavior can be documented from this file?**
   `logLevel: "Info"` shows error UI to end-users but does NOT log to console. `logLevel: "Debug"` logs full error details to console (format, value, error message). `logLevel: "None"` suppresses both UI error and console logging.

4. **Is it user-facing?**
   No — developer debugging utility only.

5. **What new did you learn from this file?**
   The Info/Debug split is a deliberate UX choice: Info-level errors are for end-users ("something went wrong" in UI), Debug is for developers (detailed console output). This allows staging/production builds to show user-friendly errors without revealing implementation details.

---

## src/BarcodeGenerator.editorConfig.ts

1. **What is the purpose of this file?**
   Studio Pro design-time behavior: `getProperties()` hides irrelevant property groups based on format selection, `check()` validates property values and returns errors/warnings, `getPreview()` returns null to use the widget's icon.

2. **What kind of logic is described in this file?**
   Conditional property visibility: QRCode hides barcode-specific props and vice versa. `qrOverlay` sub-properties hidden unless QRCode + overlay enabled. `enableFlat` only visible for Custom EAN-13/EAN-8 without addons. `lastChar` only for Custom EAN-13, no flat, no addons. EAN addons only for EAN-13/EAN-8/UPC. `enableMod43` only for CODE39. `customCodeFormat` only when `codeFormat === "Custom"`. Download sub-properties hidden when `allowDownload` is false. `check()` validates: `codeWidth ≥ 1`, `codeHeight ≥ 20`, `qrSize ≥ 50`, plus static barcode value validation using shared `validateBarcodeValue`.

3. **What part of behavior can be documented from this file?**
   **Behavioral constraints from property visibility rules:**
   - `enableFlat` is NOT available for CODE128 or QRCode — only Custom EAN-13 and EAN-8 without addons
   - `lastChar` is ONLY for Custom EAN-13, not flat, no addons — very narrow applicability
   - `enableMod43` only applies to CODE39
   - `enableEan128` only appears for CODE128 format
   - EAN addons only available for EAN-13, EAN-8, UPC
   - `qrOverlayCenter: true` hides X/Y position controls — centering and manual positioning are mutually exclusive in UI
   - `codeWidth` minimum is 1; `codeHeight` minimum is 20; `qrSize` minimum is 50

4. **Is it user-facing?**
   No (Studio Pro only) — but directly shapes developer experience.

5. **What new did you learn from this file?**
   Design-time validation of static literal values uses `isDynamicExpression()` to skip validation for attribute bindings (expressions starting with `$` or containing `/`). For dynamic values, it adds a `warning` (not error) with a format hint. This means static misconfiguration blocks publishing; dynamic misconfiguration is only warned about.

---

## src/BarcodeGenerator.editorPreview.tsx

1. **What is the purpose of this file?**
   Studio Pro editor preview component. Exports `preview()` function (renders widget in design canvas) and `getPreviewCss()`. Uses static SVG assets for barcode and QR code previews instead of actual encoding.

2. **What kind of logic is described in this file?**
   Dispatches to `BarcodePreview` or `QRCodePreview` based on `codeFormat === "QRCode"`. Creates a `PreviewDownloadButton` (renders as `<a>` link, not `<button>`, for preview purposes). Exports SCSS via `getPreviewCss()`.

3. **What part of behavior can be documented from this file?**
   Preview shows static SVG images (not live barcodes) — the actual barcode value has no effect in Studio Pro preview. QR code size is clamped to 200px maximum in preview. Barcode preview height is also clamped to 200px. The download button in preview renders as `<a role="button">` (non-functional link).

4. **Is it user-facing?**
   Studio Pro designers only — not visible to end-users.

5. **What new did you learn from this file?**
   `getPreview()` in `editorConfig.ts` returns null to use the icon; `preview()` here provides the actual in-canvas preview. These are two separate mechanisms: `getPreview` for the widget's icon in the toolbox, `preview` for the in-place canvas rendering. The preview always uses static placeholder SVGs regardless of configured value.

---

## src/hooks/useBarcodePreviewSvg.ts

1. **What is the purpose of this file?**
   Preview-mode hook that selects the appropriate static SVG asset for the currently configured barcode format and (optionally) strips text elements for `displayValue: false`.

2. **What kind of logic is described in this file?**
   Calls `getBarcodeImageUrl()` to get the static SVG URL for the current format/addon/flat combination. If `displayValue` is true, returns the raw SVG URL unchanged. Otherwise, fetches the SVG text, uses `DOMParser` to remove `<text>` elements, serializes back, and returns as a data URI.

3. **What part of behavior can be documented from this file?**
   In Studio Pro preview, text-under-barcode is conditionally shown by parsing and modifying the static SVG. This is a purely preview concern — runtime rendering delegates `displayValue` to JsBarcode. The modification is async (fetch + parse); until resolved, `displayUrl` may briefly equal `imageUrl` (with text visible).

4. **Is it user-facing?**
   Studio Pro only.

5. **What new did you learn from this file?**
   The preview SVG for QR mode uses `BarcodeGeneratorPreview.svg` (a generic QR placeholder), while barcode mode uses format-specific SVGs from `barcodePreview.assets.ts`. The `displayValue` toggle has a visual effect in Studio Pro preview via DOM manipulation of static assets.

---

## src/assets/barcodePreview.assets.ts

1. **What is the purpose of this file?**
   Maps barcode format+addon+flat combinations to static SVG preview assets. Provides `getBarcodeImageUrl()` function for use in preview mode.

2. **What kind of logic is described in this file?**
   `barcodeImageMap` object maps format keys to variant objects (`default`, `flat?`, `EAN2?`, `EAN5?`). Priority order: flat > addon variant > default. Only EAN-13, EAN-8, and UPC have addon variants; only EAN-13 and EAN-8 have flat variants.

3. **What part of behavior can be documented from this file?**
   Supported preview variants: CODE128 (default only), EAN-13 (default, flat, EAN-2, EAN-5), EAN-8 (default, flat, EAN-2, EAN-5), UPC (default, EAN-2, EAN-5), CODE39/ITF-14/MSI/Pharmacode/Codabar/CODE93 (default only). If `codeFormat === "CODE128"` (not Custom), it maps to the CODE128 key — same SVG as Custom CODE128.

4. **Is it user-facing?**
   Studio Pro only.

5. **What new did you learn from this file?**
   There is no QR code entry in `barcodeImageMap` — QR preview uses a separate `BarcodeGeneratorPreview.svg` loaded directly in `QRCodePreview.tsx`. The QR code preview is fully static (no format-variant SVGs).

---

## src/utils/qrcode-preview-utils.ts

1. **What is the purpose of this file?**
   Resolves the QR overlay image source for preview mode. Returns a placeholder SVG when source is unavailable or errors.

2. **What kind of logic is described in this file?**
   `resolveQRImageSrc()`: returns `QR_IMAGE_PLACEHOLDER` (inline SVG data URI — a grey box with an X) if source is null, errored, or dynamic entity-based. Returns the static `imageUrl` for static sources.

3. **What part of behavior can be documented from this file?**
   Dynamic (entity-based) images are not resolvable in preview and always show the placeholder. Static image URLs are shown directly. The placeholder is a grey 80×80 SVG with crossed lines — clearly indicates "image not available in preview". Errors from image loading (`onError`) set `imageSrcError` state which then triggers the placeholder.

4. **Is it user-facing?**
   Studio Pro only.

5. **What new did you learn from this file?**
   The placeholder is an inline SVG data URI defined as a constant `QR_IMAGE_PLACEHOLDER`. This ensures no network request is needed for the error fallback. Dynamic entity images are explicitly handled as non-previewable (not a bug, by design).

---

## src/components/DownloadButton.tsx

1. **What is the purpose of this file?**
   Simple `DownloadButton` React component. Renders a `<button type="button">` with the `DownloadIcon` SVG icon and configurable caption and `aria-label`.

2. **What kind of logic is described in this file?**
   Functional component with `onClick`, `ariaLabel`, and `caption` props. Uses `barcode-generator-download-button` CSS class.

3. **What part of behavior can be documented from this file?**
   Button uses `type="button"` (not `submit`) to prevent form submission in form contexts. Icon and caption are both rendered inside the button — icon always appears before the text. `aria-label` is optional; when provided it overrides the button's accessible name.

4. **Is it user-facing?**
   Yes — visible to end-users when `allowDownload` is true.

5. **What new did you learn from this file?**
   Download button styling (link-like appearance with primary brand color) comes entirely from CSS (`.barcode-generator-download-button`) — it looks like a text link but is semantically a button. This is an explicit accessibility/UX decision to match Mendix's link action pattern while maintaining keyboard semantics.

---

## src/components/preview/BarcodePreview.tsx

1. **What is the purpose of this file?**
   Studio Pro preview component for barcode (non-QR) mode. Renders a static preview image using `useBarcodePreviewSvg` hook, with download button and error fallback.

2. **What kind of logic is described in this file?**
   Gets `imageUrl` and `displayUrl` from `useBarcodePreviewSvg`. If `imageUrl` is null (unsupported format), shows `alert-danger`. Preview height clamped: `Math.min(codeHeight ?? 200, 200)`. Download button position respects `buttonPosition` prop.

3. **What part of behavior can be documented from this file?**
   Preview is clamped to 200px height maximum regardless of the configured `codeHeight`. Unsupported format shows "Barcode format not supported" error in Studio Pro preview. The preview image has `alt="Barcode preview"` for Studio Pro accessibility.

4. **Is it user-facing?**
   Studio Pro only.

5. **What new did you learn from this file?**
   `displayUrl` can briefly be `imageUrl` (with text) before the async SVG modification completes. This creates a flash of text-visible preview, then text disappears when `displayValue` is false. It's a minor preview-only visual artifact.

---

## src/components/preview/QRCodePreview.tsx

1. **What is the purpose of this file?**
   Studio Pro preview component for QR code mode. Renders a static `BarcodeGeneratorPreview.svg` and optionally an overlay image. Supports all QR overlay configuration (centering, position, size, opacity, excavation border).

2. **What kind of logic is described in this file?**
   Computes overlay `imageBaseStyle` based on `qrOverlayCenter` (CSS translate -50%/-50% vs absolute left/top). Excavation is simulated with `backgroundColor: "#ffffff"` and `outline: "3px solid #ffffff"` on the overlay image. Image load errors trigger `imageSrcError` state → placeholder shown.

3. **What part of behavior can be documented from this file?**
   QR overlay in preview shows actual position/size/opacity relative to the static QR placeholder SVG. Excavation effect is a CSS approximation (white outline), not actual pixel removal as `qrcode.react` performs at runtime. Title is rendered as `<h3>` with class `qrcode-renderer-title` when `qrTitle` is non-empty (regardless of `showTitle` in preview — it always shows).

4. **Is it user-facing?**
   Studio Pro only.

5. **What new did you learn from this file?**
   In runtime (`QRCode.tsx`), title visibility is controlled by `showTitle` prop. In preview (`QRCodePreview.tsx`), the title always renders when `qrTitle` is non-empty (no `showTitle` check). This is a preview-only inconsistency — the title preview doesn't respect the `showTitle` toggle.

---

## src/__tests__/BarcodeGenerator.spec.tsx

1. **What is the purpose of this file?**
   Unit test suite using Jest + React Testing Library. Tests: core rendering, barcode formats (all 10 custom formats), QR rendering, QR image overlay, download button functionality, barcode display options, advanced barcode options, EAN addon functionality, error handling, accessibility, and integration scenarios.

2. **What kind of logic is described in this file?**
   Mocks: JsBarcode (verifies call arguments), `qrcode.react` QRCodeSVG (renders as div with data attributes), `download-code` module. Tests use `createBarcodeProps()` helper with `dynamic()` for DynamicValues. Validates: CSS classes, ARIA attributes, button positions, error state, JsBarcode call signatures.

3. **What part of behavior can be documented from this file?**
   All 10 custom barcode formats are individually tested and verified to pass the format string to JsBarcode. QR error correction levels L/M/Q/H are all tested. Download button: no button when `allowDownload: false`; top/bottom positioning tested; click triggers `downloadCode`. Error handling: JsBarcode exception → "Unable to generate barcode" alert. QR overlay: when `qrOverlaySrc` is unavailable (loading), `data-image="false"` attribute — no overlay rendered. Accessibility: download button has `aria-label`, error has `role="alert"`, QR title renders as H3.

4. **Is it user-facing?**
   No — test file.

5. **What new did you learn from this file?**
   The test confirms `logLevel: "Info"` does show the error UI (the test uses `logLevel: "Info"` in `createBarcodeProps` defaults and asserts the error alert is visible). `tabIndex` defaults to `-1` in tests (not focusable by default, but configurable). The unit tests do NOT mock the validation module — `validateBarcodeValue` and `validateAddonValue` run against real inputs in tests.

---

## e2e/BarcodeGenerator.spec.js

1. **What is the purpose of this file?**
   End-to-end Playwright test file. Currently contains a placeholder test ("renders barcode generator widget") that only checks `body` visibility — no actual barcode assertions implemented yet.

2. **What kind of logic is described in this file?**
   Session cleanup in `afterEach` (`window.mx.session.logout()`) to stay within Mendix's 5-session license limit. `beforeEach` navigates to root URL. Single placeholder test.

3. **What part of behavior can be documented from this file?**
   **Behavioral note**: The e2e test file is effectively unimplemented. The TODO comment indicates intent but no barcode-specific e2e scenarios exist yet. The session logout pattern is a standard Mendix e2e constraint (5-session license limit requires explicit logout).

4. **Is it user-facing?**
   No — test infrastructure.

5. **What new did you learn from this file?**
   There are no MODERN_CLIENT guards (no `test.skip(MODERN_CLIENT, ...)` patterns) in this e2e file — unlike some other widgets, barcode-generator-web does NOT have documented platform incompatibilities in e2e tests. The widget appears to be compatible with both Mendix client environments based on this file.

---

## src/ui/BarcodeGenerator.scss

1. **What is the purpose of this file?**
   Widget stylesheet. Defines root `.barcode-generator` container, card variant, barcode/QR code renderer containers, download button styling, and preview-specific image classes.

2. **What kind of logic is described in this file?**
   CSS/SCSS layout: root uses `display: block; width: 100%`. Both `.qrcode-renderer` and `.barcode-renderer` use flex column layout centered (`align-items: center`) with 12px gap. SVG children are `max-width: 100%; height: auto` — responsive. Download button mimics Mendix link styles using CSS custom properties (`--brand-primary`, `--card-bg`, etc.).

3. **What part of behavior can be documented from this file?**
   Card mode uses Mendix theme variables (`--card-bg`, `--card-border`, `--card-border-radius`, `--spacing-medium`). Download button uses `--brand-primary` for color with hover/focus/active/disabled states. Barcode/QR renderers are vertically stacked with centered alignment. Preview images use `object-fit: contain`. Overlay image is `position: absolute` with `object-fit: contain` — requires the renderer to be `position: relative`.

4. **Is it user-facing?**
   Yes — defines visual presentation for end-users.

5. **What new did you learn from this file?**
   The 12px `gap` in flex renderers provides consistent spacing between title, download button, and barcode/QR SVG. The `.barcode-preview-image` sets `width: 100%` (full container width) while `.qrcode-preview-image` sets `width: auto` (preserving aspect ratio to the configured size). These are different sizing strategies for the two code types in preview.

---

## CHANGELOG.md

1. **What is the purpose of this file?**
   Records version history following Keep a Changelog format and Semantic Versioning.

2. **What kind of logic is described in this file?**
   Version history. One release: v1.0.0 (2026-04-17).

3. **What part of behavior can be documented from this file?**
   v1.0.0 (2026-04-17): Initial release. Features: QR code generation from string, configurable QR properties, download functionality, comprehensive configuration and styling for various barcode types.

4. **Is it user-facing?**
   No — developer/release management.

5. **What new did you learn from this file?**
   This is a brand-new widget (v1.0.0 released April 2026). No prior history or breaking changes. The changelog describes it as "QR Code Generator widget" initially, but the final product covers both QR codes and traditional barcodes.
