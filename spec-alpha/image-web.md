# Image

## Purpose
The Image widget renders images, icons, or background images in a Mendix web application. It supports three data-source types — a Mendix entity image object, an external URL via a text template, and a Mendix WebIcon (glyph or icon collection) — and three interaction modes: no action, executing a Mendix action (Microflow, Nanoflow, Open Page), and full-screen enlargement via a lightbox overlay. The widget is offline-capable and does not require entity context.

## User Scenarios

### [P1] Display an image from a Mendix entity
**Given** `datasource` is set to `image` and `imageObject` is bound to a Mendix image entity attribute  
**When** the page loads and the entity image resolves  
**Then** an `<img>` element is rendered with the resolved file URI as its `src`; if the entity image is `Unavailable`, the widget falls back to `defaultImageDynamic` (only when that attribute is also configured and `imageObject` status is specifically `ValueStatus.Unavailable`)

#### Edge Cases
- `defaultImageDynamic` is only shown when `imageObject.status === ValueStatus.Unavailable`, not when the image is empty or loading.
- The fallback is only available when `imageObject.type === "dynamic"` (entity-bound); for static image objects there is no fallback display.

### [P2] Display an image from an external URL
**Given** `datasource` is set to `imageUrl` and `imageUrl` is bound to a text template expression  
**When** the expression resolves to a URL string  
**Then** an `<img>` element is rendered with that URL as its `src`

#### Edge Cases
- Text templates containing unresolved `{variable}` tokens are not rendered in Studio design mode (falls back to placeholder SVG).
- A dynamic URL association bound to an entity attribute is a known e2e-broken scenario (currently skipped in tests).

### [P3] Display an icon or glyph
**Given** `datasource` is set to `icon` and `imageIcon` is bound to a Mendix WebIcon  
**When** the page renders  
**Then** for a glyph, a `<span>` with the `glyphicon` and icon-class CSS classes is rendered; for an icon collection image, the icon image is rendered; `aria-label` and `role="img"` are applied for accessibility

#### Edge Cases
- Icons cannot be rendered as background images; `isBackgroundImage` is forced to `false` for icon datasource.
- Dimension properties are suppressed for icon datasource; only `iconSize` controls the display size.
- Icon container alignment issues were present before v1.5.1 (October 2025).

### [P4] Open full-screen lightbox on click
**Given** `onClickType` is set to `enlarge`  
**When** the user clicks the image  
**Then** a full-screen lightbox overlay opens showing the image at full resolution; when thumbnail mode is also active, the lightbox shows the original unprocessed URL (without `?thumb=true`); clicking the backdrop or the close button closes the lightbox

#### Edge Cases
- Background image mode does not support lightbox enlargement.
- Lightbox state is local to each Image widget instance; multiple Image widgets on the same page operate independently.
- The lightbox uses `react-overlays/Modal`. The close button requires a separate `onClose` prop because the library's `onHide` fires only on the true backdrop element, not on child elements.
- The lightbox backdrop prevents click events from propagating to parent container handlers (`stopPropagation`).

### [P5] Display thumbnail with full-resolution lightbox
**Given** `displayAs` is set to `thumbnail` and `datasource` is `image`  
**When** the image is displayed  
**Then** `?thumb=true` is appended to the image URL for the thumbnail display; the lightbox shows the original URL without the `thumb` parameter

#### Edge Cases
- Thumbnail URL processing uses `new URL()` and requires an absolute URL. Relative URLs will cause a parse error.
- `displayAs` is only available when `datasource` is `image`; it is not applicable for `imageUrl` or `icon` types.
- `previewMode` disables thumbnail URL processing and lightbox.

### [P6] Render as a CSS background image with content dropzone
**Given** `isBackgroundImage` is `true` and `datasource` is `image` or `imageUrl`  
**When** the page renders  
**Then** the widget renders as a `<div>` with `background-image: url('...')` applied as an inline style; `background-repeat: no-repeat` and `background-position: center` are applied via SCSS; child widgets are rendered inside the div

