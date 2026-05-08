# BarcodeScanner

## Purpose

The BarcodeScanner widget opens the device rear camera and continuously scans for barcodes or QR codes, writing each successfully detected value into a writable Mendix string attribute and optionally triggering a configured action. It is intended for workflows where end users need to capture barcode data without manual entry — for example, scanning a product barcode to populate a form field, or scanning a QR code to navigate to a record. The widget supports 12 barcode formats, two detection backends (native browser API and ZXing fallback), and an optional visual targeting mask that also narrows the decode region for ZXing.

## User Scenarios

### [P1] Scan a barcode and write the result to an attribute

**Given** the widget is placed in a data view with a writable String attribute bound to `datasource`, and `onDetect` is configured to run a nanoflow  
**When** the camera view opens and the user positions a valid barcode in the frame  
**Then** the scanned value is written to the attribute and the nanoflow is executed; subsequent scans of the same barcode do not re-trigger the attribute write or the action until a different barcode is detected

#### Edge Cases

- When the camera continuously reads the same barcode on successive frames, the attribute and action are only triggered once — deduplication compares each new value to the current attribute value.
- When `datasource` is not in `Available` status at the time of a successful scan, neither the attribute write nor the action fires.
- When `onDetect` is not configured, attribute write still occurs on a new scan result; the missing action is a silent no-op.

---

### [P2] Use the visual mask to target a specific scan area

**Given** `showMask = true` (default) and `detectionLogic = "zxing"` (or native API not supported)  
**When** the scanner is active  
**Then** a centered target overlay is rendered with corner markers and background panels; the ZXing decoder processes only the region within the target area, not the full frame

#### Edge Cases

- When `detectionLogic = "native"` and the browser's `BarcodeDetector` API is available, the mask is shown visually but the native reader processes the full video frame — cropping is not applied.
- When `showMask = false`, the visual overlay is hidden and ZXing processes the full frame.
- The crop area is computed from the actual rendered size of the mask div in the DOM; if the widget is in a modal that opens with animation, the crop dimensions may be calculated before the layout is fully settled.

---

### [P3] Filter by specific barcode formats

**Given** `useAllFormats = false` and a specific set of formats is selected in `barcodeFormats`  
**When** the scanner is active  
**Then** only barcodes of the selected formats are detected; other barcode types in the camera frame are ignored

#### Edge Cases

- When `useAllFormats = true` (default), the `barcodeFormats` list is hidden in Studio Pro and all 12 supported formats are active.
- For the native BarcodeDetector backend, format strings are mapped to the lowercase underscore form (e.g. `UPC_A` → `upc_a`); formats not in the native API's supported list may be silently ignored by the browser.
- `RSS_14` has no guaranteed mapping in the native API — detection of this format via the native backend may not work on all browsers.
- `ENABLE_CODE_39_EXTENDED_MODE` (full ASCII Code 39) is always enabled for ZXing regardless of format selection.

---

### [P4] Handle camera permission denied or unsupported browser context

**Given** the page is served over HTTP (not HTTPS), or the user denies camera permission  
**When** the widget renders  
**Then** an error alert is shown in place of the camera view; the scanner cannot recover without a page reload

#### Edge Cases

- When `navigator.mediaDevices.getUserMedia` is absent (HTTP or unsupported browser), the error renders before any camera interaction.
- Camera permission denial and other runtime errors produce an error alert that permanently replaces the scanner UI — there is no retry mechanism.
- The error message text is the raw `Error.message` string from the browser, which may not be user-friendly or localized.

---

## Functional Requirements

