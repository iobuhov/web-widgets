# Draft: html-element-web

Widget package: `packages/pluggableWidgets/html-element-web/`
Worker: worker | Task: EX-032 | Date: 2026-05-09

---

## src/HTMLElement.tsx

**1. What is the purpose of this file?**
This is the main widget entry point — the React component exported as the pluggable widget. It orchestrates the rendering of one or more HTML elements based on the widget configuration, coordinating data-source repetition, attribute/event preparation, and child content.

**2. What kind of logic is described in this file?**
Branching logic between single-element and repeating-element modes: when `tagUseRepeat` is true, it maps over `tagContentRepeatDataSource.items`; otherwise it maps over `[undefined]`. It resolves the tag name via `prepareTag`, generates a stable React key per item using `useId`, and delegates all prop assembly (attributes, events, HTML, children) to utility functions. Returns `null` when the items list is empty.

**3. What part of behavior can be documented from this file?**
The widget renders nothing when `tagContentRepeatDataSource` has no items (null return guard). Each repeated element gets a key composed of a stable ID prefix and the item's own ID (or its index as a fallback). The `sanitizationConfigFull` prop is forwarded directly to `HTMLTag`, making sanitization config a first-class rendering concern.

**4. Is it user-facing?**
Yes — this component produces the HTML output visible in the browser.

**5. What new did you learn from this file?**
The repeat/non-repeat duality is handled at the data-source level: both modes share the same `HTMLTag` render path; the difference is only in how `createAttributeResolver` and `createEventResolver` are called (with or without an `ObjectItem`). This makes the pattern fully symmetric.

---

## src/HTMLElement.xml

**1. What is the purpose of this file?**
Declares the widget's public API for Mendix Studio Pro: its identity (`com.mendix.widget.web.htmlelement.HTMLElement`), all configurable properties, their types, defaults, captions, and nested structure (property groups, object lists).

**2. What kind of logic is described in this file?**
Structural/declarative configuration only — no runtime logic. It defines:
- `tagName` enumeration (14 predefined HTML tags + `__customTag__`)
- `tagNameCustom` string for custom tag names
- `tagUseRepeat` boolean toggling data-source repetition
- `tagContentRepeatDataSource` list data source
- `tagContentMode` enumeration (`container` | `innerHTML`)
- Dual content properties for both modes × both repetition states (4 combinations: `tagContentHTML`, `tagContentContainer`, `tagContentRepeatHTML`, `tagContentRepeatContainer`)
- `attributes` object list with per-item `attributeName`, `attributeValueType`, and 4 value variants (expression/template × non-repeat/repeat)
- `events` object list with `eventName` enumeration (full React synthetic event set), `eventAction`/`eventActionRepeat`, `eventStopPropagation`, `eventPreventDefault`
- `sanitizationConfigFull` multiline string for DOMPurify configuration JSON

**3. What part of behavior can be documented from this file?**
The widget is declared as `needsEntityContext="true"`, `offlineCapable="true"`, `supportedPlatform="Web"`. The `script` tag is absent from the predefined tag list — it is deliberately blocked at validation level (see editorConfig). `tagContentRepeatDataSource` is marked `required="true"` even though the repeat feature is optional; this is controlled via property hiding in editorConfig. Attribute values support both expression and text-template sources, making them compatible with data binding.

**4. Is it user-facing?**
Indirectly — this file determines the configuration surface presented to developers in Studio Pro.

**5. What new did you learn from this file?**
The dual content-property design (4 property variants for content) is a deliberate Studio Pro pattern: each variant is shown/hidden by editorConfig depending on the `tagUseRepeat` + `tagContentMode` combination. The widget is offline-capable, meaning it does not require a server connection at runtime beyond data-source loading.

---

## src/utils/props-utils.ts

**1. What is the purpose of this file?**
Central utility module that converts raw Mendix widget props into React-compatible values: tag name resolution, attribute mapping, event handler creation, HTML content extraction, children resolution, void-element detection, and HTML sanitization via DOMPurify.

