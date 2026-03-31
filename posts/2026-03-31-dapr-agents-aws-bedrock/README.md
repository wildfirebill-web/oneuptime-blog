# How to Use Dapr Agents with AWS Bedrock

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Agent, AWS, Bedrock, LLM

Description: Learn how to configure Dapr Agents with AWS Bedrock to run AI agents using Claude, Llama, and other models managed through AWS's secure infrastructure.

---

## Why Use AWS Bedrock with Dapr Agents?

AWS Bedrock provides managed access to foundation models including Claude, Llama, Titan, and Cohere, with AWS IAM-based authentication and no external API keys. When running Dapr Agents on AWS (EKS, ECS, EC2), Bedrock provides enterprise-grade security, compliance, and model governance without leaving the AWS ecosystem.

## Prerequisites

Ensure your AWS environment is configured:

```bash
# Configure AWS credentials
aws configure

# Or use instance profile / IRSA on EKS
# Verify Bedrock model access
aws bedrock list-foundation-models --region us-east-1 | jq '.modelSummaries[].modelId'
```

Enable model access in the AWS console for the models you want to use.

## Installation

```bash
pip install dapr-agents boto3
```

## Configuring the AWS Bedrock LLM Client

```python
from dapr_agents.llm import AWSBedrockChat
import boto3

# Using default credentials (instance profile, IRSA, or aws configure)
llm = AWSBedrockChat(
    model="anthropic.claude-3-5-sonnet-20241022-v2:0",
    region_name="us-east-1",
    max_tokens=4096
)
```

## Building an Agent with Bedrock Claude

```python
import os
from dapr_agents import Agent, tool
from dapr_agents.llm import AWSBedrockChat

class ComplianceAgent(Agent):
    name = "compliance-agent"
    instructions = """You are a compliance analyst. Evaluate documents,
    policies, and processes against regulatory requirements.
    Provide clear findings with risk levels."""

    @tool
    def check_gdpr_compliance(self, document: str) -> str:
        """Checks document content for GDPR compliance issues.

        Args:
            document: The text content to evaluate.
        """
        issues = []
        if "personal data" in document.lower() and "consent" not in document.lower():
            issues.append("Missing consent mechanism for personal data processing")
        if "retention" not in document.lower():
            issues.append("No data retention policy specified")
        return f"GDPR check: {len(issues)} issues found - {'; '.join(issues) or 'None'}"

    @tool
    def generate_audit_report(self, findings: str, risk_level: str) -> str:
        """Generates a structured audit report from findings.

        Args:
            findings: Summary of compliance findings.
            risk_level: Risk level (low, medium, high, critical).
        """
        return f"Audit Report [{risk_level.upper()} RISK]:\n{findings}"


llm = AWSBedrockChat(
    model="anthropic.claude-3-5-sonnet-20241022-v2:0",
    region_name="us-east-1"
)

agent = ComplianceAgent(llm=llm)
result = agent.run("Review this privacy policy for GDPR compliance issues.")
print(result)
```

## Using Llama on Bedrock

Dapr Agents also supports Meta's Llama models via Bedrock:

```python
from dapr_agents.llm import AWSBedrockChat

llm = AWSBedrockChat(
    model="meta.llama3-70b-instruct-v1:0",
    region_name="us-west-2",
    max_tokens=2048
)
```

## IAM Permissions for Bedrock

The IAM role for your agent needs these permissions:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "bedrock:InvokeModel",
        "bedrock:InvokeModelWithResponseStream"
      ],
      "Resource": [
        "arn:aws:bedrock:us-east-1::foundation-model/anthropic.claude-3-5-sonnet-20241022-v2:0"
      ]
    }
  ]
}
```

## Deploying on EKS with IRSA

For EKS deployments, use IAM Roles for Service Accounts (IRSA) to avoid storing credentials:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: compliance-agent
  namespace: default
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/ComplianceAgentRole
```

```bash
dapr run --app-id compliance-agent \
  --app-port 8080 \
  --components-path ./components \
  -- python agent.py
```

## Cross-Region Inference

For high availability, enable cross-region inference on Bedrock:

```python
llm = AWSBedrockChat(
    model="us.anthropic.claude-3-5-sonnet-20241022-v2:0",  # cross-region profile
    region_name="us-east-1"
)
```

## Summary

Dapr Agents integrates with AWS Bedrock through the `AWSBedrockChat` client, supporting Claude, Llama, and other foundation models. Use IAM instance profiles or IRSA for credential-free authentication on AWS. Apply IAM policies scoped to specific model ARNs, and enable cross-region inference profiles for high availability.
