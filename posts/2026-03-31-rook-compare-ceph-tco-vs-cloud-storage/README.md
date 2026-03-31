# How to Compare Ceph TCO vs Cloud Storage Costs

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Cloud Storage, Cost Comparison, TCO, AWS S3

Description: Compare the true cost of self-managed Ceph storage against AWS S3 and Azure Blob Storage using a realistic TCO model with egress and API call costs.

---

## Why Cloud Storage Costs Are Deceptive

Cloud storage pricing looks cheap at first glance - $0.023/GB/month for AWS S3 Standard - but egress fees, API call costs, and multi-region replication can multiply the bill significantly for active workloads.

## Cloud Storage Pricing Model

```bash
# AWS S3 Standard pricing (us-east-1, approximate)
# Storage: $0.023/GB/month = $23/TB/month
# GET requests: $0.0004 per 1,000
# PUT requests: $0.005 per 1,000
# Egress (after 1 GB free): $0.09/GB = $90/TB

# Example: 100 TB stored, 10 TB/month egress, 10M GETs/month
# Storage:  100 TB x $23 = $2,300/month
# Egress:   10 TB x $90 = $900/month
# GETs:     10M x $0.0004 = $4/month
# Total:    ~$3,204/month = $38,448/year
```

## Ceph On-Premises Cost Model

```bash
# Hardware (amortized over 5 years)
# 10 nodes x $8,000 = $80,000 / 60 months = $1,333/month

# Operations
# Power + cooling: $250/month
# Staff (0.5 FTE): $6,250/month
# Support: $200/month

# Total monthly: $8,033/month for 160 TB usable
# Per TB: $50/month

# With compression (2:1 ratio): effective $25/TB/month
```

## Side-by-Side Comparison

```bash
# At 100 TB usable, moderate egress workload:
#
# Cloud (S3):
#   - Storage: $2,300/month
#   - Egress (10 TB): $900/month
#   - Total: $3,200/month
#
# Ceph on-prem:
#   - Hardware amortization: $833/month (for 100 TB portion)
#   - Staff (allocated): $3,125/month
#   - Power: $125/month
#   - Total: $4,083/month
#
# Crossover point: ~200+ TB with high egress makes Ceph cheaper
```

## When Ceph Wins on Cost

Ceph becomes more cost-effective when:
- Stored data exceeds 200 TB usable
- Egress exceeds 20% of stored data per month
- You already have staff managing Kubernetes/storage
- Regulatory requirements demand on-premises data residency

```bash
# Calculate your break-even point
python3 << 'EOF'
# Monthly cloud costs
stored_tb = 200
monthly_egress_tb = 20
cloud_storage = stored_tb * 23  # $/month
cloud_egress = monthly_egress_tb * 90
cloud_total = cloud_storage + cloud_egress

# Monthly Ceph costs (rough)
hardware_amort = stored_tb * 3.33  # $200/TB/year hardware / 12
staff = 6250
power = 250
ceph_total = hardware_amort + staff + power

print(f"Cloud: ${cloud_total:,.0f}/month")
print(f"Ceph: ${ceph_total:,.0f}/month")
print(f"{'Ceph' if ceph_total < cloud_total else 'Cloud'} is cheaper")
EOF
```

## Hidden Cloud Costs to Include

```bash
# Data transfer IN is free on most clouds, but:
# - Multi-region replication: +$0.02/GB
# - Cross-AZ traffic: $0.01/GB each way
# - Glacier restore: $0.01-0.03/GB
# - API rate limits may require premium tier
```

## Summary

Cloud storage is cost-competitive for small datasets with low egress, but self-managed Ceph storage typically wins for deployments over 200 TB with active workload patterns. Include egress, API, and replication costs in your cloud analysis - not just storage pricing - to get an accurate comparison.
