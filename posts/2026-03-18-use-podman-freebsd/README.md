# How to Use Podman on FreeBSD

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Podman, FreeBSD, Containers, OCI, Unix

Description: Learn how to install and use Podman on FreeBSD for running OCI-compatible Linux containers using FreeBSD's jail and Linux emulation capabilities.

---

> Podman brings OCI container support to FreeBSD, letting you run Linux containers alongside native FreeBSD workloads. This guide covers installing Podman on FreeBSD, configuring the Linux compatibility layer, and running containers on one of the most reliable Unix operating systems.

FreeBSD has long had its own containerization technology in the form of jails, but the growing ecosystem of OCI container images makes Podman support valuable for FreeBSD users. Podman on FreeBSD enables you to pull and run standard Linux container images, bridging the gap between the FreeBSD ecosystem and the broader container world. This support relies on FreeBSD's Linux binary compatibility layer and specific adaptations in Podman for the FreeBSD kernel.

This guide walks through setting up Podman on FreeBSD and running OCI containers.

---

## Prerequisites

Podman on FreeBSD requires FreeBSD 13.1 or later (FreeBSD 14.0 or later is recommended for the best compatibility). The container support uses a combination of FreeBSD jails, ZFS, and the Linux compatibility layer. Ensure your system is up to date:

```bash
sudo freebsd-update fetch
sudo freebsd-update install
sudo pkg update
```

Check your FreeBSD version:

```bash
freebsd-version
uname -a
```

## Installing Podman

Install Podman from the FreeBSD package repository:

```bash
sudo pkg install -y podman
```

This installs Podman along with its dependencies. Verify the installation:

```bash
podman --version
podman info
```

## Enabling the Linux Compatibility Layer

To run Linux containers on FreeBSD, enable the Linux compatibility layer:

```bash
sudo sysrc linux_enable="YES"
sudo service linux start
```

Load the Linux kernel module:

```bash
sudo kldload linux64
sudo sysrc kld_list+="linux64"
```

Verify the Linux compatibility layer is active:

```bash
kldstat | grep linux
```

## Configuring Container Storage with ZFS

FreeBSD works best with ZFS for container storage. Create a ZFS dataset for Podman:

```bash
sudo zfs create -o mountpoint=/var/db/containers zpool/containers
sudo zfs create zpool/containers/storage
```

Configure Podman to use ZFS storage. Edit `/usr/local/etc/containers/storage.conf`:

```toml
[storage]
driver = "zfs"
graphroot = "/var/db/containers/storage"

[storage.options.zfs]
fsname = "zpool/containers/storage"
```

ZFS provides copy-on-write semantics, snapshots, and efficient storage management that works well with container layers.

## Configuring Container Registries

Set up container registry configuration:

```bash
sudo tee /usr/local/etc/containers/registries.conf << 'EOF'
unqualified-search-registries = ["docker.io"]

[[registry]]
location = "docker.io"
EOF
```

## Running Your First Container

Pull and run a basic Linux container:

```bash
podman pull docker.io/library/alpine:latest
podman run --rm docker.io/library/alpine:latest echo "Hello from FreeBSD"
```

Run an interactive container:

```bash
podman run -it --rm docker.io/library/alpine:latest sh
```

Inside the container, verify the Linux environment:

```bash
cat /etc/os-release
uname -a
```

## Running a Web Server

Deploy an Nginx container:

```bash
podman run -d --name webserver \
  -p 8080:80 \
  docker.io/library/nginx:latest

podman ps
curl http://localhost:8080
```

## Working with Volumes

Create volumes for persistent data on ZFS:

```bash
podman volume create mydata
podman volume ls
podman volume inspect mydata
```

Mount a volume into a container:

```bash
podman run -d --name database \
  -v mydata:/var/lib/postgresql/data \
  -e POSTGRES_PASSWORD=secret \
  -p 5432:5432 \
  docker.io/library/postgres:16
```

Bind mount a host directory:

```bash
podman run -d --name web \
  -v /usr/local/www:/usr/share/nginx/html:ro \
  -p 8080:80 \
  docker.io/library/nginx:latest
```

## Building Container Images

Create a Containerfile and build on FreeBSD:

```dockerfile
FROM docker.io/library/alpine:latest

RUN apk add --no-cache python3 py3-pip

WORKDIR /app
COPY requirements.txt .
RUN pip3 install --no-cache-dir --break-system-packages -r requirements.txt

COPY . .

EXPOSE 8000
CMD ["python3", "app.py"]
```

Build the image:

