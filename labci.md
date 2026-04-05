# GitLab CI/CD Workshop
### Build & Push Flask App to Docker Hub

---

## Overview

```
Push code to GitLab   →   GitLab CI builds image   →   Push to Docker Hub
main branch                .gitlab-ci.yml                myuser/myapp:latest
```

---

## Step 1 — Create GitLab Project

1. Go to **gitlab.com** → **New Project** → **Create blank project**
2. Project name: `cicd`
3. Visibility: Private (or Public)
4. Click **Create project**

---

## Step 2 — Create Project Files

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

## Step 3 — Create Docker Hub Access Token

1. Log in to **hub.docker.com**
2. Click your avatar → **Account Settings**
3. Go to **Security** → **New Access Token**
4. Token description: `gitlab-cicd`
5. Access permissions: **Read & Write**
6. Click **Generate** and copy the token — it won't be shown again

---

## Step 4 — Add CI/CD Variables in GitLab

Go to your `cicd` project → **Settings** → **CI/CD** → **Variables** → **Add variable**

| Key | Value | Protected | Masked |
|-----|-------|-----------|--------|
| `DOCKERHUB_USERNAME` | Your Docker Hub username | ✓ | ✗ |
| `DOCKERHUB_TOKEN` | Your Docker Hub access token | ✓ | ✓ |

> **Masked** hides the value from CI logs.
> **Protected** restricts the variable to protected branches only (e.g. `main`).

---

## Step 5 — Create .gitlab-ci.yml

```yaml
stages:
  - build

variables:
  DOCKERFILE: "dockerfile"
  REGISTRY: "docker.io"
  IMAGE_NAME: "<YOUR_DOCKERHUB_USERNAME>/myapp"   # e.g. johndoe/myapp
  IMAGE_TAG: "latest"
  DOCKER_HOST: tcp://docker:2375/
  DOCKER_TLS_CERTDIR: ""

build_docker_image:
  stage: build
  image: docker:latest
  services:
    - docker:dind
  before_script:
    - echo "Logging into Docker Hub..."
    - echo "$DOCKERHUB_TOKEN" | docker login $REGISTRY -u "$DOCKERHUB_USERNAME" --password-stdin
  script:
    - echo "Building Docker image..."
    - docker build -f $DOCKERFILE -t $REGISTRY/$IMAGE_NAME:$IMAGE_TAG .
    - echo "Pushing Docker image..."
    - docker push $REGISTRY/$IMAGE_NAME:$IMAGE_TAG
  only:
    - main
```

---

## Step 6 — Push All Files to GitLab

```bash
# Clone your new GitLab project
git clone https://gitlab.com/<YOUR_USERNAME>/cicd.git
cd cicd

# Add all files
cp /path/to/app.py .
cp /path/to/requirements.txt .
cp /path/to/dockerfile .
cp /path/to/.gitlab-ci.yml .

# Commit and push
git add .
git commit -m "Initial commit: Flask app with Docker Hub CI/CD"
git push origin main
```

---

## Step 7 — Verify Pipeline

1. Go to your GitLab project → **CI/CD** → **Pipelines**
2. Click the running pipeline to see live logs
3. The `build_docker_image` job should:
   - Log in to Docker Hub
   - Build the Flask image
   - Push `myuser/myapp:latest` to Docker Hub

```
$ echo "$DOCKERHUB_TOKEN" | docker login docker.io -u "$DOCKERHUB_USERNAME" --password-stdin
Login Succeeded

$ docker build -f dockerfile -t docker.io/myuser/myapp:latest .
Successfully built abc123def456
Successfully tagged docker.io/myuser/myapp:latest

$ docker push docker.io/myuser/myapp:latest
latest: digest: sha256:... size: 1234
```

---

## Step 8 — Verify Image on Docker Hub

1. Go to **hub.docker.com** → **Repositories**
2. You should see `myuser/myapp` with tag `latest`

Or pull and test locally:

```bash
docker pull <YOUR_DOCKERHUB_USERNAME>/myapp:latest
docker run -p 8080:8080 <YOUR_DOCKERHUB_USERNAME>/myapp:latest

# Test it
curl http://localhost:8080
# Hello, World!
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
| `unauthorized: incorrect username or password` | Check `DOCKERHUB_USERNAME` and `DOCKERHUB_TOKEN` variables in GitLab CI/CD settings |
| `Cannot connect to the Docker daemon` | Ensure `DOCKER_HOST: tcp://docker:2375/` and `docker:dind` service are set |
| `DOCKER_TLS_CERTDIR` warning | Set `DOCKER_TLS_CERTDIR: ""` to disable TLS for dind |
| Pipeline not triggered | Check the branch is `main` — the `only: - main` rule skips other branches |
| Image not found on Docker Hub | Verify `IMAGE_NAME` matches your Docker Hub username exactly |
