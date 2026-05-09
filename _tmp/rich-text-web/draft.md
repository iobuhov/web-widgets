# rich-text-web — Draft Spec

Widget: `rich-text-web`
Package: `packages/pluggableWidgets/rich-text-web/`
Agent: worker
Date: 2026-05-09

---

## src/RichText.tsx

**1. What is the purpose of this file?**
The root Mendix pluggable widget entry component. It handles the Dojo runtime incubator workaround, renders a loading spinner during initialization, delegates to `EditorWrapper`, and renders a `ValidationAlert` for attribute validation errors.

**2. What kind of logic is described in this file?**
- A `MutationObserver` watches `.mx-incubator.mx-offscreen` to detect when the Mendix Dojo runtime has finished moving the widget out of the offscreen incubator `<div>`. Until that happens, `isIncubator` stays `true` and the editor is replaced with a loading spinner.
- The widget shows a `<div class="mx-progress">` (Mendix standard loading spinner) while either `stringAttribute.status === "loading"` or `isIncubator === true`.
- CSS class logic: read-only + `readOnlyStyle === "readPanel"` → `form-control-static`; otherwise `form-control`. Additional class `widget-rich-text-readonly-${readOnlyStyle}` when read-only.
- `enableStatusBar` is passed through but overridden to `false` when `stringAttribute.readOnly`.

**3. What part of behavior can be documented from this file?**
- The widget defers rendering until the Dojo runtime has placed it in the live DOM. This prevents Quill from losing its reference to the editor's iframe.
- `ValidationAlert` renders Mendix-standard validation error messages from `stringAttribute.validation`.
- The widget always renders `EditorWrapper` once the attribute is available and the Dojo incubator check passes.

**4. Is it user-facing?**
Partially — the loading spinner and validation alert are user-facing; the incubator logic is internal.

**5. What new did you learn from this file?**
This is the only widget in the set so far with a `MutationObserver` workaround for a specific Mendix runtime behavior (Dojo mode incubator). React/Atlas mode does not have this issue — the fix is specifically for legacy Dojo runtime compatibility.

---

## src/RichText.xml

**1. What is the purpose of this file?**
Mendix widget descriptor declaring all configurable properties, grouped into five tabs: General, Dimensions, Events, Advanced, and Custom toolbar.

**2. What kind of logic is described in this file?**
**General:**
- `stringAttribute` (String attribute — the only supported type).
- `enableStatusBar` (boolean, default true).
- `preset` (enum: basic/standard/full/custom, default basic).
- `toolbarLocation` (enum: auto/top/bottom/hide, default top — "auto" = sticky).
- `readOnlyStyle` (enum: text/bordered/readPanel, default text).
- `formOrientation` (enum: horizontal/vertical, default horizontal — for modal dialogs).

**Dimensions:**
- Width: `widthUnit` (pixels/percentage, default percentage) + `width` (default 100).
- Height: `heightUnit` (auto/pixels/percentage/viewport, default pixels) + `height` (default 250).
- Min/Max height with corresponding unit selectors and `OverflowY` (auto/scroll/hidden).

**Events:**
- `onChange` (action, optional), `onFocus` (action, optional), `onBlur` (action, optional), `onLoad` (action, optional).
- `onChangeType` (enum: onLeave/onDataChange, default onLeave).

**Advanced:**
- `spellCheck` (boolean, default false), `linkValidation` (boolean, default true).
- `defaultFontFamily` / `defaultFontSize` (textTemplate, optional).
- `customFonts` (object list: fontName + fontStyle pairs).
- `imageSource` (datasource list, optional) + `imageSourceContent` (widgets slot) + `enableDefaultUpload` (boolean, default true).
- `statusBarContent` (enum: wordCount/characterCount/characterCountHtml, default wordCount).

**Custom toolbar:** 13 boolean toggles (history, fontStyle, fontScript, list, indent, embed, align, code, fontColor, header, view, remove, tableBetter) + `advancedConfig` (object list of `ctItemType` enum items with 34 possible button types).

**3. What part of behavior can be documented from this file?**
- `offlineCapable="true"` — rich text works offline (though full functionality requires connectivity for image uploads).
- `needsEntityContext="true"` — requires a Mendix data source object.
- The toolbar can be completely hidden (`toolbarLocation="hide"`), placed at top/bottom, or made sticky (`auto`).
- When `toolbarLocation` is "hide", the preset property is also hidden in Studio Pro.
- `onChangeType` controls commit behavior: `onLeave` (fires when user leaves the field, change validated by text diff) or `onDataChange` (fires with 200ms debounce while typing).

