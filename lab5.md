# FluxCD Image Update Automation Workshop
### Registry: Docker Hub | Repo: flux-gitops | Cluster: clusters/production

---

## Overview

FluxCD Image Update Automation watches Docker Hub for new image tags,
updates the image tag in Git automatically, and Flux then deploys the new image.

```
Developer pushes         Flux scans Docker Hub      Flux commits to Git     Deploys
docker push         →    matches ImagePolicy    →    new tag in manifest →   to cluster
myuser/myapp:1.2.3       semver: >=1.0.0             image: myapp:1.2.3
```

**Three components work together:**

| Resource | API | Purpose |
|---|---|---|
| `ImageRepository` | `image.toolkit.fluxcd.io/v1beta2` | Scans Docker Hub for new tags |
| `ImagePolicy` | `image.toolkit.fluxcd.io/v1beta2` | Filters tags by semver/pattern |
| `ImageUpdateAutomation` | `image.toolkit.fluxcd.io/v1beta1` | Commits updated tag back to Git |

---

## Prerequisites

```bash
# Verify image controllers are installed
kubectl get deploy -n flux-system | grep image
# image-automation-controller
# image-reflector-controller

# If missing, re-bootstrap with image automation components
flux bootstrap github \
  --owner=<YOUR_USER> \
  --repository=flux-gitops \
  --branch=main \
  --path=clusters/production \
  --personal \
  --components-extra=image-reflector-controller,image-automation-controller
```

---

## Repo Structure

```
flux-gitops/
└── clusters/
    └── production/
        ├── flux-system/                      ← bootstrap (do not touch)
        └── flux-image-update/                ← add these files
            ├── kustomization.yaml
            ├── 02-imagerepository.yaml
            ├── 03-imagepolicy.yaml
            ├── 04-imageupdateautomation.yaml
            └── 05-deployment.yaml
```

> `01-dockerhub-secret.yaml` is applied manually — never commit credentials to Git.

---

## Step 1 — Create Folder

```bash
mkdir -p clusters/production/flux-image-update
```

---

## Step 2 — Create Docker Hub Secret

Flux needs credentials to pull image metadata from Docker Hub.

```bash
# Create secret from Docker Hub credentials
kubectl create secret docker-registry dockerhub-secret \
  --docker-server=https://index.docker.io/v1/ \
  --docker-username=<YOUR_DOCKERHUB_USERNAME> \
  --docker-password=<YOUR_DOCKERHUB_TOKEN> \
  --namespace=flux-system
```

> Use a Docker Hub **Access Token** (not your password):
> Docker Hub → Account Settings → Security → New Access Token

Verify secret was created:

```bash
kubectl get secret dockerhub-secret -n flux-system
```

---

## Step 3 — ImageRepository

Tells Flux which Docker Hub image to scan for new tags.

### clusters/production/flux-image-update/02-imagerepository.yaml

```yaml
apiVersion: image.toolkit.fluxcd.io/v1beta2
kind: ImageRepository
metadata:
  name: myapp
  namespace: flux-system
spec:
  image: docker.io/myuser/myapp       # Docker Hub image (docker.io/username/repo)
  interval: 1m                        # Scan Docker Hub every 1 minute
  secretRef:
    name: dockerhub-secret            # Credentials from Step 2
```

```bash
kubectl apply -f clusters/production/flux-image-update/02-imagerepository.yaml

# Verify Flux can reach Docker Hub and scan tags
flux get image repository -n flux-system
# NAME    READY   STATUS
# myapp   True    successful scan: found 5 tags
```

---

## Step 4 — ImagePolicy

Filters which Docker Hub tags are valid for deployment.

### clusters/production/flux-image-update/03-imagepolicy.yaml

```yaml
apiVersion: image.toolkit.fluxcd.io/v1beta2
kind: ImagePolicy
metadata:
  name: myapp
  namespace: flux-system
spec:
  imageRepositoryRef:
    name: myapp                  # References the ImageRepository above
  policy:
    semver:
      range: ">=1.0.0"          # Only deploy semver tags >= 1.0.0
```

**Other policy options:**

```yaml
# Option A — semver range (production safe, ignores dev/rc tags)
policy:
  semver:
    range: ">=1.0.0 <2.0.0"    # Only 1.x.x tags

# Option B — alphabetical (for date-based tags like 20240101-abc123)
policy:
  alphabetical:
    order: desc                  # Latest date tag wins

# Option C — filter by tag pattern before applying policy
spec:
  filterTags:
    pattern: "^main-[a-f0-9]+-(?P<ts>[0-9]+)"  # match main-abc123-1234567890
    extract: "$ts"                               # extract timestamp for ordering
  policy:
    numerical:
      order: asc
```

```bash
# Verify policy resolved the latest tag
flux get image policy -n flux-system
# NAME    READY   STATUS
# myapp   True    Latest image tag for 'docker.io/myuser/myapp' resolved to: 1.2.3
```

---

## Step 5 — Add Image Marker to Deployment

