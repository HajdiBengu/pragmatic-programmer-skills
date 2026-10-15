---
name: pragmatic-dependencies-audit
description: Read-only audit of dependency manifests, lockfiles, package-manager and registry configuration, version constraints, vendored code, build-tool plugins and wrappers, and supply-chain metadata against Pragmatic Programmer principles. Use when asked to review dependency hygiene, build reproducibility, manifest/lockfile consistency, undeclared or unnecessary dependencies, package provenance and integrity, install-time script risk, or vulnerability and license exposure.
---

# Pragmatic Dependencies Audit

## Boundaries

- Read-only. Do not install, update, add, remove, or resolve packages. Do not edit manifests or lockfiles. Do not run dependency code, lifecycle scripts, build plugins, or project builds.
- Allowed: reading, listing, searching, diffing, and commands that neither write project files nor execute package code, such as `git ls-files`, `git grep`, `npm ls`, `pnpm why`, `cargo tree --locked --offline`, `go mod verify`, `go mod tidy -diff`, `poetry check --lock`, `uv lock --check`. Vulnerability scanners only in modes that read an existing lockfile without resolving, building, or installing (`npm audit`, `pnpm audit`, `cargo audit`, `osv-scanner` on lockfiles). Skip any command whose mode would resolve, install, or build, and record it as a limitation.
- Never print secret values. Report exposure by location and type only.
- Do not flag the absence of a tool or pattern (lockfile, update bot, hash pinning, scanner) by itself. Flag it only when a concrete failure path exists in this project.

## 1. Establish context

From manifests, CI configs, Dockerfiles, docs, and release configuration, determine:

- Package managers and their versions (`packageManager` field, `.tool-versions`, wrapper files, CI setup steps).
- Project type per manifest: deployed application or service, published library, internal tool. Libraries legitimately use version ranges; deployed artifacts need reproducible resolution.
- How dependencies are installed in CI, containers, and release builds, and which dependency groups ship to production.
- Existing security and update tooling: Dependabot/Renovate config, scanners in CI, license policy, documented exceptions, `overrides`/`resolutions`/constraints files and their stated reasons.
- Monorepo layout: workspaces, version catalogs, shared constraint files.

Record unknowns. Findings that depend on an unknown are stated as conditional.

## 2. Discover assets

Use `git ls-files` to find: manifests (`package.json`, `pyproject.toml`, `requirements*.txt`, `setup.py`/`setup.cfg`, `Pipfile`, `go.mod`, `Cargo.toml`, `Gemfile`, `*.gemspec`, `pom.xml`, `build.gradle*`, `settings.gradle*`, `libs.versions.toml`, `composer.json`, `*.csproj`, `Directory.Packages.props`, `packages.config`), lockfiles (`package-lock.json`, `yarn.lock`, `pnpm-lock.yaml`, `bun.lock*`, `poetry.lock`, `uv.lock`, `Pipfile.lock`, `go.sum`, `Cargo.lock`, `Gemfile.lock`, `composer.lock`, `packages.lock.json`, `gradle.lockfile`), configuration (`.npmrc`, `.yarnrc*`, `pip.conf`, `.pypirc`, `uv.toml`, `.cargo/config*`, `settings.xml`, `nuget.config`, `gradle.properties`, `gradle-wrapper.properties`, `.mvn/`), vendored trees (`vendor/`, `third_party/`, `external/`, copied library files), and dependency-installing steps in CI and Dockerfiles. Also check `.gitignore` for ignored lockfiles.

For each manifest map: its lockfile, its consumers (CI jobs, images, release pipelines), its registries, and its dependency groups.

## 3. Inspect

Apply each check only where evidence exists. Expand inspection when a signal appears.

