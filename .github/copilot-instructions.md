# GitOps and GCP Infrastructure Validation Rules

## ⚠️ RULE: Mandatory Delegation to Specialized Agents

There are 3 specialized agents in this workspace. **Always delegate to the appropriate agent** using `runSubagent` in the situations listed below. Do not attempt to execute the task directly — the agent has optimized context and tools for the use case.

### GitOps Audit Agent
**Invoke ALWAYS when the user asks (explicitly or implicitly) for any of these actions:**
- Audit an application (e.g., "audit wp-quotes", "check wp-pre-validation")
- Validate cross-environment consistency (dev/stg/prd) of Terraform resources
- Verify IAM bindings, roles, service accounts of an app
- Compare GitOps overlays between environments
- Diagnose issues in topics, subscriptions, buckets, secrets, or ExternalSecrets
- Validate KSA ↔ GSA ↔ Workload Identity bindings
- Check if a Terraform resource exists in all 3 environments
- Any mention of "auditoria", "audit", "consistência", "consistency", "cross-env", "validar infra", "validate infra"

### Jira Agent
**Invoke ALWAYS when the user asks for any of these actions:**
- Fetch details of a card/issue (e.g., "fetch EPT-2134", "what's in EPT-2100?")
- Create issues, stories, tasks or bugs in Jira
- Add comments to issues
- Search issues by filter, sprint, assignee, or text
- List backlog or current sprint
- Any mention of "Jira", "EPT-", "card", "issue", "story", "bug", "task", "sprint", "backlog"

### Confluence Agent
**Invoke ALWAYS when the user asks for any of these actions:**
- Fetch or read documentation pages in Confluence
- Create or update pages in Confluence
- Search content in Confluence spaces (e.g., EP space)
- Navigate page structure (children, parent)
- Any mention of "Confluence", "documentação", "documentation", "wiki", "página", "page", "espaço EP", "EP space"

> **General rule**: If the user's message contains keywords from an agent (even without explicitly naming the agent), delegate to it. When in doubt, ask the user if they want to use the specialized agent.

---

## Workspace Structure

The root workspace (`/home/osorio/Documentos/github/`) contains all repositories organized as follows:

### Teams (Folders with Capital Letters)
Folders that start with capital letters represent **teams/squads**. Inside each team are the application folders:

| Team | Folder | Terraform Domain | Note |
|------|--------|------------------|------|
| Webpayments | `Webpayments/` | `ebb-ebury-connect` | Received part of digitalFx |
| FinCore (ex-FX) | `FX/` | `ebb-fincore` | Renamed from FX Engine + FX Tree |
| FinCrime (ex-Client Journey) | `Client Journey/` | `ebb-fincrime` | Renamed from Client Journey |
| Money Flows (ex-Banking) | `Money-Flows/` | `ebb-money-flows` | Renamed from Banking |
| Enablers | — | `ebb-enablers` | New domain: partial digitalFx + Ebury Now portal |

### Internal Structure of Each Application (within the team)
Each application inside the team folder follows the **3 subfolders** pattern:

```
<Team>/<app-name>/
├── ebb-<app-name>/              # Application source code
├── ebb-<app-name>-gitops/       # Manifests read by ArgoCD and ArgoWorkflow
│   ├── application/             # ArgoCD Application Kustomization
│   ├── base/                    # Base manifests (Deployment, Service, etc.)
│   ├── overlays/                # Overlays per environment
│   │   ├── ebb-dev/
│   │   ├── ebb-stg/
│   │   └── ebb-prd/
│   └── pipeline_config.yaml     # CI/CD pipeline configuration (ArgoWorkflow)
└── ebb-<app-name>-tests/        # Automated tests (used by ArgoWorkflow)
```

**Real example**: `Webpayments/wp-pre-validation/` contains:
- `ebb-wp-pre-validation/` (Go, source code)
- `ebb-wp-pre-validation-gitops/` (K8s manifests + pipeline config)
- `ebb-wp-pre-validation-tests/` (BDD tests with Behave/Python)

> **Note**: There are still some loose folders at the workspace root that are being reorganized to follow this team-based folder pattern.

