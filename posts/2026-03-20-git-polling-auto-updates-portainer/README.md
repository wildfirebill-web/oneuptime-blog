# How to Configure Git Polling for Auto-Updates in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, GitOps, Git Polling, Auto-Updates, Automation

Description: Learn how to configure Portainer to automatically poll a Git repository and redeploy stacks when changes are detected.

## What Is Git Polling?

Git polling is a pull-based update mechanism where Portainer periodically checks a Git repository for new commits. When it detects a change (new commit on the configured branch), it automatically redeployes the stack.

This is useful when:
- The Git repository host cannot send webhooks to Portainer.
- Portainer is behind a firewall with no inbound webhook access.
- You want simple setup without configuring webhook secrets.

## Enabling Polling When Creating a Stack

1. Create a stack from a Git repository (see deploy from Git guide).
2. Under **GitOps updates**, select **Polling**.
3. Set the **Polling interval** (minimum: 1 minute, recommended: 5-15 minutes).
4. Click **Deploy the stack**.

## Enabling Polling on an Existing Stack

1. Navigate to **Stacks** and click the stack name.
2. Scroll to **GitOps updates** section.
3. Toggle **Automatic updates** to On.
4. Select **Polling** and set the interval.
5. Click **Save changes**.

## Polling Interval Recommendations

| Use Case | Recommended Interval |
|----------|---------------------|
| Development/staging | 1-2 minutes |
| Production (non-critical) | 5-10 minutes |
| Production (stable) | 15-30 minutes |

Shorter intervals mean faster auto-deploys but more API calls to your Git host.

## How Portainer Detects Changes

Portainer compares the current deployed commit hash with the latest commit on the configured branch:

```bash
# Portainer internally does something equivalent to:
CURRENT_SHA=$(git rev-parse HEAD)
LATEST_SHA=$(git ls-remote origin refs/heads/main | cut -f1)

if [ "$CURRENT_SHA" != "$LATEST_SHA" ]; then
  # Redeploy the stack
fi
```

## Viewing Poll Status in Portainer

After enabling polling, the stack detail page shows:
- **GitOps status**: Last check time.
- **Current commit**: The deployed Git SHA.
- **Auto-update**: Enabled/disabled and interval.

## Forcing an Immediate Update

To redeploy without waiting for the next poll:

1. Go to the stack in Portainer.
2. Click **Pull and redeploy**.

Or via API:

```bash
# Force an immediate git pull and redeploy
curl -X POST "${PORTAINER_URL}/api/stacks/${STACK_ID}/git/redeploy?endpointId=${ENDPOINT_ID}" \
  -H "Authorization: Bearer ${API_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{"pullImage": true, "prune": false}'
```

## Best Practices

- **Use specific branches** (`main`, `production`) rather than tags for polling, since tag polling requires new tags for each deployment.
- **Combine with image tags**: Have your CI pipeline update the image tag in the Compose file and commit to Git — Portainer will detect the change and redeploy.
- **Monitor poll logs**: Check stack events in Portainer to confirm polling is working.

## Polling vs. Webhooks

| Feature | Polling | Webhooks |
|---------|---------|----------|
| Setup | Simple | Requires Git webhook config |
| Latency | Up to poll interval | Near-instant |
| Works behind NAT | Yes | No (needs inbound access) |
| API calls to Git | Regular | Only on push |

## Conclusion

Git polling in Portainer provides a zero-configuration auto-update mechanism that works in any network environment. Set a polling interval appropriate to your deployment cadence, and combine it with commit-based image tags for a clean GitOps workflow.
