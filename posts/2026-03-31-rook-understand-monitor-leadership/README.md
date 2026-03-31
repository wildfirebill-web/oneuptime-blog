# How to Understand Monitor Leadership in Ceph

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Monitor, Paxos, Architecture

Description: Learn how Ceph monitor leadership works using the Paxos algorithm, how to identify the current leader, and what happens during leader elections.

---

## The Paxos Algorithm and Monitor Leadership

Ceph monitors implement a distributed consensus system based on the Paxos algorithm. Among all active monitors in quorum, one is elected as the "leader" and the others are "peons". The leader is responsible for:

- Proposing and committing changes to the cluster map
- Coordinating OSD map updates when OSDs join or leave
- Coordinating placement group state changes
- Periodically synchronizing state to peon monitors

Peons replicate all state from the leader but do not independently initiate map changes. Any monitor can answer read queries for clients, but writes always route through the leader.

## Identifying the Current Leader

Check which monitor is currently the leader:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph mon stat
```

Look for the `leader` field in the output:

```text
e3: 3 mons at {...}, election epoch 12, leader 0 a, quorum 0,1,2 a,b,c
```

Monitor `a` (rank 0) is currently leading.

## Checking Leader via quorum_status

Get detailed quorum information including the leader:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph quorum_status --format json | python3 -c \
  "import sys,json; d=json.load(sys.stdin); print('Leader:', d['quorum_leader_name'])"
```

## How Leader Elections Work

When the current leader becomes unavailable, the remaining monitors elect a new leader. The election process:

1. Any peon that does not hear from the leader within `mon_lease` seconds (default: 5s) starts an election
2. The monitor with the lowest rank (rank 0, 1, 2...) typically wins
3. The new leader proposes a new epoch and peons acknowledge
4. Once a majority acknowledge, the new leader is established

Elections typically complete in under a second on a healthy network.

## Monitoring Election Events

Election events are logged in monitor pod logs:

```bash
kubectl -n rook-ceph logs rook-ceph-mon-a-<suffix> | grep -i "election\|leader\|peon\|quorum"
```

You will see messages like:

```text
mon.a@0(leader).paxos(active) e123
starting new election
win election
```

## Influence on Leader Selection

Monitor rank determines election preference. Monitor `a` (rank 0) is always preferred as leader. If you want to avoid a specific monitor becoming leader (e.g., due to lower resources), you can adjust the election strategy:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph mon set election_strategy connectivity
```

The `connectivity` strategy elects based on network reachability rather than rank.

## Impact of Leader Failure

If the leader fails, clients experience a brief pause (typically under 5 seconds) while the remaining monitors elect a new leader. During this time, map update requests queue. Once a new leader is elected, operations resume automatically.

Check the election epoch to see how many elections have occurred:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph mon dump | grep election
```

Frequent elections indicate monitor instability that should be investigated.

## Summary

Ceph monitor leadership is managed via Paxos consensus. The leader coordinates all cluster map changes while peons replicate state. Use `ceph mon stat` to identify the current leader and `ceph quorum_status` for full quorum details. The `connectivity` election strategy improves leader selection in complex network topologies.
