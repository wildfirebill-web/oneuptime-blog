# How to Use nmtui for Text-Based Network Configuration

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Linux, nmtui, NetworkManager, TUI, Networking, Configuration

Description: Use nmtui (NetworkManager Text User Interface) to configure network connections interactively in a terminal, useful for servers without a graphical interface.

## Introduction

`nmtui` is a curses-based text user interface for NetworkManager. It provides a menu-driven approach to network configuration without requiring knowledge of `nmcli` syntax - useful for initial server setup, remote SSH sessions, or when a GUI is unavailable.

## Launch nmtui

```bash
# Start the nmtui text interface (requires root/sudo)

nmtui

# Or explicitly call it with sudo
sudo nmtui
```

## Main Menu Options

When launched, nmtui presents three options:

```sql
┌─────────────────────────────────┐
│ NetworkManager TUI              │
│                                 │
│ Please select an option         │
│                                 │
│ Edit a connection               │
│ Activate a connection           │
│ Set system hostname             │
│                                 │
│            <OK>  <Quit>         │
└─────────────────────────────────┘
```

## Edit a Connection

Navigate to **Edit a connection** to:
- Create a new connection profile
- Modify an existing connection's IP, DNS, gateway
- Delete a connection

Navigation:
- `Tab` - move between fields
- `Space` / `Enter` - select/confirm
- `Esc` - go back

## Activate a Connection

Navigate to **Activate a connection** to:
- Enable or disable connections
- See which connection is currently active on each interface
- An asterisk `*` indicates the active connection

## Set System Hostname

Navigate to **Set system hostname** to change the machine's hostname persistently.

## Configure a Static IP in nmtui

1. Select **Edit a connection**
2. Choose the connection to edit (e.g., `Wired connection 1`)
3. Change **IPv4 CONFIGURATION** from `<Automatic>` to `<Manual>`
4. Add IP address: `192.168.1.50/24`
5. Add gateway: `192.168.1.1`
6. Add DNS server: `8.8.8.8`
7. Select `<OK>` to save

## Activate Changes After Editing

After saving in nmtui, reactivate the connection:

```bash
# The connection needs to be reactivated to apply changes
nmcli connection up "Wired connection 1"

# Or use nmtui's Activate a connection menu to do it interactively
```

## Install nmtui (if not present)

```bash
# Debian/Ubuntu
apt install network-manager

# RHEL/CentOS/AlmaLinux
dnf install NetworkManager-tui

# Verify installation
which nmtui
```

## nmtui vs nmcli

| Feature | nmtui | nmcli |
|---|---|---|
| Interface | Interactive TUI | Command line |
| Learning curve | Low | Medium |
| Scriptable | No | Yes |
| All NM features | No | Yes |
| Best for | Interactive setup | Automation/scripting |

## Conclusion

`nmtui` provides an accessible text-based interface for NetworkManager configuration. It covers the most common use cases: editing connection profiles, activating connections, and setting the hostname. For scripting and advanced configuration, use `nmcli`. After making changes in nmtui, reactivate the connection with `nmcli connection up`.
