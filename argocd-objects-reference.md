# Argo CD Objects: A Practical Reference

Argo CD models everything as Kubernetes objects — some are true Custom Resources (CRDs), others are plain Secrets or ConfigMaps that Argo CD treats specially based on labels or well-known names. This doc covers every one you'll actually run into, with purpose, key fields, and a working example for each.

## At a Glance

| Object | Type | apiVersion | Purpose |
|---|---|---|---|
| `Application` | CRD | `argoproj.io/v1alpha1` | Defines one deployable app: source → destination |
| `AppProject` | CRD | `argoproj.io/v1alpha1` | Security boundary / multi-tenancy grouping of Applications |
| `ApplicationSet` | CRD | `argoproj.io/v1alpha1` | Template that auto-generates many Applications |
| Repository | Secret | `v1` | Stores a Git/Helm/OCI repo connection |
| Repo Credential Template | Secret | `v1` | Shared credentials for a group of repo URLs |
| Cluster | Secret | `v1` | Stores a target cluster's connection details |
| `argocd-cm` | ConfigMap | `v1` | Core server settings |
| `argocd-rbac-cm` | ConfigMap | `v1` | RBAC policy rules |
| `argocd-secret` | Secret | `v1` | Admin password, signing keys, SSO secrets |
| `argocd-cmd-params-cm` | ConfigMap | `v1` | Component command-line flags |
| `argocd-notifications-cm` | ConfigMap | `v1` | Notification triggers/templates/services |
| `argocd-ssh-known-hosts-cm` | ConfigMap | `v1` | SSH known_hosts for Git over SSH |
| `argocd-tls-certs-cm` | ConfigMap | `v1` | TLS certs for self-hosted Git servers |

---

## 1. Application

The central object. One `Application` = one unit of "this source, deployed to this destination, kept in this state."

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
    # For Helm charts instead of plain manifests:
    # helm:
    #   valueFiles:
    #     - values-prod.yaml
    # For Kustomize:
    # kustomize:
    #   namePrefix: prod-

  destination:
    server: https://kubernetes.default.svc   # or use `name: in-cluster`
    namespace: guestbook

  syncPolicy:
    automated:
      prune: true       # delete resources removed from Git
      selfHeal: true     # revert manual cluster drift back to Git state
      allowEmpty: false
    syncOptions:
      - CreateNamespace=true
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m

  ignoreDifferences:
    - group: apps
      kind: Deployment
      jsonPointers:
        - /spec/replicas   # ignore HPA-managed field drift

  revisionHistoryLimit: 10
```

**Key `spec` fields:**

| Field | Purpose |
|---|---|
| `project` | Which `AppProject` governs this app's permissions |
| `source` / `sources` | Where manifests come from (Git, Helm repo, OCI). `sources` (plural) enables multi-source apps |
| `destination` | Target cluster (`server` or `name`) and `namespace` |
| `syncPolicy.automated` | Enables GitOps auto-sync (`prune`, `selfHeal`, `allowEmpty`) |
| `syncPolicy.syncOptions` | Flags like `CreateNamespace=true`, `Validate=false`, `ApplyOutOfSyncOnly=true` |
| `ignoreDifferences` | Fields Argo CD should not consider "out of sync" |
| `revisionHistoryLimit` | How many old sync revisions to keep for rollback |

**Key `status` fields (read-only, set by Argo CD):**

| Field | Meaning |
|---|---|
| `status.sync.status` | `Synced` / `OutOfSync` |
| `status.health.status` | `Healthy` / `Progressing` / `Degraded` / `Missing` / `Suspended` |
| `status.resources` | Per-resource sync/health breakdown |
| `status.history` | Past deployed revisions (used for rollback) |
| `status.operationState` | Details of the currently running or last sync operation |
| `status.conditions` | Errors/warnings (e.g., `ComparisonError`) |

---

## 2. AppProject

A logical boundary — think of it as a "tenant" or "team" container that restricts what its Applications are allowed to do.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: my-team
  namespace: argocd
spec:
  description: Applications owned by the payments team

  sourceRepos:
    - https://github.com/my-org/payments-*

  destinations:
    - server: https://kubernetes.default.svc
      namespace: payments-*

  clusterResourceWhitelist:
    - group: ""
      kind: Namespace

  namespaceResourceBlacklist:
    - group: ""
      kind: ResourceQuota

  roles:
    - name: developer
      description: Can sync but not delete
      policies:
        - p, proj:my-team:developer, applications, sync, my-team/*, allow
        - p, proj:my-team:developer, applications, delete, my-team/*, deny
      groups:
        - my-org:payments-team

  syncWindows:
    - kind: allow
      schedule: "0 9 * * MON-FRI"
      duration: 8h
      applications:
        - "*"
```

