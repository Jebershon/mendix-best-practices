# Skill 07 — UI/UX & Styling Best Practices

**Purpose:** Enforce centralized, consistent, and maintainable styling across all Mendix pages and widgets using SCSS, BEM, and responsive design principles.

---

## Core Principles

1. **Centralized styles** — all custom CSS/SCSS lives in the `styles/` folder, never scattered in page properties
2. **No inline styles** — never use the "Dynamic Classes" or style attributes to write raw CSS
3. **BEM naming** — Block, Element, Modifier for all custom class names
4. **Theme variables first** — always use Mendix theme variables before hardcoding values
5. **Responsive by default** — all pages must work on desktop and tablet at minimum

---

## Folder Structure

```
styles/
├── web/
│   ├── _variables.scss        # Custom variable overrides
│   ├── _typography.scss       # Font definitions and text styles
│   ├── _buttons.scss          # Button variants
│   ├── _forms.scss            # Input, select, textarea styles
│   ├── _tables.scss           # Data grid and list styles
│   ├── _layout.scss           # Layout helpers, containers, grids
│   ├── _navigation.scss       # Header, sidebar, nav bar
│   ├── _cards.scss            # Card components
│   ├── _badges.scss           # Status badges and tags
│   ├── _modals.scss           # Popup/modal overrides
│   ├── _utilities.scss        # Helper classes (spacing, visibility)
│   └── main.scss              # Imports all partials in order
└── native/
    └── main.js                # Native mobile styles
```

---

## Custom Widget Source Folder Management

All custom widgets developed for this project must maintain their **full source code** in a dedicated, version-controlled folder separate from the compiled `.mpk` output. This ensures that any future change — a bug fix, a prop addition, a style update — can be made cleanly without reverse-engineering the built artifact.

### Folder Structure

```
project-root/
├── custom-widgets/                          # ← Source folder for ALL custom widgets
│   ├── README.md                            # Index of all widgets, purpose, version
│   ├── EmployeeBadge/                       # One subfolder per widget
│   │   ├── src/
│   │   │   ├── EmployeeBadge.tsx
│   │   │   ├── EmployeeBadge.editorPreview.tsx
│   │   │   └── ui/
│   │   │       └── EmployeeBadge.css
│   │   ├── EmployeeBadge.xml
│   │   ├── package.json
│   │   ├── tsconfig.json
│   │   ├── rollup.config.js
│   │   └── CHANGELOG.md                    # Track changes per widget
│   ├── ShaderAnimation/
│   │   └── ... (same structure)
│   └── LeaveStatusTracker/
│       └── ... (same structure)
│
└── {MendixProjectFolder}/
    └── widgets/                             # ← Only compiled .mpk files go here
        ├── EmployeeBadge.mpk
        ├── ShaderAnimation.mpk
        └── LeaveStatusTracker.mpk
```

**Rules:**
- `custom-widgets/` is the **source of truth** — this is what you edit
- `{MendixProject}/widgets/` contains only **compiled `.mpk` files** — never edit here
- Every widget folder must have a `CHANGELOG.md` — note every change with date and reason
- Never delete a widget from `custom-widgets/` even if it is temporarily removed from the app

### Per-Widget CHANGELOG Template

```markdown
# EmployeeBadge — Changelog

## [1.2.0] — 2025-08-10
- Added `badgeStyle` enumeration property (standard / compact / highlight)
- Fixed: onClick not firing when `isVisible` was toggled dynamically

## [1.1.0] — 2025-06-22
- Added `department` attribute binding

## [1.0.0] — 2025-05-01
- Initial release: name display with click action
```

### Widget Index (custom-widgets/README.md)

Maintain a running index of all widgets:

```markdown
# Custom Widgets Index

| Widget Name | Version | Description | Last Updated |
|-------------|---------|-------------|--------------|
| EmployeeBadge | 1.2.0 | Displays employee name, dept, and role with click action | 2025-08-10 |
| ShaderAnimation | 1.0.1 | WebGL shader background animation | 2025-07-15 |
| LeaveStatusTracker | 1.0.0 | Visual leave balance indicator | 2025-08-01 |
```

