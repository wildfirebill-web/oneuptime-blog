# How to Use Dapr SMTP Binding to Send Emails

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, SMTP, Email, Bindings, Microservices

Description: Learn how to use the Dapr SMTP output binding to send emails from your microservices without managing SMTP client libraries or connection logic.

---

## What Is the Dapr SMTP Binding

The Dapr SMTP output binding allows your application to send emails through an SMTP server via the Dapr sidecar. Instead of embedding an SMTP client library (nodemailer, smtplib, etc.) in each service, you configure the SMTP connection in a Dapr component and invoke it through the standard binding API.

## Prerequisites

- Dapr CLI installed and initialized
- Access to an SMTP server (Gmail, SendMail, Mailhog for local testing, etc.)
- Basic familiarity with Dapr components

## Define the SMTP Binding Component

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: smtp-email
  namespace: default
spec:
  type: bindings.smtp
  version: v1
  metadata:
  - name: host
    value: "smtp.gmail.com"
  - name: port
    value: "587"
  - name: user
    value: "myapp@gmail.com"
  - name: password
    secretKeyRef:
      name: smtp-secret
      key: password
  - name: skipTLSVerify
    value: "false"
  - name: emailFrom
    value: "myapp@gmail.com"
```

For local testing with Mailhog:

```yaml
  - name: host
    value: "localhost"
  - name: port
    value: "1025"
  - name: skipTLSVerify
    value: "true"
```

## Send an Email via the Dapr API

```bash
curl -X POST http://localhost:3500/v1.0/bindings/smtp-email \
  -H "Content-Type: application/json" \
  -d '{
    "data": "Hello, this is a test email from Dapr!",
    "metadata": {
      "emailTo": "recipient@example.com",
      "subject": "Test Email from Dapr SMTP Binding"
    },
    "operation": "create"
  }'
```

## Send HTML Email

```bash
curl -X POST http://localhost:3500/v1.0/bindings/smtp-email \
  -H "Content-Type: application/json" \
  -d '{
    "data": "<h1>Order Confirmed</h1><p>Your order #12345 has been confirmed.</p>",
    "metadata": {
      "emailTo": "customer@example.com",
      "emailFrom": "orders@myapp.com",
      "subject": "Order Confirmation - #12345",
      "contentType": "text/html"
    },
    "operation": "create"
  }'
```

## Use in a Node.js Application

```javascript
const { DaprClient } = require('@dapr/dapr');

const client = new DaprClient();
const BINDING = 'smtp-email';

async function sendWelcomeEmail(user) {
  const htmlBody = `
    <h2>Welcome to MyApp, ${user.name}!</h2>
    <p>Your account has been created successfully.</p>
    <p>Get started by <a href="https://myapp.com/login">logging in here</a>.</p>
  `;

  await client.binding.send(BINDING, 'create', htmlBody, {
    emailTo: user.email,
    subject: `Welcome to MyApp, ${user.name}!`,
    contentType: 'text/html',
  });

  console.log('Welcome email sent to:', user.email);
}

async function sendOrderConfirmation(order) {
  const htmlBody = `
    <h2>Order Confirmed</h2>
    <p>Order ID: ${order.id}</p>
    <p>Total: $${order.total.toFixed(2)}</p>
    <p>Estimated delivery: ${order.deliveryDate}</p>
  `;

  await client.binding.send(BINDING, 'create', htmlBody, {
    emailTo: order.customerEmail,
    subject: `Order Confirmation - ${order.id}`,
    contentType: 'text/html',
  });
}

async function sendPasswordReset(email, resetToken) {
  const body = `
    You requested a password reset for your account.

    Click the link below to reset your password:
    https://myapp.com/reset-password?token=${resetToken}

    This link expires in 1 hour.

    If you did not request this, please ignore this email.
  `;

  await client.binding.send(BINDING, 'create', body, {
    emailTo: email,
    subject: 'Password Reset Request',
    contentType: 'text/plain',
  });
}
```

## Use in a Python Application

```python
from dapr.clients import DaprClient

BINDING = 'smtp-email'

def send_email(to: str, subject: str, body: str, html: bool = False):
    content_type = 'text/html' if html else 'text/plain'
    with DaprClient() as client:
        client.invoke_binding(
            binding_name=BINDING,
            operation='create',
            data=body,
            binding_metadata={
                'emailTo': to,
                'subject': subject,
                'contentType': content_type,
            }
        )
    print(f"Email sent to {to}: {subject}")

# Send a plain text alert
send_email(
    to='ops@company.com',
    subject='Alert: High CPU Usage Detected',
    body='CPU usage has exceeded 90% on server prod-01. Please investigate.'
)

# Send an HTML report
send_email(
    to='manager@company.com',
    subject='Weekly Report',
    body='<h1>Weekly Summary</h1><p>Sales were up 12% this week.</p>',
    html=True
)
```

## Send to Multiple Recipients

Pass multiple email addresses separated by semicolons:

```javascript
await client.binding.send(BINDING, 'create', emailBody, {
  emailTo: 'alice@example.com;bob@example.com',
  emailCC: 'manager@example.com',
  subject: 'Team Update',
  contentType: 'text/html',
});
```

## Summary

The Dapr SMTP binding offers a clean way to send emails from microservices by centralizing SMTP configuration in a component YAML and using the standard Dapr binding API. It supports plain text and HTML emails, multiple recipients with CC, and integrates with Dapr secret stores for secure credential management. This makes email sending portable across environments without changing application code.
