# How to Use Dapr Preview Features Safely

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Preview, Feature Flag, Configuration, Risk

Description: Enable and use Dapr preview features safely by understanding their stability guarantees, enabling them explicitly, and planning for potential breaking changes.

---

Dapr marks certain features as "preview" to indicate they are functional but may change in future releases. Using preview features carefully lets you benefit from new capabilities while managing the risk of breaking changes.

## What "Preview" Means in Dapr

Preview features in Dapr are:
- Fully implemented and tested
- Not yet considered stable API
- Subject to change or removal in future versions
- Disabled by default to prevent accidental dependency

Preview features are distinct from alpha features (experimental, may break) and stable features (fully committed API).

## Listing Available Preview Features

Check the Dapr release notes for the current version:

```bash
dapr --version
# Check: https://docs.dapr.io/operations/configuration/preview-features/
```

Common preview features include:
- Scheduler service
- App health checks (now stable in 1.13+)
- Outbox pattern
- Query API for state stores

## Enabling Preview Features in Dapr Configuration

Preview features are enabled in the Dapr Configuration resource:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Configuration
metadata:
  name: myconfig
  namespace: default
spec:
  features:
    - name: SchedulerReminders
      enabled: true
    - name: ActorTypeMetadata
      enabled: true
```

Apply it:

```bash
kubectl apply -f dapr-config.yaml
```

Reference the config from your deployment:

```yaml
annotations:
  dapr.io/enabled: "true"
  dapr.io/app-id: "myapp"
  dapr.io/config: "myconfig"
```

## Enabling Features in Self-Hosted Mode

For local development with the Dapr CLI:

```yaml
# ~/.dapr/config.yaml
apiVersion: dapr.io/v1alpha1
kind: Configuration
metadata:
  name: daprConfig
spec:
  features:
    - name: SchedulerReminders
      enabled: true
```

## Risk Management for Preview Features

Follow these practices to use preview features safely:

1. **Never enable preview features in production without testing in staging first**
2. **Pin your Dapr version** when using preview features - do not auto-upgrade:

```yaml
# In your Dapr runtime configuration
# Pin to a specific version
dapr init --runtime-version 1.13.2
```

3. **Monitor Dapr release notes** for changes to preview features you depend on:

```bash
# Subscribe to GitHub releases
# https://github.com/dapr/dapr/releases
```

4. **Write feature flags in your application** to disable the preview feature path:

```python
ENABLE_SCHEDULER_REMINDERS = os.getenv("ENABLE_SCHEDULER_REMINDERS", "false").lower() == "true"

if ENABLE_SCHEDULER_REMINDERS:
    # Use new scheduler-based reminder
    client.create_reminder_with_scheduler(...)
else:
    # Fall back to actor-based reminder
    client.register_actor_reminder(...)
```

## Graduating from Preview to Stable

When a preview feature becomes stable, the `enabled: true` flag may no longer be required. Check the Dapr changelog and remove the feature flag from your configuration after upgrading:

```bash
# After upgrading Dapr, check if feature is now stable
# If stable, remove from Configuration spec
kubectl edit configuration myconfig -n default
```

## Summary

Dapr preview features are explicitly opt-in through the Configuration resource to prevent accidental dependency. Use them in production only after staging validation, pin your Dapr version to avoid unexpected changes, and maintain application-level feature flags that let you roll back without redeployment. Monitor Dapr release notes closely when running on preview features to catch breaking changes before upgrading.
