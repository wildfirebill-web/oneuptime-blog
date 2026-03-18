# How to Use Podman on Embedded Linux Systems

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Podman, Embedded Linux, Containers, Yocto, Buildroot, DevOps, OTA Updates, Industrial

Description: Learn to run Podman on embedded Linux systems built with Yocto and Buildroot, covering custom images, OTA updates, and production deployment patterns.

---

> Embedded Linux systems power everything from industrial controllers to medical devices and automotive infotainment. Adding Podman to these systems brings container isolation and updatability to firmware that traditionally required full reflashing for every update. With Podman, you can update application logic independently of the base operating system, reducing downtime and deployment risk.

Embedded Linux differs from server Linux in important ways. The operating system image is typically built from source using tools like the Yocto Project or Buildroot. Storage is measured in megabytes rather than gigabytes. The root filesystem may be read-only. Despite these constraints, Podman's modular design allows it to be integrated into embedded Linux builds and run containers even on systems with as little as 256MB of RAM and 1GB of storage.

---

## Why Containers on Embedded Systems

Traditional embedded Linux deploys applications as part of the firmware image. Updating the application means rebuilding and reflashing the entire image. Containers change this model:

| Traditional Embedded | Container-Based Embedded |
|---------------------|-------------------------|
| Application baked into firmware | Application in container |
| Full reflash for updates | Container image swap |
| Single application version | Multiple versions coexist |
| Monolithic testing | Isolated component testing |
| Coupled to OS version | Decoupled from OS |

---

## Adding Podman to a Yocto Project Build

The Yocto Project is the most common build system for custom embedded Linux. Podman is available through the `meta-virtualization` layer.

### Setting Up the Layers

```bash
# In your Yocto build directory
cd poky
git clone https://git.yoctoproject.org/meta-virtualization
git clone https://git.openembedded.org/meta-openembedded

# Add layers to bblayers.conf
bitbake-layers add-layer ../meta-openembedded/meta-oe
bitbake-layers add-layer ../meta-openembedded/meta-filesystems
bitbake-layers add-layer ../meta-openembedded/meta-networking
bitbake-layers add-layer ../meta-openembedded/meta-python
bitbake-layers add-layer ../meta-virtualization
```

### Configuring local.conf

Add Podman and its dependencies to your image:

```bash
cat >> conf/local.conf <<EOF
# Container runtime
IMAGE_INSTALL:append = " podman podman-compose skopeo"
IMAGE_INSTALL:append = " slirp4netns fuse-overlayfs"
IMAGE_INSTALL:append = " cni"
IMAGE_INSTALL:append = " kernel-modules"

# Required kernel features
DISTRO_FEATURES:append = " virtualization"

# Storage driver support
DISTRO_FEATURES:append = " overlayfs"

# Rootless container support
IMAGE_INSTALL:append = " shadow-subids"
EOF
```

### Kernel Configuration

Ensure your kernel config includes the necessary features for containers:

```bash
cat >> conf/local.conf <<EOF
# Kernel container requirements
KERNEL_FEATURES:append = " features/cgroups/cgroups.scc"
KERNEL_FEATURES:append = " features/namespaces/namespaces.scc"
KERNEL_FEATURES:append = " features/overlayfs/overlayfs.scc"
EOF
```

Or add a kernel configuration fragment:

```bash
cat > recipes-kernel/linux/files/containers.cfg <<EOF
CONFIG_NAMESPACES=y
CONFIG_USER_NS=y
CONFIG_NET_NS=y
CONFIG_PID_NS=y
CONFIG_IPC_NS=y
CONFIG_UTS_NS=y
CONFIG_CGROUPS=y
CONFIG_CGROUP_DEVICE=y
CONFIG_CGROUP_FREEZER=y
CONFIG_CGROUP_PIDS=y
CONFIG_CGROUP_SCHED=y
CONFIG_MEMCG=y
CONFIG_CPUSETS=y
CONFIG_OVERLAY_FS=y
CONFIG_VETH=y
CONFIG_BRIDGE=y
CONFIG_BRIDGE_NETFILTER=y
CONFIG_NF_NAT=y
CONFIG_IP_NF_NAT=y
CONFIG_IP_NF_TARGET_MASQUERADE=y
CONFIG_NETFILTER_XT_MATCH_ADDRTYPE=y
CONFIG_NETFILTER_XT_MATCH_CONNTRACK=y
CONFIG_NF_CONNTRACK=y
EOF
```

### Building the Image

