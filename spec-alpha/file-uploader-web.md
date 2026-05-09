# FileUploader

## Purpose

The File Uploader widget enables end-users to upload files or images to Mendix via a drag-and-drop zone and/or file-selection dialog. It operates in one of two modes — **files** (generic documents) or **images** (raster images and SVG) — determined at design time. The widget coordinates asynchronous object creation through a configured nanoflow, tracks per-file upload state, and displays the resulting file list with download and removal actions. It is suited for use cases requiring user-driven file attachment to a Mendix entity, such as document management, image galleries, or form attachments.

---

## User Scenarios

### [P1] Upload one or more files via drag-and-drop

**Given** the widget is in files mode, not read-only, and the upload limit has not been reached  
**When** the user drags one or more files onto the dropzone and releases  
**Then** each accepted file is added to the file list with status `uploading`; on success the status transitions to `done` and the file is available for download or removal; rejected files (wrong type or too large) appear in the list as validation errors showing the rejection reason

#### Edge Cases
- Dropping files when the limit is already reached: the dropzone is visible but disabled; dropping has no effect.
- Dropping more files than the remaining quota in a single drop: all files in the batch are rejected and no upload occurs; the user sees the "too many files" warning.
- The configured create action is unavailable (misconfigured nanoflow): the widget shows `unavailableCreateActionMessage` and no files are added.
- Object creation does not complete within `objectCreationTimeout` seconds (default 10): all pending uploads are rejected with a timeout error.

### [P2] Upload files via the file-selection dialog

**Given** the widget is not read-only and the upload limit has not been reached  
**When** the user clicks the dropzone area (or the default custom button, if configured)  
**Then** the browser file-selection dialog opens; upon selection, accepted files are added and uploaded as in P1

#### Edge Cases
- Disabled dropzone (limit reached or read-only): the hidden `<input>` is not rendered; the browser dialog cannot be triggered.

### [P3] Download an uploaded file

**Given** a file entry has status `done` or `existingFile`  
**When** the user clicks the download button  
**Then** the file's download URL is fetched and opened in a new browser window named `mendix_file` with `?target=window`, triggering download or inline display per browser settings

#### Edge Cases
- Download button is disabled for files with any other status (uploading, uploadingError, removedFile, validationError, missing).

### [P4] Remove an uploaded file

**Given** a file entry has status `existingFile` and the widget is not read-only, or the file has status `done`  
**When** the user clicks the remove button  
**Then** `mx.data.remove` is called on the file's MxObject; the entry transitions to `removedFile` and is visually marked as removed

#### Edge Cases
- Files in `uploadingError` state: if the associated MxObject disappears (datasource update), the entry transitions to `removedAfterError` and displays a removed state.
- Files with status `missing` (deleted externally between renders) render `null` — they disappear silently from the list without a removed state.

### [P5] Images-only upload (image mode)

**Given** the widget is in images mode  
**When** the user drops or selects files  
**Then** only files matching the fixed image format list (PNG, JPEG, GIF, BMP, WebP, SVG) are accepted, regardless of the `allowedFileFormats` configuration; before the upload completes, a local object-URL preview is shown as the thumbnail; after upload the server-side thumbnail URL replaces it

#### Edge Cases
- SVG files are accepted in images mode — this may be unexpected for use cases expecting only raster photographs.

### [P6] Read-only mode

**Given** `readOnlyMode` is enabled  
**When** the widget is rendered  
**Then** the dropzone is not rendered; no upload, create, or remove actions are available; the file list is shown with download-only access

---

## Functional Requirements

