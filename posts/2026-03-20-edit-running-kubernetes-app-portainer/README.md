# How to Edit a Running Kubernetes Application in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Kubernetes, Application Management, Rolling Update, DevOps

Description: Learn how to update a running Kubernetes application's configuration, image, and settings through the Portainer UI.

## Editing an Application in Portainer

1. Select your Kubernetes environment.
2. Go to **Applications** in the sidebar.
3. Find and click the application you want to edit.
4. Click **Edit** (pencil icon) or the **Edit this application** button.

You'll return to the same form used during deployment, pre-populated with the current configuration.

## What You Can Edit

- **Image tag**: Update to a new version.
- **Replica count**: Scale up or down.
- **Environment variables**: Add, remove, or change values.
- **Resource limits**: Adjust CPU and memory.
- **Volume mounts**: Add new persistent volumes.
- **Service configuration**: Change service type or port mappings.

## Updating the Container Image

The most common edit is deploying a new image version:

1. Open the application edit form.
2. Change the **Image** field from `myapp:1.0.0` to `myapp:2.0.0`.
3. Click **Update the application**.

Portainer triggers a Kubernetes rolling update automatically.

## Equivalent CLI Commands

```bash
# Update the container image in a deployment

kubectl set image deployment/my-app \
  my-app=registry.mycompany.com/my-app:2.0.0 \
  --namespace=production

# Watch the rolling update progress
kubectl rollout status deployment/my-app --namespace=production

# Scale replicas
kubectl scale deployment/my-app --replicas=5 --namespace=production

# Update an environment variable
kubectl set env deployment/my-app \
  APP_ENV=production \
  --namespace=production
```

## Editing via YAML (Advanced Mode)

For full control, switch to the YAML editor in Portainer:

1. Click the **Advanced** or **YAML** toggle.
2. Edit the raw Kubernetes manifest.
3. Click **Update**.

```bash
# Direct YAML editing via CLI
kubectl edit deployment/my-app --namespace=production
```

## Patching Without Full Re-Deploy

For targeted changes, use kubectl patch:

```bash
# Patch a specific field in the deployment spec
kubectl patch deployment my-app \
  --namespace=production \
  --type='json' \
  -p='[{"op": "replace", "path": "/spec/replicas", "value": 3}]'

# Update the image using a strategic merge patch
kubectl patch deployment my-app \
  --namespace=production \
  -p '{"spec":{"template":{"spec":{"containers":[{"name":"my-app","image":"my-app:2.1.0"}]}}}}'
```

## Rolling Back After an Edit

If the update causes issues:

```bash
# Roll back to the previous deployment version
kubectl rollout undo deployment/my-app --namespace=production

# Roll back to a specific revision
kubectl rollout undo deployment/my-app --to-revision=3 --namespace=production

# View rollout history
kubectl rollout history deployment/my-app --namespace=production
```

## Conclusion

Portainer makes editing running Kubernetes applications straightforward with its form-based interface. For common changes like image updates and scaling, the UI is fast and intuitive. For complex changes, the built-in YAML editor provides the full power of kubectl.
