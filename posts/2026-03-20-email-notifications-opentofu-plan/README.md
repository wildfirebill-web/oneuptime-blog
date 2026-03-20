# How to Set Up Email Notifications for OpenTofu Plan Results

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Email, Notifications, CI/CD, DevOps, Automation

Description: Learn how to send email notifications with OpenTofu plan and apply results using SMTP, AWS SES, or SendGrid from your CI/CD pipelines.

## Introduction

Email notifications provide an audit trail and ensure stakeholders are informed about infrastructure changes. This guide covers sending formatted email reports from OpenTofu pipelines using multiple delivery methods.

## Capturing Plan Output

Before sending emails, capture the plan output to a file.

```bash
# Generate a readable plan output file

tofu plan -out=tfplan -no-color 2>&1 | tee plan_output.txt

# Generate a JSON summary for structured reporting
tofu show -json tfplan > plan.json
```

## Sending Email with sendmail/SMTP

A shell script that sends plan results via SMTP using `sendmail` or `curl`.

```bash
#!/bin/bash
# scripts/send-email.sh
SUBJECT="$1"
BODY_FILE="$2"
TO="${EMAIL_RECIPIENT:-ops@example.com}"
FROM="${EMAIL_SENDER:-noreply@example.com}"

# Use curl with SMTP (works with most SMTP servers)
curl -s --ssl-reqd \
  --url "smtps://${SMTP_HOST}:465" \
  --user "${SMTP_USER}:${SMTP_PASS}" \
  --mail-from "$FROM" \
  --mail-rcpt "$TO" \
  --upload-file - <<EOF
From: OpenTofu Pipeline <$FROM>
To: Ops Team <$TO>
Subject: $SUBJECT
Content-Type: text/plain

$(cat "$BODY_FILE")
EOF
```

## GitHub Actions with AWS SES

Use AWS SES to send plan result emails from your GitHub Actions pipeline.

```yaml
# .github/workflows/opentofu-notify.yml
name: OpenTofu with Email Notifications

on:
  push:
    branches: [main]

jobs:
  plan-and-notify:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup OpenTofu
        uses: opentofu/setup-opentofu@v1
        with:
          tofu_version: "1.9.0"

      - name: OpenTofu Init and Plan
        run: |
          tofu init
          tofu plan -out=tfplan -no-color 2>&1 | tee plan_output.txt
          echo "PLAN_SUMMARY=$(grep '^Plan:' plan_output.txt || echo 'No changes')" >> $GITHUB_ENV
        env:
          AWS_ACCESS_KEY_ID:     ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}

      - name: Send Plan Email via AWS SES
        run: |
          aws ses send-email \
            --from "infra-bot@example.com" \
            --to "${{ vars.OPS_EMAIL }}" \
            --subject "OpenTofu Plan: ${{ github.ref_name }} – ${{ env.PLAN_SUMMARY }}" \
            --text "$(cat plan_output.txt)" \
            --region us-east-1
        env:
          AWS_ACCESS_KEY_ID:     ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
```

## Sending HTML Emails with SendGrid

For richer HTML email reports, use the SendGrid API.

```bash
#!/bin/bash
# scripts/send-sendgrid.sh
PLAN_CONTENT=$(cat plan_output.txt | sed 's/</\&lt;/g' | sed 's/>/\&gt;/g')

curl -s -X POST https://api.sendgrid.com/v3/mail/send \
  -H "Authorization: Bearer ${SENDGRID_API_KEY}" \
  -H "Content-Type: application/json" \
  -d @- <<EOF
{
  "personalizations": [{"to": [{"email": "${OPS_EMAIL}"}]}],
  "from": {"email": "infra@example.com", "name": "OpenTofu Pipeline"},
  "subject": "OpenTofu Plan Result – $(date +%Y-%m-%d)",
  "content": [
    {
      "type": "text/html",
      "value": "<h2>OpenTofu Plan Results</h2><pre>${PLAN_CONTENT}</pre>"
    }
  ]
}
EOF
```

## GitLab CI Example

```yaml
# .gitlab-ci.yml
plan:
  stage: plan
  script:
    - tofu init
    - tofu plan -out=tfplan -no-color 2>&1 | tee plan_output.txt
  after_script:
    - ./scripts/send-email.sh "Plan Result for $CI_COMMIT_BRANCH" plan_output.txt
  variables:
    SMTP_HOST: smtp.sendgrid.net
    SMTP_USER: apikey
```

## Summary

Email notifications for OpenTofu plan results help operations teams stay informed about infrastructure changes without requiring constant pipeline monitoring. Whether you use AWS SES, SendGrid, or plain SMTP, a consistent notification strategy improves operational awareness and change accountability.
