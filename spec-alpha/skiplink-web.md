# SkipLink

## Purpose

The SkipLink widget implements WCAG 2.4.1 (Bypass Blocks) — a keyboard-accessible "skip to main content" anchor that enables keyboard and screen reader users to bypass repetitive navigation and jump directly to the primary content area. The link is visually hidden via a CSS `transform: translateY(-120%)` and revealed only when focused, appearing at the top of the page with a smooth 200ms CSS transition. It is categorized under "Accessibility" in Studio Pro and is web-only.

## User Scenarios

### [P1] Keyboard user bypasses navigation
**Given** a Mendix page with a navigation bar above the main content and the SkipLink widget configured  
**When** the user presses Tab as the first keyboard interaction on the page  
**Then** the skip link becomes the first focused element and becomes visible at the top of the page  

**When** the user presses Enter  
**Then** focus moves programmatically to the main content area (the element matching `mainContentId`, or the first `<main>` tag if `mainContentId` is empty)  

#### Edge Cases
- If the target element does not have a `tabindex`, a temporary `tabindex="-1"` is added to allow programmatic focus; it is removed on the first `blur` event to keep the target out of the tab order thereafter.
- If the `#root` element is not found in the DOM, a console error is logged and the link is not inserted.
- If the target element is not found (by ID or `<main>` tag), a console error is logged and focus is not moved.
- The `<a>` element has an `href` attribute for progressive enhancement — browsers without JavaScript can still navigate using the anchor link.

### [P2] Screen reader announces skip link
**Given** a configured SkipLink widget  
**When** a screen reader reaches the first interactive element on the page  
**Then** it announces the `linkText` (e.g., "Skip to main content") as a hyperlink  

## Functional Requirements

- FR-001: System MUST insert the skip link `<a>` as the first child of `document.getElementById("root")` using a React portal, ensuring it is the first interactive element in DOM order.
- FR-002: The link MUST be visually hidden by default via `transform: translateY(-120%)` with a 200ms CSS transition, and revealed via `transform: translateY(0)` on `:focus`.
- FR-003: When activated (Enter key), the system MUST programmatically focus the target element: first try `document.getElementById(mainContentId)` if `mainContentId` is non-empty; otherwise fall back to the first `<main>` element.
- FR-004: If the target element lacks a `tabindex` attribute, the system MUST temporarily add `tabindex="-1"` to enable programmatic focus, and MUST remove it on the target's first `blur` event.
- FR-005: The `<a>` element MUST have `href="#${mainContentId}"` (or `href="#"` when `mainContentId` is empty) for progressive enhancement.
- FR-006: The portal container `<div>` MUST be cleaned up (removed from the DOM) when the React component unmounts.
- FR-007: The widget MUST be web-only (`supportedPlatform="Web"`) and require no entity context.
- FR-008: Studio Pro MUST show a validation error if `linkText` is empty (since it has a default, this guards against accidental deletion).
- FR-009: The skip link visual style MUST use `z-index: 1000` so it appears above all other page content when focused.

## Props Reference

| Name | Type | Default | Caption | Description |
|------|------|---------|---------|-------------|
| `linkText` | `string` | `"Skip to main content"` | Link text | The visible text of the skip link shown when focused; required; Studio Pro shows a validation error if empty |
| `mainContentId` | `string` | `""` | Main content ID | The `id` attribute of the main content element to focus; when empty, the first `<main>` tag is used as the target |

System properties supported: Name, TabIndex (defaults to 0, ensuring the link is first in tab order).

## Changelog

- **Unreleased**: Widget created. Not yet published to the Mendix Marketplace.

## Open Questions

> Could not be determined from source code alone — requires human review
- [ ] What is the intended Mendix Marketplace release timeline for this widget?
- [ ] Should the widget support custom CSS classes on the `<a>` element for theme customization?
- [ ] Is the empty `helpUrl` in the XML intentional (pre-release), or does documentation exist at an external URL?
