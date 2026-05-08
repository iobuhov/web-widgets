# Draft: barcode-scanner-web

Widget package: `@mendix/barcode-scanner-web` v2.5.1  
Source: `packages/pluggableWidgets/barcode-scanner-web/`

---

## src/BarcodeScanner.tsx

**1. Purpose of this file?**
Container component (Mendix pluggable widget entry point) that bridges Mendix platform props to the presentational `BarcodeScanner` component. Handles the data write-back and action execution on successful scan.

**2. What kind of logic is described in this file?**
Creates an `onDetect` callback (via `useCallback`) that: (1) checks if `datasource` is in `ValueStatus.Available` state and returns early if not; (2) only sets the attribute value if the scanned data differs from the current attribute value (deduplication); (3) calls `executeAction(props.onDetect)` to trigger the configured action. Passes all other props to `BarcodeScannerComponent`.

**3. What part of behavior can be documented from this file?**
The scanned barcode value is written to the `datasource` `EditableValue<string>` attribute only when the datasource is available AND the new value differs from the current value. This prevents duplicate attribute updates and action triggers when the scanner continuously reads the same barcode. The `onDetect` action fires after every successful scan that produces a new value.

**4. Is it user-facing?**
Yes — the component renders the visible camera view.

**5. What new did you learn from this file?**
The deduplication guard (`data !== props.datasource.value`) is in the container, not the reader — the underlying scanning loop may fire `onDetect` multiple times for the same barcode (e.g. on each scan frame), but the attribute is only updated and the action only executed when the value actually changes.

---

## src/BarcodeScanner.xml

**1. Purpose of this file?**
Widget descriptor declaring identity, properties, and system configuration for the barcode scanner.

**2. What kind of logic is described in this file?**
Declares: `datasource` (attribute, String type, required — the write target for scanned results), `showMask` (boolean, default true — overlay mask), `useAllFormats` (boolean, default true), `barcodeFormats` (object list with `barcodeFormat` enum: AZTEC, CODE_39, CODE_128, DATA_MATRIX, EAN_8, EAN_13, ITF, PDF_417, QR_CODE, RSS_14, UPC_A, UPC_E), `onDetect` (action, optional), dimension props (widthUnit: percentage/pixels; width: 100; heightUnit: percentageOfWidth/pixels/percentageOfParent; height: 75), `detectionLogic` (enum: zxing | native, default native). System properties: Name, Visibility.

**3. What part of behavior can be documented from this file?**
`offlineCapable="true"` — works in offline Mendix apps. `datasource` is an attribute binding (not expression or action) — the widget directly writes to an entity attribute. The default detection mode is `native` (BarcodeDetector API with ZXing fallback). Default dimensions: 100% width, 75% of width for height (landscape ratio). `barcodeFormats` list is hidden when `useAllFormats` is true (controlled in editorConfig).

**4. Is it user-facing?**
Not directly. Defines developer configuration in Studio/Studio Pro.

**5. What new did you learn from this file?**
The widget is categorized under "Images, videos & files" in Studio Pro, not "Inputs" — suggesting it is considered a media capture widget rather than a form input. The `datasource` property uses `type="attribute"` (not `type="expression"`), meaning it requires a direct attribute reference that supports writing — read-only attributes cannot be used.

---

## typings/BarcodeScannerProps.d.ts

**1. Purpose of this file?**
Auto-generated TypeScript types from `BarcodeScanner.xml`. Provides compile-time types for runtime container and design-time preview props.

**2. What kind of logic is described in this file?**
`BarcodeScannerContainerProps`: `datasource` is `EditableValue<string>` (supports both reading current value and `setValue()`), `onDetect` is `ActionValue | undefined`, `barcodeFormats` is `BarcodeFormatsType[]`. `BarcodeScannerPreviewProps`: `datasource` is `string` (the attribute path expression), `height/width` are `number | null`.

**3. What part of behavior can be documented from this file?**
`EditableValue<string>` provides `value` (current string), `status` (`ValueStatus.Available` | loading | unavailable), and `setValue(string)`. The container reads the current value for deduplication and calls `setValue()` to write back scanned data. `onDetect` is an optional `ActionValue` — when not configured, `executeAction(undefined)` is a no-op.

**4. Is it user-facing?**
No. Type declarations only.

