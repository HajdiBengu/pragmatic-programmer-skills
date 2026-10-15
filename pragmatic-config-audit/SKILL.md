---
name: pragmatic-config-audit
description: Read-only audit of application and environment configuration (config files, environment-variable definitions, .env templates, feature flags, and config templates or overlays) and the code that consumes them, against Pragmatic Programmer principles. Use when asked to review configuration correctness, consistency across environments, defaults and overrides, startup validation, secret exposure, duplicated or dead settings, or separation of configuration from code.
---

# Pragmatic Config Audit

## Boundaries

- Read-only. Do not edit configuration, toggle flags, set variables in shared environments, deploy, restart services, or install dependencies.
- Never print secret values. Avoid output that resolves or reveals them (env dumps, `printenv`, `docker compose config`, rendered templates with real values, `git log -p` on secret-bearing files, decryption of sealed or encrypted files). Report exposure by location and type only (e.g. "AWS access key, `deploy/prod.env:7`").
- Do not call flag services, secret managers, or cloud APIs. Run only commands that read, list, search, diff, or validate offline with tools already present (e.g. `git ls-files`, `git grep`, `git log -S <key-name>` without `-p`, schema validators, `helm template`/`kustomize build` against example values, the project's own non-destructive config check). If a command could expose secrets, contact a remote service, or write files, skip it and record it as a limitation.

## 1. Establish context

Determine from code, CI, container and deploy files, and docs (README, deployment guides, ADRs, `.env.example`, config schemas):

- Loading mechanism: loader libraries, file formats, profiles, templating or overlay tools, flag system.
- Environments and how each receives values (committed files, CI variables, secret manager, orchestrator).
- Effective precedence order (defaults, files, profiles, env vars, CLI), as implemented by the loader, not as assumed.
- Ownership and stated conventions: which file or system is authoritative for which settings, naming rules, required-vs-optional policy.

Record unknowns. Findings that depend on an unknown are stated as conditional.

## 2. Discover assets

Locate config files (`config/`, `settings*`, `*.{yml,yaml,json,toml,ini,properties,conf}`), env files and templates (`.env*`), Dockerfile `ENV`/`ARG`, compose `environment`/`env_file`, Kubernetes ConfigMaps/Secrets, Helm values, IaC variable files, CI variable blocks, and feature-flag definitions. Check tracked status with `git ls-files` and `.gitignore`.

Find consumers by searching for env and config reads (`process.env`, `os.environ`, `os.getenv`, `System.getenv`, `ENV[`, `@Value`, `viper.Get`, the project's config accessor). Build a key map: key → definitions per environment, default, consumers, validation.

## 3. Inspect

Apply each check only where evidence exists. Expand inspection when a signal appears.

**DRY / single source of truth**
- The same setting maintained in multiple places that have diverged or must be synced by hand (e.g. code default differs from file default; port duplicated across compose and app config). Legitimate per-environment values are not duplication.
- Templates or schemas out of sync with what code reads: required keys missing from `.env.example`/values files, or documented keys nothing reads.

**Parameterize with external configuration**
- Environment-specific values (hosts, URLs, account IDs, bucket names, credentials) or environment-name branches (`if env == "prod"`) in code, so an environment change requires a code change. Do not flag environment-independent constants, test fixtures, or local-dev defaults that are reliably overridden.

**Crash early / design by contract**
- Required settings read lazily or with silent fallbacks (empty string, `None`, placeholder), so misconfiguration surfaces late or as wrong behavior instead of failing at startup.
- Unvalidated parsing: `"false"` treated as truthy, unchecked numeric conversion, ambiguous units (ms vs s) without naming or validation.
- Unsafe defaults that apply when an override is missing: debug enabled, auth or TLS verification disabled, production endpoints as defaults. Confirm the override is actually absent in the affected environment.

**Don't program by coincidence**
- Effective value depends on load order, implicit profile merging, or overlay deep-merge vs replace behavior, and differs from what docs or templates imply. Prove it by tracing the loader.
- Key mismatches between definition and consumer: `DATABASE_URL` vs `DB_URL`, typos, case differences, nested-key separator mismatches.

**Orthogonality / hidden dependencies**
- Config read ad hoc deep in modules, creating required keys absent from templates, schemas, or docs. Flag only when it causes undiscoverable requirements or test coupling, not as style.
- One setting silently controlling unrelated behaviors, or one logical change requiring edits across many unrelated files.

**Contract consistency across environments**
- Keys required by consumers but missing in some environment definitions; type or format differing across environments; settings defined only for production with no path exercised elsewhere.

**Secrets**
- Secrets in tracked files, CI config, image layers (`ENV`/`ARG`), or history. Distinguish real-looking credentials from placeholders, dummies, and test keys before flagging. Note tracked env files that conventions say should be ignored.

**Broken windows / dead config**
- Unused, obsolete, or contradictory settings and stale flags only when evidence confirms: no consumer anywhere searched, including scripts, IaC, and dynamic key construction; or a flag fully rolled out with its dead branch remaining. If external services or dynamic access could consume a key, mark it conditional or skip it.

**Ubiquitous automation**
- Config changes needing undocumented manual steps (hand-edited servers, regenerating derived files by hand), or no validation where the project's tooling supports it.

## 4. Verify each candidate

- Re-read the cited lines and confirm the evidence.
- Search for mitigations: overrides in other layers, startup validation elsewhere, deployment docs, comments, documented exceptions.
- For leaked secrets, assume history persists; recommend rotation, not only removal.
- Discard preference, speculation, or already-mitigated items.

## 5. Severity

- **Critical**: live credentials exposed in tracked or distributed assets; misconfiguration on the normal deploy path likely to cause a security breach or data loss.
- **High**: missing or mismatched required settings that fail deploys or cause wrong production behavior; unsafe defaults reachable in production.
- **Medium**: drift, ambiguous precedence, late validation, or cross-environment inconsistencies causing failures in some environments; conditional high-impact risks.
- **Low**: maintainability defects with limited concrete impact.

Rate by consequence and likelihood in this project's context, not by ease of detection. Leave categories empty when nothing qualifies.

## 6. Report

**Scope**: assets and environments inspected, loaders and precedence model, assumptions, tools and commands run.

**Findings**, ordered by severity:

- **Title**: name the finding
- **Severity**: Critical / High / Medium / Low (append "conditional on X" when applicable)
- **Location**: file:line, key name, environment, flag, or resource identifier
- **Finding**: what is wrong and the specific evidence (no secret values)
- **Principle**: the relevant Pragmatic Programmer principle and why it applies here
- **Recommendation**: the smallest practical correction consistent with the project's conventions

**Limitations**: checks not performed or not completable, and why (no access to CI or secret-manager values, external consumers unknown, tool unavailable, command skipped for secret safety).

If nothing actionable was found, say so explicitly and list what was checked. Never claim checks that were not performed.