```bash
bitbake core-image-minimal
# Or with a more complete userspace
bitbake core-image-base
```

---

## Adding Podman to a Buildroot System

For Buildroot-based systems, enable Podman in the configuration:

```bash
make menuconfig
```

Navigate to:
- Target packages > System tools > podman
- Target packages > Networking > slirp4netns
- Kernel > Enable namespaces, cgroups, overlayfs

Or add to your defconfig:

```bash
cat >> configs/myboard_defconfig <<EOF
BR2_PACKAGE_PODMAN=y
BR2_PACKAGE_SLIRP4NETNS=y
BR2_PACKAGE_FUSE_OVERLAYFS=y
BR2_PACKAGE_SHADOW=y
BR2_LINUX_KERNEL_CUSTOM_CONFIG_FILE="board/myboard/linux.config"
EOF
```

Build:

```bash
make myboard_defconfig
make
```

---

## Configuring Podman on Embedded Systems

### Storage Configuration for Limited Storage

Embedded systems often have limited flash storage. Configure Podman to be storage-efficient:

```bash
cat > /etc/containers/storage.conf <<EOF
[storage]
driver = "overlay"
graphroot = "/var/lib/containers/storage"

[storage.options]
additionalimagestores = []
size = ""

[storage.options.overlay]
mount_program = "/usr/bin/fuse-overlayfs"
EOF
```

### Read-Only Root Filesystem

Many embedded systems use a read-only root filesystem for reliability. Configure Podman to store data on a writable partition:

```bash
# Mount a writable data partition
mount /dev/mmcblk0p3 /data

# Symlink Podman storage to writable partition
mkdir -p /data/containers/storage
mkdir -p /data/containers/run
ln -s /data/containers/storage /var/lib/containers/storage
ln -s /data/containers/run /run/containers
```

Or configure through storage.conf:

```bash
cat > /etc/containers/storage.conf <<EOF
[storage]
driver = "overlay"
graphroot = "/data/containers/storage"
runroot = "/data/containers/run"

[storage.options.overlay]
mount_program = "/usr/bin/fuse-overlayfs"
EOF
```

### Registry Configuration for Offline Use

Embedded systems typically do not have internet access. Configure Podman to not search public registries:

```bash
cat > /etc/containers/registries.conf <<EOF
unqualified-search-registries = []

[[registry]]
location = "docker.io"
blocked = true

[[registry]]
location = "quay.io"
blocked = true

[[registry]]
location = "ghcr.io"
blocked = true
EOF
```

---

## Building Minimal Container Images for Embedded Targets

### Scratch-Based Images

For the smallest possible images, use scratch as the base:

```dockerfile
# Build on a development machine for the target architecture
FROM docker.io/library/golang:1.22 AS builder
WORKDIR /app
COPY . .
RUN CGO_ENABLED=0 GOARCH=arm64 go build -ldflags="-s -w" -o /app/controller .

FROM scratch
COPY --from=builder /app/controller /controller
ENTRYPOINT ["/controller"]
```

This produces an image that is just the static binary, typically a few megabytes.

### BusyBox-Based Images

For images that need basic Unix tools:

```dockerfile
FROM docker.io/library/busybox:musl AS base

FROM scratch
COPY --from=base /bin/busybox /bin/busybox
RUN ["/bin/busybox", "--install", "-s", "/bin/"]

WORKDIR /app
COPY myapp /app/
CMD ["/app/myapp"]
```

### Alpine-Based Images for Embedded

```dockerfile
FROM docker.io/library/alpine:3.19
RUN apk add --no-cache python3 py3-pip && \
    pip install --no-cache-dir flask && \
    apk del py3-pip
WORKDIR /app
COPY app.py .
EXPOSE 8080
CMD ["python3", "app.py"]
```

---

## Pre-Loading Container Images

Since embedded systems are typically offline, pre-load container images during manufacturing or initial provisioning.

### During Yocto Build

Create a Yocto recipe that pre-loads container images into the filesystem:

```bash
# recipes-containers/preload-images/preload-images.bb
SUMMARY = "Pre-load container images for embedded deployment"
LICENSE = "MIT"

SRC_URI = "file://app-image.tar"

do_install() {
    install -d ${D}/data/containers/preload
    install -m 0644 ${WORKDIR}/app-image.tar ${D}/data/containers/preload/
}

FILES:${PN} = "/data/containers/preload/"
```

Add a systemd service to load images on first boot:

