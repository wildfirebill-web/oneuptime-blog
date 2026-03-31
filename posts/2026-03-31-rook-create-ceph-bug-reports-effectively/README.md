# How to Create Ceph Bug Reports Effectively

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Ceph, Rook, Bug Report, Community, Troubleshooting

Description: Write effective Ceph bug reports by including version info, reproduction steps, diagnostic output, and log excerpts to get faster resolutions from maintainers.

---

A well-written bug report gets resolved faster than a vague one. Ceph maintainers are volunteers and paid engineers with limited time. Making their job easier means your issue gets attention sooner.

## Before You File

Check existing issues first:

- Ceph Tracker: https://tracker.ceph.com
- Rook GitHub: https://github.com/rook/rook/issues

Search for your error message or symptom. If an issue already exists, add your details as a comment with a "+1" or additional reproduction data rather than opening a duplicate.

## Essential Information to Include

Every bug report needs:

```bash
# 1. Ceph version
ceph version

# 2. OS and kernel
uname -a
cat /etc/os-release

# 3. Cluster state at time of issue
ceph status
ceph health detail

# 4. Rook version (if applicable)
kubectl get pods -n rook-ceph -l app=rook-ceph-operator -o jsonpath='{.items[0].spec.containers[0].image}'
```

## Writing a Clear Title

Good titles are specific:

- Bad: "OSD not working"
- Good: "OSD crashes with SIGSEGV after upgrading from Quincy 17.2.5 to Reef 18.2.1"

## Reproduction Steps

List steps in numbered order:

```
1. Deploy a 3-node Ceph cluster on Ubuntu 22.04 with kernel 5.15
2. Create an RBD pool with replication factor 3
3. Mount the RBD image on a client
4. Write 10 GB of data using fio
5. Observe OSD 2 crashes with backtrace in /var/log/ceph/ceph-osd.2.log
```

## Log Excerpts

Include the relevant portion of the log, not hundreds of lines:

```bash
# Extract the crash from the OSD log
grep -A 30 "Assertion\|SIGSEGV\|backtrace" /var/log/ceph/ceph-osd.2.log | tail -40
```

Paste the output in a code block in the issue.

## Configuration Details

Include relevant config settings:

```bash
ceph config dump
ceph osd metadata 2 | python3 -m json.tool
```

## What Not to Do

- Do not share passwords or private keys
- Do not paste thousands of lines of raw logs - attach them or use a paste service
- Do not open the issue as "URGENT" unless data loss is occurring

## Summary

Effective Ceph bug reports include version details, reproduction steps, relevant log excerpts, and cluster state output. Searching for duplicates before filing prevents wasted effort on all sides. Clear, specific titles and structured reproduction steps are the single biggest factor in how quickly a report gets triaged and resolved.
