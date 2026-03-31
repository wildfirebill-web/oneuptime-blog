# How to Use Dapr Twilio SMS Binding for Text Messaging

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Twilio, SMS, Binding, Notification

Description: Learn how to use the Dapr Twilio SMS output binding to send text messages from your microservices without embedding the Twilio SDK directly.

---

## What Is the Dapr Twilio SMS Binding

The Dapr Twilio SMS binding is an output binding that lets your application send SMS messages via Twilio through the Dapr sidecar. By configuring Twilio credentials in a Dapr component, you avoid embedding the Twilio SDK in every service that needs to send texts.

## Prerequisites

- Dapr CLI installed and initialized
- A Twilio account with an Account SID, Auth Token, and phone number
- Basic familiarity with Dapr components

## Define the Twilio SMS Binding Component

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: twilio-sms
  namespace: default
spec:
  type: bindings.twilio.sms
  version: v1
  metadata:
  - name: toNumber
    value: "+15551234567"
  - name: fromNumber
    value: "+15559876543"
  - name: accountSid
    secretKeyRef:
      name: twilio-secret
      key: accountSid
  - name: authToken
    secretKeyRef:
      name: twilio-secret
      key: authToken
```

Create the Kubernetes secret:

```bash
kubectl create secret generic twilio-secret \
  --from-literal=accountSid="ACxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx" \
  --from-literal=authToken="your_auth_token_here"
```

For self-hosted development, you can use environment variables or a local secret file.

## Send an SMS via the Dapr API

```bash
curl -X POST http://localhost:3500/v1.0/bindings/twilio-sms \
  -H "Content-Type: application/json" \
  -d '{
    "data": "Your order #12345 has been shipped and will arrive by Friday.",
    "metadata": {
      "toNumber": "+15551234567"
    },
    "operation": "create"
  }'
```

The `metadata.toNumber` overrides the default `toNumber` in the component for this specific message.

## Use in a Node.js Application

```javascript
const { DaprClient } = require('@dapr/dapr');

const client = new DaprClient();
const BINDING = 'twilio-sms';

async function sendSMS(toNumber, message) {
  await client.binding.send(BINDING, 'create', message, {
    toNumber: toNumber,
  });
  console.log(`SMS sent to ${toNumber}`);
}

async function sendOrderShippedAlert(order) {
  const message = `Hi ${order.customerName}! Your order #${order.id} has shipped. Track it at: https://track.example.com/${order.trackingCode}`;
  await sendSMS(order.customerPhone, message);
}

async function sendVerificationCode(phoneNumber, code) {
  const message = `Your verification code is: ${code}. It expires in 10 minutes. Do not share this code with anyone.`;
  await sendSMS(phoneNumber, message);
}

async function sendAlertToOpsTeam(alertMessage) {
  const opsPhones = ['+15550001111', '+15550002222'];
  const smsPromises = opsPhones.map(phone => sendSMS(phone, alertMessage));
  await Promise.all(smsPromises);
  console.log('Alert sent to ops team');
}
```

## Use in a Python Application

```python
from dapr.clients import DaprClient

BINDING = 'twilio-sms'

def send_sms(to_number: str, message: str):
    with DaprClient() as client:
        client.invoke_binding(
            binding_name=BINDING,
            operation='create',
            data=message,
            binding_metadata={'toNumber': to_number}
        )
    print(f"SMS sent to {to_number}")

def send_appointment_reminder(patient_phone: str, appointment_time: str):
    message = (
        f"Reminder: You have an appointment tomorrow at {appointment_time}. "
        f"Reply CONFIRM to confirm or CANCEL to cancel."
    )
    send_sms(patient_phone, message)

def send_two_factor_code(phone: str, code: str):
    message = f"Your login code is {code}. Valid for 5 minutes."
    send_sms(phone, message)
```

## Best Practices for SMS Messages

Keep SMS messages concise - a single SMS segment is 160 characters for GSM-7 or 70 for Unicode:

```javascript
function truncateToSMSLength(message, maxLength = 160) {
  if (message.length <= maxLength) return message;
  return message.substring(0, maxLength - 3) + '...';
}

async function sendSafeAlert(phone, alertText) {
  const safeMessage = truncateToSMSLength(alertText);
  await sendSMS(phone, safeMessage);
}
```

## Handle Rate Limiting

Twilio enforces rate limits per phone number. Use a queue for bulk sending:

```javascript
async function sendBulkSMS(recipients, message) {
  const DELAY_MS = 100; // 10 messages/second max

  for (const recipient of recipients) {
    await sendSMS(recipient.phone, message);
    await new Promise(resolve => setTimeout(resolve, DELAY_MS));
  }
}
```

## Summary

The Dapr Twilio SMS binding provides a portable, configuration-driven way to send text messages from microservices without the Twilio SDK. Credentials are stored securely in Dapr secret stores, and per-message recipient overrides make it flexible for notification systems, 2FA, and alerting. The same component definition works in both self-hosted and Kubernetes environments.