**2. What kind of logic is described in this file?**
- `prepareTag`: returns either the enum tag or the custom tag string.
- `createAttributeResolver`/`prepareAttributes`: maps `AttributesType[]` to `HTMLAttributes<Element>`. Special-cases `style` (via `convertInlineCssToReactStyle`) and `class` (renamed to `className`). Merges the Mendix `class` and `style` props into the result — `style` is merged with a spread (widget `style` wins for conflicting keys? No — `result.style = { ...style, ...result.style }` so attribute-defined style wins).
- `createEventResolver`/`prepareEvents`: creates React synthetic event handlers. Respects `eventPreventDefault` and `eventStopPropagation` flags. Chooses `eventAction` vs `eventActionRepeat` based on whether an `ObjectItem` is present.
- `prepareHtml`/`prepareChildren`: mutually exclusive — only one returns a value depending on `tagContentMode`.
- `isVoidElement`: checks a hardcoded list of 14 void HTML elements.
- `createSanitize`/`useSanitize`: wraps DOMPurify, parsing optional JSON config. The `useSanitize` hook memoizes the sanitizer via `useState` initializer so it is created only once per component mount.

**3. What part of behavior can be documented from this file?**
Attribute `style` supplied via the attributes list is merged onto the widget-level `style` prop, with the attribute-defined style taking precedence (spread order). `class` attribute merges with the widget's `class` prop and adds the fixed `widget-html-element` CSS class prefix — all three are always present: `widget-html-element`, the Mendix `class`, and any dynamic `class` attribute. `createSanitize` throws a descriptive error on malformed JSON config (caught at parse time), making configuration errors explicit rather than silent.

**4. Is it user-facing?**
Internal utility — not directly visible, but its behavior determines how user-provided attribute and style values appear in the rendered HTML.

**5. What new did you learn from this file?**
The `useSanitize` hook intentionally uses `useState` (not `useMemo`) to create the DOMPurify instance exactly once per component instance — changes to `sanitizationConfigFull` at runtime are deliberately ignored (the config is treated as mount-time-only). This is a behavioral constraint: updating the sanitization config prop does not re-sanitize; a page reload/remount is required.

---

## src/utils/style-utils.ts

**1. What is the purpose of this file?**
Converts an inline CSS string (as written in HTML `style="..."`) to a React `CSSProperties` object with camelCased property names, handling edge cases like URLs with colons, CSS custom properties, and vendor prefixes.

**2. What kind of logic is described in this file?**
Splits the inline CSS string on `;`, then for each rule uses a regex (`/(?<prop>[^:]+):(?<value>.+)/s`) with the `s` (dotAll) flag to split on the first `:` only — preserving colons inside values (e.g., `url(http://...)`). Empty rules and malformed lines (no `=`-like split) are filtered out. CSS property names are camelCased via regex substitution; vendor prefix `-ms-` is lowercased (React convention: `msColor`), while `-webkit-`, `-moz-`, `-o-` are titleCased (`WebkitColor`, `MozColor`, `OColor`). CSS custom properties (starting with `--`) bypass camelCasing entirely.

**3. What part of behavior can be documented from this file?**
The parser is tolerant of whitespace/newlines between the property name and colon, and between the colon and value. It silently drops broken CSS rules rather than throwing. CSS variables are passed through unchanged, allowing `var(--token)` patterns to work. The `s` flag on the regex means multi-line values are captured correctly.

**4. Is it user-facing?**
Internal — converts user-supplied inline CSS attribute values to React style objects.

**5. What new did you learn from this file?**
The regex-based parser is intentionally simpler than a full CSS parser and handles the documented edge cases (URLs with colons, whitespace, custom vars, vendor prefixes). Broken declarations are silently ignored, which means malformed CSS in the `style` attribute fails gracefully without breaking the widget.

---

## src/components/HTMLTag.tsx

**1. What is the purpose of this file?**
The core rendering primitive — a thin React component that renders any HTML tag with arbitrary attributes and either sanitized `innerHTML` or React children, using a dynamic tag name resolved at runtime.

**2. What kind of logic is described in this file?**
Accepts `tagName`, `unsafeHTML`, `children`, `attributes`, and optional `sanitizationConfig`. If `unsafeHTML` is defined (even as empty string), renders via `dangerouslySetInnerHTML` with DOMPurify sanitization applied. Otherwise renders with React children. The tag is dynamic (`const Tag = props.tagName`), enabling any valid HTML element. Sanitization is initialized via `useSanitize` (memoized per mount).

**3. What part of behavior can be documented from this file?**
The choice between innerHTML and container mode is determined by whether `unsafeHTML` is `undefined` (strict check — empty string `""` triggers innerHTML mode). All HTML set via `innerHTML` mode is sanitized by DOMPurify before insertion; the sanitization config from `sanitizationConfigFull` prop is respected. Attributes including `data-*` attributes are accepted via the typed spread.

