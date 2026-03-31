# How to Map Host Devices (USB, Serial) to Containers in Portainer - Map

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Device, USB, SERIAL, IoT, Hardware

Description: Configure Docker containers to access host hardware devices - USB peripherals, serial ports, and other device files - through Portainer's container creation interface and stack YAML.

---

Many IoT, industrial, and hardware integration workloads require containers to access physical devices on the host: USB barcode scanners, serial port adapters, GPIO interfaces, or hardware dongles. Docker provides the `--device` flag for this purpose, and Portainer's container creation form and stack YAML both support device mapping.

## How Device Mapping Works

Docker device mapping passes a host device file into the container with configurable read/write/mknod permissions. The container process can then interact with the device as if it were running natively on the host.

## Step 1: Identify the Device on the Host

Before mapping, identify the device file:

```bash
# List USB devices

ls /dev/tty* | grep -E "USB|ACM"
# Common: /dev/ttyUSB0 (USB serial adapter), /dev/ttyACM0 (Arduino)

# Check device permissions
ls -la /dev/ttyUSB0
# crw-rw---- 1 root dialout ... /dev/ttyUSB0

# Find USB video devices
ls /dev/video*

# Check device by USB ID
lsusb
dmesg | tail -20   # Shows recent device connect events
```

## Step 2: Map Device in Portainer Container Creation

In Portainer's **Add Container** form:

1. Scroll to **Runtime & Resources**
2. Find the **Devices** section
3. Add the device:
   - **Host**: `/dev/ttyUSB0`
   - **Container**: `/dev/ttyUSB0`
   - **Permissions**: `rwm` (read, write, mknod)

## Step 3: Device Mapping in Stack YAML

```yaml
version: "3.8"
services:
  serial-reader:
    image: my-iot-app:latest
    devices:
      # Map RS-485 serial adapter
      - /dev/ttyUSB0:/dev/ttyUSB0
    restart: unless-stopped

  modbus-gateway:
    image: modbus-gateway:2.0
    devices:
      # Map USB-to-serial adapter
      - /dev/ttyACM0:/dev/ttyACM0
    environment:
      - SERIAL_PORT=/dev/ttyACM0
      - BAUD_RATE=9600

  webcam-processor:
    image: opencv-processor:latest
    devices:
      # Map USB webcam
      - /dev/video0:/dev/video0
```

## Step 4: Handle Permissions

The container user may not have permission to access the device. Options:

```yaml
services:
  serial-reader:
    image: my-iot-app:latest
    devices:
      - /dev/ttyUSB0:/dev/ttyUSB0
    # Run as root to access the device
    user: root
    # OR add the user to the dialout group
    group_add:
      - dialout
```

Or change permissions on the host (less secure):

```bash
chmod a+rw /dev/ttyUSB0
```

## Step 5: Persistent Device Names with udev Rules

USB device paths can change after reboot. Create a udev rule for a stable symlink:

```bash
# /etc/udev/rules.d/99-usb-serial.rules
SUBSYSTEM=="tty", ATTRS{idVendor}=="0403", ATTRS{idProduct}=="6001", SYMLINK+="my-usb-serial"
```

Then map `/dev/my-usb-serial` in your container instead of `/dev/ttyUSB0`.

## Summary

Portainer's device mapping support makes it straightforward to connect containers to physical hardware. The stack YAML `devices` key is the cleanest way to declare device requirements as code, ensuring the mapping is reproducible across deployments. For IoT and industrial workloads, pairing device mapping with udev symlinks prevents path drift across reboots.
