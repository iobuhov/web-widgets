# Draft: accessibility-helper-web

Widget: `accessibility-helper-web`
Task: EX-001
Status: first run — all source files covered

---

## src/AccessibilityHelper.tsx

**1. Purpose:** This is the main runtime component of the Accessibility Helper widget. It renders a `<div>` wrapper around its `content` children and attaches dynamic HTML attributes to elements matched by a CSS selector (`targetSelector`) within that wrapper.

**2. Logic:** The component uses a `MutationObserver` to watch for DOM changes inside the content wrapper. When the observer fires, it re-evaluates all configured attribute conditions and updates or removes the target element's attributes accordingly. Attributes are only set when their associated `DynamicValue<boolean>` condition is `Available` and `true`; they are removed when the condition is `false`. Attribute values come from either a `DynamicValue<string>` expression or a text template, selected by `valueSourceType`. The update logic avoids unnecessary DOM writes by comparing the current attribute value before calling `setAttribute`.

**3. Documentable behavior:** When any condition's `DynamicValue` is in `Loading` state, the entire update cycle is skipped (no observer is started) until all conditions resolve. The MutationObserver is configured to watch `attributes`, `childList`, and `subtree` — it only observes the specific attribute names in `attributeFilter` to limit noise. The observer skips redundant updates: if the mutation is a self-caused attribute write that already matches the desired value, it will not trigger another update pass.

**4. User-facing:** No (this component renders only a transparent `<div>` wrapper; users interact with the content placed inside it, not the widget itself).

**5. New learnings:** The use of `MutationObserver` is explicitly motivated by Mendix conditional visibility: when a child widget is hidden/shown via conditional visibility, the parent widget does not re-render, so `useEffect` alone would miss the DOM state change. The observer bridges this gap. The `attributeFilter` optimization means the observer only fires for attributes the widget itself manages, preventing infinite loops.

---

## src/AccessibilityHelper.xml

**1. Purpose:** The widget descriptor XML defines the widget's identity, prop schema, Studio Pro editor UI, and platform metadata. It is the authoritative source of prop types, captions, descriptions, and allowed values.

**2. Logic:** Declares three top-level properties: `targetSelector` (string), `content` (widgets drop zone), and `attributesList` (list of objects). Each list item has five sub-properties: `attribute` (string — the HTML attribute name), `valueSourceType` (enumeration: `text` | `expression`, default `text`), `valueExpression` (optional expression returning String), `valueText` (optional text template), and `attributeCondition` (boolean expression, default `true`).

**3. Documentable behavior:** The widget requires entity context (`needsEntityContext="true"`) and supports offline (`offlineCapable="true"`). Its Studio Pro category is `Input elements`. The description on the `attribute` property explicitly lists prohibited values: `class`, `style`, `widgetid`, `data-mendix-id`. The `targetSelector` description specifies it must be a valid CSS selector (e.g. `.mx-name-texbox1 input`).

**4. User-facing:** Indirectly — the XML controls what the developer configures in Studio Pro. End users never see this file, but the constraints it defines (required fields, enumeration values, captions) shape the developer experience.

**5. New learnings:** The widget supports both `text` templates and `expression` values for attribute content, giving developers flexibility between static/parameterized text and computed values. The `defaultValue="true"` on `attributeCondition` means by default an attribute is always set (unless explicitly conditioned).

---

## src/AccessibilityHelper.editorConfig.ts

**1. Purpose:** Provides Studio Pro editor-time behavior: property visibility rules, validation checks, and the visual structure-mode preview of the widget canvas.

**2. Logic:** `getProperties` hides `valueExpression` when `valueSourceType === "text"` and hides `valueText` when `valueSourceType === "expression"`, ensuring only the relevant value field is shown in the property panel. `check` validates that no `attribute` entry in `attributesList` matches any of the four prohibited attributes (`class`, `style`, `widgetid`, `data-mendix-id`), reporting a `Problem` with a link to documentation. `getPreview` renders a bordered container with a content `DropZone` and a label row showing the target selector string.

