# How to Create Elemental Machine Registrations

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Elemental, Kubernetes, Rancher, MachineRegistration, Edge

Description: Learn how to create and configure Elemental MachineRegistration resources to enable bare metal nodes to register with your Rancher management cluster.

## Introduction

A MachineRegistration is the Kubernetes resource that defines how edge or bare metal machines identify themselves and register with the Elemental Operator. When a machine boots with an Elemental OS image, it reads the registration configuration and contacts the Rancher management cluster to join the inventory.

This guide explains the structure of a MachineRegistration and walks through creating one for your environment.

## Prerequisites

- Elemental Operator installed in your Rancher cluster
- `kubectl` access to the management cluster
- A namespace for your Elemental resources

## Understanding MachineRegistration

The `MachineRegistration` resource defines:

- **Registration configuration**: URL and credentials for machines to use when registering
- **Machine labels**: Labels applied to registered machines in the inventory
- **Cloud-config**: Initial OS configuration applied during registration
- **RBAC**: Permissions for machines to interact with the API

## Creating a Basic MachineRegistration

```yaml
# machine-registration.yaml

apiVersion: elemental.cattle.io/v1beta1
kind: MachineRegistration
metadata:
  # Name of the registration endpoint
  name: my-nodes
  namespace: fleet-default
spec:
  # Labels applied to machines that register using this endpoint
  machineLabels:
    element.cattle.io/os.management: managedosversionchannel
    location: datacenter-1
    role: worker

  # Cloud-config applied to the registering machine
  config:
    cloud-config:
      users:
        - name: root
          passwd: "$6$rounds=4096$randomsalt$hashedpassword"

    # Elemental-specific registration configuration
    elemental:
      registration:
        # The operator will populate this URL automatically
        uri: ""
        # CA certificate bundle for TLS verification
        ca-cert: ""
      install:
        # Device to install the OS onto
        device: /dev/sda
        # Reboot after installation
        reboot: true
        # PowerOff after installation (mutually exclusive with reboot)
        poweroff: false
```

```bash
# Apply the MachineRegistration
kubectl apply -f machine-registration.yaml

# Check registration status
kubectl get machineregistration -n fleet-default my-nodes -o yaml
```

## Retrieving the Registration URL

After creating the MachineRegistration, the operator generates a registration endpoint URL:

```bash
# Get the registration URL
kubectl get machineregistration my-nodes -n fleet-default \
  -o jsonpath='{.status.registrationURL}'

# Get the registration token
kubectl get machineregistration my-nodes -n fleet-default \
  -o jsonpath='{.status.registrationToken}'
```

## Advanced MachineRegistration Configuration

### With System Agent Options

```yaml
apiVersion: elemental.cattle.io/v1beta1
kind: MachineRegistration
metadata:
  name: edge-nodes
  namespace: fleet-default
spec:
  machineLabels:
    location: factory-floor
    tier: edge

  config:
    elemental:
      install:
        device: /dev/nvme0n1
        reboot: true
        # Configure system partitions
        partitions:
          persistent:
            size: 50000  # MB
          recovery:
            size: 4096

      # System agent configuration for Rancher connectivity
      system-agent:
        url: "https://rancher.example.com"
        token: "your-rancher-token"
        values:
          server-url: "https://rancher.example.com"
          token: "your-cluster-token"
```

### With Hardware Label Collection

```yaml
apiVersion: elemental.cattle.io/v1beta1
kind: MachineRegistration
metadata:
  name: hardware-labeled-nodes
  namespace: fleet-default
spec:
  # Collect hardware info as labels
  machineInventoryLabels:
    # CPU info
    cpuModel: "${System Information/Manufacturer}"
    # Memory size
    totalMemory: "${Memory Device/Size}"
    # Serial number
    serialNumber: "${System Information/Serial Number}"
```

## Verifying Machine Registration

```bash
# Watch for machines registering
kubectl get machineinventory -n fleet-default --watch

# Describe a registered machine
kubectl describe machineinventory -n fleet-default <machine-name>

# Check machine labels
kubectl get machineinventory -n fleet-default \
  --show-labels
```

## Updating a MachineRegistration

```bash
# Edit the registration in place
kubectl edit machineregistration my-nodes -n fleet-default

# Or apply updated YAML
kubectl apply -f updated-machine-registration.yaml
```

## Conclusion

MachineRegistrations are the entry point for bringing bare metal and edge machines into your Rancher-managed Kubernetes fleet. By defining labels, cloud-config, and installation parameters in a MachineRegistration, you establish a consistent, automated path for machines to self-register and become ready for Kubernetes cluster provisioning.
