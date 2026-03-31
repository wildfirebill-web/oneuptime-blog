# How to Handle Database Connection Strings Securely for MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Security, Connection, Secret, Best Practice

Description: Learn how to handle MySQL connection strings securely by using environment variables, secrets managers, and vault tools to keep credentials out of source code.

---

A hardcoded MySQL connection string in source code is one of the most common causes of credential leaks. Connection strings contain the hostname, port, username, and password in a single line - exposing all of them simultaneously when committed to version control.

## Never Hardcode Connection Strings

The most important rule is to keep credentials out of source code and configuration files that are committed to version control:

```python
# Never do this
engine = create_engine("mysql+pymysql://app_user:SuperSecret123@prod-db.example.com/myapp")

# Good: read from environment variable
import os
DATABASE_URL = os.environ["DATABASE_URL"]
engine = create_engine(DATABASE_URL)
```

## Use Environment Variables for Development and CI

Set the connection string as an environment variable. In development, use a `.env` file that is listed in `.gitignore`:

```bash
# .env (never commit this file)
DATABASE_URL=mysql+pymysql://app_user:dev_password@localhost:3306/myapp_dev
```

Load it with a dotenv library:

```python
from dotenv import load_dotenv
import os

load_dotenv()
engine = create_engine(os.environ["DATABASE_URL"])
```

Add `.env` to `.gitignore`:

```text
.env
.env.local
.env.*.local
```

## Use AWS Secrets Manager in Production

For production workloads, store the connection string in a secrets manager and retrieve it at runtime:

```python
import boto3
import json
import os

def get_db_url():
    client = boto3.client('secretsmanager', region_name=os.environ['AWS_REGION'])
    secret = client.get_secret_value(SecretId='prod/myapp/mysql')
    creds = json.loads(secret['SecretString'])
    return (
        f"mysql+pymysql://{creds['username']}:{creds['password']}"
        f"@{creds['host']}:{creds['port']}/{creds['dbname']}"
    )

engine = create_engine(get_db_url())
```

Cache the secret in memory and rotate it without redeploying by refreshing on a schedule.

## Use Kubernetes Secrets for Container Workloads

In Kubernetes, inject secrets as environment variables from a Secret resource:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mysql-credentials
type: Opaque
stringData:
  DATABASE_URL: "mysql+pymysql://app_user:secret@mysql-service:3306/myapp"
```

Reference the secret in the deployment:

```yaml
env:
  - name: DATABASE_URL
    valueFrom:
      secretKeyRef:
        name: mysql-credentials
        key: DATABASE_URL
```

Use sealed secrets or an external secrets operator to avoid storing plaintext secrets in Git.

## Use Separate Credentials per Environment

Never share database credentials between environments. Each environment should have its own user with only the permissions it needs:

```sql
-- Read-only user for reporting
CREATE USER 'app_reporting'@'%' IDENTIFIED BY 'rpt_password';
GRANT SELECT ON myapp.* TO 'app_reporting'@'%';

-- Application user for production
CREATE USER 'app_prod'@'%' IDENTIFIED BY 'prod_password';
GRANT SELECT, INSERT, UPDATE, DELETE ON myapp.* TO 'app_prod'@'%';
```

## Rotate Credentials Regularly

Automate credential rotation using AWS Secrets Manager rotation Lambda or HashiCorp Vault dynamic secrets:

```bash
# Vault example: generate a short-lived MySQL credential
vault read database/creds/myapp-role
# Key                Value
# username           v-myapp-3fgK2
# password           A1b2C3d4-xyz
# lease_duration     1h
```

## Summary

Secure MySQL connection string handling requires never hardcoding credentials in source code, using environment variables for development, secrets managers for production, and Kubernetes Secrets for container deployments. Separate credentials per environment with least-privilege permissions, and automate rotation to reduce the blast radius of any credential compromise.