**3. Documentable behavior:** Behavioral constraint: mutually exclusive visibility of `valueExpression` vs `valueText` based on `valueSourceType`. Validation constraint: four attributes are blocked at design time, matching the restriction documented in the XML's description text. The preview shows `Target [Target selector]` placeholder when `targetSelector` is empty, or `Target {value}` when filled.

**4. User-facing:** No (editor-only file; runs inside Studio Pro, not in the browser runtime).

**5. New learnings:** The prohibited attributes list (`PROHIBITED_ATTRIBUTES`) is enforced as a compile-time constant here and cross-checked at design time. This is the only place where enforcement happens programmatically — the XML description is informational only. The `getPreview` uses `palette.background.topbarStandard` as a fallback background when the content drop zone is empty.

---

## typings/AccessibilityHelperProps.d.ts

**1. Purpose:** Auto-generated TypeScript type definitions derived from `AccessibilityHelper.xml`. Provides strongly-typed interfaces for both the runtime component (`AccessibilityHelperContainerProps`) and the Studio Pro preview (`AccessibilityHelperPreviewProps`).

**2. Logic:** Defines `AttributesListType` for runtime use (with `DynamicValue<string>` and `DynamicValue<boolean>` fields) and `AttributesListPreviewType` for preview use (where all values are plain strings). `AccessibilityHelperContainerProps` adds standard Mendix widget fields (`name`, `class`, `style`, `tabIndex`) plus the three widget-specific props. `AccessibilityHelperPreviewProps` adds Studio Pro preview fields (`renderMode`, `readOnly`, `translate`).

**3. Documentable behavior:** `valueExpression` and `valueText` are both `optional` at the type level (marked `?`), reflecting that only one is required depending on `valueSourceType`. The `content` type differs between runtime (`ReactNode`) and preview (`{ widgetCount: number; renderer: ComponentType<...> }`). The `className` field in `AccessibilityHelperPreviewProps` is deprecated since Mendix 9.18.0.

**4. User-facing:** No (TypeScript types used only at development/compile time).

**5. New learnings:** The distinction between `DynamicValue<string>` (runtime — observable, can be Loading/Available/Unavailable) and plain `string` (preview — static design-time value) reveals that the runtime component must handle loading states that the editor preview does not.

---

## src/package.xml

**1. Purpose:** Mendix client module manifest. Declares the widget bundle name, version, and the file path pattern where the widget's compiled assets are placed in the mpk package.

**2. Logic:** Maps `AccessibilityHelper` version `2.2.2` to the XML descriptor file and the compiled asset path `com/mendix/widget/web/accessibilityhelper/`.

**3. Documentable behavior:** The client module version (`2.2.2`) matches `package.json`. This file is part of the mpk packaging process; it tells the Mendix runtime where to find the widget descriptor and assets.

**4. User-facing:** No (packaging/deployment artifact).

**5. New learnings:** The asset path `com/mendix/widget/web/accessibilityhelper/` follows the Java-style reverse-domain naming convention used by Mendix for widget distribution. This is consistent with the widget ID in `AccessibilityHelper.xml` (`com.mendix.widget.web.accessibilityhelper.AccessibilityHelper`).

---

## CHANGELOG.md

**1. Purpose:** Documents the release history of the widget. Serves as evidence of implied requirements, design decisions, and past fixes.

**2. Logic:** No code logic — version entries with `Added`, `Fixed`, and `Changed` sections.

**3. Documentable behavior:** v1.1.0 (2021-07-21): Structure mode preview added. v2.0.0 (2021-09-28): Toolbox category and tile image added for Studio & Studio Pro. v2.1.0 (2021-12-23): Dark mode support for structure mode preview; dark icons for tile and list views. v2.2.0 (2023-06-05): Updated light/dark icons and tiles; changed structure mode preview colors for dark/light modes. v2.2.1 (2023-09-27): Removed redundant code to improve browser load time. v2.2.2 (2026-02-09): Added license file and README documenting open-source dependencies.

