# How to Configure Anonymous vs Authenticated Docker Hub Access in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker Hub, Authentication, Rate Limiting, Container Management

Description: Learn the differences between anonymous and authenticated Docker Hub access in Portainer and how to configure each.

## Docker Hub Rate Limits

Understanding Docker Hub's rate limits helps you decide when authentication is necessary:

| Access Type | Pull Limit |
|-------------|-----------|
| Anonymous | 100 pulls per 6 hours per IP |
| Free authenticated account | 200 pulls per 6 hours |
| Docker Pro | Unlimited |
| Docker Team/Business | Unlimited |

In shared environments like CI servers or Swarm clusters where multiple nodes share an IP, anonymous limits are exhausted quickly.

## Anonymous Access (Default)

By default, Portainer uses anonymous access to Docker Hub. No configuration is needed. Images are pulled using the Docker Hub public API without credentials.

This is fine for:
- Personal/home lab use
- Environments pulling infrequently
- Only needing public images

## Authenticated Access

For production environments, add your Docker Hub credentials to avoid rate limiting and enable private image access.

### Adding Credentials in Portainer

1. Go to **Settings > Registries**.
2. Click **Add registry** and select **DockerHub**.
3. Enter your Docker Hub username and password (or access token).
4. Click **Add registry**.
5. Assign it to your environment under **Environments > Edit > Registries**.

### Creating a Docker Hub Access Token

```bash
# In the Docker Hub UI:
# Account Settings > Security > New Access Token
# Name it "portainer-<environment>" for easy identification

# Test the token from CLI
docker login docker.io \
  -u your-username \
  -p dckr_pat_XXXXXXXXXXXX  # Access token format
```

## Checking Your Current Rate Limit Status

```bash
# Check remaining rate limit (works for both anonymous and authenticated)
TOKEN=$(curl -s \
  "https://auth.docker.io/token?service=registry.docker.io&scope=repository:ratelimitpreview/test:pull" \
  | jq -r .token)

curl -I --head \
  -H "Authorization: Bearer $TOKEN" \
  https://registry-1.docker.io/v2/ratelimitpreview/test/manifests/latest \
  2>&1 | grep -i "ratelimit"
```

## When to Use Each Approach

**Use anonymous access when:**
- Running a small personal Portainer instance
- Pull frequency is low (< 10 pulls/hour across all nodes)

**Use authenticated access when:**
- Running production Swarm or Kubernetes clusters
- Multiple nodes need to pull images simultaneously
- You need access to private Docker Hub repositories
- You want consistent and predictable pull availability

## Switching from Anonymous to Authenticated

After adding credentials to Portainer, existing deployments continue working. New deployments and image updates automatically use the authenticated credentials.

## Conclusion

Authenticated Docker Hub access is a simple configuration change in Portainer that prevents rate limit errors at scale. Use access tokens instead of passwords for better security, and rotate them periodically.
