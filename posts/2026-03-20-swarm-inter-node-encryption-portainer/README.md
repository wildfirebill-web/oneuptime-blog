# How to Set Up Swarm Inter-Node Encryption in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker Swarm, Encryption, Security, Overlay Networks

Description: Enable Docker Swarm inter-node traffic encryption for overlay networks managed through Portainer to protect container-to-container communication across nodes.

---

Docker Swarm overlay networks carry container-to-container traffic across cluster nodes. By default, this traffic is unencrypted. Enabling encryption on overlay networks protects sensitive service communication without requiring application-level TLS.

## How Swarm Network Encryption Works

Swarm uses IPsec (AES-128-GCM) to encrypt traffic on overlay networks. The encryption keys are distributed via the Swarm Raft log and rotated automatically. Enabling encryption requires no changes to your applications.

## Step 1: Create an Encrypted Overlay Network

In Portainer's terminal or via stack YAML:

```bash
# Create an encrypted overlay network via CLI
docker network create \
  --driver overlay \
  --opt encrypted=true \
  secure-backend
```

Or declare it in a stack:

```yaml
version: "3.8"
services:
  api:
    image: my-api:latest
    networks:
      - secure-backend

  database:
    image: postgres:16
    networks:
      - secure-backend

networks:
  secure-backend:
    driver: overlay
    driver_opts:
      # Enable IPsec encryption for this overlay network
      encrypted: "true"
```

## Step 2: Deploy the Stack via Portainer

Paste the stack YAML in **Stacks > Add Stack**. Portainer creates the encrypted network and attaches the services to it.

## Step 3: Verify Encryption is Active

```bash
# Check network options
docker network inspect secure-backend | grep -i encrypted
# Expected: "com.docker.network.driver.overlay.vxlanid_list": "...", "encrypted": "true"
```

## Step 4: Performance Considerations

Encryption adds CPU overhead for IPsec processing — typically 5-15% on modern hardware. For workloads where performance is critical and traffic is already protected by application-layer TLS (HTTPS), the additional overlay encryption may be redundant.

Recommendation:
- **Enable** for networks carrying unencrypted database traffic or internal API calls
- **Optional** for networks where all traffic is already TLS-encrypted

## Step 5: Autolock for Cluster Key Protection

Enable Swarm Autolock to protect the cluster encryption keys at rest. Without autolock, Swarm keys are stored on disk and readable by anyone with file system access:

```bash
# Enable autolock when initializing a new swarm
docker swarm init --autolock

# Enable autolock on an existing swarm
docker swarm update --autolock=true

# Save the unlock key securely
docker swarm unlock-key
```

After enabling autolock, managers require the unlock key after restart.

## Step 6: Rotate Network Keys

Periodically rotate network encryption keys:

```bash
docker network inspect secure-backend --format '{{.ID}}' | xargs docker network update --ingress=false
```

## Summary

Swarm overlay network encryption is a transparent, application-agnostic security control. By setting `encrypted: "true"` in your network definition, all container-to-container traffic on that network is automatically encrypted with AES-128-GCM without requiring application changes. Portainer's stack interface makes enabling encryption as simple as adding a network option to your YAML.
