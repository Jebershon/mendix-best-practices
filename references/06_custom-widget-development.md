# Skill 06 — Custom Widget Development (React / Pluggable Widgets)

**Purpose:** Define standards for building production-grade Mendix pluggable widgets using React, with strict XML configuration rules, prop validation, clean styling, and correct build/packaging.

**Reference Docs:**
- Widget HOW-TO: https://docs.mendix.com/howto/extensibility/create-a-pluggable-widget-one/
- XML Property Types: https://docs.mendix.com/apidocs-mxsdk/apidocs/pluggable-widgets-property-types/

---

## Folder Structure — Two-Folder Separation (Mandatory)

Custom widgets follow a strict **source vs. compiled** separation. Source code lives outside the Mendix project; only the compiled `.mpk` is placed inside it.

```
project-root/
│
├── custom-widgets/                           # ← SOURCE FOLDER — edit here always
│   ├── README.md                             # Widget inventory index (see template below)
│   │
│   ├── {WidgetName}/                         # One subfolder per widget
│   │   ├── src/
│   │   │   ├── {WidgetName}.tsx              # Main React component
│   │   │   ├── {WidgetName}.editorPreview.tsx  # Studio Pro preview (optional)
│   │   │   ├── ui/
│   │   │   │   └── {WidgetName}.css          # Scoped BEM styles
│   │   │   └── utils/
│   │   │       └── helpers.ts                # Utility/helper functions
│   │   ├── {WidgetName}.xml                  # Widget XML descriptor (critical)
│   │   ├── package.json
│   │   ├── tsconfig.json
│   │   ├── rollup.config.js
│   │   └── CHANGELOG.md                      # Per-widget change history
│   │
│   ├── AnotherWidget/
│   │   └── ... (same structure)
│   └── ...
│
└── {MendixProjectFolder}/
    └── widgets/                              # ← COMPILED OUTPUT ONLY — never edit here
        ├── {WidgetName}.mpk
        └── AnotherWidget.mpk
```

**Core rule:** `custom-widgets/` is the single source of truth. The Mendix `widgets/` folder holds only the compiled `.mpk` artifacts — nothing is authored there directly.

---

## Widget Inventory Index (`custom-widgets/README.md`)

Maintain a running index of every custom widget in the project. Update it whenever a widget is created or its version changes.

```markdown
# Custom Widgets — Project Index

| Widget Name       | Package ID                              | Version | Description                              | Last Updated |
|-------------------|-----------------------------------------|---------|------------------------------------------|--------------|
| EmployeeBadge     | com.bahri.hr.EmployeeBadge              | 1.2.0   | Displays employee name, dept with action | 2025-08-10   |
| ShaderAnimation   | com.bahri.portal.ShaderAnimation        | 1.0.1   | WebGL shader background animation        | 2025-07-15   |
| LeaveStatusTracker| com.bahri.hr.LeaveStatusTracker         | 1.0.0   | Visual leave balance indicator           | 2025-08-01   |

## Notes
- Source for all widgets: `custom-widgets/`
- Compiled .mpk files: `{MendixProject}/widgets/`
- To rebuild any widget: cd into its folder and run `npm run release`
```

---

## Per-Widget CHANGELOG (`custom-widgets/{WidgetName}/CHANGELOG.md`)

Every widget folder must have a `CHANGELOG.md`. Record every change — even small fixes — with date and reason. This is critical for diagnosing regressions later.

```markdown
# EmployeeBadge — Changelog

## [1.2.0] — 2025-08-10
### Added
- `badgeStyle` enumeration property (standard / compact / highlight)

### Fixed
- onClick not firing when `isVisible` was toggled dynamically

## [1.1.0] — 2025-06-22
### Added
- `department` attribute binding for secondary display

## [1.0.0] — 2025-05-01
### Initial Release
- Displays employee name with configurable click action
```

---

## Widget XML Configuration Rules (STRICT — Zero Errors)

The XML descriptor defines the widget's interface in Studio Pro. Errors here break the widget.

### XML Template (Complete, Valid)

