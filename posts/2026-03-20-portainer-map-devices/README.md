# How to Map Devices to Containers in Portainer - A Practical Guide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Containers, Hardware, IoT

Description: Learn how to map host devices like serial ports, USB devices, and GPUs to Docker containers using Portainer's device mapping configuration.

## Introduction

Some containerized applications need direct access to hardware devices - serial ports for IoT/industrial equipment, USB devices for data acquisition, GPUs for inference, or specialized hardware accelerators. Docker's device mapping feature lets you expose these host devices inside containers, and Portainer makes this configuration accessible through the web UI.

## Prerequisites

- Portainer installed with a connected Docker environment
- Physical or virtual devices accessible on the Docker host
- Knowledge of your device file paths (e.g., `/dev/ttyS0`, `/dev/video0`)

## How Device Mapping Works

Device mapping uses Docker's `--device` flag, which adds a device file to the container:

```bash
# Docker CLI equivalent:

docker run --device /dev/ttyS0:/dev/ttyS0 myimage

# Format: /host/device:/container/device[:permissions]
# Permissions: r (read), w (write), m (mknod) - default: rwm
```

The device is accessible inside the container at the specified path, with the same underlying device driver as on the host.

## Step 1: Find Your Device on the Host

Before mapping, identify the device file:

```bash
# List serial/USB devices
ls -la /dev/tty* /dev/ttyUSB* /dev/ttyACM*

# List video devices (webcams, capture cards)
ls -la /dev/video*

# List disk/storage devices
ls -la /dev/sd* /dev/nvme*

# List GPIO devices (Raspberry Pi, etc.)
ls -la /dev/gpiochip*

# Identify USB devices and their /dev path
lsusb
udevadm info -n /dev/ttyUSB0 | grep DEVNAME
```

## Step 2: Map Devices in Portainer

1. Navigate to **Containers > Add container**.
2. Scroll to the **Runtime & Resources** or **Devices** section.
3. Click **+ add device**.
4. Fill in the fields:

```text
Host device path:       /dev/ttyS0
Container device path:  /dev/ttyS0
Permissions:            rwm
```

Or:

```text
Host device path:       /dev/video0
Container device path:  /dev/video0
Permissions:            r
```

## Step 3: Common Device Mapping Examples

### Serial Port (RS-232/RS-485)

For industrial equipment, modems, Arduino, etc.:

```yaml
# docker-compose.yml
devices:
  - /dev/ttyS0:/dev/ttyS0    # First serial port
  - /dev/ttyS1:/dev/ttyS1    # Second serial port
  - /dev/ttyUSB0:/dev/ttyUSB0  # USB-serial adapter

# Also may need group permissions:
group_add:
  - dialout   # Add container user to dialout group (UID 20 on most systems)
```

### USB Device

```yaml
devices:
  - /dev/bus/usb/001/003:/dev/bus/usb/001/003  # Specific USB device

# Or for USB device classes:
devices:
  - /dev/hidraw0:/dev/hidraw0  # HID device (USB barcode scanner, etc.)
```

### Webcam / Video Capture

```yaml
devices:
  - /dev/video0:/dev/video0   # Default webcam
  - /dev/video1:/dev/video1   # Second camera

# May also need access to V4L2 devices:
devices:
  - /dev/video0:/dev/video0
  - /dev/vbi0:/dev/vbi0
```

### GPIO (Raspberry Pi)

```yaml
devices:
  - /dev/gpiochip0:/dev/gpiochip0  # GPIO chip access
  - /dev/mem:/dev/mem              # Direct memory access (use with caution)
```

### Sound Devices

```yaml
devices:
  - /dev/snd:/dev/snd  # Entire sound subsystem

# Or specific ALSA devices:
  - /dev/snd/controlC0:/dev/snd/controlC0
  - /dev/snd/pcmC0D0p:/dev/snd/pcmC0D0p
```

## Step 4: Handle Device Permissions

If the container user can't access the device:

```bash
# Check device permissions on host:
ls -la /dev/ttyS0
# crw-rw---- 1 root dialout 4, 64 Mar 20 10:00 /dev/ttyS0

# The device is owned by group 'dialout'
# Find the GID of the dialout group:
getent group dialout
# dialout:x:20:user1,user2
```

In the container compose config, add the group:

```yaml
services:
  iot-app:
    image: myorg/iot-app:latest
    devices:
      - /dev/ttyS0:/dev/ttyS0
    group_add:
      - "20"   # GID of 'dialout' group
```

## Step 5: Using cgroup Rules for Dynamic Devices

For devices that may appear after container start (e.g., USB devices that are hot-plugged), use cgroup device rules instead:

```yaml
services:
  device-manager:
    image: myorg/device-manager:latest
    # Allow access to USB subsystem devices
    device_cgroup_rules:
      - 'c 189:* rmw'  # USB devices (major 189)
    volumes:
      - /dev:/dev      # Mount entire /dev (requires careful security consideration)
    privileged: false
```

## Step 6: Verify Device Access Inside Container

After starting the container, verify the device is accessible:

```bash
# In the container console (via Portainer Exec):
ls -la /dev/ttyS0

# Test a serial device:
cat /dev/ttyS0 &    # Background read
echo "test" > /dev/ttyS0

# Check device permissions:
stat /dev/ttyS0
```

## Security Considerations

- **Prefer specific device paths** over broad mounts like `/dev` to minimize attack surface.
- **Use read-only permissions** (`r`) where write access is not needed.
- **Avoid `--privileged`** when device mapping is sufficient - it's much safer.
- **Audit device access** regularly, especially in multi-tenant environments.

## Conclusion

Device mapping in Portainer bridges the gap between hardware and containerized applications. Whether you're building an industrial IoT gateway, a media processing server, or an embedded system controller, Docker's device mapping lets your containers interact with physical hardware while Portainer provides the configuration interface to set it up without writing Docker run commands.
