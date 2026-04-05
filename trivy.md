# GitLab CI/CD — Build, Push & Trivy Security Scan Lab
### Registry: GitLab Container Registry | Project: cicd

---

## Overview

```
Push to main   →   Stage 1: Build & Push image   →   Stage 2: Trivy scan image
                   GitLab Container Registry           CVE vulnerability report
```

**Two-stage pipeline:**

| Stage | Job | What it does |
|-------|-----|--------------|
| `build` | `docker_build` | Builds Flask image, pushes to GitLab registry |
| `scan` | `trivy_scan` | Pulls image, runs Trivy CVE vulnerability scan |

---

## Project Files

### app.py

```python
from flask import Flask
app = Flask(__name__)

@app.route('/')
def hello_world():
    return 'Hello, World!'

if __name__ == '__main__':
    app.run(debug=True, host='0.0.0.0', port=8080)
```

---

### requirements.txt

```
Flask==3.1.0
```

---

### dockerfile

```dockerfile
FROM python:3.9-alpine
WORKDIR /flask_app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
RUN pip install pytest
EXPOSE 8080
COPY app.py .
CMD [ "python", "app.py" ]
```

---

## Step 1 — GitLab CI/CD Variables

GitLab auto-provides these variables — no manual setup needed:

| Variable | Auto-provided | Value (example) |
|----------|--------------|-----------------|
| `CI_REGISTRY` | ✓ | `registry.gitlab.com` |
| `CI_REGISTRY_IMAGE` | ✓ | `registry.gitlab.com/myuser/cicd` |
| `CI_REGISTRY_USER` | ✓ | Your GitLab username |
| `CI_REGISTRY_PASSWORD` | ✓ | Your GitLab PAT |
| `CI_JOB_TOKEN` | ✓ | Short-lived token per job (used in scan stage) |

> Go to **Project → Settings → CI/CD → Variables** only if you need to
> override `CI_REGISTRY_USER` / `CI_REGISTRY_PASSWORD` with a PAT.
> For most cases the auto-provided variables work out of the box.

---

## Step 2 — Create .gitlab-ci.yml

```yaml
stages:
  - build
  - scan

variables:
  IMAGE_TAG: "latest"
  IMAGE_NAME: "$CI_REGISTRY_IMAGE:$IMAGE_TAG"   # registry.gitlab.com/myuser/cicd:latest

# ─────────────────────────────────────────
# Stage 1: Build and push image
# ─────────────────────────────────────────
docker_build:
  stage: build
  image: docker:latest
  services:
    - docker:dind
  variables:
    DOCKER_TLS_CERTDIR: ""       # Disable TLS for docker:dind
  before_script:
    - echo "Logging into GitLab Container Registry..."
    - docker login -u "$CI_REGISTRY_USER" -p "$CI_REGISTRY_PASSWORD" "$CI_REGISTRY"
  script:
    - echo "Building image $IMAGE_NAME ..."
    - docker build -t "$IMAGE_NAME" .
    - echo "Pushing image to registry..."
    - docker push "$IMAGE_NAME"
  only:
    - main

# ─────────────────────────────────────────
# Stage 2: Trivy vulnerability scan
# ─────────────────────────────────────────
trivy_scan:
  stage: scan
  image: docker:latest
  before_script:
    - echo "Installing Trivy..."
    - apk add --no-cache curl bash
    - curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sh -s -- -b /usr/local/bin
    - echo "Logging into registry with CI_JOB_TOKEN..."
    - echo "$CI_JOB_TOKEN" | docker login -u gitlab-ci-token --password-stdin "$CI_REGISTRY"
  script:
    - echo "Scanning image $IMAGE_NAME ..."
    - trivy image "$IMAGE_NAME"
  only:
    - main
```

---

## Step 3 — Push All Files to GitLab

```bash
git clone https://gitlab.com/<YOUR_USERNAME>/cicd.git
cd cicd

# Add all files
touch app.py requirements.txt dockerfile .gitlab-ci.yml
# paste content into each file

git add .
git commit -m "Add Flask app with build and Trivy scan pipeline"
git push origin main
```