**4. Is it user-facing?**
No — Studio Pro configuration descriptor.

**5. What new did you learn from this file?**
The custom toolbar system has two modes: "basic" (13 boolean toggles for groups) and "advanced" (ordered list of individual button keys including separators). The `advancedConfig` list includes 34 possible button types plus `separator`, enabling fine-grained toolbar composition. This is significantly more configurable than any other widget in this set.

---

## src/RichText.editorConfig.ts

**1. What is the purpose of this file?**
Provides Studio Pro property visibility rules (`getProperties`) and structure preview rendering (`getPreview`) for the rich text widget.

**2. What kind of logic is described in this file?**
- `getProperties`: hides/shows properties conditionally:
  - When `preset !== "custom"`: hides all 13 toolbar group keys, `toolbarConfig`, `advancedConfig`.
  - When `toolbarConfig === "basic"`: hides `advancedConfig`.
  - When `toolbarConfig === "advanced"`: hides all 13 individual toolbar group toggles.
  - Height: when `heightUnit === "percentageOfWidth"` (auto), hides `height`; otherwise hides min/max/overflow props.
  - `minHeight` hidden when `minHeightUnit === "none"`; `maxHeight`/`OverflowY` hidden when `maxHeightUnit === "none"`.
  - `onChangeType` hidden when no `onChange` action is set.
  - `preset` hidden when `toolbarLocation === "hide"`.
  - Image content/upload props hidden when no `imageSource` datasource is configured.
  - `statusBarContent` hidden when `enableStatusBar === false`.
- `getPreview`: renders a static SVG preview image (light/dark mode variants) with the bound attribute name substituted into the SVG text. When `imageSource` is configured, appends a dropzone slot for `imageSourceContent`.

**3. What part of behavior can be documented from this file?**
- The toolbar group properties are only visible when `preset === "custom"` — otherwise they are irrelevant.
- The advanced config item list is only visible when `toolbarConfig === "advanced"`.
- Studio Pro preview is a static SVG (not a live Quill instance).
- The attribute name from `stringAttribute` is embedded in the preview SVG text.

**4. Is it user-facing?**
No — Studio Pro only.

**5. What new did you learn from this file?**
The `getProperties` function is more complex than any other widget in this set — it manages conditional visibility for ~20+ properties across multiple dependency chains. This reflects the widget's high configurability.

---

## src/RichText.editorPreview.tsx

**1. What is the purpose of this file?**
Renders a static image preview of the rich text widget in Studio Pro live preview mode (React-based preview, not structure view).

**2. What kind of logic is described in this file?**
- Renders the light SVG preview image (always light — no dark mode handling in this file, unlike `editorConfig.ts`).
- Replaces `"[No attribute selected]"` with `[attributeName]` in the SVG source when `stringAttribute` is configured.
- When `imageSource` is configured, renders the `imageSourceContent` renderer slot with a placeholder caption.

**3. What part of behavior can be documented from this file?**
- The preview is always the light theme SVG (static image, no live Quill initialization in Studio Pro preview).
- No `getPreviewCss()` export — CSS is not injected separately for this widget in preview mode.

**4. Is it user-facing?**
No — Studio Pro design mode preview only.

**5. What new did you learn from this file?**
Unlike `editorConfig.ts` which supports both light/dark SVG variants, `editorPreview.tsx` only imports the light SVG. This is likely intentional since the React-based preview context doesn't propagate dark mode the same way as the structure view.

---

## src/components/EditorWrapper.tsx

**1. What is the purpose of this file?**
The main operational component housing all runtime logic: Quill integration, attribute read/write, toolbar rendering, debouncing, fullscreen state, status bar counts, and event handling. Wraps itself in `EditorProvider`.

