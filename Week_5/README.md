# Week 5 — CI/CD Pipeline with GitHub Actions
Automate Docker image builds and deployments using GitHub Actions

## Architecture
```
Developer pushes to main
        ↓
GitHub Actions
        ↓
┌───────────────────────┐
│  Job 1: Build & Push  │
│  - Build backend      │
│  - Build frontend     │
│  - Push to DockerHub  │
└───────────┬───────────┘
            │ needs: build-and-push
┌───────────▼───────────┐
│  Job 2: Deploy        │
│  - SCP docker-compose │
│  - SSH into EC2       │
│  - docker compose pull│
│  - docker compose up  │
└───────────────────────┘
        ↓
EC2 (abdul-cicd-server)
├── ecommerce_frontend (Port 3000)
├── ecommerce_backend  (Port 5000)
└── ecommerce_db       (Port 5432)
```

## Tech Stack
- CI/CD: GitHub Actions
- Registry: DockerHub
- Deployment Target: AWS EC2 
- Containerization: Docker

## What I Did

### 1. Set Up EC2 Deployment Server
Launched a fresh EC2 instance and installed Docker:
```bash
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
sudo usermod -aG docker ubuntu
sudo apt install docker-compose-plugin -y
```
Adding ubuntu to the docker group means Docker runs without sudo
allowing for GitHub Actions to run docker commands over the SSH.

### 2. Created DockerHub Repositories
Created two public repositories on DockerHub:
- `tuucay4/ecommerce-backend`
- `tuucay4/ecommerce-frontend`

Generated a DockerHub Access Token for GitHub Actions

### 3. Configured GitHub Secrets
Stored all sensitive values in GitHub Secrets so they never

| Secret | Function |
|---|---|
| `DOCKERHUB_USERNAME` | DockerHub login |
| `DOCKERHUB_TOKEN` | DockerHub access token |
| `EC2_HOST` | EC2 public IP address |
| `EC2_USER` | EC2 login user |
| `EC2_SSH_KEY` | .pem key file |

### 4. Set Up Branching Strategy
Created a feature branch for all CI/CD work:
```bash
git checkout -b feature/week5-cicd
```
Feature branch pushes do not trigger deployment.
Only merging to main triggers the pipeline.

### 5. Wrote the GitHub Actions Workflow
Created `.github/workflows/deploy.yml` at the root of the repo.

**Trigger — pipeline runs when:**
- Code is pushed to `main` branch
- Files inside `Week_5/` or `.github/workflows/` are changed
- and also Manual triggers

```yaml
on:
  push:
    branches: [main]
    paths:
      - 'Week_5/**'
      - '.github/workflows/**'
  workflow_dispatch:
```

**Job 1 — Build and Push:**
Uses `docker/build-push-action@v5` with Docker to build both the 
frontend and backend images and push to DockerHub:
```yaml
- name: Build and push backend image
  uses: docker/build-push-action@v5
  with:
    context: ./Week_5/backend
    dockerfile: ./Week_5/backend/dockerfile
    push: true
    tags: tuucay4/ecommerce-backend:latest
```

**Job 2 — Deploy:**
Only runs if Job 1 succeeds (`needs: build-and-push`).
Copies docker-compose.yml to EC2 via SCP, then SSH in to
pull latest images and restart containers:
```bash
docker compose pull
docker compose up -d --remove-orphans
docker image prune -f
```

### 6. Tested the Full Pipeline
Merged feature branch to main via Pull Request on GitHub.
Watched the pipeline run in the Actions tab — both jobs
completed successfully. App loaded correctly on EC2.

## Key Concepts Learned

### GitHub Actions Workflow Structure
```
Workflow (deploy.yml)
└── Job 1: build-and-push
│   ├── Step: checkout code
│   ├── Step: login to DockerHub
│   ├── Step: setup buildx
│   ├── Step: build & push backend
│   └── Step: build & push frontend
└── Job 2: deploy
    ├── Step: checkout code
    ├── Step: scp docker-compose to EC2
    └── Step: ssh and deploy
```

### actions/checkout@v3
GitHub runners are blank servers — they have nothing on them.
`checkout` downloads your repository onto the runner so subsequent
steps have access to your code and Dockerfiles.

### Docker Buildx
Extended Docker build system required by `build-push-action`.
Provides better layer caching and multi-platform build support.
Set up once with `setup-buildx-action` before building.

### @v3, @v5 Version Pinning
Every GitHub Action is code living in a GitHub repository.
Pinning to a specific version (e.g. `@v3`) prevents breaking
changes from newer releases affecting your pipeline same
principle as pinning package versions in package.json.

### needs: build-and-push
Creates a dependency between jobs. Deploy only runs if Build
succeeds. If the image build fails, broken code never reaches
the server.

### .env and Secrets
`.env` is gitignored and never reaches GitHub. In CI/CD,
sensitive values are stored as GitHub Secrets and either
generated on the server or passed as environment variables
during deployment.
### paths: filter
Prevents the pipeline from triggering on unrelated changes.
Editing a README in Week_1 won't deploy Week_5 — only changes
inside the specified paths trigger the workflow.

## Branching Strategy
```
main                → production (triggers deployment on push)
feature/week5-cicd  → development (no deployment triggered)
```
All work happens on feature branches. Pull Requests to main
trigger code review before deployment. Only tested, reviewed
code reaches the production server.

## Errors Encountered & Fixed
| Error | Cause | Fix |
|---|---|---|
| .env not copying via SCP | .env is gitignored, never in repo | Created .env manually on EC2 |