**Key fields:**

| Field | Purpose |
|---|---|
| `sourceRepos` | Whitelist of Git/Helm repo URLs Applications in this project may pull from |
| `destinations` | Whitelist of cluster + namespace combos Applications may deploy to |
| `clusterResourceWhitelist` / `Blacklist` | Which cluster-scoped kinds (e.g., `ClusterRole`) are allowed |
| `namespaceResourceWhitelist` / `Blacklist` | Which namespaced kinds are allowed/blocked |
| `roles` | Project-scoped RBAC roles with their own policies and SSO group bindings |
| `syncWindows` | Time windows that allow or deny syncing (e.g., no deploys on weekends) |
| `orphanedResources` | Detect resources in the namespace not tracked by any Application |

---

## 3. ApplicationSet

Generates and manages many `Application` objects from a single template — the standard way to do "one Application per cluster" or "one Application per Git folder" without copy-pasting YAML.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: guestbook-per-cluster
  namespace: argocd
spec:
  generators:
    - clusters: {}          # one entry per registered Cluster secret
    # Other common generators:
    # - list:
    #     elements:
    #       - cluster: dev
    #       - cluster: prod
    # - git:
    #     repoURL: https://github.com/my-org/apps.git
    #     revision: HEAD
    #     directories:
    #       - path: apps/*
    # - matrix:              # combine two generators (e.g., clusters x directories)
    #     generators: [...]

  template:
    metadata:
      name: 'guestbook-{{name}}'
    spec:
      project: default
      source:
        repoURL: https://github.com/argoproj/argocd-example-apps.git
        targetRevision: HEAD
        path: guestbook
      destination:
        server: '{{server}}'
        namespace: guestbook

  strategy:
    type: RollingSync         # progressive delivery across generated Applications
    rollingSync:
      steps:
        - matchExpressions:
            - key: envLabel
              operator: In
              values: [dev]
        - matchExpressions:
            - key: envLabel
              operator: In
              values: [prod]

  syncPolicy:
    preserveResourcesOnDeletion: false
```

**Common generators:**

| Generator | Use case |
|---|---|
| `list` | Hardcoded set of values (simplest case) |
| `clusters` | One Application per registered cluster Secret |
| `git` (directories/files) | One Application per folder or file match in a repo |
| `matrix` | Cartesian product of two generators (e.g., clusters × environments) |
| `merge` | Combine generators, overriding overlapping keys |
| `scmProvider` | One Application per repo in a GitHub/GitLab org |
| `pullRequest` | One Application per open PR (preview environments) |
| `clusterDecisionResource` | Delegate cluster selection to a custom controller (e.g., duck-typed CR) |
| `plugin` | Call an external HTTP service to generate parameters |

---

## 4. Repository (Secret)

Git/Helm/OCI repositories aren't a CRD — they're stored as ordinary Secrets labeled so Argo CD recognizes them.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: private-repo
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: repository
stringData:
  type: git
  url: https://github.com/my-org/private-repo.git
  username: my-user
  password: my-token
  # For SSH instead:
  # sshPrivateKey: |
  #   -----BEGIN OPENSSH PRIVATE KEY-----
  #   ...
  # To scope this repo to one project only:
  # project: my-team
```

