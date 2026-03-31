# How to Configure Roles in Ceph RGW

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, IAM, Role, Security, Object Storage, Authorization

Description: Learn how to create, manage, and assign IAM roles in Ceph RGW to enable role-based access control for S3 operations using STS.

---

Ceph RGW implements an IAM-compatible role system that allows you to define roles with trust policies and permission policies. Roles are assumed via STS, granting temporary credentials to users or federated identities.

## Creating a Role

A role requires a trust policy defining who can assume it:

```bash
radosgw-admin role create \
  --role-name DataAnalystRole \
  --path "/analytics/" \
  --assume-role-policy-doc '{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Allow",
        "Principal": {
          "AWS": ["arn:aws:iam:::user/alice", "arn:aws:iam:::user/bob"]
        },
        "Action": "sts:AssumeRole"
      }
    ]
  }'
```

## Listing Roles

```bash
# List all roles
radosgw-admin role list

# List roles at a specific path
radosgw-admin role list --path "/analytics/"
```

## Getting Role Details

```bash
radosgw-admin role get --role-name DataAnalystRole
```

## Attaching Permission Policies

Attach inline policies to grant specific S3 permissions:

```bash
radosgw-admin role-policy put \
  --role-name DataAnalystRole \
  --policy-name ReadAnalyticsBuckets \
  --policy-doc '{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Allow",
        "Action": [
          "s3:GetObject",
          "s3:ListBucket",
          "s3:ListAllMyBuckets"
        ],
        "Resource": [
          "arn:aws:s3:::analytics-*",
          "arn:aws:s3:::analytics-*/*"
        ]
      }
    ]
  }'
```

## Listing and Getting Role Policies

```bash
# List policies attached to a role
radosgw-admin role-policy list --role-name DataAnalystRole

# Get a specific policy
radosgw-admin role-policy get \
  --role-name DataAnalystRole \
  --policy-name ReadAnalyticsBuckets
```

## Modifying the Trust Policy

Update who can assume the role:

```bash
radosgw-admin role modify \
  --role-name DataAnalystRole \
  --assume-role-policy-doc '{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Allow",
        "Principal": {"AWS": "arn:aws:iam:::user/charlie"},
        "Action": "sts:AssumeRole"
      }
    ]
  }'
```

## Deleting a Role Policy

```bash
radosgw-admin role-policy delete \
  --role-name DataAnalystRole \
  --policy-name ReadAnalyticsBuckets
```

## Deleting a Role

A role must have no attached policies before deletion:

```bash
radosgw-admin role delete --role-name DataAnalystRole
```

## Assuming the Role via AWS CLI

```bash
aws sts assume-role \
  --role-arn "arn:aws:iam:::role/DataAnalystRole" \
  --role-session-name "alice-analytics" \
  --endpoint-url http://your-rgw-host:7480
```

## Summary

Ceph RGW roles enable a proper separation between identity and permissions. Define trust policies to control who can assume a role, then attach permission policies to grant S3 access. Roles integrate with STS for temporary credential issuance, making them the recommended way to grant access to applications and federated users.
