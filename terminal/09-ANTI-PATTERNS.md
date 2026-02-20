# Terminal Anti-Patterns

A catalogue of the most common terminal workflow mistakes — what they look like, why they are harmful, and exactly how to fix them. Use this document as a configuration review checklist, a team onboarding resource, or a self-assessment guide.

---

## Table of Contents

- [Introduction](#introduction)
- [Anti-Patterns Summary Table](#anti-patterns-summary-table)
- [1. Not Using a Terminal Multiplexer](#1-not-using-a-terminal-multiplexer)
- [2. Not Version-Controlling Dotfiles](#2-not-version-controlling-dotfiles)
- [3. Platform-Specific Configs Without Portability](#3-platform-specific-configs-without-portability)
- [4. Overloaded Shell Startup Files](#4-overloaded-shell-startup-files)
- [5. Not Using Modern CLI Replacements](#5-not-using-modern-cli-replacements)
- [6. Ignoring Keyboard-Driven Workflows](#6-ignoring-keyboard-driven-workflows)
- [7. Poor Font and Colour Choices](#7-poor-font-and-colour-choices)
- [8. Not Using SSH Config Properly](#8-not-using-ssh-config-properly)
- [9. Not Customising the Shell Prompt](#9-not-customising-the-shell-prompt)
- [10. Manual Repetitive Tasks](#10-manual-repetitive-tasks)
- [11. Ignoring Shell History](#11-ignoring-shell-history)
- [12. Running Everything in a Single Terminal Window](#12-running-everything-in-a-single-terminal-window)
- [Quick Reference Checklist](#quick-reference-checklist)
- [Next Steps](#next-steps)
- [Version History](#version-history)

---

## Introduction

### Why Anti-Patterns Matter

Terminal anti-patterns are habits that seem harmless individually but compound into significant productivity losses and fragile workflows. Engineers who spend 4–8 hours daily in the terminal lose hundreds of hours per year to suboptimal practices.

The patterns documented here represent real inefficiencies observed across engineering teams. Each one is:

- **Common** — most engineers fall into at least a few of these
- **Costly** — the cumulative time and frustration are substantial
- **Fixable** — each has a well-understood better approach

### How to Use This Document

1. **Self-assessment**: Read through each anti-pattern and honestly evaluate your current workflow
2. **Prioritise**: Fix the highest-severity items first (🔴 Critical)
3. **Incremental improvement**: Don't try to fix everything at once — adopt one change per week
4. **Team onboarding**: Share with new engineers to establish good habits early
5. **Periodic review**: Revisit quarterly to check for regression

---

## Anti-Patterns Summary Table

| # | Anti-Pattern | Category | Severity |
|---|-------------|----------|----------|
| 1 | Not using a terminal multiplexer | Workflow | 🔴 Critical |
| 2 | Not version-controlling dotfiles | Configuration | 🔴 Critical |
| 3 | Platform-specific configs without portability | Configuration | 🟠 High |
| 4 | Overloaded shell startup files | Performance | 🟠 High |
| 5 | Not using modern CLI replacements | Productivity | 🟠 High |
| 6 | Ignoring keyboard-driven workflows | Efficiency | 🟠 High |
| 7 | Poor font and colour choices | Readability | 🟡 Medium |
| 8 | Not using SSH config properly | Workflow | 🟡 Medium |
| 9 | Not customising the shell prompt | Awareness | 🟡 Medium |
| 10 | Manual repetitive tasks | Automation | 🟡 Medium |
| 11 | Ignoring shell history | Efficiency | 🟡 Medium |
| 12 | Running everything in a single terminal window | Organisation | 🟡 Medium |

---

## 1. Not Using a Terminal Multiplexer

### 🔴 Severity: Critical

### The Problem

Opening multiple terminal windows or tabs, losing running processes when SSH disconnects, and having no way to persist terminal layouts across restarts.

### What It Looks Like

```
❌ 12 terminal windows open, scattered across virtual desktops
❌ SSH session dies, long-running process is gone
❌ Every morning: open terminals, cd to project, resize windows, re-run servers
❌ Can't quickly show logs alongside code alongside shell
```

### Why It's Harmful

- **Lost work**: SSH disconnections kill running processes
- **Context switching overhead**: Finding the right terminal window takes time
- **No reproducibility**: Layouts are rebuilt from scratch every session
- **Poor visibility**: Can't see related information simultaneously

### The Fix

Use **tmux** (see [02-TMUX.md](02-TMUX.md)):

```bash
# Install tmux
brew install tmux  # macOS
sudo apt install tmux  # Linux

# Start a named session
tmux new -s myproject

# Detach (session persists): Ctrl+b, then d
# Reattach: tmux attach -t myproject

# Use tmux-resurrect + tmux-continuum for automatic persistence
```

---

## 2. Not Version-Controlling Dotfiles

### 🔴 Severity: Critical

### The Problem

Configuration files exist only on one machine. When you get a new laptop, set up a server, or your disk fails, everything is gone.

### What It Looks Like

```
❌ "I spent 3 hours setting up my terminal on the new laptop"
❌ "My .zshrc was on the old machine, I lost all my aliases"
❌ "I can't remember how I configured my tmux"
❌ Configuration changes are never tracked — you can't undo a bad change
```

### Why It's Harmful

- **No backup**: Hardware failure means complete loss
- **No reproducibility**: Can't replicate your environment on a new machine
- **No audit trail**: Can't see what changed when something breaks
- **No sharing**: Can't share working configurations with teammates

### The Fix

Use chezmoi, GNU Stow, or a bare Git repo (see [08-BEST-PRACTICES.md](08-BEST-PRACTICES.md#dotfile-management)):

```bash
# Simplest approach: bare Git repo
git init --bare "$HOME/.dotfiles"
alias dotfiles='git --git-dir=$HOME/.dotfiles --work-tree=$HOME'
dotfiles add ~/.zshrc ~/.tmux.conf ~/.config/nvim/init.lua
dotfiles commit -m "Initial dotfiles"
dotfiles remote add origin git@github.com:username/dotfiles.git
dotfiles push -u origin main
```

---

## 3. Platform-Specific Configs Without Portability

### 🟠 Severity: High

### The Problem

Hardcoding macOS-specific commands in configs used on Linux, or vice versa. Configuration breaks when moving between platforms.

### What It Looks Like

```bash
# ❌ Hardcoded macOS path in .zshrc used on both macOS and Linux
eval "$(/opt/homebrew/bin/brew shellenv)"
# Error on Linux: /opt/homebrew/bin/brew: No such file or directory

# ❌ Hardcoded clipboard command
alias clip='pbcopy'
# Fails on Linux (pbcopy doesn't exist)
```

### The Fix

```bash
# ✅ Conditional configuration
if [[ "$(uname)" == "Darwin" ]]; then
    eval "$(/opt/homebrew/bin/brew shellenv)"
    alias clip='pbcopy'
elif [[ "$(uname)" == "Linux" ]]; then
    if command -v wl-copy &>/dev/null; then
        alias clip='wl-copy'
    elif command -v xclip &>/dev/null; then
        alias clip='xclip -selection clipboard'
    fi
fi

# ✅ Conditional tool loading
command -v eza &>/dev/null && alias ls='eza --icons' || alias ls='ls --color=auto'
```

---

## 4. Overloaded Shell Startup Files

### 🟠 Severity: High

### The Problem

Putting everything in one massive `.zshrc` or `.bashrc` file, loading dozens of plugins, and not measuring startup time. Shell takes 2–5 seconds to start.

### What It Looks Like

```bash
# ❌ Monolithic .zshrc with everything
source "$ZSH/oh-my-zsh.sh"  # Loading 30+ plugins
nvm use default              # nvm adds 500ms+ to startup
eval "$(rbenv init -)"       # rbenv adds 200ms+
eval "$(pyenv init -)"       # pyenv adds 200ms+
# ... 500 more lines
# Result: 3-second shell startup
```

### Why It's Harmful

- **Slow startup** frustrates and disrupts flow
- **Difficult to maintain**: Hard to find or change specific settings
- **Hidden failures**: Errors buried in hundreds of lines

### The Fix

```bash
# 1. Measure your startup time
time zsh -i -c exit
# Target: < 200ms

# 2. Use lazy loading for slow tools
# Instead of loading nvm at startup:
nvm() {
    unset -f nvm
    export NVM_DIR="$HOME/.nvm"
    [ -s "$NVM_DIR/nvm.sh" ] && source "$NVM_DIR/nvm.sh"
    nvm "$@"
}

# 3. Split config into modules
source ~/.config/shell/path.sh
source ~/.config/shell/aliases.sh
source ~/.config/shell/functions.sh

# 4. Use fast plugin managers (zinit, antidote) instead of Oh My Zsh
# Or limit Oh My Zsh to essential plugins only
plugins=(git z fzf)  # Only what you actually use

# 5. Profile your startup
zmodload zsh/zprof  # Add to top of .zshrc
# Then run: zprof  # at the bottom or in terminal
```

---

## 5. Not Using Modern CLI Replacements

### 🟠 Severity: High

### The Problem

Using default `grep`, `find`, `cat`, and `ls` when faster, more user-friendly alternatives exist.

### What It Looks Like

```bash
# ❌ Slow, unhelpful defaults
grep -r "pattern" .                   # Slow, searches .git/, no colours
find . -name "*.py" -type f          # Verbose syntax, searches .git/
cat long-file.py                     # No syntax highlighting, no line numbers
ls -la                               # No icons, no git status, no tree view
```

### The Fix

```bash
# ✅ Modern replacements (see 07-PRODUCTIVITY-TOOLS.md)
brew install ripgrep fd bat eza fzf zoxide delta

# Replace in .zshrc
alias grep='rg'
alias find='fd'
alias cat='bat --paging=never'
alias ls='eza --icons --group-directories-first'

# Comparison:
# grep -r "TODO" .        →  rg "TODO"                    (10-100× faster)
# find . -name "*.py"     →  fd -e py                     (simpler, faster)
# cat file.py             →  bat file.py                  (syntax highlighting)
# ls -la                  →  eza -la --icons --git        (icons, git status)
```

---

## 6. Ignoring Keyboard-Driven Workflows

### 🟠 Severity: High

### The Problem

Reaching for the mouse to select text, switch windows, or navigate files in the terminal. Each mouse interaction breaks flow.

### What It Looks Like

```
❌ Using mouse to select and copy text from terminal
❌ Clicking terminal tabs instead of keyboard shortcuts
❌ Alt-tabbing between terminal windows instead of tmux panes
❌ Not using vim keybindings in shell or tmux
❌ Not using Caps Lock as Ctrl
```

### The Fix

```bash
# 1. Remap Caps Lock to Ctrl (see 08-BEST-PRACTICES.md)

# 2. Use tmux for all window/pane management
# Ctrl+a + h/j/k/l to navigate panes
# Ctrl+a + c to create windows
# Ctrl+a + 0-9 to switch windows

# 3. Use fzf for fuzzy navigation
# Ctrl+R for history search
# Ctrl+T for file finding
# Alt+C for directory jumping

# 4. Use tmux copy mode instead of mouse selection
# Ctrl+a + [ to enter copy mode
# v to select, y to copy

# 5. Use vim-style navigation in Zsh
bindkey -v  # Vi mode
# Or: keep emacs mode but learn Ctrl+A/E/W/U/K shortcuts
```

---

## 7. Poor Font and Colour Choices

### 🟡 Severity: Medium

### The Problem

Using default system fonts that lack icons, using colour schemes with poor contrast, or not using a Nerd Font that modern tools expect.

### What It Looks Like

```
❌ Unicode icons show as □ □ □ (tofu boxes)
❌ Text is hard to read due to low contrast
❌ eza, starship, and lualine show broken icons
❌ Using a proportional font in the terminal
```

### The Fix

```bash
# 1. Install a Nerd Font
brew install --cask font-jetbrains-mono-nerd-font  # macOS
# Or download from https://www.nerdfonts.com/

# 2. Set the font in your terminal emulator
# Alacritty: font.normal.family = "JetBrainsMono Nerd Font"
# iTerm2: Profiles → Text → Font → JetBrainsMono Nerd Font
# Windows Terminal: font.face: "JetBrainsMono Nerd Font"

# 3. Use a proven colour scheme (Catppuccin, Tokyo Night, Dracula)
# Apply to: terminal emulator, Neovim, tmux, bat, delta

# 4. Use 256-colour or true colour terminal
echo $TERM  # Should be xterm-256color or similar
tput colors  # Should return 256
```

---

## 8. Not Using SSH Config Properly

### 🟡 Severity: Medium

### The Problem

Typing full SSH commands with usernames, ports, and key paths every time. Not using connection multiplexing or jump hosts.

### What It Looks Like

```bash
# ❌ Every time you connect
ssh -i ~/.ssh/id_ed25519_work -p 2222 deploy@production.example.com
ssh -i ~/.ssh/id_ed25519_work -p 2222 deploy@staging.example.com
# Repeatedly typing the same long commands
```

### The Fix

```
# ✅ ~/.ssh/config

Host *
    AddKeysToAgent yes
    IdentitiesOnly yes
    ServerAliveInterval 60
    ServerAliveCountMax 3
    # Connection multiplexing (reuse connections)
    ControlMaster auto
    ControlPath ~/.ssh/sockets/%r@%h-%p
    ControlPersist 600

Host prod
    HostName production.example.com
    User deploy
    Port 2222
    IdentityFile ~/.ssh/id_ed25519_work

Host staging
    HostName staging.example.com
    User deploy
    Port 2222
    IdentityFile ~/.ssh/id_ed25519_work

# Now just: ssh prod, ssh staging
```

```bash
# Create socket directory
mkdir -p ~/.ssh/sockets
```

---

## 9. Not Customising the Shell Prompt

### 🟡 Severity: Medium

### The Problem

Using the default shell prompt that provides no useful context. Not knowing which Git branch you're on, what directory you're in, or whether the last command succeeded.

### What It Looks Like

```
# ❌ Default Bash prompt
user@hostname:~$
# No git branch, no error indicator, no environment info
```

### The Fix

```bash
# ✅ Install Starship (cross-shell, fast, informative)
curl -sS https://starship.rs/install.sh | sh
eval "$(starship init zsh)"

# Or use Powerlevel10k for Zsh:
# See 01-SHELL-FUNDAMENTALS.md for configuration

# A good prompt shows:
# ✓ Current directory (truncated)
# ✓ Git branch and status
# ✓ Last command exit code
# ✓ Language versions (when in a project)
# ✓ Kubernetes context (when relevant)
# ✓ Execution time for slow commands
```

---

## 10. Manual Repetitive Tasks

### 🟡 Severity: Medium

### The Problem

Performing the same sequence of commands repeatedly instead of creating aliases, functions, or scripts.

### What It Looks Like

```bash
# ❌ Every day
cd ~/projects/api
git pull
docker compose up -d
npm run test
# Typing this out every time
```

### The Fix

```bash
# ✅ Create a function
dev-start() {
    cd ~/projects/api &&
    git pull &&
    docker compose up -d &&
    npm run test
}

# ✅ Or a tmux startup script (see 02-TMUX.md)
# ✅ Or a Makefile for project-specific commands
# ✅ Or shell aliases for common sequences

alias gs='git status'
alias gco='git checkout'
alias dc='docker compose'
```

---

## 11. Ignoring Shell History

### 🟡 Severity: Medium

### The Problem

Not configuring history properly, losing history across sessions, and not using history search effectively.

### What It Looks Like

```bash
# ❌ Default Bash: 500 entries, not shared across terminals
# ❌ Pressing up arrow 47 times to find a command
# ❌ History lost when multiple terminals are closed
```

### The Fix

```bash
# ✅ Increase history size and share across sessions
# Zsh:
HISTSIZE=50000
SAVEHIST=50000
setopt SHARE_HISTORY
setopt HIST_IGNORE_ALL_DUPS

# ✅ Use fzf for history search
# Ctrl+R → fuzzy search through full history

# ✅ Use history expansion
!!          # Repeat last command
sudo !!     # Re-run last command with sudo
!$          # Last argument of previous command
```

---

## 12. Running Everything in a Single Terminal Window

### 🟡 Severity: Medium

### The Problem

Running dev server, watching logs, editing files, and running tests all in the same terminal by stopping and starting processes.

### What It Looks Like

```bash
# ❌ Cycle of starting and stopping
npm run dev
# Ctrl+C to stop
vim src/app.js
# :q to exit vim
npm run test
# Ctrl+C
npm run dev  # Start again
# Repeat all day
```

### The Fix

Use tmux panes for a persistent layout:

```
┌─────────────────────────────┬──────────────────┐
│                             │                  │
│     Neovim                  │   Dev Server     │
│     (editor)                │   npm run dev    │
│                             │                  │
│                             ├──────────────────┤
│                             │                  │
│                             │   Shell          │
│                             │   (tests, git)   │
│                             │                  │
└─────────────────────────────┴──────────────────┘

All running simultaneously, switch with Ctrl+h/j/k/l
```

---

## Quick Reference Checklist

| # | Check | Status |
|---|-------|--------|
| 1 | Using tmux (or equivalent multiplexer) | ☐ |
| 2 | Dotfiles in version control | ☐ |
| 3 | Cross-platform conditionals in configs | ☐ |
| 4 | Shell starts in < 200ms | ☐ |
| 5 | Using modern CLI tools (rg, fd, bat, eza) | ☐ |
| 6 | Keyboard-driven navigation (minimal mouse) | ☐ |
| 7 | Nerd Font installed, readable colour scheme | ☐ |
| 8 | SSH config with aliases and multiplexing | ☐ |
| 9 | Informative shell prompt (git, errors, context) | ☐ |
| 10 | Common tasks automated (aliases, functions, scripts) | ☐ |
| 11 | History configured and searchable (fzf + large history) | ☐ |
| 12 | Using tmux panes for simultaneous workflows | ☐ |

---

## Next Steps

- **Fix critical items first** → [02-TMUX.md](02-TMUX.md) (multiplexer) and [08-BEST-PRACTICES.md](08-BEST-PRACTICES.md) (dotfiles)
- **Upgrade tools** → [07-PRODUCTIVITY-TOOLS.md](07-PRODUCTIVITY-TOOLS.md) — Install modern CLI replacements
- **Optimise shell** → [01-SHELL-FUNDAMENTALS.md](01-SHELL-FUNDAMENTALS.md) — Fix startup time and history
- **Follow the learning path** → [LEARNING-PATH.md](LEARNING-PATH.md) — Structured improvement plan

---

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2025 | Initial terminal anti-patterns documentation |
