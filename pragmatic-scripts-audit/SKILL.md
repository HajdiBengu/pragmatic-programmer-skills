---
name: pragmatic-scripts-audit
description: Read-only audit of shell scripts, automation and build scripts, task runners (Makefile, justfile, Taskfile, package.json scripts, tasks.py, Rakefile), git hooks, maintenance utilities, and repository-defined developer or operational commands against Pragmatic Programmer principles. Use when asked to review script reliability, error handling, repeatability, destructive-action safety, portability, hidden environment assumptions, duplicated automation, or manual steps that should be automated.
---

# Pragmatic Scripts Audit

## Boundaries

- Read-only. Do not run scripts, task targets, or hooks. Do not edit files, install tools, change dependencies, deploy, or touch remote systems.
- Allowed: reading, listing, searching, diffing, and offline static checks with tools already present (`git ls-files`, `git grep`, `bash -n`/`sh -n`, `shellcheck`, the project's configured linters). Use `make -n` only when the target and its prerequisites contain no `$(shell ...)`, `+`-prefixed lines, or `$(MAKE)` recursion, since those execute anyway. If a check could execute side effects, skip it and record it as a limitation.
- Never print secret values. Report exposure by location and type only.
- Do not flag the absence of a pattern (strict mode, a particular shell option, a help flag) by itself. Flag it only when a concrete failure path exists in this project.

## 1. Establish context

Determine from shebangs, CI configs, devcontainer/Docker files, docs (README, CONTRIBUTING, runbooks), and lint configs (`.shellcheckrc`, pre-commit):

- Supported platforms and shells (Linux/macOS/Windows, bash vs POSIX sh, GNU vs BSD tools).
- Who runs each script and how: developers, CI, git hooks, cron, deploy pipelines, other scripts, task-runner targets.
- Intended purpose and run pattern: one-off, routinely rerun, retried by CI, run against shared or production resources.
- Conventions: shared helper libraries, wrapper scripts, style rules, existing lint coverage.

Record unknowns. Findings that depend on an unknown are stated as conditional.

## 2. Discover assets

List candidates with `git ls-files`: `*.sh`, `*.bash`, `*.zsh`, `*.ps1`, `*.bat`, `*.cmd`, files in `bin/`, `scripts/`, `tools/`, `hack/`, executable files with a shebang (`git ls-files -s` mode `100755`), `Makefile`/`*.mk`, `justfile`, `Taskfile.yml`, `package.json` `scripts`, `pyproject.toml` scripts, `tox.ini`/`noxfile.py`/`tasks.py`, `Rakefile`, git hook directories (`.husky/`, `.githooks/`), and non-trivial inline `run:`/`script:` blocks in CI.

For each script build a map: callers (search by filename and target name across CI, docs, other scripts, hooks), inputs (arguments, env vars, files), external commands and tools, side effects (files written or deleted, remote calls, resources created), and exit behavior.

## 3. Inspect

Apply each check only where evidence exists. Expand inspection when a signal appears.

**Crash early / error propagation**
- Failures that matter are swallowed: `|| true`, `2>/dev/null` on critical commands, unchecked exit status, pipelines masking upstream failure when the result is used, `local x=$(cmd)` hiding status, `set -e` silently inactive (in functions called from conditionals, `&&` chains), background jobs never `wait`ed, wrappers that exit 0 after a failed step.
- Prerequisites (tools, env vars, files, auth) used without a check, so failure surfaces late, cryptically, or after partial changes.
- Unchecked `cd`, especially before file-modifying commands.

**Don't program by coincidence / hidden assumptions**
- Relative paths that only resolve from a particular working directory, while some caller invokes the script from elsewhere. Prove it from the caller.
- Shell or tool-flavor features incompatible with a supported platform: bashisms under `#!/bin/sh`, GNU-only flags (`sed -i` without suffix, `date -d`, `readlink -f`, `grep -P`) where macOS/BSD is supported.
- Reliance on undeclared environment state: specific PATH entries, user-specific paths, pre-authenticated CLIs, pre-existing directories, locale.

**Safe input handling**
- Unquoted expansions of paths or external values that can contain spaces or glob characters.
- `eval`, `sh -c`, or string-built commands incorporating arguments, env vars, filenames, or CI event data (branch names, PR titles).
- Unknown or missing arguments silently ignored or defaulted to empty values that change behavior.
- Predictable temp-file names in shared directories where `mktemp` is available.

**Destructive actions need safeguards**
- `rm -rf`, truncation, force-push, `DROP`, `terraform destroy`, `kubectl delete`, bucket/database wipes whose target comes from a variable that may be empty or unset (`rm -rf "$DIR"/*`), from a production default, or from unvalidated input.
- No confirmation, dry-run, or environment guard where the script can act on shared or production resources.

**Repeatability**
- Scripts expected to be rerun (setup, provisioning, CI retries, migration wrappers) that fail or duplicate state on a second run: unconditional appends, `mkdir` without `-p`, creating resources without an existence check.
- Interrupted runs leaving temp files, locks, or half-applied state with no cleanup (`trap`) or recovery path.

**DRY**
- The same procedure implemented in several places (script, Makefile, CI step, docs) that has diverged or must be synced by hand; CI reimplementing a repo script so local and CI behavior can drift. Recommend consolidation only where divergence or sync burden is evident.

**Parameterize instead of hardcoding**
- Environment-specific hosts, account IDs, regions, bucket names, absolute personal paths, or credentials embedded in scripts.
- Secrets exposed at runtime: `set -x` around secret use, secrets passed as command-line arguments, secrets echoed to logs.

**Communicate the contract**
- Scripts taking arguments or performing side effects with no stated prerequisites, inputs, or effects (usage text, header comment, or docs), where callers would otherwise have to read the code.
- Errors written without context or to stdout where callers parse output; ambiguous exit codes that callers depend on.
- Docs or task targets referencing scripts, flags, or targets that no longer exist or behave differently.

**Ubiquitous automation**
- Repeated, error-prone procedures (setup, release, rotation) documented as manual multi-step instructions with no script.
- Scripts that require hand-editing before use ("change this value").
- Scripts excluded from lint or validation tooling the project already applies to comparable scripts.

**Broken windows / dead scripts**
- Scripts with no caller anywhere searched that also reference removed paths, targets, or tools. If external callers (cron, ops runbooks outside the repo) are possible, mark conditional.

## 4. Verify each candidate

- Re-read the cited lines and trace the real invocation (caller, working directory, arguments, environment) to confirm the failure path is reachable.
- Search for mitigations: strict mode set by a sourced library or wrapper, CI guaranteeing a tool or directory, guards earlier in the call chain, documented exceptions.
- Do not relay linter output wholesale; keep only issues with a reachable consequence.
- For leaked secrets, assume history persists; recommend rotation, not only removal.
- Discard preference, speculation, and already-mitigated items.

## 5. Severity

- **Critical**: destructive or irreversible operation reachable on a normal path against an unintended target; live credentials exposed in tracked scripts or logs; command injection from untrusted input in CI or operational scripts.
- **High**: failures masked so CI, builds, or deploys report success or proceed with partial state; script broken under normal invocation on a supported platform.
- **Medium**: non-idempotent behavior where reruns are expected; fragile assumptions that break some callers or environments; drift between duplicated logic; late or misleading failures.
- **Low**: clarity and maintainability defects with limited concrete impact.

Rate by consequence and likelihood in this project's context, not by ease of detection. Leave categories empty when nothing qualifies.

## 6. Report

**Scope**: assets inspected, callers traced, supported platforms assumed, tools and commands run.

**Findings**, ordered by severity:

- **Title**: name the finding
- **Severity**: Critical / High / Medium / Low (append "conditional on X" when applicable)
- **Location**: file:line, task-runner target, hook, or CI job/step
- **Finding**: what is wrong, the triggering condition, and the specific evidence (no secret values)
- **Principle**: the relevant Pragmatic Programmer principle and why it applies here
- **Recommendation**: the smallest practical correction consistent with the project's conventions

**Limitations**: checks not performed or not completable, and why (tool unavailable, check skipped because it could execute side effects, external callers unknown, CI or runtime environment not observable).

If nothing actionable was found, say so explicitly and list what was checked. Never claim checks that were not performed.