**5. What new did you learn from this file?**
`BarcodeFormatEnum` has 12 values: AZTEC, CODE_39, CODE_128, DATA_MATRIX, EAN_8, EAN_13, ITF, PDF_417, QR_CODE, RSS_14, UPC_A, UPC_E. Notably `RSS_14` is in the list — this is a 1D barcode format (also called GS1 DataBar). The `DetectionLogicEnum` has exactly two values: `"zxing"` and `"native"`.

---

## src/components/BarcodeScanner.tsx

**1. Purpose of this file?**
Main presentational component and overlay component for the barcode scanner. Handles camera access check, error display, reader initialization, and rendering the video element with optional mask overlay.

**2. What kind of logic is described in this file?**
`BarcodeScanner`: checks `navigator.mediaDevices.getUserMedia` availability; if absent, renders a `<Alert bootstrapStyle="danger">` error message. Initializes `useCustomErrorMessage` and `useReader` hooks. Renders `<BarcodeScannerOverlay>` with a `<video>` element inside; CSS class includes `mx-barcode-detector` or `mx-zxing-detector` based on `useBrowserAPI` flag. `BarcodeScannerOverlay`: renders the outer container with computed dimensions and optional mask UI (left/right background panels, top/bottom background panels, and a center target area with corner markers).

**3. What part of behavior can be documented from this file?**
The mask overlay creates a visual "target area" for the barcode using CSS divisions: left, right, top, bottom background panels surround the center transparent scanning area. The four corners of the center area have `div.corner` elements. The `onCanPlay` handler auto-plays the video if it was paused on load. If `navigator.mediaDevices.getUserMedia` is missing (HTTP context or unsupported browser), the widget shows a hard error instead of the scanner.

**4. Is it user-facing?**
Yes — the video camera view and mask overlay are directly visible to end users.

**5. What new did you learn from this file?**
The CSS class `mx-barcode-detector` vs `mx-zxing-detector` on the overlay container indicates which detection backend is active. This allows CSS customization per detection mode. The `Alert` component is from `@mendix/widget-plugin-component-kit/Alert` (local workspace package). `getDimensions` from `@mendix/widget-plugin-platform/utils/get-dimensions` computes the `style` object from width/height/unit props.

---

## src/hooks/useReader.ts

**1. Purpose of this file?**
React hook that selects and instantiates the appropriate barcode reader (native BarcodeDetector or ZXing) based on browser support and the configured `detectionLogic`, starts it, and returns the video ref and API flag.

**2. What kind of logic is described in this file?**
`enableBrowserAPI` is determined once via `useMemo`: `isBarcodeDetectorSupported() && args.detectionLogic === "native"`. A `reader` instance (`NativeReader` or `ZxReader`) is created via `useMemo`, keyed on `enableBrowserAPI`, `barcodeFormats`, `useAllFormats`, `useCrop`. In `useEffect`, calls `reader.start(onSuccess, onError)` and returns `reader.stop` as cleanup. Uses `useEventCallback` from `@mendix/widget-plugin-hooks/useEventCallback` to stable-ize `onSuccess`/`onError` callbacks.

**3. What part of behavior can be documented from this file?**
The reader is only recreated when detection mode, format list, crop setting, or video ref changes. The `start/stop` lifecycle is managed by the effect — when the component unmounts or reader changes, `stop` is called automatically. The `useBrowserAPI` boolean in the return value tells the overlay which CSS class to apply.

**4. Is it user-facing?**
No. Hook logic only.

**5. What new did you learn from this file?**
`useEventCallback` is used to wrap `onSuccess`/`onError` so that the reader instance captures a stable function reference — without this, the reader would be recreated on every render because callbacks change identity. This is important because the reader starts a camera stream on creation and we don't want to restart it unnecessarily.

---

## src/hooks/nativeReader.ts

**1. Purpose of this file?**
Implements the `MxBarcodeReader` interface using the browser's native `BarcodeDetector` API. Polls the video stream on a 50ms interval to detect barcodes.

**2. What kind of logic is described in this file?**
`Reader` class: `start()` acquires the camera stream via `getUserMedia(mediaStreamConstraints)`, sets up the video element, then calls `decodeStream` after 50ms. `decodeStream()` calls `detectBarcodesFromElement(barcodeDetector, videoRef)` — if barcodes found, resolves the first result's `rawValue`; if none, schedules another call in 50ms. `stop()` clears the interval and stops all video tracks.

