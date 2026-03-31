# How to Configure Ceph Storage for Gaming Industry

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Gaming, Game Server, Save Data, Asset Storage, Object Storage

Description: Configure Rook/Ceph storage for gaming workloads including game asset distribution, player save data, game server state, telemetry pipelines, and leaderboard backends.

---

## Gaming Storage Requirements

Modern game platforms require storage across the full game lifecycle:
- **Game assets**: Binary packages (1-100 GB per title) for distribution
- **Player data**: Save games, player progression, account data
- **Game server state**: Real-time game state with fast writes
- **Telemetry**: Millions of events per second from game clients
- **Leaderboards**: Fast read/write for ranking data
- **Replay data**: Large sequential video/state recordings

## Game Asset Distribution with RGW

Game assets are large binary blobs downloaded by millions of players:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephObjectStore
metadata:
  name: game-assets
  namespace: rook-ceph
spec:
  metadataPool:
    replicated:
      size: 3
  dataPool:
    replicated:
      size: 3
    parameters:
      compression_mode: none  # Game assets are already compressed
  gateway:
    instances: 8  # High gateway count for massive concurrent downloads
    securePort: 443
    sslCertificateRef: gaming-cdn-cert
```

## Configuring for High-Throughput Downloads

Optimize RGW for large object serving:

```bash
# Tune RGW for large object reads
kubectl exec -n rook-ceph deploy/rook-ceph-tools -- \
  ceph config set client.rgw rgw_max_chunk_size 4194304  # 4MB chunks

# Enable async writes for faster uploads from game patch servers
kubectl exec -n rook-ceph deploy/rook-ceph-tools -- \
  ceph config set client.rgw rgw_enable_ops_log false  # Disable for performance
```

## Player Save Data Storage

Player saves need durability and fast individual record access:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: player-saves-db
  namespace: gaming
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: rook-ceph-block
  resources:
    requests:
      storage: 1Ti
```

Use with a key-value store like Redis or DynamoDB-compatible Cassandra:

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: scylla-player-data
  namespace: gaming
spec:
  volumeClaimTemplates:
    - metadata:
        name: scylla-data
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: rook-ceph-block
        resources:
          requests:
            storage: 200Gi
```

## Game Telemetry Ingest

Use RGW S3 as a telemetry sink, then process with Spark:

```python
import boto3
import json

s3 = boto3.client(
    's3',
    endpoint_url='http://rook-ceph-rgw-game-assets.rook-ceph.svc'
)

def store_telemetry_batch(events: list, game_id: str):
    """Store a batch of telemetry events partitioned by date/hour."""
    from datetime import datetime
    now = datetime.utcnow()
    key = f"telemetry/{game_id}/{now.year}/{now.month:02d}/{now.day:02d}/{now.hour:02d}/batch-{now.timestamp()}.json"

    s3.put_object(
        Bucket='game-telemetry',
        Key=key,
        Body=json.dumps(events),
        ContentType='application/json'
    )
```

## Replay Storage

Game replays are large sequential writes followed by reads:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: replay-storage
  namespace: gaming
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: rook-ceph-block
  resources:
    requests:
      storage: 5Ti
```

## Handling Game Launch Events

Scale RGW before scheduled game launches:

```bash
# Pre-scale for game launch day
kubectl patch cephobjectstore game-assets -n rook-ceph \
  --type='json' \
  -p='[{"op": "replace", "path": "/spec/gateway/instances", "value": 16}]'

# Monitor download throughput
watch -n5 "kubectl exec -n rook-ceph deploy/rook-ceph-tools -- \
  ceph status | grep -E 'io|client'"
```

## Summary

Ceph on Rook handles the bursty, high-throughput demands of gaming storage: RGW with multiple gateway instances for game asset distribution at launch, RBD-backed databases for durable player save data, S3 bucket sinks for telemetry ingest, and block storage for replay archives. Pre-scaling RGW instances before game launch events ensures storage infrastructure doesn't become the bottleneck during peak download periods.
