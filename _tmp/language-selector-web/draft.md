# Draft: language-selector-web

EX-034 — Source exploration by worker

---

## src/LanguageSelector.tsx

**Purpose:** This is the main container component and default export of the widget. It bridges the Mendix runtime (via `window.mx`) with the UI component (`LanguageSwitcher`).

**Logic:** Manages three pieces of state: `selectedLanguage`, `languageList`, and `hideWidget`. On mount it reads `props.languageOptions.items` and `props.languageCaption` to build the language list. A second effect reads the current user's active language from `window.mx.session.getUserObject()` and maps it by `_guid` to the list. The `selectLanguage` callback mutates the user's `System.User_Language` reference on the Mendix object, commits via `window.mx.data.commit`, then calls `window.mx.reloadWithState()` to hot-reload the app in the new language.

**Behavior documented:** When `hideForSingle` is true and the language list has fewer than 2 entries, `hideWidget` is set to true and the component returns `null` — the widget is invisible. Language switching triggers a full page reload (with state preserved). The widget renders null until the language list is populated.

**User-facing:** Yes — this is the runtime entry point users interact with.

**New learnings:** Language selection is committed directly on the Mendix user object using the internal `System.User_Language` reference, making this widget tightly coupled to the Mendix platform data model.

---

## src/LanguageSelector.xml

**Purpose:** Defines the widget's public API — the properties visible in Studio Pro. This is the authoritative source for prop names, types, defaults, and captions.

**Logic:** Declares two property groups: "Languages" (with `languageOptions` datasource and `languageCaption` expression) and "General" (with `position`, `trigger`, `hideForSingle`). Also declares an "Accessibility" group with `screenReaderLabelCaption`. The `languageOptions` datasource is `isList="true"` and `required="true"`. The widget is `needsEntityContext="true"` and `offlineCapable="true"`.

**Behavior documented:** `position` controls where the dropdown appears relative to the trigger (left/right/top/bottom, default: bottom). `trigger` controls whether the dropdown opens on click (default) or hover. `hideForSingle` defaults to true. `screenReaderLabelCaption` is optional and for accessibility.

**User-facing:** Indirectly — this controls what app developers configure in Studio Pro.

**New learnings:** The widget is marked `offlineCapable="true"`, which is notable since language switching calls `window.mx.reloadWithState()` — this offline flag may refer only to rendering capability, not the switch action itself.

---

## typings/LanguageSelectorProps.d.ts

**Purpose:** Auto-generated TypeScript interface derived from the XML definition. Provides compile-time type safety for both the container (runtime) and preview (design-time) components.

**Logic:** Defines `PositionEnum` as `"left" | "right" | "top" | "bottom"`, `TriggerEnum` as `"click" | "hover"`. `LanguageSelectorContainerProps` is the runtime interface; `LanguageSelectorPreviewProps` is used for Studio Pro design canvas rendering. The `languageCaption` field is `ListExpressionValue<string>` — a Mendix-typed per-item expression evaluated lazily.

**Behavior documented:** `tabIndex` is optional in the container props (defaults to 0 in the component). `screenReaderLabelCaption` is optional (`DynamicValue<string>`). The preview props include `renderMode` with values `"design" | "xray" | "structure"`, indicating the widget supports all three Studio Pro preview modes.

**User-facing:** No — internal typing only.

**New learnings:** The deprecated `className` field in `LanguageSelectorPreviewProps` signals that this widget was migrated to use the unified `class` prop after Mendix 9.18.0.

---

## src/LanguageSelector.editorConfig.ts

**Purpose:** Provides Studio Pro editor configuration — specifically the structure-mode preview rendering and property filtering logic.

**Logic:** Imports `structurePreviewPalette` and `StructurePreviewProps` from the local `@mendix/widget-plugin-platform` package. `getProperties` returns `defaultValues` unchanged, meaning no property hiding or conditional suppression is implemented. `getPreview` renders a `RowLayout` with a "Selected language" text label, a small spacer, an SVG arrow icon (dark/light toggled by `isDarkMode`), and a right-side filler container.

**Behavior documented:** The structure preview always shows "Selected language" as a static placeholder regardless of configured data source. Arrow icon adapts to dark/light mode using separate SVG assets. No properties are conditionally hidden in the Studio Pro property panel.

**User-facing:** No — design-time preview only, not visible at runtime.

**New learnings:** The structure preview is intentionally minimal — it doesn't reflect position or trigger settings, only shows the trigger area visual.