The marker comment tells Flux exactly which field to update in Git.

### clusters/production/flux-image-update/05-deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: myapp
spec:
  replicas: 1
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
        - name: myapp
          image: docker.io/myuser/myapp:1.0.0 # {"$imagepolicy": "flux-system:myapp"}
          ports:
            - containerPort: 80
```

> The marker `# {"$imagepolicy": "flux-system:myapp"}` tells Flux:
> - `flux-system` = namespace where the ImagePolicy lives
> - `myapp` = name of the ImagePolicy
>
> When a new tag `1.2.3` is found, Flux rewrites this line to:
> `image: docker.io/myuser/myapp:1.2.3 # {"$imagepolicy": "flux-system:myapp"}`

**For HelmRelease (when image and tag are separate fields):**

```yaml
spec:
  values:
    image:
      repository: docker.io/myuser/myapp
      tag: "1.0.0" # {"$imagepolicy": "flux-system:myapp:tag"}
```

---

## Step 6 — ImageUpdateAutomation

Commits the updated image tag back to Git on your behalf.

### clusters/production/flux-image-update/04-imageupdateautomation.yaml

```yaml
apiVersion: image.toolkit.fluxcd.io/v1beta1
kind: ImageUpdateAutomation
metadata:
  name: myapp-image-update
  namespace: flux-system
spec:
  sourceRef:
    kind: GitRepository
    name: flux-system              # GitRepository created during bootstrap

  git:
    checkout:
      ref:
        branch: main               # Branch to read from
    commit:
      author:
        name: fluxbot
        email: fluxbot@example.com
      messageTemplate: |
        chore: update {{range .Updated.Images}}{{.}}{{end}}
    push:
      branch: main                 # Branch to push commits to

  update:
    strategy: Setters              # Scans for # {"$imagepolicy": ...} markers
    path: ./clusters/production    # Scan all files under this path

  interval: 1m                     # How often to check for pending updates
```

---

## Step 7 — Allow Flux to Push to GitHub

Flux commits new image tags back to Git — it needs write access.

```bash
# Get the Flux deploy public key
kubectl -n flux-system get secret flux-system \
  -o jsonpath='{.data.identity\.pub}' | base64 -d
```

Add this key to GitHub:
**GitHub → Your Repo → Settings → Deploy Keys → Add deploy key → Allow write access ✓**

---

## Step 8 — kustomization.yaml

### clusters/production/flux-image-update/kustomization.yaml

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - 02-imagerepository.yaml
  - 03-imagepolicy.yaml
  - 04-imageupdateautomation.yaml
  - 05-deployment.yaml
```

---

## Step 9 — Commit and Push

```bash
git add clusters/production/flux-image-update/
git commit -m "Add Docker Hub image update automation"
git push origin main

flux reconcile kustomization flux-system --with-source -n flux-system
```

---

## Step 10 — Test the Full Flow

```bash
# Push a new image version to Docker Hub
docker build -t myuser/myapp:1.2.3 .
docker push myuser/myapp:1.2.3

# Wait ~1 minute then check if Flux detected the new tag
flux get image repository myapp -n flux-system
flux get image policy myapp -n flux-system
# NAME    READY   STATUS
# myapp   True    Latest image tag for 'docker.io/myuser/myapp' resolved to: 1.2.3

# Check if Flux committed the update to Git
git pull
git log --oneline | head -3
# a1b2c3d chore: update docker.io/myuser/myapp:1.2.3
# ...

# Check new pod is running with updated image
kubectl get pods -n myapp
kubectl describe pod -n myapp | grep Image:
# Image: docker.io/myuser/myapp:1.2.3
```

---

## Full Automated Flow

```
1. Developer builds & pushes
   docker push myuser/myapp:1.2.3
           │
           ▼ (interval: 1m)
2. ImageRepository scans Docker Hub
   finds tag: 1.2.3
           │
           ▼
3. ImagePolicy evaluates tag
   semver 1.2.3 matches >=1.0.0 ✓
           │
           ▼
4. ImageUpdateAutomation commits to Git
   "chore: update docker.io/myuser/myapp:1.2.3"
   updates: image: myuser/myapp:1.2.3 # {"$imagepolicy": ...}
           │
           ▼ (interval: 5m)
5. Flux detects Git change
   Kustomization/HelmRelease reconciles
   New pod deployed with image:1.2.3
```

---

## Useful Commands

```bash
# Force scan Docker Hub now
flux reconcile image repository myapp -n flux-system

# Check resolved latest tag
flux get image policy myapp -n flux-system

# Check automation last commit
flux get image update -n flux-system

# View all image resources
flux get images all -n flux-system

# Suspend image auto-updates
flux suspend image update myapp-image-update -n flux-system

# Resume image auto-updates
flux resume image update myapp-image-update -n flux-system

# View image controller logs
kubectl logs -n flux-system deploy/image-reflector-controller --tail=50
kubectl logs -n flux-system deploy/image-automation-controller --tail=50
```
