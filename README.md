# Softever Angular Upgrade Plugin

A Claude Code plugin encoding battle-tested methodology for Angular major version upgrades. Built from real experience upgrading Angular 9 to 21 across a monorepo with 93 commits, where ~23% were corrective fixes for preventable mistakes.

## The 8 Anti-Pattern Rules

| # | Rule | What Goes Wrong Without It |
|---|------|---------------------------|
| 1 | **Pre-scan for input mutations** before signal migration | Components with `@Input` mutation break with `model()` — requires `linkedSignal` or alias+local-copy |
| 2 | **Migrate complex core components atomically** | Grid component took 5 incremental fix commits over 3 days |
| 3 | **Audit SCSS internalization** for missing rules | Lost CSS rules (icon margins, push classes, block spacing) invisible to compilers |
| 4 | **Audit subscribe() callbacks** before OnPush | Every `subscribe()` that sets a property silently breaks with OnPush CD |
| 5 | **Track by component inventory**, not phase number | Same "Phase 5" was done 16 times across different component scopes |
| 6 | **Scan for duplicate selectors** in monorepo | NG0912 duplicate component ID errors in Angular 19+ |
| 7 | **Regex scan for missing signal `()`** in templates | Signal functions are truthy — `@if (canEdit)` compiles but always passes |
| 8 | **Never skip ng update schematics** | Manual upgrade missed `standalone: false` on 243 components |

## Skills

| Skill | Trigger | Purpose |
|-------|---------|---------|
| `angular-version-upgrade` | "upgrade Angular", "ng update" | Core version upgrade methodology |
| `angular-signals-migration` | "migrate to signals", "convert @Input" | Signal patterns, edge cases, anti-patterns |
| `angular-onpush-zoneless` | "OnPush migration", "zoneless" | Change detection strategy migration |
| `angular-scss-migration` | "SCSS migration", "@import to @use" | SCSS modernization and framework internalization |
| `angular-upgrade-testing` | "upgrade tests", "baseline tests" | Self-verification testing for upgrades |

## Agents

| Agent | Purpose | Mode |
|-------|---------|------|
| `upgrade-analyzer` | Pre-migration risk assessment | Read-only |
| `migration-executor` | Execute migrations with verification | Read/write |
| `migration-reviewer` | Post-migration bug detection | Read-only |

## Command

`/angular-upgrade` — Full orchestration workflow: analyze, plan, execute, review, verify.

## Tool Strategy

**Tools first (~70%), manual expertise where tools fail (~30%):**

- `ng update @angular/core@X @angular/cli@X` — version upgrades with auto-schematics
- `ng generate @angular/core:signals` — bulk signal migration
- `ng generate @angular/core:control-flow` — template syntax migration
- `mcp__angular-cli__onpush_zoneless_migration` — OnPush guidance
- `mcp__angular-cli__get_best_practices` — version-specific standards

Manual handling needed for: input-mutating components, complex two-way bindings, monorepo collisions, SCSS internalization, subscribe callback timing, and post-migration `()` verification.

## Installation

The plugin is auto-loaded from `~/.claude/plugins/softever-angular-upgrade/`.

To verify: start a new Claude Code session and check that skills, agents, and the `/upgrade` command are available.

## Escalation Protocol

When stuck, follow this chain instead of reverting:

1. **Research** — Angular docs, examples, web search for the exact error
2. **Diagnose** — Read full stack trace, identify root cause
3. **Try alternatives** — Document failed approach, try 2+ alternatives
4. **Document** — Record issue + solution for future reference
5. **Ask user** — Only when the design decision itself is wrong
