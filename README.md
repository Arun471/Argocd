# Argo CD End-to-End: From Zero to Mastery (with AKS & Terraform)

> A complete, practical guide to learning Argo CD from scratch, deploying it on Azure Kubernetes Service (AKS), automating everything with Terraform, and applying production-grade best practices and security hardening.

---

## Table of Contents

1. [What is Argo CD and Why GitOps](#1-what-is-argo-cd-and-why-gitops)
2. [Core Concepts](#2-core-concepts)
3. [Architecture](#3-architecture)
4. [Prerequisites](#4-prerequisites)
5. [Stage 1 — Beginner: Install Argo CD and Deploy Your First App](#5-stage-1--beginner-install-argo-cd-and-deploy-your-first-app)
6. [Stage 2 — Provisioning AKS with Terraform](#6-stage-2--provisioning-aks-with-terraform)
7. [Stage 3 — Installing Argo CD on AKS via Terraform](#7-stage-3--installing-argo-cd-on-aks-via-terraform)
8. [Stage 4 — Managing Argo CD Declaratively with the Terraform Argo CD Provider](#8-stage-4--managing-argo-cd-declaratively-with-the-terraform-argo-cd-provider)
9. [Stage 5 — Intermediate: App of Apps, ApplicationSets, Sync Waves](#9-stage-5--intermediate-app-of-apps-applicationsets-sync-waves)
10. [Stage 6 — Advanced: Multi-Cluster, Progressive Delivery, Notifications](#10-stage-6--advanced-multi-cluster-progressive-delivery-notifications)
11. [Security Best Practices](#11-security-best-practices)
12. [Azure-Specific Security: Entra ID SSO & Workload Identity](#12-azure-specific-security-entra-id-sso--workload-identity)
13. [Operational Best Practices](#13-operational-best-practices)
14. [Repository Structure Patterns](#14-repository-structure-patterns)
15. [Troubleshooting & Day-2 Operations](#15-troubleshooting--day-2-operations)
16. [Learning Roadmap: Beginner → Master](#16-learning-roadmap-beginner--master)
17. [Reference Links](#17-reference-links)

---

## 1. What is Argo CD and Why GitOps

Argo CD is a declarative, GitOps continuous delivery tool for Kubernetes. Instead of pushing changes to a cluster (like a traditional CI/CD pipeline does), Argo CD **pulls** the desired state from a Git repository and continuously reconciles the live cluster state to match it. Git becomes the single source of truth for what should be running.

Key benefits:
- **Auditability** — every change is a Git commit; you get history, blame, and PR review for free.
- **Consistency** — the cluster can never silently drift from Git without Argo CD flagging it as `OutOfSync`.
- **Rollback** — reverting a Git commit reverts the deployment.
- **Separation of concerns** — CI builds and pushes images; CD (Argo CD) handles deployment, decoupling build from deploy.

Argo CD supports Helm, Kustomize, Jsonnet, and plain YAML as templating/config-management tools out of the box, and can be extended with custom config management plugins.

## 2. Core Concepts

| Concept | Description |
|---|---|
| **Application** | A CRD (`Application`) that maps a Git source (repo + path/chart + revision) to a destination (cluster + namespace). This is the fundamental unit Argo CD manages. |
| **AppProject** | A logical grouping of Applications with restrictions on source repos, destination clusters/namespaces, and allowed resource kinds — the backbone of multi-tenancy. |
| **Sync** | The act of applying the Git-defined manifests to the cluster to make live state match desired state. |
| **Sync Policy** | Automated (`automated`) or manual sync, with options like `prune` (delete removed resources) and `selfHeal` (revert manual cluster changes). |
| **Health Status** | Argo CD's assessment of whether a resource is `Healthy`, `Progressing`, `Degraded`, `Suspended`, or `Missing`. |
| **Sync Status** | `Synced` vs `OutOfSync` — whether live state matches Git. |
| **Sync Waves/Hooks** | Ordering and lifecycle hooks (`PreSync`, `Sync`, `PostSync`, `SyncFail`) for controlling deployment sequencing (e.g., DB migrations before app rollout). |
| **ApplicationSet** | A controller/CRD that generates many `Application` resources from templates + generators (Git directories, clusters, lists, matrix, pull requests, etc.) — the standard pattern for multi-cluster/multi-tenant fleets. |
| **App of Apps** | A pattern where one root `Application` points to a Git path containing other `Application` manifests, bootstrapping an entire environment from a single sync. |

## 3. Architecture

Argo CD is implemented as a set of Kubernetes controllers and services:

- **API Server** — gRPC/REST server used by the UI, CLI, and CI systems; handles auth (local users, SSO/OIDC via Dex or direct), RBAC enforcement, and application management.
- **Repository Server** — clones Git/Helm repos, renders manifests (Helm/Kustomize/Jsonnet/plugins), and caches results. It holds no cluster credentials.
- **Application Controller** — the reconciliation loop; continuously compares live vs. desired state per cluster and triggers sync/health assessment.
- **Dex (optional)** — bundled OIDC connector broker for SSO providers (Entra ID, GitHub, GitLab, LDAP, SAML, etc.). You can also point directly at an OIDC provider without Dex.
- **Redis** — caching layer for the API server and controller.

All inter-component traffic runs over TLS, and in newer Argo CD releases (3.5+) internal mTLS between the repo-server and other components is being introduced for supply-chain hardening.

## 4. Prerequisites

Before you start, have the following ready:

- An Azure subscription with permissions to create AKS, Key Vault, and Managed Identity resources.
- `kubectl`, `helm`, `az` CLI, and `terraform` (≥ 1.6) installed locally.
- The `argocd` CLI ([installation guide](https://argo-cd.readthedocs.io/en/stable/cli_installation/)).
- A Git repository you control, to hold your Kubernetes manifests / Helm charts (this is your GitOps "source of truth" repo).
- Basic familiarity with Kubernetes objects (Deployments, Services, Namespaces) and Terraform HCL syntax.

---

## 5. Stage 1 — Beginner: Install Argo CD and Deploy Your First App

### 5.1 Quick install (any cluster, e.g. Minikube/kind, to learn the basics first)

```bash
kubectl create namespace argocd
kubectl apply -n argocd --server-side --force-conflicts \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

The `--server-side --force-conflicts` flags are required because some Argo CD CRDs (like `ApplicationSet`) exceed the size limit for client-side `kubectl apply`. For production, pin an explicit version tag (e.g. `v3.3.x`) instead of `stable`.

### 5.2 Access the UI/CLI

```bash
# Get the initial admin password
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d && echo

# Port-forward to access locally
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Log in with `argocd login localhost:8080`, then **immediately change the admin password and delete `argocd-initial-admin-secret`** once done — it serves no purpose after rotation.

### 5.3 Deploy your first Application

```bash
argocd app create guestbook \
  --repo https://github.com/argoproj/argocd-example-apps.git \
  --path guestbook \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace default

argocd app sync guestbook
argocd app get guestbook
```

Once comfortable with this manual flow, move to declarative `Application` manifests (checked into Git) instead of CLI-created apps — this is the GitOps-correct approach:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: guestbook
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/argoproj/argocd-example-apps.git
    targetRevision: HEAD
    path: guestbook
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

---

## 6. Stage 2 — Provisioning AKS with Terraform

Now move from a local cluster to a real AKS cluster, provisioned entirely with Terraform (Infrastructure as Code).

```hcl
# versions.tf
terraform {
  required_version = ">= 1.6.0"
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 4.0"
    }
  }
}

provider "azurerm" {
  features {}
}

# main.tf
resource "azurerm_resource_group" "rg" {
  name     = "rg-gitops-aks"
  location = "eastus"
}

resource "azurerm_kubernetes_cluster" "aks" {
  name                = "aks-gitops"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
  dns_prefix          = "aksgitops"
  sku_tier            = "Standard"

  # Required for Workload Identity federation (used later for Key Vault access)
  oidc_issuer_enabled       = true
  workload_identity_enabled = true

  default_node_pool {
    name       = "system"
    node_count = 3
    vm_size    = "Standard_D4s_v5"
    zones      = ["1", "2", "3"]
  }

  identity {
    type = "SystemAssigned"
  }

  key_vault_secrets_provider {
    secret_rotation_enabled = true
  }

  network_profile {
    network_plugin    = "azure"
    network_policy    = "azure"
    load_balancer_sku = "standard"
  }

  azure_active_directory_role_based_access_control {
    tenant_id          = data.azurerm_client_config.current.tenant_id
    azure_rbac_enabled = true
  }
}

data "azurerm_client_config" "current" {}
```

Best practice notes on this step:
- Enable **`oidc_issuer_enabled`** and **`workload_identity_enabled`** from day one — retrofitting Workload Identity later is more painful than provisioning it upfront.
- Use **Azure CNI** (`network_plugin = "azure"`) with `network_policy = "azure"` (or Calico) for pod-level network policy enforcement.
- Enable **Azure RBAC for Kubernetes authorization** so cluster access is governed centrally through Entra ID rather than static `kubeconfig` credentials.
- Spread node pools across **availability zones** for resilience.
- Consider a **private AKS cluster** (`private_cluster_enabled = true`) for production; Argo CD will then need network line-of-sight (VPN/peering/self-hosted runner) to reach the API server.

```bash
terraform init
terraform plan
terraform apply
az aks get-credentials --resource-group rg-gitops-aks --name aks-gitops
```

---

## 7. Stage 3 — Installing Argo CD on AKS via Terraform

Rather than `kubectl apply`-ing manifests by hand, install Argo CD using the Terraform `helm` provider so the installation itself is version-controlled and repeatable.

```hcl
# providers.tf (add to existing)
provider "helm" {
  kubernetes {
    host                   = azurerm_kubernetes_cluster.aks.kube_config[0].host
    client_certificate     = base64decode(azurerm_kubernetes_cluster.aks.kube_config[0].client_certificate)
    client_key             = base64decode(azurerm_kubernetes_cluster.aks.kube_config[0].client_key)
    cluster_ca_certificate = base64decode(azurerm_kubernetes_cluster.aks.kube_config[0].cluster_ca_certificate)
  }
}

resource "kubernetes_namespace" "argocd" {
  metadata {
    name = "argocd"
  }
}

resource "helm_release" "argocd" {
  name       = "argocd"
  namespace  = kubernetes_namespace.argocd.metadata[0].name
  repository = "https://argoproj.github.io/argo-helm"
  chart      = "argo-cd"
  version    = "7.7.x"   # pin an explicit chart version

  values = [
    file("${path.module}/values/argocd-values.yaml")
  ]
}
```

`values/argocd-values.yaml` should turn on TLS, disable insecure mode, and set resource limits — see [Section 11](#11-security-best-practices) for the hardened settings to place here.

> Prefer this Helm-via-Terraform approach (or the official `install.yaml`/Kustomize manifests) over hand-rolling YAML, so upgrades are a version bump + `terraform apply`.

---

## 8. Stage 4 — Managing Argo CD Declaratively with the Terraform Argo CD Provider

Once Argo CD itself is running, use the **`argoproj-labs/argocd`** Terraform provider to manage Argo CD *resources* — Applications, AppProjects, clusters, repositories, RBAC — as Terraform state. Note this provider does **not** install Argo CD; it manages objects inside an already-running instance (chain it after Stage 3 in the same or a downstream Terraform run).

```hcl
terraform {
  required_providers {
    argocd = {
      source  = "argoproj-labs/argocd"
      version = "~> 7.0"
    }
  }
}

provider "argocd" {
  server_addr = "argocd.mycompany.com:443"
  auth_token  = var.argocd_auth_token   # generate via `argocd account generate-token`
}

resource "argocd_project" "platform" {
  metadata {
    name = "platform"
  }
  spec {
    description  = "Platform team applications"
    source_repos = ["https://github.com/my-org/gitops-platform.git"]

    destination {
      server    = "https://kubernetes.default.svc"
      namespace = "*"
    }

    orphaned_resources {
      warn = true
    }
  }
}

resource "argocd_application" "root_app" {
  metadata {
    name      = "root-app"
    namespace = "argocd"
  }
  spec {
    project = argocd_project.platform.metadata[0].name

    source {
      repo_url        = "https://github.com/my-org/gitops-platform.git"
      path            = "apps"
      target_revision = "main"
    }

    destination {
      server    = "https://kubernetes.default.svc"
      namespace = "argocd"
    }

    sync_policy {
      automated {
        prune       = true
        self_heal   = true
      }
      sync_options = ["CreateNamespace=true"]
    }
  }
}
```

This gives you a single Terraform state that spans **infrastructure (AKS) → platform (Argo CD install) → GitOps bootstrap (root Application)** — a fully IaC-driven path from an empty subscription to a running GitOps pipeline. Many teams split this into three Terraform layers/workspaces (infra, platform, bootstrap) so blast radius and apply frequency differ appropriately.

---

## 9. Stage 5 — Intermediate: App of Apps, ApplicationSets, Sync Waves

### 9.1 App of Apps pattern

A root `Application` (as created above) points at a Git folder containing further `Application` manifests — one per microservice/environment. Syncing the root cascades to children, giving you a single command to bootstrap an entire cluster.

### 9.2 ApplicationSets for scale

For many clusters or many similar apps, `ApplicationSet` avoids hand-writing dozens of `Application` YAMLs. A common AKS pattern uses the **Cluster generator** (auto-discovers clusters registered with Argo CD) combined with a **Git directory generator**:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: microservices
  namespace: argocd
spec:
  generators:
    - matrix:
        generators:
          - clusters: {}
          - git:
              repoURL: https://github.com/my-org/gitops-platform.git
              revision: main
              directories:
                - path: apps/*
  template:
    metadata:
      name: '{{path.basename}}-{{name}}'
    spec:
      project: platform
      source:
        repoURL: https://github.com/my-org/gitops-platform.git
        targetRevision: main
        path: '{{path}}'
      destination:
        server: '{{server}}'
        namespace: '{{path.basename}}'
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
```

The **matrix generator** combines every cluster with every app directory, so adding a new cluster or a new app folder automatically fans out to the right combinations — this is the pattern to reach for once you're managing more than a handful of apps or clusters.

### 9.3 Sync waves and hooks

Use `argocd.argoproj.io/sync-wave` annotations to sequence resource application (e.g., Namespaces → CRDs → Secrets → Database migration Job → Deployment), and `PreSync`/`PostSync` hooks for one-off tasks like migrations or smoke tests.

```yaml
metadata:
  annotations:
    argocd.argoproj.io/hook: PreSync
    argocd.argoproj.io/sync-wave: "-1"
```

---

## 10. Stage 6 — Advanced: Multi-Cluster, Progressive Delivery, Notifications

- **Multi-cluster management**: register additional AKS (or other) clusters with `argocd cluster add <context>`, or manage `argocd_cluster` resources via the Terraform provider. Each managed cluster gets a scoped `argocd-manager` ServiceAccount rather than reusing Argo CD's own cluster.
- **Progressive delivery**: pair Argo CD with **Argo Rollouts** for canary/blue-green deployments with automated analysis (Prometheus metrics, AKS-native Azure Monitor queries) gating promotion.
- **Notifications**: the **Argo CD Notifications** controller can post sync/health-state changes to Slack, Teams, email, or webhooks — wire this to Microsoft Teams for AKS platform teams already living there.
- **Observability**: scrape Argo CD's Prometheus metrics (`argocd_app_info`, sync/health gauges) into Azure Monitor Managed Prometheus / Grafana for fleet-wide dashboards.
- **Disaster recovery**: because Git + the `argocd-cm`/`argocd-rbac-cm`/`argocd-secret` objects define nearly all state, DR mainly means backing up these ConfigMaps/Secrets (or, better, managing them via Terraform/Git so they're reproducible) plus registered cluster/repo credentials.

---

## 11. Security Best Practices

Argo CD typically holds broad (often cluster-admin-equivalent) permissions across every cluster it manages — treat its own security posture with the same rigor as the clusters it deploys to.

1. **Disable the local admin account after SSO is configured**
   ```yaml
   # argocd-cm ConfigMap
   admin.enabled: "false"
   ```
2. **Enforce SSO/OIDC (with MFA at the IdP)** instead of local users — see [Section 12](#12-azure-specific-security-entra-id-sso--workload-identity) for Entra ID specifics.
3. **Deny-by-default RBAC.** Set `policy.default: role:readonly` (or an even more restrictive custom role) in `argocd-rbac-cm`, and grant only the minimum permissions each team/CI identity needs, scoped by `AppProject`.
   ```yaml
   policy.csv: |
     p, role:dev-team, applications, sync, dev-project/*, allow
     p, role:dev-team, applications, get, dev-project/*, allow
     g, my-org:dev-team, role:dev-team
   policy.default: role:readonly
   ```
4. **Scope `AppProject`s tightly** — restrict `sourceRepos`, `destinations`, and allowed resource `kinds` (e.g., disallow `ClusterRoleBinding` creation from a namespace-scoped team's project) so a compromised or careless team can't affect other tenants.
5. **Never store plaintext credentials in Git.** Use an external secrets solution — **External Secrets Operator** or the **Secrets Store CSI Driver**, both backed by **Azure Key Vault** — rather than committing Kubernetes `Secret` manifests or Helm `values` with embedded credentials. Sealed Secrets is a viable alternative if you want secrets encrypted-at-rest directly in Git.
6. **Reduce the application-controller's cluster permissions.** The default install grants near cluster-admin; for multi-tenant setups, bind the `argocd-application-controller` service account to a narrower `ClusterRole` and use per-project destination restrictions instead of relying solely on RBAC in the UI/API layer.
7. **Enforce TLS everywhere and pin the minimum version:**
   ```yaml
   # argocd-cmd-params-cm
   server.insecure: "false"
   server.tls.minversion: "1.2"
   ```
8. **Limit repository access.** Grant Argo CD's repo-server credentials read-only, deploy-key-scoped access to only the repos it needs — the repo-server holds no cluster privileges itself, so a leaked Git credential there is lower-blast-radius than a leaked cluster token, but it can still expose source code and pipeline definitions.
9. **Run Argo CD pods with a restricted Pod Security Standard:** `runAsNonRoot: true`, dropped Linux capabilities, `readOnlyRootFilesystem: true`, and label the `argocd` namespace `pod-security.kubernetes.io/enforce: restricted`.
10. **Rotate credentials regularly** — cluster ServiceAccount tokens (especially on Kubernetes 1.24+, where Argo CD creates non-expiring tokens by default), Git deploy keys/PATs, and any local Argo CD API tokens.
11. **Audit and log everything.** Ship Argo CD server/controller/repo-server logs to a central sink (Azure Log Analytics) and alert on repeated failed logins, unexpected `admin` usage, or RBAC-denied events.
12. **Verify supply-chain integrity.** Where available (Argo CD 3.5+), enable Source Integrity / Git commit-signature verification so only signed commits can be synced, and verify Argo CD release artifacts against their published signatures/SBOMs before upgrading.
13. **Network-restrict the API server.** Put Argo CD behind a private ingress/Application Gateway with IP allow-listing or require VPN/Private Link access rather than exposing it on the public internet — especially important when paired with a private AKS cluster.

## 12. Azure-Specific Security: Entra ID SSO & Workload Identity

### 12.1 SSO with Microsoft Entra ID (via Dex or native OIDC)

Register an App Registration in Entra ID, then configure `argocd-cm`:

```yaml
data:
  url: https://argocd.mycompany.com
  oidc.config: |
    name: Azure
    issuer: https://login.microsoftonline.com/<TENANT_ID>/v2.0
    clientID: <APPLICATION_CLIENT_ID>
    clientSecret: $oidc.azure.clientSecret
    requestedScopes: ["openid", "profile", "email"]
    requestedIDTokenClaims: {"groups": {"essential": true}}
```

Map Entra ID groups to Argo CD roles in `argocd-rbac-cm` (`g, <entra-group-object-id>, role:admin`). If you have many group memberships, Argo CD 3.5+ can resolve groups via the Microsoft Graph API to avoid OIDC token size limits.

### 12.2 Workload Identity for secret access (no stored credentials)

Instead of embedding Key Vault credentials in Argo CD or in application manifests, use **Azure Workload Identity Federation**, enabled at the AKS layer in [Section 6](#6-stage-2--provisioning-aks-with-terraform):

```hcl
resource "azurerm_user_assigned_identity" "eso" {
  name                = "id-external-secrets"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
}

resource "azurerm_role_assignment" "eso_kv_reader" {
  scope                = azurerm_key_vault.main.id
  role_definition_name = "Key Vault Secrets User"
  principal_id         = azurerm_user_assigned_identity.eso.principal_id
}

resource "azurerm_federated_identity_credential" "eso" {
  name                = "eso-federated-credential"
  resource_group_name = azurerm_resource_group.rg.name
  parent_id           = azurerm_user_assigned_identity.eso.id
  audience            = ["api://AzureADTokenExchange"]
  issuer              = azurerm_kubernetes_cluster.aks.oidc_issuer_url
  subject             = "system:serviceaccount:external-secrets:eso-sa"
}
```

Then in-cluster, `SecretStore`/`ExternalSecret` (External Secrets Operator) resources reference the federated `ServiceAccount` and pull secrets straight from Key Vault at sync time — Argo CD deploys the `ExternalSecret` CRD declaratively, but never sees or stores the secret value itself.

```yaml
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: azure-keyvault
  namespace: my-app
spec:
  provider:
    azurekv:
      authType: WorkloadIdentity
      vaultUrl: "https://kv-mycompany.vault.azure.net/"
      serviceAccountRef:
        name: eso-sa
```

This pattern — Workload Identity + Key Vault + External Secrets Operator — is the current Microsoft- and community-recommended approach over storing a Service Principal secret or using pod-managed identities.

---

## 13. Operational Best Practices

- **Pin versions everywhere**: Argo CD itself, the Helm chart, and the Terraform providers. Avoid `stable`/`latest` tags in production installs.
- **Automated sync + selfHeal for most apps**, but consider manual sync gates for production-critical or compliance-sensitive namespaces.
- **Use `prune: true` cautiously** — combine with `PreSync` validation/backups for stateful workloads so removed manifests don't unexpectedly delete data-bearing resources.
- **Keep Helm values and Kustomize overlays in Git, never generated ad hoc** — reproducibility is the entire point of GitOps.
- **Separate CI from CD.** CI builds/tests/pushes images and updates a manifest/tag reference in Git (e.g., via Image Updater or a CI step); Argo CD handles the actual cluster apply. Don't give CI pipelines direct `kubectl apply` access to production.
- **Use Argo CD Image Updater** if you want automated tag bumps based on new registry pushes (e.g., from Azure Container Registry), while still routing the change through Git.
- **Set resource requests/limits** on the Argo CD components themselves — under-provisioned `argocd-repo-server` pods are a very common source of slow/failed syncs at scale.
- **Enable High Availability manifests** in production (`manifests/ha/install.yaml` or the `argo-cd` Helm chart's HA values) — the default single-replica install is not resilient to node loss.
- **Test upgrades in a lower environment first**, and read the specific `vX.Y to vX.Z` upgrade notes — Argo CD has shipped several RBAC and metrics behavior changes across major versions (e.g., 2.x → 3.0).

## 14. Repository Structure Patterns

Two common approaches:

**Mono-repo (single GitOps repo):**
```
gitops-platform/
├── apps/
│   ├── team-a/service-x/
│   └── team-b/service-y/
├── infra/
│   └── argocd-appset.yaml
└── projects/
    └── platform-project.yaml
```

**Multi-repo (app source repo separate from GitOps config repo):**
- `app-source-repo` — application code + CI pipeline that builds images and bumps a tag in...
- `gitops-config-repo` — Kubernetes manifests/Helm values only, watched by Argo CD.

Multi-repo gives cleaner separation of duties (app devs vs. platform/security review of deploy config) and is generally preferred once you have more than a few teams; mono-repo is simpler to start with for small platforms.

## 15. Troubleshooting & Day-2 Operations

| Symptom | Likely cause / fix |
|---|---|
| App stuck `Progressing` | Check `argocd app get <app> --show-operation`; inspect the target resource's own status/events with `kubectl describe`. |
| App `OutOfSync` right after sync | Another controller (e.g., HPA, cert-manager) is mutating fields Argo CD compares — configure `ignoreDifferences` for that field path. |
| Repo-server slow/timeouts | Usually resource starvation or a very large monorepo; increase `repo-server` replicas/resources and enable manifest caching. |
| RBAC "permission denied" for a valid user | Check group claim mapping (`scopes` field) and confirm the OIDC token actually contains the expected `groups` claim (Entra ID group overflow is a common culprit for large orgs). |
| Cluster registration fails for AKS | Confirm the `argocd-manager` ServiceAccount/ClusterRoleBinding exists on the target cluster, and that network policy/firewall rules allow Argo CD to reach the AKS API server (especially for private clusters). |

---

## 16. Learning Roadmap: Beginner → Master

1. **Beginner** — Install Argo CD on a local cluster, deploy the sample `guestbook` app via CLI, then convert it to a declarative `Application` manifest. Understand Sync vs. Health status.
2. **Novice → Intermediate** — Provision a real AKS cluster with Terraform; install Argo CD via the Terraform `helm` provider; connect a private Git repo with a deploy key; build your first App of Apps.
3. **Intermediate** — Adopt `ApplicationSet` generators (Git, cluster, matrix) for multi-app/multi-cluster fleets; implement sync waves/hooks for ordered rollouts; set up RBAC with SSO groups.
4. **Advanced** — Integrate External Secrets Operator + Azure Key Vault via Workload Identity; harden security per [Section 11](#11-security-best-practices); add Argo CD Notifications and Prometheus/Grafana monitoring.
5. **Expert / Master** — Run Argo CD HA in production; manage the full stack (AKS + Argo CD install + Applications) as layered Terraform state; implement progressive delivery with Argo Rollouts; contribute to or extend Argo CD with custom Config Management Plugins or resource health checks; participate in the upstream project's release/upgrade cadence to stay ahead of breaking changes.

---

## 17. Reference Links

**Argo CD official**
- Docs home: https://argo-cd.readthedocs.io/
- Getting Started: https://argo-cd.readthedocs.io/en/stable/getting_started/
- Installation guide: https://argo-cd.readthedocs.io/en/stable/operator-manual/installation/
- RBAC configuration: https://argo-cd.readthedocs.io/en/stable/operator-manual/rbac/
- Security overview: https://argo-cd.readthedocs.io/en/stable/operator-manual/security/
- ApplicationSet docs: https://argo-cd.readthedocs.io/en/stable/operator-manual/applicationset/
- GitHub repo / releases: https://github.com/argoproj/argo-cd

**Terraform providers**
- Argo CD Terraform provider: https://registry.terraform.io/providers/argoproj-labs/argocd/latest/docs
- AzureRM `azurerm_kubernetes_cluster`: https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/kubernetes_cluster
- Helm provider: https://registry.terraform.io/providers/hashicorp/helm/latest/docs

**Azure / AKS**
- AKS documentation: https://learn.microsoft.com/en-us/azure/aks/
- Workload Identity on AKS: https://learn.microsoft.com/en-us/azure/aks/workload-identity-overview
- Secrets Store CSI Driver for AKS: https://learn.microsoft.com/en-us/azure/aks/csi-secrets-store-driver

**Secrets management**
- External Secrets Operator: https://external-secrets.io/
- External Secrets — Azure Key Vault provider: https://external-secrets.io/latest/provider/azure-key-vault/

**Ecosystem**
- Argo Rollouts (progressive delivery): https://argo-rollouts.readthedocs.io/
- Argo CD Notifications: https://argocd-notifications.readthedocs.io/
- Argo CD Helm chart (argo-helm): https://github.com/argoproj/argo-helm

---

*This guide reflects Argo CD 3.x behavior as of mid-2026. Always check the official docs' upgrade notes before moving between major versions, since RBAC, metrics, and default security behaviors have changed across releases.*
