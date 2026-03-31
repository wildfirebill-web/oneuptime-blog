# How to Use Dapr with GCP Identity-Aware Proxy

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, GCP, Identity-Aware Proxy, Authentication, Security

Description: Configure Dapr to authenticate with GCP services using Workload Identity and service accounts, and integrate with GCP Identity-Aware Proxy for secure access control.

---

## GCP Authentication in Dapr

Dapr GCP components authenticate using Google Cloud credentials. On GKE, the recommended approach is Workload Identity, which links Kubernetes ServiceAccounts to GCP service accounts without embedding credentials.

## Setting Up Workload Identity on GKE

Enable Workload Identity on your GKE cluster:

```bash
# Enable Workload Identity on cluster
gcloud container clusters update my-cluster \
  --workload-pool=my-project.svc.id.goog \
  --region=us-central1

# Enable on node pool
gcloud container node-pools update default-pool \
  --cluster=my-cluster \
  --region=us-central1 \
  --workload-metadata=GKE_METADATA
```

## Creating GCP Service Account

```bash
# Create GCP service account
gcloud iam service-accounts create dapr-workload \
  --display-name="Dapr Workload SA"

# Grant Pub/Sub permissions
gcloud projects add-iam-policy-binding my-project \
  --member="serviceAccount:dapr-workload@my-project.iam.gserviceaccount.com" \
  --role="roles/pubsub.publisher"

# Allow Kubernetes SA to impersonate GCP SA
gcloud iam service-accounts add-iam-policy-binding \
  dapr-workload@my-project.iam.gserviceaccount.com \
  --role roles/iam.workloadIdentityUser \
  --member "serviceAccount:my-project.svc.id.goog[production/my-app]"
```

## Annotating Kubernetes ServiceAccount

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-app
  namespace: production
  annotations:
    iam.gke.io/gcp-service-account: dapr-workload@my-project.iam.gserviceaccount.com
```

## Configuring Dapr GCP Component

With Workload Identity, no credentials are needed in the component:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: gcp-pubsub
spec:
  type: pubsub.gcp.pubsub
  version: v1
  metadata:
    - name: projectId
      value: "my-project"
    # No authProviderX509CertUrl or private key needed
    # Workload Identity provides credentials automatically
```

## Using Identity-Aware Proxy for Service Authentication

If your Dapr app calls a service protected by GCP Identity-Aware Proxy (IAP), obtain an ID token and pass it in the request:

```python
import google.auth
import google.auth.transport.requests
from google.oauth2 import id_token
import requests

def call_iap_protected_service(url: str, client_id: str) -> str:
    auth_req = google.auth.transport.requests.Request()
    token = id_token.fetch_id_token(auth_req, client_id)

    headers = {
        "Authorization": f"Bearer {token}",
        "Content-Type": "application/json"
    }

    response = requests.get(url, headers=headers)
    return response.text

# Call from within a Dapr-enabled service
result = call_iap_protected_service(
    "https://my-service.example.com/api/data",
    "123456789-abc.apps.googleusercontent.com"
)
```

## Verifying Workload Identity

```bash
# Exec into the pod and verify the identity
kubectl exec -it <pod-name> -- \
  curl -s -H "Metadata-Flavor: Google" \
  http://metadata.google.internal/computeMetadata/v1/instance/service-accounts/default/email
```

The output should show the GCP service account email, confirming Workload Identity is active.

## Summary

Configure Dapr with GCP authentication using Workload Identity on GKE for seamless, credential-free access to GCP services. Link the Kubernetes ServiceAccount to a GCP service account via IAM binding and annotation. For services protected by GCP Identity-Aware Proxy, fetch ID tokens programmatically using the GCP auth library and include them in request headers.
