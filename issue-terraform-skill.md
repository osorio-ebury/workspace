## Overview

Map and develop a new `@ebb-ai` skill focused on reviewing, validating, and standardizing Terraform/Terragrunt resources across GCP environments. The skill should analyze `terragrunt.hcl` files, detect cross-environment inconsistencies, enforce naming conventions, and guide developers toward Ebury Platform's infrastructure-as-code standards.

---

## Motivation

The Platform team manages hundreds of Terraform resources across 3 environments (dev, stg, prd) and multiple GCP domains. Common issues include:

- Resources existing in one environment but missing in others
- Inconsistent IAM bindings, missing `iam_members` or `dependency` blocks
- Naming convention violations (missing `ebb-` prefix, wrong suffixes)
- Forgotten `terragrunt hcl fmt` formatting before commits
- Missing `[EPT-XXXX]` Jira card reference in commit messages (required by CI pipeline)

A dedicated skill can automate detection of these issues during code review and resource creation, reducing manual review effort and preventing environment drift.

---

## Proposed Skill: `/terraform`

| Aspect | Detail |
|---|---|
| **Skill name** | `terraform` |
| **Invocation** | `/terraform` via `@ebb-ai` chat participant |
| **Location** | `SKILLS/terraform/SKILL.md` + `SKILLS/terraform/directives/` |

---

## Scope

