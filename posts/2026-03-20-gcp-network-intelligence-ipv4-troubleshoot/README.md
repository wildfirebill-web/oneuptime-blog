# How to Use GCP Network Intelligence Center to Troubleshoot IPv4 Issues

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: GCP, Network Intelligence Center, Troubleshooting, IPv4, VPC, Connectivity

Description: Use GCP Network Intelligence Center's connectivity tests, network topology, and firewall insights to diagnose and resolve IPv4 connectivity issues in GCP VPCs.

## Introduction

GCP Network Intelligence Center is a suite of tools for monitoring, analyzing, and troubleshooting network configurations. It includes Connectivity Tests, Network Topology, Firewall Insights, and Performance Dashboard — providing visibility into why IPv4 traffic flows or fails.

## Tool Overview

| Tool | Purpose |
|------|---------|
| Connectivity Tests | Simulate and verify packet paths between endpoints |
| Network Topology | Visual map of VPC topology and traffic flows |
| Firewall Insights | Identify overly permissive or unused firewall rules |
| Performance Dashboard | Latency and packet loss between Google services |

## Connectivity Tests

Connectivity Tests simulate packet paths and tell you whether traffic between two endpoints is allowed or blocked — and why.

```bash
# Test TCP connectivity from a VM to an external IP
gcloud network-management connectivity-tests create vm-to-internet \
  --source-instance projects/my-project/zones/us-east1-b/instances/my-vm \
  --destination-ip-address 8.8.8.8 \
  --protocol TCP \
  --destination-port 443

# Test connectivity between two internal VMs
gcloud network-management connectivity-tests create internal-test \
  --source-instance projects/my-project/zones/us-east1-b/instances/vm-a \
  --destination-instance projects/my-project/zones/us-east1-c/instances/vm-b \
  --protocol TCP \
  --destination-port 8080
```

## Running and Retrieving Test Results

```bash
# Run an existing connectivity test
gcloud network-management connectivity-tests run vm-to-internet

# Get the test result
gcloud network-management connectivity-tests describe vm-to-internet \
  --format json | jq '.reachabilityDetails'
```

The result includes a `result` field (`REACHABLE`, `UNREACHABLE`, `AMBIGUOUS`) and a trace showing every hop and the rule that allowed or blocked it.

## Firewall Insights

Identify shadowed, overly permissive, or unused rules:

```bash
# Enable Firewall Insights (requires Firewall Insights API)
gcloud services enable firewallinsights.googleapis.com

# View insights for your project via Cloud Console or REST API
gcloud compute firewall-rules list --filter="network=my-vpc"
```

Navigate to **Network Intelligence Center > Firewall Insights** in the console to see:
- Rules with no hits in the last 30 days
- Rules shadowed by higher-priority rules
- Overly broad source ranges (e.g., 0.0.0.0/0 on sensitive ports)

## Network Topology

View and analyze VPC topology in the console:

```bash
# Enable the Network Management API
gcloud services enable networkmanagement.googleapis.com
```

Navigate to **Network Intelligence Center > Network Topology** to see a visual graph of:
- VPC networks and subnets
- VM-to-VM and VM-to-internet traffic flows
- Cloud Interconnect and VPN connections

## Diagnosing a Specific Connectivity Problem

A common workflow for debugging an unreachable VM:

```bash
# Step 1: Run a connectivity test
gcloud network-management connectivity-tests create debug-test \
  --source-ip 203.0.113.50 \
  --destination-instance projects/my-project/zones/us-east1-b/instances/target-vm \
  --protocol TCP \
  --destination-port 22

# Step 2: Check if the route exists
gcloud compute routes list --filter="network=my-vpc destRange=0.0.0.0/0"

# Step 3: Verify firewall rules
gcloud compute firewall-rules list \
  --filter="targetTags=bastion OR targetTags='' direction=INGRESS" \
  --format="table(name,sourceRanges,allowed,targetTags)"
```

## Performance Dashboard

Monitor latency between GCP regions and to external endpoints:

```bash
# View performance dashboard
gcloud services enable networkmanagement.googleapis.com
# Then navigate to Network Intelligence Center > Performance Dashboard in console
```

## Conclusion

GCP Network Intelligence Center removes the guesswork from network troubleshooting. Connectivity Tests provide definitive answers about reachability, Firewall Insights surface security gaps, and Network Topology gives context for complex multi-VPC environments.
