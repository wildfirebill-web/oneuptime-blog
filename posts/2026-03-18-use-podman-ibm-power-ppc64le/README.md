# How to Use Podman on IBM Power (ppc64le)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Podman, IBM Power, ppc64le, Containers, Linux, DevOps, Enterprise, POWER10

Description: Learn to install and run Podman on IBM Power (ppc64le) servers, covering image compatibility, multi-arch builds, and AI workloads on POWER9/POWER10.

---

> IBM Power servers deliver exceptional throughput for data-intensive workloads like databases, AI inference, and enterprise applications. Podman runs natively on the ppc64le (PowerPC 64-bit Little Endian) architecture, bringing modern container management to Power infrastructure without requiring a daemon or root privileges.

IBM Power systems are widely deployed in enterprises running SAP, Oracle, and AI workloads. The POWER9 and POWER10 processors offer high memory bandwidth, hardware threading with up to 8 threads per core, and specialized accelerators. Podman on ppc64le lets you containerize workloads that benefit from these hardware capabilities while maintaining the same container workflow you use on x86_64 and ARM.

---

## Prerequisites

Verify your Power system is running Linux in little-endian mode:

```bash
uname -m
# Expected output: ppc64le

```

Supported distributions on ppc64le:
- RHEL 8/9 for ppc64le (most common in enterprise Power environments)
- Ubuntu 22.04/24.04 for ppc64le
- SUSE Linux Enterprise Server 15 for ppc64le
- Fedora for ppc64le
- Debian 12 for ppc64le

---

## Installing Podman on IBM Power

### RHEL 9 for ppc64le

```bash
sudo dnf install -y podman podman-compose skopeo buildah
```

### RHEL 8 for ppc64le

```bash
sudo dnf module enable -y container-tools:rhel8
sudo dnf install -y podman podman-compose skopeo buildah
```

### Ubuntu for ppc64le

```bash
sudo apt update
sudo apt install -y podman slirp4netns fuse-overlayfs uidmap
```

### SUSE Linux Enterprise Server 15

```bash
sudo zypper install -y podman
```

Verify the installation:

```bash
podman version
podman info --format '{{.Host.Arch}}'
# Expected: ppc64le
```

---

## Container Image Compatibility

### Checking ppc64le Support

Many official images include ppc64le builds. Verify support before deploying:

```bash
skopeo inspect --raw docker://docker.io/library/python:3.12-slim | \
  python3 -c "
import sys, json
data = json.load(sys.stdin)
if 'manifests' in data:
    for m in data['manifests']:
        if m['platform']['architecture'] == 'ppc64le':
            print('ppc64le is supported')
            break
    else:
        print('ppc64le is NOT supported')
"
```

### Commonly Available ppc64le Images

These official images typically include ppc64le variants:

```bash
# Verified ppc64le support
podman pull docker.io/library/ubuntu:24.04
podman pull docker.io/library/alpine:3.19
podman pull docker.io/library/python:3.12
podman pull docker.io/library/golang:1.22
podman pull docker.io/library/node:20
podman pull docker.io/library/postgres:16
podman pull docker.io/library/redis:7
podman pull docker.io/library/nginx:alpine
```

IBM provides Power-optimized images:

```bash
podman pull icr.io/ppc64le-oss/nginx-ppc64le:latest
```

---

## Running Workloads on IBM Power

### Database Workloads

IBM Power excels at database workloads due to high memory bandwidth and I/O throughput. Run PostgreSQL:

```bash
podman run -d --name postgres \
  -p 5432:5432 \
  -e POSTGRES_USER=poweruser \
  -e POSTGRES_PASSWORD=secure_password \
  -e POSTGRES_DB=enterprise \
  -v pgdata:/var/lib/postgresql/data:Z \
  --memory 8g \
  --cpus 4 \
  docker.io/library/postgres:16

podman exec -it postgres psql -U poweruser -d enterprise -c "SHOW server_version;"
```

Configure PostgreSQL for Power hardware by tuning shared buffers and work memory:

```bash
cat > postgresql-power.conf <<EOF
shared_buffers = 2GB
effective_cache_size = 6GB
work_mem = 256MB
maintenance_work_mem = 512MB
max_worker_processes = 8
max_parallel_workers_per_gather = 4
max_parallel_workers = 8
EOF

podman run -d --name postgres-tuned \
  -p 5432:5432 \
  -e POSTGRES_USER=poweruser \
  -e POSTGRES_PASSWORD=secure_password \
  -v pgdata:/var/lib/postgresql/data:Z \
  -v $(pwd)/postgresql-power.conf:/etc/postgresql/postgresql.conf:ro,Z \
  --memory 8g \
  docker.io/library/postgres:16 \
  -c config_file=/etc/postgresql/postgresql.conf
```

### Redis on Power

```bash
podman run -d --name redis \
  -p 6379:6379 \
  -v redis-data:/data:Z \
  --memory 4g \
  docker.io/library/redis:7-alpine \
  redis-server --appendonly yes --maxmemory 3gb --maxmemory-policy allkeys-lru
```

---

## Leveraging POWER Hardware Features

### SMT (Simultaneous Multi-Threading)

POWER9 supports 4 or 8 threads per core, and POWER10 supports 8 threads per core. Container workloads automatically benefit from SMT:

```bash
# Check SMT status
ppc64_cpu --smt

# Check available threads
nproc
```

Assign CPU resources based on thread count:

```bash
# On a POWER10 with 8 threads per core, allocate 1 full core (8 threads)
podman run -d --name compute-heavy \
  --cpus 8 \
  docker.io/library/python:3.12-slim \
  python -c "
import multiprocessing
print(f'Available CPUs: {multiprocessing.cpu_count()}')
"
```

