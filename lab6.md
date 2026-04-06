# GitLab CI/CD Workshop — Lab 6
### GitLab Container Registry & Flux Image Automation

---

## Overview

```
Push code to GitLab   →   GitLab CI builds image   →   Push to GitLab Registry
main branch                .gitlab-ci.yml                registry.gitlab.com/user/cicd

         ↓
Flux watches Registry   →   Detects new image tag   →   Updates K8s Deployment
flux-system namespace        ImageRepository/Policy        Git commit + apply
```

> **Prerequisites:** Complete Lab 1 — you should have a working GitLab project (`cicd`) with `app.py`, `requirements.txt`, `dockerfile`, and `.gitlab-ci.yml`.

---

## What Changes From Lab 1?

| Lab 1 | Lab 2 |
|-------|-------|
| Push image → **Docker Hub** | Push image → **GitLab Container Registry** |
| Manual `docker pull` to deploy | **Flux** auto-deploys on new image |
| No Kubernetes | Kubernetes + Flux GitOps |

---

## Part A — Use GitLab Container Registry

### Why GitLab Registry?

- Built into every GitLab project — no external account needed
- Automatically authenticated in CI/CD pipelines using `$CI_REGISTRY_*` variables
- Images are private by default, scoped to your project

---

### Step 1 — Enable Container Registry

1. Go to your `cicd` project → **Settings** → **General**
2. Expand **Visibility, project features, permissions**
3. Toggle **Container registry** → **On**
4. Click **Save changes**

> The registry is usually enabled by default on GitLab.com.

---

### Step 2 — Update `.gitlab-ci.yml`

Replace your existing `.gitlab-ci.yml` with the following:

```yaml
stages:
  - build

variables:
  DOCKERFILE: "dockerfile"
  IMAGE_NAME: $CI_REGISTRY_IMAGE          # e.g. registry.gitlab.com/youruser/cicd
  IMAGE_TAG: $CI_COMMIT_SHORT_SHA         # unique tag per commit (e.g. a1b2c3d4)
  DOCKER_HOST: tcp://docker:2375/
  DOCKER_TLS_CERTDIR: ""

build_docker_image:
  stage: build
  image: docker:latest
  services:
    - docker:dind
  before_script:
    - echo "Logging into GitLab Container Registry..."
    - echo "$CI_REGISTRY_PASSWORD" | docker login $CI_REGISTRY -u "$CI_REGISTRY_USER" --password-stdin
  script:
    - echo "Building Docker image..."
    - docker build -f $DOCKERFILE -t $IMAGE_NAME:$IMAGE_TAG .
    - docker tag $IMAGE_NAME:$IMAGE_TAG $IMAGE_NAME:latest
    - echo "Pushing Docker image..."
    - docker push $IMAGE_NAME:$IMAGE_TAG
    - docker push $IMAGE_NAME:latest
  only:
    - main
```

> **Key difference from Lab 1:** We use built-in GitLab CI variables — no secrets needed!
>
> | Variable | Value (auto-injected by GitLab) |
> |---|---|
> | `$CI_REGISTRY` | `registry.gitlab.com` |
> | `$CI_REGISTRY_USER` | `gitlab-ci-token` |
> | `$CI_REGISTRY_PASSWORD` | One-time CI job token |
> | `$CI_REGISTRY_IMAGE` | `registry.gitlab.com/<user>/cicd` |
> | `$CI_COMMIT_SHORT_SHA` | Short commit hash (e.g. `a1b2c3d4`) |

---

### Step 3 — Push and Verify

```bash
git add .gitlab-ci.yml
git commit -m "Switch to GitLab Container Registry"
git push origin main
```

After the pipeline completes:

1. Go to your project → **Deploy** → **Container Registry**
2. You should see your image with tags `latest` and the commit SHA

```
registry.gitlab.com/youruser/cicd:latest
registry.gitlab.com/youruser/cicd:a1b2c3d4
```

---

## Part B — Set Up Flux for Image Automation

Flux is a GitOps tool that keeps your Kubernetes cluster in sync with Git. The **Image Automation** feature watches a container registry and automatically updates your deployment when a new image is pushed.

