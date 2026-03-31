# How to Write a ClickHouse Cluster Status Dashboard Script

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Cluster, Script, Dashboard, Monitoring

Description: Build a terminal dashboard script that displays ClickHouse cluster status including shard health, replication lag, disk usage, and query load across all nodes.

---

## Cluster vs. Single-Node Monitoring

When running a ClickHouse cluster with multiple shards and replicas, you need visibility across all nodes simultaneously. This script queries each node and produces a unified dashboard.

## Cluster Configuration

Define your cluster nodes:

```bash
# cluster-nodes.conf
CH1="node1.internal:8123"
CH2="node2.internal:8123"
CH3="node3.internal:8123"
```

## The Cluster Dashboard Script

```bash
#!/usr/bin/env bash
# clickhouse-cluster-dashboard.sh

CLUSTER_NAME="${CLUSTER_NAME:-production}"
NODES_FILE="${NODES_FILE:-cluster-nodes.conf}"
CH_USER="${CH_USER:-default}"
CH_PASSWORD="${CH_PASSWORD:-}"

source "$NODES_FILE"
NODES=($CH1 $CH2 $CH3)

query_node() {
    local node="$1"
    local sql="$2"
    curl -s --max-time 5 "http://${node}/" \
        -u "${CH_USER}:${CH_PASSWORD}" \
        --data-binary "$sql"
}

clear
echo "======================================"
echo " ClickHouse Cluster Dashboard"
echo " Cluster: ${CLUSTER_NAME} | $(date)"
echo "======================================"
echo ""

for node in "${NODES[@]}"; do
    echo "--- Node: ${node} ---"

    UPTIME=$(query_node "$node" "SELECT uptime() FORMAT TabSeparated")
    VERSION=$(query_node "$node" "SELECT version() FORMAT TabSeparated")
    QUERIES=$(query_node "$node" "SELECT count() FROM system.processes FORMAT TabSeparated")

    echo "  Version: ${VERSION} | Uptime: ${UPTIME}s | Active queries: ${QUERIES}"

    # Disk usage
    echo "  Disk usage:"
    query_node "$node" "
    SELECT
        name,
        round((total_space - free_space) * 100.0 / total_space, 1) AS used_pct,
        formatReadableSize(free_space) AS free
    FROM system.disks
    FORMAT TabSeparated
    " | while IFS=$'\t' read -r name pct free; do
        echo "    ${name}: ${pct}% used (${free} free)"
    done

    # Replication lag
    MAX_LAG=$(query_node "$node" "
    SELECT max(absolute_delay) FROM system.replicas FORMAT TabSeparated
    ")
    echo "  Max replication lag: ${MAX_LAG}s"

    # Part count
    PARTS=$(query_node "$node" "
    SELECT count() FROM system.parts WHERE active = 1 FORMAT TabSeparated
    ")
    echo "  Total active parts: ${PARTS}"
    echo ""
done

echo "--- Cross-Cluster Replication Status ---"
# Query any node for cluster-wide view via remote()
query_node "${NODES[0]}" "
SELECT
    host_name,
    host_address,
    port,
    is_local,
    user,
    default_database
FROM system.clusters
WHERE cluster = '${CLUSTER_NAME}'
FORMAT PrettyCompactMonoBlock
"
```

## Auto-Refresh with watch

```bash
# Refresh every 5 seconds
watch -n 5 ./clickhouse-cluster-dashboard.sh
```

## Summary

The cluster status dashboard script queries each ClickHouse node individually for version, uptime, active queries, disk usage, replication lag, and part counts, then presents a unified view. Combining it with `watch` provides a live terminal dashboard for cluster operations.
