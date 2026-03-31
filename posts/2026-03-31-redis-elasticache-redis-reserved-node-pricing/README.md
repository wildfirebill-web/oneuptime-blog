# How to Optimize ElastiCache Redis Reserved Node Pricing

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, ElastiCache, AWS, Cost Optimization, Reserved Node

Description: Learn how to reduce ElastiCache Redis costs by up to 55% using Reserved Nodes - covering term options, payment models, and strategies for right-sizing before committing.

---

ElastiCache Reserved Nodes let you pay upfront or partially upfront for 1 or 3-year terms in exchange for significant hourly rate discounts. For stable, always-on workloads, reservations typically save 30-55% compared to on-demand pricing.

## Reservation Options

| Term | Payment | Discount vs On-Demand |
|---|---|---|
| 1 year | All upfront | ~38% |
| 1 year | Partial upfront | ~34% |
| 1 year | No upfront | ~28% |
| 3 year | All upfront | ~55% |
| 3 year | Partial upfront | ~50% |
| 3 year | No upfront | ~44% |

## Before Reserving: Right-Size First

Check actual utilization over the past 2-4 weeks before committing:

```bash
# Check average CPU utilization
aws cloudwatch get-metric-statistics \
  --namespace AWS/ElastiCache \
  --metric-name CPUUtilization \
  --dimensions Name=CacheClusterId,Value=my-cluster-001 \
  --start-time 2026-03-01T00:00:00Z \
  --end-time 2026-03-31T00:00:00Z \
  --period 3600 \
  --statistics Average

# Check memory usage
aws cloudwatch get-metric-statistics \
  --namespace AWS/ElastiCache \
  --metric-name DatabaseMemoryUsagePercentage \
  --dimensions Name=CacheClusterId,Value=my-cluster-001 \
  --start-time 2026-03-01T00:00:00Z \
  --end-time 2026-03-31T00:00:00Z \
  --period 3600 \
  --statistics Maximum
```

## Purchasing a Reserved Node

```bash
aws elasticache purchase-reserved-cache-nodes-offering \
  --reserved-cache-nodes-offering-id <offering-id> \
  --cache-node-count 2 \
  --reserved-cache-node-id prod-redis-reservation
```

Find the offering ID first:

```bash
aws elasticache describe-reserved-cache-nodes-offerings \
  --cache-node-type cache.r7g.large \
  --product-description redis \
  --offering-type "All Upfront" \
  --duration 31536000 \
  --query "ReservedCacheNodesOfferings[*].{Id:ReservedCacheNodesOfferingId,Cost:FixedPrice,Hourly:UsagePrice}"
```

## Listing Active Reservations

```bash
aws elasticache describe-reserved-cache-nodes \
  --query "ReservedCacheNodes[*].{Id:ReservedCacheNodeId,Type:CacheNodeType,State:State,Start:StartTime,End:Duration}"
```

## Savings Estimation

Calculate break-even to decide between on-demand and reserved:

```python
on_demand_hourly = 0.218  # cache.r7g.large us-east-1
reserved_all_upfront_1yr = 1203.00  # total cost

hours_per_year = 8760
total_on_demand = on_demand_hourly * hours_per_year

savings = total_on_demand - reserved_all_upfront_1yr
pct = (savings / total_on_demand) * 100
print(f"Annual savings: ${savings:.2f} ({pct:.1f}%)")
# Annual savings: ~$705 (37%)
```

## Strategies for Variable Workloads

- Reserve only baseline capacity (e.g., 60% of peak nodes)
- Use on-demand for burst capacity
- Use Savings Plans for compute if using ElastiCache Serverless

## Summary

Reserved Nodes are the most effective way to reduce ElastiCache Redis costs for stable workloads - saving up to 55% over 3 years with all-upfront payment. Always right-size your nodes using CloudWatch metrics before committing to a reservation. Reserve your baseline capacity and keep a smaller on-demand buffer for traffic spikes.
