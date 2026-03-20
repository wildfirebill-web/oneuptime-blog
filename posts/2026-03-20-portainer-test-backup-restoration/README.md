# How to Test Portainer Backup Restoration

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Backup, Testing, Disaster Recovery, Verification, Docker

Description: Learn how to test that Portainer backup restoration works correctly before a real disaster occurs, using a test container alongside your production instance.

---

A backup you haven't tested is not a backup. This guide shows how to verify Portainer backup restoration without disturbing your production environment, using a separate test container on a different port.

## Why Test Restores?

- Confirms the backup file is complete and not corrupted
- Verifies the backup procedure captures all necessary data
- Practices the restore procedure so you're confident in an emergency
- Checks that the restored Portainer version is compatible with your backup

## Step 1: Restore to a Test Container

Run a parallel Portainer instance on a different port to test the restore:

```bash
# Create a test volume

docker volume create portainer_test_data

# Restore the backup into the test volume
docker run --rm \
  -v portainer_test_data:/data \
  -v /backup:/backup \
  alpine \
  tar xzpf /backup/portainer-20260320.tar.gz -C /

# Start a test Portainer instance on port 9100
docker run -d \
  --name portainer-test \
  -p 9100:9000 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_test_data:/data \
  portainer/portainer-ce:latest
```

## Step 2: Verify the Restore Checklist

Open `http://localhost:9100` and verify:

```bash
# Automated verification via API
TOKEN=$(curl -s -X POST http://localhost:9100/api/auth \
  -H "Content-Type: application/json" \
  -d '{"Username":"admin","Password":"yourpassword"}' | jq -r .jwt)

# Check environments were restored
ENVS=$(curl -s -H "Authorization: Bearer $TOKEN" http://localhost:9100/api/endpoints | jq length)
echo "Environments restored: $ENVS"

# Check stacks were restored
STACKS=$(curl -s -H "Authorization: Bearer $TOKEN" http://localhost:9100/api/stacks | jq length)
echo "Stacks restored: $STACKS"

# Check users were restored
USERS=$(curl -s -H "Authorization: Bearer $TOKEN" http://localhost:9100/api/users | jq length)
echo "Users restored: $USERS"
```

## Step 3: Document What Was Lost

If any data is missing from the restore:

```bash
# Compare production vs restored data
PROD_STACKS=$(curl -s -H "Authorization: Bearer $PROD_TOKEN" http://localhost:9000/api/stacks | jq '.[].Name' | sort)
TEST_STACKS=$(curl -s -H "Authorization: Bearer $TEST_TOKEN" http://localhost:9100/api/stacks | jq '.[].Name' | sort)

diff <(echo "$PROD_STACKS") <(echo "$TEST_STACKS")
# Any differences indicate gaps in the backup
```

## Step 4: Clean Up the Test Environment

```bash
# Remove the test container and volume
docker stop portainer-test && docker rm portainer-test
docker volume rm portainer_test_data
```

## Scheduling Regular Restore Tests

Add a monthly restore test to your calendar or automate it:

```bash
# Monthly cron job: first Sunday at 3 AM
0 3 * * 0 [ $(date +\%d) -le 07 ] && /usr/local/bin/portainer-test-restore.sh
```