### 1. Cross-Environment Consistency Validation
- Scan a resource path (e.g., `ebb-ebury-connect/dev/pub-sub/topics/my-topic/`) and verify the equivalent exists in all 3 environments (`dev`, `stg`, `prd`)
- Compare `terragrunt.hcl` inputs across environments — detect missing or divergent fields
- Validate that `iam_members` blocks have equivalent entries across environments (adapted for each env's project ID and project number)
- Check that `dependency` blocks reference the correct environment-specific paths

### 2. Naming Convention Enforcement
- **Service Accounts**: must have `ebb-` prefix (e.g., `ebb-wp-portal-api`, not `wp-portal-api`)
- **Roles**: pattern `ebb_<app>_<type>_privilege_role_<env>` (general/least)
- **Topics**: `ebb-<name>-topic`
- **Subscriptions**: `ebb-<name>-sub`
- **Buckets**: `ebb-<name>-<env>`
- **Secrets**: `ebb-<name-app>`
- Validate GCP project ID suffixes: `-dev`, `-staging`, `-prod`

### 3. Terragrunt Structure Validation
- Verify each resource has a valid `terragrunt.hcl` with correct `source` referencing a Terraform module (with `?ref=vX.Y.Z` tag)
- Validate `include` blocks reference the correct root `terragrunt.hcl`
- Check `dependency` blocks point to existing paths
- Detect missing `iam_members` entries for dead-letter topic patterns (Pub/Sub service agent with `roles/pubsub.publisher` and `roles/pubsub.subscriber`)
- **`iam_members` placement**: detect `iam_members` placed as sibling of `pubsub_topics`/`pubsub_subscriptions` instead of INSIDE the object (after `labels`). The Terraform module silently ignores `iam_members` outside the object — permissions are never applied
- **Duplicate `iam_members` blocks**: detect multiple `iam_members` blocks in the same resource object. HCL uses only the LAST block, silently discarding earlier entries
- **Dependency blocks over hardcoded values**: flag `iam_members` using hardcoded SA emails or role paths instead of `dependency` block interpolation. Exception: `gcp-sa-pubsub` service agent (Google-managed SA) may use hardcoded values
- **Labels validation**: flag resources missing required labels (validated by `guardrail_labels_terragrunt.py` in CI)

### 4. Resource-Specific Rules

#### Pub/Sub Topics
- DL topic **must** have `roles/pubsub.publisher` for the Pub/Sub service agent
- DL topic must have IAM binding with the app's least privilege role

#### Pub/Sub Subscriptions
- Subscription **with** `dead_letter_topic` → must have **2 IAM members**:
  1. App SA with least privilege role (via dependency block)
  2. Pub/Sub service agent with `roles/pubsub.subscriber`
- Subscription **without** DL or push subscription → **1 IAM member** only (app SA)
- DL subscription (`.dl.sub`) → **1 IAM member** only (app SA)

#### Pub/Sub Service Agent per Environment

| Environment | Project Number | Service Agent |
|---|---|---|
| dev | `181368734837` | `service-181368734837@gcp-sa-pubsub.iam.gserviceaccount.com` |
| stg | `117934984866` | `service-117934984866@gcp-sa-pubsub.iam.gserviceaccount.com` |
| prd | `492685761630` | `service-492685761630@gcp-sa-pubsub.iam.gserviceaccount.com` |

#### IAM / Service Accounts
- Validate Workload Identity bindings (KSA ↔ GSA annotation consistency)
- Validate `project_iam_members` references the correct custom role via dependency block

#### IAM Roles — General vs Least Privilege Classification

| Resource | Role Type | Rationale |
|---|---|---|
| Pub/Sub (topics, subscriptions) | Least privilege | Can be scoped to specific resource via `iam_members` |
| Cloud Storage (buckets) | Least privilege | Can be scoped to specific bucket via `iam_members` |
| Secret Manager | Least privilege | Can be scoped to specific secret via `iam_members` |
| Firestore / Datastore | **General privilege** | Cannot be scoped to individual documents/collections |
| Cloud SQL | **General privilege** | Cannot be scoped to individual databases |

> Flag permissions placed in the wrong role type (e.g., `datastore.*` in least privilege role)

#### Cloud Storage
- Validate lifecycle rules, versioning config consistency across environments
- Validate IAM binding uses least privilege role via dependency block

#### Secret Manager
- Validate ExternalSecret references match Secret Manager entries
- **Note**: Secret Manager IAM is managed by the `ClusterSecretStore` operator SA, **not** the application SA. The app SA does NOT need `secretmanager.versions.access` in its least privilege role, and secrets do NOT need `iam_members` for the app SA

### 5. Formatting Validation
- Flag `terragrunt.hcl` files not formatted with `terragrunt hcl fmt`
- Offer to run `terragrunt hcl fmt` automatically as a fix action
- Detect common HCL syntax errors (e.g., missing trailing commas in `iam_members` blocks)

---

## Suggested Outputs

- Cross-environment consistency report (present/missing per env, with diff of divergent fields)
- Naming violations list with corrected suggestions
- Suggested `terragrunt.hcl` for missing environments (auto-generated from existing env, adapting project ID, project number, role suffix, and dependency paths)
- HCL formatting issues detected
- Anti-pattern detection: `iam_members` placement errors, duplicate blocks, hardcoded values, wrong role type classification

---

## Changes to Existing Skills

> **Rationale**: Commit and PR format rules belong in the skills that own those concerns (`commit-generator` and `pr-generator`), not in the `terraform` skill. This avoids duplication and keeps a single source of truth.

### `commit-generator` — Enrich for Terraform/Terragrunt context

**File: `SKILLS/commit-generator/directives/SOP_commit_types.md`**
- Add rule: if the diff contains `.hcl` files from `ebb-iac-resource` or `ebb-iac-iams`, the commit message **must** include `[EPT-XXXX]` at the end (required by CI pipeline `ebb-terragrunt-setup-pipeline.yml`)
- Add examples: `feat(pubsub): create dead letter topic for wp-quotes [EPT-2100]`

**File: `SKILLS/commit-generator/directives/SOP_scope_mapping.md`**
- Add Terraform-specific scope mappings:

| File path pattern | Scope |
|---|---|
| `*/iam/service-accounts/*` | `iam` |
| `*/iam/roles/*` | `iam` |
| `*/pub-sub/topics/*` | `pubsub` |
| `*/pub-sub/subscriptions/*` | `pubsub` |
| `*/cloud-storage/*` | `storage` |
| `*/security/secret-manager/*` | `secrets` |
| `*/cloud-run/*` | `cloudrun` |
| `*/cloud-scheduler/*` | `scheduler` |
| `*/cloud-sql/*` | `cloudsql` |
| `*/project-service/*` | `project` |
| `terragrunt.hcl` (generic) | `terraform` |

### `pr-generator` — Enrich for Terraform/Terragrunt context

**File: `SKILLS/pr-generator/directives/SOP_pr_template.md`**
- Add conditional rule: if PR contains `.hcl` files, the PR title **must** end with `[EPT-XXXX]`
- Add to Author Checklist (conditional on `.hcl` presence):
  - `[ ] Jira card [EPT-XXXX] included in title and commits`

---

## Implementation Steps

1. Create `SKILLS/terraform/SKILL.md` with YAML frontmatter (`name`, `description`, `metadata.model`)
2. Add SOPs to `SKILLS/terraform/directives/`:
   - `SOP_cross_env_consistency.md` — Rules for 3-env validation, project IDs, project numbers, auto-generation adaptations
   - `SOP_naming_conventions.md` — Naming patterns for all resource types, labels validation
   - `SOP_terragrunt_structure.md` — HCL structure, module references, dependency blocks vs hardcoded values, `iam_members` placement and deduplication
   - `SOP_resource_patterns.md` — Resource-specific rules (Pub/Sub, IAM, Storage, Secrets), general vs least privilege classification, Pub/Sub service agent rules per scenario
3. Enrich existing skills with Terraform context:
   - `SKILLS/commit-generator/directives/SOP_commit_types.md` — Add `[EPT-XXXX]` rule for `.hcl` diffs
   - `SKILLS/commit-generator/directives/SOP_scope_mapping.md` — Add Terraform scope mappings
   - `SKILLS/pr-generator/directives/SOP_pr_template.md` — Add Terraform checklist items and title rule
4. Register the `/terraform` slash command in the `ebb-ai-agent` extension (`extensions/ebb-ai-agent/extension.js`)
5. Update the skills table in `.github/prompts/ebb-ai.agent.md`
6. Update `README.md` with the new skill entry

---

## Reference

- Resource repository: `Platform/ebb-iac-resource/` (organized by `ebb-<domain>/<env>/`)
- IAM repository: `Platform/ebb-iac-iams/`
- Terraform modules: `Platform/Modulos Terraform/ebb-terraform-gcp-*/`
- GCP domains: `ebb-ebury-connect`, `ebb-fincore`, `ebb-fincrime`, `ebb-money-flows`, `ebb-enablers`, `ebb-bigdata`, `ebb-platform`, `ebb-shared-services`
- Environments: `dev` (project suffix `-dev`), `stg` (project suffix `-staging`), `prd` (project suffix `-prod`)
