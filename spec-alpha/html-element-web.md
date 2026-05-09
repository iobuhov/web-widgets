# HTMLElement

## Purpose
The HTMLElement widget renders any configurable HTML element — including custom tag names — directly in a Mendix web application. It enables developers to emit semantic or structural HTML that the standard Mendix widget set does not expose, while providing built-in DOMPurify sanitization for innerHTML content and full data-binding support for element attributes, events, and content. It is designed as a raw HTML building block: it carries no default styling and requires consumers to supply all visual presentation through application CSS or the widget's `class`/`style` props.

## User Scenarios

### [P1] Render a semantic HTML element with dynamic attributes
**Given** a developer configures the widget with `tagName` set to an enumerated tag (e.g. `article`) and adds one or more attribute entries with expression-bound values  
**When** the page is loaded  
**Then** the browser renders the chosen element with the computed attribute values reflected on the DOM node

#### Edge Cases
- If `attributeValueType` is `expression` but no `ObjectItem` is present (non-repeat mode), the non-repeat expression variant is used.
- Duplicate attribute names produce a design-time error in Studio Pro; at runtime, both values would coexist as separate React props, with later entries potentially overwriting earlier ones.

### [P2] Render repeated elements from a data source
**Given** `tagUseRepeat` is enabled and `tagContentRepeatDataSource` is bound to a list entity  
**When** the data source resolves items  
**Then** one element is rendered per item, each with attribute and event resolvers bound to the corresponding `ObjectItem`; when the data source resolves to zero items, the widget renders nothing (`null`)

#### Edge Cases
- An empty data source produces no DOM output (null guard); no placeholder is rendered.
- React keys are composed of a stable ID prefix (from `useId`) and the item's own ID, falling back to the item index if no ID is available.

### [P3] Inject sanitized HTML content
**Given** `tagContentMode` is `innerHTML` and `tagContentHTML` (or its repeat variant) is bound to an expression  
**When** the expression resolves to an HTML string  
**Then** the string is sanitized via DOMPurify before insertion; XSS payloads (script tags, event-handler attributes, `javascript:` hrefs) are stripped

#### Edge Cases
- An empty string (`""`) for `tagContentHTML` activates the `dangerouslySetInnerHTML` path (not the children path), resulting in an empty sanitized element.
- Changes to `sanitizationConfigFull` after widget mount have no effect; the DOMPurify instance is memoized at mount time and a page reload or remount is required to apply a new config.
- Malformed JSON in `sanitizationConfigFull` throws a descriptive error at parse time.

### [P4] Attach event handlers to the rendered element
**Given** one or more event entries are configured with an `eventName` from the React synthetic event enumeration and an associated Mendix action  
**When** the corresponding browser event fires on the rendered element  
**Then** the Mendix action is executed; `stopPropagation` and `preventDefault` are called if the respective flags are set

#### Edge Cases
- Duplicate event names produce a design-time error.
- `eventAction` vs. `eventActionRepeat` is selected automatically based on whether an `ObjectItem` is present.

### [P5] Use a custom tag name
**Given** `tagName` is set to `__customTag__` and `tagNameCustom` contains a valid identifier matching `/^[a-z][\w.-]*$/i`  
**When** the widget renders  
**Then** an element with the custom tag name is emitted to the DOM

#### Edge Cases
- A custom tag name of `script` is rejected with a design-time error.
- Other potentially dangerous tags (e.g. `iframe`, `object`) are not blocked at design time; DOMPurify provides the runtime sanitization layer for innerHTML content only.

### [P6] Render as a CSS background image container
**Given** `isBackgroundImage` is not applicable (the widget has no background mode)  
*Not applicable — this widget renders foreground HTML elements only.*

## Functional Requirements

