---
name: Angular SCSS Migration
description: Use this skill when the user asks to "migrate SCSS", "convert @import to @use", "internalize CSS framework", "Bulma migration", "trunks-ui migration", "SCSS modernization", "@forward chain", or when dealing with Sass deprecation warnings. Covers the two-phase strategy (internalize → convert), !default flag requirements, and the @forward chain architecture. Encodes RULE 3 (SCSS internalization gap prevention).
version: 1.0.0
---

# Angular SCSS Migration

## Purpose

Guide the modernization of SCSS from deprecated `@import` syntax to `@use`/`@forward`, and the internalization of CSS frameworks (Bulma, trunks-ui) into the project. This prevents the subtle visual regressions caused by missing rules during internalization.

## CRITICAL: Internalization Gap Prevention (RULE 3)

When internalizing a CSS framework (copying its source into your project instead of importing from node_modules), CSS rules can be silently lost. The compiler won't warn you — only visual inspection catches it.

### Known Gaps Found in Real Upgrade

| Issue | Original | Was Missing |
|-------|----------|-------------|
| Icon gaps | `.icon.icon-em:not(:last-child) { margin-right: 0.5em }` | Margin rules |
| Push classes | `.push-right { margin-right: $block-spacing }` | Naming changed to `.push-r-1` |
| Block spacing | `.block-2` through `.block-5` | Classes not generated |
| Level item push | `.level-item.push-2` through `.push-5` | Not implemented |
| Section padding | Mobile-specific section padding | Not included |
| Arrow decorators | `::before`/`::after` rotation | Rotation values wrong |

## Two-Phase Strategy

### Phase 1: Internalize CSS Frameworks

Copy framework source into your project:

```
src/scss/
├── bulma/           # Internalized from node_modules/bulma
├── trunks-ui/       # Internalized from node_modules/trunks-ui
├── variables.scss   # Project variable overrides
└── styles.scss      # Main entry point
```

**After copying, run the audit:** See `references/internalization-audit-checklist.md`

### Phase 2: Convert @import to @use/@forward

Process in order:
1. **Variables files** — Convert to `@use` with namespaces
2. **Mixins files** — Convert to `@use` + `@forward`
3. **Component SCSS** — Convert `@import` to `@use`

## The !default Flag Rule

When overriding framework variables, the framework must use `!default`:

```scss
// Framework file (bulma/utilities/initial-variables.scss)
$primary: #00d1b2 !default;  // !default means "use this unless already defined"

// Your override file (variables.scss)
@use 'bulma/utilities/initial-variables' with (
  $primary: #3273dc  // Your custom value
);
```

**Without `!default`:** Your overrides are silently ignored. The framework values always win.

### Audit for Missing !default

```bash
# Find variables without !default in internalized framework files
grep -rn "^\$" src/scss/bulma/ src/scss/trunks-ui/ | grep -v "!default" | head -20
```

## @forward Chain Architecture

```
App styles.scss
  └── @forward 'variables'
  └── @forward 'trunks-ui/all'
        └── @forward 'base/all'
              └── @forward 'helpers'
              └── @forward 'typography'
              └── @forward 'layout'
        └── @forward 'components/all'
  └── @forward 'bulma/all'
        └── @forward 'utilities/all'
        └── @forward 'elements/all'
        └── @forward 'components/all'
```

### Key Rules

1. **Variables flow down** — Define at the top, `@forward` carries them through
2. **Each file @use what it needs** — No implicit global scope
3. **Namespace imports** — `@use 'variables' as vars` then `vars.$primary`
4. **No circular imports** — Unlike `@import`, `@use` enforces import order

## Component SCSS Migration

```scss
// Before
@import 'src/scss/variables';
@import '~bulma/utilities/mixins';

.my-component {
  color: $primary;
  @include mobile { padding: 1rem; }
}

// After
@use 'src/scss/variables' as vars;
@use 'src/scss/bulma/utilities/mixins' as mx;

.my-component {
  color: vars.$primary;
  @include mx.mobile { padding: 1rem; }
}
```

### Batch Processing

```bash
# Find all component SCSS files still using @import
grep -rn "@import" --include="*.scss" src/app/ projects/ | grep -v node_modules
```

## Sass Deprecation Warnings

After conversion, you may still see deprecation warnings for:
- `/` division operator — Use `math.div()` instead
- `!global` flag usage
- Nested `@import` within rules

### Silencing Vendor Warnings

For internalized frameworks that you don't want to fully refactor:

```json
// angular.json
"stylePreprocessorOptions": {
  "includePaths": ["src/scss"],
  "silenceDeprecations": ["import", "global-builtin"]
}
```

## References

- `references/internalization-audit-checklist.md` — Complete audit for internalized frameworks
- `references/forward-chain-architecture.md` — How to structure the @forward chain
