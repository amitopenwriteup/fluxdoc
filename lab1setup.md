# HANDS-ON LAB 3.1
## GitLab CI/CD + FluxCD Integration
### End-to-end GitOps: Code Commit → Build → Deploy

**Duration:** 90 minutes · **Level:** Intermediate · **Prerequisites:** Kubernetes cluster, kubectl, flux CLI

---

## Lab Overview

In this lab you will build a complete GitOps pipeline that mirrors real-world production practice. GitLab CI/CD handles the Continuous Integration side: building your container image, running tests, and pushing to GitLab Container Registry. FluxCD handles the Continuous Deployment side: watching your manifest repository and reconciling your Kubernetes cluster to match the desired state.

By the end of this lab you will have a fully automated pathway from a code commit on your laptop to a running deployment on Kubernetes — with no manual `kubectl apply` steps.

| Learning Objectives | Tools & Technologies |
|---|---|
| ✓ Set up application and manifest repos in GitLab | GitLab (SaaS or self-hosted) |
| ✓ Write a CI pipeline to build and push container images | FluxCD v2 (with Helm controller) |
| ✓ Configure Flux to watch your GitLab manifest repository | GitLab Container Registry |
| ✓ Implement automated image tag updates via Flux | kubectl, flux CLI |
| ✓ Observe and validate the full commit-to-deploy workflow | SOPS or GitLab CI variables for secrets |

---

## Lab Time Plan

| Duration | Activity |
|---|---|
| 10 min | Prerequisites check & environment setup |
| 15 min | Step 1: Create GitLab repositories |
| 20 min | Step 2: Write and test the GitLab CI pipeline |
| 20 min | Step 3: Bootstrap and configure FluxCD |
| 15 min | Step 4: Set up image automation |
| 10 min | Step 5: End-to-end test & validation |

---

## Prerequisites

> **INFO:** Complete all prerequisite checks before the lab begins. Raise your hand if any check fails. Ask your lab facilitator for assistance — do not skip prerequisites.

### Environment Requirements

- A running Kubernetes cluster (k3s, kind, or managed cluster)
- `kubectl` configured and connected: `kubectl cluster-info`
- `flux` CLI installed: `flux version` (v2.0+)
- `git` installed and configured with your name and email
- `docker` or `podman` available for local testing (optional)
- A GitLab account (gitlab.com or self-hosted) with at least Developer role

### Verify Your Setup

Run the following commands and confirm all return without errors:

```bash
# Kubernetes connectivity
kubectl cluster-info

# Flux CLI version
flux version

# Git identity
git config --global user.name "Your Name"
git config --global user.email "you@example.com"
```

> **NOTE:** The flux CLI must be v2.0 or later. Flux v1 and Flux v2 are not compatible.
> Run: `flux install --version` to check what would be bootstrapped.

---

## 01 · Create GitLab Repositories

In GitLab-based GitOps you maintain two separate repositories. This separation is a best practice because it enforces a clear boundary between application code (owned by developers) and deployment state (owned by the platform team or automation).

| `app-repo` | `manifest-repo` |
|---|---|
| Application source code | Kubernetes manifests (YAML) |
| Dockerfile | Kustomize overlays per environment |
| `.gitlab-ci.yml` pipeline | FluxCD resource definitions |
| Unit tests and linting | Image tag references (updated by CI) |

### 1.1 Create the Application Repository

1. In GitLab, create a new project named `app-repo` (or your app name).
2. Set visibility to **Private**. Ensure Container Registry is enabled in **Settings > General > Visibility**.
3. Clone the repository locally:

```bash
git clone https://gitlab.com/<your-username>/app-repo.git
cd app-repo
```

4. Create a minimal Go/Python/Node app. For this lab we provide a sample app. Copy the files from the lab USB/share:

```
# App structure (sample provided)
app-repo/
  src/           ←  application source
  Dockerfile     ←  multi-stage build
  .gitlab-ci.yml
```

5. Push to GitLab:

```bash
git add . && git commit -m 'Initial commit'
git push origin main
```

### 1.2 Create the Manifest Repository

6. Create a new GitLab project named `manifest-repo`.
7. Set visibility to **Private**. This repo is the source of truth for Flux.
8. Create the following directory structure:

