---
name: mendix-best-practices
description: Mendix low-code development best practices covering project structure, naming conventions, microflows, nanoflows, Java/JavaScript actions, custom pluggable widgets, UI/UX styling, validation and security rules, logging/monitoring, clean architecture, performance optimization, and AI code generation guidelines. Use when working in a Mendix project, reviewing Mendix modules, writing microflows/nanoflows, building custom widgets, or generating Mendix code.
---

# Mendix Best Practices

This skill provides curated best-practice references for Mendix low-code development. When the user is working on Mendix artifacts (microflows, nanoflows, Java/JS actions, widgets, styling, security, performance, etc.), consult the relevant reference file below before making recommendations or writing code.

## How to use

1. Identify which topic(s) the user's task touches (e.g., naming, microflow design, widget development).
2. Read the matching reference file(s) from `references/` to ground your guidance in these conventions.
3. Apply the rules when reviewing, writing, or suggesting Mendix code. Cite the specific rule when relevant.
4. If multiple topics overlap, read each applicable file — they are complementary, not exclusive.

## Reference index

| # | Topic | File |
|---|-------|------|
| 01 | Project structure & naming conventions | [references/01_project-structure-naming-conventions.md](references/01_project-structure-naming-conventions.md) |
| 02 | Microflow best practices | [references/02_microflow-best-practices.md](references/02_microflow-best-practices.md) |
| 03 | Nanoflow best practices | [references/03_nanoflow-best-practices.md](references/03_nanoflow-best-practices.md) |
| 04 | Java action guidelines | [references/04_java-action-guidelines.md](references/04_java-action-guidelines.md) |
| 05 | JavaScript action guidelines | [references/05_javascript-action-guidelines.md](references/05_javascript-action-guidelines.md) |
| 06 | Custom pluggable widget development | [references/06_custom-widget-development.md](references/06_custom-widget-development.md) |
| 07 | UI/UX & styling best practices | [references/07_ui-ux-styling-best-practices.md](references/07_ui-ux-styling-best-practices.md) |
| 08 | Validation & security rules | [references/08_validation-security-rules.md](references/08_validation-security-rules.md) |
| 09 | Logging & monitoring strategy | [references/09_logging-monitoring-strategy.md](references/09_logging-monitoring-strategy.md) |
| 10 | Code organization & clean architecture | [references/10_code-organization-clean-architecture.md](references/10_code-organization-clean-architecture.md) |
| 11 | Performance optimization rules | [references/11_performance-optimization-rules.md](references/11_performance-optimization-rules.md) |
| 12 | AI code generation rules | [references/12_ai-code-generation-rules.md](references/12_ai-code-generation-rules.md) |

## Guidance

- Prefer grep over reading full files when hunting a specific rule.
- When the user asks for something that may conflict with a rule (e.g., a shortcut that violates naming), flag the rule and let them decide.
- The AI code generation rules (#12) take precedence when producing new Mendix artifacts from scratch.
