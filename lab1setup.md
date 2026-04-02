# HANDS-ON LAB 1
## FluxCD Bootstrap with GitLab — Installation & CLI Reference
### Flux CLI Installation · GitLab Bootstrap · Verification Commands

**Duration:** 60 minutes · **Level:** Beginner–Intermediate · **Prerequisites:** Kubernetes cluster, kubectl, GitLab account

---

## Lab Overview

This lab walks you through everything needed to get FluxCD running on a Kubernetes cluster using GitLab as the GitOps source of truth. You will install the Flux CLI, bootstrap Flux onto your cluster via GitLab, and use the Flux CLI to inspect and verify your installation.

By the end of this lab you will have a live Flux installation that continuously reconciles your cluster state from a GitLab repository.

| Learning Objectives | Tools & Technologies |
|---|---|
| ✓ Install the Flux CLI on Linux, macOS, or Windows | FluxCD v2 |
| ✓ Verify CLI installation and pre-flight checks | GitLab (SaaS or self-hosted) |
| ✓ Create a GitLab Personal Access Token for bootstrap | kubectl |
| ✓ Bootstrap Flux onto a Kubernetes cluster via GitLab | flux CLI |
| ✓ Use `flux` CLI commands to inspect and manage Flux | GitLab Container Registry |

---

## Lab Time Plan

| Duration | Activity |
|---|---|
| 10 min | Prerequisites check & environment setup |
| 15 min | Step 1: Install the Flux CLI |
| 10 min | Step 2: Pre-flight checks |
| 15 min | Step 3: Bootstrap Flux with GitLab |
| 10 min | Step 4: Verify the installation |

---

## Prerequisites

> **INFO:** Complete all prerequisite checks before the lab begins. Raise your hand if any check fails.