### Build & Deploy Workflow

```
1. Edit source in:    custom-widgets/{WidgetName}/src/
2. Build:             cd custom-widgets/{WidgetName} && npm run release
3. Output goes to:    custom-widgets/{WidgetName}/dist/{WidgetName}.mpk
4. Copy .mpk to:      {MendixProject}/widgets/
5. Sync in Studio:    F4 (Synchronize App Directory)
6. Update:            custom-widgets/{WidgetName}/CHANGELOG.md
```

> ✅ **Rule:** Never edit widget behavior by modifying the `.mpk` directly. Always go back to the source in `custom-widgets/`.

---



**BEM = Block__Element--Modifier**

```scss
// Block: The standalone component
.employee-card { }

// Element: A part of the block
.employee-card__header { }
.employee-card__name { }
.employee-card__avatar { }

// Modifier: A variant or state of the block or element
.employee-card--highlighted { }
.employee-card--compact { }
.employee-card__name--inactive { }
```

**Rules:**
- Use **lowercase hyphen-separated** words: `leave-request` not `LeaveRequest`
- Only two levels of nesting: Block and Element. Never `block__element__sub-element`
- Use modifiers for states: `--active`, `--disabled`, `--loading`, `--error`

---

## Using Mendix Theme Variables

Always prefer Mendix Atlas theme variables over hardcoded values:

```scss
// ✅ Use theme variables
.my-widget {
    color: var(--brand-primary);
    background: var(--bg-color-secondary);
    border-radius: var(--border-radius-default);
    font-size: var(--font-size-default);
    padding: var(--spacing-medium);
}

// ❌ Hardcode values only as last resort
.my-widget {
    color: #264ae5;       // Only if no variable exists
    font-size: 14px;      // Avoid
}
```

**Commonly used Atlas variables:**
| Variable | Usage |
|----------|-------|
| `--brand-primary` | Primary action color |
| `--brand-success` | Success/positive state |
| `--brand-warning` | Warning state |
| `--brand-danger` | Error/danger state |
| `--font-size-default` | Body text size |
| `--font-size-large` | Large text |
| `--spacing-small` | 4px equivalent |
| `--spacing-medium` | 8px equivalent |
| `--spacing-large` | 16px equivalent |
| `--border-radius-default` | Standard border radius |
| `--bg-color-secondary` | Secondary background |

---

## Typography Rules

- **Do not set font-family** at the widget level — inherit from the theme
- **Use relative units (rem/em)** for font sizes — never `px` for text
- **Line height:** minimum `1.4` for readability
- **Heading hierarchy:** Respect `H1 → H2 → H3` — never skip levels

```scss
// _typography.scss
.page-title {
    font-size: 1.75rem;
    font-weight: 700;
    color: var(--text-color-default);
    line-height: 1.3;
    margin-bottom: var(--spacing-large);
}

.section-label {
    font-size: 0.875rem;
    font-weight: 600;
    text-transform: uppercase;
    letter-spacing: 0.05em;
    color: var(--text-color-secondary);
}
```

---

## Responsive Design Rules

- **Mobile-first** — base styles target small screens, media queries enhance for larger
- **Breakpoints to use:**

```scss
// _variables.scss
$breakpoint-sm: 576px;
$breakpoint-md: 768px;
$breakpoint-lg: 1024px;
$breakpoint-xl: 1280px;
```

```scss
// Example: Stack columns on mobile, side-by-side on tablet+
.form-row {
    display: flex;
    flex-direction: column;
    gap: var(--spacing-medium);

    @media (min-width: 768px) {
        flex-direction: row;
        align-items: flex-start;
    }
}

.form-row__field {
    flex: 1;
    min-width: 0;
}
```

---