- **FR-001**: The system MUST support two upload modes: `files` (generic documents) and `images` (images only). The active mode is set at design time via `uploadMode`.
- **FR-002**: In files mode, the system MUST use `associatedFiles` as the datasource and `createFileAction` as the object-creation nanoflow. In images mode, the system MUST use `associatedImages` and `createImageAction`.
- **FR-003**: The system MUST serialize object creation requests: only one creation nanoflow call is in-flight at a time; additional requests are queued and fulfilled as each created empty object is delivered by the datasource.
- **FR-004**: The system MUST enforce `objectCreationTimeout` (default 10 s): if the datasource does not deliver the created object within the configured timeout, all pending creation promises MUST reject and the affected file entries MUST show an error state.
- **FR-005**: Object creation MUST be blocked until the datasource initial load completes (`existingItemsLoaded`), preventing race conditions on newly-created context objects.
- **FR-006**: The system MUST display rejected files (failed type or size validation) in the file list as `validationError` entries so the user can see why their file was rejected.
- **FR-007**: The system MUST enforce `maxFilesPerUpload` (a dynamic Integer expression; 0 = unlimited). Files already in statuses `missing`, `removedFile`, or `validationError` MUST NOT count toward the limit.
- **FR-008**: When the upload limit is reached, the dropzone MUST be rendered in a disabled state (visible but non-interactive); it MUST NOT be hidden.
- **FR-009**: `maxFileSize` (static integer, MB) MUST be passed to react-dropzone. A value of 0 MUST be treated as no size limit.
- **FR-010**: `allowedFileFormats` MUST be ignored in images mode; the fixed image format list (`anyImageFile`) MUST be used instead.
- **FR-011**: `allowedFileFormats` supports two sub-modes: `simple` (predefined format enum) and `advanced` (MIME type + extension list). Advanced entries with the same description key MUST be merged into a single `FileCheckFormat` entry.
- **FR-012**: Extension validation in advanced mode MUST allow special characters (dashes, plus signs) in the extension body (e.g., `.tar-gz`, `.c++`). Extensions without a leading dot or containing interior dots MUST be rejected.
- **FR-013**: In custom button mode (`enableCustomButtons = true`), custom buttons MUST replace (not supplement) the default download and remove buttons. At most one custom button MAY have `buttonIsDefault = true`; that button's action fires on file entry click and keyboard activation.
- **FR-014**: Custom button visibility is evaluated per file entry from a `DynamicValue<boolean>` expression. Custom buttons are rendered only when the file's MxObject is available (`canExecuteActions`).
- **FR-015**: On upload success, the system MUST call `onUploadSuccessFile` (or `onUploadSuccessImage`) list action on the file's ObjectItem, if configured. On upload failure, the system MUST call `onUploadFailureFile` (or `onUploadFailureImage`).
- **FR-016**: File sizes MUST be displayed using 1024-based division with KB/MB/GB labels (not KiB/MiB/GiB). A size of -1 (unknown) MUST display as blank.
- **FR-017**: All 14 UI message strings MUST be localizable and support `###` as a positional substitution placeholder for dynamic values (e.g., file count, max size).
- **FR-018**: The widget MUST be offline-capable (`offlineCapable="true"`) and requires entity context (`needsEntityContext="true"`).

---

## Props Reference

| Name | Type | Default | Caption | Description |
|------|------|---------|---------|-------------|
| `uploadMode` | `"files" \| "images"` | `"files"` | Upload mode | Selects files or images mode; controls active datasource and create action |
| `associatedFiles` | `ListValue` | pre-configured | Associated files | Datasource for uploaded files (files mode) |
| `associatedImages` | `ListValue` | pre-configured | Associated images | Datasource for uploaded images (images mode) |
| `readOnlyMode` | `boolean` | `false` | Read only | Disables upload and remove; shows download-only list |
| `createFileAction` | `ActionValue?` | pre-configured | Create file action | Nanoflow that creates an empty UploadedFile object (files mode) |
| `createImageAction` | `ActionValue?` | pre-configured | Create image action | Nanoflow that creates an empty UploadedImage object (images mode) |
| `allowedFileFormats` | `AllowedFileFormatsType[]` | `[]` | Allowed file formats | Format restrictions (ignored in images mode); empty = unrestricted |
| `maxFilesPerUpload` | `DynamicValue<Big>` | unlimited | Max files per upload | Dynamic expression; 0 = unlimited |
| `maxFileSize` | `number` (MB) | unlimited | Max file size (MB) | Static integer; 0 = unlimited |
| `enableCustomButtons` | `boolean` | `false` | Enable custom buttons | Replaces default download/remove with custom button list |
| `customButtons` | `CustomButtonsType[]` | `[]` | Custom buttons | List of custom buttons; max one `buttonIsDefault = true` |
| `objectCreationTimeout` | `number` (s) | `10` | Object creation timeout | Max seconds to wait for a created object from the datasource |
| `onUploadSuccessFile` | `ListActionValue?` | — | On upload success (file) | List action called per file on successful upload |
| `onUploadFailureFile` | `ListActionValue?` | — | On upload failure (file) | List action called per file on upload failure |
| `onUploadSuccessImage` | `ListActionValue?` | — | On upload success (image) | List action called per image on successful upload |
| `onUploadFailureImage` | `ListActionValue?` | — | On upload failure (image) | List action called per image on upload failure |
| *(14 message props)* | `DynamicValue<string>` | *(localizable)* | Texts group | Localizable message templates; use `###` for substitutions |

---

## Changelog

### v2.4.2 (2026-04-23)
- **Fixed**: Download button broken on Mendix 11.8.

### v2.4.1 (2026-02-12)
- **Added**: Missing Dutch translations for all message strings.

### v2.4.0 (2025-12-10)
- **Added**: `onUploadSuccessFile`, `onUploadFailureFile`, `onUploadSuccessImage`, `onUploadFailureImage` action callbacks — called per file/image item on upload outcome.

---

## Open Questions
> Could not be determined from source code alone — requires human review
- [ ] What is the expected behavior when `createFileAction` resolves but the datasource does not return a new item (e.g., nanoflow creates the object but the association is not committed)? The timeout fires, but is there a way to differentiate this from a genuinely slow nanoflow?
- [ ] `anyImageFile` accepts SVG (`image/svg+xml`). Is SVG support intentional for all image-mode use cases, or should it be a separate configurable option?
- [ ] Are there documented maximum file size limits imposed by the Mendix runtime or server configuration that interact with `maxFileSize`?
- [ ] The download URL is re-fetched on each download click. Is there a scenario where `mx.data.getDocumentUrl` could fail silently (e.g., permission change after page load)?
