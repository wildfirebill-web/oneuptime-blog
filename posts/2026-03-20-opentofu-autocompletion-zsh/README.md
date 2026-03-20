# How to Set Up OpenTofu Autocompletion in Zsh

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Zsh, Shell, Autocompletion, DevOps, Productivity, Infrastructure as Code

Description: Learn how to enable tab autocompletion for OpenTofu commands in Zsh, including setup with Oh My Zsh and without it.

---

Tab autocompletion in Zsh gives you instant access to OpenTofu commands, flags, and options without memorizing them. Zsh's completion system is particularly powerful — it shows descriptions alongside completions and supports filtering. This guide covers setup with and without Oh My Zsh.

---

## Option 1: Built-In Install Command

OpenTofu's built-in autocomplete installer works for both Bash and Zsh.

```zsh
# Install OpenTofu autocompletion for Zsh
tofu -install-autocomplete

# This appends a line to ~/.zshrc similar to:
# autoload -U +X bashcompinit && bashcompinit
# complete -o nospace -C /usr/local/bin/tofu tofu
```

```zsh
# Reload your Zsh config to activate
source ~/.zshrc

# Test it
tofu <TAB>
```

---

## Option 2: Manual Zsh Completion Setup

For more control, add the completion manually to `~/.zshrc`.

```zsh
# Add these lines to ~/.zshrc

# Enable bash compatibility for OpenTofu completion
autoload -U +X bashcompinit && bashcompinit
autoload -U +X compinit && compinit

# Register OpenTofu completion
complete -o nospace -C "$(which tofu)" tofu
```

```zsh
source ~/.zshrc
```

---

## Option 3: Oh My Zsh with Terraform Plugin

Oh My Zsh's `terraform` plugin is broadly compatible with OpenTofu since they share the same CLI interface.

```zsh
# In ~/.zshrc, add terraform to your plugins list
plugins=(
  git
  docker
  terraform   # works with OpenTofu too
  kubectl
)

# Then add the OpenTofu-specific alias
echo 'alias tf=tofu' >> ~/.zshrc

source ~/.zshrc
```

---

## Option 4: Zsh Completion Function

For the best Zsh-native experience, create a custom completion function.

```zsh
# ~/.zsh/completions/_tofu — create this file
# Place it in a directory on your $fpath

#compdef tofu

# OpenTofu subcommand completion
_tofu_commands() {
  local commands
  commands=(
    'apply:Create or update infrastructure'
    'console:Try OpenTofu expressions interactively'
    'destroy:Destroy previously-created infrastructure'
    'fmt:Reformat your configuration in the standard style'
    'force-unlock:Release a stuck lock on the current workspace'
    'get:Install or upgrade remote modules'
    'graph:Generate a Graphviz graph of the steps in an operation'
    'import:Associate existing infrastructure with an OpenTofu resource'
    'init:Prepare your working directory for other commands'
    'login:Obtain and save credentials for a remote host'
    'logout:Remove locally-stored credentials for a remote host'
    'output:Show output values from your root module'
    'plan:Show changes required by the current configuration'
    'providers:Show the providers required for this configuration'
    'refresh:Update the state to match remote systems'
    'show:Show the current state or a saved plan'
    'state:Advanced state management'
    'taint:Mark a resource instance as not fully functional'
    'test:Run module tests'
    'untaint:Remove the taint from a resource instance'
    'validate:Check whether the configuration is valid'
    'version:Show the current OpenTofu version'
    'workspace:Workspace management'
  )
  _describe 'commands' commands
}

_arguments \
  '1: :_tofu_commands' \
  '*: :->args'
```

```zsh
# Add the completions directory to fpath in .zshrc
fpath=(~/.zsh/completions $fpath)
autoload -Uz compinit && compinit
```

---

## Testing Autocompletion

```zsh
# Test subcommand completion
tofu <TAB>

# Test flag completion
tofu plan -<TAB>
# Shows: -compact-warnings  -detailed-exitcode  -input  ...

# Test workspace completion
tofu workspace select <TAB>
# Shows existing workspace names
```

---

## Summary

The simplest Zsh autocompletion setup for OpenTofu is `tofu -install-autocomplete`, which adds the necessary `bashcompinit` configuration to `~/.zshrc`. For Oh My Zsh users, the existing `terraform` plugin works directly with OpenTofu. For the best Zsh-native experience, create a `_tofu` completion function in your `$fpath` directory.
