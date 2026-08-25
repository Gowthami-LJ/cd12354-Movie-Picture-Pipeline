# Movie Picture Pipeline

CI/CD pipeline for a two-part Movie Picture catalog application, built with GitHub Actions,
Docker, Amazon ECR, and Amazon EKS.

- **Frontend** — React (TypeScript), served from `starter/frontend`
- **Backend** — Flask (Python), served from `starter/backend`

## Architecture

```
GitHub PR/Push
      │
      ▼
GitHub Actions (lint → test → build → [deploy])
      │
      ▼
Amazon ECR (Docker image storage)
      │
      ▼
Amazon EKS (Kubernetes cluster, LoadBalancer services)
```

## Workflows

| File | Trigger | Purpose |
|---|---|---|
| `.github/workflows/frontend-ci.yaml` | PR to `main` (frontend changes) + manual | Lint, test, build the frontend |
| `.github/workflows/backend-ci.yaml` | PR to `main` (backend changes) + manual | Lint, test, build the backend |
| `.github/workflows/frontend-cd.yaml` | Push to `main` (frontend changes) + manual | Lint, test, build, push to ECR, deploy to EKS |
| `.github/workflows/backend-cd.yaml` | Push to `main` (backend changes) + manual | Lint, test, build, push to ECR, deploy to EKS |

Each CI pipeline runs lint and test jobs in parallel, then builds only if both succeed
(`needs: [lint, test]`). Each CD pipeline additionally tags the Docker image with the
Git commit SHA, pushes it to ECR, and deploys it to the EKS cluster via `kustomize`.

## Infrastructure

Provisioned with Terraform (`setup/terraform`): a VPC with public/private subnets, an
EKS cluster (Kubernetes 1.32) with one managed node group, two ECR repositories
(`frontend`, `backend`), and the IAM roles/user (`github-action-user`) that GitHub
Actions uses to authenticate to AWS.

```bash
cd setup/terraform
terraform init
terraform apply
```

After applying, grant the GitHub Actions IAM user `kubectl` access:

```bash
cd setup
./init.sh
```

## Required GitHub configuration

**Secrets** (Settings → Secrets and variables → Actions → Secrets):
- `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` — credentials for `github-action-user`

**Variables** (Settings → Secrets and variables → Actions → Variables):
- `REACT_APP_MOVIE_API_URL` — the deployed backend's LoadBalancer URL, so the frontend
  is built pointing at the live backend rather than `localhost`

## Notable fixes made during setup

The starter template needed a few updates to run on current infrastructure:

- **`starter/backend/Dockerfile`** — added `ENV CFLAGS="-Wno-error=incompatible-pointer-types"`
  before the `pipenv install` step. Newer GCC versions in the Alpine base image treat this
  warning as a hard error, which broke building the pinned `uwsgi` package.
- **Backend CI/CD workflows** — changed `pipenv install` to `pipenv install --dev`, since
  `flake8` is listed under `[dev-packages]` in the `Pipfile` and is skipped by default.
- **`setup/terraform/variables.tf`** — `k8s_version` set to `1.32`. Amazon EKS retired
  Kubernetes 1.25; 1.32 is the newest version that still supports Amazon Linux 2 (AL2)
  worker nodes. Kubernetes 1.33+ requires AL2023, whose node bootstrap process calls
  `ec2:DescribeInstances`, an API blocked by an organization-level Service Control Policy
  in this AWS sandbox account — so 1.32/AL2 was used instead.

## Verifying the deployment

```bash
kubectl get svc
```

Then visit the `frontend` service's `EXTERNAL-IP` in a browser, and hit
`http://<backend-EXTERNAL-IP>/movies` directly to confirm the API response.

## Deployed Application URLs

The applications were successfully deployed to Amazon EKS using the GitHub Actions CD pipelines.

### Frontend

http://af24cb7fabac942a4852cac9455369c4-1091898408.us-east-1.elb.amazonaws.com/

### Backend API

http://a2791ea8c23674e848359faf29707519-1539298759.us-east-1.elb.amazonaws.com/movies

The backend `/movies` endpoint returns the list of movies, and the frontend retrieves the movie data from the deployed backend service.

## Tearing down

To avoid ongoing AWS charges once finished:

```bash
kubectl delete svc frontend backend
cd setup/terraform
terraform destroy
```