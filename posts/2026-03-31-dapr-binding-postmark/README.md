# How to Use Dapr Postmark Binding for Transactional Email

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Binding, Postmark, Email, Transactional

Description: Learn how to configure the Dapr Postmark binding to send transactional emails such as receipts, notifications, and alerts from microservices.

---

## Why Use Postmark for Transactional Email

Postmark specializes in transactional email with high deliverability, detailed send statistics, and fast delivery. The Dapr Postmark binding gives any service access to Postmark's API without managing the HTTP client or authentication.

## Configure the Postmark Binding Component

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: postmark-email
spec:
  type: bindings.postmark
  version: v1
  metadata:
  - name: accountToken
    secretKeyRef:
      name: postmark-secret
      key: accountToken
  - name: serverToken
    secretKeyRef:
      name: postmark-secret
      key: serverToken
```

## Create the Kubernetes Secret

```bash
kubectl create secret generic postmark-secret \
  --from-literal=accountToken=your-account-token \
  --from-literal=serverToken=your-server-token
```

## Send a Simple Email

```bash
curl -X POST http://localhost:3500/v1.0/bindings/postmark-email \
  -H "Content-Type: application/json" \
  -d '{
    "operation": "create",
    "metadata": {
      "subject": "Your order has shipped",
      "toEmail": "customer@example.com",
      "fromEmail": "orders@mycompany.com"
    },
    "data": "<h1>Your order is on its way!</h1><p>Track your package at example.com/track/1001</p>"
  }'
```

## Send Using a Template

Postmark supports server-side templates. Pass the template ID and model:

```bash
curl -X POST http://localhost:3500/v1.0/bindings/postmark-email \
  -H "Content-Type: application/json" \
  -d '{
    "operation": "create",
    "metadata": {
      "templateId": "12345678",
      "toEmail": "user@example.com",
      "fromEmail": "noreply@mycompany.com"
    },
    "data": {
      "customer_name": "Alice",
      "order_id": "ORD-1001",
      "total_amount": "$49.99",
      "support_email": "help@mycompany.com"
    }
  }'
```

## Application Code Integration

```typescript
interface EmailOptions {
  to: string;
  subject: string;
  htmlBody: string;
  from?: string;
}

async function sendEmail(options: EmailOptions): Promise<void> {
  const response = await fetch(
    "http://localhost:3500/v1.0/bindings/postmark-email",
    {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({
        operation: "create",
        metadata: {
          subject: options.subject,
          toEmail: options.to,
          fromEmail: options.from ?? "noreply@mycompany.com",
        },
        data: options.htmlBody,
      }),
    }
  );

  if (!response.ok) {
    throw new Error(`Failed to send email: ${response.statusText}`);
  }
}

// Usage
await sendEmail({
  to: "alice@example.com",
  subject: "Welcome to Our Platform",
  htmlBody: "<h1>Welcome, Alice!</h1><p>Your account is ready.</p>",
});
```

## Send to Multiple Recipients

Postmark supports multiple `To` addresses separated by commas:

```bash
curl -X POST http://localhost:3500/v1.0/bindings/postmark-email \
  -H "Content-Type: application/json" \
  -d '{
    "operation": "create",
    "metadata": {
      "subject": "Weekly Report",
      "toEmail": "manager@example.com,team@example.com",
      "fromEmail": "reports@mycompany.com"
    },
    "data": "<p>See attached weekly summary.</p>"
  }'
```

## Summary

The Dapr Postmark binding enables any microservice to send transactional emails through Postmark's API without writing HTTP client code. Configure your account and server tokens via Kubernetes secrets, then invoke the binding with recipient, subject, and HTML body or template ID. This keeps email logic consistent across all services in your platform.