**4. Is it user-facing?**
Yes — this is what the browser ultimately renders.

**5. What new did you learn from this file?**
The `undefined` check (not falsy) for `unsafeHTML` is semantically important: an empty string `""` will trigger the `dangerouslySetInnerHTML` path, resulting in an empty sanitized element — not the children path. This means if `tagContentHTML` evaluates to an empty string, the widget renders an empty tag without children.

---

## src/HTMLElement.editorConfig.ts

**1. What is the purpose of this file?**
Configures the Mendix Studio Pro design-time experience: which properties to show/hide depending on the current configuration, validation checks (errors and warnings), the structure-mode preview rendering, and the custom caption shown in the widget palette/canvas.

**2. What kind of logic is described in this file?**
- `getProperties`: hides irrelevant properties based on `tagName`, `tagUseRepeat`, and `tagContentMode`. Void elements (like `img`, `input`) suppress all content-related props. Non-repeating mode hides the repeat-variant content props and vice versa. Attribute value fields: only one of the 4 variants is shown per attribute item. Event action fields: only one of `eventAction`/`eventActionRepeat` is shown per event item based on `tagUseRepeat`.
- `check`: validates custom tag name (must match `/^[a-z][\w.-]*$/i`), blocks `script` tags (error), warns on empty attribute values, errors on duplicate attribute or event names, validates `sanitizationConfigFull` as valid JSON.
- `getPreview`: renders a structure-mode preview showing the tag name with either a dropzone (container mode) or the raw HTML text (innerHTML mode). Void elements show as self-closing tags.
- `getCustomCaption`: shows `<tagName />` as the widget label on the canvas.

**3. What part of behavior can be documented from this file?**
The `script` tag is explicitly blocked with an error (not a warning), making script injection via custom tag names a hard error at design time. Duplicate attribute names and duplicate event names both produce errors. A missing attribute value produces a warning (not an error), allowing partial configurations during development. The structure preview adapts to Studio Pro version: `canHideDataSourceHeader` is enabled from SP 9.20+.

**4. Is it user-facing?**
Indirectly — only visible to developers in Studio Pro, not to end users at runtime.

**5. What new did you learn from this file?**
The `disabledElements = ["script"]` array is the sole blocklist — only `script` is explicitly forbidden. Other potentially dangerous tags (like `iframe`, `object`, `embed`) are not blocked at design time; they may work if entered as a custom tag. The design-time validation is a first line of defense, not comprehensive security — DOMPurify at runtime provides the actual sanitization layer.

---

## src/HTMLElement.editorPreview.tsx

**1. What is the purpose of this file?**
Provides the live preview rendering inside Studio Pro's canvas when the widget is selected or in preview mode. Renders a simplified but structurally accurate representation of what the widget will produce.

**2. What kind of logic is described in this file?**
For repeat mode: renders 3 placeholder items. For non-repeat + innerHTML mode where `tagContentHTML` is set: renders the raw HTML string via `unsafeHTML` prop (note: the preview canvas does not sanitize — it passes the HTML template literal directly). Void elements render as `<div>{<tag />}</div>` text. Uses the same `HTMLTag` component as runtime.

**3. What part of behavior can be documented from this file?**
In the Studio Pro preview, the repeat mode always shows 3 items regardless of data source. The innerHTML preview shows the raw text template (including `{variable}` tokens) rather than resolved values, since data is not available at design time. The preview CSS is intentionally empty (`getPreviewCss` returns `""`), meaning the widget has no default styling by design.

**4. Is it user-facing?**
No — only visible to developers in Studio Pro's design canvas.

**5. What new did you learn from this file?**
The explicit comment `// html element has no styling by design` in `getPreviewCss` confirms the intentional zero-default-style approach — the widget is a raw HTML building block, not a styled component. This is architecturally significant: all visual styling must come from the Mendix app's CSS or the widget's `style`/`class` props.

---

## typings/HTMLElementProps.d.ts

**1. What is the purpose of this file?**
Auto-generated TypeScript type declarations derived from `HTMLElement.xml`. Provides compile-time type safety for all widget props used in the source code.

**2. What kind of logic is described in this file?**
Defines the `HTMLElementContainerProps` interface (runtime props), `HTMLElementPreviewProps` interface (design-time props), and sub-interfaces `AttributesType` and `EventsType`. The `TagNameEnum` union covers the 14 predefined tags plus `__customTag__`. `EventNameEnum` covers the full React synthetic event set (~120+ event names). The preview props use simplified types (`string`, `{}`) compared to the runtime Mendix SDK types (`DynamicValue`, `ListValue`, etc.).