- FR-001: The widget MUST render the configured tag name as a real HTML element in the browser DOM.
- FR-002: When `tagUseRepeat` is `true` and `tagContentRepeatDataSource` has no items, the widget MUST render nothing (return `null`).
- FR-003: All content rendered via `innerHTML` mode MUST be sanitized through DOMPurify before insertion into the DOM.
- FR-004: The DOMPurify instance MUST be created once at component mount; subsequent changes to `sanitizationConfigFull` MUST NOT alter the active sanitizer without remount.
- FR-005: Malformed JSON in `sanitizationConfigFull` MUST produce a descriptive thrown error at parse time.
- FR-006: The `script` tag MUST be rejected as a valid `tagName` or `tagNameCustom` value with a Studio Pro design-time error.
- FR-007: An empty string for `tagContentHTML` MUST activate the `dangerouslySetInnerHTML` rendering path (not the children path).
- FR-008: The widget MUST apply DOMPurify sanitization to HTML content before setting `dangerouslySetInnerHTML`; attribute values passed via the `attributes` list MUST NOT be sanitized by DOMPurify.
- FR-009: The CSS class applied to the rendered element MUST always include `widget-html-element`, the Mendix `class` prop value, and any dynamic `class` attribute value — in that order.
- FR-010: Attribute-defined `style` values MUST take precedence over the Mendix `style` prop (attribute style wins on conflicting keys).
- FR-011: The `class` attribute in the attributes list MUST be mapped to the React `className` prop and merged with the existing class string.
- FR-012: Inline CSS strings supplied via the `style` attribute MUST be converted to React `CSSProperties` objects, with CSS custom properties (`--*`) passed through unchanged and vendor prefixes camelCased per React conventions (`-ms-` → `ms`, others → title-cased).
- FR-013: Void elements (as defined by the HTML specification) MUST suppress all content-related properties in Studio Pro.
- FR-014: Duplicate attribute names or duplicate event names MUST produce Studio Pro design-time errors.
- FR-015: The widget MUST be offline-capable (declared `offlineCapable="true"`).
- FR-016: The widget MUST have no default CSS styling; all presentation MUST be supplied by the consuming application.

## Props Reference

| Name | Type | Default | Caption | Description |
|------|------|---------|---------|-------------|
| `tagName` | enum (`div`, `span`, `p`, `h1`–`h6`, `ul`, `ol`, `li`, `a`, `img`, `input`, `button`, `section`, `article`, `__customTag__`) | — | Tag | Predefined HTML tag to render, or `__customTag__` to supply a custom name. |
| `tagNameCustom` | string | — | Custom tag name | Used when `tagName` is `__customTag__`. Must match `/^[a-z][\w.-]*$/i`. `script` is blocked. |
| `tagUseRepeat` | boolean | `false` | Repeat | When enabled, the widget renders one element per item from `tagContentRepeatDataSource`. |
| `tagContentRepeatDataSource` | ListValue | — | Data source | List data source driving element repetition. Required when `tagUseRepeat` is `true`. |
| `tagContentMode` | enum (`container` \| `innerHTML`) | — | Content mode | `container` renders child widgets; `innerHTML` sets raw HTML (sanitized by DOMPurify). |
| `tagContentHTML` | expression | — | HTML content | HTML string for innerHTML mode (non-repeat). |
| `tagContentContainer` | widgets | — | Content | Child widgets for container mode (non-repeat). |
| `tagContentRepeatHTML` | expression | — | HTML content | HTML string for innerHTML mode (repeat). Receives ObjectItem context. |
| `tagContentRepeatContainer` | widgets | — | Content | Child widgets for container mode (repeat). |
| `attributes` | object list | — | Attributes | List of HTML attributes to apply. Each item: `attributeName`, `attributeValueType`, and four value variants (expression/template × non-repeat/repeat). |
| `events` | object list | — | Events | List of DOM event handlers. Each item: `eventName` (React synthetic event), `eventAction`/`eventActionRepeat`, `eventStopPropagation`, `eventPreventDefault`. |
| `sanitizationConfigFull` | string (JSON) | `""` | Sanitization config | DOMPurify configuration as a JSON string. Applied at mount time; runtime changes require remount. Empty string uses DOMPurify defaults. |
| `class` | string | — | Class | Mendix `class` prop; merged into the element's `className` alongside `widget-html-element` and any dynamic `class` attribute. |
| `style` | CSSProperties | — | Style | Mendix `style` prop; merged with the `style` attribute value (attribute-defined style takes precedence). |

## Changelog

### v1.2.7 (2026-04-20)
- Security: Updated DOMPurify to 3.4.0.

### v1.2.6 (2026-03-31)
- Security: Updated DOMPurify to 3.3.3.

### v1.2.5 (2026-02-10)
- Added: license file and README for open-source dependencies.

## Open Questions
> Could not be determined from source code alone — requires human review
- [ ] The `tagContentRepeatDataSource` property is marked `required="true"` in the XML but is hidden when `tagUseRepeat` is false. Clarify whether Studio Pro enforces the required constraint even when the property is hidden, or whether the hidden state suppresses validation.
- [ ] Tags other than `script` that are potentially dangerous (e.g. `iframe`, `object`, `embed`) are not in the design-time blocklist. Confirm whether this is intentional policy or an oversight.
- [ ] The attribute-level `class`/`style` are not sanitized by DOMPurify — only innerHTML content is sanitized. Confirm whether attribute injection (e.g. injecting `onerror` via the attributes list) is a known accepted risk or should be mitigated.
