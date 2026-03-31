# How to Install ClickHouse from Source

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Source Build, CMake, Clang, Development

Description: Build and install ClickHouse from source code on Linux using CMake and Clang, covering prerequisites, build configuration, and installation steps.

---

Building ClickHouse from source gives you access to unreleased features, allows custom patches, and is required for some platforms where pre-compiled binaries are unavailable. The build requires a modern Linux system with at least 16 GB RAM.

## Prerequisites

ClickHouse requires Clang 17+ and CMake 3.20+:

```bash
# Ubuntu/Debian
sudo apt-get update
sudo apt-get install -y \
  git cmake ninja-build clang-17 \
  libssl-dev libicu-dev python3 \
  nasm yasm gawk

# Set Clang 17 as default
sudo update-alternatives --install /usr/bin/clang clang /usr/bin/clang-17 100
sudo update-alternatives --install /usr/bin/clang++ clang++ /usr/bin/clang++-17 100
```

## Cloning the Repository

```bash
git clone --recursive https://github.com/ClickHouse/ClickHouse.git
cd ClickHouse
```

The `--recursive` flag is required to pull submodules. The repository is large (several GB).

## Configuring the Build

```bash
mkdir build
cd build

cmake .. \
  -DCMAKE_BUILD_TYPE=RelWithDebInfo \
  -DCMAKE_C_COMPILER=clang \
  -DCMAKE_CXX_COMPILER=clang++ \
  -G Ninja
```

Build type options:
- `Release` - Fully optimized, no debug symbols
- `RelWithDebInfo` - Optimized with debug symbols (recommended)
- `Debug` - Unoptimized, useful for development

## Building

```bash
# Build the server and client
ninja clickhouse-server clickhouse-client

# Or build everything (takes much longer)
ninja
```

The build takes 30-90 minutes depending on your hardware. Use all available cores:

```bash
ninja -j$(nproc) clickhouse-server clickhouse-client
```

## Installing

```bash
# Copy binaries
sudo cp programs/clickhouse /usr/local/bin/
sudo ln -s /usr/local/bin/clickhouse /usr/local/bin/clickhouse-server
sudo ln -s /usr/local/bin/clickhouse /usr/local/bin/clickhouse-client

# Create user and directories
sudo useradd -r -s /sbin/nologin -d /var/lib/clickhouse clickhouse
sudo mkdir -p /var/lib/clickhouse /var/log/clickhouse-server /etc/clickhouse-server
sudo chown -R clickhouse:clickhouse /var/lib/clickhouse /var/log/clickhouse-server

# Copy default config
sudo cp ../programs/server/config.xml /etc/clickhouse-server/
sudo cp ../programs/server/users.xml /etc/clickhouse-server/
```

## Building a Specific Version

```bash
git checkout v24.3.1.2672
git submodule update --recursive
mkdir build-24.3 && cd build-24.3
cmake .. -DCMAKE_BUILD_TYPE=Release -G Ninja
ninja -j$(nproc) clickhouse-server clickhouse-client
```

## Minimal Build for Testing

For a faster development build with fewer features:

```bash
cmake .. \
  -DCMAKE_BUILD_TYPE=Debug \
  -DENABLE_TESTS=ON \
  -DENABLE_CLICKHOUSE_ALL=OFF \
  -DENABLE_CLICKHOUSE_SERVER=ON \
  -DENABLE_CLICKHOUSE_CLIENT=ON \
  -G Ninja
```

## Verifying the Build

```bash
./programs/clickhouse server --config-file=../programs/server/config.xml &
./programs/clickhouse client --query "SELECT version()"
```

## Summary

Building ClickHouse from source requires Clang 17+, CMake 3.20+, Ninja, and at least 16 GB of RAM. Clone with `--recursive` to get submodules, configure with CMake using RelWithDebInfo for production or Debug for development, and build only the server and client binaries to save time. Source builds enable custom patches and access to unreleased features.