#### Edge Cases
- Background images do not support lightbox. Clicks on the background div fire `onClick` directly without lightbox mechanics.
- `isBackgroundImage` is not available for icon datasource.
- `alternativeText` is suppressed for background images (replaced by the child widget dropzone).

### [P7] Execute a Mendix action on click
**Given** `onClickType` is set to `action` and `onClick` is bound to a Mendix action (Microflow, Nanoflow, Open Page)  
**When** the user clicks the image (mouse or keyboard — `Enter`/`Space`)  
**Then** the configured action executes; the image receives `role="button"` and `tabIndex=0` for keyboard accessibility

#### Edge Cases
- Keyboard activation (`Enter`/`Space`) fires the same handler as a mouse click.
- `role="img"` is applied instead when no `onClick` is configured.
- When no alt text is provided, neither `alt` nor `aria-label` is set on the element.

## Functional Requirements

- FR-001: The widget MUST support three datasource types: `image` (entity image object), `imageUrl` (text template), and `icon` (WebIcon).
- FR-002: When `datasource` is `image` and `imageObject.status` is `ValueStatus.Unavailable`, the widget MUST display `defaultImageDynamic` if configured; this fallback MUST NOT trigger for other status values.
- FR-003: When `displayAs` is `thumbnail`, the widget MUST append `?thumb=true` to the image URL for thumbnail display using the URL API.
- FR-004: When `onClickType` is `enlarge`, the lightbox MUST show the full-resolution image URL without the `?thumb=true` parameter, regardless of the `displayAs` setting.
- FR-005: When `datasource` is `icon`, `isBackgroundImage` MUST be forced to `false`; icons MUST NOT be rendered as CSS background images.
- FR-006: Lightbox state MUST be local per widget instance; opening or closing one lightbox MUST NOT affect other Image widgets on the same page.
- FR-007: The lightbox backdrop MUST call `event.stopPropagation()` to prevent click events from reaching parent container handlers.
- FR-008: Clickable images (with `onClick`) MUST receive `role="button"` and `tabIndex=0`; non-clickable images MUST receive `role="img"`.
- FR-009: Keyboard activation (`Enter` or `Space`) on a clickable image or icon MUST fire the same handler as a mouse click.
- FR-010: `alternativeText` MUST be applied as the HTML `alt` attribute on `<img>` elements and as `aria-label` on icon/glyph `<span>` elements.
- FR-011: When no `alternativeText` is provided, the `alt` and `aria-label` attributes MUST NOT be set.
- FR-012: The widget MUST be offline-capable (declared `offlineCapable="true"`) and MUST NOT require entity context.
- FR-013: When the resolved image source is `undefined` or an empty string, the widget wrapper MUST receive the `hidden` CSS class.
- FR-014: The `responsive` prop MUST add the `mx-image-viewer-responsive` CSS class, constraining the image to `max-width: 100%; max-height: 100%`.
- FR-015: Width and height dimensions MUST support pixels and percentage units; height MUST additionally support `auto` and `viewport` (vh) units.
- FR-016: `minHeight` and `maxHeight` constraints MUST only apply when `heightUnit` is `auto`; zero values for min/max height MUST be suppressed (treated as not set).
- FR-017: The widget MUST be compatible with Content Security Policy strict mode (no `unsafe-inline` required).
- FR-018: Text template image URLs containing unresolved `{variable}` tokens MUST NOT be rendered in Studio design mode; a placeholder SVG MUST be shown instead.

## Props Reference

