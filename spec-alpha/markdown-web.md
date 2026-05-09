# Markdown Viewer

## Purpose

The Markdown Viewer widget renders a Mendix string attribute as formatted HTML on a page using the `markdown-it` library. It is a read-only, display-only widget that converts Markdown syntax (headings, lists, tables, links, emphasis, etc.) into styled HTML output. It is suited for displaying rich-text content stored as Markdown strings in a Mendix domain model.

## User Scenarios

### [P1] Render Markdown content from an entity attribute
**Given** a Mendix page with a Markdown Viewer bound to a String attribute containing Markdown text  
**When** the page loads and the attribute becomes available  
**Then** the widget renders the Markdown as styled HTML, with tables, lists, and typographic enhancements applied

#### Edge Cases
- While the attribute is loading or unavailable, a Mendix progress spinner (`.mx-progress`) is displayed; there is no empty/placeholder state.
- Once the attribute becomes available, the widget transitions directly from spinner to rendered content.
- Both `ValueStatus.Loading` and `ValueStatus.Unavailable` are handled identically — both show the spinner.

### [P2] Auto-link and typographic enhancement
**Given** Markdown content containing bare URLs or straight quotation marks  
**When** the widget renders the content  
**Then** bare URLs are converted to clickable `<a>` links, straight quotes become typographic quotes, double dashes become em-dashes, and triple dots become ellipsis characters

#### Edge Cases
- These enhancements are always active (configured at module level); they cannot be disabled per widget instance.
- All instances of the Markdown Viewer on a page share a single `markdown-it` parser instance (module-level singleton).

### [P3] Inline HTML suppression
**Given** Markdown content containing raw HTML tags (e.g., `<script>`, `<div>`)  
**When** the widget renders the content  
**Then** the HTML tags are displayed as escaped text rather than executed as HTML

#### Edge Cases
- This is the `markdown-it` "default" preset behavior; raw HTML output is disabled by default.
- This is a security-relevant constraint: untrusted Markdown content cannot inject arbitrary HTML.

### [P4] Conditional visibility
**Given** the widget is configured with a Mendix visibility condition  
**When** the condition evaluates to false  
**Then** the widget is hidden; when it evaluates to true, the widget renders its content correctly

#### Edge Cases
- Conditional visibility is handled by the Mendix system property, not by widget code.

## Functional Requirements

- FR-001: The widget MUST render Markdown content using the `markdown-it` library with `"default"` preset, `typographer: true`, and `linkify: true`.
- FR-002: The widget MUST inject rendered HTML via `innerHTML` directly into its container element (not via React virtual DOM).
- FR-003: The widget MUST NOT render raw inline HTML from the Markdown source — HTML tags MUST be escaped and displayed as text.
- FR-004: The widget MUST display a Mendix progress spinner (`.mx-progress`) when the bound attribute is not yet available.
- FR-005: The widget MUST re-render whenever `stringAttribute.value` or `stringAttribute.status` changes.
- FR-006: The `markdown-it` parser instance MUST be created once at module scope and shared across all widget instances on the page.
- FR-007: Tables rendered from Markdown MUST display with bordered cells (1px solid `#ccc`), 8px padding, left-aligned text, and a `#f2f2f2` header background.
- FR-008: Images embedded in Markdown content MUST be constrained to a maximum width of 35% of the widget container.
- FR-009: The widget MUST be offline capable — it requires no server round-trip at render time.
- FR-010: The attribute type MUST be `String`; an unlimited string data type is recommended to avoid content truncation.

## Props Reference

| Name | Type | Default | Caption | Description |
|------|------|---------|---------|-------------|
| `stringAttribute` | `EditableValue<string>` | — | Content attribute | The Mendix string attribute containing the Markdown source text. The widget reads this value but never writes back to it. |
| `tabIndex` | `number?` | — | Tab index | Controls tab order for keyboard navigation. |

## Changelog

**v1.0.2 (2026-04-13):** Updated dependencies for security vulnerability mitigation. No functional changes.

**v1.0.1 (2026-02-10):** Added license file and open source dependency documentation. No functional changes.

**v1.0.0 (2024-08-16):** Initial release of the Markdown Viewer widget.

## Open Questions
> Could not be determined from source code alone — requires human review
- [ ] Is the 35% max-width constraint on images intentional and configurable in future versions, or a design oversight?
- [ ] Is there a plan to support Markdown extensions (e.g., footnotes, code highlighting) via `markdown-it` plugins?
- [ ] Is `linkify: true` always desirable, or should it be a configurable prop for scenarios where URL auto-linking is not wanted?
