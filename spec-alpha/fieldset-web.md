# Fieldset

## Purpose

The Fieldset widget wraps child Mendix widgets in a native HTML `<fieldset>` element with an optional `<legend>` caption. It solves the need to semantically group related form inputs — communicating their shared purpose to screen readers and browsers — while remaining a lightweight, layout-neutral container. Use it whenever a page section contains a cohesive set of form controls that benefit from a visible or accessible group label.

## User Scenarios

### [P1] Display a labeled group of form inputs

**Given** a Mendix page with input widgets placed inside a Fieldset widget and a legend configured  
**When** the page renders in a browser  
**Then** the system renders a `<fieldset>` element containing a `<legend>` as its first child, followed by the child input widgets

#### Edge Cases

- If the legend is bound to a Mendix attribute or expression and the value is not yet available, the legend resolves to `undefined` and no `<legend>` element is rendered until the value loads.
- An empty string legend value causes the `<legend>` element to be omitted (falsy check).

---

### [P2] Display a fieldset without a legend

**Given** a Mendix page with a Fieldset widget where no legend is configured  
**When** the page renders  
**Then** the system renders a `<fieldset>` element without any `<legend>` child; content widgets render directly inside the fieldset

#### Edge Cases

- The absence of a legend is valid; the fieldset remains structurally and semantically correct as an unlabeled group.
- In Studio Pro's page tree, the widget caption falls back to the static string `"Fieldset"` when no legend is configured.

---

### [P3] Conditionally hide content while keeping the fieldset visible

**Given** child widgets inside the Fieldset widget with Mendix conditional visibility configured  
**When** the visibility condition evaluates to hidden for all child widgets  
**Then** the `<fieldset>` container and `<legend>` remain visible in the DOM; only the child widgets are hidden

#### Edge Cases

- The fieldset itself has no built-in conditional visibility — it is always rendered when placed on a page. Only its children may be conditionally hidden.

---

### [P4] Reactively update the legend from a bound attribute

**Given** the Fieldset legend bound to a Mendix attribute whose value changes at runtime  
**When** the user edits the field driving the legend expression  
**Then** the `<legend>` text updates live without a page reload

#### Edge Cases

- The reactive update is provided by the Mendix DynamicValue mechanism; no widget-level polling or explicit subscription is required.

---

## Functional Requirements

- FR-001: The system MUST render a native HTML `<fieldset>` element as the widget's root element.
- FR-002: The system MUST render a `<legend>` element as the first child of `<fieldset>` when the `legend` prop is a non-empty truthy string.
- FR-003: The system MUST omit the `<legend>` element when `legend` is `undefined` or an empty string.
- FR-004: The system MUST render child widgets inside the `<fieldset>` regardless of whether a legend is present.
- FR-005: The system MUST apply the Mendix-provided `class`, `style`, `name`, and `tabIndex` attributes to the `<fieldset>` DOM element.
- FR-006: The `legend` property MUST support dynamic Mendix expressions and attribute bindings (`textTemplate`), not only static strings.
- FR-007: The legend value MUST update reactively when the underlying Mendix attribute or expression changes without a page reload.
- FR-008: The widget MUST be usable in offline Mendix applications (`offlineCapable: true`).
- FR-009: The widget MUST require Mendix platform version 9.6.0 or higher.
- FR-010: The widget MUST allow the `<fieldset>` to be keyboard-focusable via the `tabIndex` system property.

## Props Reference

| Name | Type | Default | Caption | Description |
|------|------|---------|---------|-------------|
| `legend` | `DynamicValue<string>` (optional) | — | Legend | The text displayed in the `<legend>` element. Supports attribute bindings and expressions. Omitted when not configured or when the resolved value is falsy. |
| `content` | `ReactNode` (widgets slot) | — | Content | Drop zone for child Mendix widgets rendered inside the `<fieldset>`. Accepts arbitrary widgets. |
| `name` | `string` | — | Name | Standard Mendix system property. Applied as the `name` attribute on the `<fieldset>` DOM element. |
| `class` | `string` | — | Class | Mendix CSS class system property. Applied as `className` on the `<fieldset>`. No default class names are added by the widget. |
| `style` | `CSSProperties` (optional) | — | Style | Mendix inline style system property. Applied to the `<fieldset>` DOM element. |
| `tabIndex` | `number` (optional) | — | Tab index | System property controlling keyboard focus order. Applied as `tabindex` on the `<fieldset>`. When set, the fieldset itself becomes keyboard-focusable. |

## Changelog

**v3.2.2** (2026-02-09): Added license file and readme for open source dependencies.

**v3.2.1** (2023-09-27): Removed redundant code to improve widget load time.

**v3.2.0** (2023-06-05): Page explorer caption now displays the legend value; updated icons and dark/light mode Structure preview colors.

## Open Questions

> Could not be determined from source code alone — requires human review

- [ ] Is there a recommended approach to prevent the `<fieldset>` border from affecting page layout, or should developers use custom CSS to remove the default browser border?
- [ ] Does the `tabIndex` prop on the `<fieldset>` element interact correctly with all child widgets' own tab order in Mendix's focus management?
- [ ] The `translate` i18n function is present in `FieldsetPreviewProps` but unused — is localization of the legend or caption label planned for a future version?
