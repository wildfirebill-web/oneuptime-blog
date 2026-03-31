# How to Estimate Total Cost of Ownership for ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Total Cost of Ownership, Cost Estimation, Infrastructure, Cloud Cost

Description: A practical framework for estimating the total cost of ownership for ClickHouse, covering hardware, operations, storage, networking, and licensing.

---

## TCO Components for ClickHouse

Total cost of ownership (TCO) for ClickHouse has five main components: compute, storage, networking, operations, and licensing. Whether self-hosted or managed, each component must be quantified.

## Self-Hosted TCO Calculation

### Compute Costs

```text
Monthly compute cost = (number of nodes * hourly instance cost * 730 hours)

Example: 3 x r6i.4xlarge (128GB RAM, 16 vCPU) at $1.008/hour each
= 3 * $1.008 * 730 = $2,208/month
```

### Storage Costs

```sql
-- Get current and projected storage needs
SELECT
    formatReadableSize(sum(data_compressed_bytes)) AS current_compressed,
    formatReadableSize(sum(data_compressed_bytes) * 1.5) AS with_growth_buffer,
    round(sum(data_compressed_bytes) / 1e9, 1) AS compressed_gb
FROM system.parts
WHERE active;
```

```text
Storage cost = compressed_gb * 1.5 (growth) * $0.08/GB/month (EBS gp3)
Example: 2TB * 1.5 * $0.08 = $245/month per node
```

### Operations Costs

Self-hosted ClickHouse requires ongoing operational work:

```text
Operations overhead estimate:
- Initial setup and configuration: 40 hours * $150/hour = $6,000 one-time
- Monthly maintenance and monitoring: 10 hours * $150/hour = $1,500/month
- On-call incident response: 5 hours/month * $150/hour = $750/month
- Total monthly ops: ~$2,250/month
```

## ClickHouse Cloud TCO

```text
ClickHouse Cloud pricing (as of 2025):
- Compute: $0.21 per GiB-hour (memory-based)
- Storage: $0.023/GB/month

Example: 96GB service running 12 hours/day (auto-suspend nights/weekends)
Compute: 96 * $0.21 * (12 * 30) = $7,257/month (full time)
With 40% utilization via auto-suspend: $2,903/month
Storage: 2TB * $0.023 * 1024 = $47/month
Total: ~$2,950/month (no ops overhead for managed service)
```

## Side-by-Side Comparison Template

```text
Item                       | Self-Hosted    | ClickHouse Cloud
---------------------------|----------------|------------------
Compute (3 nodes)          | $2,208/month   | $2,903/month
Storage (2TB)              | $735/month     | $47/month
Networking                 | $100/month     | Included
Operations                 | $2,250/month   | $0/month
Backup storage             | $50/month      | Included
Security/patching          | 5hrs/month     | Included
Total                      | $5,343/month   | $2,950/month
```

## Hidden Costs to Include

1. **Disaster recovery**: Multi-AZ replication adds 2x storage cost for self-hosted
2. **Monitoring tooling**: Grafana, Prometheus, alerting stack
3. **Support contracts**: Enterprise support for production ClickHouse
4. **Data migration**: One-time cost when switching providers

```bash
# Estimate data migration time
# 1TB at 500MB/s = 2000 seconds = ~33 minutes
# Factor in transformation + validation: 3-5x = 1.5-2.5 hours per TB
```

## Summary

ClickHouse TCO includes compute, storage, networking, and operations. Self-hosted often appears cheaper on compute/storage but becomes more expensive once operations overhead is included. ClickHouse Cloud eliminates ops burden but costs more on compute. Build a complete model including all five cost components and at least 12 months of projected data growth before choosing a deployment model.
