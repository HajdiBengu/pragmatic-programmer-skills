---
name: pragmatic-database-audit
description: Read-only audit of database schemas, DDL, migration scripts and history, schema snapshots, ORM models, and database constraints, indexes, and procedures against Pragmatic Programmer principles. Use when asked to review schema design, migration safety, data integrity, schema/model drift, or the maintainability of schema evolution.
---

# Pragmatic Database Audit

## Boundaries

- Read-only. Never apply or roll back migrations, connect with write access, create shadow or temporary databases, alter a database, or install or upgrade dependencies.
- Do not query production data. Do not print credentials from connection strings, env files, or CI config.
- Run only commands that read, list, diff, lint, or render SQL offline using tools already present (e.g. `git log`/`git blame`, `alembic heads`/`history`, `alembic upgrade --sql`, `manage.py sqlmigrate`, `manage.py makemigrations --check --dry-run`, `prisma validate`, an installed SQL linter). If a command would connect to a database or write files, skip it and record it as a limitation.

## 1. Establish context

Determine from config, CI, container files, docs, and conventions (README, ADRs, CONTRIBUTING, migration guides):

- Database engine and every supported version (dev, CI, production, minimum supported).
- Migration framework and how order is determined (revision graph, timestamps, version numbers).
- Authoritative source of truth: migrations, snapshot (`schema.rb`, `structure.sql`, `schema.prisma`, dumps), or ORM models.
- Deployment model: when migrations run relative to app rollout, whether old and new app versions run concurrently, whether migrations may be rerun or run manually.
- Stated policies: reversibility, data retention, backward compatibility, approved rollout patterns.

Record unknowns. Do not fill them with worst-case assumptions; findings that depend on an unknown are stated as conditional.

## 2. Discover assets

Locate migration directories and changelogs, `*.sql`, schema snapshots, ORM model definitions, stored procedures/triggers/views, and DB-related config. Build a map of: the migration chain in order, which tables each model maps to, and which code paths write to each table (ORM, raw SQL, jobs, other services). Seed and fixture data are out of scope unless they alter schema.

## 3. Inspect

Apply each check only where evidence exists. Expand inspection when a signal appears; do not exhaustively enumerate harmless cases.

**DRY / single source of truth**
- Same table, column, index, or constraint defined in more than one place with different definitions.
- Model declares a field, type, nullability, default, index, or relation that the migration history does not produce, or vice versa.
- Snapshot out of sync with migrations.

**Design by contract / crash early**
- An invariant the code relies on (uniqueness check, required field, relation, allowed values, ranges) is enforced only in application code, while writes can bypass that code or race (concurrent requests, raw SQL, other writers). Cite both the invariant and the bypass path.
- Do not flag a missing constraint without evidence that the invariant exists. Accept documented deliberate omissions (e.g. FKs avoided for sharding or partitioning).

**Don't program by coincidence / don't assume, prove it**
- Migrations that assume undocumented data states: `NOT NULL` without default or backfill; unique constraint or FK added without deduplication or orphan cleanup; type casts that can fail, truncate, or lose precision/timezone; backfills relying on hard-coded IDs, row order, or environment-specific data.
- Data migrations that import current application models instead of historical/frozen ones, so they break as models evolve.
- SQL or features unsupported by any supported engine version.

**Reversibility / no final decisions**
- Destructive operations (drop, truncate, narrowing, lossy rename/recast) without a data-preservation or recovery path where retention or rollback obligations apply.
- Down migrations that silently lose data, are empty no-ops, or cannot run, where the project requires reversibility. An explicitly marked irreversible migration is acceptable unless policy says otherwise.

**Orthogonality / decoupling app from schema**
- Under the project's actual deploy model, a change that breaks the app version still running: dropping or renaming a column or table it reads, adding a required column it does not write, changing a type it binds to. Flag only when old and new versions can coexist against the same schema.

**Locking and downtime (engine- and version-specific)**
- Operations that rewrite tables or hold strong locks on tables that are large or hot, e.g. PostgreSQL index creation without `CONCURRENTLY`, FK or check constraints added without `NOT VALID` + later validation, column type changes forcing a rewrite; MySQL `ALTER` that cannot use `INSTANT`/`INPLACE` on the deployed version.
- Conflicts with the framework's transaction wrapping (e.g. `CONCURRENTLY` inside a transaction).
- If table size or traffic is unknown, mark the finding conditional.

**Migration ordering and history integrity**
- Duplicate or colliding versions, unmerged multiple heads, references to objects created by later migrations, missing dependency declarations.
- Migrations edited after release or merge (check `git log --follow`), which diverges environments that already applied the original.

**Ubiquitous automation / reproducibility**
- Schema cannot be rebuilt from migrations alone: required manual steps, out-of-band DDL, or procedures existing only in the database.
- Engine version mismatches across dev, CI, and production that change behavior.

**Idempotence**
- Flag non-idempotent operations only when reruns are expected or plausible: no tracking table, manually run scripts, repeatable migrations (e.g. Flyway `R__`), startup or bootstrap DDL.

**Broken windows**
- Inconsistencies (naming, conventions) only when they cause concrete confusion, errors, or drift. Skip pure style preferences.

## 4. Verify each candidate

- Re-read the cited lines and confirm the evidence.
- Search for mitigations: a later migration that fixes it, deployment docs, comments, wrapper tooling, policy exceptions.
- For migrations already applied in production, recommend a new forward migration or operational step, not editing the applied file.
- Discard anything that is preference, speculation, or already mitigated.

## 5. Severity

- **Critical**: likely data loss, corruption, or prolonged outage on the normal deploy path.
- **High**: realistic integrity violations, failed deploys, or significant locking/downtime.
- **Medium**: drift, ordering, compatibility, or reproducibility defects that cause failures or wrong schema in some environments; conditional high-impact risks.
- **Low**: maintainability defects with limited concrete impact.

Rate by consequence and likelihood in this project's context, not by ease of detection. Leave categories empty when nothing qualifies.

## 6. Report

**Scope**: assets inspected, engine and versions, migration framework, deploy-model assumptions, tools/commands run.

**Findings**, ordered by severity:

- **Title**: name the finding
- **Severity**: Critical / High / Medium / Low (append "conditional on X" when applicable)
- **Location**: file:line, migration revision, table/column/constraint, or model declaration
- **Finding**: what is wrong and the specific evidence
- **Principle**: the relevant Pragmatic Programmer principle and why it applies here
- **Recommendation**: the smallest safe correction supported by the engine, framework, and deploy context

**Limitations**: checks not performed or not completable, and why (no DB access, unknown table sizes, undocumented deploy model, tool unavailable).

If nothing actionable was found, say so explicitly and list what was checked. Never claim checks that were not performed.