> **Overlay exception**: Some legacy repos (e.g., `bexs-dollar`) use different overlay names (`staging`, `production`) instead of the standard `ebb-dev/ebb-stg/ebb-prd`. The pipeline's `commit-deploy` step supports both patterns.

### `pipeline_config.yaml` (CI/CD Configuration)
File present in the gitops repository that defines pipeline parameters:
```yaml
pipeline:
  team:
    name: <NomeDoTime>
    slack: <canal-slack>
  notification:
    slack: true
  cd:
    ebb_cluster: true
    type: k8s          # ou cloudrun
  ci:
    type: golang       # ou nodejs
    main_path: cmd/app/main.go
    test:
      framework: cucumber   # ou cypress, playwright
      service_account: <nome-da-ksa>  # opcional — KSA usada pelo pod de testes (default: "default")
      cypress_version: 18.16.0  # opcional — versão da imagem cypress/base (default: 18.16.0)
      environment:              # opcional — env vars injetadas no container de testes
        CYPRESS_BASE_URL: "https://<app>-dev.bexs.com.br"
    node_version: "20.10.0"  # optional — Node.js version (default: 18.16.0)
    go_version: "1.26.3"     # optional — Go version (default: 1.26.3)
```

> **Field `test.service_account`**: when defined, `wft_pipeline_selector_step_github` exports the value as global parameter `test-service-account`, propagated via `podSpecPatch` to the `golang-automated-tests` step pod. Allows a repository to use Workload Identity in the test pod without modifying shared templates. The referenced KSA must exist in the `argo` namespace of the tools cluster (`ebb-iac-bexs-platform`). Implemented via EPT-2134.

> **Field `test.environment.CYPRESS_BASE_URL`**: when defined, `wft_pipeline_selector_step_github` reads the value via `parse()` and exports it as global parameter `cypress-base-url`. This value is propagated through the chain `nodejs-startup-pipeline → automated-tests-selector → nodejs-automated-tests`, where it's injected as **env var `CYPRESS_BASE_URL`** in the Cypress test container. Repos without this field are not affected (env var remains empty). Implemented via EPT-2461.

---

### Platform Repositories (`Platform/`)

#### `Platform/ebb-iac-resource` — GCP Terraform Resources
Main **infrastructure as code** repository for creating resources in GCP projects.
- Organized by domain and environment: `ebb-<domain>/<env>/`
- Environments: `dev`, `stg`, `prd`
- Resource types:
  - `iam/service-accounts/` — Google Service Accounts
  - `iam/roles/` — Custom IAM Roles (general e least privilege)
  - `pub-sub/topics/` e `pub-sub/subscriptions/` — Pub/Sub
  - `cloud-storage/` — Buckets GCS
  - `cloud-sql/` — Cloud SQL
  - `cloud-run/` — Cloud Run services
  - `cloud-scheduler/` — Cloud Scheduler jobs
  - `security/secret-manager/` — Secret Manager
  - `security/key-management/` — KMS
  - `firestore/` — Firestore indexes/rules
  - `artifact-registry/` — Artifact Registry
  - `compute-engine/` — Compute Engine
  - `kubernetes-engine/` — GKE configs
  - `project-service/` — APIs enabled in the project
- Each resource has a `terragrunt.hcl` that references a Terraform module
- Existing domains: `ebb-ebury-connect`, `ebb-ebury-connect-pci`, `ebb-fincore`, `ebb-fincrime`, `ebb-money-flows`, `ebb-enablers`, `ebb-bigdata`, `ebb-platform`, `ebb-shared-services`
- Legacy domains (under migration): `ebb-banking`, `ebb-client-journey`, `ebb-fx-engine`

