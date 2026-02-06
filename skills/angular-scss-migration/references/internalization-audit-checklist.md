# Internalization Audit Checklist

## Purpose

When copying a CSS framework's source into your project (internalization), rules can be silently lost. This checklist ensures nothing is missed.

## Audit Process

### Step 1: File Size Comparison

Compare the original framework files with your internalized versions:

```bash
# Count lines in original vs internalized
wc -l node_modules/trunks-ui/base/helpers.scss
wc -l src/scss/trunks-ui/helpers/*.scss

wc -l node_modules/bulma/utilities/*.scss
wc -l src/scss/bulma/utilities/*.scss
```

**Red flag:** Significant size difference (>20% fewer lines) means rules were dropped.

### Step 2: Category-by-Category Audit

#### A. Spacing & Margin Rules

- [ ] `:not(:last-child)` margin rules for elements, icons, buttons
- [ ] `:not(:first-child)` margin rules
- [ ] Responsive spacing variants (mobile/tablet/desktop)
- [ ] Block spacing multipliers (`.block-2`, `.block-3`, `.block-4`, `.block-5`)
- [ ] Push utilities (`.push-right`, `.push-left`, `.push-auto`)
- [ ] Gap utilities

#### B. Helper Classes

- [ ] Display utilities (`.is-hidden`, `.is-invisible`, `.is-block`, `.is-flex`)
- [ ] Text alignment (`.has-text-centered`, `.has-text-right`)
- [ ] Color utilities (`.has-text-primary`, `.has-background-primary`)
- [ ] Responsive display helpers (`.is-hidden-mobile`, `.is-hidden-tablet`)
- [ ] Flex utilities (`.is-justify-content-*`, `.is-align-items-*`)

#### C. Layout Rules

- [ ] Section padding (including mobile overrides)
- [ ] Container widths at each breakpoint
- [ ] Column gap calculations
- [ ] Level item spacing
- [ ] Tile nesting rules

#### D. Component-Specific Rules

- [ ] Button group spacing (`.buttons > .button:not(:last-child)`)
- [ ] Form field spacing (`.field:not(:last-child)`)
- [ ] Card shadow and border rules
- [ ] Modal overlay z-index
- [ ] Navbar dropdown positioning

#### E. Pseudo-Element Rules

- [ ] `::before` / `::after` content for decorators
- [ ] Arrow rotation values (dropdown arrows, select arrows)
- [ ] Loading spinner animations
- [ ] Checkbox/radio custom styling

#### F. Variable Overrides

- [ ] All variables have `!default` flag
- [ ] Custom overrides are imported BEFORE the framework
- [ ] Derived variables recalculate correctly

### Step 3: Template Class Usage Verification

Search for every class name used in templates and verify it exists in the internalized CSS:

```bash
# Extract class names from templates
grep -roh 'class="[^"]*"' --include="*.html" src/ projects/ | \
  tr ' "' '\n' | sort -u | grep -v "^class=$\|^$" > /tmp/template-classes.txt

# For each class, check it exists in internalized SCSS
while read cls; do
  if ! grep -rq "$cls" src/scss/; then
    echo "MISSING: $cls"
  fi
done < /tmp/template-classes.txt
```

### Step 4: Visual Comparison

For critical pages, take screenshots before and after internalization:

1. Login page
2. Main dashboard
3. Data table with pagination
4. Form with all field types
5. Modal dialog
6. Dropdown menus

Compare side-by-side for:
- Spacing differences
- Missing icons/decorators
- Color changes
- Font size/weight changes
- Border/shadow changes

## Common Gotchas

### 1. Naming Conflicts

Framework uses `.push-right`, your internalized version renamed to `.push-r-1`.

**Fix:** Use the original class names, or add backward-compatible aliases:
```scss
.push-right, .push-r-1 { margin-right: $block-spacing !important; }
```

### 2. Sass Compilation Order

With `@import`, everything was global. With `@use`, variables must be explicitly imported.

**Fix:** Ensure variables are `@forward`ed at the top of the chain.

### 3. Missing @each Loops

Framework may generate classes via `@each` loops (e.g., `.block-2` through `.block-5`):
```scss
@each $i in 2, 3, 4, 5 {
  .block-#{$i} { margin-bottom: $block-spacing * $i !important; }
}
```

**Fix:** Check that all `@each` and `@for` loops are carried over.

### 4. Conditional Rules

Framework may have `@if` conditions based on variables:
```scss
@if $variable-columns {
  .columns.is-variable { /* ... */ }
}
```

**Fix:** Ensure the controlling variables are set correctly.