**3. What part of behavior can be documented from this file?**
`sanitizationConfigFull` is typed as `string` (not optional in the runtime props), meaning it is always present — an empty string represents "no custom config". `tagContentRepeatDataSource` is typed as `ListValue` (required), but the XML marks it `required="true"` while the UI hides it when `tagUseRepeat` is false. The `HTMLElementPreviewProps.className` is marked deprecated since 9.18.0 — `class` should be used instead.

**4. Is it user-facing?**
No — purely a TypeScript development artifact.

**5. What new did you learn from this file?**
The `AttributesType` interface exposes all 4 attribute-value variants as optional fields on every attribute object; the runtime code selects the correct one based on `attributeValueType` and whether an `ObjectItem` is present. This means unused variants are present but `undefined` at runtime.

---

## src/utils/__tests__/props-utils.spec.ts

**1. What is the purpose of this file?**
Unit tests for all functions in `props-utils.ts`, covering tag preparation, children resolution, HTML content resolution, event handler creation (with/without repeat item, with/without stop-propagation/prevent-default flags), attribute resolver (both template and expression modes, with/without item), `prepareAttributes` merging behavior, and DOMPurify sanitization.

**2. What kind of logic is described in this file?**
Tests confirm: `prepareTag` with `__customTag__` uses the custom string; `prepareChildren` returns `undefined` in innerHTML mode and the correct node in container mode; `prepareHtml` returns `undefined` in container mode and the resolved value in innerHTML mode; event handlers call `stopPropagation`/`preventDefault` only when the respective flags are true; `prepareAttributes` merges `class` → `className` with the `widget-html-element` prefix, converts inline style, and preserves other attributes; `createSanitize` strips script injection, onerror handlers, and javascript: hrefs.

**3. What part of behavior can be documented from this file?**
The test for `prepareAttributes` explicitly verifies the class merging: `"widget-html-element mx-name-hello dynamic-class"` — confirming the order is `widget-html-element` + Mendix class + dynamic class attribute. Style merging preserves the Mendix `style` prop (`borderRadius: "2px"`) alongside the inline style from the attribute. The sanitize test shows that `<body onload=...>` is reduced to an empty string — DOMPurify strips the entire body tag in this context.

**4. Is it user-facing?**
No — developer-facing tests only.

**5. What new did you learn from this file?**
The `createSanitize` test confirms that `javascript:` protocol links are sanitized to `<a>hello</a>` (href removed). This is a behavioral security constraint: even if a user sets an `href` attribute to a `javascript:` URL via the attributes list, the innerHTML path would sanitize it — but note that attributes set via the `attributes` prop list go through `prepareAttributes`, not through DOMPurify; only `innerHTML` content is sanitized by DOMPurify.

---

## src/utils/__tests__/style-utils.spec.ts

**1. What is the purpose of this file?**
Unit tests for `convertInlineCssToReactStyle`, covering normal property conversion, no-space syntax, multi-space/newline whitespace, properties with colons in values (URLs), broken/empty declarations, CSS custom properties, and vendor prefixes.

**2. What kind of logic is described in this file?**
Tests verify: camelCase conversion of hyphenated properties; parsing of `background-image: url(http://...)` without splitting on the URL colon; silent dropping of malformed rules like `"foo-bar"` (no value) or empty segments `;;`; CSS variables (`--super-custom-var`) pass through unchanged; vendor prefixes produce `MozColor`, `OColor`, `WebkitColor`, `msColor`.

**3. What part of behavior can be documented from this file?**
The `-ms-` vendor prefix produces `msColor` (lowercase first character), while all other vendor prefixes produce title-cased output. This matches the React CSSProperties naming convention for Microsoft vendor prefixes. CSS custom properties are never camelCased — the `--` prefix is the guard condition.

**4. Is it user-facing?**
No — developer-facing tests.

**5. What new did you learn from this file?**
The `s` dotAll flag on the CSS regex is tested implicitly via the multiline whitespace test — `"background-color \n\n : \n    #FF00FF"` parses correctly. This is a non-obvious correctness detail for users who configure style attributes with line breaks.

---

## src/components/__tests__/HTMLTag.spec.tsx