```
manifest-repo/
  clusters/
    production/
      flux-system/    ←  Flux bootstrap files (auto-generated)
  apps/
    base/             ←  Base Kubernetes manifests
      deployment.yaml
      service.yaml
      kustomization.yaml
    overlays/
      production/     ←  Environment-specific patches
        kustomization.yaml
        image-patch.yaml
```

9. Commit and push the directory structure:

```bash
git add . && git commit -m 'Initial manifest structure'
git push origin main
```

> **TIP:** The `clusters/` directory is where Flux will write its own bootstrap files. Keep `apps/` separate so you can manage it independently of Flux internals. Using overlays lets you promote the same base manifests across staging and production.

---

## 02 · Create the GitLab CI Pipeline

The CI pipeline is responsible for: building the container image, running tests, pushing the image to GitLab Container Registry, and updating the image tag in the manifest repository. This last step is the handoff point between CI and GitOps.

### 2.1 Write `.gitlab-ci.yml`

In your `app-repo`, create `.gitlab-ci.yml` with the following content:

```yaml
# .gitlab-ci.yml

variables:
  IMAGE: "$CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA"
  MANIFEST_REPO: "https://oauth2:${MANIFEST_TOKEN}@gitlab.com/<user>/manifest-repo.git"

stages:
  - build
  - test
  - update-manifest

build-image:
  stage: build
  image: docker:24
  services:
    - docker:24-dind
  before_script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
  script:
    - docker build -t $IMAGE .
    - docker push $IMAGE
    - docker tag $IMAGE $CI_REGISTRY_IMAGE:latest
    - docker push $CI_REGISTRY_IMAGE:latest

update-manifest:
  stage: update-manifest
  image: alpine/git
  script:
    - apk add --no-cache yq
    - git clone $MANIFEST_REPO manifest
    - cd manifest
    - yq e '.images[0].newTag = env(CI_COMMIT_SHORT_SHA)' -i apps/overlays/production/kustomization.yaml
    - git config user.email "ci@gitlab.com"
    - git config user.name "GitLab CI"
    - git add .
    - git commit -m "ci: update image tag to $CI_COMMIT_SHORT_SHA"
    - git push
  only:
    - main
```

> **⚠️ WARN:** Never hardcode credentials in `.gitlab-ci.yml`. `MANIFEST_TOKEN` must be stored as a CI/CD Variable (**Settings > CI/CD > Variables**). Mark it as **Masked** so it does not appear in job logs.

### 2.2 Configure CI/CD Variables

In your `app-repo`: **Settings > CI/CD > Variables**. Add the following:

| Variable | Value | Flags |
|---|---|---|
| `MANIFEST_TOKEN` | GitLab Project Access Token (read_write on manifest-repo) | Masked, Protected |
| `CI_REGISTRY` | `registry.gitlab.com` (auto-populated) | Auto |
| `CI_REGISTRY_USER` | `CI_REGISTRY_USER` (auto-populated) | Auto |
| `CI_REGISTRY_PASSWORD` | `CI_REGISTRY_PASSWORD` (auto-populated) | Masked, Auto |

### 2.3 Test Your Pipeline

10. Push a commit to the `main` branch of `app-repo`.
11. Navigate to **CI/CD > Pipelines** in GitLab and watch the stages run.
12. After the pipeline completes, check:
    - **Deploy > Container Registry**: your image should appear with the git SHA tag.
    - `manifest-repo`: the `kustomization.yaml` should have an updated `newTag`.
13. If the pipeline fails, check the job log for the error. Common issues:
    - Docker-in-Docker not enabled (add `services: - docker:24-dind`)
    - `MANIFEST_TOKEN` lacks write permission to `manifest-repo`
    - `yq` not found: ensure `apk add yq` is in `before_script`

---

## 03 · Bootstrap and Configure FluxCD

Bootstrapping Flux installs the Flux controllers on your cluster and configures them to reconcile from your `manifest-repo`. You only need to do this once per cluster.

### 3.1 Create a GitLab Deploy Token for Flux

Flux needs read access to `manifest-repo`. A Deploy Token is the recommended approach: it is scoped to a single repository and can be revoked independently of any user account.

14. In `manifest-repo`: **Settings > Repository > Deploy tokens**.
15. Create a token with scopes: `read_repository` and `read_registry`.
16. Copy the token username and password — you will only see the password once.
17. Create a Kubernetes secret for Flux to use:

```bash
flux create secret git flux-gitlab-auth \
  --url=https://gitlab.com/<user>/manifest-repo \
  --username=<deploy-token-username> \
  --password=<deploy-token-password> \
  --namespace=flux-system
```