**2. What kind of logic is described in this file?**
- **Attribute write-back**: `setAttributeValueDebounce` (200ms) calls `stringAttribute.setValue(semanticHTML)`. For `onDataChange` mode, also fires `executeAction(onChange)`. Fires only when value has actually changed.
- **onLoad event**: fires once on first `quillRef.current` availability via a `useEffect` + `isFirstLoad` ref.
- **Status bar count**: `calculateCounts` computes wordCount/characterCount/charCountHtml from Quill's `getText()` or `getSemanticHTML()`. Updates on `stringAttribute.value` change.
- **Default font**: `updateDefaultFontFamily`/`updateDefaultFontSize` called on `MendixTheme` instance when `defaultFontFamily?.value` or `defaultFontSize?.value` change.
- **Toolbar**: `createPreset(props)` builds toolbar config from preset; hides toolbar when read-only + non-text style, or `toolbarLocation === "hide"`. Toolbar is hidden via CSS class `hide-toolbar`.
- **Fullscreen**: reads `isFullscreen` from `EditorContext`; applies `fullscreen` CSS class; editor height becomes `"100%"` in fullscreen.
- **Click-to-focus**: outer `<div onClick>` focuses the Quill editor when the user clicks the toolbar or the editor's container.
- **Sticky toolbar**: `<StickySentinel>` is rendered before the toolbar when `toolbarLocation === "auto"`.
- **Key re-mount**: `Editor` uses `key={toolbarId_readOnly}` to force re-mount when read-only state changes.

**3. What part of behavior can be documented from this file?**
- The editor is an uncontrolled Quill component; attribute updates flow out via `onTextChange` → debounce → `setValue`.
- Status bar is hidden in read-only mode.
- `onChangeType: "onLeave"` fires `onChange` in the `useActionEvents` blur handler (not here), while `"onDataChange"` fires here in the debounce callback.
- The `EditorWrapper` export default wraps `EditorWrapperInner` in `<EditorProvider>` — fullscreen state is scoped to each widget instance.

**4. Is it user-facing?**
Partially — the status bar and fullscreen wrapper are user-facing.

**5. What new did you learn from this file?**
The 200ms debounce on `setValue` is guarded by `onChange?.isExecuting ?? false` — if an onChange action is currently executing, the debounce is paused. This prevents race conditions between the Mendix action queue and the attribute write.

---

## src/components/Editor.tsx

**1. What is the purpose of this file?**
An uncontrolled React component that owns the Quill instance lifecycle: construction, configuration, module registration, event wiring, and cleanup.

**2. What kind of logic is described in this file?**
- **Quill construction**: creates a new `MxQuill` in a `useEffect` with `[ref, toolbarId]` deps. Quill is mounted into a dynamically created `<div>` appended to the container ref.
- **Modules configured**: `keyboard` (custom bindings), `table: false` (disables default Quill table), `table-better` (custom table plugin with 8 menu types), `toolbar` (container = `#toolbarId` or empty array if hidden), resize module (from config), image uploader.
- **Custom toolbar handlers**: link, video, indent, `view-code`, image — all custom handlers replacing Quill's defaults.
- **ACTION_DISPATCHER**: Quill emits events on this custom event channel; the Editor listens and routes to appropriate handlers (link dialog, image dialog, video dialog, fullscreen toggle).
- **External value update**: when Quill is not focused and `defaultValue !== quill.getSemanticHTML()`, clipboard-converts the HTML and calls `setContents`.
- **Image source entity upload**: when `imageSource.status === "available"`, calls `MxUploader.setEntityUpload(true)`.
- **Modal dialog**: `<Dialog>` component rendered alongside the editor for link/image/video insertion.

