---
allowed-tools: ["Read", "Grep", "Glob", "Write", "Edit", "Bash", "Task", "WebSearch", "mcp__angular-cli__get_best_practices", "mcp__angular-cli__search_documentation", "mcp__angular-cli__find_examples", "mcp__angular-cli__list_projects", "mcp__angular-cli__onpush_zoneless_migration"]
description: Full Angular upgrade orchestration — analyze risks, plan migration, execute changes, review for bugs, verify build. Encodes 8 anti-pattern rules from real upgrade experience.
---

# Angular Upgrade Orchestration

You are executing a full Angular upgrade workflow. Follow the phases below in order. Never skip a phase. Never revert without trying at least 2 alternatives first.

## Context

```
Project workspace: `! mcp__angular-cli__list_projects`
Current Angular version: `! grep "@angular/core" package.json | head -1`
Current branch: `! git branch --show-current`
Git status: `! git status --short`
```

## Phase 1: Analysis

**Spawn the upgrade-analyzer agent** to perform a read-only risk assessment.

The analyzer will check all 8 rules:
1. RULE 1: Input mutation pre-scan
2. RULE 2: Complex component identification (10+ inputs)
3. RULE 3: SCSS internalization gaps (if applicable)
4. RULE 4: Subscribe callback audit for OnPush
5. RULE 5: Component inventory (not phase numbers)
6. RULE 6: Duplicate selector scan for monorepo
7. RULE 7: Missing signal () in existing code
8. RULE 8: Ensure ng update schematics will run

**Output:** Risk assessment report with recommended migration order.

## Phase 2: Planning

Based on the analyzer's report:

1. **Create a component inventory** tracking each component's migration status
2. **Order components** by dependency (base → shared → feature)
3. **Group by risk level** (trivial → moderate → complex)
4. **Identify blocking dependencies** (e.g., BaseDropdown must be migrated before 40+ child dropdowns)

Present the plan to the user for approval before proceeding.

## Phase 3: Execution

**Spawn the migration-executor agent** for each migration task.

### For Version Upgrades:
1. One major version at a time
2. `ng update` with schematics (RULE 8)
3. Update third-party libraries
4. Build + lint + test after each version

### For Signal Migration:
1. Pre-scan inputs (RULE 1) — classify by binding type
2. Complex components atomically (RULE 2)
3. Run `ng generate @angular/core:signals` for bulk migration
4. Manually fix edge cases the tool can't handle
5. Post-migration () verification (RULE 7)

### For OnPush Migration:
1. Subscribe callback audit (RULE 4)
2. Run `mcp__angular-cli__onpush_zoneless_migration` for guidance
3. Three phases: trivial → outputs → complex
4. Fix object mutations
5. Enable zoneless after all components are OnPush

### For SCSS Migration:
1. Internalize frameworks
2. Run internalization audit (RULE 3)
3. Convert @import to @use
4. Add !default flags

**After EACH component migration:**
```bash
yarn build:dev    # Both apps must compile
yarn lint         # Must pass
```

## Phase 4: Review

**Spawn the migration-reviewer agent** to check for common post-migration bugs.

The reviewer checks:
1. Missing signal () calls (RULE 7)
2. Object mutations on signals
3. Two-way binding issues
4. Effect timing issues
5. Subscribe leaks
6. Test compatibility
7. SCSS gaps (RULE 3)

**Fix all CRITICAL and HIGH issues** before proceeding.

## Phase 5: Verification

Final verification checklist:

```bash
# Build both apps
yarn build:dev

# Run linter
yarn lint

# Run tests
yarn test --watch=false

# Check for remaining @Input()
grep -rc "@Input()" --include="*.ts" src/ projects/ | grep -v ":0$" | sort -t: -k2 -nr

# Check for remaining @Output()
grep -rc "@Output()" --include="*.ts" src/ projects/ | grep -v ":0$" | sort -t: -k2 -nr

# Check for remaining ngOnChanges
grep -rc "ngOnChanges" --include="*.ts" src/ projects/ | grep -v ":0$"

# Verify no missing signal ()
grep -rn "= input\|= model" --include="*.ts" src/ projects/ | wc -l
```

Present final results to the user with:
- Components migrated
- Issues found and fixed
- Remaining items (if any)
- Recommended next steps

## Guard Rails

Before executing any migration step, mentally verify:

- [ ] RULE 1: Have I scanned for input mutations on THIS component?
- [ ] RULE 2: Is this a complex component? If so, am I migrating it atomically?
- [ ] RULE 3: If SCSS, have I run the internalization audit?
- [ ] RULE 4: Have I checked subscribe callbacks for OnPush compatibility?
- [ ] RULE 5: Am I tracking by component name, not phase number?
- [ ] RULE 6: Have I checked for duplicate selectors (if upgrading to 19+)?
- [ ] RULE 7: After migration, have I verified all signal () calls?
- [ ] RULE 8: Am I using ng update schematics (not manual upgrade)?

## Escalation Protocol

When an unfamiliar error arises:

1. **Research** — Angular docs, examples, web search
2. **Diagnose** — Full stack trace, root cause
3. **Try 2+ alternatives** — Document each attempt
4. **Document** — Record for future reference
5. **Ask user** — Only when design decision is wrong

**NEVER revert silently.** Always explain what happened and propose alternatives.