```xml
<?xml version="1.0" encoding="utf-8" ?>
<widget id="com.{company}.{module}.{WidgetName}"
        pluginWidget="true"
        needsEntityContext="true"
        xmlns="http://www.mendix.com/widget/1.0/"
        xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:schemaLocation="http://www.mendix.com/widget/1.0/
          https://apidocs.mendix.com/pluggable-widgets/1.0/widget.xsd">

    <name>{Widget Display Name}</name>
    <description>{Short description of the widget}</description>
    <icon />

    <properties>
        <!-- === REQUIRED PROPERTIES === -->

        <!-- String attribute binding -->
        <property key="labelText" type="string" required="true">
            <caption>Label Text</caption>
            <description>Text to display as label.</description>
        </property>

        <!-- Attribute binding (reads from entity) -->
        <property key="valueAttribute" type="attribute" required="true">
            <caption>Value Attribute</caption>
            <description>The entity attribute to display.</description>
            <attributeTypes>
                <attributeType name="String" />
                <attributeType name="Integer" />
            </attributeTypes>
        </property>

        <!-- Action (on-click, on-change) -->
        <property key="onClickAction" type="action" required="false">
            <caption>On Click Action</caption>
            <description>Action to trigger on click.</description>
        </property>

        <!-- Boolean toggle -->
        <property key="isVisible" type="boolean" defaultValue="true">
            <caption>Visible</caption>
            <description>Toggle widget visibility.</description>
        </property>

        <!-- Enumeration (dropdown in Studio Pro) -->
        <property key="displayMode" type="enumeration" defaultValue="default">
            <caption>Display Mode</caption>
            <description>Choose how the widget renders.</description>
            <enumerationValues>
                <enumerationValue key="default">Default</enumerationValue>
                <enumerationValue key="compact">Compact</enumerationValue>
                <enumerationValue key="expanded">Expanded</enumerationValue>
            </enumerationValues>
        </property>

        <!-- Expression (dynamic value from Mendix expression) -->
        <property key="dynamicClass" type="expression" required="false" defaultValue="''">
            <caption>Dynamic Class</caption>
            <description>CSS class expression.</description>
            <returnType type="String" />
        </property>
    </properties>
</widget>
```

### Common XML Mistakes to Avoid

| ❌ Mistake | ✅ Fix |
|-----------|-------|
| Missing `pluginWidget="true"` | Always include — required for pluggable widgets |
| `id` not matching package.json `name` | Must match exactly: `com.company.module.WidgetName` |
| Using `type="object"` without `isList` | Specify `isList="false"` or `isList="true"` |
| Empty `<attributeTypes>` on attribute property | List at least one `<attributeType>` |
| `required="true"` on action properties | Actions should be `required="false"` |
| Wrong XSD schema URL | Use: `https://apidocs.mendix.com/pluggable-widgets/1.0/widget.xsd` |
| Missing `defaultValue` on `enumeration` | Always provide a `defaultValue` |
| Boolean without `defaultValue` | Always set `defaultValue="true"` or `"false"` |

---

## Main React Component Template

```tsx
// src/{WidgetName}.tsx
import { ReactElement, createElement } from "react";
import { {WidgetName}ContainerProps } from "../typings/{WidgetName}Props";
import "./{WidgetName}.css";

export function {WidgetName}(props: {WidgetName}ContainerProps): ReactElement {
    const {
        labelText,
        valueAttribute,
        onClickAction,
        isVisible,
        displayMode
    } = props;

    // Guard: hide if not visible
    if (!isVisible) {
        return <></>;
    }

    // Guard: handle missing required attribute
    if (!valueAttribute || valueAttribute.status === "loading") {
        return <div className="{widget-name}__loading">Loading...</div>;
    }

    if (valueAttribute.status === "unavailable") {
        return <div className="{widget-name}__unavailable">Data unavailable.</div>;
    }

    const handleClick = (): void => {
        if (onClickAction && onClickAction.canExecute) {
            onClickAction.execute();
        }
    };

    return (
        <div
            className={`{widget-name} {widget-name}--${displayMode}`}
            onClick={handleClick}
            role="button"
            tabIndex={0}
            aria-label={labelText}
        >
            <span className="{widget-name}__label">{labelText}</span>
            <span className="{widget-name}__value">{valueAttribute.value}</span>
        </div>
    );
}
```

---

## Props Validation Rules

- **Always check `attribute.status`** — it can be `"loading"`, `"unavailable"`, or `"available"`
- **Always check `action.canExecute`** before calling `action.execute()`
- **Never access `attribute.value` without checking status first**
- **Use TypeScript types** generated by Mendix tooling — do not bypass with `any`

```tsx
// ✅ Correct
if (props.valueAttribute.status === "available") {
    const value = props.valueAttribute.value;
}

// ❌ Wrong — crashes if status is "loading" or "unavailable"
const value = props.valueAttribute.value;
```

---

## Styling Rules

- **One CSS file per widget**, scoped with a unique root class name
- **Use BEM naming**: `.widget-name__element--modifier`
- **Never use inline styles** — all styles go in the CSS file
- **Never override Mendix theme variables directly** — use widget-scoped classes
- **Support theme integration** via `class` and `style` props