- FR-001: The system MUST write the scanned barcode value to the `datasource` `EditableValue<string>` attribute on each successful detection.
- FR-002: The system MUST NOT write to `datasource` or execute `onDetect` when the scanned value is identical to the current attribute value (deduplication).
- FR-003: The system MUST NOT write to `datasource` or execute `onDetect` when `datasource.status` is not `Available`.
- FR-004: The system MUST execute the `onDetect` action on each new unique scan result, after the attribute write.
- FR-005: The system MUST use the native `BarcodeDetector` Web API when `detectionLogic = "native"` and the API is available in the browser; otherwise it MUST fall back to ZXing.
- FR-006: The system MUST use ZXing when `detectionLogic = "zxing"`, regardless of native API availability.
- FR-007: The system MUST request the rear-facing camera (`facingMode: "environment"`) with an ideal resolution of 4096×2160 and a minimum of 1280×720.
- FR-008: The system MUST display a visual targeting overlay (mask) with corner markers when `showMask = true`.
- FR-009: When `showMask = true` and ZXing is the active backend, the system MUST decode only the region within the visible mask target area.
- FR-010: When the native `BarcodeDetector` is the active backend, the system MUST decode the full video frame regardless of `showMask` setting.
- FR-011: The system MUST support 12 barcode formats: AZTEC, CODE_39, CODE_128, DATA_MATRIX, EAN_8, EAN_13, ITF, PDF_417, QR_CODE, RSS_14, UPC_A, UPC_E.
- FR-012: When `useAllFormats = false`, the system MUST restrict detection to the formats listed in `barcodeFormats`.
- FR-013: When `useAllFormats = true`, the system MUST attempt to detect all supported formats.
- FR-014: The system MUST render a Bootstrap danger alert in place of the camera view when `navigator.mediaDevices.getUserMedia` is unavailable.
- FR-015: The system MUST render a Bootstrap danger alert and cease camera rendering when a runtime error occurs during scanning.
- FR-016: The widget MUST require a writable String attribute bound to `datasource` (entity context required).
- FR-017: The widget MUST function in offline-enabled Mendix applications (`offlineCapable = true`).

## Props Reference

| Name | Type | Default | Caption | Description |
|------|------|---------|---------|-------------|
| `datasource` | EditableValue\<string\> | _(required)_ | Data source | Writable String attribute that receives the scanned barcode value. Must be in a data context (e.g. data view). |
| `onDetect` | ActionValue | _(none)_ | On detect | Action executed after each new unique scan result, following the attribute write. Supports microflows, nanoflows, page navigation. Optional. |
| `showMask` | boolean | true | Show mask | Renders the visual targeting overlay with corner markers. When active with ZXing, also restricts decoding to the mask region. |
| `useAllFormats` | boolean | true | Use all formats | Enables detection of all 12 supported barcode formats. When false, exposes the `barcodeFormats` list. |
| `barcodeFormats` | list | _(none)_ | Barcode formats | Specific formats to detect when `useAllFormats = false`. One or more of: AZTEC, CODE_39, CODE_128, DATA_MATRIX, EAN_8, EAN_13, ITF, PDF_417, QR_CODE, RSS_14, UPC_A, UPC_E. |
| `detectionLogic` | enum | `"native"` | Detection logic | `"native"`: use browser BarcodeDetector API with ZXing fallback. `"zxing"`: always use ZXing library. |
| `widthUnit` | enum | `"percentage"` | Width unit | `"percentage"` or `"pixels"`. |
| `width` | integer | 100 | Width | Widget width in the selected unit. Default: 100 (percentage). |
| `heightUnit` | enum | `"percentageOfWidth"` | Height unit | `"percentageOfWidth"`, `"pixels"`, or `"percentageOfParent"`. |
| `height` | integer | 75 | Height | Widget height in the selected unit. Default: 75 (% of width). |

## Changelog

### [2.5.1] - 2026-02-09
- Added: License file and readme documenting open source dependencies.

### [2.5.0] - 2025-09-15
- Added: Native `BarcodeDetector` Web API support as an experimental fast-scan backend with automatic ZXing fallback.
- Improved: Increased ideal camera resolution to 4K (4096×2160) for better detection performance on capable devices.

### [2.4.2] - 2024-08-30
- Fixed: Portrait/landscape mode switching bug.

## Open Questions

> Could not be determined from source code alone — requires human review

- [ ] Is RSS-14 format detection functional with the native BarcodeDetector API? The format has no standard mapping in the BarcodeDetector spec and may be silently unsupported on some browsers.
- [ ] Is there a plan to add error recovery (retry button) for camera permission denial or runtime scan errors? Currently the error state is unrecoverable without a page reload.
- [ ] Should the error message displayed to end-users be localized or replaced with a friendlier string? Currently the raw `Error.message` from the browser is shown directly.
- [ ] The native reader does not apply crop even when `showMask = true`. Is this intended behavior or a known limitation? The mask provides accurate visual feedback but does not constrain native detection.
- [ ] What is the intended behavior when the widget is used on a device without a rear camera (e.g. a desktop browser)? `facingMode: "environment"` may fall back to a front camera silently or fail depending on browser behavior.
