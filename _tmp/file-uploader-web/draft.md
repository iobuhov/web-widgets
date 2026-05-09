# Draft: file-uploader-web

Worker: worker | Task: EX-028 | Date: 2026-05-09

---

## FileUploader.xml

1. **Purpose:** Widget descriptor for the File Uploader Mendix pluggable widget. Declares the widget identity (`com.mendix.widget.web.fileuploader.FileUploader`), platform (`Web`), offline capability, entity context requirement, and Studio/StudioPro categorization ("Images, videos & files"). Defines all configurable properties exposed to Studio Pro.

2. **Logic:** No runtime logic — this is a declarative schema. Defines three property groups: General (upload mode, datasource associations, read-only, create actions, allowed formats, file count/size limits), Texts (14 localizable string templates for UI messages), and Advanced (object creation timeout, success/failure callbacks, custom buttons list).

3. **Behavioral constraints documented:** `uploadMode` toggles between "files" and "images", which controls which datasource (`associatedFiles` vs `associatedImages`) and create action (`createFileAction` vs `createImageAction`) are active. `allowedFileFormats` is optional (no restriction if empty) and supports "simple" (predefined type) or "advanced" (MIME + extension list) modes. `maxFilesPerUpload` is an expression of type Integer (dynamically settable, not a static integer). `maxFileSize` is a static integer in MB. `readOnlyMode` disables upload/creation properties. `enableCustomButtons` gates the `customButtons` list. Custom buttons support `buttonIsDefault` (at most one allowed — triggers on file entry click) and `buttonIsVisible` (expression-based visibility).

4. **User-facing:** Yes. Defines all end-user-visible properties and their translations (English and Dutch provided). Message templates use `###` as the substitution placeholder for dynamic values (e.g., file count, size).

5. **New learned:** The widget needs entity context (`needsEntityContext="true"`) and is offline-capable (`offlineCapable="true"`). Default datasource paths are pre-wired (e.g., `FileUploader.FileUploadContext/FileUploader.UploadedFile_FileUploadContext/FileUploader.UploadedFile`). The create actions have default nanoflow values (`FileUploader.ACT_CreateUploadedFileDocument`, etc.) making zero-config setup possible.

---

## typings/FileUploaderProps.d.ts

1. **Purpose:** TypeScript type declarations auto-generated from FileUploader.xml. Provides strongly-typed interfaces for both the runtime container props (`FileUploaderContainerProps`) and the Studio Pro preview props (`FileUploaderPreviewProps`).

2. **Logic:** Pure type definitions. `AllowedFileFormatsType` captures per-format config (configMode, predefinedType, mimeType, extensions, typeFormatDescription as `DynamicValue<string>`). `CustomButtonsType` has per-mode list actions (`buttonActionFile?: ListActionValue`, `buttonActionImage?: ListActionValue`) and `buttonIsVisible: DynamicValue<boolean>`. `maxFilesPerUpload` is `DynamicValue<Big>` (expression result is Big.js decimal).

3. **Behavioral constraints:** `createFileAction` and `createImageAction` are optional (`ActionValue | undefined`), enabling detection of misconfigured widgets. The upload success/failure callbacks (`onUploadSuccessFile`, `onUploadFailureFile`, `onUploadSuccessImage`, `onUploadFailureImage`) are all `ListActionValue` — i.e., they receive the individual file/image object as context. Custom button visibility is a `DynamicValue<boolean>` evaluated per file entry.

4. **User-facing:** Internal/developer-facing type file; not directly user-visible but shapes all runtime behavior.

5. **New learned:** `maxFilesPerUpload` is `DynamicValue<Big>` (not `DynamicValue<number>`), so the expression result is a Big.js object requiring `.toNumber()` conversion at runtime. The preview types flatten DynamicValues to primitive strings/booleans for Studio Pro rendering.

---

## src/FileUploader.tsx (main entry point)

1. **Purpose:** Root React component exported as the widget entry point. Wraps the widget in a `TranslationsStoreProvider` context before rendering the main `FileUploaderRoot`.

2. **Logic:** Minimal orchestration — instantiates the `TranslationsStoreProvider` (which initializes the `TranslationsStore` MobX store and makes it available via React context), then renders `FileUploaderRoot` with all props passed through.

