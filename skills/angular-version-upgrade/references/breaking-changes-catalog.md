# Breaking Changes Catalog

## Angular 10
- **ViewEncapsulation.Native removed** — Use `ViewEncapsulation.ShadowDom`
- **Generic type required for ModuleWithProviders** — Add type parameter
- **Warnings for CommonJS imports** — Convert to ESM where possible

## Angular 11
- **Router `relativeLinkResolution` default changed** — May affect relative route resolution
- **async() test helper renamed** — Use `waitForAsync()` instead
- **IE 9/10/mobile support dropped** — Remove related polyfills

## Angular 12
- **TSLint support removed** — Must migrate to ESLint before or at v12
  ```bash
  ng add @angular-eslint/schematics
  ng g @angular-eslint/schematics:convert-tslint-to-eslint
  ```
- **Ivy is the only engine** — ViewEngine no longer available
- **Strict mode by default** — May surface new type errors
- **Sass: `~` imports deprecated** — Sass discourages tilde imports

## Angular 13
- **IE 11 support removed** — Remove IE-specific polyfills and CSS
- **TestBed auto-teardown enabled** — Tests automatically destroy components
- **APF v13** — Libraries must use Angular Package Format v13

## Angular 14
- **Typed Forms** — `FormControl<string>` instead of `FormControl`
- **Standalone components preview** — New component authoring model
- **Inject function** — Alternative to constructor injection

## Angular 15
- **Standalone APIs stable** — Full standalone support
- **Router with standalone** — `provideRouter()` available
- **Image directive** — `NgOptimizedImage` for automatic image optimization

## Angular 16
- **ngcc REMOVED** (Critical) — ViewEngine libraries will NOT work
  - All dependencies must be compiled with Ivy
  - Local .tgz packages must be rebuilt with `partial` compilation
  - This is the highest-risk version upgrade
- **Signals (developer preview)** — Optional reactive primitives
- **Hydration (developer preview)** — For SSR applications
- **esbuild builder (developer preview)** — Faster builds

## Angular 17
- **Node 18+ required** — `nvm use 20`
- **New control flow syntax** — `@if`, `@for`, `@switch` replace structural directives
  ```bash
  ng generate @angular/core:control-flow
  ```
- **Deferrable views** — `@defer` for lazy-loading template sections
- **esbuild builder stable** — Recommended for new projects
- **New application builder** — `application` builder replaces `browser`

## Angular 18
- **Signals stable** — Production-ready reactive primitives
- **Zoneless (experimental)** — `provideExperimentalZonelessChangeDetection()`
- **Route redirects with functions** — `redirectTo` can be a function
- **Material 3** — New Material Design components

## Angular 19
- **Standalone by default** (Critical) — All components/directives/pipes default to `standalone: true`
  - Schematics should add `standalone: false` to existing module-based components
  - If missed, use: `grep -r "@Component" --include="*.ts" -l | xargs grep -L "standalone"`
- **Stable signals input/output** — `input()`, `output()`, `model()`
- **Incremental hydration** — For SSR apps
- **Linked signals** — `linkedSignal()` for derived mutable state

## Angular 20
- **`standalone: false` required explicitly** — No longer just a default
- **Zoneless stable** — `provideZonelessChangeDetection()`

## Angular 21
- **Incremental improvements** — Mostly polish and performance
- **TypeScript 5.9 required** — New TS features available

## Common Cross-Version Issues

### NG0912: Duplicate Component ID
- **Versions:** 19+
- **Cause:** Two components with identical selector + template in the same runtime
- **Common in:** Monorepos where components are copy-pasted between apps
- **Fix:** Convert to standalone, share from single source

### NG0201: No Provider for Service
- **When:** Converting components to standalone
- **Cause:** Standalone components don't inherit NgModule providers
- **Fix:** Add `providers: [Service]` to the standalone component

### Expression Changed After Check
- **When:** Adding OnPush or signals
- **Cause:** Signal updates during change detection cycle
- **Fix:** Use `untracked()` for reads that shouldn't trigger re-evaluation