**4. User-facing:** No (developer/operator documentation).

**5. New learnings:** The widget has been stable at its core feature set since v1.x — all changes since v2.0.0 have been UI polish (dark mode, icons, colors) and housekeeping (redundant code removal, license). No behavioral changes are recorded after v1.1.0. The "redundant code removal" in v2.2.1 suggests there was once additional logic that was simplified.

---

## e2e/AccessibilityHelper.spec.js

**1. Purpose:** Playwright end-to-end test suite covering the widget's runtime behavior in a live Mendix application. Provides behavioral evidence that cannot be inferred from source alone.

**2. Logic:** Tests are grouped under "with single target" and "with multiple targets". Each test manipulates radio buttons and text inputs in the test app to trigger attribute changes, then asserts the presence or absence of specific HTML attributes on target elements (`.mx-name-text3`, `.mx-name-text4`).

**3. Documentable behavior:** Confirmed behaviors from tests: (1) When condition is `true`, the configured attribute is set on the target. (2) When condition is `false`, the attribute is removed from the target. (3) Expression-typed values track input field changes and update the attribute in real-time. (4) Nanoflow (NF) changes to the expression value also propagate correctly. (5) Attributes are re-applied even when the target widget's own props change (e.g. after a textbox re-render). (6) Attributes are re-applied after a target element is conditionally hidden and then shown again. (7) All behaviors apply consistently to multiple targets matched by the same selector.

**4. User-facing:** No (test code), but the behaviors it verifies are entirely user-observable: accessibility attributes appear/disappear on DOM elements in response to application state.

**5. New learnings:** The test "sets target attributes even though target is conditionally shown after being hidden" directly confirms the MutationObserver rationale in `AccessibilityHelper.tsx`. The "updates target attributes using a NF" test shows that Nanoflow-driven expression updates are supported, meaning the `DynamicValue` lifecycle handles microflow/nanoflow action results. The session logout hook (`window.mx.session.logout()`) reveals a Mendix-specific licensing constraint: each test opens a session, and the license allows only 5 concurrent sessions.

---

## packages/shared/widget-plugin-platform/src/preview/structure-preview-api.ts

_(Local workspace dependency, imported by `AccessibilityHelper.editorConfig.ts`)_

**1. Purpose:** Provides the TypeScript type definitions and factory functions for Mendix Studio Pro's structure-mode preview API. This is a shared library used by all widgets in the monorepo to build their canvas preview representations.

**2. Logic:** Defines a union type `StructurePreviewProps` covering seven preview element types: `Image`, `Container`, `RowLayout`, `Text`, `DropZone`, `Selectable`, and `Datasource`. Each type has a corresponding factory function (e.g. `container()`, `rowLayout()`, `text()`, `dropzone()`) that accepts styling options and returns a typed object. Also exports `structurePreviewPalette` with `dark` and `light` color themes for consistent preview styling across widgets. The `colorWithAlpha` utility converts a hex color + integer alpha percentage to a hex color with alpha prefix.

**3. Documentable behavior:** `AccessibilityHelper.editorConfig.ts` uses `Container`, `RowLayout`, `DropZone`, and `Text` types — exactly the four types needed to render a bordered box with a content area and a label. The palette provides `topbarStandard` as the empty-state background color. No behavioral constraints flow from this file into the runtime widget.

**4. User-facing:** No (editor/preview only; not included in the runtime widget bundle).

**5. New learnings:** The palette distinguishes between `topbarData` (used for data-bound widgets) and `topbarStandard` (used for standard/non-data widgets). `AccessibilityHelper` uses `topbarStandard` when its content drop zone is empty, indicating it is categorized as a non-data widget in preview styling terms.