- A running Kubernetes cluster (k3s, kind, GKE, EKS, AKS, or similar)
- `kubectl` installed and configured: `kubectl cluster-info` should return without error
- A GitLab account at [gitlab.com](https://gitlab.com) or a self-hosted instance
- Internet access from your workstation to GitLab and the cluster

---

## 01 · Install the Flux CLI

The Flux CLI (`flux`) is the primary tool for interacting with FluxCD. It is used to bootstrap, inspect, reconcile, suspend, and export Flux resources.

### 1.1 Linux / macOS — Official Install Script

The quickest way to install the latest stable version:

```bash
curl -s https://fluxcd.io/install.sh | sudo bash
```

This downloads the correct binary for your OS and architecture, installs it to `/usr/local/bin/flux`, and sets executable permissions.

> **TIP:** If you do not have `sudo` access, download the binary manually (see Section 1.3) and place it in a directory on your `$PATH`.

### 1.2 macOS — Homebrew

```bash
brew install fluxcd/tap/flux
```

To upgrade an existing installation:

```bash
brew upgrade fluxcd/tap/flux
```

### 1.3 Linux — Manual Binary Download

Use this method if the install script is unavailable or you need a specific version:

```bash
# Set the desired version
FLUX_VERSION=2.3.0

# Download the binary for Linux amd64
curl -Lo flux.tar.gz \
  https://github.com/fluxcd/flux2/releases/download/v${FLUX_VERSION}/flux_${FLUX_VERSION}_linux_amd64.tar.gz

# Extract and install
tar -xzf flux.tar.gz
chmod +x flux
sudo mv flux /usr/local/bin/flux

# Clean up
rm flux.tar.gz
```

For **ARM64** (e.g., Raspberry Pi, Apple M-series in a Linux VM), replace `linux_amd64` with `linux_arm64`.

### 1.4 Windows — Chocolatey

```powershell
choco install flux
```

### 1.5 Windows — Scoop

```powershell
scoop install flux
```

### 1.6 Windows — Manual (PowerShell)

```powershell
$FLUX_VERSION = "2.3.0"
$url = "https://github.com/fluxcd/flux2/releases/download/v$FLUX_VERSION/flux_${FLUX_VERSION}_windows_amd64.zip"
Invoke-WebRequest -Uri $url -OutFile flux.zip
Expand-Archive flux.zip -DestinationPath $env:USERPROFILE\bin
# Ensure $env:USERPROFILE\bin is in your PATH
```

### 1.7 Verify the Installation

After installation, confirm the CLI is accessible and returns a version:

```bash
flux version
```

Expected output (version numbers will vary):

```
flux: v2.3.0
distribution: flux-v2.3.0
server: v2.3.0
```

> **NOTE:** The Flux CLI must be **v2.0 or later**. Flux v1 and Flux v2 are architecturally incompatible — do not mix them. If you see `flux: v0.x` you have an old binary; reinstall using the steps above.

---

## 02 · Pre-flight Checks

Before bootstrapping, run the Flux pre-flight check. This validates that your cluster meets all requirements to run FluxCD.

### 2.1 Run the Pre-flight Check

```bash
flux check --pre
```

Flux will test the following:

| Check | What It Validates |
|---|---|
| `kubectl` context | A valid Kubernetes context is active |
| API server connectivity | The cluster API server is reachable |
| Kubernetes version | The cluster runs a supported Kubernetes version (≥ 1.26) |
| RBAC permissions | The current user can create namespaces and CRDs |
| CRD availability | No conflicting CRD versions are installed |

A passing check looks like:

```
► checking prerequisites
✔ Kubernetes 1.29.2 >=1.26.0-0
✔ prerequisites checks passed
```

If any check fails, resolve the issue before proceeding.

> **WARN:** Do not proceed to bootstrap if `flux check --pre` reports failures. A failed bootstrap leaves partial resources in the cluster that can be difficult to clean up.

### 2.2 Verify kubectl Context

Ensure you are connected to the correct cluster:

```bash
# Show the active context
kubectl config current-context

# List all available contexts
kubectl config get-contexts

# Switch context if needed
kubectl config use-context <context-name>
```

---

## 03 · Bootstrap Flux with GitLab

Bootstrapping is the process of installing the Flux controllers on your cluster and linking them to a GitLab repository. Flux writes its own configuration files to the repository and then continuously reconciles the cluster from that source.

### 3.1 Create a GitLab Personal Access Token

Flux requires a GitLab token with permission to create and push to a repository during bootstrap.

1. Log in to GitLab and go to **User Settings > Access Tokens** (or **Profile > Preferences > Access Tokens**).
2. Click **Add new token**.
3. Give it a name, e.g., `flux-bootstrap`.
4. Set an expiry date (recommended: at least 1 year).
5. Select the following scopes:

| Scope | Purpose |
|---|---|
| `api` | Create the repository if it does not exist |
| `read_repository` | Read manifest files |
| `write_repository` | Commit Flux bootstrap files |

6. Click **Create personal access token** and **copy the token immediately** — it will not be shown again.

### 3.2 Export the Token as an Environment Variable

```bash
export GITLAB_TOKEN=<your-personal-access-token>
```

> **WARN:** Never paste your token directly into shell commands that will appear in history. Always export it as an environment variable first.

### 3.3 Bootstrap — GitLab.com (Personal Namespace)

```bash
flux bootstrap gitlab \
  --owner=<your-gitlab-username> \
  --repository=flux-gitops \
  --branch=main \
  --path=clusters/production \
  --token-auth
```

| Flag | Description |
|---|---|
| `--owner` | Your GitLab username or group name |
| `--repository` | Name of the GitLab repository (created if it does not exist) |
| `--branch` | Branch Flux will watch and commit to |
| `--path` | Directory in the repository where Flux writes its config |
| `--token-auth` | Use token-based authentication (instead of SSH) |

### 3.3a Workaround — Manually Create the Repository First

If bootstrap fails with `404 Not Found` when trying to create the repository under your personal namespace, create the repository manually in GitLab first and then re-run bootstrap.

This is a known issue with personal accounts where the GitLab API returns 404 even with a valid token. Pre-creating the repository bypasses the creation step entirely.

**Option A — Create via GitLab UI (recommended):**

1. Go to [https://gitlab.com/projects/new](https://gitlab.com/projects/new).
2. Select **Create blank project**.
3. Set **Project name** to `flux-gitops`.
4. Set **Visibility** to `Private`.
5. Check **Initialize repository with a README** — the repository must have at least one commit.
6. Click **Create project**.

Then re-run bootstrap — Flux will detect the existing repository and skip creation:

```bash
flux bootstrap gitlab \
  --owner=<your-gitlab-username> \
  --repository=flux-gitops \
  --branch=main \
  --path=clusters/production \
  --token-auth
```

**Option B — Create via GitLab API:**

```bash
curl -s --request POST \
  --header "PRIVATE-TOKEN: $GITLAB_TOKEN" \
  --header "Content-Type: application/json" \
  --data '{
    "name": "flux-gitops",
    "visibility": "private",
    "initialize_with_readme": true
  }' \
  https://gitlab.com/api/v4/projects | python3 -m json.tool | grep '"http_url_to_repo"'
```

Confirm the repository URL in the output, then re-run bootstrap:

```bash
flux bootstrap gitlab \
  --owner=<your-gitlab-username> \
  --repository=flux-gitops \
  --branch=main \
  --path=clusters/production \
  --token-auth
```

> **NOTE:** The `initialize_with_readme: true` flag is important. Flux cannot bootstrap into a completely empty repository with no commits — it needs an initialised `main` branch to clone from.

> **TIP:** To confirm your exact GitLab username before running bootstrap, run:
> ```bash
> curl -s --header "PRIVATE-TOKEN: $GITLAB_TOKEN" \
>   https://gitlab.com/api/v4/user | python3 -m json.tool | grep '"username"'
> ```
> The value returned must match exactly what you pass to `--owner`.

---

### 3.4 Bootstrap — GitLab Group Repository

If your manifests live in a group:

```bash
flux bootstrap gitlab \
  --owner=<your-gitlab-group> \
  --repository=flux-gitops \
  --branch=main \
  --path=clusters/production \
  --token-auth
```

### 3.5 Bootstrap — Self-Hosted GitLab

For a self-hosted GitLab instance, add the `--hostname` flag:

```bash
flux bootstrap gitlab \
  --hostname=gitlab.your-company.com \
  --owner=<username-or-group> \
  --repository=flux-gitops \
  --branch=main \
  --path=clusters/production \
  --token-auth
```

### 3.6 What Bootstrap Does

When `flux bootstrap gitlab` runs successfully, it performs the following steps in order:

1. **Creates the GitLab repository** `flux-gitops` if it does not already exist.
2. **Installs Flux controllers** into the `flux-system` namespace on your cluster:
   - `source-controller` — fetches Git repos, Helm charts, OCI artifacts
   - `kustomize-controller` — applies Kustomize manifests to the cluster
   - `helm-controller` — manages Helm releases
   - `notification-controller` — handles alerts and webhooks
3. **Commits bootstrap manifests** to `clusters/production/flux-system/` in the repository.
4. **Creates a `GitRepository` resource** pointing to the repository itself.
5. **Creates a `Kustomization` resource** that applies everything under `clusters/production/`.

After bootstrap, the cluster is self-managing: any change pushed to the repository will be picked up and applied by Flux automatically.

### 3.7 Bootstrap Output

A successful bootstrap produces output similar to:

```
► connecting to https://gitlab.com
► cloning branch "main" from Git repository "https://gitlab.com/<user>/flux-gitops.git"
✔ cloned repository
► generating component manifests
✔ generated component manifests
✔ committed component manifests to "main" ("clusters/production/flux-system/")
► applying component manifests
✔ installed components
✔ reconciled components
► determining the components health
✔ source-controller: deployment ready
✔ kustomize-controller: deployment ready
✔ helm-controller: deployment ready
✔ notification-controller: deployment ready
✔ all components are healthy
```

---

## 04 · Verify the Installation with the Flux CLI

Once bootstrap is complete, use the following commands to confirm Flux is healthy and reconciling correctly.

### 4.1 Full Health Check

```bash
flux check
```

This checks all Flux controllers and CRDs are installed and healthy. All items should show a `✔`.

### 4.2 List All Flux Resources

```bash
# All namespaces
flux get all -A

# Specific namespace
flux get all -n flux-system
```

Expected resources after bootstrap:

| Kind | Name | Description |
|---|---|---|
| `GitRepository` | `flux-system` | Points to your GitLab repository |
| `Kustomization` | `flux-system` | Applies `clusters/production/flux-system/` |

### 4.3 Check GitRepository Status

```bash
flux get source git -A
```

Look for `READY = True` and a recent `LAST FETCHED` timestamp. Example output:

```
NAMESPACE    NAME         REVISION        SUSPENDED  READY  MESSAGE
flux-system  flux-system  main@sha1:abc1  False      True   stored artifact for revision 'main@sha1:abc1'
```

### 4.4 Check Kustomization Status

```bash
flux get kustomization -A
```

Look for `READY = True` and a matching revision. Example:

```
NAMESPACE    NAME         REVISION        SUSPENDED  READY  MESSAGE
flux-system  flux-system  main@sha1:abc1  False      True   Applied revision: main@sha1:abc1
```

### 4.5 Watch Flux Controllers in Real Time

```bash
# Watch all resources update live (Ctrl+C to stop)
flux get all -A --watch
```

### 4.6 View Flux Logs

```bash
# Info-level logs from all controllers
flux logs --level=info

# Follow logs in real time
flux logs --follow --level=info

# Error-level logs only (useful for troubleshooting)
flux logs --level=error

# Logs from a specific controller
flux logs --kind=GitRepository
flux logs --kind=Kustomization
```

### 4.7 Force an Immediate Reconciliation

By default Flux reconciles on a scheduled interval. To trigger an immediate sync:

```bash
# Reconcile the source (fetch latest from GitLab)
flux reconcile source git flux-system

# Reconcile the kustomization (re-apply manifests)
flux reconcile kustomization flux-system

# Do both in one command
flux reconcile kustomization flux-system --with-source
```

### 4.8 Verify Flux Pods Are Running

Use `kubectl` to confirm all Flux pods are in the `Running` state:

```bash
kubectl get pods -n flux-system
```

Expected output:

```
NAME                                       READY   STATUS    RESTARTS   AGE
helm-controller-xxxxxxxxx-xxxxx            1/1     Running   0          2m
kustomize-controller-xxxxxxxxx-xxxxx       1/1     Running   0          2m
notification-controller-xxxxxxxxx-xxxxx    1/1     Running   0          2m
source-controller-xxxxxxxxx-xxxxx          1/1     Running   0          2m
```

---

## Validation Checklist

| Status | Validation Step |
|---|---|
| ☐ | `flux version` returns v2.0 or later |
| ☐ | `flux check --pre` passes all checks |
| ☐ | GitLab repository `flux-gitops` exists with `clusters/production/flux-system/` committed |
| ☐ | `flux check` shows all controllers healthy |
| ☐ | `kubectl get pods -n flux-system` shows all 4 pods `Running` |
| ☐ | `flux get source git -A` shows `READY True` |
| ☐ | `flux get kustomization -A` shows `READY True` |
| ☐ | `flux logs --level=error` shows no errors |

---

## Troubleshooting Guide

| Symptom | Resolution |
|---|---|
| `flux check --pre` fails on RBAC | Ensure your `kubectl` context has cluster-admin or equivalent permissions. Run: `kubectl auth can-i create namespaces --all-namespaces` |
| Bootstrap fails with `404 Not Found` on repository creation | Personal namespace API restriction. Pre-create the repository manually in GitLab UI or via API with `initialize_with_readme: true`, then re-run bootstrap. See Section 3.3a. |
| Bootstrap fails with `401 Unauthorized` | Verify `$GITLAB_TOKEN` is exported and has `api`, `read_repository`, and `write_repository` scopes. |
| Bootstrap fails with `403 Forbidden` | The token may lack permission to create repositories. Check group-level access if using a group namespace. |
| `flux get source git` shows `READY False` | Check the Kubernetes secret `flux-system` in the `flux-system` namespace contains valid credentials. Re-run bootstrap or recreate the secret. |
| Pods stuck in `CrashLoopBackOff` | Run `kubectl logs -n flux-system deploy/source-controller` to inspect startup errors. |
| Bootstrap hangs indefinitely | Check cluster connectivity: `kubectl cluster-info`. Ensure the API server is reachable. |
| `flux check` shows missing CRDs | A previous partial install may have left stale state. Run `flux uninstall` then bootstrap again. |

---

## Quick Reference — Flux CLI Commands

### Installation

```bash
# Install latest (Linux/macOS)
curl -s https://fluxcd.io/install.sh | sudo bash

# Install via Homebrew (macOS)
brew install fluxcd/tap/flux

# Verify version
flux version
```

### Pre-flight & Health

```bash
flux check --pre          # Pre-flight checks only
flux check                # Full health check (post-install)
```

### Bootstrap

```bash
# GitLab.com
flux bootstrap gitlab \
  --owner=<user-or-group> \
  --repository=<repo-name> \
  --branch=main \
  --path=clusters/production \
  --token-auth

# Self-hosted GitLab
flux bootstrap gitlab \
  --hostname=<gitlab.hostname.com> \
  --owner=<user-or-group> \
  --repository=<repo-name> \
  --branch=main \
  --path=clusters/production \
  --token-auth
```

### Inspect Resources

```bash
flux get all -A                        # All Flux resources, all namespaces
flux get source git -A                 # GitRepository sources
flux get kustomization -A              # Kustomizations
flux get helmrelease -A                # Helm releases
flux get image repository -A           # Image repositories (if image automation is used)
```

### Reconcile

```bash
flux reconcile source git flux-system              # Fetch latest from Git
flux reconcile kustomization flux-system           # Re-apply manifests
flux reconcile kustomization flux-system --with-source   # Both at once
```

### Suspend & Resume

```bash
flux suspend kustomization flux-system     # Pause reconciliation (for maintenance)
flux resume kustomization flux-system      # Resume reconciliation
```

### Logs

```bash
flux logs --level=info                   # All info logs
flux logs --level=error                  # Errors only
flux logs --follow --level=info          # Live tail
flux logs --kind=GitRepository           # Logs for a specific resource kind
```

### Export & Uninstall

```bash
flux export kustomization flux-system    # Export resource as YAML
flux uninstall                           # Remove all Flux components from cluster
```

---

## Key Documentation Links

- **FluxCD installation docs:** https://fluxcd.io/flux/installation/
- **Flux CLI reference:** https://fluxcd.io/flux/cmd/
- **GitLab bootstrap guide:** https://fluxcd.io/flux/installation/bootstrap/gitlab/
- **Flux releases (GitHub):** https://github.com/fluxcd/flux2/releases
- **Flux supported Kubernetes versions:** https://fluxcd.io/flux/installation/#prerequisites

---

*Lab 1 Complete · Advanced GitOps Workshop · FluxCD + GitLab*
