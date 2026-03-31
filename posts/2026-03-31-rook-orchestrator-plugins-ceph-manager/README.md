# How to Write Orchestrator Plugins for Ceph Manager

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Ceph, Rook, Orchestrator, Plugin Development, Python

Description: Learn how to write a custom Orchestrator plugin for Ceph Manager to implement daemon lifecycle management for a custom deployment backend.

---

The Ceph Manager Orchestrator interface defines a standard API for deploying and managing Ceph daemons. You can implement this interface as a custom plugin to integrate Ceph management with your own infrastructure automation platform.

## Orchestrator Interface Overview

The Orchestrator interface is defined in `mgr/orchestrator/_interface.py`. Key methods to implement:

- `get_hosts()` - List available hosts
- `get_daemons()` - List deployed daemons
- `apply_*()` - Deploy or scale service types (OSD, MON, MDS, etc.)
- `remove_daemons()` - Remove specific daemons
- `add_host()` / `remove_host()` - Manage host inventory

## Basic Plugin Structure

```python
from mgr_module import MgrModule
from orchestrator import (
    Orchestrator,
    OrchestratorError,
    DaemonDescription,
    HostSpec,
    OrchResult
)
from typing import List

class Module(MgrModule, Orchestrator):
    MODULE_OPTIONS = []

    def get_hosts(self) -> OrchResult[List[HostSpec]]:
        hosts = self._fetch_hosts_from_backend()
        return OrchResult([
            HostSpec(hostname=h["name"], addr=h["ip"])
            for h in hosts
        ])

    def get_daemons(self) -> OrchResult[List[DaemonDescription]]:
        daemons = self._fetch_daemons_from_backend()
        return OrchResult([
            DaemonDescription(
                daemon_type=d["type"],
                daemon_id=d["id"],
                hostname=d["host"],
                status=1  # running
            )
            for d in daemons
        ])

    def _fetch_hosts_from_backend(self):
        # Implement backend-specific host discovery
        return []

    def _fetch_daemons_from_backend(self):
        # Implement backend-specific daemon discovery
        return []
```

## Implementing Apply Methods

The `apply_*` methods handle service deployment requests:

```python
from orchestrator import ServiceSpec, OSDSpec

class Module(MgrModule, Orchestrator):
    def apply_osd(self, spec: OSDSpec) -> OrchResult[str]:
        # Translate spec into backend deployment action
        for host in spec.placement.hosts:
            self._deploy_osd_on_host(host)
        return OrchResult(f"Deployed OSDs per spec")

    def _deploy_osd_on_host(self, hostname: str):
        # Call backend API to deploy OSD
        self.log.info(f"Deploying OSD on {hostname}")
```

## Registering the Backend

Set your plugin as the active orchestrator:

```bash
ceph mgr module enable my_orchestrator
ceph orch set backend my_orchestrator
```

Verify:

```bash
ceph orch status
```

## Handling Async Operations

Many orchestrator operations are asynchronous. Use Completion objects for long-running tasks:

```python
from orchestrator import Completion

class Module(MgrModule, Orchestrator):
    def apply_mon(self, spec: ServiceSpec) -> OrchResult[str]:
        # Start async operation
        completion = self._start_mon_deployment(spec)
        return OrchResult(completion)
```

## Testing Your Plugin

Use the orchestrator test fixtures from the Ceph test suite:

```bash
cd src/pybind/mgr/orchestrator
python3 -m pytest tests/ -v
```

Test your plugin against the interface contract:

```bash
ceph orch host ls
ceph orch ps
ceph orch ls
```

## Summary

Writing a Ceph Manager Orchestrator plugin involves implementing the `Orchestrator` interface in a `MgrModule` subclass, with methods for host inventory, daemon listing, and service application. Once deployed and set as the active backend with `ceph orch set backend`, all standard `ceph orch` commands are delegated to your custom implementation.
