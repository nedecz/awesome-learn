# Shell Fundamentals

A comprehensive guide to Unix shells — covering Bash, Zsh, and Fish, their configuration, environment variables, aliases, functions, scripting basics, PATH management, and prompt customisation with tools like Starship, Oh My Zsh, and Powerlevel10k.

---

## Table of Contents

1. [Overview](#overview)
2. [Shell Comparison](#shell-comparison)
3. [Bash Fundamentals](#bash-fundamentals)
4. [Zsh Fundamentals](#zsh-fundamentals)
5. [Fish Shell](#fish-shell)
6. [Shell Configuration Files](#shell-configuration-files)
7. [Environment Variables](#environment-variables)
8. [Aliases and Functions](#aliases-and-functions)
9. [Shell Scripting Basics](#shell-scripting-basics)
10. [PATH Management](#path-management)
11. [Prompt Customisation](#prompt-customisation)
12. [Shell History](#shell-history)
13. [Job Control](#job-control)
14. [Next Steps](#next-steps)
15. [Version History](#version-history)

---

## Overview

The shell is the primary interface between you and the operating system. Choosing the right shell and configuring it properly is the foundation of an effective terminal workflow. This document covers the three most popular interactive shells and teaches you how to configure them for maximum productivity.

### Target Audience

- **Developers** setting up their local development environment
- **DevOps Engineers** writing shell scripts and managing remote systems
- **New engineers** learning the command line for the first time

### Scope

- Comparison of Bash, Zsh, and Fish shells
- Configuration files and their loading order
- Environment variables, aliases, and shell functions
- Shell scripting fundamentals
- PATH management across platforms
- Prompt customisation with Starship, Oh My Zsh, and Powerlevel10k

---

## Shell Comparison

| Feature | Bash | Zsh | Fish |
|---------|------|-----|------|
| **Default on** | Most Linux distros | macOS (since Catalina) | None (install separately) |
| **POSIX compatible** | ✅ Yes | ✅ Mostly | ❌ No (own syntax) |
| **Config file** | `~/.bashrc` | `~/.zshrc` | `~/.config/fish/config.fish` |
| **Tab completion** | Basic (programmable) | Advanced (context-aware) | Excellent (out of the box) |
| **Syntax highlighting** | ❌ (needs plugin) | ❌ (needs plugin) | ✅ Built-in |
| **Autosuggestions** | ❌ (needs plugin) | ❌ (needs plugin) | ✅ Built-in |
| **Scripting language** | POSIX sh + extensions | POSIX sh + extensions | Own language |
| **Plugin ecosystem** | bash-it | Oh My Zsh, zinit, antidote | fisher, oh-my-fish |
| **Startup speed** | Fast | Can be slow (with plugins) | Fast |
| **Glob patterns** | Basic | Extended (`**/*`, qualifiers) | Basic |
| **Right prompt** | ❌ | ✅ `RPROMPT` | ✅ Built-in |

### When to Choose Each Shell

```
Choose Bash when:
  ● Writing scripts that must run on any Unix system
  ● Working on servers where only Bash is available
  ● Maximum compatibility is required
  ● You prefer the POSIX standard

Choose Zsh when:
  ● You want a powerful interactive shell with POSIX compatibility
  ● You want the Oh My Zsh ecosystem
  ● You need advanced globbing and completion
  ● You're on macOS (it's the default)

Choose Fish when:
  ● You want the best out-of-the-box experience
  ● You don't need POSIX script compatibility
  ● You prefer sane defaults over configuration
  ● You're new to the command line
```

---

## Bash Fundamentals

### Checking Your Bash Version

```bash
bash --version
# GNU bash, version 5.2.x

# macOS ships with an old Bash (3.2) due to GPLv3 licensing
# Install a modern version:
brew install bash
# Add to /etc/shells and change default:
echo '/opt/homebrew/bin/bash' | sudo tee -a /etc/shells
chsh -s /opt/homebrew/bin/bash
```

### Bash Configuration

Bash uses different configuration files depending on the session type:

```
Login Shell (ssh, console login, bash --login):
  /etc/profile → ~/.bash_profile → ~/.bash_login → ~/.profile
  (reads the FIRST one found in order)

Interactive Non-Login Shell (new terminal window):
  ~/.bashrc

Non-Interactive Shell (scripts):
  $BASH_ENV (if set)
```

A practical setup that avoids confusion:

```bash
# ~/.bash_profile — source .bashrc for login shells
if [ -f "$HOME/.bashrc" ]; then
    source "$HOME/.bashrc"
fi

# ~/.bashrc — main configuration file
# Guard against non-interactive shells
[[ $- != *i* ]] && return

# Shell options
set -o vi                    # Vi mode (or: set -o emacs for default)
shopt -s histappend          # Append to history, don't overwrite
shopt -s checkwinsize        # Update LINES and COLUMNS after each command
shopt -s globstar            # Enable ** recursive globbing (Bash 4+)
shopt -s cdspell             # Autocorrect minor cd typos
shopt -s dirspell            # Autocorrect directory name typos in completion
shopt -s nocaseglob          # Case-insensitive globbing

# History configuration
export HISTSIZE=50000
export HISTFILESIZE=50000
export HISTCONTROL=ignoreboth:erasedups
export HISTIGNORE="ls:cd:pwd:exit:clear:history"
export HISTTIMEFORMAT="%Y-%m-%d %H:%M:%S  "

# Prompt
export PS1='\[\033[01;32m\]\u@\h\[\033[00m\]:\[\033[01;34m\]\w\[\033[00m\]\$ '
```

### Bash Keyboard Shortcuts (Emacs Mode)

| Shortcut | Action |
|----------|--------|
| `Ctrl+A` | Move to beginning of line |
| `Ctrl+E` | Move to end of line |
| `Ctrl+W` | Delete word before cursor |
| `Alt+D` | Delete word after cursor |
| `Ctrl+U` | Delete from cursor to beginning of line |
| `Ctrl+K` | Delete from cursor to end of line |
| `Ctrl+R` | Reverse search through history |
| `Ctrl+L` | Clear screen (same as `clear`) |
| `Ctrl+P` / `Ctrl+N` | Previous / next history entry |
| `Alt+B` / `Alt+F` | Move back / forward one word |
| `Ctrl+D` | Exit shell (when line is empty) |
| `Ctrl+Z` | Suspend current foreground process |
| `Ctrl+C` | Send SIGINT to foreground process |

---

## Zsh Fundamentals

### Installing Zsh

```bash
# macOS (already default, but update to latest)
brew install zsh

# Debian/Ubuntu
sudo apt update && sudo apt install zsh

# Fedora
sudo dnf install zsh

# Set as default shell
chsh -s $(which zsh)
```

### Zsh Configuration

Zsh loads configuration files in a specific order:

```
Login Shell:
  /etc/zshenv → ~/.zshenv → /etc/zprofile → ~/.zprofile →
  /etc/zshrc → ~/.zshrc → /etc/zlogin → ~/.zlogin

Interactive Shell:
  /etc/zshenv → ~/.zshenv → /etc/zshrc → ~/.zshrc

Non-Interactive Shell:
  /etc/zshenv → ~/.zshenv
```

Recommended file usage:

| File | Purpose | Load Order |
|------|---------|------------|
| `~/.zshenv` | Environment variables needed everywhere (PATH, EDITOR) | Always loaded first |
| `~/.zprofile` | Login-specific setup (runs once per login session) | Login shells only |
| `~/.zshrc` | Interactive shell config (aliases, prompts, plugins, completions) | Interactive shells |
| `~/.zlogin` | Commands to run after `.zshrc` in login shells | Login shells only |
| `~/.zlogout` | Cleanup on logout | On logout |

```bash
# ~/.zshenv — always loaded
export EDITOR="nvim"
export VISUAL="nvim"
export PAGER="less"
export LANG="en_US.UTF-8"

# ~/.zshrc — interactive shell configuration
# Options
setopt AUTO_CD              # cd into directories by typing their name
setopt AUTO_PUSHD           # Push directories onto the stack with cd
setopt PUSHD_IGNORE_DUPS    # Don't push duplicate directories
setopt CORRECT              # Suggest corrections for mistyped commands
setopt EXTENDED_GLOB        # Enable extended globbing (#, ~, ^)
setopt INTERACTIVE_COMMENTS # Allow comments in interactive shell
setopt NO_BEEP              # Disable terminal beep

# History
setopt SHARE_HISTORY        # Share history across all Zsh sessions
setopt HIST_IGNORE_ALL_DUPS # Remove older duplicate entries
setopt HIST_REDUCE_BLANKS   # Remove superfluous blanks from history
setopt HIST_VERIFY          # Show substituted command before executing
HISTSIZE=50000
SAVEHIST=50000
HISTFILE=~/.zsh_history

# Completion system
autoload -Uz compinit && compinit
zstyle ':completion:*' matcher-list 'm:{a-z}={A-Za-z}'  # Case-insensitive
zstyle ':completion:*' list-colors "${(s.:.)LS_COLORS}"  # Coloured completions
zstyle ':completion:*' menu select                        # Menu-style selection
zstyle ':completion:*:descriptions' format '%F{green}-- %d --%f'

# Key bindings
bindkey -e                           # Emacs key bindings
bindkey '^[[A' history-search-backward  # Up arrow searches history
bindkey '^[[B' history-search-forward   # Down arrow searches history
bindkey '^[[3~' delete-char             # Delete key works properly
```

### Zsh Unique Features

```bash
# Extended globbing
ls **/*.md                    # Recursive glob (all .md files in all subdirs)
ls *(.)                       # Only regular files
ls *(/)                       # Only directories
ls *(@)                       # Only symlinks
ls *(m-7)                     # Modified in the last 7 days
ls *(Lk+100)                  # Larger than 100KB

# Named directories
hash -d proj=~/projects       # Access with ~proj
cd ~proj

# Suffix aliases
alias -s md=nvim              # Typing file.md opens it in nvim
alias -s json=jq              # Typing data.json pipes through jq
alias -s git="git clone"      # Typing repo.git clones it

# Global aliases
alias -g G='| grep'           # ls G pattern
alias -g L='| less'           # command L
alias -g C='| wc -l'          # ls C
alias -g H='| head'           # command H
alias -g T='| tail'           # command T
alias -g J='| jq .'           # curl ... J
```

---

## Fish Shell

### Installing Fish

```bash
# macOS
brew install fish

# Debian/Ubuntu
sudo apt-add-repository ppa:fish-shell/release-3
sudo apt update && sudo apt install fish

# Fedora
sudo dnf install fish

# Set as default shell
chsh -s $(which fish)
```

### Fish Configuration

Fish uses a different configuration structure from POSIX shells:

```
~/.config/fish/
├── config.fish          # Main configuration (like .bashrc/.zshrc)
├── fish_variables       # Universal variables (managed by Fish)
├── functions/           # Autoloaded function files
│   ├── fish_prompt.fish # Custom prompt function
│   └── myfunction.fish  # One function per file
├── completions/         # Custom completions
│   └── mytool.fish
└── conf.d/              # Auto-sourced configuration snippets
    ├── aliases.fish
    └── environment.fish
```

```fish
# ~/.config/fish/config.fish

# Environment variables (Fish uses 'set' instead of 'export')
set -gx EDITOR nvim
set -gx VISUAL nvim
set -gx LANG en_US.UTF-8

# PATH management (Fish uses fish_add_path)
fish_add_path /usr/local/bin
fish_add_path $HOME/.local/bin
fish_add_path $HOME/.cargo/bin

# Aliases (Fish calls them 'abbreviations' — they expand as you type)
abbr -a g git
abbr -a ga "git add"
abbr -a gc "git commit"
abbr -a gp "git push"
abbr -a gst "git status"
abbr -a k kubectl
abbr -a d docker
abbr -a dc "docker compose"

# Disable greeting
set -g fish_greeting
```

### Fish Functions

```fish
# ~/.config/fish/functions/mkcd.fish
function mkcd --description "Create directory and cd into it"
    mkdir -p $argv[1] && cd $argv[1]
end

# ~/.config/fish/functions/backup.fish
function backup --description "Create a timestamped backup of a file"
    cp $argv[1] $argv[1].(date +%Y%m%d%H%M%S).bak
end
```

### Fish Plugin Management with Fisher

```fish
# Install Fisher
curl -sL https://raw.githubusercontent.com/jorgebucaran/fisher/main/functions/fisher.fish | source && fisher install jorgebucaran/fisher

# Popular plugins
fisher install PatrickF1/fzf.fish           # fzf integration
fisher install jethrokuan/z                  # z directory jumper
fisher install jorgebucaran/autopair.fish    # Auto-close brackets/quotes
fisher install meaningful-ooo/sponge         # Clean typos from history
```

---

## Shell Configuration Files

### Configuration Loading Order Summary

```
┌──────────────┬─────────────────────────────────────────────────┐
│              │               Shell Type                         │
│   File       ├───────────────┬────────────────┬────────────────┤
│              │ Login          │ Interactive     │ Script         │
│              │ (ssh, login)   │ (new terminal)  │ (bash file.sh) │
├──────────────┼───────────────┼────────────────┼────────────────┤
│ Bash:        │               │                │                │
│ .bash_profile│ ✅             │ ❌              │ ❌              │
│ .bashrc      │ ❌ (unless     │ ✅              │ ❌              │
│              │  sourced)      │                │                │
├──────────────┼───────────────┼────────────────┼────────────────┤
│ Zsh:         │               │                │                │
│ .zshenv      │ ✅             │ ✅              │ ✅              │
│ .zprofile    │ ✅             │ ❌              │ ❌              │
│ .zshrc       │ ✅             │ ✅              │ ❌              │
│ .zlogin      │ ✅             │ ❌              │ ❌              │
├──────────────┼───────────────┼────────────────┼────────────────┤
│ Fish:        │               │                │                │
│ config.fish  │ ✅             │ ✅              │ ❌              │
│ conf.d/*.fish│ ✅             │ ✅              │ ❌              │
└──────────────┴───────────────┴────────────────┴────────────────┘
```

### Best Practice: Keep Configuration Organised

```bash
# Bash/Zsh: Source separate files from your main rc file
# ~/.bashrc or ~/.zshrc

# Load modular configuration
for config_file in ~/.config/shell/{aliases,functions,path,prompt,local}.sh; do
    [ -f "$config_file" ] && source "$config_file"
done
```

```
~/.config/shell/
├── aliases.sh      # All aliases
├── functions.sh    # Shell functions
├── path.sh         # PATH modifications
├── prompt.sh       # Prompt configuration
└── local.sh        # Machine-specific config (git-ignored)
```

---

## Environment Variables

### Essential Environment Variables

| Variable | Purpose | Example |
|----------|---------|---------|
| `PATH` | Directories to search for executables | `/usr/local/bin:/usr/bin:/bin` |
| `HOME` | Current user's home directory | `/home/username` |
| `EDITOR` | Default text editor | `nvim` |
| `VISUAL` | Default visual editor (used by git, etc.) | `nvim` |
| `SHELL` | Current user's login shell | `/bin/zsh` |
| `TERM` | Terminal type for capability detection | `xterm-256color` |
| `LANG` | Locale setting | `en_US.UTF-8` |
| `PAGER` | Default pager for man, git, etc. | `less` |
| `XDG_CONFIG_HOME` | User configuration directory | `~/.config` |
| `XDG_DATA_HOME` | User data directory | `~/.local/share` |
| `XDG_CACHE_HOME` | User cache directory | `~/.cache` |

### Setting Environment Variables

```bash
# Bash/Zsh — set for current session and child processes
export EDITOR="nvim"
export VISUAL="nvim"

# Bash/Zsh — set only for current shell (not inherited)
MYVAR="value"

# Fish — set globally (persists) and export
set -gx EDITOR nvim

# View all environment variables
env | sort

# View specific variable
echo $EDITOR
printenv EDITOR

# Unset a variable
unset MYVAR            # Bash/Zsh
set -e MYVAR           # Fish
```

### XDG Base Directory Specification

The XDG spec standardises where applications store configuration, data, and cache:

```bash
# Add to ~/.zshenv or ~/.bashrc
export XDG_CONFIG_HOME="$HOME/.config"
export XDG_DATA_HOME="$HOME/.local/share"
export XDG_CACHE_HOME="$HOME/.cache"
export XDG_STATE_HOME="$HOME/.local/state"
```

---

## Aliases and Functions

### Aliases

```bash
# ~/.config/shell/aliases.sh (sourced from .bashrc or .zshrc)

# Navigation
alias ..='cd ..'
alias ...='cd ../..'
alias ....='cd ../../..'

# Safety nets
alias rm='rm -i'
alias mv='mv -i'
alias cp='cp -i'

# ls with colours and details
alias ls='ls --color=auto'
alias ll='ls -alFh'
alias la='ls -A'

# grep with colours
alias grep='grep --color=auto'
alias fgrep='fgrep --color=auto'
alias egrep='egrep --color=auto'

# Git shortcuts
alias g='git'
alias gs='git status'
alias ga='git add'
alias gc='git commit'
alias gp='git push'
alias gl='git log --oneline --graph --decorate -20'
alias gd='git diff'
alias gco='git checkout'
alias gb='git branch'

# Docker shortcuts
alias d='docker'
alias dc='docker compose'
alias dps='docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"'

# Kubernetes shortcuts
alias k='kubectl'
alias kgp='kubectl get pods'
alias kgs='kubectl get svc'
alias kgn='kubectl get nodes'

# Quick edit config files
alias zshrc='${EDITOR:-nvim} ~/.zshrc'
alias bashrc='${EDITOR:-nvim} ~/.bashrc'
alias vimrc='${EDITOR:-nvim} ~/.config/nvim/init.lua'
alias tmuxconf='${EDITOR:-nvim} ~/.tmux.conf'

# System
alias df='df -h'
alias du='du -h'
alias free='free -h'
alias ports='ss -tulanp'
```

### Shell Functions

Functions are more powerful than aliases — they accept arguments and can contain logic:

```bash
# ~/.config/shell/functions.sh

# Create directory and cd into it
mkcd() {
    mkdir -p "$1" && cd "$1"
}

# Extract any archive format
extract() {
    if [ -f "$1" ]; then
        case "$1" in
            *.tar.bz2) tar xjf "$1"    ;;
            *.tar.gz)  tar xzf "$1"    ;;
            *.tar.xz)  tar xJf "$1"    ;;
            *.bz2)     bunzip2 "$1"    ;;
            *.rar)     unrar x "$1"    ;;
            *.gz)      gunzip "$1"     ;;
            *.tar)     tar xf "$1"     ;;
            *.tbz2)    tar xjf "$1"    ;;
            *.tgz)     tar xzf "$1"    ;;
            *.zip)     unzip "$1"      ;;
            *.Z)       uncompress "$1" ;;
            *.7z)      7z x "$1"       ;;
            *)         echo "'$1' cannot be extracted" ;;
        esac
    else
        echo "'$1' is not a valid file"
    fi
}

# Find process by name
psg() {
    ps aux | grep -v grep | grep -i "$1"
}

# Quick HTTP server in current directory
serve() {
    local port="${1:-8000}"
    echo "Serving on http://localhost:$port"
    python3 -m http.server "$port"
}

# Git log with fzf for interactive commit browsing
glog() {
    git log --oneline --graph --decorate --all | fzf --preview 'git show --color=always {1}' --no-sort
}
```

---

## Shell Scripting Basics

### Script Structure

```bash
#!/usr/bin/env bash
# Script: setup-dev.sh
# Description: Set up development environment
# Usage: ./setup-dev.sh [--force]

set -euo pipefail  # Exit on error, undefined vars, pipe failures
IFS=$'\n\t'        # Safer word splitting

# Constants
readonly SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
readonly LOG_FILE="/tmp/setup-dev.log"

# Functions
log() {
    echo "[$(date +'%Y-%m-%d %H:%M:%S')] $*" | tee -a "$LOG_FILE"
}

die() {
    log "ERROR: $*" >&2
    exit 1
}

# Argument parsing
FORCE=false
while [[ $# -gt 0 ]]; do
    case "$1" in
        --force) FORCE=true; shift ;;
        --help)  echo "Usage: $0 [--force]"; exit 0 ;;
        *)       die "Unknown option: $1" ;;
    esac
done

# Main logic
main() {
    log "Starting development environment setup"
    
    # Check prerequisites
    command -v git >/dev/null 2>&1 || die "git is required"
    command -v curl >/dev/null 2>&1 || die "curl is required"
    
    log "All prerequisites met"
    log "Setup complete"
}

main "$@"
```

### Common Patterns

```bash
# Check if command exists
if command -v nvim &>/dev/null; then
    echo "Neovim is installed"
fi

# Default values
name="${1:-World}"
echo "Hello, $name"

# String operations
str="Hello, World"
echo "${str^^}"       # HELLO, WORLD (uppercase)
echo "${str,,}"       # hello, world (lowercase)
echo "${str/World/Bash}"  # Hello, Bash (substitution)
echo "${#str}"        # 12 (length)

# Arrays
files=("file1.txt" "file2.txt" "file3.txt")
echo "${files[0]}"           # First element
echo "${files[@]}"           # All elements
echo "${#files[@]}"          # Array length

for f in "${files[@]}"; do
    echo "Processing: $f"
done

# Conditional file operations
[ -f "$file" ]   # File exists and is regular file
[ -d "$dir" ]    # Directory exists
[ -r "$file" ]   # File is readable
[ -w "$file" ]   # File is writable
[ -x "$file" ]   # File is executable
[ -s "$file" ]   # File exists and is not empty
[ -L "$file" ]   # File is a symbolic link
```

---

## PATH Management

### How PATH Works

The `PATH` variable is a colon-separated list of directories. When you type a command, the shell searches each directory in order:

```bash
# View current PATH (one directory per line)
echo "$PATH" | tr ':' '\n'

# Find where a command lives
which nvim
type nvim
command -v nvim
```

### Managing PATH

```bash
# Bash/Zsh — prepend (higher priority)
export PATH="$HOME/.local/bin:$PATH"

# Bash/Zsh — append (lower priority)
export PATH="$PATH:$HOME/scripts"

# Avoid duplicate entries (Zsh)
typeset -U PATH path  # Unique entries only

# Fish — use fish_add_path (handles deduplication)
fish_add_path $HOME/.local/bin
fish_add_path $HOME/.cargo/bin
```

### Recommended PATH Setup

```bash
# ~/.config/shell/path.sh

# User-local binaries (highest priority)
export PATH="$HOME/.local/bin:$PATH"

# Language-specific package managers
[ -d "$HOME/.cargo/bin" ] && export PATH="$HOME/.cargo/bin:$PATH"     # Rust
[ -d "$HOME/.go/bin" ] && export PATH="$HOME/.go/bin:$PATH"           # Go
[ -d "$HOME/.npm-global/bin" ] && export PATH="$HOME/.npm-global/bin:$PATH"  # Node.js

# Platform-specific
if [[ "$(uname)" == "Darwin" ]]; then
    # Homebrew (Apple Silicon)
    eval "$(/opt/homebrew/bin/brew shellenv)"
fi

# Remove duplicates (Bash)
PATH="$(echo "$PATH" | awk -v RS=: -v ORS=: '!seen[$0]++')"
export PATH="${PATH%:}"  # Remove trailing colon
```

---

## Prompt Customisation

### Starship (Cross-Shell Prompt)

[Starship](https://starship.rs/) is a fast, customisable, cross-shell prompt written in Rust:

```bash
# Install Starship
curl -sS https://starship.rs/install.sh | sh

# Activate in Bash (~/.bashrc)
eval "$(starship init bash)"

# Activate in Zsh (~/.zshrc)
eval "$(starship init zsh)"

# Activate in Fish (~/.config/fish/config.fish)
starship init fish | source
```

```toml
# ~/.config/starship.toml

# Don't add a blank line between prompts
add_newline = false

# Replace default prompt symbol
[character]
success_symbol = "[❯](bold green)"
error_symbol = "[❯](bold red)"

# Directory display
[directory]
truncation_length = 3
truncation_symbol = "…/"

# Git branch
[git_branch]
symbol = " "
style = "bold purple"

# Git status
[git_status]
ahead = "⇡${count}"
behind = "⇣${count}"
diverged = "⇕⇡${ahead_count}⇣${behind_count}"
modified = "!${count}"
staged = "+${count}"
untracked = "?${count}"

# Language versions (only show when relevant files are present)
[nodejs]
symbol = " "

[python]
symbol = " "

[rust]
symbol = " "

[golang]
symbol = " "

# Command duration (show for commands > 2 seconds)
[cmd_duration]
min_time = 2_000
format = "took [$duration](bold yellow)"

# Kubernetes context
[kubernetes]
disabled = false
symbol = "☸ "
format = '[$symbol$context( \($namespace\))]($style) '
```

### Oh My Zsh

```bash
# Install Oh My Zsh
sh -c "$(curl -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"

# Configure in ~/.zshrc
export ZSH="$HOME/.oh-my-zsh"
ZSH_THEME="robbyrussell"  # or "agnoster", "powerlevel10k/powerlevel10k"

# Plugins
plugins=(
    git              # Git aliases and functions
    docker           # Docker completions
    kubectl          # Kubernetes completions
    z                # Directory jumping
    fzf              # fzf integration
    sudo             # Press Esc twice to add sudo
    history          # History search improvements
    colored-man-pages # Coloured man pages
    command-not-found # Suggest packages for unknown commands
)

source $ZSH/oh-my-zsh.sh
```

### Powerlevel10k (Zsh Theme)

```bash
# Install Powerlevel10k
git clone --depth=1 https://github.com/romkatv/powerlevel10k.git \
    ${ZSH_CUSTOM:-$HOME/.oh-my-zsh/custom}/themes/powerlevel10k

# Set theme in ~/.zshrc
ZSH_THEME="powerlevel10k/powerlevel10k"

# Run configuration wizard
p10k configure

# Or manually configure in ~/.p10k.zsh
# Key customisation points:
typeset -g POWERLEVEL9K_LEFT_PROMPT_ELEMENTS=(
    dir                     # Current directory
    vcs                     # Git status
    prompt_char             # Prompt symbol
)
typeset -g POWERLEVEL9K_RIGHT_PROMPT_ELEMENTS=(
    status                  # Exit code of last command
    command_execution_time  # Duration of last command
    background_jobs         # Background job indicator
    node_version            # Node.js version
    python_env              # Python virtual environment
    kubecontext             # Kubernetes context
    time                    # Current time
)
```

---

## Shell History

### History Configuration

```bash
# Bash — ~/.bashrc
export HISTSIZE=50000              # History entries in memory
export HISTFILESIZE=50000          # History entries on disk
export HISTCONTROL=ignoreboth:erasedups  # Ignore duplicates and space-prefixed
export HISTIGNORE="ls:cd:pwd:exit:clear"  # Ignore trivial commands
export HISTTIMEFORMAT="%F %T  "    # Timestamp format
shopt -s histappend                # Append, don't overwrite

# Zsh — ~/.zshrc
HISTSIZE=50000
SAVEHIST=50000
HISTFILE=~/.zsh_history
setopt SHARE_HISTORY               # Share across sessions
setopt HIST_IGNORE_ALL_DUPS        # Remove older duplicates
setopt HIST_REDUCE_BLANKS          # Clean up whitespace
setopt HIST_VERIFY                 # Show before executing ! commands
setopt INC_APPEND_HISTORY          # Write immediately, not on exit

# Fish — managed automatically, configure with:
set -g fish_history_max_count 50000
```

### History Search Techniques

```bash
# Reverse search (all shells)
Ctrl+R  # Type to search backward through history

# History expansion (Bash/Zsh)
!!       # Repeat last command
!$       # Last argument of previous command
!^       # First argument of previous command
!*       # All arguments of previous command
!-2      # Command before last
!cmd     # Most recent command starting with 'cmd'
^old^new # Replace 'old' with 'new' in last command

# Examples
sudo !!          # Re-run last command with sudo
vim !$           # Open the last argument in vim
mkdir /some/path && cd !$   # Create and cd into directory
```

---

## Job Control

### Managing Background Processes

```bash
# Run command in background
long-running-command &

# Suspend foreground process
Ctrl+Z

# Resume in foreground
fg

# Resume in background
bg

# List jobs
jobs
jobs -l  # Include PIDs

# Bring specific job to foreground
fg %1     # Job number 1
fg %2     # Job number 2

# Kill a background job
kill %1

# Disown a job (detach from shell — keeps running after shell exit)
disown %1
disown -a  # Disown all jobs

# Run command immune to hangups (survives shell exit)
nohup long-running-command &
```

---

## Next Steps

- **Learn tmux** → [02-TMUX.md](02-TMUX.md) — Terminal multiplexer for managing multiple sessions, windows, and panes
- **Set up Neovim** → [03-NEOVIM.md](03-NEOVIM.md) — Terminal-based editor with IDE features
- **Platform setup** → [04-WINDOWS-TERMINAL.md](04-WINDOWS-TERMINAL.md), [05-LINUX-TERMINAL.md](05-LINUX-TERMINAL.md), or [06-MACOS-TERMINAL.md](06-MACOS-TERMINAL.md)
- **Modern CLI tools** → [07-PRODUCTIVITY-TOOLS.md](07-PRODUCTIVITY-TOOLS.md) — fzf, ripgrep, bat, eza, and more

---

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2025 | Initial shell fundamentals documentation |