#### `Platform/ebb-iac-iams` — Group IAM Permissions
Repository to manage **user group permissions** in GCP projects.
- Used to grant permissions to Google Workspace groups (developers, tech leads, support, etc.)
- Structure: `ebb-<domain>/<env>/iam/roles/<RoleName>/terragrunt.hcl`
- Role examples: `EBB_Ebury_Connect_Developer_Dev`, `EBB_Network_Admin_Dev`, `EBB_Security_Engineer_Dev`, `Ebb_Tech_Support_Dev`
- Each role defines a `google_project_iam_custom_role` with hundreds of GCP permissions and binds it via `google_project_iam_member` to an email group (e.g., `group:ebb.eburyconnect.tech.engineers@ebury.com`)
- State backend in GCS: bucket `ebb-iac-iam-tfstate-<env>`
- Also has additional domains: `ebb-organization`, `ebb-security`, `ebb-backup-infra`
- Has Python helpers in `helpers/`:
  - `deploy_terragrunt.py` — Deploy orchestrator with dependency resolution (DAG), change detection via git diff, topological sort
  - `guardrail_labels_terragrunt.py` — Validation of mandatory labels on GCP resources
  - `check_deleted_directories.py` — Checks deleted directories for state cleanup

#### `Platform/Modulos Terraform` — Reusable Terraform Modules
Repositories of **Terraform modules** used as sources in resource `terragrunt.hcl` files.
- Each module is an independent repository with `main.tf`, `variables.tf`, `outputs.tf`
- Versioned via git tags (e.g., `?ref=v1.3.1`)
- Available modules:
  - `ebb-terraform-gcp-pubsub` — Topics, subscriptions, schemas (includes dead letter, retry, push config)
  - `ebb-terraform-gcp-service-account` — Service accounts + IAM bindings + project IAM members
  - `ebb-terraform-gcp-cloud-storage` — Buckets + transfer jobs (CORS, lifecycle, versioning)
  - `ebb-terraform-gcp-iam` — Custom roles + IAM members (used by ebb-iac-iams)
  - `ebb-terraform-gcp-cloud-functions` — Cloud Functions
  - `ebb-terraform-gcp-cloud-run` — Cloud Run services
  - `ebb-terraform-gcp-cloud-dns` — Cloud DNS
  - `ebb-terraform-gcp-scheduler` — Cloud Scheduler
  - `ebb-terraform-gcp-compute-engine` — Compute Engine
  - `ebb-terraform-gcp-kubernetes` — GKE
  - `ebb-terraform-gcp-compliance-scc` — Security Command Center
  - `ebb-terraform-gcp-cloud-build-github-connector` — Cloud Build GitHub connector

#### `Platform/platform-cicd` — CI/CD Workflows (Argo Workflow)
Repository with **CI/CD pipeline workflows** based on Argo Workflow + Argo Events.
- **Sensor** (`sensor.yaml`): Listens to Bitbucket/GitHub webhooks via Pub/Sub, filters PRs merged to `master`/`main`/`develop`, and creates Workflows
- **Workflow Templates** (`manifests/workflows/`):
  - `wft_base_startup.yaml` — Entry point that detects type (golang/nodejs) and directs pipeline
  - `wft_golang_startup_pipeline.yaml` / `wft_nodejs_startup_pipeline.yaml` — Pipelines per language
  - Available steps: build+test, image build, commit deploy, automated tests, approval, deployment, scan (Semgrep, Snyk, Trivy), notifications (Slack)
  - `_github.yaml` variants for GitHub repositories (vs Bitbucket)
- **CD** (`manifests/cd/`): ArgoCD ApplicationSet template for deployment to staging/sandbox/production
- **Standard Service** (`manifests/standard-service/`): Base + overlays for standard service configuration
- **Configs** (`conf/`): Secrets and credentials for ArgoCD, Bitbucket, Snyk, NPM, storage

**⚠️ IMPORTANT — Templates and Sensor are shared by ALL repositories:**
- The **Sensor `pubsub-sensor-github`** is unique and triggers workflows for any GitHub repository that merges a PR. It's not repository-specific.
- The **Workflow Templates** (`wft_*`) are used by all teams. Changes to them affect the entire organization.
- Workflow pods run with the **`default`** SA in the `argo` namespace by default — without GCP credentials.
- To customize the SA of the test pod for a specific repository **without modifying shared templates**, use `podSpecPatch` with the `service-account` parameter read from `pipeline_config.yaml` (field `ci.test.service_account`). This is the correct pattern — implemented via EPT-2134.
- **Never add GCP credentials directly to shared templates** — use the opt-in pattern via `pipeline_config.yaml`.

