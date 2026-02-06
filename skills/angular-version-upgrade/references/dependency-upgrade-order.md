# Dependency Upgrade Order

## Principle

Always upgrade in dependency order: Angular core first, then Angular ecosystem, then third-party libraries that depend on Angular, then standalone utilities.

## Order for Each Version Bump

### 1. Angular Core (always first)

```bash
npx ng update @angular/core@{VERSION} @angular/cli@{VERSION}
```

This handles:
- `@angular/core`
- `@angular/common`
- `@angular/compiler`
- `@angular/forms`
- `@angular/platform-browser`
- `@angular/platform-browser-dynamic`
- `@angular/router`
- `@angular/animations`
- `@angular/compiler-cli`

### 2. Angular Supplementary Packages

```bash
npx ng update @angular/localize@{VERSION}  # If used
npx ng update @angular/cdk@{VERSION}       # If used
npx ng update @angular/material@{VERSION}  # If used
```

### 3. Build Tooling

```bash
# These are usually updated by ng update, but verify:
@angular-devkit/build-angular
@angular-eslint/schematics
```

### 4. Third-Party Angular Libraries (in order)

Update these after Angular core matches the target version:

```bash
# UI frameworks
yarn add ngx-bootstrap@{MATCHING_VERSION}
yarn add @swimlane/ngx-datatable@{MATCHING_VERSION}
yarn add @ng-select/ng-select@{MATCHING_VERSION}

# Auth
yarn add angular-oauth2-oidc@{MATCHING_VERSION}

# Charts
yarn add angular-highcharts@{MATCHING_VERSION}
yarn add highcharts@{MATCHING_VERSION}

# Other
yarn add ngx-mask@{MATCHING_VERSION}
yarn add ng2-pdf-viewer@{MATCHING_VERSION}
```

### 5. Non-Angular Dependencies

These can be updated independently:
- RxJS (usually stays at 6.x or 7.x)
- TypeScript (constrained by Angular version)
- Zone.js (constrained by Angular version)
- SCSS tooling

### 6. Local Packages

Update these LAST, after verifying Angular core works:
- `soft-ngx` — Rebuild or migrate into monorepo
- `trunks-ui` — Rebuild or patch

## Common Errors and Fixes

### Peer Dependency Conflicts

```bash
# If yarn fails due to peer deps:
yarn add package@version --ignore-engines

# If ng update fails:
npx ng update package@version --force
```

**Use `--force` sparingly** — understand what it skips first.

### Version Mismatch After Update

If `@angular/core` and `@angular/cli` are on different major versions:
```bash
npx ng update @angular/cli@{SAME_VERSION_AS_CORE}
```

### RxJS 6 to 7 Migration

If upgrading RxJS alongside Angular:
```typescript
// toPromise() deprecated
// Before:
const result = await observable.toPromise();
// After:
import { lastValueFrom } from 'rxjs';
const result = await lastValueFrom(observable);
```
