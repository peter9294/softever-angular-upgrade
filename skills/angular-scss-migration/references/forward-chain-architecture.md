# @forward Chain Architecture

## Concept

With `@use`/`@forward`, SCSS modules form a directed acyclic graph (DAG). Each file explicitly declares its dependencies. Variables, mixins, and functions only propagate through `@forward` declarations.

## Architecture for Angular Projects

```
src/scss/
├── styles.scss              ← Main entry (angular.json)
├── variables.scss           ← Project overrides
├── bulma/
│   ├── _all.scss            ← @forward all Bulma modules
│   ├── utilities/
│   │   ├── _all.scss
│   │   ├── _initial-variables.scss
│   │   ├── _derived-variables.scss
│   │   ├── _mixins.scss
│   │   └── _functions.scss
│   ├── elements/
│   │   ├── _all.scss
│   │   └── ...
│   └── components/
│       ├── _all.scss
│       └── ...
└── trunks-ui/
    ├── _all.scss
    ├── base/
    │   ├── _all.scss
    │   └── ...
    └── components/
        ├── _all.scss
        └── ...
```

## Flow Direction

```
styles.scss
  @use 'variables' with ($primary: #3273dc)     ← Override variables
  @forward 'bulma/all'                            ← Framework modules
  @forward 'trunks-ui/all'                        ← Extension modules
  @forward 'app-specific'                         ← App-specific styles

bulma/all
  @forward 'utilities/all'                        ← Variables first
  @forward 'elements/all'                         ← Then elements
  @forward 'components/all'                       ← Then components

utilities/all
  @forward 'initial-variables'                    ← Base variables
  @forward 'derived-variables'                    ← Derived from base
  @forward 'mixins'                               ← Reusable mixins
  @forward 'functions'                            ← Reusable functions
```

## Key Rules

### 1. Variables Always Flow Down

Variables defined at the top must be `@forward`ed through every intermediate module:

```scss
// utilities/initial-variables.scss
$primary: #00d1b2 !default;

// utilities/all.scss
@forward 'initial-variables';   ← $primary is now available to consumers

// elements/button.scss
@use '../utilities/all' as utils;
.button { background: utils.$primary; }
```

### 2. Override with @use...with

```scss
// styles.scss
@use 'bulma/utilities/initial-variables' with (
  $primary: #3273dc,
  $link: #3273dc,
  $family-sans-serif: 'Prompt', sans-serif
);
```

### 3. No Circular Dependencies

Unlike `@import` (which was purely textual), `@use` enforces a DAG:

```scss
// INVALID — circular dependency
// a.scss: @use 'b';
// b.scss: @use 'a';

// VALID — use a shared dependency
// a.scss: @use 'shared';
// b.scss: @use 'shared';
```

### 4. Namespace Everything

```scss
@use 'variables' as vars;        // vars.$primary
@use 'mixins' as mx;             // @include mx.mobile { }
@use 'sass:math';                // math.div(100, 3)
@use 'sass:color';               // color.adjust($primary, $lightness: 10%)
```

### 5. @forward with show/hide

Control what gets exported:

```scss
// Only forward specific members
@forward 'variables' show $primary, $link, $family-sans-serif;

// Forward everything except certain members
@forward 'variables' hide $internal-var;
```

## Component SCSS Pattern

Each Angular component SCSS file should:

```scss
// component.scss
@use 'src/scss/variables' as vars;

// If need mixins:
@use 'src/scss/bulma/utilities/mixins' as mx;

.my-component {
  color: vars.$primary;

  @include mx.mobile {
    padding: vars.$gap;
  }
}
```

**Never `@forward` from component SCSS** — components are leaf nodes in the graph.

## Troubleshooting

### "Variable not found" Error

The variable isn't being `@forward`ed through the chain. Check:
1. Is the variable file `@forward`ed from the `_all.scss` file?
2. Are you using the correct namespace?
3. Is the `@use` path correct relative to the file?

### "Module loop" Error

You have a circular dependency. Break it by:
1. Extracting shared code into a separate file
2. Having both sides `@use` the shared file instead of each other

### "Already loaded" Warning

A module is being `@use`d multiple times. This is OK — `@use` caches modules.
The warning appears when the same module is loaded with different configurations.

**Fix:** Only configure with `@use...with` in ONE place (the main entry point).
