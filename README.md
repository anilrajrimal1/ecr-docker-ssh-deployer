# 🚀 Deploy Docker Image GitHub Action

> Pulls and Deploys a Docker image to a remote server using **SSH** and **Docker Compose**, with support for **Amazon ECR**, **environmental secrets from Phase**, and GitHub Actions CI/CD.

[![GitHub release (latest by date)](https://img.shields.io/github/v/release/anilrajrimal1/ecr-docker-ssh-deployer)](https://github.com/anilrajrimal1/ecr-docker-ssh-deployer/releases)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

### 🧾 Description
This custom GitHub Action pulls Docker images from Amazon ECR and deploys them to a remote server over SSH. It uses `docker compose` and supports environment-specific secrets fetched from [Phase](https://console.phase.dev) — enabling smooth and secure production or staging deployments.


### 📥 Inputs

| Name | Required | Description |
|------|----------|-------------|
| `SSH_PRIVATE_KEY` | ✅ | SSH private key to connect to the deployment server |
| `AWS_ACCESS_KEY_ID` | ✅ | AWS access key for ECR login |
| `AWS_SECRET_ACCESS_KEY` | ✅ | AWS secret key for ECR login |
| `AWS_REGION` | ✅ | AWS region for ECR |
| `ECR_REGISTRY` | ✅ | Full ECR registry URL |
| `ECR_REPOSITORY` | ✅ | ECR repository name |
| `DEPLOYMENT_SERVER_IP` | ✅ | IP address of the deployment target server |
| `DEPLOYMENT_SERVER_USERNAME` | ✅ | SSH username (default: `ubuntu`) |
| `PHASE_SERVICE_TOKEN` | ✅ | Phase Service Token |
| `PHASE_APP_ID` | ✅ | Unique identifier for the app in Phase |
| `BRANCH_NAME` | ✅ | Git branch name (used for env + compose file) |
| `PHASE_HOST` | ❌ | Phase host URL, only for self-hosted environments |


### ⚙️ How It Works

1. Sets up SSH credentials and ECR access.
2. Logs in to AWS ECR.
3. Fetches secrets and environment variables from **Phase** using `phase-secrets-fetch-action`.
4. Copies branch-specific Docker Compose file.
5. Executes `docker compose pull` and `docker compose up -d` on the remote server over SSH.


### 🛠 Usage Example

```yaml
name: Deploy to Server

on:
  workflow_dispatch:
  workflow_run:
    workflows: ["Build and Push Image to ECR"]
    types:
      - completed

env:
  ECR_REGISTRY: aws_id.dkr.ecr.your-region.amazonaws.com
  ECR_REPOSITORY: demo-django-backend
  AWS_REGION: your-region
  PHASE_HOST: https://phase.your-domain.com

jobs:
  deploy:
    runs-on: self-hosted
    if: github.event_name == 'workflow_dispatch' || (github.event_name == 'workflow_run' && github.event.workflow_run.conclusion == 'success')
    steps:
      - name: Extract Branch Name
        id: extract_branch
        run: |
          echo "branch=${GITHUB_HEAD_REF:-${GITHUB_REF#refs/heads/}}" >> $GITHUB_OUTPUT

      - name: Get VM SSH host and user
        id: get_vm_conf
        run: |
          case "${{ github.ref }}" in
          refs/heads/develop)
            echo "SERVER_IP=123.45.6.7" >> $GITHUB_OUTPUT
            echo "SERVER_USERNAME=ubuntu" >> $GITHUB_OUTPUT
            ;;
          esac

      - name: Deploy to VM
        uses: anilrajrimal1/ecr-docker-ssh-deployer@v1.0.0
        with:
          BRANCH_NAME: ${{ steps.extract_branch.outputs.branch }}
          SSH_PRIVATE_KEY: ${{ secrets.SSH_PRIVATE_KEY }}
          ECR_REGISTRY: ${{ env.ECR_REGISTRY }}
          ECR_REPOSITORY: ${{ env.ECR_REPOSITORY }}
          AWS_REGION: ${{ env.AWS_REGION }}
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          DEPLOYMENT_SERVER_IP: ${{ steps.get_vm_conf.outputs.SERVER_IP }}
          DEPLOYMENT_SERVER_USERNAME: ${{ steps.get_vm_conf.outputs.SERVER_USERNAME }}
          PHASE_SERVICE_TOKEN: ${{ secrets.PHASE_SERVICE_TOKEN }}
          PHASE_APP_ID: ${{ secrets.PHASE_APP_ID }}
          PHASE_HOST: ${{ env.PHASE_HOST }}
```


### 📝 Notes

- This action uses my [Phase Secrets Fetch Action](https://github.com/anilrajrimal1/phase-secrets-fetch-action) to fetch `.env` files dynamically.
- It assumes your **docker-compose file path** follows this format: `./docker/server/docker-compose.<branch>.yml`
- SSH access must be whitelisted and key-based (no password support).
- Be sure to store all sensitive credentials like SSH keys and AWS secrets in **GitHub Secrets**.

---
