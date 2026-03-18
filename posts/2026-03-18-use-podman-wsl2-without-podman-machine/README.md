# How to Use Podman on WSL2 Without Podman Machine

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Podman, WSL2, Windows, Containers, Linux

Description: Learn how to run Podman directly inside WSL2 without using Podman Machine, giving you a lightweight and efficient container runtime on Windows.

---

> Running Podman directly in WSL2 bypasses the overhead of Podman Machine and gives you a native Linux container experience on Windows. This guide shows you how to install, configure, and use Podman inside WSL2 without the additional virtual machine layer.

Podman Machine on Windows creates a dedicated virtual machine to run containers, but if you already have WSL2 installed, you have a Linux kernel running on your system. Running Podman directly inside WSL2 eliminates the extra VM overhead of Podman Machine, reduces memory consumption, and gives you a more integrated experience with your Windows development environment. This approach is especially useful for developers who already use WSL2 for their daily workflow.

This guide walks through setting up Podman inside WSL2 without Podman Machine, configuring it for rootless operation, and integrating it with your Windows development tools.

---

## Prerequisites

Ensure WSL2 is installed and configured on your Windows system. Open PowerShell as Administrator:

```powershell
wsl --install
wsl --set-default-version 2
```

Install a Linux distribution. Ubuntu is the most common choice:

```powershell
wsl --install -d Ubuntu-24.04
```

Verify WSL2 is running:

```powershell
wsl --list --verbose
```

The output should show your distribution running with VERSION 2.

## Installing Podman in WSL2

Open your WSL2 distribution and install Podman. For Ubuntu:

```bash
sudo apt update
sudo apt install -y podman podman-compose uidmap slirp4netns
```

For Fedora on WSL2:

```bash
sudo dnf install -y podman podman-compose
```

Verify the installation:

```bash
podman --version
podman info
```

## Why Not Use Podman Machine

Podman Machine creates a separate QEMU or Hyper-V virtual machine to run containers. When using WSL2, this means you have two virtual machines running: WSL2 and the Podman Machine VM. This wastes resources and adds complexity.

By running Podman directly in WSL2, you get a single Linux environment that handles both your shell work and container workloads. The benefits include lower memory usage, faster startup times, and simpler networking since everything runs in the same WSL2 instance.

## Configuring Rootless Containers

Set up rootless containers in your WSL2 instance:

```bash
cat /etc/subuid
cat /etc/subgid
```

If your user is missing entries:

```bash
sudo usermod --add-subuids 100000-165535 --add-subgids 100000-165535 $USER
```

Run a test to verify rootless mode works:

```bash
podman run --rm docker.io/library/alpine:latest echo "Rootless Podman in WSL2"
```

## Configuring Container Registries

Set up default registries:

```bash
sudo tee /etc/containers/registries.conf.d/docker.conf << 'EOF'
unqualified-search-registries = ["docker.io"]
EOF
```

## Fixing Common WSL2 Issues

WSL2 has some differences from a standard Linux installation. Address common issues.

If systemd is not enabled (older WSL2 versions), enable it by editing `/etc/wsl.conf`:

```ini
[boot]
systemd=true
```

Then restart WSL2 from PowerShell:

```powershell
wsl --shutdown
```

Reopen your WSL2 terminal and verify systemd is running:

```bash
systemctl --version
systemctl is-system-running
```

If you encounter cgroup-related errors, ensure cgroups v2 is available:

```bash
mount | grep cgroup
cat /proc/filesystems | grep cgroup
```

## Running Containers

Run a web server accessible from Windows:

```bash
podman run -d --name webserver -p 8080:80 docker.io/library/nginx:latest
```

Access it from your Windows browser at `http://localhost:8080`. WSL2 automatically forwards ports to Windows.

Run a development database:

```bash
podman volume create pgdata

podman run -d --name postgres \
  -e POSTGRES_PASSWORD=dev \
  -e POSTGRES_DB=myapp \
  -v pgdata:/var/lib/postgresql/data \
  -p 5432:5432 \
  docker.io/library/postgres:16
```

Connect to it from Windows tools like pgAdmin or DBeaver using `localhost:5432`.

## Building Container Images

Create and build images in your WSL2 environment:

```dockerfile
FROM docker.io/library/node:20-slim

WORKDIR /app

COPY package*.json ./
RUN npm ci

COPY . .

EXPOSE 3000
CMD ["npm", "start"]
```

Build the image:

```bash
cd /home/user/projects/myapp
podman build -t myapp:latest .
podman run -d -p 3000:3000 myapp:latest
```

Access the application from Windows at `http://localhost:3000`.

## Accessing Windows Files from Containers

WSL2 mounts your Windows drives under `/mnt`. You can mount Windows directories into containers:

```bash
podman run -it --rm \
  -v /mnt/c/Users/YourName/Projects:/projects \
  docker.io/library/node:20 \
  bash
```

For better performance, keep project files inside the WSL2 filesystem rather than on the Windows mount:

