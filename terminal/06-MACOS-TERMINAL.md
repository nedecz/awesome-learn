# macOS Terminal Setup

A comprehensive guide to setting up a productive terminal environment on macOS — covering Terminal.app, iTerm2, Alacritty, WezTerm, Homebrew, macOS-specific shell configuration, key binding considerations, font configuration, and integration with macOS services.

---

## Table of Contents

1. [Overview](#overview)
2. [Terminal Emulator Comparison](#terminal-emulator-comparison)
3. [Terminal.app](#terminalapp)
4. [iTerm2](#iterm2)
5. [Alacritty on macOS](#alacritty-on-macos)
6. [WezTerm on macOS](#wezterm-on-macos)
7. [Homebrew Setup](#homebrew-setup)
8. [macOS Shell Configuration](#macos-shell-configuration)
9. [Key Binding Considerations](#key-binding-considerations)
10. [Font Configuration](#font-configuration)
11. [macOS Integration](#macos-integration)
12. [Spotlight and Alfred Integration](#spotlight-and-alfred-integration)
13. [Next Steps](#next-steps)
14. [Version History](#version-history)

---

## Overview

macOS provides a Unix-based environment with Zsh as the default shell since Catalina (10.15). Combined with Homebrew for package management and a choice of excellent terminal emulators, macOS offers a first-class terminal experience. This guide covers macOS-specific configuration that differs from Linux setups.

### Target Audience

- **macOS developers** setting up or optimising their terminal environment
- **Engineers switching to macOS** from Linux or Windows
- **iOS/Swift developers** who also work with command-line tools

---

## Terminal Emulator Comparison

| Emulator | GPU Accel | Native macOS | Tabs/Splits | Config Format | Best For |
|----------|-----------|-------------|-------------|---------------|----------|
| **Terminal.app** | ❌ | ✅ (built-in) | Tabs only | GUI | Quick tasks, no setup needed |
| **iTerm2** | ❌ | ✅ | Both | GUI + JSON | Feature-rich, most popular on macOS |
| **Alacritty** | ✅ (Metal) | Partial | ❌ (use tmux) | TOML | Speed and minimalism |
| **WezTerm** | ✅ (Metal) | Partial | Both | Lua | Cross-platform, programmable |
| **kitty** | ✅ (OpenGL) | Partial | Both | kitty.conf | Power users, image support |

---

## Terminal.app

macOS includes **Terminal.app** (`/Applications/Utilities/Terminal.app`), a capable terminal emulator that requires zero setup.

### Configuration

Terminal.app is configured through Preferences (`Cmd+,`):

```
Terminal → Preferences:
  ├── General
  │   ├── Default shell: /bin/zsh
  │   └── On startup, open: Default Profile
  ├── Profiles
  │   ├── Text: Font, colours, cursor style
  │   ├── Window: Size, title, background
  │   ├── Tab: Tab bar behaviour
  │   ├── Shell: Startup command, when shell exits
  │   ├── Keyboard: Key mappings
  │   └── Advanced: Terminfo, bell, encoding
  └── Encodings: UTF-8 (ensure this is set)
```

### Importing Colour Schemes

```bash
# Terminal.app uses .terminal files for profiles
# Download a colour scheme (e.g., Catppuccin)
curl -LO https://raw.githubusercontent.com/catppuccin/Terminal.app/main/Catppuccin-Mocha.terminal

# Double-click to import, then set as default in Preferences → Profiles
open Catppuccin-Mocha.terminal
```

### When to Upgrade from Terminal.app

Consider switching if you need:
- Split panes (Terminal.app only has tabs)
- GPU-accelerated rendering
- Inline image display
- Advanced key binding customisation
- tmux integration features
- Extensive scripting capabilities

---

## iTerm2

**iTerm2** is the most popular third-party terminal emulator for macOS. It offers an extensive feature set including split panes, search, autocomplete, image display, and deep macOS integration.

### Installation

```bash
brew install --cask iterm2

# Or download from https://iterm2.com/
```

### Key Features

| Feature | Description |
|---------|-------------|
| **Split panes** | Horizontal and vertical splits within a tab |
| **Search** | Regex and plain-text search with highlighting |
| **Autocomplete** | Suggests completions from terminal history |
| **Instant replay** | Rewind terminal output like a time machine |
| **Shell integration** | Track command history, directories, uploads/downloads |
| **Triggers** | Run actions based on text patterns in output |
| **Profiles** | Multiple configurations for different tasks |
| **Status bar** | Configurable bottom bar with system info |
| **Python API** | Script iTerm2 with Python for automation |

### Recommended Settings

```
iTerm2 → Preferences (Cmd+,):

General:
  ├── Closing: Confirm quit only when multiple tabs
  ├── Selection: Applications in terminal may access clipboard ✅
  └── Window: Native full screen windows ❌ (faster non-native)

Appearance:
  ├── Theme: Minimal
  ├── Tab bar location: Top
  └── Status bar: Enable and configure

Profiles → Default:
  ├── General:
  │   └── Working Directory: Reuse previous session's directory
  ├── Colors:
  │   └── Color Presets: Import Catppuccin Mocha
  ├── Text:
  │   ├── Font: JetBrainsMono Nerd Font, 12pt
  │   ├── Use ligatures: ✅
  │   └── Anti-aliased: ✅
  ├── Window:
  │   ├── Transparency: 5%
  │   └── Columns: 120, Rows: 35
  ├── Terminal:
  │   └── Scrollback lines: 10000
  └── Keys:
      ├── Key Mappings: Natural Text Editing preset
      └── Left Option key: Esc+ (CRITICAL for Alt key bindings)

Keys → Hotkey:
  └── Show/hide iTerm2 with a system-wide hotkey: Ctrl+` (or your preference)
```

### iTerm2 Shell Integration

```bash
# Install shell integration (enables marks, command status, etc.)
curl -L https://iterm2.com/shell_integration/zsh -o ~/.iterm2_shell_integration.zsh
source ~/.iterm2_shell_integration.zsh

# Add to ~/.zshrc for persistence
echo 'test -e "${HOME}/.iterm2_shell_integration.zsh" && source "${HOME}/.iterm2_shell_integration.zsh"' >> ~/.zshrc

# Features enabled by shell integration:
# - Click on a command to select its output
# - Right-click → "Select Command" to copy command output
# - Toolbelt → Command History for searchable history
# - Toolbelt → Recent Directories for quick navigation
# - imgcat: display images inline
# - it2dl/it2ul: download/upload files over SSH
```

### iTerm2 Profiles for Different Tasks

```
Create profiles for different contexts:

Profile: "Default" — everyday shell work
  Font: JetBrainsMono 12pt, Catppuccin Mocha

Profile: "SSH Production" — red border for production
  Badge: ⚠️ PRODUCTION
  Background colour: slightly reddish tint
  Command: ssh prod-server

Profile: "Presentation" — large font for demos
  Font: JetBrainsMono 18pt
  Window: larger initial size
```

---

## Alacritty on macOS

### Installation

```bash
brew install --cask alacritty

# First launch: System Preferences → Security → Allow Alacritty
# macOS Gatekeeper will block unsigned apps on first run
```

### macOS-Specific Configuration

```toml
# ~/.config/alacritty/alacritty.toml

[window]
decorations = "Buttonless"     # macOS: removes title bar, keeps traffic lights
option_as_alt = "Both"         # CRITICAL: makes Option key work as Alt/Meta

[font]
size = 13.0                    # macOS Retina: slightly larger looks better

[font.normal]
family = "JetBrainsMono Nerd Font"

# macOS-specific key bindings
[[keyboard.bindings]]
key = "N"
mods = "Command"
action = "CreateNewWindow"

[[keyboard.bindings]]
key = "Q"
mods = "Command"
action = "Quit"

[[keyboard.bindings]]
key = "W"
mods = "Command"
action = "Quit"

[[keyboard.bindings]]
key = "F"
mods = "Command|Control"
action = "ToggleFullscreen"

[[keyboard.bindings]]
key = "C"
mods = "Command"
action = "Copy"

[[keyboard.bindings]]
key = "V"
mods = "Command"
action = "Paste"

[[keyboard.bindings]]
key = "Key0"
mods = "Command"
action = "ResetFontSize"

[[keyboard.bindings]]
key = "Equals"
mods = "Command"
action = "IncreaseFontSize"

[[keyboard.bindings]]
key = "Minus"
mods = "Command"
action = "DecreaseFontSize"
```

---

## WezTerm on macOS

### Installation

```bash
brew install --cask wezterm
```

### macOS-Specific Configuration

```lua
-- ~/.config/wezterm/wezterm.lua

local wezterm = require("wezterm")
local config = wezterm.config_builder()

-- macOS-specific settings
config.font = wezterm.font("JetBrainsMono Nerd Font")
config.font_size = 13.0

-- Native macOS appearance
config.window_decorations = "RESIZE"
config.native_macos_fullscreen_mode = false  -- Use non-native (faster)
config.macos_window_background_blur = 20

-- Make Option key work as Alt/Meta
config.send_composed_key_when_left_alt_is_pressed = false
config.send_composed_key_when_right_alt_is_pressed = false

-- macOS key bindings
config.keys = {
    -- Cmd+T for new tab (matches macOS convention)
    { key = "t", mods = "CMD", action = wezterm.action.SpawnTab("CurrentPaneDomain") },
    -- Cmd+W to close tab
    { key = "w", mods = "CMD", action = wezterm.action.CloseCurrentTab({ confirm = true }) },
    -- Cmd+D for vertical split
    { key = "d", mods = "CMD", action = wezterm.action.SplitHorizontal({ domain = "CurrentPaneDomain" }) },
    -- Cmd+Shift+D for horizontal split
    { key = "d", mods = "CMD|SHIFT", action = wezterm.action.SplitVertical({ domain = "CurrentPaneDomain" }) },
}

config.color_scheme = "Catppuccin Mocha"

return config
```

---

## Homebrew Setup

**Homebrew** is the essential package manager for macOS. Install it first before anything else.

### Installation

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# Apple Silicon (M1/M2/M3) — add to PATH
echo 'eval "$(/opt/homebrew/bin/brew shellenv)"' >> ~/.zprofile
eval "$(/opt/homebrew/bin/brew shellenv)"

# Intel Mac — Homebrew installs to /usr/local (already in PATH)

# Verify
brew --version
```

### Essential Packages

```bash
# Core development tools
brew install git neovim tmux

# Modern CLI replacements
brew install fzf ripgrep fd bat eza zoxide delta jq

# Shell tools
brew install starship zsh-autosuggestions zsh-syntax-highlighting

# Development languages and tools
brew install node python go rust

# Fonts
brew install --cask font-jetbrains-mono-nerd-font

# Terminal emulators
brew install --cask iterm2     # or: alacritty, wezterm

# GUI applications
brew install --cask visual-studio-code rectangle raycast
```

### Brewfile for Reproducible Setup

```ruby
# ~/Brewfile — define all packages for reproducible setup

# Taps
tap "homebrew/cask-fonts"

# CLI tools
brew "git"
brew "neovim"
brew "tmux"
brew "fzf"
brew "ripgrep"
brew "fd"
brew "bat"
brew "eza"
brew "zoxide"
brew "delta"
brew "jq"
brew "gh"
brew "starship"
brew "zsh-autosuggestions"
brew "zsh-syntax-highlighting"
brew "tldr"
brew "httpie"

# Languages
brew "node"
brew "python"
brew "go"

# Fonts
cask "font-jetbrains-mono-nerd-font"

# Applications
cask "iterm2"
cask "rectangle"
cask "raycast"
```

```bash
# Install everything from Brewfile
brew bundle install --file=~/Brewfile

# Generate Brewfile from currently installed packages
brew bundle dump --file=~/Brewfile --force

# Check for missing packages
brew bundle check --file=~/Brewfile
```

---

## macOS Shell Configuration

### Zsh as Default Shell

macOS uses Zsh as the default shell since Catalina. However, the system Zsh may be outdated:

```bash
# Check system Zsh version
/bin/zsh --version

# Install latest Zsh via Homebrew
brew install zsh

# Add Homebrew Zsh to allowed shells
sudo sh -c 'echo /opt/homebrew/bin/zsh >> /etc/shells'

# Change default shell
chsh -s /opt/homebrew/bin/zsh
```

### macOS-Specific .zshrc

```bash
# ~/.zshrc — macOS-specific configuration

# ─── Homebrew ────────────────────────────────────────────────────
eval "$(/opt/homebrew/bin/brew shellenv)"

# Homebrew completions
if type brew &>/dev/null; then
    FPATH="$(brew --prefix)/share/zsh/site-functions:${FPATH}"
fi

# ─── Zsh Plugins (installed via Homebrew) ────────────────────────
source $(brew --prefix)/share/zsh-autosuggestions/zsh-autosuggestions.zsh
source $(brew --prefix)/share/zsh-syntax-highlighting/zsh-syntax-highlighting.zsh

# ─── Prompt ──────────────────────────────────────────────────────
eval "$(starship init zsh)"

# ─── fzf ─────────────────────────────────────────────────────────
source <(fzf --zsh)

# ─── zoxide ──────────────────────────────────────────────────────
eval "$(zoxide init zsh)"

# ─── macOS-Specific Aliases ──────────────────────────────────────
# Open files with default macOS application
alias o='open'
alias oo='open .'

# Quick Look a file from terminal
alias ql='qlmanage -p'

# Flush DNS cache
alias flushdns='sudo dscacheutil -flushcache; sudo killall -HUP mDNSResponder'

# Show/hide hidden files in Finder
alias showfiles='defaults write com.apple.finder AppleShowAllFiles YES; killall Finder'
alias hidefiles='defaults write com.apple.finder AppleShowAllFiles NO; killall Finder'

# Lock screen
alias afk='/System/Library/CoreServices/Menu\ Extras/User.menu/Contents/Resources/CGSession -suspend'

# Airport CLI (Wi-Fi diagnostics)
alias airport='/System/Library/PrivateFrameworks/Apple80211.framework/Versions/Current/Resources/airport'

# Clipboard (already available as pbcopy/pbpaste, but alias for consistency)
# These work in tmux, pipes, and scripts
alias clip='pbcopy'
alias paste='pbpaste'

# ─── macOS PATH ──────────────────────────────────────────────────
# Homebrew paths are handled by brew shellenv above
# Add user-local binaries
export PATH="$HOME/.local/bin:$PATH"
```

---

## Key Binding Considerations

### The Option Key Problem

On macOS, the **Option (⌥)** key by default inserts special characters (e.g., Option+N = ñ) instead of acting as **Alt/Meta** for terminal applications. This breaks many terminal key bindings (e.g., `Alt+B` for word-back in Bash/Zsh).

### Fix by Terminal Emulator

| Emulator | Setting | Location |
|----------|---------|----------|
| **iTerm2** | Left Option key → Esc+ | Profiles → Keys → General |
| **Alacritty** | `option_as_alt = "Both"` | `alacritty.toml` → `[window]` |
| **WezTerm** | `send_composed_key_when_left_alt_is_pressed = false` | `wezterm.lua` |
| **kitty** | `macos_option_as_alt yes` | `kitty.conf` |
| **Terminal.app** | Use Option as Meta key ✅ | Profiles → Keyboard |

### macOS Keyboard Shortcuts That Conflict

Some system shortcuts may conflict with terminal key bindings:

```
Common conflicts and fixes:
  Ctrl+Space   — System: Spotlight → Change to Cmd+Space or disable
  Ctrl+Up/Down — System: Mission Control → Disable in System Preferences
  Ctrl+Left/Right — System: Desktop switching → Disable or rebind

System Preferences → Keyboard → Shortcuts:
  Disable Mission Control shortcuts that conflict with tmux/vim
```

---

## Font Configuration

### Installing Nerd Fonts via Homebrew

```bash
# Install Nerd Fonts
brew install --cask font-jetbrains-mono-nerd-font
brew install --cask font-fira-code-nerd-font
brew install --cask font-cascadia-code-nerd-font
brew install --cask font-hack-nerd-font
brew install --cask font-meslo-lg-nerd-font

# List installed fonts
fc-list | grep -i "nerd"

# macOS Font Book will also show installed fonts
# Applications → Font Book
```

### Recommended Sizes for Retina Displays

| Display | Font Size | Notes |
|---------|-----------|-------|
| **Retina (built-in)** | 13–14pt | Default 12pt can be too small |
| **External 4K** | 12–13pt | Depends on scaling |
| **External 1080p** | 11–12pt | Standard size |

---

## macOS Integration

### Using macOS Services from Terminal

```bash
# Open files/URLs with default application
open file.pdf
open https://example.com
open -a "Visual Studio Code" .        # Open current dir in VS Code
open -a Finder .                       # Open current dir in Finder

# Copy/paste with system clipboard
echo "text" | pbcopy                   # Copy to clipboard
pbpaste                                # Paste from clipboard
pbpaste | wc -l                        # Count lines in clipboard

# Quick Look preview from terminal
qlmanage -p image.png

# Say text aloud
say "Build complete"

# Notification Centre
osascript -e 'display notification "Build complete" with title "Terminal"'

# Open man page in Preview as PDF
man -t git | open -f -a Preview
```

### launchd for Background Services

```xml
<!-- ~/Library/LaunchAgents/com.user.tmux-server.plist -->
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.user.tmux-server</string>
    <key>ProgramArguments</key>
    <array>
        <string>/opt/homebrew/bin/tmux</string>
        <string>new-session</string>
        <string>-d</string>
        <string>-s</string>
        <string>main</string>
    </array>
    <key>RunAtLoad</key>
    <true/>
    <key>KeepAlive</key>
    <false/>
</dict>
</plist>
```

```bash
# Load the agent (starts tmux session at login)
launchctl load ~/Library/LaunchAgents/com.user.tmux-server.plist

# Unload
launchctl unload ~/Library/LaunchAgents/com.user.tmux-server.plist
```

---

## Spotlight and Alfred Integration

### Raycast (Modern Spotlight Replacement)

```bash
brew install --cask raycast

# Raycast can:
# - Launch terminal commands
# - Run shell scripts
# - Search clipboard history
# - Manage windows
# - Quick open projects in terminal

# Create a Raycast script command to open project in terminal:
# ~/.local/share/raycast/scripts/open-project.sh
```

```bash
#!/bin/bash

# Required parameters:
# @raycast.schemaVersion 1
# @raycast.title Open Project
# @raycast.mode silent
# @raycast.packageName Developer Utils

# Optional parameters:
# @raycast.icon 💻
# @raycast.argument1 { "type": "text", "placeholder": "project name" }

cd "$HOME/projects/$1" && open -a "Alacritty" .
```

### Alfred Workflows for Terminal

```
Alfred → Preferences → Workflows:
  Custom Terminal: Set to iTerm2 or Alacritty
  Integration: Alfred can open terminal in current Finder directory
  
  Useful keywords:
  - "term" → Open new terminal window
  - "ssh <host>" → SSH to saved hosts
  - "> <command>" → Run shell command directly
```

---

## Next Steps

- **Shell setup** → [01-SHELL-FUNDAMENTALS.md](01-SHELL-FUNDAMENTALS.md) — Configure Zsh with plugins and prompt
- **tmux** → [02-TMUX.md](02-TMUX.md) — Essential for Alacritty users (no built-in splits)
- **Neovim** → [03-NEOVIM.md](03-NEOVIM.md) — Terminal-based editor configuration
- **Productivity tools** → [07-PRODUCTIVITY-TOOLS.md](07-PRODUCTIVITY-TOOLS.md) — fzf, ripgrep, bat, and more
- **Best practices** → [08-BEST-PRACTICES.md](08-BEST-PRACTICES.md) — Brewfile and dotfile management

---

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2025 | Initial macOS terminal setup documentation |
