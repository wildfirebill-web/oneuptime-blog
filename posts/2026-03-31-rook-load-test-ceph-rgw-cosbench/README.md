# How to Load Test Ceph RGW with COSBench

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, COSBench, Load Testing, RGW, Performance

Description: Use COSBench to load test Ceph RADOS Gateway (RGW) S3 endpoints, measuring throughput, IOPS, and latency under sustained object storage workloads.

---

COSBench (Cloud Object Storage Benchmark) is a distributed load testing tool designed specifically for object storage systems. It drives realistic S3 workloads against Ceph RGW and produces detailed performance reports.

## Installing COSBench

```bash
# Download COSBench
wget https://github.com/intel-cloud/cosbench/releases/download/v0.4.2.c4/0.4.2.c4.zip
unzip 0.4.2.c4.zip
cd 0.4.2.c4

# Install Java dependency
sudo apt-get install -y openjdk-11-jdk

# Make scripts executable
chmod +x *.sh
```

## Configuring COSBench for Ceph RGW

Create a workload XML file that targets your RGW endpoint:

```xml
<!-- ceph-rgw-workload.xml -->
<?xml version="1.0" encoding="UTF-8" ?>
<workload name="ceph-rgw-load-test" description="Ceph RGW S3 benchmark">

  <storage type="s3" config="accesskey=testuser;secretkey=testpass;endpoint=http://rgw.example.com:80;path_style_access=true"/>

  <workflow>

    <!-- Prepare: create buckets and fill with objects -->
    <workstage name="prepare">
      <work type="init" workers="4" config="cprefix=bench-bucket;containers=r(1,4)"/>
      <work type="prepare" workers="8" workers="8"
        config="cprefix=bench-bucket;containers=r(1,4);objects=r(1,1000);sizes=c(1)MB"/>
    </workstage>

    <!-- Mixed read/write workload -->
    <workstage name="main">
      <work name="write" workers="16" runtime="300">
        <operation type="write" ratio="30" config="cprefix=bench-bucket;containers=u(1,4);objects=u(1,2000);sizes=c(1)MB"/>
      </work>
      <work name="read" workers="16" runtime="300">
        <operation type="read" ratio="60" config="cprefix=bench-bucket;containers=u(1,4);objects=u(1,1000)"/>
      </work>
      <work name="delete" workers="4" runtime="300">
        <operation type="delete" ratio="10" config="cprefix=bench-bucket;containers=u(1,4);objects=u(1,500)"/>
      </work>
    </workstage>

    <!-- Cleanup -->
    <workstage name="cleanup">
      <work type="cleanup" workers="4" config="cprefix=bench-bucket;containers=r(1,4);objects=r(1,2000)"/>
    </workstage>

  </workflow>
</workload>
```

## Starting COSBench and Running the Test

```bash
# Start the controller
./start-controller.sh

# Start workers (run on multiple machines for distributed load)
./start-driver.sh

# Submit the workload
./cli.sh submit ceph-rgw-workload.xml

# Check job status
./cli.sh info

# List all runs
./cli.sh ls
```

## Scaling COSBench Workers in Kubernetes

Run COSBench workers as Kubernetes pods for distributed load generation:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cosbench-driver
  namespace: bench
spec:
  replicas: 4
  selector:
    matchLabels:
      app: cosbench-driver
  template:
    metadata:
      labels:
        app: cosbench-driver
    spec:
      containers:
        - name: driver
          image: cosbench/cosbench:0.4.2
          ports:
            - containerPort: 18088
          env:
            - name: COSBENCH_ROLE
              value: driver
          resources:
            requests:
              cpu: "2"
              memory: "2Gi"
---
apiVersion: v1
kind: Service
metadata:
  name: cosbench-driver
  namespace: bench
spec:
  selector:
    app: cosbench-driver
  ports:
    - port: 18088
      targetPort: 18088
  clusterIP: None
```

## Reading COSBench Results

```bash
# Results are stored in archive/ directory
ls archive/w1-ceph-rgw-load-test/

# Key metrics in the CSV files
# - Throughput (ops/sec)
# - Bandwidth (MB/s)
# - Average Response Time (ms)
# - 95th/99th percentile latency

# Parse results with Python
python3 << 'EOF'
import csv
import glob

for report in glob.glob("archive/*/w1-main/s1-main/c1-read/*.csv"):
    with open(report) as f:
        reader = csv.DictReader(f)
        for row in reader:
            print(f"Throughput: {row.get('Throughput', 'N/A')} ops/sec")
            print(f"Bandwidth: {row.get('Bandwidth', 'N/A')} MB/s")
            print(f"Avg RT: {row.get('Avg-ResTime', 'N/A')} ms")
EOF
```

## Analyzing RGW Performance During the Test

```bash
# While COSBench is running, monitor RGW stats in another terminal
watch -n 2 'kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  radosgw-admin usage show --show-log-entries=false'

# Monitor RGW pod resource usage
kubectl -n rook-ceph top pods -l app=rook-ceph-rgw
```

## Summary

COSBench provides realistic, sustained S3 workload testing against Ceph RGW with configurable read/write ratios, object sizes, and worker counts. Running it as Kubernetes pods scales the load generation horizontally, and the CSV result files give detailed per-operation throughput and latency metrics for comparing RGW configurations and hardware setups.
