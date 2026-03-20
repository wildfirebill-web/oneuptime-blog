# How to Self-Host a Code Server (VS Code) with Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Self-Hosted, VS Code, Code Server, Development, IDE

Description: Deploy code-server to run VS Code in your browser from anywhere using Portainer, with persistent workspaces and extensions.

## Introduction

code-server is VS Code running on a remote server, accessible through a web browser. It means you can code from any device - a tablet, Chromebook, or thin client - with a full VS Code experience. This guide covers deploying a secure, persistent code-server instance using Portainer.

## Prerequisites

- Portainer installed and running
- At least 2GB RAM (4GB recommended)
- A domain name with HTTPS
- Reverse proxy for secure access

## Step 1: Deploy code-server

```yaml
# docker-compose.yml - code-server (VS Code in browser)

version: "3.8"

networks:
  dev_network:
    driver: bridge

volumes:
  codeserver_config:
  codeserver_workspace:

services:
  code-server:
    image: lscr.io/linuxserver/code-server:latest
    container_name: code-server
    restart: unless-stopped
    ports:
      - "8443:8443"
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=America/New_York
      # Password for the IDE (or use HASHED_PASSWORD)
      - PASSWORD=your_secure_password
      # Or use hashed password: echo -n "password" | npx argon2-cli -e
      # - HASHED_PASSWORD=
      # Sudo password for installing packages
      - SUDO_PASSWORD=sudo_password
      # Default workspace
      - DEFAULT_WORKSPACE=/workspace
      # Proxy domain for CSP headers
      - PROXY_DOMAIN=code.yourdomain.com
    volumes:
      # code-server configuration (extensions, settings)
      - codeserver_config:/config
      # Workspace persistence
      - codeserver_workspace:/workspace
      # Optional: mount host directories
      - /opt/projects:/workspace/projects
    networks:
      - dev_network
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.code-server.rule=Host(`code.yourdomain.com`)"
      - "traefik.http.routers.code-server.entrypoints=websecure"
      - "traefik.http.routers.code-server.tls.certresolver=letsencrypt"
      - "traefik.http.services.code-server.loadbalancer.server.port=8443"
```

## Step 2: Custom code-server with Development Tools

Build a custom image with your preferred development tools:

```dockerfile
# Dockerfile - code-server with dev tools
FROM lscr.io/linuxserver/code-server:latest

# Switch to root to install packages
USER root

# Install development tools
RUN apt-get update && apt-get install -y \
    # Version control
    git \
    # Build tools
    build-essential \
    make \
    # Programming languages
    python3 \
    python3-pip \
    nodejs \
    npm \
    golang \
    # Container tools
    docker.io \
    # Database clients
    postgresql-client \
    redis-tools \
    # Network tools
    curl \
    wget \
    jq \
    netcat \
    # Editor utilities
    vim \
    tmux \
    # Cleanup
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

# Install Node.js LTS
RUN curl -fsSL https://deb.nodesource.com/setup_20.x | bash - \
    && apt-get install -y nodejs

# Install Python packages
RUN pip3 install \
    black \
    pylint \
    mypy \
    pytest \
    requests \
    fastapi \
    uvicorn

# Install global npm packages
RUN npm install -g \
    typescript \
    ts-node \
    @types/node \
    eslint \
    prettier

# Install Go tools
ENV GOPATH=/go
ENV PATH=$PATH:/usr/local/go/bin:$GOPATH/bin

# Install popular VS Code extensions
USER abc
RUN code-server \
    --install-extension ms-python.python \
    --install-extension golang.go \
    --install-extension ms-azuretools.vscode-docker \
    --install-extension dbaeumer.vscode-eslint \
    --install-extension esbenp.prettier-vscode \
    --install-extension GitLab.gitlab-workflow \
    --install-extension ms-vscode.vscode-typescript-next
```

```yaml
# Updated docker-compose to use custom image
services:
  code-server:
    build:
      context: /opt/code-server
      dockerfile: Dockerfile
    image: my-code-server:latest
    container_name: code-server
    # ... rest of config
```

## Step 3: Configure VS Code Settings

```json
// Default user settings (place in /config/data/User/settings.json)
{
  "editor.fontSize": 14,
  "editor.tabSize": 2,
  "editor.formatOnSave": true,
  "editor.wordWrap": "on",
  "terminal.integrated.defaultProfile.linux": "bash",
  "files.autoSave": "afterDelay",
  "files.autoSaveDelay": 1000,
  "git.autofetch": true,
  "git.confirmSync": false,
  "python.defaultInterpreterPath": "/usr/bin/python3",
  "[python]": {
    "editor.defaultFormatter": "ms-python.black-formatter"
  },
  "[typescript]": {
    "editor.defaultFormatter": "esbenp.prettier-vscode"
  },
  "workbench.colorTheme": "Default Dark Modern",
  "workbench.iconTheme": "material-icon-theme"
}
```

## Step 4: Set Up Git Authentication

```bash
# Configure Git credentials inside the container
docker exec -it code-server bash -c "
  git config --global user.name 'Your Name'
  git config --global user.email 'your@email.com'
  git config --global credential.helper store
"

# For SSH key authentication
docker exec -it code-server bash -c "
  mkdir -p /config/.ssh
  ssh-keygen -t ed25519 -f /config/.ssh/id_ed25519 -N ''
  cat /config/.ssh/id_ed25519.pub
"
# Copy the public key to your GitHub/GitLab settings
```

## Step 5: Mount Docker Socket for Container Development

```yaml
# Allow code-server to manage Docker containers
services:
  code-server:
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      # Install docker CLI in the container
    group_add:
      - "999"  # docker group GID
```

## Step 6: Configure Multiple Users

```yaml
# Run separate code-server instances for different users
services:
  code-server-user1:
    image: lscr.io/linuxserver/code-server:latest
    container_name: code-server-user1
    ports:
      - "8444:8443"
    environment:
      - PUID=1001
      - PGID=1001
      - PASSWORD=user1_password
      - DEFAULT_WORKSPACE=/workspace/user1
    volumes:
      - /opt/code-server/user1/config:/config
      - /opt/code-server/user1/workspace:/workspace/user1

  code-server-user2:
    image: lscr.io/linuxserver/code-server:latest
    container_name: code-server-user2
    ports:
      - "8445:8443"
    environment:
      - PUID=1002
      - PGID=1002
      - PASSWORD=user2_password
```

## Step 7: Enable Workspace Sharing via Gitpod Protocol

```bash
# In code-server, open a workspace from git
# URL format: https://code.yourdomain.com/#https://github.com/user/repo

# Enable gitpod-compatible workspace URLs in settings.json
{
  "workbench.startupEditor": "readme",
  "git.openRepositoryInParentFolders": "always"
}
```

## Security Considerations

```yaml
# Add security headers via Traefik middleware
labels:
  - "traefik.http.middlewares.code-server-headers.headers.browserXssFilter=true"
  - "traefik.http.middlewares.code-server-headers.headers.contentTypeNosniff=true"
  - "traefik.http.middlewares.code-server-headers.headers.frameDeny=true"
  - "traefik.http.routers.code-server.middlewares=code-server-headers"
```

## Conclusion

You now have a full VS Code development environment accessible from any browser, deployed through Portainer. code-server is ideal for coding on devices you don't own, working from tablets, or standardizing your development environment across a team. Use Portainer to manage updates, monitor resource usage, and restart the container if needed. The persistent volumes ensure your settings, extensions, and code survive container restarts.
