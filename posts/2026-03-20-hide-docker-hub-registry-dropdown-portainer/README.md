# How to Hide Docker Hub from the Registry Dropdown in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker Hub, Registry Management, Security, DevOps

Description: Learn how to hide Docker Hub from Portainer's registry dropdown to enforce the use of private registries in your organization.

## Why Hide Docker Hub?

Organizations with strict security policies may want to prevent developers from pulling images directly from Docker Hub and instead enforce the use of a private, audited registry. Portainer allows administrators to disable Docker Hub visibility in the registry selection dropdown.

## Disabling Docker Hub in Portainer

This is a per-environment setting in Portainer:

1. Log in to Portainer as an administrator.
2. Go to **Settings > Registries**.
3. Find the **DockerHub** entry in the registry list.
4. Toggle it **Off** or click the disable icon.

Alternatively, in **Environments**, select an environment and configure which registries are available.

## Restricting Public Repositories via Endpoint Settings

For stricter control, you can disable the use of public images entirely:

1. Go to **Environments** and select your environment.
2. Click **Edit**.
3. Under **Security settings**, toggle **Allow users to use public images** to **Off**.

## Portainer API Approach

You can also manage registry visibility via the Portainer API:

```bash
# List all registries to find the Docker Hub registry ID
curl -H "Authorization: Bearer $TOKEN" \
  http://localhost:9000/api/registries

# Update the registry to disable it (set access to none)
curl -X PUT \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  http://localhost:9000/api/registries/1 \
  -d '{"restricted": true}'
```

## Enforcing a Specific Registry

To guide users toward a specific registry, you can:

1. Add your private registry (e.g., Harbor or ECR) to Portainer.
2. Disable Docker Hub.
3. Set the private registry as the default.

This ensures all image pulls go through your approved registry with vulnerability scanning enabled.

## Communicating the Policy

When Docker Hub is hidden, users who attempt to reference a Docker Hub image in a stack deployment will see a pull failure if no credentials are configured. Provide documentation explaining:

```
All images must be pulled from registry.mycompany.com
Tag your images as: registry.mycompany.com/myteam/myimage:tag
```

## Conclusion

Hiding Docker Hub from Portainer's dropdown is a simple but effective control for enforcing private registry usage policies. Combine it with vulnerability scanning in your approved registry for a complete supply chain security posture.
