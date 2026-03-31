# How to Set Bucket Policies in Ceph RGW

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, RGW, Bucket Policy, S3, Security

Description: Learn how to configure S3-compatible bucket policies in Ceph RGW to control access by user, IP address, action, and condition for flexible fine-grained permission management.

---

## Bucket Policies vs ACLs

While ACLs control basic read/write permissions, bucket policies provide more expressive access control using JSON policy documents. Bucket policies support:

- Allowing or denying specific S3 actions (GetObject, PutObject, DeleteObject, etc.)
- Restricting access by source IP address
- Granting access to specific users or service accounts
- Time-based and tag-based conditions

## Basic Bucket Policy Structure

An S3 bucket policy follows the IAM policy format:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {"AWS": "arn:aws:iam:::user/myuser"},
      "Action": ["s3:GetObject", "s3:PutObject"],
      "Resource": ["arn:aws:s3:::my-bucket/*"]
    }
  ]
}
```

## Apply a Read-Only Policy

Allow all users to read objects but only the owner to write:

```bash
cat > read-only-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::my-bucket/*"
    },
    {
      "Effect": "Allow",
      "Principal": {"AWS": "arn:aws:iam:::user/owner-user"},
      "Action": "s3:*",
      "Resource": [
        "arn:aws:s3:::my-bucket",
        "arn:aws:s3:::my-bucket/*"
      ]
    }
  ]
}
EOF

aws s3api put-bucket-policy \
  --bucket my-bucket \
  --policy file://read-only-policy.json \
  --endpoint-url http://rook-ceph-rgw-my-store.rook-ceph.svc:80
```

## IP-Based Access Restriction

Restrict bucket access to a specific CIDR range:

```bash
cat > ip-restrict-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:*",
      "Resource": [
        "arn:aws:s3:::secure-bucket",
        "arn:aws:s3:::secure-bucket/*"
      ],
      "Condition": {
        "NotIpAddress": {
          "aws:SourceIp": ["10.0.0.0/8", "172.16.0.0/12"]
        }
      }
    }
  ]
}
EOF

aws s3api put-bucket-policy \
  --bucket secure-bucket \
  --policy file://ip-restrict-policy.json \
  --endpoint-url http://rook-ceph-rgw-my-store.rook-ceph.svc:80
```

## Cross-User Access Policy

Grant a secondary user specific permissions:

```bash
cat > cross-user-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {"AWS": "arn:aws:iam:::user/backup-service"},
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::app-backup-bucket",
        "arn:aws:s3:::app-backup-bucket/*"
      ]
    }
  ]
}
EOF

aws s3api put-bucket-policy \
  --bucket app-backup-bucket \
  --policy file://cross-user-policy.json \
  --endpoint-url http://rook-ceph-rgw-my-store.rook-ceph.svc:80
```

## View and Delete Bucket Policy

```bash
# Get current policy
aws s3api get-bucket-policy \
  --bucket my-bucket \
  --endpoint-url http://rook-ceph-rgw-my-store.rook-ceph.svc:80 | \
  jq -r '.Policy' | jq .

# Delete policy
aws s3api delete-bucket-policy \
  --bucket my-bucket \
  --endpoint-url http://rook-ceph-rgw-my-store.rook-ceph.svc:80
```

## Manage Policies via radosgw-admin

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- bash -c "
  # Get bucket policy
  radosgw-admin bucket policy --bucket=my-bucket

  # Remove bucket policy
  radosgw-admin bucket policy --bucket=my-bucket --delete
"
```

## Verify Policy Enforcement

Test that the policy is working:

```bash
# Test read access with authorized user
AWS_ACCESS_KEY_ID=authorized-key AWS_SECRET_ACCESS_KEY=authorized-secret \
  aws s3 ls s3://my-bucket/ \
  --endpoint-url http://rook-ceph-rgw-my-store.rook-ceph.svc:80

# Test read access with unauthorized user (should fail with 403)
AWS_ACCESS_KEY_ID=other-key AWS_SECRET_ACCESS_KEY=other-secret \
  aws s3 cp s3://my-bucket/file.txt . \
  --endpoint-url http://rook-ceph-rgw-my-store.rook-ceph.svc:80
```

## Summary

Ceph RGW bucket policies use the same S3-compatible JSON policy format as AWS IAM, supporting Allow/Deny effects, principal-based permissions, action-level granularity, and condition-based restrictions like IP filtering. Apply policies with the AWS CLI `put-bucket-policy` command or manage them through `radosgw-admin`. Bucket policies are more expressive than ACLs and are the recommended approach for production access control in multi-user Ceph RGW deployments.