| Name | Type | Default | Caption | Description |
|------|------|---------|---------|-------------|
| `datasource` | enum (`image` \| `imageUrl` \| `icon`) | — | Data source | Selects the image type: entity image object, external URL text template, or Mendix WebIcon. |
| `imageObject` | DynamicValue\<WebImage\> | — | Image | Mendix image entity attribute. Only for `datasource=image`. |
| `defaultImageDynamic` | DynamicValue\<WebImage\> | — | Default image | Fallback shown when `imageObject.status` is `Unavailable`. Only for `datasource=image` with a dynamic (entity-bound) imageObject. |
| `imageUrl` | DynamicValue\<string\> | — | URL | External image URL via text template. Only for `datasource=imageUrl`. |
| `imageIcon` | DynamicValue\<WebIcon\> | — | Icon | Mendix WebIcon (glyph, image, or icon collection). Only for `datasource=icon`. |
| `iconSize` | number | `14` | Icon size | Font size in pixels for glyph/icon rendering. Only for `datasource=icon`. |
| `displayAs` | enum (`fullImage` \| `thumbnail`) | — | Display as | Appends `?thumb=true` to the image URL for thumbnail display. Only for `datasource=image`. |
| `onClickType` | enum (`action` \| `enlarge`) | — | On click | Interaction type: execute a Mendix action or open the lightbox. |
| `onClick` | ActionValue | — | On click action | Mendix action (Microflow, Nanoflow, Open Page). Only when `onClickType=action`. |
| `alternativeText` | DynamicValue\<string\> | — | Alternative text | Alt text for `<img>` elements; `aria-label` for icons/glyphs. |
| `isBackgroundImage` | boolean | `false` | Background image | Renders the widget as a CSS background with a content dropzone for child widgets. Not available for `datasource=icon`. |
| `responsive` | boolean | `false` | Responsive | Constrains image to its natural size (`max-width: 100%; max-height: 100%`). Not available for `datasource=icon`. |
| `widthUnit` | enum (`pixels` \| `percentage`) | — | Width unit | Unit for the width value. |
| `width` | number | — | Width | Numeric width value in the selected unit. |
| `heightUnit` | enum (`auto` \| `pixels` \| `percentage` \| `viewport`) | — | Height unit | Unit for the height value. `auto` activates min/max height constraints. |
| `height` | number | — | Height | Numeric height value in the selected unit. Not shown when `heightUnit=auto`. |
| `minHeightUnit` | enum (`none` \| `pixels` \| `percentage` \| `viewport`) | `none` | Min height unit | Only active when `heightUnit=auto`. |
| `minHeight` | number | — | Min height | Minimum height value. Suppressed when 0. |
| `maxHeightUnit` | enum (`none` \| `pixels` \| `percentage` \| `viewport`) | `none` | Max height unit | Only active when `heightUnit=auto`. |
| `maxHeight` | number | — | Max height | Maximum height value. Suppressed when 0. |
| `class` | string | — | Class | CSS class applied to the outer wrapper element. |
| `style` | CSSProperties | — | Style | Inline style applied to the outer wrapper element. |

## Changelog

### v1.5.1 (2025-10-29)
- Fixed: icon container scaling and alignment issue.

### v1.5.0 (2025-08-29)
- Added: min/max height constraints and viewport (vh) height unit.

### v1.4.3 (2023-10-05)
- Fixed: icon size configurability for new icon collections; improved structure preview.

## Open Questions
> Could not be determined from source code alone — requires human review
- [ ] The dynamic URL association e2e test is marked `test.skip` with no explanation. Confirm whether this is a known platform bug, a configuration issue in the test environment, or a deprecated feature.
- [ ] The widget is skipped entirely for the Mendix modern React client (`MODERN_CLIENT=true`). Confirm the official support policy for the modern client and whether migration is planned.
- [ ] Firefox is excluded from template grid and tab container e2e tests. Confirm whether this is a widget issue, a test infrastructure issue, or an accepted platform limitation.
- [ ] Width does not support an `auto` unit in `constructStyleObject`; the `auto` case is handled upstream by skipping width emission. Confirm intended behavior when no explicit width is configured (browser default vs. explicit 100%).