---

## Step 4 — Pipeline Execution Flow

### Stage 1 — docker_build

```
$ docker login -u "$CI_REGISTRY_USER" -p "$CI_REGISTRY_PASSWORD" registry.gitlab.com
Login Succeeded

$ docker build -t registry.gitlab.com/myuser/cicd:latest .
Step 1/7 : FROM python:3.9-alpine
Step 2/7 : WORKDIR /flask_app
...
Successfully built a1b2c3d4e5f6
Successfully tagged registry.gitlab.com/myuser/cicd:latest

$ docker push registry.gitlab.com/myuser/cicd:latest
latest: digest: sha256:abc123... size: 12345
Job succeeded
```

### Stage 2 — trivy_scan

```
$ apk add --no-cache curl bash
$ curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sh -s -- -b /usr/local/bin
trivy installed

$ echo "$CI_JOB_TOKEN" | docker login -u gitlab-ci-token --password-stdin registry.gitlab.com
Login Succeeded

$ trivy image registry.gitlab.com/myuser/cicd:latest

registry.gitlab.com/myuser/cicd:latest (alpine 3.18.0)
================================================
Total: 3 (UNKNOWN: 0, LOW: 2, MEDIUM: 1, HIGH: 0, CRITICAL: 0)

┌──────────────┬────────────────┬──────────┬────────┐
│   Library    │ Vulnerability  │ Severity │  Fix   │
├──────────────┼────────────────┼──────────┼────────┤
│ libssl       │ CVE-2023-xxxx  │ MEDIUM   │ 3.1.x  │
│ busybox      │ CVE-2022-xxxx  │ LOW      │ none   │
└──────────────┴────────────────┴──────────┴────────┘
Job succeeded
```

---

## Step 5 — Verify in GitLab UI

**Check pipeline:**
Project → **CI/CD** → **Pipelines** → click the latest pipeline

**Check image was pushed:**
Project → **Deploy** → **Container Registry** → `cicd:latest`

**Check Trivy scan results:**
Click `trivy_scan` job → view full CVE report in job logs

---

## Optional Enhancements

### Fail pipeline on HIGH or CRITICAL vulnerabilities

```yaml
trivy_scan:
  script:
    - trivy image --exit-code 1 --severity HIGH,CRITICAL "$IMAGE_NAME"
```

> `--exit-code 1` makes the job fail if HIGH or CRITICAL CVEs are found.
> Remove `--exit-code 1` to always pass (report only).

### Save Trivy report as artifact

```yaml
trivy_scan:
  script:
    - trivy image --format json --output trivy-report.json "$IMAGE_NAME"
    - trivy image "$IMAGE_NAME"
  artifacts:
    paths:
      - trivy-report.json
    expire_in: 7 days
```

### Tag image with Git commit SHA instead of latest

```yaml
variables:
  IMAGE_TAG: "$CI_COMMIT_SHORT_SHA"    # e.g. a1b2c3d
  IMAGE_NAME: "$CI_REGISTRY_IMAGE:$IMAGE_TAG"
```

---

## File Structure

```
cicd/
├── app.py
├── requirements.txt
├── dockerfile
└── .gitlab-ci.yml
```

---

## Troubleshooting

| Error | Fix |
|-------|-----|
| `unauthorized` on docker login | Check `CI_REGISTRY_USER` / `CI_REGISTRY_PASSWORD` in project CI/CD variables |
| `Cannot connect to Docker daemon` | Add `DOCKER_TLS_CERTDIR: ""` and `docker:dind` service |
| `trivy: command not found` | Ensure `apk add curl bash` runs before install script |
| Trivy can't pull image in scan stage | Use `CI_JOB_TOKEN` with `gitlab-ci-token` user — not `CI_REGISTRY_PASSWORD` |
| Pipeline only runs on `main` | The `only: - main` rule skips all other branches by design |
