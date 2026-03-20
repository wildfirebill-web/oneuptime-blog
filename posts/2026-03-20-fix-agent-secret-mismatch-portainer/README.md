# How to Fix Agent Secret Mismatch Between Server and Agent

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Troubleshooting, Agent Secret, Authentication, Docker, Security

Description: Learn how to diagnose and fix agent secret mismatches between the Portainer server and agent, which cause silent authentication failures when connecting environments.

---

Portainer uses a shared secret for mutual authentication between the server and agent. When the secrets do not match, the TCP connection succeeds but authentication fails, leaving the environment stuck in "Offline" state with no clear error message.

## How Agent Secrets Work

Both sides must be configured with the same secret:

```bash
# Portainer Server side
docker run ... portainer/portainer-ce:latest --agent-secret mysecrettoken

# Portainer Agent side
docker run ... -e AGENT_SECRET=mysecrettoken portainer/agent:latest
```

If the server uses no `--agent-secret` flag, any agent can connect. If the server has a secret but the agent does not (or has a different one), connections will fail silently.

## Step 1: Verify Current Server Secret

```bash
# Check how Portainer server was started
docker inspect portainer --format '{{.Args}}'

# Look for --agent-secret in the output
# If absent, no secret is configured (any agent connects)
```

## Step 2: Verify Current Agent Secret

```bash
# Check agent environment variables
docker inspect portainer_agent \
  --format '{{range .Config.Env}}{{println .}}{{end}}' | grep AGENT_SECRET

# If empty, no secret is set on the agent
```

## Step 3: Align the Secrets

Decide on a new shared secret and update both sides:

```bash
# Generate a strong secret
SECRET=$(openssl rand -hex 32)
echo "Your new secret: $SECRET"

# Stop and remove both containers
docker stop portainer portainer_agent
docker rm portainer portainer_agent

# Restart Portainer server with the secret
docker run -d \
  --name portainer \
  --restart=always \
  -p 9000:9000 -p 9443:9443 -p 8000:8000 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest \
  --agent-secret "$SECRET"

# Restart agent with the same secret
docker run -d \
  --name portainer_agent \
  --restart=always \
  -p 9001:9001 \
  -e AGENT_SECRET="$SECRET" \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v /var/lib/docker/volumes:/var/lib/docker/volumes \
  portainer/agent:latest
```

## Step 4: Verify Connection

```bash
# After restart, check agent logs for successful auth
docker logs portainer_agent --tail 20

# And check server logs
docker logs portainer --tail 20 | grep -i "agent\|secret\|auth"
```

In Portainer UI, the environment should now show as "Online" after 30–60 seconds.