## Status Badge Pattern

Use consistent color coding across the app for status indicators:

```scss
// _badges.scss
.status-badge {
    display: inline-flex;
    align-items: center;
    padding: 2px 10px;
    border-radius: 12px;
    font-size: 0.75rem;
    font-weight: 600;
    text-transform: capitalize;
}

.status-badge--pending    { background: #FFF3CD; color: #856404; }
.status-badge--approved   { background: #D1FAE5; color: #065F46; }
.status-badge--rejected   { background: #FEE2E2; color: #991B1B; }
.status-badge--processing { background: #DBEAFE; color: #1E40AF; }
.status-badge--inactive   { background: #F3F4F6; color: #6B7280; }
```

---

## Utility Classes (Spacing / Visibility)

```scss
// _utilities.scss

// Spacing helpers
.mt-sm { margin-top: var(--spacing-small); }
.mt-md { margin-top: var(--spacing-medium); }
.mt-lg { margin-top: var(--spacing-large); }
.mb-sm { margin-bottom: var(--spacing-small); }
.mb-md { margin-bottom: var(--spacing-medium); }
.mb-lg { margin-bottom: var(--spacing-large); }

// Visibility
.hidden   { display: none !important; }
.invisible { visibility: hidden; }

// Text alignment
.text-left   { text-align: left; }
.text-center { text-align: center; }
.text-right  { text-align: right; }

// Flex helpers
.flex        { display: flex; }
.flex-center { display: flex; align-items: center; justify-content: center; }
.flex-between{ display: flex; align-items: center; justify-content: space-between; }
.flex-wrap   { flex-wrap: wrap; }
```

---

## Mendix Page Styling Rules

- **Use CSS classes on containers** — not on individual widgets inside snippets
- **Define a layout class** on the outermost container of every page template
- **Use Snippet-level classes** to scope styles to a reusable snippet
- **Never duplicate class definitions** across SCSS files — use `@extend` or a shared partial

---

## Do's and Don'ts

| ✅ Do | ❌ Don't |
|-------|---------|
| Keep all widget source code in `custom-widgets/` | Edit widgets only in the Mendix `widgets/` folder |
| Maintain a `CHANGELOG.md` per widget | Make silent widget changes with no record |
| Keep `custom-widgets/README.md` index updated | Let the widget inventory go stale |
| Build from source and copy `.mpk` to Mendix project | Modify `.mpk` files directly |
| Use BEM for all custom class names | Use random or undescriptive class names |
| Use `var(--theme-variable)` for colors | Hardcode hex colors everywhere |
| Centralize all styles in `styles/web/` | Write styles inside widget or page properties |
| Use rem/em for font sizes | Use px for font sizes |
| Write mobile-first with media queries | Build desktop-only layouts |
| Use modifier classes for states | Use JavaScript to set inline style |
| Comment complex SCSS sections | Leave styles undocumented |

---

## Example Usage

> **Scenario:** Employee card component with compact and highlighted variants

```scss
// _cards.scss

.employee-card {
    background: var(--bg-color-secondary);
    border: 1px solid var(--border-color-default);
    border-radius: var(--border-radius-default);
    padding: var(--spacing-large);
    display: flex;
    align-items: center;
    gap: var(--spacing-medium);
    transition: box-shadow 0.2s ease;
}

.employee-card:hover {
    box-shadow: 0 4px 12px rgba(0, 0, 0, 0.08);
}

.employee-card__avatar {
    width: 48px;
    height: 48px;
    border-radius: 50%;
    object-fit: cover;
}

.employee-card__name {
    font-size: 1rem;
    font-weight: 600;
    color: var(--text-color-default);
}

.employee-card__role {
    font-size: 0.85rem;
    color: var(--text-color-secondary);
}

.employee-card--compact {
    padding: var(--spacing-small);
}

.employee-card--highlighted {
    border-left: 4px solid var(--brand-primary);
    background: #F0F4FF;
}
```
