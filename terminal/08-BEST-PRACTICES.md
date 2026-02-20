# Terminal Workflow Best Practices

A comprehensive guide to terminal workflow best practices — covering dotfile management with chezmoi, GNU Stow, and bare Git repos, portable configurations across platforms, backup strategies, version control for configs, ergonomic key bindings, multiplexer workflows, efficient navigation, and clipboard integration.

---

## Table of Contents

1. [Overview](#overview)
2. [Dotfile Management](#dotfile-management)
3. [Portable Configurations](#portable-configurations)
4. [Backup Strategies](#backup-strategies)
5. [Version Control for Configs](#version-control-for-configs)
6. [Ergonomic Key Bindings](#ergonomic-key-bindings)
7. [Terminal Multiplexer Workflows](#terminal-multiplexer-workflows)
8. [Efficient Navigation Patterns](#efficient-navigation-patterns)
9. [Clipboard Integration](#clipboard-integration)
10. [Configuration Organisation](#configuration-organisation)
11. [Quick Reference Checklist](#quick-reference-checklist)
12. [Next Steps](#next-steps)
13. [Version History](#version-history)

---

## Overview

Terminal workflows compound — small inefficiencies in your configuration, navigation, or key bindings multiply across thousands of daily interactions. This guide codifies the practices that experienced terminal users follow to maintain fast, portable, and reliable terminal environments.

### Target Audience

- **Any terminal user** who wants to optimise their workflow
- **Engineers maintaining configurations** across multiple machines or platforms
- **Teams** standardising terminal setups for consistent developer experience

### Scope

- Dotfile management strategies (chezmoi, GNU Stow, bare Git repo)
- Cross-platform configuration portability
- Backup and version control for terminal configurations
- Ergonomic key bindings and navigation patterns
- Clipboard integration across platforms and tools

---

## Dotfile Management

Dotfiles are the configuration files that define your terminal environment (`.zshrc`, `.tmux.conf`, `nvim/init.lua`, etc.). Managing them properly ensures you can reproduce your environment on any machine.

### Strategy Comparison

| Strategy | Complexity | Templates | Secrets | Multi-Machine | Best For |
|----------|-----------|-----------|---------|---------------|----------|
| **chezmoi** | Medium | ✅ | ✅ (encrypted) | ✅ | Cross-platform, multiple machines |
| **GNU Stow** | Low | ❌ | ❌ | ❌ | Linux, simple symlink management |
| **Bare Git Repo** | Low | ❌ | ❌ | ❌ | Minimal deps, Git-only |
| **Nix Home Manager** | High | ✅ | ✅ | ✅ | Declarative, reproducible |
| **Ansible** | High | ✅ | ✅ (vault) | ✅ | Many machines, automation |

### chezmoi (Recommended)

**chezmoi** is a purpose-built dotfile manager that handles templates, secrets, and cross-platform differences.

```bash
# Install
brew install chezmoi        # macOS
sudo apt install chezmoi    # Debian/Ubuntu
sh -c "$(curl -fsLS get.chezmoi.io)"  # Universal

# Initialise (creates ~/.local/share/chezmoi/)
chezmoi init

# Add existing dotfiles
chezmoi add ~/.zshrc
chezmoi add ~/.tmux.conf
chezmoi add ~/.config/nvim/init.lua
chezmoi add ~/.config/alacritty/alacritty.toml
chezmoi add ~/.config/starship.toml

# Edit a managed file
chezmoi edit ~/.zshrc

# Preview changes before applying
chezmoi diff

# Apply changes (copy from source to destination)
chezmoi apply

# Push to remote repository
chezmoi cd
git add . && git commit -m "Update dotfiles" && git push

# Set up on a new machine
chezmoi init --apply https://github.com/username/dotfiles.git
```

#### chezmoi Templates

Templates let you handle differences between machines:

```bash
# ~/.local/share/chezmoi/dot_zshrc.tmpl

# Common configuration for all machines
export EDITOR="nvim"
export VISUAL="nvim"

{{ if eq .chezmoi.os "darwin" -}}
# macOS-specific
eval "$(/opt/homebrew/bin/brew shellenv)"
source $(brew --prefix)/share/zsh-autosuggestions/zsh-autosuggestions.zsh
source $(brew --prefix)/share/zsh-syntax-highlighting/zsh-syntax-highlighting.zsh
alias pbcopy='pbcopy'
alias pbpaste='pbpaste'
{{ else if eq .chezmoi.os "linux" -}}
# Linux-specific
{{ if eq .chezmoi.osRelease.id "ubuntu" -}}
# Ubuntu-specific
{{ end -}}
{{ if lookPath "wl-copy" -}}
alias pbcopy='wl-copy'
alias pbpaste='wl-paste'
{{ else -}}
alias pbcopy='xclip -selection clipboard'
alias pbpaste='xclip -selection clipboard -o'
{{ end -}}
{{ end -}}

# Machine-specific configuration
{{ if eq .chezmoi.hostname "work-laptop" -}}
export HTTP_PROXY="http://proxy.corp.example.com:8080"
{{ end -}}
```

#### chezmoi Secrets Management

```bash
# Encrypt sensitive files
chezmoi add --encrypt ~/.ssh/config

# Use password manager integration
# .chezmoi.toml.tmpl
[data]
    email = "{{ (bitwarden "item" "my-email").login.username }}"

# Use environment variables
# In template: {{ env "API_KEY" }}
```

### GNU Stow

See the detailed example in [05-LINUX-TERMINAL.md](05-LINUX-TERMINAL.md#dotfile-management). Summary:

```bash
# Structure
~/dotfiles/
├── zsh/
│   └── .zshrc
├── tmux/
│   └── .tmux.conf
├── nvim/
│   └── .config/nvim/init.lua
└── alacritty/
    └── .config/alacritty/alacritty.toml

# Create symlinks
cd ~/dotfiles
stow zsh tmux nvim alacritty

# Remove symlinks
stow -D zsh
```

### Bare Git Repository

A minimal approach using only Git — no additional tools required:

```bash
# Initialise bare repo
git init --bare "$HOME/.dotfiles"

# Create alias
alias dotfiles='git --git-dir=$HOME/.dotfiles --work-tree=$HOME'

# Configure to not show untracked files
dotfiles config --local status.showUntrackedFiles no

# Add dotfiles
dotfiles add ~/.zshrc
dotfiles add ~/.tmux.conf
dotfiles add ~/.config/nvim/init.lua
dotfiles commit -m "Add initial dotfiles"
dotfiles remote add origin git@github.com:username/dotfiles.git
dotfiles push -u origin main

# Clone on a new machine
git clone --bare git@github.com:username/dotfiles.git "$HOME/.dotfiles"
alias dotfiles='git --git-dir=$HOME/.dotfiles --work-tree=$HOME'
dotfiles checkout
dotfiles config --local status.showUntrackedFiles no
```

---

## Portable Configurations

### Cross-Platform Detection

```bash
# Detect OS in shell scripts and configs
detect_os() {
    case "$(uname -s)" in
        Darwin)  echo "macos" ;;
        Linux)   echo "linux" ;;
        MINGW*|MSYS*|CYGWIN*) echo "windows" ;;
        *)       echo "unknown" ;;
    esac
}

# Use in .zshrc/.bashrc
case "$(uname -s)" in
    Darwin)
        # macOS-specific configuration
        eval "$(/opt/homebrew/bin/brew shellenv)"
        export BROWSER="open"
        ;;
    Linux)
        # Linux-specific configuration
        export BROWSER="xdg-open"

        # Wayland vs X11 clipboard
        if [ -n "$WAYLAND_DISPLAY" ]; then
            alias pbcopy='wl-copy'
            alias pbpaste='wl-paste'
        elif [ -n "$DISPLAY" ]; then
            alias pbcopy='xclip -selection clipboard'
            alias pbpaste='xclip -selection clipboard -o'
        fi
        ;;
esac
```

### Conditional Tool Loading

```bash
# Only load a tool if it's installed
command -v fzf &>/dev/null && source <(fzf --zsh)
command -v zoxide &>/dev/null && eval "$(zoxide init zsh)"
command -v direnv &>/dev/null && eval "$(direnv hook zsh)"
command -v starship &>/dev/null && eval "$(starship init zsh)"

# Conditional aliases
command -v eza &>/dev/null && alias ls='eza --icons --group-directories-first' || alias ls='ls --color=auto'
command -v bat &>/dev/null && alias cat='bat --paging=never' || alias cat='cat'
command -v nvim &>/dev/null && alias vim='nvim'
```

### Shared tmux Configuration

```bash
# ~/.tmux.conf — cross-platform clipboard support

# Detect platform and set clipboard command
if-shell "uname | grep -q Darwin" {
    bind -T copy-mode-vi y send-keys -X copy-pipe-and-cancel "pbcopy"
} {
    if-shell "[ -n \"$WAYLAND_DISPLAY\" ]" {
        bind -T copy-mode-vi y send-keys -X copy-pipe-and-cancel "wl-copy"
    } {
        bind -T copy-mode-vi y send-keys -X copy-pipe-and-cancel "xclip -selection clipboard"
    }
}
```

---

## Backup Strategies

### Three-Layer Backup Approach

```
Layer 1: Version Control (Git)
  ● All dotfiles in a Git repository
  ● Pushed to GitHub/GitLab as remote backup
  ● History of all changes

Layer 2: Machine-Level Backup
  ● Time Machine (macOS)
  ● Borg Backup / restic (Linux)
  ● Includes non-version-controlled files

Layer 3: Cloud Sync (Optional)
  ● Brewfile/package lists synced
  ● SSH keys in password manager (1Password, Bitwarden)
  ● Not for sensitive unencrypted data
```

### Automated Setup Scripts

```bash
#!/usr/bin/env bash
# setup.sh — Bootstrap a new machine

set -euo pipefail

OS="$(uname -s)"
echo "Setting up for $OS"

# 1. Install package manager
if [ "$OS" = "Darwin" ]; then
    if ! command -v brew &>/dev/null; then
        /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
        eval "$(/opt/homebrew/bin/brew shellenv)"
    fi
fi

# 2. Install essential packages
if [ "$OS" = "Darwin" ]; then
    brew bundle install --file=Brewfile
elif [ "$OS" = "Linux" ]; then
    sudo apt update && sudo apt install -y \
        zsh tmux neovim fzf ripgrep fd-find bat eza zoxide git curl
fi

# 3. Clone and apply dotfiles
if ! command -v chezmoi &>/dev/null; then
    sh -c "$(curl -fsLS get.chezmoi.io)"
fi
chezmoi init --apply https://github.com/username/dotfiles.git

# 4. Set default shell
if [ "$SHELL" != "$(which zsh)" ]; then
    chsh -s "$(which zsh)"
fi

# 5. Install tmux plugins
if [ ! -d "$HOME/.tmux/plugins/tpm" ]; then
    git clone https://github.com/tmux-plugins/tpm ~/.tmux/plugins/tpm
fi

# 6. Install Neovim plugins (headless)
nvim --headless "+Lazy! sync" +qa 2>/dev/null

echo "Setup complete. Restart your terminal."
```

---

## Version Control for Configs

### Commit Message Conventions

```
# Use clear, descriptive commit messages for dotfile changes
git commit -m "zsh: add fzf key bindings and configuration"
git commit -m "tmux: change prefix to Ctrl+a, add vim navigation"
git commit -m "nvim: add LSP configuration for Python and Go"
git commit -m "alacritty: switch to Catppuccin Mocha theme"
git commit -m "shell: add cross-platform clipboard aliases"
```

### .gitignore for Dotfiles

```gitignore
# ~/.local/share/chezmoi/.gitignore (or ~/dotfiles/.gitignore)

# Secrets and machine-specific files
*.local
*.secret
.env
.envrc

# Editor backups
*~
*.swp
*.swo
.netrwhist

# Plugin directories (managed by plugin managers)
.tmux/plugins/
.config/nvim/lazy-lock.json

# OS files
.DS_Store
Thumbs.db

# Compiled files
*.pyc
__pycache__/
```

---

## Ergonomic Key Bindings

### Principles

1. **Consistency across tools**: Use the same keys for similar actions in tmux, Neovim, and shell
2. **Minimise reaching**: Keep common actions on home row
3. **Prefer Ctrl over Alt**: Ctrl is more reliably transmitted across terminals and SSH
4. **Use leader/prefix keys**: Namespace your shortcuts to avoid conflicts

### Recommended Key Binding Map

```
Navigation (consistent across tmux + Neovim):
  Ctrl+h  →  Move left  (tmux pane / Neovim window)
  Ctrl+j  →  Move down  (tmux pane / Neovim window)
  Ctrl+k  →  Move up    (tmux pane / Neovim window)
  Ctrl+l  →  Move right (tmux pane / Neovim window)

tmux Prefix: Ctrl+a (instead of default Ctrl+b)
  Prefix + |  →  Vertical split
  Prefix + -  →  Horizontal split
  Prefix + r  →  Reload config
  Prefix + h/j/k/l  →  Navigate panes (backup)
  Prefix + H/J/K/L  →  Resize panes

Neovim Leader: Space
  Space + f + f  →  Find files (Telescope)
  Space + f + g  →  Live grep (Telescope)
  Space + f + b  →  Find buffers
  Space + w      →  Save file
  Space + q      →  Quit

Shell:
  Ctrl+R  →  Fuzzy history search (fzf)
  Ctrl+T  →  Fuzzy file finder (fzf)
  Alt+C   →  Fuzzy directory jump (fzf)
```

### Caps Lock as Ctrl

Remapping Caps Lock to Ctrl is the single most impactful ergonomic improvement for terminal users:

```bash
# macOS: System Preferences → Keyboard → Modifier Keys → Caps Lock → Control

# Linux (X11):
setxkbmap -option ctrl:nocaps

# Linux (permanent — /etc/default/keyboard):
XKBOPTIONS="ctrl:nocaps"

# Linux (Wayland/GNOME):
gsettings set org.gnome.desktop.input-sources xkb-options "['ctrl:nocaps']"
```

---

## Terminal Multiplexer Workflows

### Named Sessions for Context Switching

```bash
# Create sessions for different projects
tmux new -s api -d
tmux new -s frontend -d
tmux new -s infra -d

# Quick switch script (add to .bashrc/.zshrc)
ts() {
    # Switch to session or create it
    if tmux has-session -t "$1" 2>/dev/null; then
        tmux switch-client -t "$1" || tmux attach -t "$1"
    else
        tmux new-session -d -s "$1" -c "${2:-$HOME}"
        tmux switch-client -t "$1" || tmux attach -t "$1"
    fi
}

# Tab completion for ts
_ts() {
    local sessions
    sessions=$(tmux list-sessions -F "#{session_name}" 2>/dev/null)
    COMPREPLY=($(compgen -W "$sessions" -- "${COMP_WORDS[COMP_CWORD]}"))
}
complete -F _ts ts
```

### Standard Window Layout

```bash
# Consistent window naming convention
# Window 0: editor   (Neovim)
# Window 1: shell    (general commands)
# Window 2: server   (dev server / process)
# Window 3: git      (version control)
# Window 4: logs     (log tailing)

# Add to tmux startup script or session manager
```

---

## Efficient Navigation Patterns

### Directory Navigation

```bash
# 1. zoxide for frequently-visited directories
z proj           # Jump to ~/projects (or wherever you visit most)
z api src        # Jump with multiple keywords

# 2. fzf Alt+C for exploring
Alt+C            # Fuzzy-find and cd to any subdirectory

# 3. Shell directory stack
setopt AUTO_PUSHD         # Zsh: auto-push dirs onto stack
dirs -v                   # View directory stack
cd -2                     # Jump to second-most-recent directory
```

### File Navigation

```bash
# 1. fzf Ctrl+T for files
Ctrl+T              # Fuzzy-find files, insert path at cursor

# 2. Neovim Telescope for project files
<leader>ff          # Find files in project
<leader>fg          # Live grep in project
<leader>fr          # Recent files
<leader>fb          # Open buffers

# 3. Harpoon for pinned files
<leader>a           # Add file to harpoon
<leader>1-4         # Jump to harpooned file
```

### Search Patterns

```bash
# Search in files — ripgrep
rg "pattern"                    # Recursive search
rg -t py "import"               # Search only Python files
rg -l "TODO"                    # List files with TODOs
rg -C 3 "error" --glob "*.log" # Context + file filter

# Search file names — fd
fd "readme"                     # Find files named readme
fd -e md                        # Find all .md files
fd -t d "config"                # Find directories named config

# Search command history — fzf
Ctrl+R                          # Fuzzy search history
```

---

## Clipboard Integration

### Cross-Platform Clipboard Commands

```bash
# Add to .bashrc or .zshrc — provides consistent 'clip' and 'paste' commands

if [[ "$(uname)" == "Darwin" ]]; then
    alias clip='pbcopy'
    alias paste='pbpaste'
elif [[ -n "$WSL_DISTRO_NAME" ]]; then
    alias clip='clip.exe'
    alias paste='powershell.exe -c Get-Clipboard'
elif [[ -n "$WAYLAND_DISPLAY" ]]; then
    alias clip='wl-copy'
    alias paste='wl-paste'
elif [[ -n "$DISPLAY" ]]; then
    alias clip='xclip -selection clipboard'
    alias paste='xclip -selection clipboard -o'
fi

# Usage:
# echo "text" | clip      # Copy to clipboard
# paste                    # Paste from clipboard
# paste | wc -l            # Count lines in clipboard
```

### tmux Clipboard (See [02-TMUX.md](02-TMUX.md#clipboard-integration))

### Neovim Clipboard

```lua
-- In Neovim options (already set in our config)
vim.opt.clipboard = "unnamedplus"  -- Use system clipboard
```

### OSC 52 (Remote Clipboard via SSH)

```bash
# OSC 52 allows copying to local clipboard from remote SSH sessions
# Supported by: kitty, WezTerm, iTerm2, Alacritty, Windows Terminal

# tmux configuration
set -g set-clipboard on

# Neovim: use OSC 52 provider
# Neovim 0.10+ auto-detects OSC 52 support
```

---

## Configuration Organisation

### Recommended Directory Structure

```
~/
├── .config/
│   ├── alacritty/alacritty.toml     # Terminal emulator
│   ├── bat/config                    # bat configuration
│   ├── fish/config.fish              # Fish shell (if used)
│   ├── git/config                    # Git configuration
│   ├── kitty/kitty.conf              # kitty terminal (if used)
│   ├── nvim/                         # Neovim configuration
│   │   ├── init.lua
│   │   └── lua/
│   ├── ripgrep/config                # ripgrep defaults
│   ├── shell/                        # Shared shell config
│   │   ├── aliases.sh
│   │   ├── functions.sh
│   │   ├── path.sh
│   │   └── local.sh                  # Machine-specific (git-ignored)
│   ├── starship.toml                 # Prompt configuration
│   └── wezterm/wezterm.lua           # WezTerm (if used)
├── .bashrc                           # Bash config
├── .zshrc                            # Zsh config (sources .config/shell/)
├── .tmux.conf                        # tmux configuration
└── .ssh/config                       # SSH configuration
```

---

## Quick Reference Checklist

| Practice | Status | Priority |
|----------|--------|----------|
| Dotfiles in version control | ☐ | 🔴 Critical |
| Cross-platform clipboard aliases | ☐ | 🟠 High |
| tmux for session persistence | ☐ | 🟠 High |
| Ergonomic prefix key (Ctrl+a) | ☐ | 🟠 High |
| Caps Lock → Ctrl remap | ☐ | 🟠 High |
| Consistent Ctrl+h/j/k/l navigation | ☐ | 🟠 High |
| fzf integration in shell | ☐ | 🟠 High |
| zoxide for directory jumping | ☐ | 🟡 Medium |
| Machine-specific config separation | ☐ | 🟡 Medium |
| Automated setup script | ☐ | 🟡 Medium |
| Nerd Font installed | ☐ | 🟡 Medium |
| Modern CLI tool aliases | ☐ | 🟡 Medium |
| OSC 52 clipboard for SSH | ☐ | 🟢 Nice to have |
| Brewfile / package list maintained | ☐ | 🟢 Nice to have |

---

## Next Steps

- **Avoid anti-patterns** → [09-ANTI-PATTERNS.md](09-ANTI-PATTERNS.md) — Common terminal mistakes to avoid
- **tmux setup** → [02-TMUX.md](02-TMUX.md) — Detailed tmux configuration and workflows
- **Neovim setup** → [03-NEOVIM.md](03-NEOVIM.md) — IDE-like editor configuration
- **Productivity tools** → [07-PRODUCTIVITY-TOOLS.md](07-PRODUCTIVITY-TOOLS.md) — Modern CLI tool installation and setup
- **Learning path** → [LEARNING-PATH.md](LEARNING-PATH.md) — Structured curriculum for building your terminal workflow

---

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2025 | Initial terminal best practices documentation |
