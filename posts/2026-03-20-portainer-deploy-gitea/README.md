# How to Deploy Gitea via Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Gitea, Git, Self-Hosted, DevOps

Description: Deploy Gitea via Portainer as a lightweight self-hosted Git service with an intuitive web interface, low resource requirements, and optional CI/CD with Gitea Actions.

## Introduction

Gitea is a lightweight, self-hosted Git service written in Go. It uses far fewer resources than GitLab (as little as 100MB RAM) while providing most features teams need: repositories, pull requests, issues, wikis, and CI/CD via Gitea Actions. Deploy it via Portainer for a fast, private Git server.

## Deploy as a Stack

```yaml
version: "3.8"

services:
  gitea:
    image: gitea/gitea:latest
    container_name: gitea
    environment:
      - USER_UID=1000
      - USER_GID=1000
      - GITEA__database__DB_TYPE=postgres
      - GITEA__database__HOST=gitea-db:5432
      - GITEA__database__NAME=gitea
      - GITEA__database__USER=gitea
      - GITEA__database__PASSWD=gitea_password
      - GITEA__server__DOMAIN=git.example.com
      - GITEA__server__HTTP_PORT=3000
      - GITEA__server__SSH_PORT=22
      - GITEA__service__DISABLE_REGISTRATION=false
      - GITEA__mailer__ENABLED=true
      - GITEA__mailer__SMTP_ADDR=smtp.example.com
      - GITEA__mailer__SMTP_PORT=587
      - GITEA__mailer__FROM=gitea@example.com
      - GITEA__mailer__USER=gitea@example.com
      - GITEA__mailer__PASSWD=smtp_password
    volumes:
      - gitea_data:/data
      - /etc/timezone:/etc/timezone:ro
      - /etc/localtime:/etc/localtime:ro
    ports:
      - "3000:3000"    # Web UI
      - "222:22"       # SSH (using 222 to avoid conflict)
    depends_on:
      - gitea-db
    restart: unless-stopped

  gitea-db:
    image: postgres:16-alpine
    container_name: gitea-db
    environment:
      POSTGRES_DB: gitea
      POSTGRES_USER: gitea
      POSTGRES_PASSWORD: gitea_password
    volumes:
      - gitea_db:/var/lib/postgresql/data
    restart: unless-stopped
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U gitea"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  gitea_data:
  gitea_db:
```

## Initial Setup

Navigate to `http://<host>:3000`. The first visit presents the installation wizard — most settings are pre-filled from environment variables. Set an admin username and password, then click **Install Gitea**.

## SSH Access Configuration

Add to `~/.ssh/config` on your client:

```
Host git.example.com
    HostName git.example.com
    Port 222
    User git
    IdentityFile ~/.ssh/id_rsa
```

Then push using:

```bash
git remote add origin ssh://git@git.example.com:222/username/repo.git
```

## Gitea Actions CI/CD

Gitea 1.19+ includes Gitea Actions (GitHub Actions compatible). Create `.gitea/workflows/ci.yml`:

```yaml
name: CI Pipeline

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Set up Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '20'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Run tests
        run: npm test
      
      - name: Build Docker image
        run: |
          docker build -t myapp:${{ gitea.sha }} .
```

Deploy an Actions runner:

```yaml
version: "3.8"

services:
  gitea-runner:
    image: gitea/act_runner:latest
    container_name: gitea-runner
    environment:
      GITEA_INSTANCE_URL: http://gitea:3000
      GITEA_RUNNER_REGISTRATION_TOKEN: YOUR_RUNNER_TOKEN
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - gitea_runner_data:/data
    depends_on:
      - gitea
    restart: unless-stopped

volumes:
  gitea_runner_data:
```

## Gitea vs GitLab Resource Comparison

| Metric | Gitea | GitLab CE |
|--------|-------|-----------|
| Minimum RAM | 100MB | 4GB |
| Disk Space | 200MB | 10GB+ |
| Startup Time | 2-5 seconds | 2-5 minutes |
| Full Featured | Yes (lighter) | Yes (heavier) |

## Conclusion

Gitea deployed via Portainer is the ideal Git server for resource-constrained environments like home labs, small teams, or ARM devices. With PostgreSQL for reliability and Gitea Actions for CI/CD, it covers most development team needs at a fraction of GitLab's resource cost. The low memory footprint makes it particularly suitable for running alongside many other services.
