# How to Use the dapr completion Command for Shell Autocompletion

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, CLI, Shell, Autocompletion, Developer Experience

Description: Learn how to use the dapr completion command to enable tab completion for the Dapr CLI in bash, zsh, fish, and PowerShell.

---

## Overview

The `dapr completion` command generates shell completion scripts for the Dapr CLI. Once installed, pressing Tab while typing a `dapr` command completes subcommands, flags, and arguments automatically. This saves time and reduces typos during development.

## Supported Shells

`dapr completion` supports:
- `bash`
- `zsh`
- `fish`
- `powershell`

## Setting Up bash Completion

Generate and install the completion script for bash:

```bash
# Add to current session
source <(dapr completion bash)

# Persist across sessions
dapr completion bash > /etc/bash_completion.d/dapr
```

On macOS with Homebrew bash:

```bash
dapr completion bash > $(brew --prefix)/etc/bash_completion.d/dapr
```

## Setting Up zsh Completion

```bash
# Add to ~/.zshrc
echo 'source <(dapr completion zsh)' >> ~/.zshrc

# Or install to zsh completion directory
dapr completion zsh > "${fpath[1]}/_dapr"
```

Enable completions in `~/.zshrc` if not already done:

```bash
autoload -U compinit
compinit
```

Reload your shell:

```bash
source ~/.zshrc
```

## Setting Up fish Completion

```bash
dapr completion fish > ~/.config/fish/completions/dapr.fish
```

Reload:

```bash
source ~/.config/fish/config.fish
```

## Setting Up PowerShell Completion

```powershell
dapr completion powershell | Out-String | Invoke-Expression

# Persist across sessions - add to your profile
dapr completion powershell >> $PROFILE
```

## Testing That Completion Works

After installation, type `dapr ` and press Tab:

```text
dapr <TAB>
annotate    build-info  completion  components  configurations
dashboard   help        init        invoke      list
logs        mtls        publish     run         scheduler
status      stop        uninstall   upgrade     version
workflow
```

Type `dapr run --` and press Tab to see flag completions:

```text
--app-id             --app-port           --app-protocol
--dapr-http-port     --dapr-grpc-port     --enable-api-logging
--log-level          --resources-path
```

## Using completion in a Docker Container

If you run the Dapr CLI in a container, add completion to the image:

```dockerfile
FROM ubuntu:22.04
RUN curl -fsSL https://raw.githubusercontent.com/dapr/cli/master/install/install.sh | bash
RUN dapr completion bash > /etc/bash_completion.d/dapr
RUN echo 'source /etc/bash_completion.d/dapr' >> /root/.bashrc
```

## Summary

`dapr completion` makes working with the Dapr CLI significantly faster by enabling tab completion for all commands and flags. Install it once for your shell and it persists across sessions. This is a small setup step with a large productivity payoff, especially when exploring unfamiliar subcommands and their available options.