---

## src/LanguageSelector.editorPreview.tsx

**Purpose:** Exports the `preview` function and `getPreviewCss` for the Studio Pro design canvas preview (non-structure mode).

**Logic:** Renders `LanguageSwitcherPreview` with hardcoded mock data: a single language item `{ _guid: "1", value: "Selected language" }`. Passes `screenReaderLabelCaption="LanguageSwitcherOptions"` as a fixed string. `getPreviewCss` returns the SCSS module for the widget's styles.

**Behavior documented:** In design canvas preview, the dropdown always shows "Selected language" as the current language regardless of the configured data source. The `preview` flag is passed as `true` to `LanguageSwitcherPreview`.

**User-facing:** No — design-time preview for Studio Pro only.

**New learnings:** The preview component accepts a `preview: boolean` prop (distinct from `LanguageSwitcher`), suggesting some conditional rendering exists for design-time vs. runtime, though the preview component does not currently use this flag visually.

---

## src/components/LanguageSwitcher.tsx

**Purpose:** The main runtime UI component responsible for rendering the trigger button and the floating dropdown menu.

**Logic:** Accepts `className`, `currentLanguage`, `languageList`, `onSelect`, `position`, `screenReaderLabelCaption`, `tabIndex`, and `trigger`. Delegates all floating/interaction logic to `useFloatingUI`. The trigger element has `aria-label` defaulting to "Language selector". The dropdown renders inside a `FloatingFocusManager` (from `@floating-ui/react`) with `modal={false}`. Each language item is a `div` with `role="option"`, supports `Enter` and `Space` key selection, and mouse click selection. The active (currently selected) item receives class `active`; the keyboard-highlighted item receives `highlighted`.

**Behavior documented:** When `isOpen` is false, the dropdown is not rendered at all (conditional rendering). The arrow indicator flips between `arrow-up` and `arrow-down` CSS classes to reflect open/closed state. Space key is blocked from selecting if `isTypingRef.current` is true (to allow typeahead without accidentally selecting).

**User-facing:** Yes — this is the visible dropdown widget.

**New learnings:** `FloatingFocusManager` traps focus within the dropdown while open but is non-modal, so screen reader virtual cursor can move outside. The `aria-activedescendant` is set to an empty string when open — a partial ARIA implementation.

---

## src/components/LanguageSwitcherPreview.tsx

**Purpose:** A simplified, static version of `LanguageSwitcher` used in Studio Pro's design canvas preview. No interaction, no floating behavior.

**Logic:** Renders the same outer structure as `LanguageSwitcher` but with no floating UI, no event handlers, and no conditional dropdown rendering. The menu div is always present but empty. Position class `popupmenu-position-{position}` is applied to the menu div. Arrow is always `arrow-down` (never flips). Accepts a `preview: boolean` prop but does not use it in the current implementation.

**Behavior documented:** Menu position is reflected in the preview via CSS class `popupmenu-position-{position}`, allowing Studio Pro to show positional styling. No interactivity exists — clicking does nothing.

**User-facing:** No — design-time preview for Studio Pro canvas only.

**New learnings:** The unused `preview` prop may be a placeholder for future conditional rendering in preview mode.

---

## src/hooks/useFloatingUI.ts

**Purpose:** Encapsulates all floating dropdown positioning and interaction logic using `@floating-ui/react`.

**Logic:** Uses `useFloating` with `strategy: "fixed"`, `middleware: [offset(2), flip({fallbackPlacements: ["top","right","bottom","left"]})]`. `autoUpdate` keeps position synchronized while mounted. Hover interaction uses `safePolygon()` to prevent menu close when moving cursor from trigger to menu. Click uses `mousedown` event. Keyboard navigation via `useListNavigation`; typeahead via `useTypeahead`. `handleSelect` calls `onSelect`, updates `selectedIndex`, and closes the menu.

**Behavior documented:** The flip middleware allows automatic position fallback — if the configured position doesn't fit the viewport, the menu flips to one of the four cardinal positions. Hover trigger includes a "safe polygon" hit zone to avoid menu flicker. Typeahead navigation is supported (typing letters moves focus to matching option). The selected index is initialized from the current language value.

**User-facing:** Indirectly — this hook drives all visible dropdown interaction behavior.

**New learnings:** The hook uses `strategy: "fixed"` which ensures correct positioning inside scrollable containers (e.g., the v1.1.3 fix for accordion nesting).