**⚠️ IMPORTANT — `yq` version in `mikefarah/yq` container:**
- The container uses `yq` v4 but **does not support advanced subcommands** like `yq e -o=shell`, `yq e -o=p`, nor complex expressions like `keys | .[]`.
- **Always use the `parse()` pattern**: `yq "expression // \"default\"" file.yaml`
- When needing to read a new field from `pipeline_config.yaml`, use the existing `parse()` function in the script (e.g., `parse .pipeline.ci.test.environment.CYPRESS_BASE_URL ""`).

**Node.js pipeline parameter chain (GitHub):**
```
pipeline-selector-github
  → (output params: globalName) → propagated via workflow.outputs.parameters
    → nodejs-startup-pipeline-github
      → (arguments) → automated-tests-selector-step-github
        → (arguments) → nodejs-automated-tests-step-github (container)
```
Parameters propagated in this chain: `tags`, `repo`, `test-dependencies-commands`, `cypress-base-url`, `node-version`, `cypress-version`.

#### `Platform/ebb-platform-argocd` — ArgoCD and Flux Central Repository
**Main** repository for ArgoCD management and cluster infrastructure via Flux CD.

**Arquitetura de Bootstrap (Flux → ArgoCD):**
```
GKE Cluster
  └── Flux bootstrap → clusters/gke_ebb-platform-<env>/flux-system/
        ├── gotk-components.yaml (Flux v2.7.2 CRDs + controllers)
        ├── gotk-sync.yaml (GitRepository + Kustomizations)
        └── Flux reconcilia:
              ├── resources/argocd-install.yaml → instala ArgoCD via HelmRelease
              └── ArgoCD assume a gestão das aplicações
                    ├── appset-bootstrap.yaml → Application "Root App" (bootstrap/)
                    └── metadata-bootstrap.yaml → Application (AppProjects via _metadata/)
```

**Estrutura principal:**
```
ebb-platform-argocd/
├── argocd-install/                    # Manifests de instalação do ArgoCD (Flux CRDs)
│   ├── dev/                           # HelmRelease, HelmRepository, Gateway, VS, Service, Namespace
│   ├── stg/
│   └── prd/
├── bootstrap/                         # ApplicationSets (um por ferramenta/serviço)
│   ├── external-secrets.yaml          # Todos os domínios × 3 envs
│   ├── monitoring.yaml                # Todos os domínios × 3 envs
│   ├── nginx-external.yaml            # client-journey, platform, money-flows
│   ├── nginx-internal.yaml            # client-journey, bigdata, platform, money-flows
│   ├── istio-gateway-external.yaml    # ebury-connect, client-journey, platform
│   ├── istio-gateway-internal.yaml    # ebury-connect, client-journey, platform
│   ├── velero.yaml                    # client-journey, platform, money-flows, ebury-connect
│   ├── kong.yaml                      # ebury-connect
│   ├── cert-manager.yaml              # platform (stg/prd), client-journey
│   ├── arc.yaml / arc-runners.yaml    # platform (GitHub Actions runners)
│   ├── keda.yaml                      # platform (dev only)
│   └── sloth.yaml                     # platform
├── clusters/                          # Configuração Flux por cluster GKE
│   ├── gke_ebb-platform-dev/
│   │   ├── flux-system/               # gotk-components.yaml, gotk-sync.yaml, kustomization.yaml
│   │   └── resources/
│   │       └── argocd-install.yaml    # Flux Kustomization → argocd-install/dev/
│   ├── gke_ebb-platform-stg/          # (mesma estrutura)
│   └── gke_ebb-platform-prd/          # (mesma estrutura)
├── ebb-{bigdata,client-journey,ebury-connect,money-flows,platform}/  # Configs por domínio
│   └── {dev,staging,prod}/
│       ├── _metadata/app-project.yaml # AppProject do ArgoCD
│       └── <service>/                 # Valores Helm + k8s-manifests por ferramenta
│           ├── kustomization.yaml     # Usa helmCharts: (Kustomize nativo, NÃO Flux HelmRelease)
│           ├── values.yaml
│           └── k8s-manifests/
├── appset-bootstrap.yaml              # Root App: aplica tudo de bootstrap/
└── metadata-bootstrap.yaml            # Aplica **/_metadata/*.yaml recursivamente
```