3. **Behavioral constraints:** The translation context is established at the top level, so all child components can access localized messages via `useTranslationsStore()` without prop drilling. The widget is entirely user-context-dependent (requires entity context from its parent page).

4. **User-facing:** Yes — this is the widget's public surface. All props flow through here.

5. **New learned:** The separation of `TranslationsStoreProvider` at this level (not inside `FileUploaderRoot`) ensures the MobX translations store is initialized once per widget instance and available to all components including `Dropzone` and `ActionsBar`.

---

## src/components/FileUploaderRoot.tsx

1. **Purpose:** Main observable React component that renders the widget UI: conditionally shows the Dropzone (when not read-only and upload limit not reached), then renders the list of file entries.

2. **Logic:** Uses `useRootStore(props)` to obtain/update the `FileUploaderStore` MobX store. On file drop, delegates to `rootStore.processDrop(acceptedFiles, fileRejections)`. The Dropzone is hidden when `rootStore.isReadOnly` is true or `rootStore.isFileUploadLimitReached` is true. The file list iterates `rootStore.files` (observable array of `FileStore` instances), rendering each as `FileEntryContainer`. Custom buttons are passed to `FileEntryContainer` only when `props.enableCustomButtons` is true.

3. **Behavioral constraints:** When `isFileUploadLimitReached` is true, the dropzone becomes `disabled` (still visible but not interactive, not hidden). Custom buttons replace the default download/remove buttons when enabled — not supplemented. The `acceptFileTypes` passed to Dropzone are derived from `rootStore.acceptedFileTypes` via `prepareAcceptForDropzone` (converts internal format to react-dropzone's `accept` object shape).

4. **User-facing:** Yes — the root UI component.

5. **New learned:** The dropzone is disabled (not hidden) when the upload limit is reached, so users can see it but cannot interact with it. MobX `observer` wrapping ensures the component re-renders reactively when `rootStore.files`, `rootStore.isReadOnly`, or `rootStore.isFileUploadLimitReached` changes.

---

## src/components/Dropzone.tsx

1. **Purpose:** Renders the drag-and-drop upload area using `react-dropzone`. Displays contextual messages depending on drag state and shows error warnings below the zone.

2. **Logic:** Uses `useDropzone` hook with `onDrop`, `maxSize`, `maxFiles`, `accept`, and `disabled` props. Derives the display message from drag state: `isDragAccept` → accepted message, `isDragReject` → rejected message, otherwise → idle message. Three CSS states: `active` (drag accept), `warning` (drag reject), and base (idle). Warning messages from the store (e.g., "too many files") appear below the dropzone in a separate `dropzone-message` div.

3. **Behavioral constraints:** `maxSize` is passed as `maxSize || undefined` — if 0, it is treated as unlimited (no size check by react-dropzone). `disabled` prop fully disables the dropzone (no input rendered, no interaction). The warning message is separate from drag-state messaging: it persists until cleared by the store (e.g., after a new valid drop resets it).

4. **User-facing:** Yes — primary upload interaction surface.

5. **New learned:** The `file-icon` div inside the dropzone is a purely decorative CSS-rendered icon (no img/svg element). The hidden `<input>` is not rendered at all when disabled, which prevents browser file dialogs.

---

## src/components/FileEntry.tsx

1. **Purpose:** Renders a single file entry in the uploaded files list. Shows file icon (or image thumbnail), file name, file size, upload progress, status messages, and action buttons.

2. **Logic:** `FileEntryContainer` (observer) reads from a `FileStore`. Returns `null` for `missing` status files (silently removes them from the DOM). Finds the default custom button action and wires click/keyboard handlers. `FileEntry` (pure component) renders the entry structure: icon/thumbnail, name, size, actions bar, progress bar, and upload info. Files in `removedFile`/`removedAfterError` state get a `removed` CSS class; `validationError` files get an `invalid` class.

3. **Behavioral constraints:** The default custom button action handles both click (`onClick`) and keyboard (`Enter`/`Space` key via `onKeyDown`). `tabIndex=0` is set only when a default action exists, making entries focusable only when interactive. File size shows blank (no element) when `size === -1` (unknown/not yet fetched). Thumbnail image uses empty `alt=""` (decorative).

4. **User-facing:** Yes — the file list UI.

5. **New learned:** The `fileSize()` utility is used for display. When `fileStatus === "missing"`, the entry renders `null` (not a placeholder or error state), so the entry disappears from the list silently. The progress bar is visible only when `fileStatus === "uploading"`.

---

## src/components/ActionsBar.tsx

1. **Purpose:** Renders per-file action buttons. In default mode (no custom buttons), shows download and remove buttons. In custom button mode, renders the configured custom buttons with expressions-based visibility.

2. **Logic:** When `actions` prop is undefined (default mode), renders `DefaultActionsBar` with download (`getDownloadUrl()` + open in new window) and remove (`store.remove()`) buttons. When custom buttons are configured and `store.canExecuteActions` is true, renders each button that has `buttonIsVisible.value === true` (falsy buttons are skipped, returning `null`). Each custom button uses `buttonActionImage ?? buttonActionFile` as the action.

3. **Behavioral constraints:** Download opens in a new window named `"mendix_file"`, with `?target=window` query param appended to the URL. The default download button is disabled when `!store.canDownload` (only enabled for `done` and `existingFile` statuses). The default remove button is disabled when `!store.canRemove`. Custom buttons are only rendered when `store.canExecuteActions` (i.e., `_objectItem` is set) — so they appear only after the file object is created in Mendix.

4. **User-facing:** Yes — action buttons for each file entry.

5. **New learned:** The download URL is fetched asynchronously via `store.getDownloadUrl()` each time the button is clicked (not pre-cached). File download opens with `target=window` parameter which instructs Mendix to serve the file inline/for download. Custom action visibility is evaluated from a DynamicValue expression per file item.

---

## src/stores/FileUploaderStore.ts

1. **Purpose:** Central MobX store for the file uploader widget. Manages the list of `FileStore` instances, processes datasource updates, coordinates object creation, and enforces upload constraints.

2. **Logic:** Initializes `ObjectCreationHelper` and `DatasourceUpdateProcessor`. On `updateProps`, syncs max file limits, translations, and processes the active datasource. `processDrop` validates the create action availability and file count, then creates `FileStore` instances for each accepted and rejected file. `isFileUploadLimitReached` counts active files (excludes `missing`, `removedFile`, `validationError`) against `maxFilesPerUpload`. In images mode, `allowedFileFormats` config is ignored — `getImageUploaderFormats()` hardcodes the allowed image types.

3. **Behavioral constraints:** Max file size in the store is stored both in MiB (`_maxFileSizeMiB`) and bytes (`_maxFileSize`). `maxFilesPerUpload` returns 0 when the expression is unavailable or resolves to 0, which means unlimited. File count check counts files by their status, not by object ID — files with status `missing`, `removedFile`, `validationError` do not count toward the limit. In images upload mode, file format restrictions are fixed to `anyImageFile` (PNG, JPEG, GIF, BMP, WebP, SVG) regardless of `allowedFileFormats` configuration.

4. **User-facing:** No — internal store. Drives the observable UI components.

5. **New learned:** `processDrop` checks `objectCreationHelper.canCreateFiles` before processing any file — if the create action is unavailable, it shows the `unavailableCreateActionMessage` and returns early without adding any files. Rejected files (too large, wrong type) are still added to the files list as `validationError` entries so the user sees why their file was rejected.

---

## src/stores/FileStore.ts

1. **Purpose:** MobX store for a single file entry. Manages the upload lifecycle state machine, file metadata, thumbnail fetching, and download/remove operations.

2. **Logic:** State machine via `fileStatus`: `new → uploading → done` (success) or `new → uploading → uploadingError` (failure). `existingFile` → `missing` (object deleted from datasource) or `removedFile` (user removes). `uploadingError` → `removedAfterError` (object goes missing after error). Upload flow: request object creation from `objectCreationHelper` → call `mx.data.saveDocument` → fetch full MxObject → update status. Remove flow: call `mx.data.remove` → mark `removedFile`.

3. **Behavioral constraints:** `canRemove` is true only for `existingFile` (when not read-only) and `done` statuses. `canDownload` is true for `done` and `existingFile`. `mimeType` is derived from filename via `mime-types.lookup()`, defaulting to `application/octet-stream`. In images mode, thumbnail is fetched from `fetchImageThumbnail` (server-side thumbnail URL); for newly-dropped image files before upload completes, `URL.createObjectURL` is used for the preview. The `imagePreviewUrl` computed property uses object URL for the local file preview and switches to the server thumbnail after upload.

4. **User-facing:** No — drives the `FileEntryContainer` component.

5. **New learned:** `FileStore` uses a module-level incrementing `fileKey` counter for React list keys (not object IDs), ensuring stable keys even before Mendix object IDs are assigned. Static factory methods `existingFile`, `newFile`, `newFileWithError` provide named constructors with clear intent. The `title` property falls back from MxObject name → File.name → `"..."` as MxObject is loaded asynchronously.

---

## src/stores/TranslationsStore.ts

1. **Purpose:** MobX store holding all localized UI message strings. Provides a `get(key, ...substitutions)` method for string template interpolation.

2. **Logic:** On construction and each `updateProps`, scans all props whose key ends with `"Message"` and stores their `DynamicValue<string>.value` (or `"..."` while loading). The `get` method replaces `###` placeholders with positional substitution strings.

3. **Behavioral constraints:** Any prop key ending in `"Message"` is treated as a translation — this is a naming convention contract between the XML and the store. While a `DynamicValue<string>` is loading, the message displays as `"..."`. Missing keys log a console error and return `"<...>"`.

4. **User-facing:** No — internal store. All user-visible strings pass through this store.

5. **New learned:** The `###` substitution pattern (sequential, not named) is used throughout error messages to inject dynamic values like file count or max size. The store is observable so UI components react to translation changes automatically.

---

## src/utils/ObjectCreationHelper.ts

1. **Purpose:** Manages the async object-creation pipeline — coordinates the Mendix nanoflow that creates file/image objects, serializes creation requests, and enforces a timeout for misconfigured actions.

2. **Logic:** Maintains a queue of pending Promise resolvers (`currentWaiting`). When `enable()` is called (after datasource first load), it begins processing the queue by calling the nanoflow action. Each call to `request()` adds a resolver to the queue. When the datasource update delivers a new empty object (`processEmptyObjectItem`), the first waiting resolver is fulfilled. A `setTimeout` guards against the nanoflow failing to create an object within `actionTimeoutInSeconds` — on timeout, all waiting promises reject.

3. **Behavioral constraints:** Only one object creation is in-flight at a time (timer guards re-entry). Object creation is blocked until `enable()` is called, which is triggered only after the datasource initial load (`existingItemsLoaded`). This prevents race conditions on newly-created context objects. On upload failure, the widget calls `onUploadFailure` action on the file's ObjectItem (default: delete nanoflow). On upload success, it calls `onUploadSuccess` action.

4. **User-facing:** No — internal orchestration.

5. **New learned:** Unit tests confirm: creation action fires only once per in-flight request (serialized); all pending requests fail on timeout; requests queued before `enable()` are held until datasource loads. The `actionTimeoutInSeconds` default is 10 (configurable via `objectCreationTimeout` prop).

---

## src/utils/DatasourceUpdateProcessor.ts

1. **Purpose:** Tracks the lifecycle of Mendix ObjectItems in the datasource, distinguishing between existing (already-uploaded) files, new (just-created empty) objects, and missing (deleted) objects across datasource re-renders.

2. **Logic:** First call with `status === "available"` triggers `loaded` callback and calls `processExisting` for all current items. Subsequent updates compute set differences: items in current but not in seen → call `processNew` or `processExisting` based on `fileHasContents()`; items in seen but not in current → call `processMissing`.

3. **Behavioral constraints:** `fileHasContents` determines whether a new object already has file content (was uploaded before this widget instance loaded) vs. is freshly created and waiting for upload. An object with contents is treated as existing; without contents it is treated as new (triggers `objectCreationHelper.processEmptyObjectItem`). The `loaded` callback fires exactly once, enabling the object creation queue.

4. **User-facing:** No — internal utility.

5. **New learned:** Unit tests confirm: `loaded` is called exactly once; `processExisting` fires for all initial items; subsequent updates correctly identify new/missing/existing items; `processNew` is called for objects without contents (empty, freshly-created object).

---

## src/utils/parseAllowedFormats.ts

1. **Purpose:** Parses the `allowedFileFormats` prop array into a normalized `FileCheckFormat[]` structure suitable for both the react-dropzone `accept` object and user-facing format descriptions.

2. **Logic:** Two modes: "simple" (looks up from `predefinedFormats` map) and "advanced" (parses MIME type and extension list, merges entries by description key). MIME type must match `type/subtype` pattern; empty MIME becomes `dummy/mime` (a sentinel that react-dropzone effectively ignores, allowing extension-only matching). Extensions must start with `.` and contain only valid filename characters (no `/ \ ? * < > | : " .` in the extension body).

3. **Behavioral constraints:** Special characters in extensions (dashes, plus signs) are supported (e.g., `.tar-gz`, `.c++`). Extensions with dots in the middle are rejected (e.g., `.config.json`). Extensions without a leading dot are rejected. Multiple formats with the same description key are merged into a single `FileCheckFormat` entry with combined entries. Advanced mode formats with the same description key but different MIME types accumulate as additional entries (not overwritten).

4. **User-facing:** No — internal parsing. Results used by Dropzone `accept` and error messages.

5. **New learned:** Unit tests confirm: duplicate description keys merge entries; mime-only formats (empty extensions) work; extension-only formats (empty MIME → `dummy/mime`) work; special chars `.tar-gz`, `.c++` are valid; invalid MIME format throws; extension without dot throws. This was fixed in v2.3.0.

---

## src/utils/predefinedFormats.ts

1. **Purpose:** Defines the catalog of predefined file format options available in "simple" mode. Maps each `PredefinedTypeEnum` value to its MIME type entries and human-readable description.

2. **Logic:** Static lookup table. Each format specifies an array of `[mimeType, extensions[]]` pairs. `anyImageFile` covers PNG, JPEG, GIF, BMP, WebP, SVG with explicit extensions. `anyAudioFile` and `anyVideoFile` use wildcard MIME (`audio/*`, `video/*`). `anyTextFile` uses `text/*`. `zipArchiveFile` covers both `application/x-zip-compressed` and `application/zip`.

3. **Behavioral constraints:** `pdfFile` uses only MIME (no extensions), relying on browser MIME detection. `plainTextFile` specifies `.txt` extension alongside `text/plain` MIME. Image mode uses this catalog exclusively (`getImageUploaderFormats()` returns `[predefinedFormats.anyImageFile]`).

4. **User-facing:** Indirectly — the `description` field appears in error messages ("supported formats are...") and in the Studio Pro property panel.

5. **New learned:** `anyImageFile` accepts SVG (`image/svg+xml`) in addition to raster formats. This means image-mode uploads accept SVG files, which may be unexpected for use cases expecting only photographs.

---

## src/utils/mx-data.ts

1. **Purpose:** Low-level wrapper around the Mendix client-side `mx.data` API. Provides Promise-based functions for file save, object removal, MxObject fetch, document URL generation, and image thumbnail fetch.

2. **Logic:** All functions access `(window as any).mx.data.*` — the Mendix client JavaScript API. `saveFile` uses `mx.data.saveDocument` to upload binary content. `removeObject` uses `mx.data.remove` by GUID. `fetchDocumentUrl` uses `mx.data.getDocumentUrl` with `changedDate` for cache-busting. `fetchImageThumbnail` fetches the thumbnail URL then calls `mx.data.getImageUrl` to get the actual image content URL. `fileHasContents` reads the `HasContents` attribute directly from the internal object symbol property.

3. **Behavioral constraints:** `fileHasContents` uses a symbol property access (`Object.getOwnPropertySymbols(item)[0]`) — a non-public implementation detail of Mendix ObjectItem. `fetchDocumentUrl` passes `false` as the thumbnail flag; `fetchImageThumbnail` passes `true`. The `changedDate` attribute is used as a cache-buster in document URLs.

4. **User-facing:** No — internal Mendix API bridge.

5. **New learned:** `fetchDocumentUrl` calls `get("changedDate")` on the MxObject (not `get2`), while `FileStore.size` uses `get2("Size")`. The difference between `get` and `get2` is a Mendix API distinction (likely `get` returns raw value, `get2` returns typed Big/string). Document URLs opened in the browser use `?target=window` to trigger download behavior.

---

## src/utils/fileSize.ts

1. **Purpose:** Formats a raw byte count into a human-readable string with appropriate unit (B, KB, MB, GB, etc.) using 1024-based divisions.

2. **Logic:** Iterates through unit array dividing by 1024 until below threshold or at maximum unit. Precision: two decimal places for values < 10, one decimal place for values < 100 (when fractional), integer otherwise. Returns empty string for negative sizes.

3. **Behavioral constraints:** Uses KB/MB/GB labels (not KiB/MiB/GiB) even though the underlying math is 1024-based — a user-familiarity choice explicitly noted in the code comment and CHANGELOG (v2.2.1). Negative size (-1 sentinel for unknown) displays as blank, not "unknown".

4. **User-facing:** Yes — size is displayed in the file entry UI.

5. **New learned:** Unit tests confirm the precision rules: 5.009 KB → "5.01 KB", 85.4 KB → "85.4 KB", 100.91 KB → "100 KB". The function handles petabytes and beyond (PB, EB, ZB, YB defined in the units array).

---

## src/FileUploader.editorConfig.ts

1. **Purpose:** Studio Pro design-time configuration — defines property visibility rules, validation checks, and custom caption generation for the widget in Studio Pro's property panel.

2. **Logic:** `getProperties` hides/shows properties based on `uploadMode` and `readOnlyMode`: in files mode, `associatedImages` and `createImageAction` are hidden; in images mode, `associatedFiles`, `createFileAction`, and `allowedFileFormats` are hidden. In read-only mode, additional properties (create actions, format restrictions, file limits, timeout) are hidden. Custom buttons: in files mode, `buttonActionImage` is hidden per button; in images mode, `buttonActionFile` is hidden. `check` validates allowed format configuration and enforces the one-default-button constraint. `getCustomCaption` shows `"File uploader (files)"` or `"File uploader (images, read only)"` in the Studio Pro canvas.

3. **Behavioral constraints:** `processAllowedFormats` sets `objectHeaders: ["File type", ""]` for the formats list and custom `captions` per format entry — this is the Studio Pro table display. Validation for allowed formats runs `parseAllowedFormats` and surfaces parse errors as Studio Pro problems. Only one custom button can have `buttonIsDefault: true` — enforced by `check`.

4. **User-facing:** Studio Pro designer-facing, not end-user-facing.

5. **New learned:** The `getPreview` function is commented out — the widget uses the default Studio Pro preview (no custom canvas rendering). `getCustomCaption` is user-accessible in the Studio Pro canvas, showing the current mode configuration.

---

## src/utils/__tests__/parseAllowedFormats.spec.ts

1. **Purpose:** Unit tests for `parseAllowedFormats` — validates correct parsing, merging, and error-throwing behavior.

2. **Logic:** Tests: correct advanced format parsing with MIME+extensions; duplicate description key merging (separate entries, not overwritten); incorrect MIME format throws; special character extensions (`.tar-gz`, `.js-map`, `.c++`) are accepted; extensions without leading dot throw; extension with interior dot (`.config.json`) throws.

3. **Behavioral constraints confirmed:** Advanced mode with same description key merges into single `FileCheckFormat`; empty MIME → `dummy/mime`; empty extensions → `[]`; multiple MIME types for same description accumulate as separate `entries` (not merged by MIME). Special chars in extension body (dash, plus) are explicitly validated.

4. **User-facing:** No — test file.

5. **New learned:** The test for `.tar-gz`, `.js-map`, `.c++` was added in v2.3.0 (CHANGELOG confirms "improved file extension validation to allow special characters"). This is a regression test for a previously failing case.

---

## src/utils/__tests__/DatasourceUpdateProcessor.spec.ts

1. **Purpose:** Unit tests for `DatasourceUpdateProcessor` — validates datasource lifecycle callbacks.

2. **Logic:** Tests: `loaded` not called while datasource is loading; `loaded` called once after first successful load; `processExisting` called for all initial items; subsequent updates correctly invoke `processMissing`, `processExisting` (for items with contents), `processNew` (for empty items); `loaded` not called again on updates.

3. **Behavioral constraints confirmed:** `fileHasContents` mock controls whether new items route to `processExisting` or `processNew`. Object `D` (no contents) → `processNew`; object `E` (has contents) → `processExisting`; object `B` (in seen but not current) → `processMissing`.

4. **User-facing:** No — test file.

5. **New learned:** The processor correctly handles mixed updates (new empty objects, new objects with contents, and missing objects) in a single `processUpdate` call.

---

## src/utils/__tests__/ObjectCreationHelper.spec.ts

1. **Purpose:** Unit tests for `ObjectCreationHelper` — validates serialized creation, timeout behavior, and enable-gate behavior.

2. **Logic:** Tests: only one creation action fires at a time (serialized); timeout rejects all pending promises after `actionTimeoutInSeconds * 1000` ms; requests queued before `enable()` are held; after `enable()`, creation proceeds in order.

3. **Behavioral constraints confirmed:** Creation is strictly serialized — second request fires only after first resolves. Timeout of 11 seconds (with 10-second configured timeout) rejects all waiting requests. Requests created before `enable()` fire immediately when `enable()` is called.

4. **User-facing:** No — test file.

5. **New learned:** The timeout test uses `jest.useFakeTimers()` with `jest.advanceTimersByTime(11000)` — confirms the timeout implementation is timer-based (not Promise-based).

---

## src/utils/__tests__/fileSize.spec.ts

1. **Purpose:** Unit tests for `fileSize` utility — validates unit selection, precision rules, and edge cases.

2. **Logic:** Tests: 0 → "0 B"; 1 → "1 B"; 1024 → "1 KB"; 1024² → "1 MB"; 1024³ → "1 GB"; petabyte via fractional power; -1 → ""; precision at various thresholds.

3. **Behavioral constraints confirmed:** Two decimal places for size < 10 (when fractional); one decimal for < 100 (when fractional); integer otherwise. The function uses KB (not KiB) labels with 1024 divisors.

4. **User-facing:** No — test file.

5. **New learned:** `fileSize(1024 ** 5.0001)` returns `"1 PB"` — confirms the unit progression reaches PB. `fileSize(1023)` → `"1023 B"` (stays in bytes, not promoted to 0.999 KB). `fileSize(1024 * 1024 - 1)` → `"1023 KB"`.

---

## CHANGELOG.md

1. **Purpose:** Version history for the file-uploader-web widget following Keep a Changelog / SemVer conventions.

2. **Logic:** Documents changes across all releases from v1.0.1 (initial release, 2024-12-19) through v2.4.2 (2026-04-23).

3. **Key version findings:**
   - **v1.0.1 (2024-12-19):** Initial release — drag-and-drop and file selection dialog.
   - **v1.0.2 (2025-02-14):** Fixed multi-file upload on newly-created context objects; fixed unsupported image format uploads in image mode; added timeout for file/image creation action; added read-only mode; improved misconfigured action detection.
   - **v1.0.3 (2025-02-28):** Fixed long filename overflow.
   - **v2.0.0 (2025-03-14):** `createFileAction`/`createImageAction` now pre-configured with default nanoflows.
   - **v2.1.0 (2025-04-16):** Minimum Studio Pro version raised to 10.21 (Mendix 11 support).
   - **v2.2.0 (2025-05-07):** `associatedFiles`/`associatedImages` pre-configured with default entities; custom buttons feature added; dropzone hover color fix.
   - **v2.2.1 (2025-05-28):** File size units corrected to KB/MB/GB (was Kb/Mb/Gb).
   - **v2.2.2 (2025-07-01):** Fixed image thumbnail reload on page refresh.
   - **v2.3.0 (2025-08-15):** Extension validation now allows special characters (dashes, plus signs); error messages clarified; fixed file count exceeding limit on page refresh; `maxFilesPerUpload` changed from static integer to expression.
   - **v2.4.0 (2025-12-10):** Added `onUploadSuccessFile`/`onUploadFailureFile`/`onUploadSuccessImage`/`onUploadFailureImage` action callbacks.
   - **v2.4.1 (2026-02-12):** Added missing Dutch translations.
   - **v2.4.2 (2026-04-23):** Fixed Download button on Mendix 11.8.

4. **User-facing:** Yes — changelog is developer/admin-facing documentation.

5. **New learned:** The widget shipped rapidly in early 2025 (3 patch releases in 2 months after initial). The `maxFilesPerUpload` changing from static int to expression in v2.3.0 explains why the typing uses `DynamicValue<Big>`. The Download button issue on Mendix 11.8 (v2.4.2) indicates platform-version-specific behavior around document serving.