```bash
# Fast - files in WSL2 filesystem
podman run -v ~/projects/myapp:/app docker.io/library/node:20 npm test

# Slower - files on Windows mount
podman run -v /mnt/c/projects/myapp:/app docker.io/library/node:20 npm test
```

## Integrating with VS Code

Visual Studio Code with the WSL extension works seamlessly with Podman in WSL2. Install the VS Code Remote - WSL extension, then open your project:

```bash
# From inside WSL2
cd ~/projects/myapp
code .
```

VS Code opens in the WSL2 context where Podman is available. The integrated terminal has direct access to Podman commands.

For the Podman extension in VS Code, point it to the WSL2 Podman socket:

```bash
echo $XDG_RUNTIME_DIR/podman/podman.sock
```

## Systemd Integration with Quadlet

With systemd enabled in WSL2, you can use Quadlet for persistent container services:

```ini
# ~/.config/containers/systemd/devstack.container
[Container]
ContainerName=devstack-db
Image=docker.io/library/postgres:16
PublishPort=5432:5432
Environment=POSTGRES_PASSWORD=dev
Volume=pgdata:/var/lib/postgresql/data

[Service]
Restart=always

[Install]
WantedBy=default.target
```

Enable the service:

```bash
systemctl --user daemon-reload
systemctl --user enable --now devstack.service
systemctl --user status devstack.service
```

## Using Podman Compose

Run multi-container development stacks:

```yaml
version: "3.8"
services:
  frontend:
    image: docker.io/library/node:20
    working_dir: /app
    volumes:
      - ./frontend:/app
    ports:
      - "3000:3000"
    command: npm run dev
  api:
    build: ./api
    ports:
      - "8000:8000"
    environment:
      - DATABASE_URL=postgresql://postgres:dev@db:5432/myapp
    depends_on:
      - db
  db:
    image: docker.io/library/postgres:16
    environment:
      POSTGRES_PASSWORD: dev
      POSTGRES_DB: myapp
    volumes:
      - pgdata:/var/lib/postgresql/data
    ports:
      - "5432:5432"
  redis:
    image: docker.io/library/redis:7
    ports:
      - "6379:6379"
volumes:
  pgdata:
```

Run the stack:

```bash
podman-compose up -d
podman-compose ps
podman-compose logs -f
```

All ports are accessible from Windows browsers and tools.

## Networking Between WSL2 and Windows

WSL2 handles port forwarding to Windows automatically. Containers publishing ports in WSL2 are accessible on `localhost` from Windows.

For container-to-container networking, create Podman networks:

```bash
podman network create dev-network
podman run -d --network dev-network --name api myapi:latest
podman run -d --network dev-network --name worker myworker:latest
```

## Pods for Development

Group development services into a pod:

```bash
podman pod create --name dev-pod -p 3000:3000 -p 5432:5432 -p 6379:6379

podman run -d --pod dev-pod --name dev-db \
  -e POSTGRES_PASSWORD=dev \
  docker.io/library/postgres:16

podman run -d --pod dev-pod --name dev-cache \
  docker.io/library/redis:7

podman run -d --pod dev-pod --name dev-app \
  -v ~/projects/myapp:/app \
  -w /app \
  docker.io/library/node:20 \
  npm run dev
```

## Performance Optimization

Optimize Podman performance in WSL2:

```bash
# Use the overlay storage driver
cat ~/.config/containers/storage.conf
```

Ensure you are using the `overlay` driver with `fuse-overlayfs`:

```toml
[storage]
driver = "overlay"

[storage.options.overlay]
mount_program = "/usr/bin/fuse-overlayfs"
```

Limit WSL2 memory usage by creating or editing `%USERPROFILE%\.wslconfig` on Windows:

```ini
[wsl2]
memory=4GB
processors=4
swap=2GB
```

## Cleanup and Maintenance

Regular cleanup:

```bash
podman system df
podman system prune -a
podman volume prune
```

Free up WSL2 disk space after pruning:

```powershell
# From PowerShell
wsl --shutdown
diskpart
# select vdisk file="C:\Users\YourName\AppData\Local\Packages\...\ext4.vhdx"
# compact vdisk
```

## Creating a Podman Alias for Docker Compatibility

If you have scripts or tools that expect the docker command:

```bash
echo 'alias docker=podman' >> ~/.bashrc
echo 'alias docker-compose=podman-compose' >> ~/.bashrc
source ~/.bashrc
```

You can also create a symlink:

```bash
sudo ln -s /usr/bin/podman /usr/local/bin/docker
```

## Conclusion

Running Podman directly in WSL2 without Podman Machine gives you a lean, efficient container runtime on Windows. You avoid the overhead of an additional virtual machine, get direct filesystem access between Windows and your containers, and benefit from automatic port forwarding. Combined with VS Code's WSL integration and systemd support for persistent services, this setup provides a first-class container development experience on Windows that rivals native Linux workstations. For developers already using WSL2, adding Podman directly to your WSL2 instance is the simplest and most resource-efficient way to work with containers on Windows.
