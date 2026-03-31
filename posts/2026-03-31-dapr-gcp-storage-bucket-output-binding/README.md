# How to Use Dapr GCP Storage Bucket Output Binding

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, GCP, Cloud Storage, Binding, Object Storage

Description: Learn how to configure and use the Dapr GCP Storage Bucket output binding to upload, retrieve, and manage objects in Google Cloud Storage from microservices.

---

## What Is the Dapr GCP Storage Bucket Output Binding?

Google Cloud Storage (GCS) is a highly durable object storage service. The Dapr GCP Storage Bucket output binding allows your microservices to upload, retrieve, delete, and list objects in GCS using the Dapr binding API, without managing the GCP Storage client library.

## Setting Up a GCS Bucket

```bash
# Create a bucket
gsutil mb -p my-gcp-project -c STANDARD -l us-central1 gs://my-dapr-documents

# Set lifecycle policy to delete objects older than 365 days
cat > lifecycle.json <<EOF
{
  "lifecycle": {
    "rule": [{
      "action": {"type": "Delete"},
      "condition": {"age": 365}
    }]
  }
}
EOF

gsutil lifecycle set lifecycle.json gs://my-dapr-documents
```

## Configuring the Binding Component

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: document-store
  namespace: default
spec:
  type: bindings.gcp.bucket
  version: v1
  metadata:
    - name: bucket
      value: "my-dapr-documents"
    - name: type
      value: "service_account"
    - name: project_id
      value: "my-gcp-project"
    - name: private_key_id
      secretKeyRef:
        name: gcp-storage-secrets
        key: privateKeyId
    - name: private_key
      secretKeyRef:
        name: gcp-storage-secrets
        key: privateKey
    - name: client_email
      value: "dapr-storage@my-gcp-project.iam.gserviceaccount.com"
    - name: client_id
      value: "123456789012345678901"
    - name: auth_uri
      value: "https://accounts.google.com/o/oauth2/auth"
    - name: token_uri
      value: "https://oauth2.googleapis.com/token"
    - name: decodeBase64
      value: "false"
```

## Uploading Objects

```javascript
const { DaprClient } = require("@dapr/dapr");
const client = new DaprClient();

async function uploadReport(reportName, reportData, contentType = "application/json") {
  await client.binding.send(
    "document-store",
    "create",
    typeof reportData === "string" ? reportData : JSON.stringify(reportData),
    {
      name: reportName,
      contentType,
    }
  );
  console.log(`Uploaded: gs://my-dapr-documents/${reportName}`);
}

// Upload a JSON analytics report
await uploadReport(
  "analytics/2026/03/31/daily-metrics.json",
  { pageViews: 45230, sessions: 12800, bounceRate: 0.42 }
);

// Upload a CSV export
const csvContent = "date,revenue,orders\n2026-03-31,45230.50,312\n";
await uploadReport(
  "exports/sales-2026-03-31.csv",
  csvContent,
  "text/csv"
);
```

## Downloading Objects

```javascript
async function downloadObject(objectName) {
  const result = await client.binding.send(
    "document-store",
    "get",
    null,
    { name: objectName }
  );
  return result;
}

const reportContent = await downloadObject("analytics/2026/03/31/daily-metrics.json");
const metrics = JSON.parse(reportContent);
console.log("Page views:", metrics.pageViews);
```

## Deleting Objects

```javascript
async function deleteObject(objectName) {
  await client.binding.send(
    "document-store",
    "delete",
    null,
    { name: objectName }
  );
  console.log(`Deleted: ${objectName}`);
}

await deleteObject("temp/upload-staging-99.tmp");
```

## Listing Objects

```javascript
async function listObjects(prefix) {
  const result = await client.binding.send(
    "document-store",
    "list",
    null,
    { prefix }
  );
  return result;
}

const todayReports = await listObjects("analytics/2026/03/31/");
console.log(`Found ${todayReports.length} objects for today`);
todayReports.forEach((obj) => console.log("-", obj.name));
```

## Setting Object Metadata

GCS supports custom metadata on objects. Pass metadata via the request:

```javascript
await client.binding.send(
  "document-store",
  "create",
  reportContent,
  {
    name: "reports/q1-2026.json",
    contentType: "application/json",
    "x-goog-meta-environment": "production",
    "x-goog-meta-generated-by": "analytics-service",
    "x-goog-meta-report-version": "3.0",
  }
);
```

## Serving Public Objects via CDN

For public assets, set the bucket's IAM to allow `allUsers`:

```bash
gsutil iam ch allUsers:objectViewer gs://my-dapr-documents/public

# Objects uploaded to public/ prefix are accessible at:
# https://storage.googleapis.com/my-dapr-documents/public/logo.png
```

## Summary

The Dapr GCP Storage Bucket output binding covers all essential object storage operations: upload with custom content types and metadata, download, delete, and list with prefix filtering. By using the Dapr binding API instead of the GCS client library, your service stays decoupled from GCP specifics and benefits from Dapr's secret management and resiliency policies.
