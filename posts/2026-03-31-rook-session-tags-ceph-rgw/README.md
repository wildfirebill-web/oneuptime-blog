# How to Use Session Tags with Ceph RGW

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Security, IAM, Session, Object Storage, Authorization

Description: Learn how to use session tags with Ceph RGW STS to pass attributes that can be used in IAM policy conditions for fine-grained access control.

---

Session tags are key-value pairs that you can pass when assuming an IAM role via STS. These tags become available as condition variables in IAM policies, enabling attribute-based access control (ABAC) without creating separate roles for each user or group.

## What Are Session Tags

When you call `AssumeRole` or `AssumeRoleWithWebIdentity`, you can include session tags. These tags are then accessible in policy conditions via the `aws:PrincipalTag` prefix.

## Setting Up a Role with Tag-Based Policies

Create a role and attach a policy that uses session tags as conditions:

```bash
radosgw-admin role create \
  --role-name TaggedAccessRole \
  --path "/" \
  --assume-role-policy-doc '{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Allow",
        "Principal": {"AWS": "arn:aws:iam:::user/myuser"},
        "Action": "sts:AssumeRole",
        "Condition": {
          "StringLike": {"sts:TagKeys": "department"}
        }
      }
    ]
  }'
```

Attach a policy that restricts access based on the `department` session tag:

```bash
radosgw-admin role-policy put \
  --role-name TaggedAccessRole \
  --policy-name DepartmentBucketPolicy \
  --policy-doc '{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Allow",
        "Action": ["s3:GetObject", "s3:PutObject"],
        "Resource": "arn:aws:s3:::${aws:PrincipalTag/department}/*"
      }
    ]
  }'
```

## Passing Session Tags with AssumeRole

```bash
aws sts assume-role \
  --role-arn "arn:aws:iam:::role/TaggedAccessRole" \
  --role-session-name "alice-session" \
  --tags '[{"Key":"department","Value":"engineering"},{"Key":"team","Value":"backend"}]' \
  --endpoint-url http://your-rgw-host:7480
```

## Using Tags in Python with boto3

```python
import boto3

sts = boto3.client(
    'sts',
    endpoint_url='http://your-rgw-host:7480',
    aws_access_key_id='myuser-access-key',
    aws_secret_access_key='myuser-secret-key'
)

response = sts.assume_role(
    RoleArn='arn:aws:iam:::role/TaggedAccessRole',
    RoleSessionName='alice-session',
    Tags=[
        {'Key': 'department', 'Value': 'engineering'},
        {'Key': 'project', 'Value': 'infra-tools'}
    ]
)

creds = response['Credentials']
print(f"Access Key: {creds['AccessKeyId']}")
```

## Transitive Session Tags

Mark tags as transitive so they persist when the assumed role assumes another role:

```bash
aws sts assume-role \
  --role-arn "arn:aws:iam:::role/TaggedAccessRole" \
  --role-session-name "alice-session" \
  --tags '[{"Key":"department","Value":"engineering"}]' \
  --transitive-tag-keys department \
  --endpoint-url http://your-rgw-host:7480
```

## Inheriting Tags from OIDC Tokens

When using `AssumeRoleWithWebIdentity`, RGW can map OIDC claims to session tags based on the role's tag mapping configuration. Ensure the JWT contains the relevant claims and the role trust policy references them via `sts:RequestTag`.

## Summary

Session tags in Ceph RGW enable attribute-based access control by allowing callers to inject key-value pairs into their assumed role sessions. Policy conditions then use `aws:PrincipalTag` to dynamically restrict access based on those tags, reducing the number of roles needed and enabling a more scalable permission model.
