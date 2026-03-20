# How to Set Up Rancher on AWS GovCloud

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, AWS, GovCloud, Compliance

Description: Deploy Rancher on AWS GovCloud to manage Kubernetes clusters in air-gapped, FedRAMP, and government-compliant environments.

## Introduction

AWS GovCloud is AWS's isolated region designed for US government workloads requiring FedRAMP High, DoD IL2-IL5, and ITAR compliance. Deploying Rancher on GovCloud requires handling air-gapped image pulling, using GovCloud-specific endpoints, and ensuring all components meet compliance requirements. This guide covers the complete deployment process.

## Key Differences in GovCloud

- EC2 endpoint: `ec2.us-gov-west-1.amazonaws.com`
- ECR endpoint: `ACCOUNT.dkr.ecr.us-gov-west-1.amazonaws.com`
- No internet access by default (private VPC)
- Must use a private container registry
- IAM requires GovCloud-specific ARNs (`arn:aws-us-gov:...`)

## Step 1: Prepare a Private Container Registry

```bash
# Log in to GovCloud ECR
aws ecr get-login-password \
  --region us-gov-west-1 \
  | docker login \
  --username AWS \
  --password-stdin \
  <account-id>.dkr.ecr.us-gov-west-1.amazonaws.com

# Create repositories for Rancher images
for repo in rancher/rancher rancher/rancher-agent rancher/shell; do
  aws ecr create-repository \
    --repository-name "${repo}" \
    --region us-gov-west-1
done
```

## Step 2: Mirror Rancher Images to ECR

```bash
# Download the Rancher image list for the target version
VERSION="v2.9.0"
curl -L "https://github.com/rancher/rancher/releases/download/${VERSION}/rancher-images.txt" \
  -o rancher-images.txt

# Pull, retag, and push each image (run from an internet-connected bastion)
PRIVATE_REGISTRY="<account-id>.dkr.ecr.us-gov-west-1.amazonaws.com"

while IFS= read -r image; do
  echo "Mirroring: ${image}"

  # Pull from public registry
  docker pull "${image}" || continue

  # Retag for private registry
  new_image="${PRIVATE_REGISTRY}/${image#*/}"
  docker tag "${image}" "${new_image}"

  # Push to ECR
  docker push "${new_image}"
done < rancher-images.txt
```

## Step 3: Create the RKE2 Cluster on GovCloud EC2

```bash
# Create EC2 instances in a private subnet
# Launch template (cloud-init):
cat << 'EOF' > govcloud-userdata.sh
#!/bin/bash
# Configure private registry for RKE2
mkdir -p /etc/rancher/rke2

cat > /etc/rancher/rke2/registries.yaml << 'REGEOF'
mirrors:
  "docker.io":
    endpoint:
      - "https://<account-id>.dkr.ecr.us-gov-west-1.amazonaws.com"
  "registry.k8s.io":
    endpoint:
      - "https://<account-id>.dkr.ecr.us-gov-west-1.amazonaws.com"
  "gcr.io":
    endpoint:
      - "https://<account-id>.dkr.ecr.us-gov-west-1.amazonaws.com"
REGEOF

# Install RKE2 from private location
curl -sfL https://private-s3.us-gov-west-1.amazonaws.com/rke2-install.sh | sh -

# Configure and start RKE2
cat > /etc/rancher/rke2/config.yaml << 'CONFEOF'
tls-san:
  - <api-server-lb-dns>
cloud-provider-name: aws
CONFEOF

systemctl enable --now rke2-server
EOF
```

## Step 4: Configure AWS GovCloud Endpoints

When using the AWS cloud provider in GovCloud, override the API endpoints:

```ini
# /etc/rancher/rke2/aws-cloud-config.conf
[Global]
zone=us-gov-west-1a
region=us-gov-west-1
vpc=vpc-xxxxxxxx
role-arn=arn:aws-us-gov:iam::ACCOUNT_ID:role/RancherNodeRole  # Note: aws-us-gov ARN

[ServiceOverride "ec2"]
Service=ec2
Region=us-gov-west-1
URL=https://ec2.us-gov-west-1.amazonaws.com

[ServiceOverride "elb"]
Service=elasticloadbalancing
Region=us-gov-west-1
URL=https://elasticloadbalancing.us-gov-west-1.amazonaws.com
```

## Step 5: Install Rancher in Air-Gapped Mode

```bash
# Fetch Rancher Helm chart and unpack locally
helm fetch rancher-stable/rancher \
  --version 2.9.0 \
  --untar \
  --untardir /tmp/rancher-chart

# Install with private registry
helm install rancher /tmp/rancher-chart/rancher \
  --namespace cattle-system \
  --create-namespace \
  --set hostname=rancher.govcloud.internal \
  --set bootstrapPassword=ChangeMeNow! \
  --set rancherImage=<account-id>.dkr.ecr.us-gov-west-1.amazonaws.com/rancher/rancher \
  --set systemDefaultRegistry=<account-id>.dkr.ecr.us-gov-west-1.amazonaws.com \
  --set useBundledSystemChart=true \
  --set ingress.tls.source=secret \
  --set replicas=3
```

## Step 6: Configure FIPS Mode (DoD/IL Compliance)

For DoD IL4/IL5 requirements, enable FIPS-140-2 mode:

```yaml
# /etc/rancher/rke2/config.yaml
fips: true   # Enables FIPS-approved algorithms only
```

```bash
# Verify FIPS mode on the OS (RHEL/FIPS-enabled AMIs)
sudo fips-mode-setup --check
# Expected: FIPS mode is enabled
```

## Step 7: Configure Compliance Audit Logging

```yaml
# /etc/rancher/rke2/config.yaml
kube-apiserver-arg:
  - "audit-log-path=/var/log/kube-audit/audit.log"
  - "audit-log-maxage=90"       # Retain 90 days per NIST 800-53
  - "audit-log-maxbackup=10"
  - "audit-log-maxsize=100"
  - "audit-policy-file=/etc/rancher/rke2/audit-policy.yaml"
```

## Conclusion

Deploying Rancher on AWS GovCloud requires careful attention to air-gapped image mirroring, GovCloud-specific service endpoints, FIPS compliance, and IAM ARN formatting. Once deployed, Rancher provides the same powerful multi-cluster management capabilities in the GovCloud isolated environment as in commercial AWS, enabling government agencies to adopt modern Kubernetes workflows while meeting stringent compliance requirements.
