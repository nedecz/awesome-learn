# tmux — Terminal Multiplexer

A comprehensive guide to tmux — covering installation, core concepts (sessions, windows, panes), key bindings, configuration, plugins, session management, copy mode, pair programming, scripting, and session persistence with resurrect and continuum.

---

## Table of Contents

1. [Overview](#overview)
2. [Installation](#installation)
3. [Core Concepts](#core-concepts)
4. [Key Bindings](#key-bindings)
5. [Configuration](#configuration)
6. [Session Management](#session-management)
7. [Window and Pane Management](#window-and-pane-management)
8. [Copy Mode](#copy-mode)
9. [Plugins (TPM)](#plugins-tpm)
10. [Session Persistence](#session-persistence)
11. [Pair Programming with tmux](#pair-programming-with-tmux)
12. [Scripting tmux](#scripting-tmux)
13. [Troubleshooting](#troubleshooting)
14. [Next Steps](#next-steps)
15. [Version History](#version-history)

---

## Overview

**tmux** (terminal multiplexer) is a program that lets you run multiple terminal sessions inside a single terminal window. Sessions persist on the server even when you disconnect, making tmux essential for remote work, long-running processes, and organised terminal workflows.

### Why tmux

| Problem | tmux Solution |
|---------|--------------|
| SSH disconnects kill running processes | tmux sessions persist; reattach after reconnecting |
| Switching between terminal windows is slow | tmux panes and windows keep everything in one view |
| No way to share terminal sessions | tmux socket sharing enables pair programming |
| Setting up complex layouts every time | tmux scripts and plugins restore entire layouts |
| Can't see logs, code, and shell simultaneously | tmux panes give a dashboard-like terminal layout |

### Target Audience

- **Developers** who work in the terminal and want persistent, organised sessions
- **DevOps Engineers / SREs** managing remote servers via SSH
- **Anyone** who wants to split their terminal into multiple panes and windows

---

## Installation

```bash
# macOS
brew install tmux

# Debian/Ubuntu
sudo apt update && sudo apt install tmux

# Fedora
sudo dnf install tmux

# Arch Linux
sudo pacman -S tmux

# From source (latest version)
git clone https://github.com/tmux/tmux.git
cd tmux
sh autogen.sh
./configure && make
sudo make install

# Verify installation
tmux -V
# tmux 3.4
```

---

## Core Concepts

tmux organises your terminal work into a hierarchy:

```
Server
 └── Session (named group of windows — e.g., "project-api")
      ├── Window 0: "editor"  (like a tab)
      │    ├── Pane 0: nvim (top)
      │    └── Pane 1: shell (bottom)
      ├── Window 1: "server"
      │    └── Pane 0: npm run dev
      └── Window 2: "git"
           └── Pane 0: lazygit
```

| Concept | Description | Analogy |
|---------|-------------|---------|
| **Server** | Background process managing all sessions | The tmux daemon |
| **Session** | Named group of windows; persists after detaching | A workspace or project |
| **Window** | Full-screen view within a session | A tab in your browser |
| **Pane** | A split within a window | A split view in an editor |
| **Prefix key** | Key combination that tells tmux "the next key is a command" | Default: `Ctrl+b` |

### Starting tmux

```bash
# Start a new unnamed session
tmux

# Start a new named session
tmux new-session -s work

# Shorter form
tmux new -s work

# Start detached (in the background)
tmux new -d -s background-job

# Attach to existing session
tmux attach -t work
tmux a -t work

# List sessions
tmux ls

# Kill a session
tmux kill-session -t work

# Kill all sessions
tmux kill-server
```

---

## Key Bindings

All tmux commands are triggered by the **prefix key** (default: `Ctrl+b`) followed by a command key. The notation `<prefix> d` means "press `Ctrl+b`, release, then press `d`".

### Session Commands

| Key Binding | Action |
|-------------|--------|
| `<prefix> d` | Detach from session |
| `<prefix> s` | List and switch sessions |
| `<prefix> $` | Rename current session |
| `<prefix> (` | Switch to previous session |
| `<prefix> )` | Switch to next session |

### Window Commands

| Key Binding | Action |
|-------------|--------|
| `<prefix> c` | Create new window |
| `<prefix> ,` | Rename current window |
| `<prefix> n` | Next window |
| `<prefix> p` | Previous window |
| `<prefix> 0-9` | Switch to window by number |
| `<prefix> w` | List all windows (interactive) |
| `<prefix> &` | Close current window (with confirmation) |
| `<prefix> l` | Toggle to last active window |
| `<prefix> f` | Find window by name |

### Pane Commands

| Key Binding | Action |
|-------------|--------|
| `<prefix> %` | Split pane vertically (left/right) |
| `<prefix> "` | Split pane horizontally (top/bottom) |
| `<prefix> o` | Cycle through panes |
| `<prefix> ;` | Toggle to last active pane |
| `<prefix> x` | Close current pane (with confirmation) |
| `<prefix> z` | Toggle pane zoom (fullscreen) |
| `<prefix> {` | Swap pane with previous |
| `<prefix> }` | Swap pane with next |
| `<prefix> q` | Show pane numbers (press number to select) |
| `<prefix> Space` | Cycle through pane layouts |
| `<prefix> Arrow` | Navigate to pane in direction |
| `<prefix> Ctrl+Arrow` | Resize pane in direction |

### Other Commands

| Key Binding | Action |
|-------------|--------|
| `<prefix> :` | Enter command mode |
| `<prefix> ?` | List all key bindings |
| `<prefix> t` | Show clock |
| `<prefix> [` | Enter copy mode |
| `<prefix> ]` | Paste from tmux buffer |

---

## Configuration

tmux is configured via `~/.tmux.conf`. Changes are applied by reloading the config:

```bash
# Reload config from within tmux
tmux source-file ~/.tmux.conf

# Or bind a reload key (add to .tmux.conf)
bind r source-file ~/.tmux.conf \; display "Config reloaded!"
```

### Complete Configuration Example

```bash
# ~/.tmux.conf — tmux configuration

# ─── General ──────────────────────────────────────────────────────
# Set prefix to Ctrl+a (more ergonomic than Ctrl+b)
unbind C-b
set -g prefix C-a
bind C-a send-prefix

# Enable true colour support
set -g default-terminal "tmux-256color"
set -ag terminal-overrides ",xterm-256color:RGB"

# Enable mouse support
set -g mouse on

# Start window and pane numbering at 1 (not 0)
set -g base-index 1
setw -g pane-base-index 1

# Renumber windows when one is closed
set -g renumber-windows on

# Increase scrollback buffer
set -g history-limit 50000

# Reduce escape time (important for Neovim)
set -sg escape-time 0

# Enable focus events (for Neovim autoread)
set -g focus-events on

# Display tmux messages for 4 seconds
set -g display-time 4000

# Refresh status bar every 5 seconds
set -g status-interval 5

# ─── Key Bindings ─────────────────────────────────────────────────
# Reload configuration
bind r source-file ~/.tmux.conf \; display "Config reloaded!"

# Better split keys (more intuitive)
bind | split-window -h -c "#{pane_current_path}"
bind - split-window -v -c "#{pane_current_path}"
unbind '"'
unbind %

# New window in current path
bind c new-window -c "#{pane_current_path}"

# Navigate panes with vim-style keys
bind h select-pane -L
bind j select-pane -D
bind k select-pane -U
bind l select-pane -R

# Resize panes with vim-style keys
bind -r H resize-pane -L 5
bind -r J resize-pane -D 5
bind -r K resize-pane -U 5
bind -r L resize-pane -R 5

# Quick window switching
bind -r C-h previous-window
bind -r C-l next-window

# Swap windows
bind -r "<" swap-window -d -t -1
bind -r ">" swap-window -d -t +1

# Toggle synchronise panes (send same input to all panes)
bind S setw synchronize-panes \; display "Sync #{?synchronize-panes,ON,OFF}"

# ─── Copy Mode (Vi Style) ────────────────────────────────────────
setw -g mode-keys vi

# Enter copy mode
bind Enter copy-mode

# Vi-style selection and copy
bind -T copy-mode-vi v send-keys -X begin-selection
bind -T copy-mode-vi y send-keys -X copy-pipe-and-cancel "pbcopy"  # macOS
# For Linux: "xclip -selection clipboard" or "xsel --clipboard"
bind -T copy-mode-vi C-v send-keys -X rectangle-toggle
bind -T copy-mode-vi Escape send-keys -X cancel

# ─── Appearance ───────────────────────────────────────────────────
# Status bar
set -g status-position top
set -g status-style "bg=#1e1e2e,fg=#cdd6f4"

# Left status
set -g status-left-length 40
set -g status-left "#[bg=#89b4fa,fg=#1e1e2e,bold] #S #[default] "

# Right status
set -g status-right-length 80
set -g status-right "#[fg=#a6adc8] %Y-%m-%d %H:%M #[bg=#89b4fa,fg=#1e1e2e,bold] #H "

# Window status
setw -g window-status-format " #I:#W "
setw -g window-status-current-format "#[bg=#cba6f7,fg=#1e1e2e,bold] #I:#W "

# Pane borders
set -g pane-border-style "fg=#45475a"
set -g pane-active-border-style "fg=#89b4fa"

# Message style
set -g message-style "bg=#89b4fa,fg=#1e1e2e,bold"
```

---

## Session Management

### Named Sessions for Projects

```bash
# Create project-specific sessions
tmux new -s api -d
tmux new -s frontend -d
tmux new -s database -d

# Switch between sessions
tmux switch-client -t api
# Or use <prefix> s to interactively select

# List sessions
tmux ls
# api: 3 windows (created Mon Jan  6 10:00:00 2025)
# frontend: 2 windows (created Mon Jan  6 10:05:00 2025)
# database: 1 windows (created Mon Jan  6 10:10:00 2025)
```

### Session Groups

```bash
# Create a session group (shared windows, independent current window)
tmux new-session -s main
tmux new-session -t main -s secondary
# Both sessions share windows, but each can be on a different window
```

---

## Window and Pane Management

### Common Layouts

```bash
# Predefined layouts (cycle with <prefix> Space)
# even-horizontal:  All panes side by side
# even-vertical:    All panes stacked
# main-horizontal:  One large pane on top, smaller ones below
# main-vertical:    One large pane on left, smaller ones on right
# tiled:            All panes equal size

# Apply a specific layout
tmux select-layout main-vertical
tmux select-layout tiled

# Set main pane size for main-vertical/main-horizontal layouts
tmux set-window-option main-pane-width 60%
tmux set-window-option main-pane-height 70%
```

### Practical Layout: IDE-Style

```bash
# Create an IDE-like layout
# ┌────────────────┬──────────┐
# │                │          │
# │    Editor      │  Files   │
# │    (Neovim)    │          │
# │                ├──────────┤
# │                │  Shell   │
# │                │          │
# └────────────────┴──────────┘

tmux new-session -s dev -d
tmux send-keys -t dev "nvim ." Enter         # Main pane: editor
tmux split-window -t dev -h -p 35            # Right pane (35% width)
tmux send-keys -t dev "ls -la" Enter
tmux split-window -t dev -v -p 50            # Split right pane
tmux select-pane -t dev -L                   # Focus editor pane
tmux attach -t dev
```

---

## Copy Mode

tmux's copy mode lets you scroll through output, search, and copy text — essential when using tmux over SSH or when you need to copy text that's no longer visible.

### Copy Mode with Vi Keys

```
Enter copy mode:  <prefix> [  (or <prefix> Enter with our config)

Navigation (Vi mode):
  h, j, k, l     Move cursor
  w, b, e         Word movement
  0, $            Start/end of line
  g, G            Top/bottom of buffer
  Ctrl+u, Ctrl+d  Half-page up/down
  /pattern        Search forward
  ?pattern        Search backward
  n, N            Next/previous search match

Selection and Copy:
  Space or v      Start selection
  Enter or y      Copy selection and exit
  Ctrl+v          Toggle rectangle selection
  Escape          Cancel and exit copy mode

Paste:
  <prefix> ]      Paste from tmux buffer
```

### Clipboard Integration

```bash
# macOS — copy to system clipboard
bind -T copy-mode-vi y send-keys -X copy-pipe-and-cancel "pbcopy"

# Linux (X11) — copy to system clipboard
bind -T copy-mode-vi y send-keys -X copy-pipe-and-cancel "xclip -selection clipboard"

# Linux (Wayland) — copy to system clipboard
bind -T copy-mode-vi y send-keys -X copy-pipe-and-cancel "wl-copy"

# WSL — copy to Windows clipboard
bind -T copy-mode-vi y send-keys -X copy-pipe-and-cancel "clip.exe"

# Cross-platform using OSC 52 (if terminal supports it)
set -g set-clipboard on
```

---

## Plugins (TPM)

**TPM** (Tmux Plugin Manager) is the standard way to manage tmux plugins.

### Installing TPM

```bash
# Clone TPM
git clone https://github.com/tmux-plugins/tpm ~/.tmux/plugins/tpm

# Add to ~/.tmux.conf (at the very bottom)
# List of plugins
set -g @plugin 'tmux-plugins/tpm'
set -g @plugin 'tmux-plugins/tmux-sensible'

# Initialize TPM (keep this at the very bottom of .tmux.conf)
run '~/.tmux/plugins/tpm/tpm'
```

After adding plugins to `.tmux.conf`:
- **Install plugins**: `<prefix> I` (capital I)
- **Update plugins**: `<prefix> U`
- **Remove plugins**: `<prefix> Alt+u`

### Essential Plugins

```bash
# ~/.tmux.conf — plugin list

# Plugin manager
set -g @plugin 'tmux-plugins/tpm'

# Sensible defaults
set -g @plugin 'tmux-plugins/tmux-sensible'

# Save and restore sessions across restarts
set -g @plugin 'tmux-plugins/tmux-resurrect'

# Automatic session saving (every 15 minutes)
set -g @plugin 'tmux-plugins/tmux-continuum'
set -g @continuum-restore 'on'
set -g @continuum-save-interval '15'

# Better copy/paste integration
set -g @plugin 'tmux-plugins/tmux-yank'

# Navigate between tmux panes and Neovim splits seamlessly
set -g @plugin 'christoomey/vim-tmux-navigator'

# Regex search in copy mode
set -g @plugin 'tmux-plugins/tmux-copycat'

# Open highlighted file/URL from copy mode
set -g @plugin 'tmux-plugins/tmux-open'

# Show battery, CPU, and other system info in status bar
set -g @plugin 'tmux-plugins/tmux-cpu'

# Quick session switching with fzf
set -g @plugin 'sainnhe/tmux-fzf'

# Initialize TPM (MUST be at the very bottom)
run '~/.tmux/plugins/tpm/tpm'
```

---

## Session Persistence

### tmux-resurrect

Saves and restores tmux environment (sessions, windows, panes, and running programs) across system restarts:

```bash
# Add to ~/.tmux.conf
set -g @plugin 'tmux-plugins/tmux-resurrect'

# Save pane contents
set -g @resurrect-capture-pane-contents 'on'

# Restore Neovim sessions (requires vim-obsession or similar)
set -g @resurrect-strategy-nvim 'session'

# Save and restore additional programs
set -g @resurrect-processes 'ssh psql mysql'

# Key bindings:
# <prefix> Ctrl+s — Save session
# <prefix> Ctrl+r — Restore session
```

### tmux-continuum

Automatically saves sessions at regular intervals and optionally restores on tmux start:

```bash
# Add to ~/.tmux.conf
set -g @plugin 'tmux-plugins/tmux-continuum'

# Auto-save every 15 minutes
set -g @continuum-save-interval '15'

# Auto-restore when tmux starts
set -g @continuum-restore 'on'

# Show continuum status in status bar
set -g status-right 'Continuum: #{continuum_status}'
```

---

## Pair Programming with tmux

### Shared Session (Same View)

Both users see and control the same session:

```bash
# Host starts a session
tmux new -s pair

# Guest connects to the same session (via SSH)
ssh user@host
tmux attach -t pair

# Both users now share the same view and can type simultaneously
# Limitation: both users see the same window
```

### Shared Session (Independent Views)

Both users share windows but can be on different ones:

```bash
# Host starts a session
tmux new -s pair

# Guest creates a grouped session
ssh user@host
tmux new-session -t pair -s guest

# Now both users share the same windows
# but each can independently switch between windows
```

### Using a Shared Socket

```bash
# Host creates session with a named socket
tmux -S /tmp/pair new -s pair

# Set socket permissions for the guest user
chmod 777 /tmp/pair

# Guest connects via the shared socket
tmux -S /tmp/pair attach -t pair

# For read-only access (guest can observe but not type)
tmux -S /tmp/pair attach -t pair -r
```

---

## Scripting tmux

### Project Startup Script

```bash
#!/usr/bin/env bash
# tmux-dev.sh — Start a development session

SESSION="myproject"
PROJECT_DIR="$HOME/projects/myproject"

# Kill existing session if it exists
tmux kill-session -t "$SESSION" 2>/dev/null

# Create new session (detached, start in project dir)
tmux new-session -d -s "$SESSION" -c "$PROJECT_DIR"

# Window 0: Editor
tmux rename-window -t "$SESSION:0" "editor"
tmux send-keys -t "$SESSION:0" "nvim ." Enter

# Window 1: Server
tmux new-window -t "$SESSION" -n "server" -c "$PROJECT_DIR"
tmux send-keys -t "$SESSION:1" "npm run dev" Enter

# Window 2: Git and shell
tmux new-window -t "$SESSION" -n "git" -c "$PROJECT_DIR"
tmux split-window -t "$SESSION:2" -h -c "$PROJECT_DIR"
tmux send-keys -t "$SESSION:2.0" "git status" Enter
tmux send-keys -t "$SESSION:2.1" "git log --oneline -20" Enter

# Window 3: Database
tmux new-window -t "$SESSION" -n "db" -c "$PROJECT_DIR"

# Window 4: Logs
tmux new-window -t "$SESSION" -n "logs" -c "$PROJECT_DIR"
tmux send-keys -t "$SESSION:4" "tail -f /var/log/app.log" Enter

# Select the editor window and attach
tmux select-window -t "$SESSION:0"
tmux attach -t "$SESSION"
```

### tmux Commands in Scripts

```bash
# Useful scripting commands
tmux has-session -t mysession 2>/dev/null  # Check if session exists ($? == 0)
tmux list-sessions                          # List all sessions
tmux list-windows -t mysession              # List windows in session
tmux list-panes -t mysession:0              # List panes in window

# Send keys to a specific pane
tmux send-keys -t mysession:0.1 "echo hello" Enter

# Capture pane output to a file
tmux capture-pane -t mysession:0.0 -p > output.txt

# Run a command in a new window
tmux new-window -t mysession "htop"

# Wait for a pane to finish (useful in CI)
tmux wait-for -S channel-name
```

---

## Troubleshooting

### Common Issues

| Issue | Cause | Fix |
|-------|-------|-----|
| Colours look wrong | TERM not set correctly | Add `set -g default-terminal "tmux-256color"` to `.tmux.conf` |
| Escape key is slow (Neovim) | tmux escape time | Add `set -sg escape-time 0` to `.tmux.conf` |
| Mouse scrolling doesn't work | Mouse mode disabled | Add `set -g mouse on` to `.tmux.conf` |
| Clipboard doesn't work | No clipboard integration | Configure `tmux-yank` plugin or use OSC 52 |
| Status bar not updating | Low refresh interval | Add `set -g status-interval 5` to `.tmux.conf` |
| tmux-256color unknown | Missing terminfo entry | Run `tic -x` with tmux terminfo, or use `screen-256color` |
| Nerd Font icons broken | Font not set in emulator | Set a Nerd Font in your terminal emulator settings |

### Debugging tmux

```bash
# Show all tmux options and their values
tmux show-options -g          # Global options
tmux show-options -w          # Window options
tmux show-options -s          # Server options

# Show active key bindings
tmux list-keys

# Show tmux info (useful for debugging)
tmux info

# Check which config file tmux is using
tmux display-message -p "Config: #{config_files}"

# Verbose logging
tmux -vv new-session 2>/tmp/tmux-debug.log
```

---

## Next Steps

- **Set up Neovim** → [03-NEOVIM.md](03-NEOVIM.md) — Configure Neovim to work seamlessly with tmux using vim-tmux-navigator
- **Productivity tools** → [07-PRODUCTIVITY-TOOLS.md](07-PRODUCTIVITY-TOOLS.md) — fzf integration with tmux for session switching
- **Best practices** → [08-BEST-PRACTICES.md](08-BEST-PRACTICES.md) — Dotfile management and portable tmux configurations
- **Shell setup** → [01-SHELL-FUNDAMENTALS.md](01-SHELL-FUNDAMENTALS.md) — Configure your shell to work optimally inside tmux

---

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2025 | Initial tmux documentation |
