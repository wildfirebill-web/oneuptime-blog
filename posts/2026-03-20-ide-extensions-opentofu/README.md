# How to Configure IDE Extensions for OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, IDE, VS Code, JetBrains, Developer Experience

Description: Learn how to configure IDE extensions and plugins for OpenTofu development in VS Code, JetBrains IDEs, Neovim, and other editors to get syntax highlighting, auto-completion, and inline validation.

## Introduction

IDE extensions for OpenTofu provide syntax highlighting, auto-completion, inline error detection, and formatting integration. Since OpenTofu uses the same HCL syntax as Terraform, most Terraform extensions work directly with OpenTofu files. This guide covers setup for the most common development environments.

## VS Code Setup

```json
// .vscode/extensions.json - recommend to team
{
  "recommendations": [
    "hashicorp.terraform",
    "editorconfig.editorconfig",
    "ms-azuretools.vscode-docker",
    "redhat.vscode-yaml",
    "github.vscode-github-actions"
  ]
}
```

Install the HashiCorp Terraform extension (works with OpenTofu):

```bash
code --install-extension hashicorp.terraform
```

```json
// .vscode/settings.json
{
  "editor.formatOnSave": true,
  "editor.rulers": [120],
  "[terraform]": {
    "editor.defaultFormatter": "hashicorp.terraform",
    "editor.formatOnSave": true
  },
  "terraform.languageServer.enable": true,
  "terraform.languageServer.path": "/usr/local/bin/tofu",
  "files.associations": {
    "*.tf": "terraform",
    "*.tfvars": "terraform",
    "*.hcl": "hcl"
  },
  "terraform.experimentalFeatures.prefillRequiredFields": true,
  "terraform.experimentalFeatures.validateOnSave": true
}
```

## Configuring the Language Server for OpenTofu

The HashiCorp extension uses terraform-ls by default. Point it to the OpenTofu binary for better compatibility:

```json
// .vscode/settings.json
{
  "terraform.languageServer.enable": true,
  "terraform.languageServer.path": "/usr/local/bin/terraform-ls",
  "terraform.languageServer.args": ["serve"],
  // Alternatively, use tofu as the backend
  "terraform.languageServer.terraform.path": "/usr/local/bin/tofu"
}
```

Install terraform-ls separately for the best experience:

```bash
# macOS
brew install hashicorp/tap/terraform-ls

# Linux
curl -fsSL https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor > /usr/share/keyrings/hashicorp-archive-keyring.gpg
sudo apt-get install terraform-ls
```

## JetBrains IDEs (IntelliJ, GoLand, PyCharm)

Install the HashiCorp Terraform/HCL plugin:

1. Settings → Plugins → Marketplace → Search "HashiCorp Terraform and HCL"
2. Install and restart

Configure the plugin to use OpenTofu:

```
Settings → Languages & Frameworks → Terraform and HCL
  → Terraform binary path: /usr/local/bin/tofu
  → Enable formatting on save: ✓
  → Enable validation: ✓
```

```ini
# .editorconfig for JetBrains-specific settings
[*.tf]
ij_terraform_hcl_space_before_assign_operator = true
ij_terraform_hcl_space_after_assign_operator = true
ij_terraform_hcl_indent_before_blocks = true
ij_terraform_hcl_keep_blank_lines_in_code = 1
```

## Neovim Setup (LSP)

```lua
-- ~/.config/nvim/lua/lsp.lua
-- Install terraform-ls first: brew install hashicorp/tap/terraform-ls

require('lspconfig').terraformls.setup({
  cmd = { "terraform-ls", "serve" },
  filetypes = { "terraform", "terraform-vars", "hcl" },
  root_dir = require('lspconfig.util').root_pattern(".terraform", ".git"),
  settings = {
    ["terraform-ls"] = {
      terraformExecPath = "/usr/local/bin/tofu"
    }
  }
})

-- Format on save
vim.api.nvim_create_autocmd("BufWritePre", {
  pattern = { "*.tf", "*.tfvars" },
  callback = function()
    vim.lsp.buf.format({ async = false })
  end,
})
```

```lua
-- Install treesitter parser for syntax highlighting
-- In your treesitter config:
require('nvim-treesitter.configs').setup({
  ensure_installed = { "hcl", "terraform" },
  highlight = { enable = true }
})
```

## Vim Setup

```vim
" ~/.vimrc or ~/.vim/plugs.vim

" Using vim-plug:
Plug 'hashivim/vim-terraform'
Plug 'vim-syntastic/syntastic'

" Configuration
let g:terraform_align = 1
let g:terraform_fmt_on_save = 1
let g:terraform_binary_path = '/usr/local/bin/tofu'

" Syntastic with tflint
let g:syntastic_terraform_checkers = ['tflint']
```

## Emacs Setup

```elisp
;; .emacs or init.el
;; Install terraform-mode from MELPA

(use-package terraform-mode
  :hook (terraform-mode . terraform-format-on-save-mode)
  :config
  (setq terraform-binary-path "/usr/local/bin/tofu"))

;; LSP with terraform-ls
(use-package lsp-mode
  :hook (terraform-mode . lsp-deferred)
  :config
  (lsp-register-client
    (make-lsp-client
      :new-connection (lsp-stdio-connection '("terraform-ls" "serve"))
      :major-modes '(terraform-mode)
      :server-id 'terraform-ls)))
```

## Workspace Recommendations File

Commit this to share extension recommendations with your team:

```json
// .vscode/extensions.json
{
  "recommendations": [
    "hashicorp.terraform",
    "editorconfig.editorconfig",
    "redhat.vscode-yaml",
    "timonwong.shellcheck",
    "ms-python.python",
    "github.vscode-pull-request-github",
    "eamodio.gitlens"
  ],
  "unwantedRecommendations": [
    "4ops.terraform"
  ]
}
```

## Conclusion

The HashiCorp Terraform extension for VS Code and the JetBrains Terraform plugin both work with OpenTofu by pointing the language server path to the `tofu` binary. The terraform-ls language server provides the most features — auto-completion, go-to-definition, hover documentation, and inline validation — across all editors that support LSP. Commit `.vscode/extensions.json` and `.vscode/settings.json` to ensure the whole team uses the same configuration.