**3. What part of behavior can be documented from this file?**
Native reader polls at 50ms intervals (20fps). Only the first detected barcode in each frame is used — if multiple barcodes are in frame, the first one wins. The reader stops scanning after the first successful detection (does not loop back to detect more). Camera constraints: `facingMode: "environment"` (rear camera), ideal resolution 4096×2160 (4K), minimum 1280×720.

**4. Is it user-facing?**
No. Internal detection backend.

**5. What new did you learn from this file?**
The native reader does NOT handle the crop mask — it passes the full video frame to `BarcodeDetector.detect()`. Looking at the code, `useCrop` is stored but not used in the detect call. This means the mask visual overlay with `showMask` is only used for crop-based detection in the ZXing reader, while the native reader ignores it functionally (though the visual mask is still shown).

---

## src/hooks/zxReader.ts

**1. Purpose of this file?**
Implements the `MxBarcodeReader` interface using the ZXing library (`@zxing/library`). Supports crop-based decoding (using the visible mask area) and full-frame decoding.

**2. What kind of logic is described in this file?**
`Reader` class: `start()` acquires camera stream. If `useCrop` is true: sets up video, creates a capture canvas via `createCaptureCanvas()`, then polls `decodeCropFromVideo()` in a 50ms loop. If not: uses `barcodeDetector.decodeOnceFromStream(stream, videoRef)` directly. `decodeCropFromVideo` calls `drawCropOnCanvas()` (center-crops the video to the mask area) and `decodeCanvas()` to decode that region. On `NotFoundException`, retries after 50ms; on other errors, rejects. `stop()` calls `stopAsyncDecode()` and `reset()` on ZXing reader.

**3. What part of behavior can be documented from this file?**
When `showMask=true` (useCrop), ZXing only decodes the center mask area, not the full video frame. This is the crop feature: the mask div dimensions are used to calculate the crop region. The `NotFoundException` is suppressed silently during normal scan loops (it means "no barcode found yet" not "error"). Actual errors (camera permission denied, etc.) propagate to `onError`.

**4. Is it user-facing?**
No. Internal detection backend.

**5. What new did you learn from this file?**
The crop functionality uses `drawCropOnCanvas()` which computes the crop dimensions by comparing the video element's client dimensions with the `canvasMiddleMiddleRef` div dimensions. This means the crop is determined by the actual rendered size of the mask overlay div in the DOM — the crop area is visually what the user sees in the mask target.

---

## src/helpers/barcode-detector.ts

**1. Purpose of this file?**
TypeScript type definitions for the browser's native `BarcodeDetector` Web API (not yet standardized, experimental). Also defines the `MxBarcodeReader` interface shared by both reader implementations.

**2. What kind of logic is described in this file?**
Declares `BarcodeFormat`, `DetectedBarcode`, `BarcodeDetector`, `BarcodeDetectorConstructor` interfaces matching the Web API shape. Extends `Window` to include optional `BarcodeDetector` property. Defines `MxBarcodeReader` interface with `start(onSuccess, onError): Promise<void>` and `stop(): void`.

**3. What part of behavior can be documented from this file?**
`DetectedBarcode` includes `boundingBox`, `cornerPoints`, and `rawValue` — the widget only uses `rawValue`. The native API supports 13 format strings (lowercase snake_case: `aztec`, `code_128`, `code_39`, etc.) which differ from the Mendix enum format (UPPERCASE: `AZTEC`, `CODE_128`, etc.) — requiring the format mapping in `barcode-detector-utils.ts`.

**4. Is it user-facing?**
No. Type definitions only.

**5. What new did you learn from this file?**
The `MxBarcodeReader` interface's `start` method uses a callback pattern (not Promises for results) — it takes `onSuccess(data: string)` and `onError(e: Error)` callbacks and manages them internally. The `stop` method is synchronous (no return value) while `start` returns a Promise (for async setup like camera acquisition).

---

## src/helpers/barcode-detector-utils.tsx

**1. Purpose of this file?**
Utility functions for the native `BarcodeDetector` API: availability check, options creation, detector instantiation, frame capture, and barcode detection.