---

## src/ui/LanguageSelector.scss

**Purpose:** Styles the language selector widget. Covers trigger layout, arrow animation, focus rings, dropdown menu, and navbar context overrides.

**Logic:** Defines CSS variables with fallbacks: `--bg-color-secondary` (defaults to white), `--brand-primary` (defaults to `#264ae5`). The `.arrow-image` rotates 180deg with transition when `arrow-up` class is applied (open state). Inside `.navbar-brand`, text and arrow colors are overridden to use `--bg-color-secondary` and the arrow is inverted (for dark navbar backgrounds).

**Behavior documented:** Focus ring is `2px solid` outline with `4px` offset on trigger; focus on menu items uses `border-radius: 8px` with no offset. Active language item is `font-weight: 600`. Highlighted (keyboard-focused) items have `background: #f5f6f6`. Arrow transition is `0.2s ease-in-out` with `50ms` delay.

**User-facing:** Yes — determines all visual appearance.

**New learnings:** The navbar override using `.navbar-brand .widget-language-selector` suggests the widget is commonly embedded in Mendix navigation bars, requiring special color treatment.

---

## CHANGELOG.md

**Purpose:** Documents version history of the widget since initial release in 2022.

**Logic:** Version history spans v1.0.0 (2022-09-26) through v1.1.4 (2026-02-12), with an Unreleased section.

**Behavior documented:**
- v1.1.3 (2024-11-28): Fixed dropdown not showing properly inside accordion — resolved by using `strategy: "fixed"` in `useFloating`.
- v1.1.2 (2024-09-20): Improved keyboard navigation (a11y).
- v1.1.1 (2023-09-27): Removed redundant code to improve load time.
- v1.1.0 (2023-06-05): Updated icons/tiles; changed structure mode preview colors for dark/light modes.
- v1.0.1 (2022-10-28): Changed arrow rendering approach.
- v1.0.0 (2022-09-26): Initial release.
- v1.1.4 (2026-02-12): Added license file and open-source dependency readme.

**User-facing:** No — changelog is for developers/operators.

**New learnings:** The accordion fix (v1.1.3) and the a11y keyboard improvement (v1.1.2) are both confirmed implemented in the current source code.

---

## e2e/LanguageSelector.spec.js

**Purpose:** End-to-end Playwright tests validating rendering, language switching, and accessibility compliance.

**Logic:** Four tests: (1) visual snapshot of initial render, (2) switch to Arabic and snapshot, (3) switch to Chinese and verify translated text ("欢迎") appears, (4) run axe-core WCAG 2.1 AA scan excluding several known violations. Session logout is forced after each test due to Mendix license limits (max 5 sessions). Several axe rules are disabled: `aria-required-children`, `label`, `aria-roles`, `button-name`, `duplicate-id-active`, `duplicate-id`, `aria-allowed-attr`.

**Behavior documented:** The widget renders with class `.mx-name-languageSelector1`. Language switch is triggered by clicking `.current-language-text`. After selecting a language, the page content updates to reflect the new locale. The widget has known ARIA attribute violations that are explicitly excluded from the a11y scan.

**User-facing:** No — test infrastructure only.

**New learnings:** The disabled axe rules (`aria-roles`, `button-name`, `aria-allowed-attr`) suggest the trigger element's ARIA attributes are not fully compliant — consistent with the empty `aria-activedescendant` found in `LanguageSwitcher.tsx`.

---

## src/components/__tests__/LanguageSwitcher.spec.tsx

**Purpose:** Unit tests for the `LanguageSwitcher` component using React Testing Library and Jest.

**Logic:** Two snapshot tests: (1) empty language list — opens the dropdown via click and snapshots; (2) single language with selection — snapshots after click. Uses `userEvent.setup` with fake timers for timing control. The trigger element is queried by `role="combobox"`, though the component renders it as a plain `div` — the combobox role comes from `useRole(context, { role: "listbox" })` in `useFloatingUI`.

**Behavior documented:** The trigger element is assigned ARIA `role="combobox"` by the `useRole` hook from `@floating-ui/react`. Snapshot tests confirm structural stability of the dropdown render.

**User-facing:** No — test infrastructure only.

**New learnings:** The `role="combobox"` on the trigger and `role="listbox"` on the floating menu is the ARIA pattern used — despite the e2e tests disabling `aria-roles`, the roles are present and used for keyboard/screen reader interaction.