### 3.2 Bootstrap Flux on Your Cluster

```bash
# Export your GitLab personal access token (for bootstrapping only)
export GITLAB_TOKEN=<your-personal-access-token>

flux bootstrap gitlab \
  --owner=<your-gitlab-username-or-group> \
  --repository=manifest-repo \
  --branch=main \
  --path=clusters/production \
  --token-auth
```

This command installs Flux controllers (`source-controller`, `kustomize-controller`, etc.) and creates the initial `GitRepository` and `Kustomization` resources in the `flux-system` namespace. It then commits these to your `manifest-repo`.

### 3.3 Verify Flux Is Reconciling

```bash
# Check Flux controllers are running
flux check

# Watch reconciliation events
flux get all -A

# Tail the source controller logs
flux logs --follow --level=info
```

### 3.4 Apply Application Manifests

Create the base Deployment and Service in `apps/base/`. The sample manifests are provided on the lab share. Key points:

- The Deployment image must include the Kustomize image marker comment: `# {"$imagepolicy": "flux-system:my-app"}`
- The Kustomization in `apps/overlays/production/` must reference the base.
- Create a Flux Kustomization resource to tell Flux to apply this path:

```bash
flux create kustomization my-app \
  --source=GitRepository/flux-system \
  --path=./apps/overlays/production \
  --prune=true \
  --interval=1m \
  --namespace=flux-system
```

> **NOTE:** `--prune=true` enables drift correction: if a resource is removed from Git, Flux will delete it from the cluster. `--interval=1m` means Flux will check for changes every minute. Reduce to `30s` for faster testing during this lab.

---

## 04 · Set Up Image Automation

Image automation removes the manual (or CI-scripted) step of updating image tags in the manifest repo. Flux watches your container registry for new tags, selects the right one using a policy, and commits the updated tag back to Git automatically.

### 4.1 Create an ImageRepository

This tells Flux which container registry image to watch:

```bash
flux create image repository my-app \
  --image=registry.gitlab.com/<user>/app-repo \
  --interval=1m \
  --secret-ref=gitlab-registry-auth \
  --namespace=flux-system
```

You also need a registry pull secret. Create it from your deploy token credentials:

```bash
kubectl create secret docker-registry gitlab-registry-auth \
  --docker-server=registry.gitlab.com \
  --docker-username=<deploy-token-username> \
  --docker-password=<deploy-token-password> \
  --namespace=flux-system
```

### 4.2 Create an ImagePolicy

An ImagePolicy selects which tag Flux should use. For this lab we use a semver policy that matches git SHA-style tags (all non-`latest` tags):

```bash
flux create image policy my-app \
  --image-ref=my-app \
  --select-semver='>=0.0.0' \
  --namespace=flux-system
```

Alternatively, for tags like `main-abc1234` (branch + SHA), use a filter regex:

```bash
flux create image policy my-app \
  --image-ref=my-app \
  --filter-regex='^main-[a-f0-9]+$' \
  --select-numeric=asc \
  --namespace=flux-system
```

### 4.3 Create an ImageUpdateAutomation

This resource commits tag updates back to the manifest repo:

```bash
flux create image update flux-system \
  --interval=1m \
  --git-repo-ref=flux-system \
  --git-repo-path=./apps/overlays/production \
  --checkout-branch=main \
  --push-branch=main \
  --author-name=FluxBot \
  --author-email=fluxbot@cluster.local \
  --commit-template="{{range .Updated.Images}}chore: update {{.Identifier}} to {{.NewTag}}{{end}}" \
  --namespace=flux-system
```

### 4.4 Add Image Marker to Manifest

For Flux to know which field to update, add a marker comment next to the image tag in your Deployment YAML in `manifest-repo`:

```yaml
# apps/base/deployment.yaml (relevant excerpt)
spec:
  containers:
  - name: my-app
    image: registry.gitlab.com/<user>/app-repo:latest # {"$imagepolicy": "flux-system:my-app"}
```

> **INFO:** The marker comment tells `ImageUpdateAutomation` exactly which YAML field to update. Without this comment, Flux will find no fields to update and the automation will do nothing. The format is: `# {"$imagepolicy": "<namespace>:<policy-name>"}`

---

## 05 · End-to-End Test & Validation

