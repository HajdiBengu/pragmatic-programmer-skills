---
name: pragmatic-iac-audit
description: Read-only audit of infrastructure-as-code and deployment definitions (Terraform, OpenTofu, CloudFormation, CDK output, Pulumi, Kubernetes manifests, Kustomize, Helm charts) against Pragmatic Programmer principles. Use when asked to review IaC for drift, reproducibility, hidden dependencies, unsafe defaults, destructive or replacing changes, public exposure, excessive privileges, secret handling, unpinned external inputs, or whether changes can be validated before apply.
---

# Pragmatic IaC Audit

## Boundaries

- Read-only. Never run `apply`, `destroy`, `import`, `state` subcommands that write, `force-unlock`, `taint`, `kubectl apply/create/delete/patch`, `helm install/upgrade/uninstall`, CloudFormation create/update/execute-change-set, or anything that alters infrastructure, state, or locks.
- Do not run commands that write into the project tree (`terraform init`, `tofu init`, `helm dependency build/update`, `tflint --init`). If initialization is needed, use a temporary copy with `-backend=false` and `-lockfile=readonly`; otherwise skip and record it as a limitation.
- Allowed offline: `git ls-files`, `git grep`, `git log`/`diff`, `terraform|tofu fmt -check -recursive`, `validate` on already-initialized or temp-copied modules, `cfn-lint`, `helm lint`, `helm template` (vendored dependencies only), `kustomize build`, `kubeconform`, and static scanners (`checkov`, `trivy config`, `tfsec`, `kube-linter`, `conftest` with the project's own policies).
- Run anything that contacts a cloud account or cluster (`plan`, `kubectl diff`, `--dry-run=server`, change-set preview) only if the user explicitly authorizes it and credentials are already present. Use `-lock=false`, do not write plan files into the repo, and do not print values from plans or state.
- Never print secret values. Report exposure by location and type only.
- Do not flag the absence of a tool, module layout, or pinning style by itself. Flag it only when a concrete failure path exists in this project.

## 1. Establish context

From READMEs, CI/CD pipelines, backend configs, Makefiles, and policy files, determine:

- Tools and versions (`required_version`, `.terraform-version`, `.tool-versions`, CI setup steps, `Chart.yaml` `apiVersion`/`kubeVersion`).
- Root modules/stacks/releases vs. reusable modules/charts. Reusable modules legitimately accept ranges and leave choices to callers; roots and environments need reproducible inputs.
- Environments and how they differ (directories, workspaces, tfvars, overlays, values files).
- State backends, locking, and who owns each state or stack.
- How changes reach production: who plans, who reviews, who applies, from which job, with which credentials.
- Declared policies: tagging, encryption, network exposure, approved regions, exceptions, scanner suppressions and their reasons.

Record unknowns. Findings that depend on an unknown are stated as conditional.

## 2. Discover assets

Use `git ls-files` to find `*.tf`, `*.tofu`, `*.tfvars`, `*.tf.json`, `.terraform.lock.hcl`, backend and `terragrunt.hcl` files; CloudFormation/SAM templates (`AWSTemplateFormatVersion`, `Transform`) and parameter files; Kubernetes YAML (`apiVersion` + `kind`), `kustomization.yaml`, `Chart.yaml`, `values*.yaml`, `Chart.lock`, `templates/`; Pulumi/CDK entry points; and IaC steps in CI. Check `.gitignore` for ignored lock or state files and search the tree and history for committed `*.tfstate`, `*.tfstate.backup`, plan files, and kubeconfigs.

Map each root/stack/release to its modules or charts, environments, backend, and the pipeline that applies it.

## 3. Inspect

Apply each check only where evidence exists. Expand inspection when a signal appears.

**Reproducibility (don't program by coincidence)**
- Root modules without version constraints for providers, or with `.terraform.lock.hcl` missing or gitignored, where CI init would resolve different providers than were reviewed.
- Mutable external inputs consumed by a shipped environment: module `source` on a git branch or without `ref`, registry modules without `version`, Helm dependencies without exact versions or without `Chart.lock`, remote templates/URLs, container images by mutable tag in production manifests.
- Data sources whose result changes over time and feeds a replace-forcing argument (`most_recent = true` AMI lookups, latest image/version queries), causing unplanned replacement on unrelated applies.
- Behavior that depends on the operator's environment: provider region, profile, project, or kube context taken implicitly; Kubernetes resources without `namespace` applied via whatever context is active; local-exec scripts relying on locally installed tools.

**State and drift (single source of truth)**
- Shared environments using local state, backends without locking, or state committed to git (Critical if it contains secrets — state usually does).
- The same real resource managed in two places: duplicate names/IDs across stacks, a Kubernetes object in both a Helm chart and raw manifests or Kustomize, a resource both declared and referenced by hardcoded ID.
- `lifecycle { ignore_changes = all }` or broad `ignore_changes` on security- or capacity-relevant attributes, hiding drift without a recorded reason.
- Resources referenced by hardcoded IDs/ARNs/IPs with no declaration or data source anywhere, indicating hand-made infrastructure the code silently depends on.

**Explicit dependencies (no hidden coupling)**
- Cross-stack coupling through naming conventions or copied literals instead of outputs, remote state, exports, or data sources.
- Required ordering not expressed: missing `depends_on` where only side effects link resources (IAM attachment before use, CRDs before custom resources, namespaces before namespaced objects), CloudFormation `DependsOn` gaps, Helm hooks relied on for ordering without weights.
- Kubernetes references (ConfigMap, Secret, ServiceAccount, PVC, StorageClass, IngressClass) to objects defined nowhere in scope and not documented as externally provided.

**Explicit defaults and contracts (design by contract)**
- Variables or parameters without `type`, `description`, or `validation`/`AllowedValues` where an invalid value would cause outages or exposure.
- Unsafe defaults that environments inherit silently: deletion protection off, public access on, encryption off, single-AZ, `skip_final_snapshot = true` on production databases.
- Outputs exposing secrets without `sensitive = true`; module interfaces that leak internal structure callers then depend on.

**Change safety (no surprises, reversible steps)**
- Stateful resources (databases, buckets, volumes, KMS keys, DNS zones, PVCs) without `prevent_destroy`, deletion protection, `DeletionPolicy`/`UpdateReplacePolicy: Retain`, or `persistentVolumeReclaimPolicy: Retain`, where the project has no other recorded protection.
- Edits that force replacement: changes to immutable arguments, renamed resources or modules without `moved` blocks (check `git diff`/history), `count` over lists where reordering shifts indexes, StatefulSet `volumeClaimTemplates` or selector changes.
- Name-unique resources replaced without `create_before_destroy` handling, or with it where uniqueness makes it fail.
- Pipelines that apply without a reviewed plan: `-auto-approve` on unreviewed plans, apply on push without a plan artifact, plan and apply from different commits, change sets executed without preview.

**Least privilege and exposure (trust nothing by default)**
- IAM/RBAC with wildcard actions or resources, admin-equivalent roles, `cluster-admin` bindings, trust policies open to any principal, or CI roles that can modify their own permissions.
- Network exposure beyond intent: `0.0.0.0/0` or `::/0` on admin or data ports, public buckets/endpoints/load balancers not documented as public, open security group egress where policy restricts it.
- Workload hardening gaps with a reachable consequence: `privileged`, `hostPath`, `hostNetwork`/`hostPID`, added capabilities, running as root, unneeded `automountServiceAccountToken`.

**Secrets**
- Plaintext secrets in `*.tfvars`, `values*.yaml`, defaults, CloudFormation parameters without `NoEcho`, `user_data`, or Kubernetes `Secret` manifests (base64 is encoding, not protection).
- Secrets generated or read in Terraform without acknowledging they persist in state, when the state backend lacks encryption or access control.
- Provider credentials or kubeconfigs committed. Assume history persists; recommend rotation.

**DRY and consistency**
- Environments implemented as copied modules or manifests that have diverged unintentionally (compare file sets and diff key values), versus differences that are intentional and documented.
- The same value (CIDR, version, account ID, image) repeated across files with divergent copies.
- Declared tagging/labeling or ownership policy not applied, where it drives cost allocation, backups, or access control.

**Cost and reliability**
- Unbounded or very large autoscaling maximums, expensive capacity provisioned identically in every environment, or missing resource requests/limits in a shared cluster — only with evidence of a concrete cost or contention consequence.
- Single points of failure in resources documented or configured as production-critical (single replica, single AZ, no PodDisruptionBudget for a multi-replica service).

## 4. Verify each candidate

- Re-read the cited lines and trace whether the definition reaches a real environment through a root module, overlay, values file, or pipeline. Unused modules and examples are lower impact.
- Search for mitigations: org policies (SCPs, OPA/Kyverno, Sentinel) referenced in the repo, CI wrappers adding flags, values overridden in environment files, documented exceptions, scanner suppressions with reasons.
- Do not relay scanner output wholesale; keep only items with a reachable consequence in this project.
- Discard preference, speculation, and already-mitigated items.

## 5. Severity

- **Critical**: committed state, credentials, or live secrets; public exposure of data stores or admin interfaces in production; admin-equivalent privileges reachable by untrusted principals; a pending or routine change path that destroys stateful production resources.
- **High**: production not reproducible on the normal path (unlocked providers, mutable module/chart/image sources, implicit context); applies without reviewed plans; missing protection on stateful production resources; wildcard privileges or plaintext secrets in shipped configuration; one resource managed in two places.
- **Medium**: drift hidden by `ignore_changes`; hidden cross-stack coupling; unsafe defaults overridden today but inherited by new environments; unintended environment divergence; missing ordering that currently works by coincidence; non-production exposure.
- **Low**: clarity and maintainability defects with limited concrete impact.

Rate by consequence and likelihood in this project's context, not by ease of detection. Leave categories empty when nothing qualifies.

## 6. Report

**Scope**: roots/stacks/charts and environments inspected; tools and versions assumed; pipelines traced; commands and scanners run (with versions) and whether any contacted live accounts.

**Findings**, ordered by severity:

- **Title**: name the finding
- **Severity**: Critical / High / Medium / Low (append "conditional on X" when applicable)
- **Location**: file:line plus resource address (`module.x.aws_s3_bucket.y`), logical ID, `kind/namespace/name`, or CI job/step
- **Finding**: what is wrong, the triggering condition, and the specific evidence (no secret values)
- **Principle**: the relevant Pragmatic Programmer principle and why it applies here
- **Recommendation**: the smallest practical correction consistent with the project's tools, module conventions, and deployment workflow

**Limitations**: checks not performed or not completable, and why (no initialized modules, plan not authorized, no credentials, remote modules or charts not vendored, org-level policies not visible, live drift not observable).

If nothing actionable was found, say so explicitly and list what was checked. Never claim checks that were not performed.