**2. What kind of logic is described in this file?**
`isBarcodeDetectorSupported()`: checks `"BarcodeDetector" in globalThis`. `createBarcodeDetectorOptions()`: maps Mendix format enums to native format strings (uppercase → lowercase with underscores, e.g. `UPC_A` → `upc_a`). When `useAllFormats=true`, no formats are specified in options (uses all browser-supported). `detectBarcodesFromElement()`: guards for null detector, null element, and `readyState < HAVE_CURRENT_DATA` before calling `detect()`. `captureVideoFrame()`: draws video frame to canvas. `setupVideoElement()`: sets `autofocus=true`, `playsInline=true` (Safari fix), `muted=true`.

**3. What part of behavior can be documented from this file?**
If the browser's BarcodeDetector doesn't support a mapped format, it will either silently ignore it or throw — the code doesn't validate supported formats against the options. `detectBarcodesFromElement` returns empty array on any error (catches and logs), making detection failures transparent to the caller. `playsInline=true` is explicitly set for Safari (required for inline video playback on iOS without fullscreen).

**4. Is it user-facing?**
No. Internal utility.

**5. What new did you learn from this file?**
The `RSS_14` format (a Mendix enum value) has no direct mapping in the `formatMap` — it falls through to the default `format.toLowerCase()` → `"rss_14"`. This format is not in the standard BarcodeDetector API format list defined in `barcode-detector.ts` (which lists `pdf417` but not `rss_14`), so RSS-14 detection via the native API may not work on all browsers.

---

## src/helpers/utils.tsx

**1. Purpose of this file?**
Shared utilities for ZXing-based barcode decoding: camera constraints, crop canvas drawing, ZXing hint map creation, and type exports.

**2. What kind of logic is described in this file?**
`mediaStreamConstraints`: `facingMode: "environment"`, resolution `min:1280×720`, `ideal:4096×2160`, `max:4096×2160` (4K). `createHints()`: builds ZXing `DecodeHintType` hints — sets `POSSIBLE_FORMATS` to all 12 formats if `useAllFormats=true`, else maps enum strings to `BarcodeFormat` constants; always enables `ENABLE_CODE_39_EXTENDED_MODE`. `drawCropOnCanvas()`: calculates crop dimensions by comparing video client dimensions with the mask div dimensions, accounting for aspect ratio, then draws the center region of the video onto the canvas. `decodeCanvas()`: creates `HTMLCanvasElementLuminanceSource` → `HybridBinarizer` → `BinaryBitmap` → calls `reader.decodeBitmap()`.

**3. What part of behavior can be documented from this file?**
`ENABLE_CODE_39_EXTENDED_MODE` is always enabled for ZXing, meaning Code 39 Extended Mode (full ASCII) is always active regardless of user configuration. Camera ideal resolution is 4K (4096×2160) — requesting high resolution improves scanning accuracy on capable devices (v2.5.0 changelog note). The crop algorithm is aspect-ratio-aware: if video aspect ratio differs from client, the crop square is sized to fit within the smaller dimension.

**4. Is it user-facing?**
No. Internal utilities.

**5. What new did you learn from this file?**
The `drawCropOnCanvas` function handles aspect ratio mismatch between the video's native resolution and its rendered client size. This is important for high-resolution cameras (e.g. 4K video displayed in a small DOM element) — the crop coordinates are in native video pixels, not CSS pixels, so the conversion is necessary for accurate cropping.

---

## src/hooks/useCustomErrorMessage.ts

**1. Purpose of this file?**
React hook that manages error message state for the scanner — converts an `Error` object to a displayable string and stores it in component state.

**2. What kind of logic is described in this file?**
Returns `[message: string | null, setError: ErrorCb]`. `setError(error)` extracts `error.message` from any `Error` instance, or sets a generic fallback for non-Error throws. State is managed via `useState`.

**3. What part of behavior can be documented from this file?**
Once an error is set, it replaces the scanner view permanently — there is no recovery mechanism or reset. The error message shown to the user is the raw `Error.message` string (not localized or user-friendly). The fallback message for non-Error throws is "Unexpected error in barcode scanner."

**4. Is it user-facing?**
No directly, but its output is rendered as a visible error alert.

