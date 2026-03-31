# How to Install ClickHouse on ARM Architecture

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, ARM, aarch64, Raspberry Pi, Installation

Description: Install ClickHouse on ARM architecture (aarch64) including AWS Graviton, Apple Silicon, and Raspberry Pi, using official ARM binaries or Docker.

---

ARM processors are increasingly common in production environments - from AWS Graviton instances to Apple Silicon Macs to Raspberry Pi edge devices. ClickHouse provides official ARM64 binaries that deliver excellent performance on ARM hardware.

## ARM64 Official Binary

ClickHouse provides pre-compiled aarch64 binaries:

```bash
# Download ARM64 binary
curl -fsSL \
  "https://github.com/ClickHouse/ClickHouse/releases/latest/download/clickhouse-common-static-aarch64.tgz" \
  -o clickhouse-aarch64.tgz

tar xzf clickhouse-aarch64.tgz
sudo mv clickhouse-common-static-aarch64/usr/bin/clickhouse /usr/local/bin/
```

Or use the installer script which auto-detects architecture:

```bash
curl https://clickhouse.com/ | sh
```

## Installing on AWS Graviton (Ubuntu/Amazon Linux)

Graviton instances run standard Linux distributions. Use the package manager:

```bash
# Ubuntu on Graviton
sudo apt-get install -y apt-transport-https ca-certificates curl gnupg
curl -fsSL 'https://packages.clickhouse.com/deb/archive/apt/stable.sources' | \
  sudo tee /etc/apt/sources.list.d/clickhouse.sources
sudo apt-get update && sudo apt-get install -y clickhouse-server clickhouse-client
sudo systemctl enable --now clickhouse-server
```

The official ClickHouse packages include ARM64 binaries.

## Docker on ARM

The official ClickHouse Docker image supports `linux/arm64`:

```bash
docker run -d \
  --name clickhouse \
  --platform linux/arm64 \
  -p 8123:8123 \
  -p 9000:9000 \
  -v /var/lib/clickhouse:/var/lib/clickhouse \
  clickhouse/clickhouse-server:latest
```

On Apple Silicon Macs, Docker Desktop automatically uses the ARM image.

## Installing on Raspberry Pi 5 (aarch64)

Raspberry Pi 5 runs 64-bit Ubuntu 24.04, making it compatible with the standard ARM64 packages:

```bash
# Raspberry Pi OS (64-bit) / Ubuntu arm64
sudo apt-get update
curl -fsSL 'https://packages.clickhouse.com/deb/archive/apt/stable.sources' | \
  sudo tee /etc/apt/sources.list.d/clickhouse.sources
sudo apt-get update && sudo apt-get install -y clickhouse-server clickhouse-client
```

Note: Pi 4 and earlier have limited RAM. Configure ClickHouse memory limits appropriately:

```xml
<clickhouse>
  <profiles>
    <default>
      <max_memory_usage>3000000000</max_memory_usage>
    </default>
  </profiles>
</clickhouse>
```

## ARM Performance Considerations

ClickHouse includes ARM-specific optimizations including NEON SIMD instructions. On Graviton3 and Apple Silicon:

```sql
-- Check if vectorized execution is working
SELECT value FROM system.settings WHERE name = 'allow_experimental_query_cache';

-- Run a vectorized query benchmark
SELECT sum(number) FROM numbers(1000000000);
```

## Verifying ARM Architecture

```bash
# Confirm ClickHouse is using the ARM binary
file /usr/local/bin/clickhouse
# Output: ELF 64-bit LSB executable, ARM aarch64

clickhouse-client --query "SELECT version()"
```

## Building from Source for ARM

If pre-built binaries are unavailable for your ARM platform:

```bash
cmake .. \
  -DCMAKE_BUILD_TYPE=Release \
  -DCMAKE_C_COMPILER=clang \
  -DCMAKE_CXX_COMPILER=clang++ \
  -DARCH_NATIVE=ON \
  -G Ninja
ninja -j$(nproc) clickhouse-server clickhouse-client
```

The `-DARCH_NATIVE=ON` flag enables CPU-specific optimizations.

## Summary

Installing ClickHouse on ARM architecture uses the same official package repositories as x86-64, as ClickHouse provides native arm64 binaries. AWS Graviton instances offer the best production performance, while Docker provides a convenient option for Apple Silicon development. For resource-constrained ARM devices like Raspberry Pi, configure memory limits in ClickHouse's profile settings.
