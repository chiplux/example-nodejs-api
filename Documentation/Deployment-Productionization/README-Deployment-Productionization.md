# Production Deployment Guide (GitHub Actions)

This document describes the automated deployment process for the Node.js API and PostgreSQL database using GitHub Actions and Docker Compose.

## 🚀 Overview

The deployment pipeline is designed to automatically deploy changes to the production server whenever code is pushed to the `main` branch. It can also be triggered manually.

**Mechanism:**
1.  **Trigger**: Push to `main` branch or manual `workflow_dispatch`.
2.  **Connection**: SSH into the production server.
3.  **Sync**: Pull latest code from GitHub.
4.  **Environment**: Create/update `.env` file from GitHub Secrets.
5.  **Provision**: Build and restart containers using `docker compose`.
6.  **Cleanup**: Remove unused Docker images and build cache to save disk space.

---

## 🛠️ Prerequisites

Before the deployment can work, the production server must have:
- **Docker** and **Docker Compose** (V2) installed.
- **Git** installed and configured with SSH keys to pull from the repository.
- The repository initialy cloned at `~/example-nodejs-api`.

---

## 🔐 GitHub Secrets Setup

The following secrets must be configured in your GitHub repository (**Settings > Secrets and variables > Actions**):

| Secret Name   | Description              | Example Value                        |
| :------------ | :----------------------- | :----------------------------------- |
| `SSH_HOST`    | Server IP or Hostname    | `203.0.113.10`                       |
| `SSH_USER`    | SSH Username             | `ubuntu`                             |
| `SSH_KEY`     | Private SSH Key          | `-----BEGIN RSA PRIVATE KEY-----...` |
| `DB_USER`     | PostgreSQL Username      | `citiesdb`                           |
| `DB_PASSWORD` | PostgreSQL Password      | `********`                           |
| `DB_NAME`     | PostgreSQL Database Name | `citiesdb`                           |
| `DB_HOST`     | Database host (Docker)   | `db`                                 |
| `DB_PORT`     | Database port            | `5432`                               |
| `PORT`        | Application Port         | `3000`                               |

---

## ⚡ How to Trigger

### 1. Automatic Deployment
Simply push your changes to the `main` branch:
```bash
git add .
git commit -m "Your message"
git push origin main
```

### 2. Manual Deployment
1. Go to your repository on GitHub.
2. Click on the **Actions** tab.
3. Select **Deploy to Production** from the sidebar.
4. Click the **Run workflow** dropdown and then **Run workflow**.

---

## 🧹 Maintenance & Cleanup

Currently, the image build directly on the production server. It is not the best practice, but it is the simplest way to deploy the application and to reduce the cost.

The deployment script includes automatic cleanup commands to prevent the server from running out of disk space, we use the following commands:
:

```bash
docker image prune -af
docker builder prune -af
```
- `image prune -af`: Deletes all unused images, not just dangling ones.
- `builder prune -af`: Clears the Docker build cache.

---

## 🔍 Troubleshooting

- **Permission Denied (publickey)**: Ensure the production server's public key (usually `~/.ssh/id_rsa.pub`) is added to your GitHub **Deploy Keys** or account settings.
- **Docker Compose Found**: Ensure you are using `docker compose` (V2). Older installations may use `docker-compose` (V1).
- **Initialization Issues**: If `init_db.sql` is updated but changes aren't reflected, you may need to delete the database volume on the server: `docker compose down -v`. **CAUTION: This will delete all data in the database.**

## Example Deployment

### Example of successful deployment
https://github.com/chiplux/example-nodejs-api/actions/runs/22043660904/job/63688655853

![alt text](image.png)

### Example of secret variables
![alt text](image-1.png)

## Deployed Website

![alt text](image-3.png)

![alt text](image-2.png)