**5. What new did you learn from this file?**
The error state is one-way — once set, it cannot be cleared. This means if camera permission is denied, the only way to recover is to reload the page. The error display completely replaces the scanner UI (no retry button, no dismiss).

---

## src/BarcodeScanner.editorConfig.ts

**1. Purpose of this file?**
Design-time configuration for Studio Pro: property visibility rules and structure preview.

**2. What kind of logic is described in this file?**
`getProperties()`: transforms property groups into tabs when `platform === "web"`; hides the `barcodeFormats` object list when `useAllFormats=true`. `getPreview()`: returns an `Image` type structure preview using the barcode scanner SVG icon (dark/light mode variants), 275×275 pixels.

**3. What part of behavior can be documented from this file?**
When `useAllFormats=true`, the `barcodeFormats` list is hidden — developers cannot select specific formats unless they toggle `useAllFormats` to false. The structure preview uses a static SVG icon (not a live preview), showing the barcode scanner concept image. No `check()` function is exported — no validation beyond what the property types enforce.

**4. Is it user-facing?**
No. Design-time only.

**5. What new did you learn from this file?**
`transformGroupsIntoTabs(defaultProperties)` is called for the web platform, converting the property group hierarchy into a tabbed UI in Studio Pro. This is a cosmetic Studio Pro improvement. The `check` function is absent (not exported), meaning there are no design-time validation errors for misconfigured barcode format lists.

---

## src/BarcodeScanner.editorPreview.tsx

**1. Purpose of this file?**
Renders the design canvas preview in Studio Pro using the `BarcodeScannerOverlay` component with a static QR code preview image.

**2. What kind of logic is described in this file?**
`preview()` renders `<BarcodeScannerOverlay>` (the same component used at runtime for layout/mask structure) with the configured dimensions and mask setting, but with a static `<img>` (previewQrCode.svg) inside instead of a live `<video>`. `getPreviewCss()` exports both the runtime and preview SCSS.

**3. What part of behavior can be documented from this file?**
The Studio Pro design canvas shows the actual mask overlay structure (if `showMask=true`) with a representative QR code image inside. The overlay layout and dimensions reflect the configured width/height exactly, giving developers an accurate size preview. The `class` prop uses `props.className` (deprecated) rather than `props.class` — this is intentional for the preview context (Studio Pro compatibility).

**4. Is it user-facing?**
No. Design-time preview only.

**5. What new did you learn from this file?**
The preview reuses `BarcodeScannerOverlay` from the runtime component — this ensures the design canvas preview has the same DOM structure as the runtime widget. This means the mask overlay (four corner markers, background panels) is accurately represented in Studio Pro preview.

---

## src/components/__tests__/BarcodeScanner.spec.tsx

**1. Purpose of this file?**
Unit tests for the `BarcodeScanner` component covering rendering, error handling, and callback wiring.

**2. What kind of logic is described in this file?**
Tests (using snapshot tests): renders video and overlay correctly (with mask); renders without mask; renders error alert when `mediaDevices` API is absent (HTTP context). Behavioral tests: `onDetect` prop is passed as `onSuccess` callback and fires with the scanned value; error from `Error` instance shows in an alert; `NotFoundException` error also shows in alert. Uses snapshot tests for DOM structure validation.

**3. What part of behavior can be documented from this file?**
When `mediaDevices` is undefined (e.g. HTTP context), a hard error renders without any camera interaction. The `onDetect` callback receives the raw scanned string. Both generic errors and `NotFoundException` from ZXing propagate to the UI as visible error alerts. Snapshot tests freeze the overlay DOM structure.

**4. Is it user-facing?**
No. Tests only.

**5. What new did you learn from this file?**
`useReader` is fully mocked in unit tests — all unit tests exercise the component rendering and prop wiring without actually starting a camera stream. The test confirms the `NotFoundException` from ZXing (which normally means "no barcode found yet") is propagated to `onError` when it reaches the component — this is the scenario where ZXing gives up (e.g. component unmounts while scanning).

---

## cypress/integration/BarCodeScanner.spec.js

**1. Purpose of this file?**
E2E test using Cypress that verifies the barcode scanner widget can be opened and the scanner content becomes visible.