**ArgoCD Installation (via Flux):**
Each `argocd-install/<env>/` contains:

| File | Type | Purpose |
|------|------|---------|
| `argocd-helmrelease-minimal.yaml` | `HelmRelease` (Flux) | Installs `argo-cd` chart v9.3.1 in the `argocd` namespace |
| `argocd-helmrepository.yaml` | `HelmRepository` (Flux) | Source: `argoproj.github.io/argo-helm` |
| `argocd-gw-vs.yaml` | Istio Gateway/VS | Internal ingress (dev/stg: own Gateway, prd: shared gateway) |
| `argocd-svc.yaml` | Service | ClusterIP for argocd-server |
| `argocd-oauth-external-secret.yaml` | ExternalSecret | GitHub OAuth via GCP Secret Manager |
| `namespace.yaml` | Namespace | `argocd` (istio-injection: disabled) |
| `kustomization.yaml` | Kustomize | Groups the above files |
| `repo-secret.yaml` | Secret | SSH key to access repos (commented in kustomization) |

**Differences per environment:**

| Config | DEV | STG | PRD |
|--------|-----|-----|-----|
| URL | `argocd-dev-int.ebury.com.br` | `argocd-stg-int.ebury.com.br` | `argocd.ebury.com.br` |
| server replicas | 1 | 1 | 3 (anti-affinity) |
| repoServer replicas | 1 | 1 | 3 (anti-affinity) |
| applicationSet replicas | 1 | 1 | 2 (anti-affinity) |
| Istio Gateway | Próprio | Próprio | Shared (`internal-shared-gateway`) |

**Cluster IPs per domain:**

| Domain | DEV | STG | PRD |
|---------|-----|-----|-----|
| ebb-platform | `10.7.255.2` | `10.107.255.2` | `10.207.255.2` |
| ebb-client-journey | `10.27.255.2` | `10.127.255.2` | `10.227.255.2` |
| ebb-ebury-connect | `10.11.255.2` | `10.111.255.2` | `10.211.255.2` |
| ebb-money-flows | `10.23.255.2` | `10.123.255.2` | `10.223.255.2` |
| ebb-bigdata | `10.39.255.2` | `10.139.255.2` | `10.239.255.2` |

**Flux resources no `gotk-sync.yaml`:**

| Resource | Kind | Propósito |
|----------|------|-----------|
| `flux-system` | `GitRepository` | Points to `ebb-platform-argocd.git` (branch main) |
| `flux-system` | `Kustomization` | Applies `./clusters/gke_ebb-platform-<env>` (interval: 10m) |
| `platform-resources` | `Kustomization` | Applies `./clusters/gke_ebb-platform-<env>/resources` (interval: 5m, dependsOn: flux-system) |

**ApplicationSet pattern in `bootstrap/`:**
Each file uses a **List generator** with hardcoded elements:
```yaml
generators:
  - list:
      elements:
        - project: ebb-ebury-connect-dev
          env: dev
          path: ebb-ebury-connect/dev/<service>
          cluster_ip: "10.11.255.2"
```
All point to `repoURL: git@github.com:Ebury-Brazil/ebb-platform-argocd.git`, `targetRevision: HEAD`.

**Service pattern in domains (`<domain>/<env>/<service>/`):**
Uses `helmCharts:` in `kustomization.yaml` (native Kustomize, **not** Flux HelmRelease):
```yaml
helmCharts:
  - name: external-secrets
    repo: https://charts.external-secrets.io
    version: 1.3.1
    releaseName: external-secrets
    namespace: external-secrets-system
    valuesFile: values.yaml
resources:
  - k8s-manifests/cluster-secret-store.yaml
```

