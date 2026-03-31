# How to Set Up MySQL High Availability with MHA

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, MHA, High Availability, Replication, Failover

Description: Set up MySQL Master High Availability (MHA) to automate primary failover and minimize downtime when the MySQL primary server becomes unavailable.

---

## What Is MHA

MHA (Master High Availability Manager) is an open-source tool that automates MySQL primary failover. When the primary fails, MHA identifies the most up-to-date replica, applies any unapplied relay logs to bring all replicas to the same point, promotes the best replica to primary, and reconfigures the remaining replicas to replicate from the new primary - typically completing in 10-30 seconds.

## Prerequisites

MHA requires:
- One MySQL primary and at least two replicas
- SSH key-based access from the MHA manager node to all MySQL servers
- Semi-synchronous replication recommended to minimize data loss

## Installing MHA

Install the MHA node package on every MySQL server:

```bash
sudo apt-get install -y libdbd-mysql-perl libdbi-perl
wget https://github.com/yoshinorim/mha4mysql-node/releases/download/v0.58/mha4mysql-node_0.58-0_all.deb
sudo dpkg -i mha4mysql-node_0.58-0_all.deb
```

Install the manager package only on the MHA manager node:

```bash
wget https://github.com/yoshinorim/mha4mysql-node/releases/download/v0.58/mha4mysql-manager_0.58-0_all.deb
sudo dpkg -i mha4mysql-manager_0.58-0_all.deb
```

## MHA Configuration File

Create `/etc/mha/app1.cnf` on the manager node:

```text
[server default]
user=mha_user
password=mha_secret
ssh_user=ubuntu
manager_workdir=/var/log/mha/app1
manager_log=/var/log/mha/app1/manager.log
remote_workdir=/tmp/mha
secondary_check_script=masterha_secondary_check -s replica2.db.local -s replica3.db.local

[server1]
hostname=primary.db.local
candidate_master=1

[server2]
hostname=replica1.db.local
candidate_master=1

[server3]
hostname=replica2.db.local
no_master=1
```

Create the MHA user on MySQL:

```sql
CREATE USER 'mha_user'@'%' IDENTIFIED BY 'mha_secret';
GRANT ALL PRIVILEGES ON *.* TO 'mha_user'@'%' WITH GRANT OPTION;
FLUSH PRIVILEGES;
```

## Verifying SSH and Replication Connectivity

```bash
masterha_check_ssh --conf=/etc/mha/app1.cnf
masterha_check_repl --conf=/etc/mha/app1.cnf
```

Both commands should exit cleanly with "OK" before proceeding.

## Starting MHA Manager

```bash
nohup masterha_manager \
  --conf=/etc/mha/app1.cnf \
  --remove_dead_master_conf \
  --ignore_last_failover \
  > /var/log/mha/app1/manager.log 2>&1 &
```

MHA manager runs in the foreground and exits after completing a failover. Use a process supervisor like `systemd` or `supervisord` to restart it automatically.

## Handling Virtual IP During Failover

MHA can run a custom script to move a virtual IP from the failed primary to the new one. Create `/usr/local/bin/master_ip_failover` (a Perl script) and reference it in `app1.cnf`:

```text
master_ip_failover_script=/usr/local/bin/master_ip_failover
```

The script receives `--command=start/stop/stopssh` arguments and is responsible for moving the virtual IP using `ip addr` or ARP broadcasting.

## Summary

MHA automates MySQL primary failover by applying missing relay logs across replicas, selecting the best candidate, and reconfiguring the replication topology in under 30 seconds. It requires SSH access between nodes and a dedicated manager process. Combine MHA with a virtual IP failover script so applications transparently connect to the new primary without connection string changes.
