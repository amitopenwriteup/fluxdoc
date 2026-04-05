# FluxCD Helm Lab
### Deploy nginx via Bitnami Helm Chart
### Repo: flux-gitops | Cluster: clusters/production

---

## Your Repo Structure (Already Bootstrapped)

```
flux-gitops/
└── clusters/
    └── production/
        └── flux-system/          ← already exists (bootstrap files)
            ├── gotk-components.yaml
            ├── gotk-sync.yaml
            └── kustomization.yaml
```

Flux is already running. You only need to **add your app manifests** and push to Git.

---

## Step 1 — Create App Folder in Your Repo

```bash
git clone https://github.com/<YOUR_USER>/flux-gitops.git
cd flux-gitops

mkdir -p clusters/production/flux-helm-lab
```

Your structure will become:

```
flux-gitops/
└── clusters/
    └── production/
        ├── flux-system/          ← existing (do not touch)
        └── flux-helm-lab/        ← new (your helm app lives here)
            ├── 01-helmrepository.yaml
            ├── 02-values-configmap.yaml
            ├── 03-helmrelease.yaml
            └── 04-kustomization.yaml
```

---

## Step 2 — Create Manifests

### clusters/production/flux-helm-lab/01-helmrepository.yaml

```yaml
apiVersion: source.toolkit.fluxcd.io/v1beta2
kind: HelmRepository
metadata:
  name: helmrepo
  namespace: flux-system
spec:
  type: default
  url: https://charts.bitnami.com/bitnami
  interval: 10m
```

### clusters/production/flux-helm-lab/02-values-configmap.yaml

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-helm-values
  namespace: flux-system
data:
  values.yaml: |
    replicaCount: 1
    image:
      registry: docker.io
      repository: bitnami/nginx
      tag: ""
    service:
      type: ClusterIP
      port: 80
    ingress:
      enabled: false
    resources:
      requests:
        cpu: 100m
        memory: 128Mi
      limits:
        cpu: 250m
        memory: 256Mi
    autoscaling:
      enabled: false
```

### clusters/production/flux-helm-lab/03-helmrelease.yaml

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2beta2
kind: HelmRelease
metadata:
  name: myhelm
  namespace: flux-system
spec:
  targetNamespace: myhelm-nginx
  interval: 5m
  chart:
    spec:
      chart: nginx
      version: "22.4.3"
      sourceRef:
        kind: HelmRepository
        name: helmrepo
        namespace: flux-system
      interval: 10m
  install:
    createNamespace: true
    remediation:
      retries: 3
  upgrade:
    remediation:
      retries: 3
      remediateLastFailure: true
    cleanupOnFail: true
  valuesFrom:
    - kind: ConfigMap
      name: nginx-helm-values
      namespace: flux-system
      valuesKey: values.yaml
```

### clusters/production/flux-helm-lab/04-kustomization.yaml

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: flux-helm-lab
  namespace: flux-system
spec:
  interval: 5m
  prune: false
  sourceRef:
    kind: GitRepository
    name: flux-system          # References the GitRepository created during bootstrap
    namespace: flux-system
  path: "./clusters/production/flux-helm-lab"
```

---

## Step 3 — Commit and Push

```bash
git add clusters/production/flux-helm-lab/
git commit -m "Add nginx HelmRelease via Bitnami"
git push origin main
```

Flux picks up the change automatically. To trigger immediately without waiting:

```bash
flux reconcile kustomization flux-system --with-source -n flux-system
```

---

## Step 4 — Verify

```bash
# Check HelmRepository is fetched
flux get sources helm -n flux-system
# NAME       READY   STATUS
# helmrepo   True    Fetched chart index

# Check HelmRelease reconciled
flux get helmreleases -n flux-system
# NAME     READY   STATUS
# myhelm   True    Release reconciliation succeeded

# Check nginx pods
kubectl get pods -n myhelm-nginx

# Check service
kubectl get svc -n myhelm-nginx
```

---

## Step 5 — GitOps Change Flow

To update values or chart version — never `kubectl apply` directly, always go through Git:

```bash
# Edit any manifest, e.g. bump replicaCount or chart version
vi clusters/production/flux-helm-lab/02-values-configmap.yaml

git add .
git commit -m "Update nginx values"
git push origin main

# Force immediate reconciliation
flux reconcile kustomization flux-helm-lab --with-source -n flux-system

# Watch HelmRelease update
flux get helmreleases -n flux-system --watch
```

---

## Useful Commands

```bash
# View all Flux resources
flux get all -n flux-system

# Force reconcile HelmRelease only
flux reconcile helmrelease myhelm -n flux-system

# Suspend auto-sync
flux suspend helmrelease myhelm -n flux-system

# Resume auto-sync
flux resume helmrelease myhelm -n flux-system

# View Flux logs
flux logs -n flux-system --follow

# Describe HelmRelease events
kubectl describe helmrelease myhelm -n flux-system
```