## 5. Repository Credential Template (Secret)

Applies credentials to *all* repos matching a URL prefix, instead of repeating them per repo.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: github-org-creds
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: repo-creds
stringData:
  url: https://github.com/my-org
  username: my-user
  password: my-token
```

## 6. Cluster (Secret)

Registers an external Kubernetes cluster as a deployment target.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: prod-cluster
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: cluster
stringData:
  name: prod
  server: https://prod-cluster-api.example.com
  config: |
    {
      "bearerToken": "...",
      "tlsClientConfig": {
        "insecure": false,
        "caData": "..."
      }
    }
```

---

## 7. Core ConfigMaps & Secrets (Global Settings)

These aren't per-app; they configure the Argo CD installation itself.

| Name | Type | Contains |
|---|---|---|
| `argocd-cm` | ConfigMap | Server URL, SSO (`oidc.config`/`dex.config`), resource health/action customizations, resource exclusions, `application.instanceLabelKey` |
| `argocd-rbac-cm` | ConfigMap | `policy.csv` (who can do what), `policy.default` (fallback role), `scopes` (which SSO claims map to groups) |
| `argocd-secret` | Secret | Admin password hash, server JWT signing key, SSO client secrets, webhook secrets |
| `argocd-cmd-params-cm` | ConfigMap | CLI flags for `argocd-server`, `repo-server`, `application-controller` (e.g., enabling `--insecure`, setting timeouts) |
| `argocd-ssh-known-hosts-cm` | ConfigMap | SSH `known_hosts` entries for connecting to Git over SSH |
| `argocd-tls-certs-cm` | ConfigMap | Custom TLS certs for self-hosted Git servers with private CAs |

Example `argocd-rbac-cm` policy snippet:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-rbac-cm
  namespace: argocd
data:
  policy.csv: |
    p, role:readonly, applications, get, */*, allow
    p, role:ci, applications, sync, my-team/*, allow
    g, my-org:platform-team, role:admin
  policy.default: role:readonly
```

---

## 8. Notifications (via `argocd-notifications-cm`)

Notifications aren't a CRD either — triggers, templates, and services live as keys inside a ConfigMap, and Applications/AppProjects "subscribe" to them via annotations.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-notifications-cm
  namespace: argocd
data:
  service.slack: |
    token: $slack-token
  template.app-deployed: |
    message: Application {{.app.metadata.name}} is now running new version.
  trigger.on-deployed: |
    - when: app.status.operationState.phase in ['Succeeded']
      send: [app-deployed]
```

Subscribing an Application to a trigger:

```yaml
metadata:
  annotations:
    notifications.argoproj.io/subscribe.on-deployed.slack: my-channel
```

---

## 9. Project Roles (part of `AppProject`)

Not a separate object, but worth calling out: `spec.roles` inside `AppProject` is Argo CD's project-scoped RBAC layer, distinct from the global `argocd-rbac-cm`. Each role has its own `policies` list and can be bound to SSO `groups` or issued static `jwtTokens` for CI pipelines.

---

## Quick Decision Guide

| I want to... | Use |
|---|---|
| Deploy one app from Git to one cluster | `Application` |
| Give a team its own sandbox with guardrails | `AppProject` |
| Deploy the same app to 20 clusters, or one app per PR | `ApplicationSet` |
| Connect a private Git/Helm repo | Repository Secret |
| Share credentials across many repos under one org | Repo Credential Template Secret |
| Register a new target cluster | Cluster Secret |
| Change global UI/SSO/health-check behavior | `argocd-cm` |
| Control who can do what across the whole instance | `argocd-rbac-cm` |
| Get Slack/email alerts on sync or health events | `argocd-notifications-cm` |
