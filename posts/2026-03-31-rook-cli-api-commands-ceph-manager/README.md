# How to Use CLI API Commands Module in Ceph Manager

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Ceph, Rook, CLI, API, Module Development

Description: Learn how to use the Ceph Manager CLI API commands module to expose custom REST endpoints and CLI commands from manager module code.

---

Ceph Manager modules can expose both CLI commands and REST API endpoints through the built-in command framework. Understanding this infrastructure is essential for building modules that operators can interact with via `ceph` CLI or HTTP.

## CLI Commands in Manager Modules

Every manager module can define CLI commands through the `COMMANDS` list. Each command entry uses a Ceph command description string syntax:

```python
from mgr_module import MgrModule

class Module(MgrModule):
    COMMANDS = [
        {
            "cmd": "mymodule stats name=pool,type=CephString,req=false",
            "desc": "Show stats, optionally filtered by pool name",
            "perm": "r"
        },
        {
            "cmd": "mymodule reset",
            "desc": "Reset module state",
            "perm": "rw"
        }
    ]

    def handle_command(self, inbuf, cmd):
        if cmd["prefix"] == "mymodule stats":
            pool = cmd.get("pool", "all")
            return 0, f"Stats for pool: {pool}", ""
        elif cmd["prefix"] == "mymodule reset":
            return 0, "Reset complete", ""
        return -1, "", "Unknown command"
```

## Command Syntax Reference

Command parameter types used in the `cmd` string:

| Type           | Description                 |
|----------------|-----------------------------|
| `CephString`   | Arbitrary string value      |
| `CephInt`      | Integer value               |
| `CephFloat`    | Floating-point value        |
| `CephBool`     | Boolean flag                |
| `CephPoolname` | Validated pool name         |

## REST API Endpoints

Modules can also serve HTTP endpoints via the built-in manager web framework:

```python
from mgr_module import MgrModule, CLIReadCommand
import cherrypy

class Module(MgrModule):
    def serve(self):
        cherrypy.tree.mount(self._api, "/api/mymodule", {})
        cherrypy.engine.start()
        self.event.wait()

    class _api:
        @cherrypy.expose
        @cherrypy.tools.json_out()
        def status(self):
            return {"status": "ok", "version": "1.0"}
```

## Using the REST API

The manager's built-in REST gateway listens on port 8003 by default:

```bash
# Get the manager API address
ceph mgr services

# Query a module endpoint
curl -k https://192.168.1.10:8443/api/mymodule/status
```

## Using Decorators for Commands

Modern Ceph modules support decorator-based command registration:

```python
from mgr_module import MgrModule, CLIReadCommand

class Module(MgrModule):
    @CLIReadCommand("mymodule info")
    def cmd_info(self, format: str = "plain") -> tuple:
        return 0, "Module info here", ""
```

## Testing Module Commands

Test your commands without reloading the module by calling the Ceph CLI:

```bash
ceph mymodule stats
ceph mymodule stats --pool production
ceph mymodule reset
```

## Summary

Ceph Manager modules expose functionality through two mechanisms: CLI commands declared in the `COMMANDS` list and processed in `handle_command`, and HTTP REST endpoints served via CherryPy. Both mechanisms integrate into the existing `ceph` CLI and manager web server, making it straightforward to build both operator tools and programmatic integrations from a single module.
