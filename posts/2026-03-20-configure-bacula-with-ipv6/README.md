# How to Configure Bacula with IPv6

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Bacula, IPv6, Backup, Disaster Recovery, Linux, Enterprise Backup

Description: Configure Bacula backup system to use IPv6 for backup director, storage daemon, and file daemon communications across IPv6 networks.

---

Bacula is an enterprise-grade backup solution consisting of three daemons: the Director (BCDir), Storage Daemon (BSD), and File Daemon (BFD). All three can be configured to communicate over IPv6.

## Bacula IPv6 Architecture

```
Director (bcdir)       Storage Daemon (bsd)     File Daemon (bfd)
[2001:db8::1]:9101 ←→ [2001:db8::2]:9103   ←→ [2001:db8::3]:9102
  Orchestrates           Stores data            Client to backup
```

## Configuring the Bacula Director for IPv6

```bash
# /etc/bacula/bacula-dir.conf

Director {
  Name = backup-dir
  # Listen on IPv6 address
  DirAddress = 2001:db8::1
  DIRport = 9101
  QueryFile = "/etc/bacula/scripts/query.sql"
  WorkingDirectory = /var/spool/bacula
  PidDirectory = /var/run/bacula
  Maximum Concurrent Jobs = 20
  Password = "DirectorPassword"
}

# Storage daemon connection
Storage {
  Name = backup-sd
  # Storage daemon IPv6 address
  Address = 2001:db8::2
  SDPort = 9103
  Password = "StoragePassword"
  Device = FileStorage
  Media Type = File
}

# Client (file daemon) connection
Client {
  Name = webserver-fd
  # File daemon IPv6 address
  Address = 2001:db8::10
  FDPort = 9102
  Catalog = MyCatalog
  Password = "ClientPassword"
  File Retention = 60 days
  Job Retention = 6 months
}
```

## Configuring the Bacula Storage Daemon for IPv6

```bash
# /etc/bacula/bacula-sd.conf

Storage {
  Name = backup-sd
  # Listen on IPv6 address
  SDAddress = 2001:db8::2
  SDPort = 9103
  SDAddresses {
    ip6 = { addr = 2001:db8::2; port = 9103; }
    # Also listen on IPv4 if needed:
    # ip = { addr = 192.168.1.2; port = 9103; }
  }
  WorkingDirectory = /var/spool/bacula
  PidDirectory = /var/run/bacula
  Maximum Concurrent Jobs = 20
}

Device {
  Name = FileStorage
  Device Type = File
  Archive Device = /backup/bacula
  LabelMedia = yes
  Random Access = Yes
  AutomaticMount = yes
  RemovableMedia = no
  AlwaysOpen = no
}
```

## Configuring the Bacula File Daemon (Client) for IPv6

```bash
# /etc/bacula/bacula-fd.conf (on each client)

FileDaemon {
  Name = webserver-fd
  # Listen on the client's IPv6 address
  FDAddress = 2001:db8::10
  FDPort = 9102
  FDAddresses {
    ip6 = { addr = 2001:db8::10; port = 9102; }
  }
  WorkingDirectory = /var/spool/bacula
  PidDirectory = /var/run/bacula
}

# Allow the Director to connect
Director {
  Name = backup-dir
  Password = "ClientPassword"
}
```

## Defining Jobs for IPv6 Clients

```bash
# /etc/bacula/bacula-dir.conf (job section)

JobDefs {
  Name = "DefaultIPv6Job"
  Type = Backup
  Level = Incremental
  FileSet = "FullSet"
  Schedule = "WeeklyCycle"
  Storage = backup-sd
  Messages = Standard
  SpoolAttributes = yes
  Priority = 10
  Write Bootstrap = "/var/spool/bacula/%c.bsr"
}

Job {
  Name = "WebServerBackup"
  JobDefs = "DefaultIPv6Job"
  Client = webserver-fd  # Connects over IPv6 to 2001:db8::10
  Pool = Weekly
}
```

## Firewall Rules for Bacula over IPv6

```bash
# Open Bacula daemon ports for IPv6
# Director
sudo ip6tables -A INPUT -p tcp --dport 9101 -j ACCEPT

# File daemon (on clients)
sudo ip6tables -A INPUT -p tcp --dport 9102 -j ACCEPT

# Storage daemon
sudo ip6tables -A INPUT -p tcp --dport 9103 -j ACCEPT

# Save rules
sudo ip6tables-save > /etc/ip6tables/rules.v6
```

## Testing Bacula IPv6 Connectivity

```bash
# Test connection to Storage Daemon
echo quit | bacula-sd -t -c /etc/bacula/bacula-sd.conf

# Test Director configuration
sudo bacula-dir -t -c /etc/bacula/bacula-dir.conf

# From bconsole, test client connectivity
echo "status client=webserver-fd" | bconsole

# Run a test backup
echo "run job=WebServerBackup level=Full" | bconsole
echo "messages" | bconsole
```

Bacula's configurable address binding in each daemon component enables full IPv6 communication between the Director, Storage Daemon, and File Daemons for enterprise backup operations on IPv6 networks.