#### `Platform/ebb-argocd-manifests-repo` — ⚠️ LEGACY (migrating to ebb-platform-argocd)
> **Status**: This repository is being discontinued. ArgoCD installation manifests have been migrated to `ebb-platform-argocd/argocd-install/`. Migration is being done per environment (DEV first, then STG, then PRD). Ref: EPT-2068.

Legacy ArgoCD manifests repository, with HelmRelease per environment.
- Original structure: `ebb-argocd-{dev,stg,prd}/` — same 8 files now in `ebb-platform-argocd/argocd-install/`
- Contains RBAC configuration (policy.csv) with access control per AppProject
- AppProjects, ApplicationSets, and Apps are in `appprojects/`, `applicationsets/`, `apps/` (templates/examples)
- CI/CD: `.github/workflows/create-app.yml` — automatic app scaffolding via GitHub Issue
- Scripts: `scripts/new-app.sh` — gitops structure scaffolding (base + overlays)

**ArgoCD RBAC Rules** (configured in `argocd-helmrelease-minimal.yaml` for each env):

| GitHub Team | ArgoCD Role | AppProject |
|-------------|-------------|------------|
| EBB Platform Team | admin | (full access) |
| EBB FinCrime | fincrime-dev | ebb-fincrime |
| EBB FinCore | fincore-dev | ebb-fincore |
| EBB Money Flows | money-flows-dev | ebb-money-flows |
| EBB Enablers | enablers-dev | ebb-enablers |
| EBB Ebury Connect Webpayments | ebury-connect-dev | ebb-ebury-connect |
| EBB Ebury Connect Pay | ebury-connect-pci-dev | ebb-ebury-connect-pci |
| EBB Data | data-dev | ebb-data |
| EBB FrontEnd-Shared | frontend-shared | permissions on accessed apps |
| EBB-BackEnd-Shared | backend-shared | permissions on accessed apps |

