# How to Set Up Samba Guest Shares Accessible Over IPv4

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Samba, Guest Share, IPv4, SMB, Linux, File Sharing, Anonymous Access

Description: Learn how to configure a Samba guest share that allows unauthenticated IPv4 clients to access shared files without a username or password.

---

Guest shares allow any user on the network to connect without credentials. They're appropriate for public drop-off directories or read-only content distribution on trusted IPv4 networks.

## Server Configuration

```ini
# /etc/samba/smb.conf

[global]
    workgroup = MYORG
    server string = File Server

    # Map unknown users (guests) to the 'nobody' unix user
    map to guest = bad user
    # Options:
    # Never        - never map to guest
    # Bad User     - map if authentication fails (user not found)
    # Bad Password - map if password is wrong (less secure)

    # Restrict to internal IPv4 only
    interfaces = 192.168.1.10/24 lo
    bind interfaces only = yes

#-----------------------------------------------
# Read-only guest share

#-----------------------------------------------
[public]
    comment = Public Read-Only Share
    path = /srv/samba/public

    # Allow unauthenticated (guest) access
    guest ok = yes

    # Read-only
    read only = yes
    write list =       # No one has write access

    # Restrict to the internal IPv4 subnet
    hosts allow = 192.168.1.0/24
    hosts deny = ALL

    browseable = yes

#-----------------------------------------------
# Read-write guest share (drop-box)
#-----------------------------------------------
[dropbox]
    comment = Upload Drop Box
    path = /srv/samba/dropbox

    guest ok = yes
    writable = yes

    # Guests can upload but not read other users' files
    create mask = 0600
    directory mask = 0700

    # Force all operations to run as the 'samba-guest' user
    force user = samba-guest
    force group = samba-guest

    hosts allow = 192.168.1.0/24
    hosts deny = ALL
```

## Setting Up the Guest Account and Directories

```bash
# Create a restricted user for guest operations
useradd -M -s /usr/sbin/nologin -c "Samba Guest" samba-guest

# Create and set permissions on share directories
mkdir -p /srv/samba/public /srv/samba/dropbox

# Public: readable by everyone
chown root:samba-guest /srv/samba/public
chmod 755 /srv/samba/public

# Dropbox: writable by samba-guest, not readable by others
chown samba-guest:samba-guest /srv/samba/dropbox
chmod 1733 /srv/samba/dropbox   # sticky bit: can write, can't delete others' files

# Verify nobody is the guest fallback user
getent passwd nobody
```

## Applying the Configuration

```bash
# Validate configuration
testparm

# Restart Samba
systemctl restart smb nmb
```

## Testing Guest Access

```bash
# Connect without credentials (anonymous guest)
smbclient //192.168.1.10/public

# Or explicitly as guest
smbclient //192.168.1.10/public -N   # -N = no password

# Mount the guest share on a Linux client
mount -t cifs //192.168.1.10/public /mnt/public -o guest

# Access from Windows
# Map a network drive to \\192.168.1.10\public without credentials
```

## Key Takeaways

- `map to guest = bad user` allows clients who fail authentication to connect as guest.
- `guest ok = yes` in a share definition enables anonymous access.
- Always combine with `hosts allow` to restrict guest access to trusted IPv4 ranges.
- Use `force user` and `force group` with a dedicated low-privilege account to isolate guest file operations.
