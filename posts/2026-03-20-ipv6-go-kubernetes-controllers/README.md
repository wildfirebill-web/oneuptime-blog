# How to Handle IPv6 in Go Kubernetes Controllers

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Go, Kubernetes, IPv6, Controller, Operator, Networking

Description: Handle IPv6 in Go Kubernetes controllers including dual-stack service creation, IPv6 address validation in CRDs, and IPv6-aware networking policies.

## Kubernetes Dual-Stack Overview

Kubernetes 1.21+ supports dual-stack natively. Controllers need to handle both IPv4 and IPv6 addresses for Services, Pods, and Nodes.

Key dual-stack Kubernetes concepts:
- Services can have both IPv4 and IPv6 ClusterIPs
- Pods get both IPv4 and IPv6 addresses in dual-stack clusters
- IPFamilyPolicy controls whether services are single-stack or dual-stack

## Detecting Cluster IPv6 Support

```go
package main

import (
    "context"
    "fmt"
    "net"

    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    "k8s.io/client-go/kubernetes"
    "k8s.io/client-go/rest"
)

func isClusterDualStack(clientset *kubernetes.Clientset) (bool, error) {
    // Check if cluster has IPv6 service CIDR
    nodes, err := clientset.CoreV1().Nodes().List(context.Background(), metav1.ListOptions{})
    if err != nil {
        return false, err
    }

    for _, node := range nodes.Items {
        for _, addr := range node.Status.Addresses {
            ip := net.ParseIP(addr.Address)
            if ip != nil && ip.To4() == nil {
                return true, nil  // Found an IPv6 address
            }
        }
    }
    return false, nil
}
```

## Creating a Dual-Stack Service

```go
package main

import (
    "context"
    "net/netip"

    corev1 "k8s.io/api/core/v1"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    "k8s.io/client-go/kubernetes"
)

func createDualStackService(
    clientset *kubernetes.Clientset,
    namespace, name string,
    selector map[string]string,
    port int32,
) error {
    // Dual-stack service requires IPv6 support in cluster
    ipFamilyPolicyPreferDualStack := corev1.IPFamilyPolicyPreferDualStack
    ipFamilies := []corev1.IPFamily{
        corev1.IPv4Protocol,
        corev1.IPv6Protocol,
    }

    service := &corev1.Service{
        ObjectMeta: metav1.ObjectMeta{
            Name:      name,
            Namespace: namespace,
        },
        Spec: corev1.ServiceSpec{
            Selector:       selector,
            IPFamilyPolicy: &ipFamilyPolicyPreferDualStack,
            IPFamilies:     ipFamilies,
            Ports: []corev1.ServicePort{
                {
                    Port:     port,
                    Protocol: corev1.ProtocolTCP,
                },
            },
            Type: corev1.ServiceTypeClusterIP,
        },
    }

    _, err := clientset.CoreV1().Services(namespace).Create(
        context.Background(), service, metav1.CreateOptions{},
    )
    return err
}
```

## IPv6 Address Validation in CRD Webhook

```go
package main

import (
    "fmt"
    "net/netip"

    "k8s.io/apimachinery/pkg/runtime"
    "sigs.k8s.io/controller-runtime/pkg/webhook/admission"
)

// NetworkDevice is a custom resource with IPv6 address
type NetworkDevice struct {
    metav1.TypeMeta   `json:",inline"`
    metav1.ObjectMeta `json:"metadata,omitempty"`
    Spec              NetworkDeviceSpec `json:"spec,omitempty"`
}

type NetworkDeviceSpec struct {
    IPv6Address string `json:"ipv6Address"`
    PrefixLen   int    `json:"prefixLen,omitempty"`
}

// ValidateCreate validates IPv6 on CRD creation
func (d *NetworkDevice) ValidateCreate() (admission.Warnings, error) {
    return nil, d.validateIPv6()
}

func (d *NetworkDevice) ValidateUpdate(old runtime.Object) (admission.Warnings, error) {
    return nil, d.validateIPv6()
}

func (d *NetworkDevice) validateIPv6() error {
    addr, err := netip.ParseAddr(d.Spec.IPv6Address)
    if err != nil || !addr.Is6() {
        return fmt.Errorf("spec.ipv6Address is not a valid IPv6 address: %s",
            d.Spec.IPv6Address)
    }

    if d.Spec.PrefixLen > 0 {
        if d.Spec.PrefixLen < 0 || d.Spec.PrefixLen > 128 {
            return fmt.Errorf("spec.prefixLen must be between 0 and 128")
        }
    }
    return nil
}
```

## Getting Pod IPv6 Addresses in a Controller

```go
package main

import (
    "context"
    "fmt"
    "net"

    corev1 "k8s.io/api/core/v1"
    "sigs.k8s.io/controller-runtime/pkg/client"
)

func getPodIPv6Addresses(
    ctx context.Context,
    k8sClient client.Client,
    namespace, name string,
) ([]string, error) {
    pod := &corev1.Pod{}
    if err := k8sClient.Get(ctx, client.ObjectKey{
        Namespace: namespace,
        Name:      name,
    }, pod); err != nil {
        return nil, err
    }

    var ipv6Addrs []string
    for _, podIP := range pod.Status.PodIPs {
        ip := net.ParseIP(podIP.IP)
        if ip != nil && ip.To4() == nil {
            ipv6Addrs = append(ipv6Addrs, podIP.IP)
        }
    }

    return ipv6Addrs, nil
}
```

## NetworkPolicy for IPv6

```go
package main

import (
    "context"
    "net"

    networkingv1 "k8s.io/api/networking/v1"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    "k8s.io/client-go/kubernetes"
)

func createIPv6NetworkPolicy(
    clientset *kubernetes.Clientset,
    namespace string,
) error {
    policy := &networkingv1.NetworkPolicy{
        ObjectMeta: metav1.ObjectMeta{
            Name:      "allow-ipv6-internal",
            Namespace: namespace,
        },
        Spec: networkingv1.NetworkPolicySpec{
            PodSelector: metav1.LabelSelector{},
            PolicyTypes: []networkingv1.PolicyType{
                networkingv1.PolicyTypeIngress,
            },
            Ingress: []networkingv1.NetworkPolicyIngressRule{
                {
                    From: []networkingv1.NetworkPolicyPeer{
                        {
                            // Allow from internal IPv6 prefix only
                            IPBlock: &networkingv1.IPBlock{
                                CIDR:   "2001:db8::/48",
                                Except: []string{"2001:db8:ff::/64"},
                            },
                        },
                    },
                },
            },
        },
    }

    _, err := clientset.NetworkingV1().NetworkPolicies(namespace).Create(
        context.Background(), policy, metav1.CreateOptions{},
    )
    return err
}
```

## Conclusion

Handling IPv6 in Go Kubernetes controllers involves using dual-stack service specs, validating IPv6 addresses in admission webhooks, and working with both IPv4 and IPv6 pod addresses. The `net/netip` package provides reliable IPv6 address validation for CRD admission. Always check `IPFamilyPolicy` when creating services to ensure dual-stack behavior in clusters that support it.