**Permissions per environment (apps with suffix -dev, -stg, -prd):**
- `-dev` / `-stg`: get, logs, sync, override, action/*
- `-prd`: only get + logs (read-only)

**Domain renaming (2026):**
- Client Journey → **FinCrime**
- FX (Engine + Tree) → **FinCore**
- Banking → **Money Flows**
- **Enablers** = new domain (partial digitalFx + Ebury Now portal)
- **Ebury Connect** received another part of digitalFx

---

## ⚠️ FUNDAMENTAL RULE: Cross-Environment Consistency

**All Terraform resources (topics, subscriptions, buckets, secrets, roles, service accounts) must exist equivalently in all 3 environments (dev, stg, prd).**

### Principle
If a resource exists in one environment, the equivalent must exist in the other two. This includes:
- The resource itself (topic, subscription, bucket, etc.)
- The `iam_members` block with equivalent permissions
- The corresponding `dependency` blocks (role and service account for the environment)
- Associated dead letter topics and subscriptions

### When creating or modifying a resource
1. **Always check all 3 environments** (dev, stg, prd) for the same resource
2. If missing in any environment, create the equivalent by adapting:
   - GCP project name (`ebb-ebury-connect-dev`, `ebb-ebury-connect-staging`, `ebb-ebury-connect-prod`)
   - Role suffix (`_dev`, `_stg`, `_prd` — or variants like `_dev2`, `_stg_v2`, `_prd_v2`)
   - Pub/Sub service agent project number (for `gcp-sa-pubsub`)
3. If an `iam_member` exists in one environment, the equivalent must exist in the others

> **Cross-environment audits**: Use the **GitOps Audit Agent** for detailed validations.

---

## ⚠️ IMPORTANT: Git Workflow

**ALWAYS before performing validations or adding/modifying content:**

1. **Check current branch**:
```bash
git branch --show-current
```

2. **Switch to main/master branch** (if not already):
```bash
git checkout main
```

3. **Update repository**:
```bash
git pull
```

4. **Check status** to ensure it's clean:
```bash
git status
```

**Rule**: Never perform validations or modifications without first ensuring you're on the main branch with updated code.

---

## ⚠️ IMPORTANT: Commit and PR Convention (Terragrunt)

**The Terragrunt pipeline (`ebb-terragrunt-setup-pipeline.yml`) automatically validates the Jira card number in every PR before the plan.** If missing, the PR will be blocked.

### Mandatory format
```
type(scope): description [EPT-XXXX]
```

- `type`: `feat`, `fix`, `chore`, `refactor`, `docs`, `ci`, `test`, `perf`, `style`
- `scope` (optional): change context (e.g., `iam`, `pubsub`, `storage`)
- `[EPT-XXXX]`: Jira card number, **mandatory** at the end in brackets

### Where to apply
- **PR title** → `feat: add IAM bindings for payment topics [EPT-2063]`
- **Commit message** → `fix: resolve auth token refresh [EPT-2063]`

### Examples
```
feat: add IAM bindings for payment topics [EPT-2063]
fix: resolve auth token refresh [EPT-2063]
chore: update labels configuration [EPT-2063]
feat(pubsub): create dead letter topic for wp-quotes [EPT-2100]
refactor(iam): consolidate least privilege roles [EPT-2150]
```

### Rules
- Always add `[EPT-XXXX]` at the end of the message in brackets
- The pipeline will block the PR if the card number is missing from the PR title or the last commit
- When generating commits or PR titles for Terraform/Terragrunt repositories, **always include the Jira card** — ask the user if not provided
- **Always use `gh` CLI to create PRs**: `gh pr create --title "..." --body "..."`
- **Always write PR titles and descriptions in English** — this is mandatory for all repositories

---

## ⚠️ IMPORTANT: Terragrunt Formatting

**ALWAYS after editing `terragrunt.hcl` files**, run the formatter:

```bash
terragrunt hcl fmt
```

**Rules**:
- Run `terragrunt hcl fmt` at the root of the `ebb-iac-resource` (or `ebb-iac-iams`) repository after any `.hcl` edit
- If the command reports syntax errors (e.g., `Missing item separator`), fix the errors before committing
- **Common error cause**: last entry of an `iam_members` block (or similar) without a trailing comma before adding a new entry
- The command is idempotent — can be run multiple times without side effects
- **NEVER commit `.hcl` files without formatting first**

> **Note**: Newer Terragrunt versions use `terragrunt hcl fmt` (with space), not `terragrunt hclfmt`.

---

#### `Platform/ebb-iac-bexs-platform` — Cluster Tools (Argo Workflow + Legacy ArgoCD)
Repository managed by **Flux CD** that maintains the **tools cluster** (`gke_tools`) state, where Argo Workflow, Argo Events, and legacy ArgoCD (under migration) run.

- Main branch: **`master`** (not `main`)
- Cluster GCP project: **`bexs-platform`**
- Flux reconciles resources in `gke/gke_tools/flux-system/kustomization.yaml`, which references manifests in `gke/gke_tools/resources/`
- To add a new Kubernetes resource to the tools cluster (e.g., ServiceAccount, Role, ConfigMap), create the YAML file in `gke/gke_tools/resources/` and reference it in `gke/gke_tools/flux-system/kustomization.yaml`

**KSAs for Argo Workflow test pods:**
- Workflow pods run in the **`argo`** namespace of the tools cluster
- Dedicated KSAs for tests with Workload Identity must be created in this repository in `gke/gke_tools/resources/serviceaccount-<name>.yaml`
- The corresponding GCP SA is in the **`bexs-platform`** project (not in domain projects `ebb-ebury-connect-*`)
- WI binding: `serviceAccount:bexs-platform.svc.id.goog[argo/<ksa-name>]`

**Example KSA for tests:**
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: <nome-ksa>
  namespace: argo
  annotations:
    iam.gke.io/gcp-service-account: <nome-sa>@bexs-platform.iam.gserviceaccount.com
```

---

## Quick Path Reference
- Applications: `<Team>/<app-name>/ebb-<app-name>/` (source code)
- GitOps: `<Team>/<app-name>/ebb-<app-name>-gitops/` (Kubernetes manifests + ArgoCD + ArgoWorkflow)
- Tests: `<Team>/<app-name>/ebb-<app-name>-tests/` (automated tests)
- Terraform Resources: `Platform/ebb-iac-resource/ebb-<domain>/<env>/` (GCP infrastructure)
- Group IAMs: `Platform/ebb-iac-iams/ebb-<domain>/<env>/iam/roles/` (group permissions)
- Terraform Modules: `Platform/Modulos Terraform/ebb-terraform-gcp-<resource>/` (reusable modules)
- CI/CD Workflows: `Platform/platform-cicd/` (Argo Workflow templates and sensors)
- Cluster Tools (Argo + CI): `Platform/ebb-iac-bexs-platform/gke/gke_tools/` (tools cluster resources via Flux, branch **master**)
- ArgoCD + Flux: `Platform/ebb-platform-argocd/` (ArgoCD installation, bootstrap ApplicationSets, configs per domain)
- ArgoCD Install: `Platform/ebb-platform-argocd/argocd-install/<env>/` (HelmRelease + HelmRepository + Istio)
- Flux Clusters: `Platform/ebb-platform-argocd/clusters/gke_ebb-platform-<env>/` (gotk-sync, argocd-install ref)
- Legacy ArgoCD: `Platform/ebb-argocd-manifests-repo/` (⚠️ migrating → ebb-platform-argocd)

> **GitOps/Terraform Audits**: Use the **GitOps Audit Agent** to validate overlays, IAM, PubSub, External Secrets, and cross-environment consistency.

## Naming Patterns

### Environments
- `dev` or `ebb-dev`: Development
- `stg` or `ebb-stg`: Staging
- `prd` or `ebb-prd`: Production

### GCP Projects and Project Numbers

| Environment | Project ID | Project Number | Pub/Sub Service Agent |
|----------|-----------|----------------|------------------------|
| dev | `ebb-ebury-connect-dev` | `181368734837` | `service-181368734837@gcp-sa-pubsub.iam.gserviceaccount.com` |
| stg | `ebb-ebury-connect-staging` | `117934984866` | `service-117934984866@gcp-sa-pubsub.iam.gserviceaccount.com` |
| prd | `ebb-ebury-connect-prod` | `492685761630` | `service-492685761630@gcp-sa-pubsub.iam.gserviceaccount.com` |

> The Pub/Sub service agent is used in DL topics with `roles/pubsub.publisher` and in subscriptions with DL using `roles/pubsub.subscriber`.

### Service Accounts

**⚠️ RULE: All Service Accounts (GCP and Kubernetes) must have the `ebb-` prefix.**

- GCP (application): `ebb-<app-name>@ebb-ebury-connect-<env>.iam.gserviceaccount.com`
- GCP (CI tests): `ebb-<app-name>-test@bexs-platform.iam.gserviceaccount.com`
- K8s (application): `ebb-<app-name>`
- K8s (CI tests): `ebb-<app-name>-test` (namespace `argo` in tools cluster)

**Correct examples:**
- `ebb-wp-portal-api@ebb-ebury-connect-dev.iam.gserviceaccount.com` (application SA)
- `ebb-wp-hedges-test@bexs-platform.iam.gserviceaccount.com` (test SA)
- `ebb-wp-uploads-test` (test KSA in tools cluster)

**Incorrect examples:**
- ~~`wp-portal-api-test@bexs-platform.iam.gserviceaccount.com`~~ (missing `ebb-` prefix)
- ~~`hedges-test@bexs-platform.iam.gserviceaccount.com`~~ (missing `ebb-` and `wp-` prefixes)

> For Webpayments team test SAs, the pattern is `ebb-wp-<name>-test` (keeps the team's `wp-` prefix).

### Roles
- General: `ebb_<app_name>_general_privilege_role_<env>`
- Least: `ebb_<app_name>_least_privilege_role_<env>`
- Least (CI tests): `ebb_<app_name>_test_least_privilege_role_<env>`

### GCP Resources
- Buckets: `ebb-<name>-<env>`
- Topics: `ebb-<name>-topic`
- Subscriptions: `ebb-<name>-sub`
- Secrets: `ebb-<app-name>`
