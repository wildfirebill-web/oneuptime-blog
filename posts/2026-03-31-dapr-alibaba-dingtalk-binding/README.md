# How to Use Dapr Alibaba Cloud DingTalk Binding

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Alibaba Cloud, DingTalk, Binding, Notification

Description: Learn how to configure and use the Dapr Alibaba Cloud DingTalk output binding to send messages and alerts to DingTalk groups from your microservices.

---

## What Is the Dapr DingTalk Binding?

DingTalk is Alibaba Cloud's enterprise communication and collaboration platform, widely used in China and Asia-Pacific. The Dapr DingTalk output binding lets microservices send messages to DingTalk group chatbots without managing DingTalk Webhook authentication and message formatting.

## Setting Up a DingTalk Group Bot

1. Open DingTalk and navigate to the group chat
2. Click the group settings icon and select "Intelligent Group Assistant"
3. Click "Add Robot" and choose "Custom"
4. Name the bot (e.g., "Alert Bot") and note the Webhook URL
5. Enable security settings - use "Signature" for production

The webhook URL looks like:
```yaml
https://oapi.dingtalk.com/robot/send?access_token=<your-token>
```

## Configuring the DingTalk Binding Component

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: dingtalk-alert
  namespace: default
spec:
  type: bindings.dingtalk.webhook
  version: v1
  metadata:
    - name: id
      value: "alert-bot"
    - name: url
      secretKeyRef:
        name: dingtalk-secrets
        key: webhookUrl
    - name: secret
      secretKeyRef:
        name: dingtalk-secrets
        key: signingSecret
```

```bash
kubectl create secret generic dingtalk-secrets \
  --from-literal=webhookUrl="https://oapi.dingtalk.com/robot/send?access_token=..." \
  --from-literal=signingSecret="SEC..."
```

## Sending a Text Message

```javascript
const { DaprClient } = require("@dapr/dapr");
const client = new DaprClient();

async function sendDingTalkText(message, atMobiles = []) {
  await client.binding.send("dingtalk-alert", "create", {
    msgtype: "text",
    text: {
      content: message,
    },
    at: {
      atMobiles,
      isAtAll: false,
    },
  });

  console.log("DingTalk text message sent");
}

await sendDingTalkText("Deployment to production completed successfully!");
await sendDingTalkText("Critical: Database CPU at 95%!", ["13800138000"]);
```

## Sending a Markdown Message

DingTalk's Markdown format supports rich formatting for alerts and reports:

```javascript
async function sendDingTalkMarkdown(title, markdownContent, atAll = false) {
  await client.binding.send("dingtalk-alert", "create", {
    msgtype: "markdown",
    markdown: {
      title,
      text: markdownContent,
    },
    at: {
      isAtAll: atAll,
    },
  });
}

const report = {
  date: "2026-03-31",
  totalOrders: 312,
  revenue: 45230.50,
  errorRate: 0.02,
};

await sendDingTalkMarkdown(
  "Daily Operations Report",
  `## Daily Operations Report - ${report.date}

| Metric | Value |
|--------|-------|
| Total Orders | ${report.totalOrders} |
| Revenue | ¥${report.revenue.toFixed(2)} |
| Error Rate | ${(report.errorRate * 100).toFixed(2)}% |

${report.errorRate > 0.05 ? "> **Warning**: Error rate exceeded 5% threshold" : "> All metrics within normal range"}
  `
);
```

## Sending Action Card Messages

Action cards add interactive buttons to messages:

```javascript
async function sendDeploymentApproval(deploymentId, environment, changes) {
  await client.binding.send("dingtalk-alert", "create", {
    msgtype: "actionCard",
    actionCard: {
      title: `Deployment Approval Required - ${environment}`,
      text: `## Deployment ${deploymentId}

**Environment**: ${environment}
**Changes**: ${changes.join(", ")}
**Requested by**: CI/CD Pipeline
**Time**: ${new Date().toLocaleString("zh-CN")}

Please review and approve or reject this deployment.`,
      btnOrientation: "0",
      btns: [
        {
          title: "Approve",
          actionURL: `https://deploy.example.com/approve/${deploymentId}`,
        },
        {
          title: "Reject",
          actionURL: `https://deploy.example.com/reject/${deploymentId}`,
        },
      ],
    },
  });
}

await sendDeploymentApproval(
  "DEPLOY-2026-0331-001",
  "production",
  ["v2.3.1 -> v2.4.0", "Config updates"]
);
```

## Building an Alert Routing Layer

```javascript
async function sendAlert(severity, title, details) {
  const severityEmoji = {
    info: "INFO",
    warning: "WARNING",
    critical: "CRITICAL - @all",
  };

  const markdownBody = `## [${severityEmoji[severity] || severity}] ${title}

${details}

**Time**: ${new Date().toISOString()}
**Source**: order-service`;

  await sendDingTalkMarkdown(
    `[${severity.toUpperCase()}] ${title}`,
    markdownBody,
    severity === "critical"
  );
}

await sendAlert("critical", "Payment Service Down", "Payment processor returning 503 errors. Immediate action required.");
await sendAlert("warning", "High Memory Usage", "order-service pods at 85% memory capacity.");
await sendAlert("info", "Batch Job Complete", "Nightly sales report generated and uploaded.");
```

## Summary

The Dapr DingTalk binding provides a simple way to deliver operational alerts, deployment notifications, and daily reports to DingTalk group chats. Supporting text, Markdown, and action card message types, it covers the most common enterprise notification patterns. Configure signature-based security for production bots and use Dapr's secret references to keep webhook URLs and signing secrets out of your component YAML files.