```bash
cat > /etc/systemd/system/container-preload.service <<EOF
[Unit]
Description=Load pre-installed container images
After=local-fs.target
ConditionPathExists=/data/containers/preload

[Service]
Type=oneshot
ExecStart=/bin/sh -c 'for f in /data/containers/preload/*.tar; do podman load -i "$f"; done'
ExecStartPost=/bin/rm -rf /data/containers/preload
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
EOF
```

---

## OTA Container Updates for Embedded Devices

### Dual-Container Update Strategy

Keep two versions of the application container and switch between them:

```bash
#!/bin/bash
# ota-update.sh
NEW_IMAGE_TAR=$1
APP_NAME="controller"

# Load new image
NEW_TAG=$(podman load -i "$NEW_IMAGE_TAR" | awk '{print $NF}')

# Stop current container
podman stop "$APP_NAME" 2>/dev/null

# Rename current as rollback
podman rename "$APP_NAME" "${APP_NAME}-rollback" 2>/dev/null

# Start new version
podman run -d --name "$APP_NAME" \
  --restart always \
  -v /data/app:/data:Z \
  "$NEW_TAG"

# Verify health
sleep 10
if podman healthcheck run "$APP_NAME" 2>/dev/null; then
  echo "Update successful, removing rollback"
  podman rm "${APP_NAME}-rollback" 2>/dev/null
  podman image prune -f
else
  echo "Update failed, rolling back"
  podman stop "$APP_NAME"
  podman rm "$APP_NAME"
  podman rename "${APP_NAME}-rollback" "$APP_NAME"
  podman start "$APP_NAME"
fi
```

### Using systemd for Container Lifecycle

```bash
cat > /etc/containers/systemd/controller.container <<EOF
[Unit]
Description=Embedded Controller Application
After=local-fs.target network.target

[Container]
Image=localhost/controller:latest
Volume=/data/app:/data:Z
Device=/dev/ttyS0:/dev/ttyS0
Environment=DEVICE_SERIAL=%m
Network=host

[Service]
Restart=always
WatchdogSec=60
TimeoutStartSec=30

[Install]
WantedBy=multi-user.target
EOF
```

---

## Hardware Access Patterns

### Serial Port Communication

```bash
podman run -d --name serial-controller \
  --device /dev/ttyS0:/dev/ttyS0 \
  --device /dev/ttyS1:/dev/ttyS1 \
  localhost/serial-app:latest
```

### CAN Bus Access

```bash
podman run -d --name can-controller \
  --network host \
  --cap-add NET_ADMIN \
  localhost/can-app:latest
```

### SPI and I2C

```bash
podman run -d --name sensor-hub \
  --device /dev/spidev0.0:/dev/spidev0.0 \
  --device /dev/i2c-1:/dev/i2c-1 \
  --privileged=false \
  localhost/sensor-hub:latest
```

---

## Monitoring and Diagnostics

### Embedded Watchdog Integration

```bash
#!/bin/bash
# watchdog-check.sh - Called by hardware watchdog
CONTAINERS="controller sensor-hub gateway"

all_healthy=true
for name in $CONTAINERS; do
  status=$(podman inspect "$name" --format '{{.State.Status}}' 2>/dev/null)
  if [ "$status" != "running" ]; then
    logger -t container-watchdog "$name is not running (status: $status), restarting"
    podman start "$name" 2>/dev/null || podman restart "$name" 2>/dev/null
    all_healthy=false
  fi
done

if $all_healthy; then
  # Pet the hardware watchdog
  echo "1" > /dev/watchdog
fi
```

### Resource Usage Tracking

```bash
# Log container resource usage to a file for later analysis
podman stats --no-stream --format \
  '{{.Name}},{{.CPUPerc}},{{.MemUsage}},{{.NetIO}},{{.BlockIO}}' \
  >> /data/logs/container-stats.csv
```

---

## Conclusion

Podman on embedded Linux systems brings the benefits of containerization to firmware development and deployment. By integrating Podman into Yocto or Buildroot builds, you can separate application logic from the base operating system, enabling independent updates without full reflashing. Build minimal container images using scratch or BusyBox bases to conserve the limited storage on embedded hardware. Use read-only root filesystems with dedicated writable partitions for container data. Pre-load images during manufacturing, and implement OTA update workflows with rollback capabilities for reliable field updates. The combination of Podman's lightweight runtime with embedded Linux build systems creates a modern, maintainable platform for embedded applications across industrial, medical, automotive, and consumer devices.
