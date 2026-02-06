---
name: Angular Version Upgrade
description: Use this skill when the user asks to "upgrade Angular", "update Angular version", "ng update", "migrate Angular X to Y", "update dependencies", or when performing any Angular major version upgrade. Provides the complete methodology for safely upgrading Angular through multiple major versions, including dependency ordering, third-party library compatibility, and build verification protocols. Encodes RULE 6 (duplicate selectors) and RULE 8 (ng update schematics).
version: 1.0.0
---

# Angular Version Upgrade

## Purpose

Guide Angular major version upgrades through a safe, incremental process that avoids the common pitfalls discovered during real upgrades. This skill encodes the methodology for upgrading through 12+ major versions with minimal rework.

## Core Principles

1. **One major version at a time** — Never skip versions. Run `ng update` for each major version sequentially.
2. **Build verification after every bump** — `yarn build:dev` must succeed before proceeding.
3. **Never skip ng update schematics** (RULE 8) — Schematics handle hundreds of mechanical changes. Skipping them caused 243 components to break silently.
4. **Scan for monorepo collisions** (RULE 6) — Duplicate component selectors across apps cause NG0912 errors in Angular 19+.

## Pre-Upgrade Checklist

Before starting any version upgrade:

1. **Clean git state** — Commit or stash all changes
2. **Run `list_projects`** — Understand workspace structure, identify all apps
3. **Run `get_best_practices`** — Get version-specific standards for the TARGET version
4. **Create baseline tests** — See `angular-upgrade-testing` skill
5. **Audit for duplicate selectors** (RULE 6):
   ```bash
   # Find all component selectors across both apps
   grep -r "selector:" --include="*.ts" src/ projects/ | grep -oP "'[^']+'" | sort | uniq -d
   ```

## Upgrade Protocol

### Step 1: Run ng update with schematics (RULE 8)

```bash
npx ng update @angular/core@{TARGET} @angular/cli@{TARGET}
```

**CRITICAL:** Let schematics run. They handle:
- TypeScript compatibility updates
- API deprecation replacements
- Module system changes (e.g., `standalone: false` in Angular 19)
- Import path updates

If schematics fail, read the error and fix the specific issue — do NOT bypass with `--force`.

### Step 2: Update third-party libraries

Consult `references/dependency-upgrade-order.md` for the correct order. Key rule: update Angular packages first, then third-party libraries that depend on Angular.

### Step 3: Build verification

```bash
yarn build:dev    # Build ALL apps in the monorepo
yarn lint         # Check for linting errors
yarn test         # Run test suite
```

All three must pass before proceeding to the next version.

### Step 4: Commit

```bash
git commit -m "Upgrade Angular {FROM} → {TO}"
```

### Step 5: Repeat for next version

## Critical Version Checkpoints

Consult `references/version-compatibility-matrix.md` for full details.

| Version | Critical Change | Action Required |
|---------|----------------|-----------------|
| 12 | TSLint removed | Migrate to ESLint before upgrading past 12 |
| 16 | ngcc removed | ALL libraries must be Ivy-compatible. Rebuild local .tgz packages |
| 17 | Node 20 required | `nvm use 20` before upgrading |
| 17 | New control flow | Optional: `ng generate @angular/core:control-flow` |
| 19 | Standalone default | Schematics add `standalone: false` to module-based components |
| 19 | Stable Signals | Optional: `ng generate @angular/core:signals` for bulk migration |

## Monorepo Collision Protocol (RULE 6)

Before upgrading to Angular 19+:

1. **Find duplicate selectors:**
   ```bash
   grep -rh "selector:" --include="*.ts" src/app projects/penalty-inform/src | \
     grep -oP "'[a-z][-a-z0-9]*'" | sort | uniq -d
   ```

2. **For each duplicate:** Convert to standalone component in a shared location, import from single source.

3. **Verify DI:** Standalone components may need explicit providers:
   ```typescript
   @Component({
     standalone: true,
     imports: [CommonModule],
     providers: [SomeService]  // If needed
   })
   ```

## Local Package Strategy

For local .tgz packages (soft-ngx, trunks-ui):

### Before Angular 16 (ngcc removal)
- Obtain source code
- Rebuild with `partial` compilation mode
- OR migrate functionality into the monorepo

### Recommended: Monorepo Migration
Instead of maintaining separate .tgz files:
1. Create `projects/soft-ngx/` directory
2. Move source code into monorepo
3. Update path aliases in tsconfig.json
4. Remove .tgz from dependencies

## Escalation: When ng update Fails

1. Read the full error message
2. Check `references/breaking-changes-catalog.md` for known issues
3. Search Angular docs: `mcp__angular-cli__search_documentation`
4. Try `--force` flag ONLY after understanding what it skips
5. If a peer dependency blocks the upgrade, update that dependency first

## References

- `references/version-compatibility-matrix.md` — Angular/Node/TS/RxJS version matrix
- `references/dependency-upgrade-order.md` — Correct order for updating dependencies
- `references/breaking-changes-catalog.md` — Known breaking changes by version