This is the moment the whole lab has been building toward. Make a code change, push it, and watch the entire pipeline execute automatically — culminating in a running deployment on Kubernetes.

### 5.1 Trigger the Pipeline

18. Open `app-repo` and make a visible code change (e.g., update a version string or a greeting message).
19. Commit and push:

```bash
git commit -am "feat: update greeting to v2"
git push origin main
```

20. In GitLab, navigate to **CI/CD > Pipelines** and watch the `build-image` and `update-manifest` stages complete.

### 5.2 Observe Flux Reconciliation

After the pipeline completes, observe Flux picking up the change:

```bash
# Watch image scan for new tags (run in a new terminal)
flux get image repository my-app --watch

# Watch Flux detect the manifest change and reconcile
flux get kustomization my-app --watch

# Watch the Deployment rollout
kubectl rollout status deployment/my-app -n default
```

### 5.3 Validation Checklist

Use this checklist to confirm everything is working:

| Status | Validation Step |
|---|---|
| ☐ | GitLab CI pipeline completed successfully (all stages green) |
| ☐ | Container image with SHA tag visible in GitLab Container Registry |
| ☐ | `manifest-repo` has a new commit from GitLab CI updating the image tag |
| ☐ | `flux get image repository my-app` shows LAST SCAN recent timestamp |
| ☐ | `flux get image policy my-app` shows the latest SHA tag as LATEST IMAGE |
| ☐ | ImageUpdateAutomation has committed a tag update to `manifest-repo` |
| ☐ | `flux get kustomization my-app` shows READY True and the latest revision |
| ☐ | `kubectl get deployment my-app` shows AVAILABLE replicas > 0 |
| ☐ | `kubectl describe deployment my-app` shows the new image SHA in the spec |
| ☐ | Application responds correctly (`curl` the service endpoint) |

---

## Troubleshooting Guide

| Symptom | Resolution |
|---|---|
| Flux shows `NoSuchBucket` or `401` on GitRepository | Check the Kubernetes secret name matches `--secret-ref`. Re-create with correct deploy token credentials. |
| ImageRepository not scanning (last scan time stuck) | Verify the `docker-registry` secret exists in `flux-system` and contains valid credentials for `registry.gitlab.com`. |
| ImageUpdateAutomation makes no commits | Confirm the `# {"$imagepolicy"}` marker comment is present in the manifest. Check write permission of the deploy token. |
| Pipeline fails at `update-manifest` stage | Check `MANIFEST_TOKEN` variable is set, Masked, and has `api` or `write_repository` scope on the `manifest-repo`. |
| Flux Kustomization shows `ReconcileFailed` | Run `flux logs --level=error` to see the exact YAML error. Validate manifests locally with `kubectl apply --dry-run=server`. |
| Deployment not updating despite tag change in Git | Check `flux get kustomization`. If stale, run `flux reconcile kustomization my-app --with-source`. |

---

## Extension Challenges

Finished early? Try one of these extension tasks to deepen your understanding:

**EXT 1** — Add a staging environment: create `apps/overlays/staging/` with a separate Flux Kustomization. Configure Flux to auto-update staging but require a manual MR approval for production.

**EXT 2** — Add a GitLab MR notification: configure Flux Notifications to post a Slack or GitLab comment when a Kustomization reconciles successfully. *Hint: use `flux create alert` and `flux create alert-provider`.*

**EXT 3** — Encrypt a secret in `manifest-repo` using SOPS + age. Configure Flux to decrypt it at deploy time using `flux create secret sops`.

---

## Quick Reference

### Useful `flux` CLI Commands

```bash
# Force immediate reconciliation
flux reconcile source git flux-system
flux reconcile kustomization my-app --with-source

# View all Flux resources
flux get all -A

# Suspend / resume reconciliation (for maintenance)
flux suspend kustomization my-app
flux resume kustomization my-app

# Export resource definitions to YAML
flux export kustomization my-app
flux export image repository my-app
```

### Key Documentation Links

- **FluxCD docs:** https://fluxcd.io/flux/
- **GitLab CI/CD docs:** https://docs.gitlab.com/ee/ci/
- **Flux image automation guide:** https://fluxcd.io/flux/guides/image-update/
- **Flux GitLab bootstrap:** https://fluxcd.io/flux/installation/bootstrap/gitlab/

---

*Lab 3.1 Complete · Advanced GitOps Workshop · GitLab + FluxCD*
