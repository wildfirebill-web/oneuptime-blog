# How to Use Flexible Sync vs Partition-Based Sync in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Atlas, Device Sync, Flexible Sync, Mobile

Description: Compare Flexible Sync and Partition-Based Sync in MongoDB Atlas Device Sync to choose the right strategy for your mobile application data model.

---

## Overview

MongoDB Atlas Device Sync offers two synchronization strategies: **Partition-Based Sync** (the original model) and **Flexible Sync** (the recommended modern approach). Understanding the differences helps you choose the right strategy and migrate if needed.

## Partition-Based Sync

In Partition-Based Sync, all data is divided into partitions identified by a **partition key** - a field on every synced document that groups documents together. Each Realm instance on a device syncs exactly one partition.

```javascript
// All documents in a partition share the same partitionValue
const config = {
  schema: [TaskSchema],
  sync: {
    user,
    partitionValue: `user=${user.id}`,
  },
};
const realm = await Realm.open(config);
```

**Limitations:**
- A device can only sync one partition at a time.
- All documents must have a partition key field.
- Sharing data across partitions requires complex workarounds.

## Flexible Sync

Flexible Sync replaces the partition model with **query-based subscriptions**. Devices subscribe to specific queries and sync only the documents matching those queries, regardless of a partition key.

```javascript
const config = {
  schema: [TaskSchema, ProjectSchema],
  sync: {
    user,
    flexible: true,
    initialSubscriptions: {
      update(subs, realm) {
        subs.add(
          realm.objects('Task').filtered('ownerId == $0', user.id),
          { name: 'myTasks' }
        );
        subs.add(
          realm.objects('Project').filtered('teamId == $0', user.customData.teamId),
          { name: 'teamProjects' }
        );
      },
    },
  },
};
```

## Side-by-Side Comparison

```text
Feature                   | Partition-Based    | Flexible Sync
--------------------------|--------------------|-----------------
Data grouping             | Partition key      | Query subscriptions
Multiple collections      | One per realm      | Multiple collections
Cross-partition queries   | Not supported      | Supported
Schema requirement        | partitionKey field | No special field needed
Subscription management   | Implicit           | Explicit (add/remove)
Recommended for new apps  | No (legacy)        | Yes
```

## Migrating from Partition-Based to Flexible Sync

Atlas provides a migration path in the App Services UI:

1. Go to **Device Sync > Migrate to Flexible Sync**.
2. Atlas automatically converts your partition key permissions to equivalent Flexible Sync rules.
3. Update your client SDK code to use `flexible: true` and add subscriptions.

Client SDK migration example:

```javascript
// Before (Partition-Based)
sync: { user, partitionValue: `user=${user.id}` }

// After (Flexible Sync)
sync: {
  user,
  flexible: true,
  initialSubscriptions: {
    update(subs, realm) {
      subs.add(realm.objects('Task').filtered('ownerId == $0', user.id));
    },
  },
}
```

## Managing Subscriptions Dynamically

Flexible Sync allows adding and removing subscriptions at runtime:

```javascript
const realm = useRealm();

// Add a new subscription
await realm.subscriptions.update((mutableSubs) => {
  mutableSubs.add(
    realm.objects('Task').filtered('projectId == $0', selectedProjectId),
    { name: 'projectTasks' }
  );
});

// Remove a subscription
await realm.subscriptions.update((mutableSubs) => {
  mutableSubs.removeByName('projectTasks');
});
```

## Best Practices

- Use Flexible Sync for all new applications - it offers far more flexibility and is the direction Atlas is investing in.
- Keep subscriptions focused and filtered - syncing large, unfiltered datasets increases bandwidth usage and device storage.
- Name your subscriptions so you can update or remove them by name without iterating the full subscription list.

## Summary

Flexible Sync supersedes Partition-Based Sync with query-based subscriptions that give you fine-grained control over what data syncs to each device. For new apps, always use Flexible Sync. For existing Partition-Based Sync apps, use the Atlas migration tool to convert to Flexible Sync.
