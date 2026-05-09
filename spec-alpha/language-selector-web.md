# LanguageSelector

## Purpose

The LanguageSelector widget enables end-users to switch the active UI language of a Mendix application at runtime. It renders a floating dropdown menu that lists configured language options, reflects the user's currently active language, and on selection commits the chosen language to the Mendix `System.User_Language` reference on the user object before reloading the application with preserved state. The widget is designed to be embedded in navigation bars or page headers and optionally hidden when only a single language is available.

## User Scenarios

### [P1] Switch application language

**Given** a user is authenticated in a Mendix app with two or more configured languages  
**When** the user clicks (or hovers, depending on configuration) the language selector trigger  
**Then** a floating dropdown listing all available languages appears, with the current language visually indicated (bold weight + `active` CSS class), and the user may select a different language

**When** the user selects a new language (by click, Enter, or Space)  
**Then** the selection is committed to `System.User_Language` on the Mendix user object via `window.mx.data.commit`, and the page reloads via `window.mx.reloadWithState()` — preserving navigation state — with the UI rendered in the selected language

#### Edge Cases
- If the language list contains fewer than 2 entries AND `hideForSingle` is `true`, the widget renders `null` and is invisible.
- The widget renders `null` until the language list is fully populated from the datasource.
- If an item is in the `to` side of the typeahead buffer (`isTypingRef.current === true`), Space does not trigger selection, allowing continued typeahead.

### [P2] Keyboard navigation

**Given** the dropdown is open  
**When** the user presses arrow keys  
**Then** keyboard focus cycles through language options, with the focused item receiving the `highlighted` CSS class

**When** the user types characters  
**Then** typeahead navigation moves focus to the first option whose label begins with the typed prefix

#### Edge Cases
- Focus is trapped within the dropdown via `FloatingFocusManager` (`modal={false}`) — the screen reader virtual cursor can still move outside the dropdown.
- `aria-activedescendant` is set to an empty string while the dropdown is open (partial ARIA implementation — see Open Questions).

### [P3] Navbar embedding

**Given** the widget is placed inside a Mendix `.navbar-brand` element  
**When** the page renders  
**Then** text color and arrow icon color are overridden using `--bg-color-secondary` and an inverted arrow filter, ensuring legibility against dark navbar backgrounds

## Functional Requirements

- FR-001: The system MUST commit the selected language to the `System.User_Language` Mendix attribute before reloading.
- FR-002: The system MUST reload the application using `window.mx.reloadWithState()` after a language selection, preserving navigation state.
- FR-003: The system MUST hide the widget (render `null`) when `hideForSingle` is `true` and fewer than 2 language options are available.
- FR-004: The system MUST NOT render any visible UI until the language list is populated from the datasource.
- FR-005: The system MUST support two open triggers: `click` (default) and `hover`.
- FR-006: The hover trigger MUST use a safe-polygon hit zone to prevent unintended menu close when moving the cursor from the trigger to the menu.
- FR-007: The dropdown position MUST fall back automatically to one of the four cardinal positions (`top`, `right`, `bottom`, `left`) when the configured position does not fit the viewport.
- FR-008: The dropdown MUST use `position: fixed` strategy so it is correctly positioned inside scrollable containers (e.g., nested accordion panels).
- FR-009: Typeahead MUST be supported — typing letters while the dropdown is open moves keyboard focus to the first matching option.
- FR-010: The trigger element MUST have `aria-label` defaulting to `"Language selector"`.
- FR-011: The trigger element MUST receive ARIA `role="combobox"` and the floating menu MUST receive `role="listbox"`, assigned by the `useRole` floating-ui hook.
- FR-012: Each language item in the dropdown MUST have `role="option"`.
- FR-013: The arrow indicator MUST animate between `arrow-up` and `arrow-down` states (180° CSS rotation, `0.2s ease-in-out` with `50ms` delay) to reflect dropdown open/closed state.
- FR-014: The widget MUST expose an optional `screenReaderLabelCaption` prop (`DynamicValue<string>`) for a custom accessible label on the trigger.
- FR-015: The widget MUST be offline-capable (`offlineCapable="true"`) for rendering; the language switch action itself requires connectivity.

## Props Reference

| Name | Type | Default | Caption | Description |
|------|------|---------|---------|-------------|
| `languageOptions` | Datasource (list, required) | — | Languages | Data source providing the list of available language objects. |
| `languageCaption` | Expression (string, per-item) | — | Language caption | Expression evaluated per item to derive the display name of each language. |
| `position` | Enum (`left` \| `right` \| `top` \| `bottom`) | `bottom` | Position | Controls where the dropdown appears relative to the trigger element. |
| `trigger` | Enum (`click` \| `hover`) | `click` | Trigger | Whether the dropdown opens on click or hover interaction. |
| `hideForSingle` | Boolean | `true` | Hide for single language | When `true`, hides the widget entirely if fewer than 2 languages are available. |
| `screenReaderLabelCaption` | Expression (string, optional) | — | Screen reader label | Optional accessible label text applied to the trigger element for screen readers. |

## Changelog

- **v1.1.4** (2026-02-12): Added license file and open-source dependency readme.
- **v1.1.3** (2024-11-28): Fixed dropdown not displaying properly inside accordion — resolved by switching `useFloating` to `strategy: "fixed"`.
- **v1.1.2** (2024-09-20): Improved keyboard navigation accessibility.

## Open Questions

> Could not be determined from source code alone — requires human review

- [ ] The trigger element's `aria-activedescendant` is set to an empty string while the dropdown is open rather than pointing to the active option ID. Multiple axe-core ARIA rules (`aria-roles`, `button-name`, `aria-allowed-attr`) are disabled in e2e tests. Confirm whether this is an accepted gap or a pending accessibility fix.
- [ ] The widget is marked `offlineCapable="true"` in the XML, but language switching calls `window.mx.reloadWithState()` which likely requires connectivity. Clarify the intended offline behavior — does the widget render only, with switching silently disabled offline?
- [ ] The `preview: boolean` prop on `LanguageSwitcherPreview` is accepted but not used in the current implementation. Confirm whether this is a planned extension point or dead code to be removed.