**3. What part of behavior can be documented from this file?**
- Quill is remounted whenever `toolbarId` or `ref` changes (controlled by `EditorWrapper` key prop on readOnly change).
- Custom link/video/image/view-code toolbar buttons open modal dialogs instead of Quill's built-in popups.
- Tables use `quill-table-better` plugin (not Quill's native table module which is disabled).
- The editor container is cleaned up (`container.innerHTML = ""`) on unmount.

**4. Is it user-facing?**
No — internal Quill integration component.

**5. What new did you learn from this file?**
The `ACTION_DISPATCHER` custom Quill event is a key architectural pattern: it allows Quill blots/modules to communicate back to React (e.g., clicking an existing link opens the link dialog, clicking an existing image opens the image dialog). This decouples Quill's imperative DOM world from React's declarative component world.

---

## src/store/store.ts + src/store/EditorProvider.tsx

**1. What is the purpose of these files?**
A React context-based state store (reducer pattern) for editor global state. Currently manages only `isFullscreen: boolean`.

**2. What kind of logic is described in these files?**
- `store.ts`: `EditorState` has one field: `isFullscreen: boolean` (default `false`). `editorReducer` handles `SET_FULLSCREEN` action — toggles if no value provided, sets to provided value otherwise.
- `EditorProvider.tsx`: creates two contexts (`EditorContext` for reading state, `EditorDispatchContext` for dispatching). Uses `useReducer`. `EditorProvider` wraps children in both context providers.

**3. What part of behavior can be documented from this file?**
- Fullscreen is toggled when `SET_FULLSCREEN` action is dispatched with no value or null value; it is set to a specific boolean when a value is provided.
- Each widget instance gets its own fullscreen state (the provider is inside `EditorWrapper`).

**4. Is it user-facing?**
No — internal state management.

**5. What new did you learn from these files?**
The store is minimal — only fullscreen state. This suggests the architectural decision was to keep all other state local to `EditorWrapper` (word count, quill ref, etc.) rather than centralizing it. The context exists specifically to allow the `Fullscreen` toolbar button (a deep descendant) to toggle state without prop-drilling.

---

## src/store/useActionEvents.ts

**1. What is the purpose of this file?**
A hook that wraps the Mendix `onFocus`, `onBlur`, and `onChange` action events with focus/blur handlers that correctly exclude internal focus transitions (toolbar clicks, modal interactions).

**2. What kind of logic is described in this file?**
- `isInternalTarget`: checks if `relatedTarget` (element gaining focus) is a descendant of `currentTarget`, OR is inside `.widget-rich-text-modal-body`. This prevents blur/focus from firing when the user clicks within the editor's toolbar or opens a modal dialog.
- `onFocus`: fires `executeAction(onFocus)` and captures `editorValueRef.current` (current text snapshot) when focus enters from outside.
- `onBlur`: fires `executeAction(onBlur)` and, for `onChangeType === "onLeave"`, fires `executeAction(onChange)` only if `quill.getText()` differs from the captured snapshot.

**3. What part of behavior can be documented from this file?**
- `onFocus` and `onBlur` Mendix actions only fire on true focus/blur transitions — not on internal element switching (toolbar, modals).
- `onLeave` onChange compares text snapshots to avoid spurious change events when the user focuses and immediately leaves without editing.
- The modal body exclusion (`.widget-rich-text-modal-body`) means clicking the Insert Link dialog does not trigger an `onBlur` event.

**4. Is it user-facing?**
No — internal event management hook.

**5. What new did you learn from this file?**
The `editorValueRef.current` approach (capturing text on focus, comparing on blur) provides a lightweight dirty-check for `onLeave` change detection. This avoids comparing full HTML content (which could differ due to whitespace) by using Quill's plain text representation.

---

## src/utils/MxQuill.ts

**1. What is the purpose of this file?**
Extends the Quill editor class to override `setContents`, fix HTML serialization for lists, and register custom modules (fonts, link validation).

**2. What kind of logic is described in this file?**
- `MxEditor` extends Quill's internal `Editor` class, overriding `getHTML` to return empty string when blank (matching Mendix's "no content" case).
- `MxQuill` extends `Quill`:
  - `constructor`: replaces `this.editor` with an `MxEditor` instance.
  - `setContents`: clears then updates (works around a Quill bug where `setContents` doesn't properly reset).
  - `registerCustomModules`: registers `FontStyleAttributor` (dynamic custom fonts) and either `CustomLink` (with URL validation) or `CustomLinkNoValidation` based on `linkValidation` prop.
- `convertListHTML` / `convertHTML`: heavily modified versions of Quill's internal HTML conversion, adding proper `list-style-type` CSS for nested lists (ordered → lower-alpha → lower-roman cycling by indent level), and adding `border: 1px solid #000` to tables.

**3. What part of behavior can be documented from this file?**
- Nested ordered lists cycle: level 0 = decimal, level 1 = lower-alpha, level 2 = lower-roman, level 3 = decimal again.
- Tables in the serialized HTML always get `style="border: 1px solid #000"`.
- When `linkValidation=true`, only valid URLs are accepted in links; `linkValidation=false` allows any string.
- Custom fonts are registered as Quill attributors dynamically based on the `customFonts` prop.

**4. Is it user-facing?**
No — internal Quill customization.

**5. What new did you learn from this file?**
The list HTML serialization is a significant fork from Quill's default — Quill's default doesn't add `list-style-type` CSS, so nested lists would display incorrectly in read-only mode (where Quill's built-in CSS isn't applied). The widget must serialize lists with explicit CSS to ensure correct rendering outside the editor.

---

## src/utils/helpers.ts

**1. What is the purpose of this file?**
Utility for computing the wrapper `CSSProperties` style object from the widget's dimension props.

**2. What kind of logic is described in this file?**
- `constructWrapperStyle`: maps `widthUnit`/`width` → `width` CSS (e.g., `"100%"` or `"300px"`).
- `heightUnit === "percentageOfWidth"` (auto mode): `height = "auto"` + optional `minHeight` and `maxHeight`/`overflowY`.
- Other height modes: `height` is set directly in px, vh, or %.
- `getHeightScale`: converts integer + unit → CSS string (`px`, `vh`, or `%`).
- Also exports `ACTION_DISPATCHER` constant (the custom Quill event name).

**3. What part of behavior can be documented from this file?**
- When `heightUnit === "percentageOfWidth"` (labeled "Auto" in Studio Pro), the editor grows with content up to `maxHeight` if set.
- Min/max height and overflow only apply in auto height mode.
- The `ACTION_DISPATCHER` string is the shared constant connecting Quill's emitter to React event handling.

**4. Is it user-facing?**
No — internal utility.

**5. What new did you learn from this file?**
`ACTION_DISPATCHER` is defined in `helpers.ts` and imported by both `Editor.tsx` and other modules — it's the central communication channel name for Quill-to-React events.

---

## src/components/CustomToolbars/presets.ts

**1. What is the purpose of this file?**
Builds the Quill toolbar configuration array from the widget's preset/custom toolbar props.

**2. What kind of logic is described in this file?**
- `createPreset`: when `preset !== "custom"`, returns `DEFAULT_TOOLBAR` (a constant from `constants.ts`). When `custom`:
  - `toolbarConfig === "basic"`: calls `defineBasicGroups` — iterates widget props, checks if each prop key exists in `TOOLBAR_GROUP` (a key→buttons map), includes the group if enabled.
  - `toolbarConfig === "advanced"`: calls `defineAdvancedGroups` — parses `advancedConfig` item list, splits on `separator` items to create groups, maps `ctItemType` to Quill toolbar buttons.
- Returns `toolbarContentType[]` — an array of `{ children: [...] }` group objects.

**3. What part of behavior can be documented from this file?**
- Basic custom toolbar: 13 boolean props map to predefined button groups from `constants.ts`.
- Advanced custom toolbar: ordered list of up to 34 button types, with explicit separator placement controlling group boundaries.
- Non-custom presets (basic, standard, full) bypass all customization and use a hardcoded default toolbar.

**4. Is it user-facing?**
No — internal toolbar configuration.

**5. What new did you learn from this file?**
The `DEFAULT_TOOLBAR` is shared across basic/standard/full presets — it appears that the preset enum key (basic/standard/full) does not actually change the toolbar buttons (they all use the same `DEFAULT_TOOLBAR`). The preset may affect other editor behavior (e.g., which Quill formats are enabled) rather than toolbar button visibility.

---

## e2e/RichText.spec.js

**1. What is the purpose of this file?**
Playwright end-to-end tests verifying visual rendering across multiple modes and dialog interactions.

**2. What kind of logic is described in this file?**
Tests navigate to `/p/basic`, `/p/advanced`, `/p/custom` pages and take screenshot comparisons for:
- Inline basic mode, toolbar basic mode.
- Bottom toolbar advanced mode, toolbar advanced mode.
- Insert Image dialog (triggered by `.ql-image` button).
- View Code dialog (triggered by `.ql-view-code` button).
- Inline custom mode, toolbar custom mode.
- Custom mode with all options enabled, custom mode with no options enabled.
- Various read-only styles.

**3. What part of behavior can be documented from this file?**
- The test uses `threshold: 0.4` screenshot comparison (high tolerance), suggesting UI details may vary slightly between runs.
- Insert Image dialog uses selector `.widget-rich-text-modal-body`.
- The "View Code" feature (raw HTML editing) has its own toolbar button (`.ql-view-code`) and dialog.
- E2E tests require a "Generate Data" click on the homepage before navigating to the test pages (for data setup).
- Multiple widget instances coexist on a single page (`mx-name-richText1` through `mx-name-richText4`).

**4. Is it user-facing?**
No — test file.

**5. What new did you learn from this file?**
The widget has a "View Code" feature (raw HTML editor dialog) accessible via toolbar. This is a powerful developer-facing feature not immediately apparent from the widget props. The `.ql-view-code` button and `viewCodeDialog` suggest users can directly edit the raw HTML of the rich text content.

---

## src/__tests__/RichText.spec.tsx

**1. What is the purpose of this file?**
Unit tests for the `RichText` entry component — snapshot tests for various configurations.

**2. What kind of logic is described in this file?**
- Default render snapshot.
- Toolbar at top + preset "full" snapshot.
- Read-only with "bordered" style snapshot.
- Status bar with character count.
- Status bar with HTML character count.
- Status bar with "both" count (note: `StatusBarContentEnum` does not seem to include "both" in the XML — this may be a test for a missing/unreleased option).

**3. What part of behavior can be documented from this file?**
- `tableBetter: false` in `defaultProps` — table support is disabled by default in tests.
- `formOrientation: "vertical"` in default test props (XML default is "horizontal").
- `linkValidation: true` in test props (matches XML default).

**4. Is it user-facing?**
No — test file.

**5. What new did you learn from this file?**
The test includes `statusBarContent: "both"` which is not in the XML enum (which has only wordCount/characterCount/characterCountHtml). This suggests either a bug in the test, a planned but unreleased feature, or the type is looser than the XML implies (using string cast `as StatusBarContentEnum`). The test still passes since it renders a snapshot rather than validating enum membership.

---

## CHANGELOG.md

**1. What is the purpose of this file?**
Version history documenting significant changes.

**2. What kind of logic is described in this file?**
Key recent milestones:
- **v4.12.0 (2026-04-22)**: Optional link URL validation, fixed infinite empty lines at end, fixed lower-alpha/lower-roman list styles, improved character count trimEnd.
- **v4.11.2 (2026-03-05)**: Fixed hyperlink `target` attribute reading from code viewer.
- **v4.11.1 (2026-01-27)**: Fixed `<br />` on end of line, `\t` tab removal, link tooltip clipping.
- **v4.11.0 (2025-11-06)**: Fixed blur/change events on modal focus; Tab key adds indentation; `&nbsp;` → `<br />`; Alt+F11/Alt+F10 keyboard shortcuts; Shift+Enter for `<br />`.
- **v4.10.0 (2025-10-02)**: Character count status bar; default font-family/size config; white-space styling; Dojo modal double scrollbar fix.
- **Earlier (not shown)**: Major versions 1-3 introduced the Quill-based editor, custom toolbar, image/video/link dialogs, fullscreen mode.

**3. What part of behavior can be documented from this file?**
- `linkValidation` is a v4.12.0 addition — older versions always validated URLs.
- The status bar was added in v4.10.0 — not present in earlier versions.
- Default font config was added in v4.10.0.
- Tab key behavior (indent rather than focus-exit) was changed in v4.11.0.
- `<br />` is now used instead of `&nbsp;` for empty lines (v4.11.0).

**4. Is it user-facing?**
No — developer changelog.

**5. What new did you learn from this file?**
This is the most actively developed widget in the set (version 4.12 vs others at 3.x). The character count, default font, link validation, and several keyboard behaviors are all recent additions. The widget's HTML serialization has been iteratively refined to handle edge cases (tables, lists, line breaks, tabs).

---

## Summary of Key Findings

- **Core technology**: Built on [Quill](https://quilljs.com/) (v2, slab fork) — extended as `MxQuill` with custom HTML serialization for lists/tables and custom module registration.
- **Uncontrolled component pattern**: Quill owns the DOM; React reads back via `getSemanticHTML()` → debounced → `stringAttribute.setValue()`.
- **Two change modes**: `onLeave` (fires when user leaves field, text diff check) vs `onDataChange` (fires while typing with 200ms debounce).
- **Toolbar configuration**: 4 presets (basic/standard/full/custom); custom has "basic" (13 group toggles) and "advanced" (ordered item list with 34 button types + separators) modes.
- **Custom list serialization**: Nested ordered lists cycle through decimal → lower-alpha → lower-roman by indent level, serialized as explicit `list-style-type` CSS.
- **Modal dialogs**: Link, Image, Video, ViewCode dialogs replace Quill's built-in popups. Modal focus is excluded from blur event detection.
- **Fullscreen**: Context-based state, scoped per widget instance.
- **Dojo runtime workaround**: `MutationObserver` on `.mx-incubator.mx-offscreen` defers editor rendering until the Dojo runtime has placed the widget in the live DOM.
- **Image uploads**: Supports both default file upload and entity-based image selection (via datasource + widget slot pattern).
- **Status bar**: Word count, character count (text), or character count (HTML) — added in v4.10.0.
- **offlineCapable**: `true` in XML, but full image upload/entity features require connectivity.
- **Most complex widget**: ~15 source modules, two separate Quill extension classes, a custom event bus, and a React context store.
