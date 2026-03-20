# How to Set Up OpenTofu Autocompletion in Bash

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Bash, Autocomplete, Productivity, Infrastructure as Code, DevOps

Description: A guide to enabling tab completion for OpenTofu commands in Bash to boost productivity.

## Introduction

Shell autocompletion dramatically improves productivity when working with OpenTofu. With tab completion enabled, you can quickly complete commands, subcommands, flags, and resource types without memorizing them all. This guide covers setting up autocompletion for OpenTofu in Bash.

## Prerequisites

- OpenTofu installed
- Bash 4.0 or later
- `bash-completion` package installed

## Step 1: Install bash-completion

```bash
# Ubuntu/Debian

sudo apt-get install -y bash-completion

# Fedora/RHEL/CentOS
sudo dnf install -y bash-completion

# macOS (via Homebrew)
brew install bash-completion@2

# Verify it's installed
type _init_completion 2>/dev/null && echo "bash-completion is active"
```

## Step 2: Enable OpenTofu Autocomplete

```bash
# Install autocomplete for OpenTofu (adds to ~/.bashrc automatically)
tofu -install-autocomplete

# Reload your shell configuration
source ~/.bashrc
```

## What the Command Does

The `-install-autocomplete` flag adds the following to your `~/.bashrc`:

```bash
# This is what tofu -install-autocomplete adds:
complete -C /usr/local/bin/tofu tofu
```

## Manual Setup (if automatic installation fails)

```bash
# Find the tofu binary location
TOFU_BIN=$(which tofu)
echo $TOFU_BIN

# Manually add completion to ~/.bashrc
echo "complete -C ${TOFU_BIN} tofu" >> ~/.bashrc

# Reload
source ~/.bashrc
```

## Verifying Autocompletion Works

```bash
# Type 'tofu' and press Tab twice to see all subcommands
tofu <TAB><TAB>

# Should display:
# apply       console     destroy     fmt         graph
# import      init        output      plan        providers
# refresh     show        state       taint       untaint
# validate    version     workspace

# Autocomplete flags
tofu apply -<TAB><TAB>
# -auto-approve  -backup  -compact-warnings  -input  -lock  ...

# Autocomplete state subcommands
tofu state <TAB><TAB>
# list  mv  pull  push  replace-provider  rm  show
```

## Setting Up System-Wide Completion

```bash
# For system-wide completion (requires root)
TOFU_BIN=$(which tofu)
echo "complete -C ${TOFU_BIN} tofu" | sudo tee /etc/bash_completion.d/tofu

# Reload
source /etc/bash_completion.d/tofu
```

## Completion in .bash_profile vs .bashrc

```bash
# On interactive login shells (typically macOS), add to .bash_profile
TOFU_BIN=$(which tofu)
echo "complete -C ${TOFU_BIN} tofu" >> ~/.bash_profile
source ~/.bash_profile

# On interactive non-login shells (typically Linux), add to .bashrc
echo "complete -C ${TOFU_BIN} tofu" >> ~/.bashrc
source ~/.bashrc
```

## Testing Completion

```bash
# Test various completion scenarios:

# 1. Subcommand completion
tofu p<TAB>    # completes to: plan, providers

# 2. Flag completion
tofu plan -<TAB>   # shows available flags

# 3. Complete a partial subcommand
tofu val<TAB>  # completes to: validate

# 4. State subcommand
tofu state l<TAB>  # completes to: list
```

## Removing Autocompletion

```bash
# Remove the completion line from ~/.bashrc
sed -i '/complete -C.*tofu/d' ~/.bashrc

# Reload
source ~/.bashrc
```

## Conclusion

Setting up Bash autocompletion for OpenTofu takes less than a minute but saves significant time in daily use. The `complete` directive enables the built-in OpenTofu completion engine to provide context-aware suggestions for commands, flags, and arguments. This productivity boost is especially valuable when learning new OpenTofu subcommands and their available options.
