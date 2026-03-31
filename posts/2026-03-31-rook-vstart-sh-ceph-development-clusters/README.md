# How to Use vstart.sh for Ceph Development Clusters

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Development, vstart, Testing, Debugging

Description: Use the vstart.sh script from the Ceph source tree to spin up minimal local clusters for core Ceph development, debugging, and unit testing without Kubernetes.

---

`vstart.sh` is the developer's go-to tool for running a minimal Ceph cluster from a local build. It starts monitors, managers, OSDs, and gateways in seconds, making it the fastest way to test Ceph code changes without full cluster setup.

## What is vstart.sh?

`vstart.sh` is a shell script in the Ceph source tree (`src/vstart.sh`) that launches Ceph daemons from your local build directory. It creates a temporary cluster for quick iteration without Kubernetes or production infrastructure.

## Building Ceph from Source

```bash
# Clone the Ceph repository
git clone https://github.com/ceph/ceph.git
cd ceph
git submodule update --init --recursive

# Install build dependencies
sudo apt-get install -y cmake gcc g++ python3-dev libboost-dev \
  libssl-dev liblz4-dev libsnappy-dev libbz2-dev

# Build (use -DCMAKE_BUILD_TYPE=Debug for development)
mkdir build && cd build
cmake .. \
  -DCMAKE_BUILD_TYPE=Debug \
  -DWITH_TESTS=ON \
  -DWITH_PYTHON3=ON

make -j$(nproc) vstart
```

## Starting a Development Cluster

```bash
# Navigate to the build directory
cd ~/ceph/build

# Start a minimal cluster with 3 OSDs using directory-backed storage
MON=1 OSD=3 MDS=1 MGR=1 RGW=1 ../src/vstart.sh -n -d

# Options:
# -n = start fresh (delete existing data)
# -d = use directory-backed OSDs (no raw devices needed)
# -b = use bluestore (default)
# -x = start with erasure coded pools

# The cluster config lands in /tmp/ceph-*
export CEPH_CONF=/tmp/ceph-<timestamp>/ceph.conf
```

## Interacting with the vstart Cluster

```bash
# The vstart.sh script adds bin/ to your PATH
# Commands work directly:
./bin/ceph status
./bin/ceph osd tree
./bin/ceph df

# Create a test pool
./bin/ceph osd pool create test 16 16

# Write and read an object
echo "hello ceph" | ./bin/rados -p test put myobj -
./bin/rados -p test get myobj -

# Run the RGW admin API
./bin/radosgw-admin user create --uid=testuser --display-name="Test"
```

## Running Ceph Unit Tests

```bash
# From the build directory, run the full test suite
ctest -j4 -L unittest_librados

# Run a specific test
./unittest_librados

# Run tests with vstart cluster backend
cd ~/ceph/src
python3 -m pytest tests/pybind/test_rados.py -v
```

## Restarting and Resetting

```bash
# Stop the cluster
../src/stop.sh

# Restart without losing data
MON=1 OSD=3 ../src/vstart.sh -d

# Start fresh (clears all data)
../src/vstart.sh -n -d

# Remove all cluster data manually
rm -rf /tmp/ceph-<timestamp>
```

## Useful vstart.sh Options

```bash
# Start with Ceph dashboard enabled
MON=1 OSD=3 MGR=1 ../src/vstart.sh -n -d --with-dashboard

# Start with verbose logging
CEPH_ARGS="--debug-ms=1 --debug-osd=5" \
  MON=1 OSD=3 ../src/vstart.sh -n -d

# Use a custom configuration overlay
cat > /tmp/vstart-extra.conf << 'EOF'
[global]
debug_osd = 10
debug_ms = 1
EOF
MON=1 OSD=3 ../src/vstart.sh -n -d --extra-config /tmp/vstart-extra.conf
```

## Summary

`vstart.sh` provides the fastest path from Ceph source change to running cluster for developers. By using directory-backed OSDs you avoid needing raw block devices, and the script's `-n` flag makes it easy to start fresh for each test run. For Rook contributors, combining vstart.sh for core daemon testing with a kind-based cluster for operator testing covers the full development workflow.
