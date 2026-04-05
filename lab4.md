# FluxCD Helm Lab
### Equivalent to ArgoCD Helm App Deployment (nginx via Bitnami)

---

## ArgoCD → FluxCD Mapping

| ArgoCD | FluxCD | Resource |
|--------|--------|----------|
| Settings → Connect Repo (Helm) | `HelmRepository` | `source.toolkit.fluxcd.io/v1beta2` |
| Application → source.chart | `HelmRelease` | `helm.toolkit.fluxcd.io/v2beta2` |
| `helm.valueFiles: [values.yaml]` | `ConfigMap` + `valuesFrom` | `v1/ConfigMap` |
| `project: myproj` | `Kustomization` | `kustomize.toolkit.fluxcd.io/v1` |
| `syncPolicy.automated.selfHeal: true` | `interval` reconciliation | default Flux behavior |
| `syncPolicy.automated.prune: false` | `prune: false` | Kustomization spec |
| `syncOptions: CreateNamespace=true` | `install.createNamespace: true` | HelmRelease spec |

---

## Step 1 — HelmRepository
> Equivalent to: **ArgoCD → Settings → Connect Repo → Type: Helm → URL: https://charts.bitnami.com/bitnami**

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

```bash
kubectl apply -f 01-helmrepository.yaml

# Verify
flux get sources helm -n flux-system
```

---

## Step 2 — Values ConfigMap
> Equivalent to: **ArgoCD → source.helm.valueFiles: [values.yaml]**

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

```bash
kubectl apply -f 02-values-configmap.yaml
```

---

## Step 3 — HelmRelease
> Equivalent to: **ArgoCD → New Application → chart: nginx, targetRevision: 22.4.3, Sync: Automatic, CreateNamespace: true**

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
    createNamespace: true       # syncOptions: CreateNamespace=true
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

```bash
kubectl apply -f 03-helmrelease.yaml

# Watch reconciliation
flux get helmreleases -n flux-system

# Verify pods
kubectl get pods -n myhelm-nginx
```

---

## Step 4 — Kustomization
> Equivalent to: **ArgoCD → spec.project: myproj + prune: false**

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: myproj-helmlab
  namespace: flux-system
spec:
  interval: 5m
  prune: false                  # syncPolicy.automated.prune: false
  sourceRef:
    kind: GitRepository
    name: flux-system
    namespace: flux-system
  path: "./flux-helm-lab"
```

```bash
kubectl apply -f 04-kustomization.yaml
```

---

## Useful Commands

```bash
# Force immediate sync (like ArgoCD "Sync Now")
flux reconcile helmrelease myhelm -n flux-system

# Suspend auto-sync (like disabling ArgoCD automated sync)
flux suspend helmrelease myhelm -n flux-system

# Resume auto-sync
flux resume helmrelease myhelm -n flux-system

# View all Flux resources
flux get all -n flux-system

# View logs
flux logs -n flux-system --follow

# Describe HelmRelease events
kubectl describe helmrelease myhelm -n flux-system
```