```css
/* ui/{WidgetName}.css */

/* Root container */
.my-widget {
    display: flex;
    align-items: center;
    padding: 8px 12px;
    border-radius: 4px;
    cursor: pointer;
}

/* Elements */
.my-widget__label {
    font-weight: 600;
    margin-right: 8px;
}

.my-widget__value {
    color: var(--brand-primary, #264ae5);
}

/* Modifiers */
.my-widget--compact {
    padding: 4px 8px;
    font-size: 12px;
}

.my-widget--expanded {
    padding: 16px;
    font-size: 16px;
}

/* States */
.my-widget__loading {
    opacity: 0.5;
    pointer-events: none;
}
```

---

## package.json Template

```json
{
  "name": "com.company.module.widgetname",
  "widgetName": "WidgetName",
  "version": "1.0.0",
  "description": "Your widget description",
  "copyright": "Your Company",
  "license": "Apache-2.0",
  "packagePath": "com/company/module",
  "repository": "",
  "devDependencies": {
    "@mendix/pluggable-widgets-tools": "^9.24.0"
  },
  "scripts": {
    "start": "pluggable-widgets-tools start:web",
    "build": "pluggable-widgets-tools build:web",
    "release": "pluggable-widgets-tools release:web",
    "lint": "pluggable-widgets-tools lint"
  }
}
```

---

## Build & Deploy Workflow

Follow these steps every time a widget is created or updated. Never skip step 6.

```
Step 1 — Edit source
  → Work inside: custom-widgets/{WidgetName}/src/
  → Never touch files in {MendixProject}/widgets/

Step 2 — Build (dev check)
  → cd custom-widgets/{WidgetName}
  → npm run build
  → Fix any TypeScript or lint errors before proceeding

Step 3 — Release (generate .mpk)
  → npm run release
  → Output: custom-widgets/{WidgetName}/dist/{WidgetName}.mpk

Step 4 — Deploy to Mendix project
  → Copy .mpk to: {MendixProjectFolder}/widgets/{WidgetName}.mpk
  → Overwrite the existing file

Step 5 — Synchronize in Studio Pro
  → Press F4 (or: App menu → Synchronize App Directory)
  → Verify widget appears/updates correctly in the widget toolbox

Step 6 — Update CHANGELOG
  → Open: custom-widgets/{WidgetName}/CHANGELOG.md
  → Add an entry with version, date, and what changed
  → If version changed, also update: custom-widgets/README.md
```

> ✅ **Rule:** Never edit widget behaviour by modifying or inspecting the `.mpk` directly. Always return to the source in `custom-widgets/` and rebuild.

> ❌ **Never** commit a changed `.mpk` without also committing the corresponding source changes in `custom-widgets/`.

---

## Do's and Don'ts

| ✅ Do | ❌ Don't |
|-------|---------|
| Keep all widget source in `custom-widgets/` | Author or edit anything inside `{MendixProject}/widgets/` |
| Maintain a `CHANGELOG.md` per widget | Make silent widget changes with no recorded history |
| Keep `custom-widgets/README.md` index up to date | Let the widget inventory go stale |
| Always rebuild from source and copy the fresh `.mpk` | Modify or reuse a stale `.mpk` after source changes |
| Commit source changes and `.mpk` together | Push an updated `.mpk` without the matching source |
| Validate `attribute.status` before accessing `.value` | Access attribute values without a status check |
| Check `action.canExecute` before `action.execute()` | Call actions unconditionally |
| Use BEM CSS classes scoped to the widget root | Use global or generic CSS class names |
| Keep each widget focused on a single UI concern | Build multi-purpose mega-widgets |
| Generate typings with Mendix pluggable-widgets-tools | Manually write or guess props interfaces |
| Test in both Studio Pro preview and runtime | Only test in the running app |
| Use semantic HTML (role, aria-label) | Build inaccessible widgets without ARIA attributes |

---

## Example Usage

> **Scenario:** Render a styled employee badge showing name and department

```
Widget: EmployeeBadge
XML Properties:
  - employeeName  (attribute: String)
  - department    (attribute: String)
  - onClickAction (action)
  - badgeStyle    (enumeration: "standard" | "compact" | "highlight")

React Component:
  → Check employeeName.status === "available"
  → Render: <div class="employee-badge employee-badge--{badgeStyle}">
                <span class="employee-badge__name">{employeeName.value}</span>
                <span class="employee-badge__dept">{department.value}</span>
            </div>
  → onClick: if onClickAction.canExecute → onClickAction.execute()
```