**2. What kind of logic is described in this file?**
Single test: clicks `.mx-name-actionButton1` (a button that presumably opens the scanner, e.g. in a modal), waits 1000ms, then asserts `.mx-barcode-scanner-content` is visible. Uses Mendix session logout in `afterEach`.

**3. What part of behavior can be documented from this file?**
The widget is shown/hidden via an action button (not permanently on the page). The test only verifies the scanner container is visible — it does not test actual barcode detection (which would require camera hardware). The `mx-barcode-scanner-content` class is the inner container of `BarcodeScannerOverlay`.

**4. Is it user-facing?**
Tests only, but validates user-visible widget appearance.

**5. What new did you learn from this file?**
The widget appears to be used inside a popup/modal triggered by a button (`mx-name-actionButton1`), not placed directly on a page as a persistent element. This suggests the primary use case is an "open scanner" button that reveals the camera view in a modal dialog. The 1-second wait (`cy.wait(1000)`) is needed for the camera to initialize.

---

## CHANGELOG.md

**1. Purpose of this file?**
Version history for barcode-scanner-web.

**2. What kind of logic is described in this file?**
v2.5.1 (2026-02-09): Added license file and readme with open source dependencies. v2.5.0 (2025-09-15): Added native BarcodeDetector API support; increased ideal image resolution for better performance. v2.4.2 (2024-08-30): Fixed portrait/landscape mode switching bug. v2.4.1 (2024-07-01): Fixed canvas reference bug with crop feature. v2.4.0 (2024-06-11): Added barcode format controls to prevent incorrect scanning. v2.3.1 (2023-09-27): Removed redundant code. v2.3.0 (2023-06-05): Updated icons. v2.2.4 (2023-04-04): Improved 1D barcode recognition with mask. v2.2.3 and earlier (2023): Additional bug fixes.

**3. What part of behavior can be documented from this file?**
The format filter (`barcodeFormats` list) was added in v2.4.0 to prevent incorrect format detection. The native BarcodeDetector API was added as an optional fast-path in v2.5.0. The crop/mask feature for 1D barcodes was improved in v2.2.4.

**4. Is it user-facing?**
No. Developer-facing.

**5. What new did you learn from this file?**
The widget has been in production since at least 2022 (v2.2.x). The native BarcodeDetector API integration is relatively recent (v2.5.0, Sep 2025) and is experimental — the description says "experimental fast scan, fallback to ZXing". The resolution increase in v2.5.0 was specifically for "better performance on higher end devices," suggesting the higher resolution helps the detection algorithms on capable cameras.

---

## Summary of Key Findings

- **Widget identity**: Camera-based barcode/QR code scanner. Scans using device rear camera, writes result to a Mendix string attribute. Version 2.5.1, mature widget with history back to 2022.
- **Context requirement**: `datasource` requires an `EditableValue<string>` — must be placed in a data context (data view or similar) with a writable String attribute.
- **Detection backends**: Two backends: (1) native `BarcodeDetector` Web API (fast, experimental, browser-support-dependent); (2) ZXing library (universal fallback). Default is `native` mode with automatic ZXing fallback if the browser API is unavailable.
- **Format support**: 12 formats: AZTEC, CODE_39, CODE_128, DATA_MATRIX, EAN_8, EAN_13, ITF, PDF_417, QR_CODE, RSS_14, UPC_A, UPC_E. `useAllFormats=true` (default) scans all; `useAllFormats=false` exposes a format list for selection.
- **Camera constraints**: Rear-facing camera (`facingMode: "environment"`), ideal 4K resolution (4096×2160), min 1280×720.
- **Mask/crop**: `showMask=true` (default) shows a visual targeting overlay. For ZXing, this also crops the decoded region to the mask area. Native reader does not use crop — it processes the full frame.
- **Deduplication**: Attribute is only set and action only fired when scanned value changes from current attribute value.
- **Error handling**: Camera API unavailable → hard error (no recovery). Scan error → error replaces scanner view (no retry). Both are shown as Bootstrap danger alerts.
- **Offline capable**: `offlineCapable=true`.
- **E2E tests**: Cypress-based. Single test verifying widget opens and scanner content is visible — no actual barcode detection tested.
- **Action**: `onDetect` action fires after each new successful scan. Combined with the deduplication guard, this means the action fires once per unique barcode detected.
