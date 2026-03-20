# How to Disable Host PID Access for Non-Admin Users in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Security, Host PID, Container Security, Hardening

Description: Learn how to prevent non-admin Portainer users from creating containers with host PID namespace access.

## What Is Host PID Access?

The `--pid=host` Docker flag makes a container share the host's PID (process ID) namespace. With this setting:

- The container can see all host processes.
- The container can potentially kill or interact with host processes.
- Combined with other capabilities, it can be used for privilege escalation.

This is a significant security risk in multi-tenant Portainer deployments.

## Disabling Host PID in Portainer

1. Go to **Environments** in Portainer.
2. Select your Docker environment.
3. Click **Edit**.
4. Under **Security**, find **Allow host PID** (or similar).
5. Toggle it **Off**.
6. Click **Update environment**.

## Why This Matters

Consider this scenario: a developer deploys the `nsenter` tool in a container with host PID access:

```bash
# What an attacker could do with hostPID=true

docker run -it --pid=host ubuntu nsenter -t 1 -m -u -i -n -p -- bash
# This gives root shell on the host!
```

Portainer's setting prevents non-admin users from enabling `hostPID` through the UI or API.

## Corresponding Kubernetes Setting

In Kubernetes environments, you can enforce this via Pod Security Standards or Portainer's cluster policies:

```yaml
# Kubernetes: Block hostPID via Pod Security Standards
# Label the namespace to use the "restricted" security policy
kubectl label namespace production \
  pod-security.kubernetes.io/enforce=restricted

# The restricted policy blocks hostPID=true at the API level
```

## Checking for Containers Using Host PID

```bash
# Find any running containers with hostPID enabled
docker inspect $(docker ps -q) | \
  jq '.[] | select(.HostConfig.PidMode == "host") | .Name'
```

## Security Implications by Feature

| Feature | Risk When Enabled for Non-Admins |
|---------|----------------------------------|
| Host PID | Process snooping, kill arbitrary processes |
| Host IPC | Shared memory attacks between containers |
| Host Network | Network snooping, port conflicts |
| Privileged mode | Full root on host |
| Bind mounts | Host filesystem exposure |

## Portainer Security Settings Summary

For maximum security, disable all these for non-admin users:

```text
Settings > Environments > [Env] > Security:
☒ Allow privileged mode
☒ Allow bind mounts
☒ Allow host PID
☒ Allow host IPC
☒ Allow host network
☒ Allow sysctl settings
```

## Conclusion

Disabling host PID access is a simple configuration change in Portainer that prevents a significant container escape vector. Along with disabling privileged mode, bind mounts, and host networking, it forms a comprehensive container security baseline for shared Portainer environments.