### Large Pages

IBM Power supports large pages (16MB and 2MB) for memory-intensive workloads:

```bash
# Check huge pages on the host
cat /proc/meminfo | grep Huge

# Run a container with huge page access
podman run -d --name db-hugepages \
  --memory 8g \
  --shm-size 2g \
  docker.io/library/postgres:16
```

### Hardware Random Number Generator

Power systems include a hardware RNG. Containers can access it:

```bash
podman run --rm \
  --device /dev/hwrng:/dev/hwrng \
  docker.io/library/alpine:3.19 \
  dd if=/dev/hwrng bs=32 count=1 2>/dev/null | xxd
```

---

## Building ppc64le Container Images

### Native Builds

Building directly on Power hardware produces optimized ppc64le images:

```bash
cat > Dockerfile <<EOF
FROM docker.io/library/golang:1.22 AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOARCH=ppc64le go build -o /server .

FROM docker.io/library/alpine:3.19
COPY --from=builder /server /usr/local/bin/server
EXPOSE 8080
CMD ["server"]
EOF

podman build -t myorg/server:ppc64le .
```

### Power-Optimized Compiler Flags

When building C/C++ applications, use Power-specific optimizations:

```bash
cat > Dockerfile.power-optimized <<EOF
FROM docker.io/library/gcc:13
WORKDIR /app
COPY . .

# POWER9 optimizations
RUN gcc -O3 -mcpu=power9 -mtune=power10 \
  -mvsx -maltivec \
  -o myapp main.c

FROM docker.io/library/ubuntu:24.04
COPY --from=0 /app/myapp /usr/local/bin/
CMD ["myapp"]
EOF

podman build -f Dockerfile.power-optimized -t myorg/optimized-app:ppc64le .
```

### Multi-Architecture Manifests

Include ppc64le in your multi-architecture builds:

```bash
# Build natively on each platform, then combine
podman manifest create myorg/app:latest
podman manifest add myorg/app:latest docker://myorg/app:amd64
podman manifest add myorg/app:latest docker://myorg/app:arm64
podman manifest add myorg/app:latest docker://myorg/app:ppc64le
podman manifest push myorg/app:latest docker://docker.io/myorg/app:latest
```

---

## AI and Machine Learning on Power

IBM Power systems are used for AI inference workloads. Run containerized ML applications:

```bash
cat > Dockerfile.ml <<EOF
FROM docker.io/library/python:3.12-slim
WORKDIR /app

RUN pip install --no-cache-dir numpy pandas scikit-learn

COPY model.py .
COPY data/ ./data/
EXPOSE 8080
CMD ["python", "model.py"]
EOF

podman build -f Dockerfile.ml -t myorg/ml-inference:ppc64le .

podman run -d --name ml-service \
  -p 8080:8080 \
  --memory 16g \
  --cpus 8 \
  myorg/ml-inference:ppc64le
```

---

## Production Deployment with Systemd

### Quadlet Configuration

```bash
mkdir -p ~/.config/containers/systemd

cat > ~/.config/containers/systemd/enterprise-app.container <<EOF
[Unit]
Description=Enterprise Application on Power
After=network-online.target

[Container]
Image=myorg/enterprise-app:ppc64le
PublishPort=8443:8443
Volume=app-data:/data:Z
Volume=app-config:/etc/app:ro,Z
Environment=JAVA_OPTS=-Xmx4g -XX:+UseG1GC
Environment=APP_THREADS=32
AutoUpdate=registry

[Service]
Restart=always
TimeoutStartSec=180

[Install]
WantedBy=default.target
EOF

systemctl --user daemon-reload
systemctl --user enable --now enterprise-app
```

### Monitoring Resource Usage

```bash
# Real-time container stats
podman stats --format "table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}\t{{.NetIO}}\t{{.BlockIO}}"

# Check Power-specific CPU info inside a container
podman exec enterprise-app cat /proc/cpuinfo | head -30
```

---

## Cross-Building ppc64le Images on x86_64

If you do not have access to Power hardware for building:

```bash
# Install QEMU user-static on x86_64
sudo apt install -y qemu-user-static

# Verify ppc64le emulation is registered
ls /proc/sys/fs/binfmt_misc/qemu-ppc64le

# Build for ppc64le
podman build --platform linux/ppc64le -t myorg/app:ppc64le .
```

Note that emulated builds are significantly slower than native builds. For production images, build natively on Power hardware or use a CI/CD pipeline with Power runners.

---

## Troubleshooting

### Exec Format Error

If you see "exec format error," you are running an image built for a different architecture:

```bash
# Verify image architecture
podman inspect myimage:latest --format '{{.Architecture}}'

# Pull the correct platform
podman pull --platform linux/ppc64le docker.io/library/alpine:3.19
```

### Performance Issues

Check that SMT is properly configured:

```bash
ppc64_cpu --smt
ppc64_cpu --frequency
```

Ensure the container is not being throttled:

```bash
podman stats --no-stream
```

---

## Conclusion

Podman on IBM Power brings containerization to one of the most powerful server architectures available. The ppc64le platform offers exceptional throughput for database, AI, and enterprise workloads, and Podman's rootless, daemonless design fits well in the security-conscious Power ecosystem. Build natively on Power hardware to take advantage of POWER9 and POWER10 instruction sets, use multi-architecture manifests for cross-platform compatibility, and leverage SMT and large page support for performance-critical applications. With Quadlet and systemd integration, you get production-grade container lifecycle management that aligns with enterprise operations practices.
