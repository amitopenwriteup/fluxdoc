# 4.1 Working with GitRepository and Kustomization

## Overview

This section covers the two most fundamental FluxCD resources: `GitRepository` and `Kustomization`. Together they form the core of any GitOps workflow — `GitRepository` tells Flux where to fetch manifests from, and `Kustomization` tells Flux what to do with them once fetched.

---

## Table of Contents

- [Creating GitRepository Resources](#creating-gitrepository-resources)
- [Defining Kustomization Resources](#defining-kustomization-resources)
- [Health Checks and Dependencies](#health-checks-and-dependencies)
- [Managing Multi-Environment Deployments](#managing-multi-environment-deployments)
- [GitLab Repository Structure for FluxCD](#gitlab-repository-structure-for-fluxcd)
- [Sync Intervals and Automation](#sync-intervals-and-automation)

---

## Creating GitRepository Resources

A `GitRepository` resource instructs the Flux `source-controller` to clone a Git repository at a defined interval and make it available as an artifact for other Flux resources to consume.

### Basic GitRepository — Public Repository

```yaml
# gitrepository-public.yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: my-app
  namespace: flux-system
spec:
  interval: 1m
  url: https://gitlab.com/<username>/manifest-repo
  ref:
    branch: main
```

### GitRepository — Private Repository with Token Auth

First create the Kubernetes secret:

```bash
kubectl create secret generic gitlab-auth \
  --namespace=flux-system \
  --from-literal=username=<deploy-token-username> \
  --from-literal=password=<deploy-token-or-pat>
```

Then reference it in the `GitRepository`:

```yaml
# gitrepository-private.yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: my-app
  namespace: flux-system
spec:
  interval: 1m
  url: https://gitlab.com/<username>/manifest-repo
  secretRef:
    name: gitlab-auth
  ref:
    branch: main
```

### GitRepository — Pin to a Specific Tag or Commit

```yaml
spec:
  ref:
    # Track a specific tag
    tag: v1.2.0

    # OR pin to an exact commit SHA
    commit: 4e7a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a

    # OR track a semver range
    semver: ">=1.0.0 <2.0.0"
```

### GitRepository — Ignore Specific Paths

Use `.sourceignore` or inline `ignore` to skip files that should not trigger reconciliation:

```yaml
spec:
  interval: 1m
  url: https://gitlab.com/<username>/manifest-repo
  ref:
    branch: main
  ignore: |
    # Ignore docs and CI files — changes here won't trigger a sync
    /docs/
    /.gitlab-ci.yml
    /README.md
    *.png
```

### Apply and Verify a GitRepository

```bash
# Apply the resource
kubectl apply -f gitrepository-private.yaml

# Check status
flux get source git -A

# Watch until READY=True
flux get source git my-app -n flux-system --watch

# Describe for detailed events
kubectl describe gitrepository my-app -n flux-system
```

Expected healthy output:

```
NAME     REVISION          SUSPENDED  READY  MESSAGE
my-app   main@sha1:4e7a1b  False      True   stored artifact for revision 'main@sha1:4e7a1b'
```

### GitRepository Field Reference

| Field | Description | Example |
|---|---|---|
| `interval` | How often Flux polls the repository | `1m`, `5m`, `30s` |
| `url` | HTTPS or SSH URL of the repository | `https://gitlab.com/user/repo` |
| `ref.branch` | Branch to track | `main` |
| `ref.tag` | Exact tag to pin | `v1.0.0` |
| `ref.semver` | Semver range to track | `>=1.0.0 <2.0.0` |
| `ref.commit` | Exact commit SHA to pin | `4e7a1b2c...` |
| `secretRef.name` | Kubernetes secret with Git credentials | `gitlab-auth` |
| `ignore` | Glob patterns to exclude from artifact | `/docs/` |
| `timeout` | Timeout for Git operations | `60s` |

---

## Defining Kustomization Resources

A `Kustomization` resource instructs the Flux `kustomize-controller` to build and apply Kubernetes manifests from a path within a `GitRepository` artifact. It is the bridge between source and cluster state.

### Basic Kustomization

```yaml
# kustomization-basic.yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: my-app
  namespace: flux-system
spec:
  interval: 5m
  sourceRef:
    kind: GitRepository
    name: my-app
  path: ./apps/overlays/production
  prune: true
  wait: true
  timeout: 2m
```

### Kustomization with Target Namespace

```yaml
spec:
  interval: 5m
  sourceRef:
    kind: GitRepository
    name: my-app
  path: ./apps/base
  prune: true
  targetNamespace: production        # Override namespace for all resources
```

### Kustomization with Post-Build Variable Substitution

Inject runtime values into manifests without modifying source files:

```yaml
spec:
  interval: 5m
  sourceRef:
    kind: GitRepository
    name: my-app
  path: ./apps/overlays/production
  prune: true
  postBuild:
    substitute:
      cluster_name: "prod-cluster-eu"
      region: "eu-west-1"
    substituteFrom:
      - kind: ConfigMap
        name: cluster-vars
      - kind: Secret
        name: cluster-secrets
```

In your manifest files, reference variables with `${VAR_NAME}`:

```yaml
# apps/overlays/production/deployment.yaml
env:
  - name: CLUSTER_NAME
    value: "${cluster_name}"
  - name: REGION
    value: "${region}"
```

### Kustomization — Force Apply

Force recreation of resources that cannot be patched (e.g., immutable fields):

```yaml
spec:
  force: true       # Deletes and recreates resources if a patch fails
```

> **WARN:** Use `force: true` with caution in production. It can cause brief downtime when resources are recreated.

### Apply and Verify a Kustomization

```bash
# Apply the resource
kubectl apply -f kustomization-basic.yaml

# Check status
flux get kustomization -A

# Watch until READY=True
flux get kustomization my-app -n flux-system --watch

# Force an immediate reconciliation
flux reconcile kustomization my-app --with-source -n flux-system

# View detailed events
kubectl describe kustomization my-app -n flux-system
```

Expected healthy output:

```
NAME    REVISION          SUSPENDED  READY  MESSAGE
my-app  main@sha1:4e7a1b  False      True   Applied revision: main@sha1:4e7a1b
```

### Kustomization Field Reference

| Field | Description | Default |
|---|---|---|
| `interval` | How often to reconcile | Required |
| `sourceRef` | Reference to a `GitRepository` or `Bucket` | Required |
| `path` | Path within the source artifact | `./` |
| `prune` | Delete cluster resources removed from Git | `false` |
| `wait` | Wait for all applied resources to become ready | `false` |
| `timeout` | Timeout for apply and health check operations | `5m` |
| `targetNamespace` | Override namespace for all applied resources | — |
| `force` | Recreate resources that cannot be patched | `false` |
| `suspend` | Pause reconciliation without deleting resources | `false` |
| `postBuild.substitute` | Inline variable substitution map | — |
| `postBuild.substituteFrom` | Variable substitution from ConfigMap or Secret | — |

---

## Health Checks and Dependencies

### Built-in Health Checks with `wait`

When `wait: true` is set, Flux monitors all applied resources and only marks the `Kustomization` as `READY` once every resource reports healthy. This prevents dependent resources from applying before their dependencies are up.

```yaml
spec:
  interval: 5m
  sourceRef:
    kind: GitRepository
    name: my-app
  path: ./apps/overlays/production
  prune: true
  wait: true            # Wait for Deployments, StatefulSets, DaemonSets to be ready
  timeout: 5m           # Fail if not healthy within 5 minutes
```

### Custom Health Checks

Define explicit health checks for resources that Flux does not natively understand (e.g., CRDs):

```yaml
spec:
  interval: 5m
  sourceRef:
    kind: GitRepository
    name: my-app
  path: ./apps/overlays/production
  prune: true
  healthChecks:
    - apiVersion: apps/v1
      kind: Deployment
      name: my-app
      namespace: production
    - apiVersion: v1
      kind: StatefulSet
      name: my-db
      namespace: production
    - apiVersion: batch/v1
      kind: Job
      name: db-migration
      namespace: production
```

### Dependencies Between Kustomizations

Use `dependsOn` to enforce ordering. Flux will not apply a `Kustomization` until all listed dependencies are `READY`.

```yaml
# infrastructure/kustomization.yaml — applied first
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: infrastructure
  namespace: flux-system
spec:
  interval: 10m
  sourceRef:
    kind: GitRepository
    name: my-app
  path: ./infrastructure
  prune: true
  wait: true
---
# apps/kustomization.yaml — waits for infrastructure
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: apps
  namespace: flux-system
spec:
  interval: 5m
  sourceRef:
    kind: GitRepository
    name: my-app
  path: ./apps/overlays/production
  prune: true
  dependsOn:
    - name: infrastructure          # Will not apply until infrastructure is READY
```

### Multi-layer Dependency Chain

```yaml
# Layer 1: CRDs
metadata:
  name: crds
# No dependsOn — applied first

---
# Layer 2: Infrastructure (cert-manager, ingress, etc.)
metadata:
  name: infrastructure
spec:
  dependsOn:
    - name: crds

---
# Layer 3: Databases and stateful services
metadata:
  name: databases
spec:
  dependsOn:
    - name: infrastructure

---
# Layer 4: Applications
metadata:
  name: apps
spec:
  dependsOn:
    - name: databases
    - name: infrastructure
```

### Check Health Status

```bash
# Overall health of all kustomizations
flux get kustomization -A

# Detailed health events for a specific kustomization
kubectl describe kustomization apps -n flux-system

# Check which resources Flux is health-checking
flux get kustomization apps -n flux-system --verbose
```

---

## Managing Multi-Environment Deployments

### Pattern: One GitRepository, Multiple Kustomizations

A single `GitRepository` source can feed multiple `Kustomization` resources, each pointing to a different overlay path. This is the recommended pattern for multi-environment GitOps.

```yaml
# source — shared across all environments
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: my-app
  namespace: flux-system
spec:
  interval: 1m
  url: https://gitlab.com/<username>/manifest-repo
  secretRef:
    name: gitlab-auth
  ref:
    branch: main
---
# dev environment
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: my-app-dev
  namespace: flux-system
spec:
  interval: 1m
  sourceRef:
    kind: GitRepository
    name: my-app
  path: ./apps/overlays/dev
  prune: true
  targetNamespace: dev
---
# staging environment
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: my-app-staging
  namespace: flux-system
spec:
  interval: 5m
  sourceRef:
    kind: GitRepository
    name: my-app
  path: ./apps/overlays/staging
  prune: true
  targetNamespace: staging
---
# production environment
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: my-app-production
  namespace: flux-system
spec:
  interval: 10m
  sourceRef:
    kind: GitRepository
    name: my-app
  path: ./apps/overlays/production
  prune: true
  wait: true
  timeout: 5m
  targetNamespace: production
```

### Verify Multi-Environment Status

```bash
# See all environments at a glance
flux get kustomization -A

# Sample output
# NAMESPACE    NAME                 REVISION          READY  MESSAGE
# flux-system  my-app-dev           main@sha1:4e7a1b  True   Applied revision: main@sha1:4e7a1b
# flux-system  my-app-staging       main@sha1:4e7a1b  True   Applied revision: main@sha1:4e7a1b
# flux-system  my-app-production    main@sha1:4e7a1b  True   Applied revision: main@sha1:4e7a1b

# Reconcile a specific environment immediately
flux reconcile kustomization my-app-staging --with-source -n flux-system

# Suspend production auto-sync during a maintenance window
flux suspend kustomization my-app-production -n flux-system

# Resume after maintenance
flux resume kustomization my-app-production -n flux-system
```

---

## GitLab Repository Structure for FluxCD

### Recommended Repository Layout

The standard GitLab + FluxCD repository structure separates Flux internals (`clusters/`) from application manifests (`apps/`) and shared infrastructure (`infrastructure/`):

```
manifest-repo/
├── clusters/
│   ├── production/
│   │   └── flux-system/              ← Flux bootstrap files (auto-generated)
│   │       ├── gotk-components.yaml
│   │       ├── gotk-sync.yaml
│   │       └── kustomization.yaml
│   └── staging/
│       └── flux-system/
├── infrastructure/
│   ├── base/
│   │   ├── cert-manager/
│   │   ├── ingress-nginx/
│   │   └── kustomization.yaml
│   └── overlays/
│       ├── production/
│       └── staging/
└── apps/
    ├── base/
    │   ├── deployment.yaml
    │   ├── service.yaml
    │   ├── configmap.yaml
    │   └── kustomization.yaml
    └── overlays/
        ├── dev/
        │   ├── kustomization.yaml
        │   └── patch-replicas.yaml
        ├── staging/
        │   ├── kustomization.yaml
        │   ├── patch-replicas.yaml
        │   └── patch-resources.yaml
        └── production/
            ├── kustomization.yaml
            ├── patch-replicas.yaml
            ├── patch-resources.yaml
            └── patch-hpa.yaml
```

### GitLab Branch Strategy for FluxCD

| Branch | Purpose | Flux Watches? |
|---|---|---|
| `main` | Production-ready manifests | ✔ Production Kustomization |
| `staging` | Staging-validated manifests | ✔ Staging Kustomization |
| `dev` | Development manifests | ✔ Dev Kustomization |
| `feature/*` | Work in progress | ✗ Not watched by Flux |

### Multi-Branch GitRepository Setup

```yaml
# Production — watches main branch
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: manifest-repo-production
  namespace: flux-system
spec:
  interval: 1m
  url: https://gitlab.com/<username>/manifest-repo
  secretRef:
    name: gitlab-auth
  ref:
    branch: main
---
# Staging — watches staging branch
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: manifest-repo-staging
  namespace: flux-system
spec:
  interval: 1m
  url: https://gitlab.com/<username>/manifest-repo
  secretRef:
    name: gitlab-auth
  ref:
    branch: staging
```

### GitLab Deploy Token Setup for Flux

```bash
# Create a deploy token in GitLab:
# manifest-repo > Settings > Repository > Deploy Tokens
# Scopes: read_repository

# Create the Kubernetes secret
kubectl create secret generic gitlab-auth \
  --namespace=flux-system \
  --from-literal=username=<deploy-token-username> \
  --from-literal=password=<deploy-token-password>

# Verify the secret exists
kubectl get secret gitlab-auth -n flux-system
```

---

## Sync Intervals and Automation

### Choosing the Right Interval

Sync interval controls how frequently Flux polls the GitRepository and reconciles the cluster. Choosing the right value balances responsiveness against API server load.

| Environment | Recommended Interval | Reasoning |
|---|---|---|
| Dev | `30s` – `1m` | Fast feedback during active development |
| Staging | `1m` – `5m` | Moderate — balances speed and stability |
| Production | `5m` – `10m` | Conservative — reduces reconciliation churn |
| Infrastructure | `10m` – `30m` | Rarely changes, low urgency |

### Setting Different Intervals per Layer

```yaml
# Source fetched frequently
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: my-app
spec:
  interval: 1m           # Poll GitLab every 1 minute

---
# Infrastructure reconciled infrequently
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: infrastructure
spec:
  interval: 30m          # Reconcile every 30 minutes
  sourceRef:
    kind: GitRepository
    name: my-app

---
# Apps reconciled more frequently
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: apps
spec:
  interval: 5m           # Reconcile every 5 minutes
  sourceRef:
    kind: GitRepository
    name: my-app
```

> **NOTE:** The `GitRepository` interval and the `Kustomization` interval are independent. Flux fetches the source on the `GitRepository` schedule and reconciles on the `Kustomization` schedule. Set the source interval shorter than or equal to the shortest kustomization interval.

### Force Immediate Sync (Bypass Interval)

```bash
# Sync source only (fetch latest commits from GitLab)
flux reconcile source git my-app -n flux-system

# Sync a specific kustomization only
flux reconcile kustomization my-app-production -n flux-system

# Sync source AND all kustomizations that depend on it
flux reconcile kustomization my-app-production --with-source -n flux-system
```

### GitLab Webhook — Trigger Flux on Push (Instant Sync)

Instead of polling, configure GitLab to notify Flux immediately on every push. This effectively makes the sync interval irrelevant for push-triggered reconciliation.

**Step 1 — Create the Flux webhook receiver:**

```yaml
# receiver.yaml
apiVersion: notification.toolkit.fluxcd.io/v1
kind: Receiver
metadata:
  name: gitlab-receiver
  namespace: flux-system
spec:
  type: gitlab
  events:
    - Push Hook
  secretRef:
    name: webhook-token
  resources:
    - apiVersion: source.toolkit.fluxcd.io/v1
      kind: GitRepository
      name: my-app
```

**Step 2 — Create the webhook secret:**

```bash
# Generate a random token
TOKEN=$(head -c 12 /dev/urandom | shasum | cut -d ' ' -f1)
echo $TOKEN

kubectl create secret generic webhook-token \
  --namespace=flux-system \
  --from-literal=token=$TOKEN
```

**Step 3 — Get the receiver URL:**

```bash
kubectl get receiver gitlab-receiver -n flux-system -o jsonpath='{.status.webhookPath}'
# Output: /hook/sha256sum-of-token
```

**Step 4 — Register in GitLab:**

Go to `manifest-repo > Settings > Webhooks` and add:
- **URL:** `https://<your-flux-ingress>/hook/<path-from-step-3>`
- **Secret token:** the token from Step 2
- **Trigger:** Push events

### Suspend and Resume Automation

```bash
# Suspend a kustomization (stops reconciliation, keeps existing resources)
flux suspend kustomization my-app-production -n flux-system

# Suspend the source (stops fetching from GitLab)
flux suspend source git my-app -n flux-system

# Resume reconciliation
flux resume kustomization my-app-production -n flux-system
flux resume source git my-app -n flux-system

# Check suspended status
flux get kustomization -A
# SUSPENDED column shows True/False
```

### Export Resource Definitions

```bash
# Export GitRepository as YAML
flux export source git my-app -n flux-system

# Export all Kustomizations
flux export kustomization --all -n flux-system

# Export everything and save to file
flux export source git --all -n flux-system > gitrepositories.yaml
flux export kustomization --all -n flux-system > kustomizations.yaml
```

---

## Quick Reference

### GitRepository CLI Commands

```bash
flux get source git -A                              # List all GitRepositories
flux get source git <name> -n flux-system --watch   # Watch a specific source
flux reconcile source git <name> -n flux-system     # Force fetch from GitLab
flux suspend source git <name> -n flux-system       # Pause source polling
flux resume source git <name> -n flux-system        # Resume source polling
flux export source git <name> -n flux-system        # Export as YAML
```

### Kustomization CLI Commands

```bash
flux get kustomization -A                                        # List all Kustomizations
flux get kustomization <name> -n flux-system --watch             # Watch a specific kustomization
flux reconcile kustomization <name> -n flux-system               # Force reconcile
flux reconcile kustomization <name> --with-source -n flux-system # Force source + reconcile
flux suspend kustomization <name> -n flux-system                 # Pause reconciliation
flux resume kustomization <name> -n flux-system                  # Resume reconciliation
flux export kustomization <name> -n flux-system                  # Export as YAML
flux logs --kind=Kustomization --name=<name>                     # View controller logs
```

### Common Status Messages

| Message | Meaning |
|---|---|
| `Applied revision: main@sha1:abc123` | Reconciliation succeeded |
| `stored artifact for revision 'main@sha1:abc123'` | Source fetched successfully |
| `health check failed` | A resource did not become ready within `timeout` |
| `dependency not ready` | A `dependsOn` target is not yet `READY` |
| `ReconciliationSucceeded` | Full cycle completed without errors |
| `kustomize build failed` | YAML error in manifests — check `flux logs --level=error` |

---

## Key Documentation Links

- **GitRepository API reference:** https://fluxcd.io/flux/components/source/gitrepositories/
- **Kustomization API reference:** https://fluxcd.io/flux/components/kustomize/kustomizations/
- **Multi-tenancy guide:** https://fluxcd.io/flux/guides/repository-structure/
- **Webhook receivers:** https://fluxcd.io/flux/guides/webhook-receivers/
- **Variable substitution:** https://fluxcd.io/flux/components/kustomize/kustomizations/#post-build-variable-substitution

---

*Section 4.1 · Advanced GitOps Workshop · FluxCD + GitLab*