**1. What is the purpose of this file?**
Integration/unit tests for the `HTMLTag` component covering innerHTML rendering (snapshot), container rendering (snapshot), HTML sanitization of XSS payloads, and event handler firing.

**2. What kind of logic is described in this file?**
Tests use `@testing-library/react` and `userEvent`. Sanitization tests cover: script tag injection (`<script>alert(1)</script>`), `onerror` attribute injection (`<img src=x onerror=alert(1)>`), `onmouseover` attribute, and a complex nested style/img XSS attempt. The event test verifies that the onClick handler fires when the element is clicked via `userEvent.click`.

**3. What part of behavior can be documented from this file?**
All tested XSS vectors are sanitized: script content is removed, event handler attributes are stripped from img tags, and the complex nested XSS (`<option><style><img ...></style>`) is also cleaned. The snapshot test for innerHTML confirms that DOMPurify is applied before insertion. The `fires events` test confirms the component correctly wires up React synthetic events from the `attributes` spread.

**4. Is it user-facing?**
No — developer-facing tests.

**5. What new did you learn from this file?**
The XSS test with `<a>123</a><option><style><img src=x onerror=alert(1)></style>` is a known DOMPurify edge case — the widget explicitly tests this pattern, suggesting it was encountered as a real attack vector. This confirms the security posture: DOMPurify is not bolted on as an afterthought but is tested against specific real-world XSS payloads.

---

## CHANGELOG.md

**1. What is the purpose of this file?**
Records all notable changes per release following Keep a Changelog conventions.

**2. What kind of logic is described in this file?**
Release history from v1.0.0 (2022-11-24) to v1.2.7 (2026-04-20). Organized by: Security (DOMPurify updates), Fixed (bug fixes), Added (new features), Changed (icon/style updates).

**3. What part of behavior can be documented from this file?**
- v1.0.0 (2022-11-24): Initial release.
- v1.0.1 (2023-01-05): Fixed Studio Pro compatibility below 9.18; fixed inline CSS style parsing.
- v1.1.0 (2023-06-05): Updated icons and structure-preview colors for dark/light mode.
- v1.1.1 (2023-09-27): Removed redundant code to improve load time.
- v1.2.0 (2024-02-01): Added `sanitizationConfigFull` — configurable DOMPurify JSON config for advanced use cases where the default is too restrictive.
- v1.2.1 (2024-08-23): DOMPurify 2.5.6 (template injection prevention).
- v1.2.2 (2025-03-14): DOMPurify 3.2.4.
- v1.2.4 (2025-12-08): Fixed non-unique React `key` prop warning in certain scenarios.
- v1.2.5 (2026-02-10): Added license file and README for open-source dependencies.
- v1.2.6 (2026-03-31): DOMPurify 3.3.3.
- v1.2.7 (2026-04-20): DOMPurify 3.4.0.

**4. Is it user-facing?**
No — developer/operator reference.

**5. What new did you learn from this file?**
The recurring DOMPurify updates (5 security releases in ~2 years) indicate the team actively tracks DOMPurify CVEs. The `sanitizationConfigFull` prop was added in v1.2.0 specifically because the default DOMPurify config was blocking legitimate advanced use cases — this is an important behavioral note: default sanitization may reject valid HTML that requires explicit allow-listing.

---

## Summary of Key Behavioral Constraints

1. **Repeat mode null guard**: Widget renders nothing (`null`) when data source has no items.
2. **innerHTML vs container mode**: Determined by `tagContentMode`; the two modes are mutually exclusive per configuration.
3. **Empty string triggers innerHTML**: `unsafeHTML=""` (empty string) activates `dangerouslySetInnerHTML`, not the children path.
4. **DOMPurify sanitization**: Always applied to innerHTML content. Config is memoized at mount — runtime changes to `sanitizationConfigFull` have no effect without remount.
5. **`script` tag blocked**: Design-time validation rejects `script` as a custom tag name with an error.
6. **Class merging order**: `widget-html-element` + Mendix `class` prop + dynamic `class` attribute (all always present).
7. **Style merging**: Attribute-defined `style` takes precedence over Mendix `style` prop (spread order in `prepareAttributes`).
8. **Attribute-level `class`/`style` not sanitized**: Only innerHTML content goes through DOMPurify; attributes set via the `attributes` list are passed directly as React props.
9. **No default styling**: The widget intentionally has no CSS by design (`getPreviewCss` returns `""`).
10. **Offline-capable**: Declared `offlineCapable="true"` in widget XML.
