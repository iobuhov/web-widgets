# AccessibilityHelper

## Purpose

The Accessibility Helper widget enables Mendix developers to dynamically set or remove HTML attributes on arbitrary DOM elements within its content area at runtime. It is intended for accessibility compliance scenarios where standard Mendix widget properties do not expose the required ARIA or HTML attributes directly — for example, dynamically setting `aria-expanded`, `aria-label`, or `role` on nested elements based on application state. The widget does not render any visible UI of its own; it acts as a transparent wrapper around its content.

## User Scenarios

### [P1] Set ARIA attribute based on boolean condition
**Given** a developer places a widget (e.g., a Text Box) inside the Accessibility Helper content drop zone  
**When** a configured boolean expression evaluates to `true`  
**Then** the specified HTML attribute is set on the DOM element matched by `targetSelector`

#### Edge Cases
- When the boolean condition is `false`, the attribute is removed from the target element.
- When the boolean condition's `DynamicValue` is in `Loading` state, no attribute update occurs until all conditions resolve.
- When the target element is hidden via Mendix conditional visibility and then re-shown, attributes are re-applied automatically via the MutationObserver.

### [P2] Use an expression value as the attribute value
**Given** `valueSourceType` is set to `expression`  
**When** the bound expression value changes (e.g., via nanoflow or user input)  
**Then** the target element's attribute is updated in real-time to reflect the new value

#### Edge Cases
- If `valueSourceType` is `text`, the value is taken from a static or parameterized text template instead.
- Only one of `valueExpression` or `valueText` is active at a time; the other is hidden in the property panel.

### [P3] Configure multiple attributes on the same target
**Given** multiple entries are added to `attributesList`  
**When** any condition changes  
**Then** each attribute is independently evaluated and set or removed on the matched element

#### Edge Cases
- All conditions must resolve before any update cycle begins.
- Attributes are only written when the new value differs from the current DOM value (optimized DOM writes).

## Functional Requirements

- FR-001: The widget MUST observe DOM mutations within its content wrapper using `MutationObserver` with `attributes`, `childList`, and `subtree` observation, filtered to the specific attribute names it manages.
- FR-002: The widget MUST set a configured attribute on any DOM element matching `targetSelector` when the associated `attributeCondition` evaluates to `true` and is `Available`.
- FR-003: The widget MUST remove a configured attribute from matching elements when `attributeCondition` evaluates to `false`.
- FR-004: The widget MUST skip all attribute updates when any condition's `DynamicValue` is in `Loading` state.
- FR-005: The widget MUST NOT set attributes named `class`, `style`, `widgetid`, or `data-mendix-id`. This constraint is enforced at design time in Studio Pro via a validation error.
- FR-006: The widget MUST support both `text` template and `expression` value sources, selectable per attribute entry via `valueSourceType`.
- FR-007: The widget MUST avoid redundant DOM writes: if the current attribute value already matches the desired value, `setAttribute` MUST NOT be called again.
- FR-008: The widget MUST support offline Mendix applications (`offlineCapable="true"`).
- FR-009: The widget MUST require entity context (`needsEntityContext="true"`).

## Props Reference

| Name | Type | Default | Caption | Description |
|------|------|---------|---------|-------------|
| `targetSelector` | String | — | Target selector | CSS selector targeting the element(s) to receive attributes. Must be a valid CSS selector (e.g., `.mx-name-textbox1 input`). Required. |
| `content` | Widgets | — | Content | Drop zone for child widgets. The MutationObserver watches this area. Required. |
| `attributesList` | List | — | Attributes | List of attribute entries to apply. Each entry configures one HTML attribute. |
| `attributesList.attribute` | String | — | Attribute | HTML attribute name to set or remove. Prohibited values: `class`, `style`, `widgetid`, `data-mendix-id`. |
| `attributesList.valueSourceType` | Enum (`text` \| `expression`) | `text` | Value type | Determines whether the attribute value comes from a text template or a computed expression. |
| `attributesList.valueExpression` | Expression (String) | — | Value expression | Expression returning the attribute value string. Active when `valueSourceType` is `expression`. Optional. |
| `attributesList.valueText` | Text template | — | Value text | Text template for the attribute value. Active when `valueSourceType` is `text`. Optional. |
| `attributesList.attributeCondition` | Boolean expression | `true` | Condition | When `true`, the attribute is set; when `false`, it is removed. Defaults to always-set. |

## Changelog

- **v2.2.2 (2026-02-09):** Added license file and README documenting open-source dependencies. No behavioral changes.
- **v2.2.1 (2023-09-27):** Removed redundant code to improve browser load time. No behavioral changes.
- **v2.2.0 (2023-06-05):** Updated light/dark icons and tiles; changed structure-mode preview colors for dark/light modes.

## Open Questions

> Could not be determined from source code alone — requires human review

- [ ] What is the supported maximum number of entries in `attributesList`? No limit is declared in the XML.
- [ ] What happens when `targetSelector` matches zero elements? The current code silently applies no changes; whether this is intentional or an unhandled edge case is not confirmed by tests.
- [ ] Are multiple elements matched by `targetSelector` all updated, or only the first? The draft states "all elements matching", but no e2e test covers a selector matching more than one element.
