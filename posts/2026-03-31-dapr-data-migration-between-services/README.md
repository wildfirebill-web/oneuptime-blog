# How to Implement Data Migration Between Services with Dapr

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Data Migration, Microservice, State Store, Architecture

Description: Migrate data between Dapr microservices safely using the strangler fig pattern, dual-write strategies, and state store bulk operations.

---

Migrating data between services is unavoidable as microservice boundaries evolve. Splitting a monolith, extracting a service, or moving data ownership requires careful migration strategies to avoid data loss or downtime. Dapr's state management APIs and pub/sub make data migration between services safer and more observable.

## Migration Strategies Overview

Three common approaches for Dapr-based data migration:

1. **Bulk copy** - copy all data at once, swap service, verify
2. **Dual write** - write to both old and new stores during migration
3. **Lazy migration** - migrate data on first read by new service

## Strategy 1: Bulk Copy Migration

Copy state from a source service's store to the destination:

```python
# migration/bulk_copy.py
from dapr.clients import DaprClient
import json

def bulk_migrate_state(
    source_store: str,
    dest_store: str,
    key_prefix: str,
    batch_size: int = 100
):
    """
    Migrates state keys with a given prefix from one store to another.
    Requires knowing the key list (from a separate index or scan).
    """
    with DaprClient() as client:
        # Get list of keys to migrate (maintained separately as an index)
        key_index = json.loads(
            client.get_state(source_store, f"{key_prefix}:index").data or "[]"
        )

        migrated = 0
        failed = 0

        for i in range(0, len(key_index), batch_size):
            batch = key_index[i:i + batch_size]
            for key in batch:
                try:
                    # Read from source
                    source_data = client.get_state(source_store, key)
                    if not source_data.data:
                        continue

                    # Write to destination
                    client.save_state(dest_store, key, source_data.data.decode())
                    migrated += 1

                    # Verify write
                    verify = client.get_state(dest_store, key)
                    if verify.data != source_data.data:
                        print(f"Verification failed for key: {key}")
                        failed += 1

                except Exception as e:
                    print(f"Failed to migrate key {key}: {e}")
                    failed += 1

            print(f"Progress: {i + len(batch)}/{len(key_index)} | "
                  f"Migrated: {migrated} | Failed: {failed}")

    return {'migrated': migrated, 'failed': failed}
```

## Strategy 2: Dual-Write Pattern

During migration, write to both old and new stores:

```python
# order_service/state_manager.py

class DualWriteStateManager:
    """
    Writes to both legacy and new state store during migration window.
    Reads from new store with fallback to legacy.
    """

    def __init__(self, legacy_store: str, new_store: str,
                 migration_active: bool = True):
        self.legacy_store = legacy_store
        self.new_store = new_store
        self.migration_active = migration_active

    def save(self, key: str, value: dict):
        with DaprClient() as client:
            # Always write to new store
            client.save_state(self.new_store, key, json.dumps(value))

            # During migration, also write to legacy store
            if self.migration_active:
                try:
                    client.save_state(self.legacy_store, key, json.dumps(value))
                except Exception as e:
                    # Log but don't fail - new store is authoritative
                    print(f"Legacy store write failed (non-critical): {e}")

    def get(self, key: str) -> dict:
        with DaprClient() as client:
            # Try new store first
            result = client.get_state(self.new_store, key)
            if result.data:
                return json.loads(result.data)

            # Fallback to legacy during migration
            if self.migration_active:
                legacy_result = client.get_state(self.legacy_store, key)
                if legacy_result.data:
                    data = json.loads(legacy_result.data)
                    # Lazy copy to new store
                    client.save_state(self.new_store, key, json.dumps(data))
                    return data

            return None
```

## Strategy 3: Event-Driven Lazy Migration

Migrate data on demand using pub/sub events:

```python
# When old service receives a read request it cannot serve from new location:
@app.route('/events/data-read-fallback', methods=['POST'])
def handle_migration_trigger():
    event_data = request.json.get('data', {})
    key = event_data['key']
    requesting_service = event_data['requesting_service']

    with DaprClient() as client:
        # Read from legacy store
        legacy_data = client.get_state("legacy-statestore", key)
        if legacy_data.data:
            # Push to new service's store
            client.invoke_method(
                app_id=requesting_service,
                method_name=f"migrate/{key}",
                http_verb="POST",
                data=legacy_data.data
            )

    return jsonify({'status': 'SUCCESS'})
```

## Validating the Migration

Run verification after bulk copy:

```bash
#!/bin/bash
# verify_migration.sh
SOURCE_STORE="legacy-statestore"
DEST_STORE="new-statestore"
SAMPLE_KEYS=("order:001" "order:002" "order:003")

for key in "${SAMPLE_KEYS[@]}"; do
  SOURCE=$(curl -s "http://localhost:3500/v1.0/state/$SOURCE_STORE/$key")
  DEST=$(curl -s "http://localhost:3500/v1.0/state/$DEST_STORE/$key")

  if [[ "$SOURCE" == "$DEST" ]]; then
    echo "PASS: $key"
  else
    echo "FAIL: $key (mismatch)"
    echo "  Source: $SOURCE"
    echo "  Dest:   $DEST"
  fi
done
```

## Summary

Dapr data migrations between services use bulk copy for one-time moves, dual-write for gradual cutover with rollback capability, and lazy migration for large datasets where copying everything upfront is impractical. All strategies benefit from Dapr's uniform state API that works the same regardless of the underlying backend.
