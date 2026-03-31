# How to Optimize ClickHouse for Cloud VM Instances

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Cloud, VM, Performance Tuning, AWS, GCP, Azure

Description: Tune ClickHouse configuration for cloud VM instances to maximize query performance and storage efficiency on AWS, GCP, and Azure compute.

---

Running ClickHouse on cloud VMs requires tuning for the specific characteristics of cloud storage, networking, and CPU. The default configuration is conservative - cloud instances can handle much more aggressive settings.

## Choose the Right VM Type

ClickHouse workloads are typically CPU and I/O bound. Recommended instance types:

```text
AWS:
- Compute: c6i.8xlarge (32 vCPU, 64GB) for query-heavy workloads
- Memory: r6i.4xlarge (16 vCPU, 128GB) for large GROUP BY queries
- Storage: i3en.3xlarge (NVMe SSD) for highest I/O throughput

GCP:
- n2-standard-32 for balanced workloads
- n2-highcpu-64 for CPU-heavy aggregations

Azure:
- Standard_D32s_v5 for general workloads
- Standard_L16s_v3 for local NVMe storage
```

## Configure CPU Parallelism

Set parallelism based on vCPU count:

```xml
<max_threads>32</max_threads>
<max_insert_threads>8</max_insert_threads>
<background_pool_size>16</background_pool_size>
<background_merges_mutations_concurrency_ratio>2</background_merges_mutations_concurrency_ratio>
```

## Use Object Storage for Cold Data

Configure S3 (or GCS/Azure Blob) as a tiered storage disk:

```xml
<storage_configuration>
  <disks>
    <s3_disk>
      <type>s3</type>
      <endpoint>https://s3.us-east-1.amazonaws.com/my-clickhouse-bucket/data/</endpoint>
      <access_key_id>AKIAIOSFODNN7EXAMPLE</access_key_id>
      <secret_access_key>wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY</secret_access_key>
    </s3_disk>
  </disks>
  <policies>
    <tiered>
      <volumes>
        <hot><disk>default</disk></hot>
        <cold><disk>s3_disk</disk></cold>
      </volumes>
      <move_factor>0.1</move_factor>
    </tiered>
  </policies>
</storage_configuration>
```

## Tune Network Buffers for High Bandwidth

Cloud VMs often have 10-25Gbps network bandwidth. Increase buffers to utilize it:

```xml
<tcp_send_buffer_size>4194304</tcp_send_buffer_size>
<tcp_receive_buffer_size>4194304</tcp_receive_buffer_size>
```

## Configure Mark Cache

Set mark cache based on available RAM (typically 20-30% of RAM for query memory):

```xml
<mark_cache_size>5368709120</mark_cache_size>  <!-- 5GB -->
<uncompressed_cache_size>8589934592</uncompressed_cache_size>  <!-- 8GB -->
```

## Disable Transparent Huge Pages

THP causes latency spikes on cloud VMs. Disable it at boot:

```bash
echo madvise | sudo tee /sys/kernel/mm/transparent_hugepage/enabled
echo defer+madvise | sudo tee /sys/kernel/mm/transparent_hugepage/defrag
```

Add to `/etc/rc.local` for persistence.

## Configure I/O Scheduler for Cloud Block Storage

For EBS/Persistent Disk (not NVMe), use the `none` scheduler:

```bash
echo none | sudo tee /sys/block/sda/queue/scheduler
```

## Monitor Cloud-Specific Metrics

Track EBS/disk I/O utilization:

```sql
SELECT
  event_time,
  ProfileEvents['ReadBufferFromFileDescriptorReadBytes'] AS disk_read_bytes,
  ProfileEvents['WriteBufferFromFileDescriptorWriteBytes'] AS disk_write_bytes,
  query_duration_ms,
  query
FROM system.query_log
WHERE type = 'QueryFinish' AND event_date = today()
ORDER BY disk_read_bytes DESC
LIMIT 10;
```

## Summary

Optimizing ClickHouse for cloud VMs requires choosing the right instance type (CPU-heavy or memory-heavy based on workload), tuning `max_threads` and cache sizes to match vCPU and RAM, configuring tiered S3 storage for cost-efficient cold data, disabling THP, and using the correct I/O scheduler for the underlying storage type. Monitor disk I/O in `system.query_log` to identify I/O-bound queries.
