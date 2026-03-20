# How to Provision GPU Instances for ML with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, GPU, Machine Learning, AWS, Azure, GCP, EC2, Training, Infrastructure as Code

Description: Learn how to provision GPU instances for machine learning training on AWS, Azure, and GCP using OpenTofu, with auto-shutdown, spot instance strategies, and NVIDIA driver configuration.

---

GPU instances accelerate deep learning training but are expensive — an unattended p3.2xlarge costs over $3/hour. OpenTofu provisions GPU compute with auto-shutdown policies, spot instance configuration, and proper NVIDIA driver setup to balance training performance with cost control.

## GPU Instance Cost Comparison

| Provider | Instance | GPUs | On-Demand | Spot/Preemptible |
|----------|----------|------|-----------|-----------------|
| AWS | p3.2xlarge | 1x V100 | $3.06/hr | ~$0.90/hr |
| AWS | p4d.24xlarge | 8x A100 | $32.77/hr | ~$10/hr |
| Azure | NC6s_v3 | 1x V100 | $3.06/hr | ~$0.92/hr |
| GCP | n1+T4 | 1x T4 | $0.95/hr | ~$0.29/hr |

## AWS GPU Instance

```hcl
# aws_gpu.tf
data "aws_ami" "deep_learning" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["Deep Learning AMI GPU PyTorch*Ubuntu*"]
  }
}

resource "aws_instance" "gpu_training" {
  ami                  = data.aws_ami.deep_learning.id
  instance_type        = var.gpu_instance_type  # "p3.2xlarge", "p4d.24xlarge"
  subnet_id            = var.private_subnet_id
  iam_instance_profile = aws_iam_instance_profile.training.name
  key_name             = var.key_pair_name

  vpc_security_group_ids = [aws_security_group.gpu.id]

  # EBS volume sized for datasets + model checkpoints
  root_block_device {
    volume_type           = "gp3"
    volume_size           = 500
    iops                  = 3000
    throughput            = 250
    delete_on_termination = true
    encrypted             = true
  }

  # Instance store for fast scratch storage (NVMe)
  # p3.2xlarge has 1x 488 GB NVMe SSD

  user_data = base64encode(<<-EOF
    #!/bin/bash
    # Auto-shutdown after training completes or 8 hours
    shutdown -h +480 &
    echo "0 * * * * sudo shutdown -c" | crontab -
  EOF
  )

  tags = {
    Name         = "${var.prefix}-gpu-training"
    Environment  = var.environment
    AutoShutdown = "480min"
    ManagedBy    = "opentofu"
  }
}

# Spot instance request for cost savings
resource "aws_spot_instance_request" "gpu_training_spot" {
  count = var.use_spot ? 1 : 0

  ami                  = data.aws_ami.deep_learning.id
  instance_type        = var.gpu_instance_type
  spot_price           = var.spot_max_price  # e.g., "1.00"
  spot_type            = "one-time"
  wait_for_fulfillment = true

  subnet_id              = var.private_subnet_id
  vpc_security_group_ids = [aws_security_group.gpu.id]
  iam_instance_profile   = aws_iam_instance_profile.training.name
  key_name               = var.key_pair_name

  root_block_device {
    volume_type = "gp3"
    volume_size = 500
    encrypted   = true
  }

  tags = {
    Name      = "${var.prefix}-gpu-spot"
    SpotPrice = var.spot_max_price
  }
}
```

## Azure GPU VM

```hcl
# azure_gpu.tf
resource "azurerm_linux_virtual_machine" "gpu" {
  name                = "${var.prefix}-gpu-vm"
  resource_group_name = azurerm_resource_group.ml.name
  location            = azurerm_resource_group.ml.location
  size                = var.gpu_vm_size  # "Standard_NC6s_v3", "Standard_ND96asr_v4"

  admin_username                  = "azureuser"
  disable_password_authentication = true

  admin_ssh_key {
    username   = "azureuser"
    public_key = var.ssh_public_key
  }

  network_interface_ids = [azurerm_network_interface.gpu.id]

  # NVIDIA GPU Optimized image with CUDA pre-installed
  source_image_reference {
    publisher = "microsoft-dsvm"
    offer     = "ubuntu-2004"
    sku       = "2004-gen2"
    version   = "latest"
  }

  os_disk {
    storage_account_type = "Premium_LRS"
    disk_size_gb         = 256
    caching              = "ReadWrite"
  }

  # Data disk for training data
  dynamic "data_disk" {
    for_each = [0]
    content {}
  }

  identity {
    type = "SystemAssigned"
  }

  # Auto-shutdown schedule
  lifecycle {
    ignore_changes = [source_image_reference]
  }
}

# Auto-shutdown at end of business day
resource "azurerm_dev_test_global_vm_shutdown_schedule" "gpu" {
  virtual_machine_id = azurerm_linux_virtual_machine.gpu.id
  location           = azurerm_resource_group.ml.location
  enabled            = true

  daily_recurrence_time = "1900"  # 7 PM
  timezone              = "UTC"

  notification_settings {
    enabled         = false
  }
}
```

## GCP GPU Instance

```hcl
# gcp_gpu.tf
resource "google_compute_instance" "gpu" {
  name         = "${var.prefix}-gpu-training"
  machine_type = "n1-standard-8"
  zone         = "${var.region}-a"

  scheduling {
    preemptible         = var.use_preemptible
    automatic_restart   = false
    on_host_maintenance = "TERMINATE"  # Required for GPU instances
  }

  guest_accelerator {
    type  = var.gpu_type   # "nvidia-tesla-t4", "nvidia-a100-80gb"
    count = var.gpu_count
  }

  boot_disk {
    initialize_params {
      image = "projects/deeplearning-platform-release/global/images/family/tf-latest-gpu"
      size  = 200
      type  = "pd-ssd"
    }
  }

  network_interface {
    network    = var.network
    subnetwork = var.subnetwork
    # No access_config = no public IP
  }

  service_account {
    email  = google_service_account.training.email
    scopes = ["cloud-platform"]
  }

  metadata = {
    install-nvidia-driver = "True"
    # Auto-shutdown via startup script
    startup-script = "sudo shutdown -h +${var.max_runtime_hours * 60} &"
  }

  labels = {
    environment = var.environment
    managed-by  = "opentofu"
  }
}
```

## Best Practices

- Always configure auto-shutdown for training instances — a forgotten GPU instance is one of the most common sources of unexpected cloud bills in ML teams.
- Use spot/preemptible instances for training jobs that can checkpoint their progress — they cost 60-80% less than on-demand and are acceptable for non-time-critical training.
- Use Deep Learning AMIs/images that include CUDA, cuDNN, and popular frameworks pre-installed — building NVIDIA drivers from scratch adds 30+ minutes to instance startup.
- Place GPU instances in private subnets with IAM-based access (SSM or IAP) rather than SSH over the internet — ML compute rarely needs inbound public access.
- For large training jobs, use SageMaker/Vertex AI managed training rather than raw instances — they handle checkpointing, spot interruption recovery, and distributed training orchestration automatically.