**Reproducibility (don't program by coincidence)**
- Deployed artifacts without a committed lockfile, or with the lockfile gitignored.
- Lockfile out of sync with its manifest: declared dependencies missing from the lock, locked versions outside declared ranges, removed dependencies still locked. Prove it by comparing entries or with a non-writing check command.
- CI, Docker, or release builds that install without enforcing the lock (`npm install` instead of `npm ci`, missing `--frozen-lockfile`/`--locked`/`--require-hashes`/`--no-update`), so shipped versions differ from tested ones.
- Mutable references in shipped dependencies: `latest`, `*`, git branches or tags without a commit, tarball URLs without integrity, unpinned plugins or wrapper distributions.
- Competing lockfiles for different managers in one package, or a package-manager version that differs between local tooling and CI where the lockfile format depends on it.

**Explicit dependencies (no hidden coupling)**
- Code importing packages not declared in its own manifest, working only through hoisting, transitive resolution, or the global environment. Search imports against declarations per package or workspace.
- Runtime imports satisfied only by dev/test/optional groups, or tools invoked by scripts and CI that no manifest declares.
- Peer or plugin dependencies loaded by configuration rather than imports (bundler, linter, test plugins, `@types/*`); check config files before calling a dependency undeclared or unused.

**Minimality and decoupling**
- Direct dependencies with no reference in code, config, scripts, or build files.
- Multiple libraries serving the same purpose in one deployable, or multiple major versions of the same package where the lock shows both are loaded at runtime.
- `overrides`, `resolutions`, `replace`, `patch`, or forced versions with no recorded reason, or whose reason (an advisory, a fixed upstream bug) no longer applies.
- Recommend removal or replacement only with evidence of redundancy, risk, or cost; never on size or popularity preference alone.

**Provenance and integrity (trust nothing by default)**
- Registries over plain HTTP, disabled TLS or checksum verification (`strict-ssl=false`, `trusted-host`, `GOSUMDB=off`, `GONOSUMDB`/`GOINSECURE` beyond private modules, `--allow-insecure`).
- Dependency-confusion exposure: internal package names resolvable from a public registry, `--extra-index-url` mixing private and public indexes, scopes without explicit registry mapping.
- Lock entries missing integrity hashes where the format supports them, or hashes present but not enforced at install.
- Gradle/Maven wrappers without distribution checksums; plugin repositories that are deprecated, unauthenticated, or HTTP.
- Git or URL dependencies from forks or personal accounts in shipped code with no stated reason.
- Vendored code with no recorded upstream source and version, or locally modified without a record, so updates and advisory matching are impossible.
- Registry credentials committed in config files or install commands. Assume history persists; recommend rotation.

**Install-time execution**
- Direct or transitive dependencies with install/postinstall hooks (`hasInstallScript`, `setup.py` builds, native extensions, Gradle/Maven build plugins) running in CI or release environments that hold secrets, where the project has no script controls (`ignore-scripts`, allowlists such as `onlyBuiltDependencies`, prebuilt wheels).
- Lifecycle scripts in the project's own manifest that fetch and execute remote content.

**Vulnerabilities and licenses**
- Use existing scanner output, CI reports, or the read-only commands above. Classify each item:
  - Confirmed: advisory applies to the exact resolved version in a lockfile, in a group that ships or runs in CI.
  - Unverified: advisory matches by name or range, but the resolved version, scope, or affected code path could not be confirmed.
  - Outdated: newer versions exist, no advisory. Report only when evidence shows end-of-life, deprecation markers in lock metadata, or an incompatibility with a supported platform.
- State the advisory database and date. Never claim a scan that was not run.
- Check licenses only against a policy the project declares or a distribution model it documents; cite the license source (lock metadata, package files, vendored `LICENSE`).

**DRY and contract**
- The same dependency versioned in multiple places (workspaces, catalog and module, requirements and pyproject, CI installs) with divergent values or manual sync burden.
- Docs or setup instructions installing dependencies differently from the lock (unpinned `pip install`, global installs) where they would produce a different environment.
- Update-bot configuration that ignores the manifests actually in use, excludes security updates, or updates manifests without lockfiles.

## 4. Verify each candidate

- Re-read the cited entries and trace whether the dependency reaches a shipped artifact or a privileged CI job.
- Search for mitigations: install flags set in shared CI templates, registry mapping in user-level config documented as required, documented exceptions, scanner suppressions with reasons, workspace-level declarations.
- Do not relay scanner or linter output wholesale; keep only items with a reachable consequence.
- Discard preference, speculation, and already-mitigated items.

## 5. Severity

- **Critical**: live registry credentials committed; dependency-confusion or disabled integrity/TLS verification on the normal production or release install path; confirmed vulnerability in a shipped dependency whose affected functionality the project uses.
- **High**: shipped builds not reproducible on the normal path (missing, ignored, or unenforced lockfile; mutable sources); confirmed advisory in a shipped dependency with usage unverified; undeclared runtime dependencies; install-time code execution in jobs holding release or deploy secrets.
- **Medium**: confirmed advisories limited to dev or CI tooling without secrets; manifest/lockfile drift; duplicated or divergent declarations; stale overrides; unused direct dependencies; unrecorded vendored code.
- **Low**: clarity and maintainability defects with limited concrete impact.

Rate by consequence and likelihood in this project's context, not by ease of detection. Leave categories empty when nothing qualifies.

## 6. Report

**Scope**: manifests, lockfiles, and configs inspected; package managers and project types assumed; install paths traced; commands and scanners run, with advisory database and date.

**Findings**, ordered by severity:

- **Title**: name the finding
- **Severity**: Critical / High / Medium / Low (append "conditional on X" when applicable)
- **Location**: file:line, manifest entry, lock entry, registry config, or CI job/step
- **Finding**: what is wrong, the triggering condition, and the specific evidence (advisory ID and resolved version for vulnerabilities; no secret values)
- **Principle**: the relevant Pragmatic Programmer principle and why it applies here
- **Recommendation**: the smallest practical correction consistent with the project's package manager, compatibility ranges, and release policy

**Limitations**: checks not performed or not completable, and why (tool unavailable, command skipped because it would resolve or install, no network for advisory data, private registries unreachable, CI or runtime environment not observable).

If nothing actionable was found, say so explicitly and list what was checked. Never claim checks that were not performed.
