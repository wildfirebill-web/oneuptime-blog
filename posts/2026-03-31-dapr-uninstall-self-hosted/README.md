# How to Uninstall Dapr from Self-Hosted Mode

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Self-Hosted, Uninstall, Docker, Cleanup

Description: Learn how to cleanly uninstall Dapr from a self-hosted environment, removing containers, binaries, and configuration files completely.

---

## What Gets Installed in Self-Hosted Mode

When you run `dapr init`, Dapr installs the following in self-hosted mode:

- Dapr placement service container
- Zipkin container (for tracing)
- Redis container (for state and pub/sub)
- Dapr binaries in `~/.dapr/bin/`
- Default component configurations in `~/.dapr/components/`

## Basic Uninstall

To remove Dapr from self-hosted mode, run:

```bash
dapr uninstall
```

This removes the Docker containers and the Dapr binaries. The output looks like:

```bash
Removing Dapr from your machine...
Removing container: dapr_zipkin
Removing container: dapr_redis
Removing container: dapr_placement
Removing Dapr from path
Dapr has been removed successfully
```

## Removing All Configuration Files

The basic uninstall does not remove configuration files. To also delete the `~/.dapr` directory:

```bash
dapr uninstall --all
```

This removes:
- `~/.dapr/bin/` - runtime binaries
- `~/.dapr/components/` - default components
- `~/.dapr/config.yaml` - default configuration

## Slim Mode Uninstall

If you initialized Dapr without Docker using `dapr init --slim`, uninstall accordingly:

```bash
dapr uninstall --slim
```

This removes only the runtime binaries without trying to stop containers.

## Manual Cleanup

If the uninstall command fails, clean up manually:

```bash
# Stop and remove Dapr containers
docker stop dapr_placement dapr_redis dapr_zipkin
docker rm dapr_placement dapr_redis dapr_zipkin

# Remove Dapr directory
rm -rf ~/.dapr

# Remove CLI binary (macOS/Linux)
sudo rm /usr/local/bin/dapr
```

On Windows:

```powershell
Remove-Item -Recurse -Force "$env:USERPROFILE\.dapr"
```

## Verifying Removal

```bash
# Confirm containers are gone
docker ps -a | grep dapr

# Confirm binary is removed
which dapr
```

## Summary

Uninstall Dapr from self-hosted mode using `dapr uninstall` to remove containers and binaries, or `dapr uninstall --all` to also delete the `~/.dapr` configuration directory. For slim installations without Docker, use the `--slim` flag.