```bash
podman build -t myapp:latest .
podman images
```

## Networking

Podman on FreeBSD supports basic networking. Create a container network:

```bash
podman network create appnet
podman network ls
```

Run containers on the same network:

```bash
podman run -d --network appnet --name backend myapp-api:latest
podman run -d --network appnet --name frontend -p 8080:80 myapp-web:latest
```

## Pod Support

Create pods to group related containers:

```bash
podman pod create --name app-stack -p 8080:80 -p 5432:5432

podman run -d --pod app-stack --name db \
  -e POSTGRES_PASSWORD=secret \
  docker.io/library/postgres:16

podman run -d --pod app-stack --name app \
  myapp:latest
```

Manage pods:

```bash
podman pod ps
podman pod stop app-stack
podman pod start app-stack
```

## Integrating with FreeBSD rc.d

Create an rc.d service script for your container:

```bash
sudo tee /usr/local/etc/rc.d/podman_webapp << 'RCEOF'
#!/bin/sh

# PROVIDE: podman_webapp
# REQUIRE: DAEMON podman
# KEYWORD: shutdown

. /etc/rc.subr

name="podman_webapp"
rcvar="${name}_enable"

start_cmd="${name}_start"
stop_cmd="${name}_stop"
status_cmd="${name}_status"

podman_webapp_start()
{
    echo "Starting ${name}."
    /usr/local/bin/podman run -d --name webapp \
        -p 8080:80 \
        docker.io/library/nginx:latest
}

podman_webapp_stop()
{
    echo "Stopping ${name}."
    /usr/local/bin/podman stop webapp
    /usr/local/bin/podman rm webapp
}

podman_webapp_status()
{
    /usr/local/bin/podman ps --filter name=webapp
}

load_rc_config $name
run_rc_command "$1"
RCEOF

sudo chmod +x /usr/local/etc/rc.d/podman_webapp
sudo sysrc podman_webapp_enable="YES"
```

Start the service:

```bash
sudo service podman_webapp start
sudo service podman_webapp status
```

## FreeBSD Jails vs Podman Containers

FreeBSD offers both native jails and Podman containers. Understanding when to use each helps you make the right architectural choice.

Use FreeBSD jails when you need native FreeBSD processes, ZFS integration, and minimal overhead:

```bash
# Create a native FreeBSD jail
sudo jail -c name=myjail path=/jails/myjail host.hostname=myjail ip4.addr=192.168.1.100
```

Use Podman when you need to run Linux container images or leverage the OCI ecosystem:

```bash
podman run -d docker.io/library/redis:7
```

You can run both jails and Podman containers on the same system for different workloads.

## Resource Limits

Use FreeBSD's rctl to limit container resources:

```bash
podman run -d --name limited \
  --memory=512m \
  --cpus=1.0 \
  myapp:latest
```

Monitor resource usage:

```bash
podman stats --no-stream
```

## Firewall Configuration with PF

FreeBSD uses PF (Packet Filter) for firewalling. Add rules for container ports:

```bash
# /etc/pf.conf
pass in on egress proto tcp to port 8080
pass in on egress proto tcp to port 5432
```

Load the rules:

```bash
sudo pfctl -f /etc/pf.conf
sudo pfctl -e
```

## Troubleshooting

Check the Linux compatibility layer status:

```bash
kldstat | grep linux
sysctl compat.linux
```

View Podman logs:

```bash
podman logs webserver
podman events --since 1h
```

If containers fail to start, check for missing Linux kernel module support:

```bash
sudo kldload linux64
sudo kldload fdescfs
sudo kldload linprocfs
sudo kldload tmpfs
```

Mount the required filesystems:

```bash
sudo mount -t fdescfs fdesc /dev/fd
sudo mount -t linprocfs linproc /compat/linux/proc
```

## Maintenance

Clean up unused resources:

```bash
podman system df
podman system prune -a
podman volume prune
```

Snapshot container storage with ZFS:

```bash
sudo zfs snapshot zpool/containers/storage@backup-$(date +%Y%m%d)
sudo zfs list -t snapshot
```

## Conclusion

Podman on FreeBSD opens the door to running the vast ecosystem of OCI-compatible Linux containers on one of the most stable and performant Unix operating systems. While FreeBSD jails remain the go-to solution for native FreeBSD workloads, Podman bridges the gap by letting you run Linux container images alongside your FreeBSD infrastructure. The combination of ZFS storage, PF firewalling, and Podman containers gives FreeBSD administrators a powerful and flexible platform for modern containerized deployments.