---

### Step 4 — Prerequisites

Make sure you have these tools installed locally:

```bash
# Flux CLI
curl -s https://fluxcd.io/install.sh | sudo bash

# kubectl (if not already installed)
# Verify cluster access
kubectl get nodes
```

> You need a running Kubernetes cluster — use **kind**, **minikube**, or any cloud cluster (GKE, EKS, AKS).

---

### Step 5 — Bootstrap Flux on Your Cluster

Flux needs a Git repository to store its own configuration. We'll use a separate repository (or a folder in your existing `cicd` repo).

```bash
export GITLAB_TOKEN=<your-gitlab-personal-access-token>
export GITLAB_USER=<your-gitlab-username>

flux bootstrap gitlab \
  --owner=$GITLAB_USER \
  --repository=flux-infra \
  --branch=main \
  --path=clusters/my-cluster \
  --token-auth \
  --personal
```

This will:
- Create a new GitLab repo called `flux-infra` (if it doesn't exist)
- Install Flux components into the `flux-system` namespace
- Commit Flux manifests to `flux-infra`

Verify Flux is running:

```bash
flux check
kubectl get pods -n flux-system
```

---

### Step 6 — Create a GitLab Deploy Token for Flux

Flux needs read access to your GitLab Container Registry to watch for new images.

1. Go to your `cicd` project → **Settings** → **Repository**
2. Expand **Deploy tokens**
3. Click **Add token**
   - Name: `flux-image-reader`
   - Scopes: ✓ `read_registry`
4. Click **Create deploy token**
5. Copy the **username** and **token** — save them now

Create a Kubernetes secret for Flux to use:

```bash
kubectl create secret docker-registry gitlab-registry-secret \
  --docker-server=registry.gitlab.com \
  --docker-username=<deploy-token-username> \
  --docker-password=<deploy-token-value> \
  --namespace=flux-system
```

---

### Step 7 — Deploy the Flask App to Kubernetes

Create a `k8s/` folder in your `cicd` project with a deployment manifest.

#### k8s/deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: flask-app
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: flask-app
  template:
    metadata:
      labels:
        app: flask-app
    spec:
      containers:
        - name: flask-app
          image: registry.gitlab.com/<YOUR_USERNAME>/cicd:latest  # {"$imagepolicy": "flux-system:flask-app"}
          ports:
            - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: flask-app
  namespace: default
spec:
  selector:
    app: flask-app
  ports:
    - port: 80
      targetPort: 8080
  type: ClusterIP
```

> The comment `# {"$imagepolicy": "flux-system:flask-app"}` is a **marker** that tells Flux which line to update when a new image is detected.

```bash
kubectl apply -f k8s/deployment.yaml
kubectl get pods
```

---

### Step 8 — Create Flux Image Automation Resources

In your `flux-infra` repo (cloned locally), add these manifests under `clusters/my-cluster/apps/`:

#### image-repository.yaml

```yaml
apiVersion: image.toolkit.fluxcd.io/v1beta2
kind: ImageRepository
metadata:
  name: flask-app
  namespace: flux-system
spec:
  image: registry.gitlab.com/<YOUR_USERNAME>/cicd
  interval: 1m
  secretRef:
    name: gitlab-registry-secret
```

This tells Flux: *"Check this registry every 1 minute for new tags."*

---

#### image-policy.yaml

```yaml
apiVersion: image.toolkit.fluxcd.io/v1beta2
kind: ImagePolicy
metadata:
  name: flask-app
  namespace: flux-system
spec:
  imageRepositoryRef:
    name: flask-app
  policy:
    semver:
      range: ">=0.0.1"
```

> For commit-SHA tags (like `a1b2c3d4`), use alphabetical ordering instead:
>
> ```yaml
> policy:
>   alphabetical:
>     order: asc
> ```

---

#### image-update-automation.yaml

```yaml
apiVersion: image.toolkit.fluxcd.io/v1beta1
kind: ImageUpdateAutomation
metadata:
  name: flask-app
  namespace: flux-system
spec:
  interval: 1m
  sourceRef:
    kind: GitRepository
    name: flux-system
  git:
    checkout:
      ref:
        branch: main
    commit:
      author:
        email: fluxbot@example.com
        name: Flux Bot
      messageTemplate: "chore: update flask-app image to {{range .Updated.Images}}{{.}}{{end}}"
    push:
      branch: main
  update:
    path: ./clusters/my-cluster
    strategy: Setters
```

This tells Flux: *"When a new image matches the policy, commit the updated image tag back to Git and apply it to the cluster."*

---

Commit and push all three files to `flux-infra`:

```bash
cd flux-infra
git add clusters/my-cluster/apps/
git commit -m "Add Flux image automation for Flask app"
git push origin main
```

Flux will automatically pick up the new manifests within its reconciliation interval.

---

### Step 9 — Verify Image Automation

Check that Flux can see the registry:

```bash
flux get image repository flask-app
```

Expected output:
```
NAME        LAST SCAN   TAGS   READY   MESSAGE
flask-app   10s ago     3      True    successful scan: found 3 tags
```

Check the image policy:

```bash
flux get image policy flask-app
```

Expected output:
```
NAME        LATEST IMAGE                                         READY
flask-app   registry.gitlab.com/youruser/cicd:a1b2c3d4          True
```

---

## Step 10 — End-to-End Test

Make a change to `app.py`:

```python
@app.route('/')
def hello_world():
    return 'Hello, Flux!'   # changed
```

Push to GitLab:

```bash
git add app.py
git commit -m "Update greeting for Flux demo"
git push origin main
```

Watch what happens automatically:

1. **GitLab CI** builds a new image tagged with the new commit SHA → pushes to GitLab Registry
2. **Flux ImageRepository** detects the new tag within 1 minute
3. **Flux ImagePolicy** selects the latest tag
4. **Flux ImageUpdateAutomation** commits the updated image ref to `flux-infra` Git repo
5. **Flux** reconciles the cluster → rolls out the new pod

```bash
# Watch the rollout
kubectl rollout status deployment/flask-app

# Test the app
kubectl port-forward svc/flask-app 8080:80
curl http://localhost:8080
# Hello, Flux!
```

---

## File Structure (End State)

```
cicd/                            ← your app repo (GitLab)
├── app.py
├── requirements.txt
├── dockerfile
├── .gitlab-ci.yml               ← updated for GitLab Registry
└── k8s/
    └── deployment.yaml          ← includes $imagepolicy marker

flux-infra/                      ← Flux config repo (GitLab)
└── clusters/
    └── my-cluster/
        ├── flux-system/         ← auto-generated by bootstrap
        └── apps/
            ├── image-repository.yaml
            ├── image-policy.yaml
            └── image-update-automation.yaml
```

---

## Troubleshooting

| Error | Fix |
|-------|-----|
| `unauthorized` when Flux scans registry | Check `gitlab-registry-secret` — verify deploy token has `read_registry` scope |
| `ImageRepository` not ready | Run `flux logs --kind=ImageRepository` to see errors |
| Image tag not updating in Git | Ensure `$imagepolicy` marker comment is on the same line as `image:` in `deployment.yaml` |
| Flux bootstrap fails | Verify `GITLAB_TOKEN` has `api` and `write_repository` scopes |
| Pod stuck in `ImagePullBackOff` | Create `imagePullSecrets` in the `default` namespace pointing to your registry secret |
| Pipeline not triggering | Ensure branch is `main` and Container Registry is enabled in project settings |

---

## Summary

| Step | What You Did |
|------|-------------|
| Part A | Switched from Docker Hub to **GitLab Container Registry** using built-in CI variables |
| Part B Step 4–5 | Bootstrapped **Flux** onto your Kubernetes cluster |
| Part B Step 6 | Created a **deploy token** for Flux to read the registry |
| Part B Step 7 | Deployed Flask app to Kubernetes with the `$imagepolicy` marker |
| Part B Step 8 | Created `ImageRepository`, `ImagePolicy`, and `ImageUpdateAutomation` manifests |
| Part B Step 9–10 | Verified full **GitOps loop**: code push → CI build → registry → Flux → cluster |
