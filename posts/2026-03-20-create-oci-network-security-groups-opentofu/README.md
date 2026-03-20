# How to Create OCI Network Security Groups with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Oracle Cloud, Network Security Group, OCI, Infrastructure as Code

Description: Learn how to create OCI Network Security Groups (NSGs) with OpenTofu to control traffic at the VNIC level with stateful rules.

OCI Network Security Groups (NSGs) provide stateful traffic control at the VNIC level, similar to AWS Security Groups. Unlike Security Lists (which apply to entire subnets), NSGs attach to individual VNICs, enabling fine-grained control per resource.

## Creating a Network Security Group

```hcl
resource "oci_core_network_security_group" "web" {
  compartment_id = var.compartment_id
  vcn_id         = oci_core_vcn.main.id
  display_name   = "web-nsg"

  freeform_tags = {
    Role       = "web"
    ManagedBy  = "opentofu"
  }
}
```

## Adding Ingress Rules

```hcl
# Allow HTTP from anywhere

resource "oci_core_network_security_group_security_rule" "http" {
  network_security_group_id = oci_core_network_security_group.web.id
  direction                 = "INGRESS"
  protocol                  = "6"       # 6 = TCP, 17 = UDP, 1 = ICMP, all = all
  source                    = "0.0.0.0/0"
  source_type               = "CIDR_BLOCK"
  description               = "Allow HTTP from internet"
  stateless                 = false

  tcp_options {
    destination_port_range {
      min = 80
      max = 80
    }
  }
}

# Allow HTTPS from anywhere
resource "oci_core_network_security_group_security_rule" "https" {
  network_security_group_id = oci_core_network_security_group.web.id
  direction                 = "INGRESS"
  protocol                  = "6"
  source                    = "0.0.0.0/0"
  source_type               = "CIDR_BLOCK"
  description               = "Allow HTTPS from internet"
  stateless                 = false

  tcp_options {
    destination_port_range {
      min = 443
      max = 443
    }
  }
}
```

## NSG-to-NSG Rules

A powerful feature of NSGs is referencing another NSG as the source/destination:

```hcl
resource "oci_core_network_security_group" "app" {
  compartment_id = var.compartment_id
  vcn_id         = oci_core_vcn.main.id
  display_name   = "app-nsg"
}

# Allow traffic from the web NSG to the app NSG on port 8080
resource "oci_core_network_security_group_security_rule" "web_to_app" {
  network_security_group_id = oci_core_network_security_group.app.id
  direction                 = "INGRESS"
  protocol                  = "6"
  source                    = oci_core_network_security_group.web.id
  source_type               = "NETWORK_SECURITY_GROUP"  # Reference another NSG
  description               = "Allow web tier to reach app tier"

  tcp_options {
    destination_port_range {
      min = 8080
      max = 8080
    }
  }
}
```

## Egress Rules

```hcl
resource "oci_core_network_security_group_security_rule" "egress_all" {
  network_security_group_id = oci_core_network_security_group.web.id
  direction                 = "EGRESS"
  protocol                  = "all"
  destination               = "0.0.0.0/0"
  destination_type          = "CIDR_BLOCK"
  description               = "Allow all outbound"
  stateless                 = false
}
```

## Attaching NSGs to Compute Instances

```hcl
resource "oci_core_instance" "web" {
  # ...
  create_vnic_details {
    subnet_id              = oci_core_subnet.public.id
    assign_public_ip       = true
    # Attach to the web NSG
    nsg_ids                = [oci_core_network_security_group.web.id]
  }
}
```

## Conclusion

OCI Network Security Groups provide VNIC-level stateful traffic control. Unlike Security Lists, NSGs can reference other NSGs as sources and destinations, enabling clean tiered architecture rules (e.g., "allow web NSG to reach app NSG"). Attach NSGs to VNICs in compute instances, load balancers, and database systems for fine-grained access control.
