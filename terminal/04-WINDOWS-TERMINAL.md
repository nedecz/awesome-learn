# Windows Terminal Setup

A comprehensive guide to setting up a productive terminal environment on Windows — covering Windows Terminal installation, settings.json configuration, profiles, WSL2 integration, PowerShell setup, key bindings, themes, fonts, SSH integration, and Windows-specific package managers.

---

## Table of Contents

1. [Overview](#overview)
2. [Installation](#installation)
3. [Settings and Configuration](#settings-and-configuration)
4. [Profiles](#profiles)
5. [WSL2 Integration](#wsl2-integration)
6. [PowerShell Setup](#powershell-setup)
7. [Key Bindings](#key-bindings)
8. [Themes and Appearance](#themes-and-appearance)
9. [Font Configuration](#font-configuration)
10. [SSH Integration](#ssh-integration)
11. [Windows Package Managers](#windows-package-managers)
12. [Essential Tools for Windows](#essential-tools-for-windows)
13. [Next Steps](#next-steps)
14. [Version History](#version-history)

---

## Overview

**Windows Terminal** is Microsoft's modern, GPU-accelerated terminal application for Windows 10 and 11. It supports multiple tabs, split panes, Unicode, true colour, and configurable profiles for PowerShell, Command Prompt, WSL distributions, and SSH connections.

### Target Audience

- **Windows developers** who want a modern terminal experience
- **WSL users** running Linux distributions alongside Windows
- **DevOps engineers** working across Windows and Linux environments

---

## Installation

```powershell
# Via Microsoft Store (recommended — auto-updates)
# Search for "Windows Terminal" in the Microsoft Store

# Via winget
winget install Microsoft.WindowsTerminal

# Via scoop
scoop install windows-terminal

# Via chocolatey
choco install microsoft-windows-terminal

# Verify installation
wt --version
```

### Setting Windows Terminal as Default

Windows 11:
- Settings → Privacy & security → For developers → Terminal → Windows Terminal

Windows 10:
- Windows Terminal Settings → Startup → Default terminal application → Windows Terminal

---

## Settings and Configuration

Windows Terminal is configured via `settings.json`. Open it with `Ctrl+Shift+,` or through the Settings UI.

### Settings File Location

```
%LOCALAPPDATA%\Packages\Microsoft.WindowsTerminal_8wekyb3d8bbwe\LocalState\settings.json
```

### Complete settings.json Example

```json
{
    "$help": "https://aka.ms/terminal-documentation",
    "$schema": "https://aka.ms/terminal-profiles-schema",

    "defaultProfile": "{574e775e-4f2a-5b96-ac1e-a2962a402336}",
    "copyOnSelect": true,
    "copyFormatting": "none",
    "trimBlockSelection": true,

    "launchMode": "default",
    "startOnUserLogin": false,
    "initialCols": 120,
    "initialRows": 35,
    "initialPosition": ",",

    "tabWidthMode": "equal",
    "showTabsInTitlebar": true,
    "useAcrylicInTabRow": true,
    "showTerminalTitleInTitlebar": true,

    "theme": "dark",
    "alwaysShowTabs": true,

    "profiles": {
        "defaults": {
            "font": {
                "face": "JetBrainsMono Nerd Font",
                "size": 12,
                "weight": "normal"
            },
            "colorScheme": "Catppuccin Mocha",
            "opacity": 95,
            "useAcrylic": false,
            "padding": "8",
            "antialiasingMode": "grayscale",
            "cursorShape": "bar",
            "cursorHeight": 25,
            "scrollbarState": "hidden",
            "altGrAliasing": true
        },
        "list": []
    },

    "schemes": [],
    "actions": [],
    "keybindings": []
}
```

---

## Profiles

### PowerShell 7 Profile

```json
{
    "name": "PowerShell",
    "guid": "{574e775e-4f2a-5b96-ac1e-a2962a402336}",
    "source": "Windows.Terminal.PowershellCore",
    "commandline": "pwsh.exe -NoLogo",
    "icon": "ms-appx:///ProfileIcons/pwsh.png",
    "startingDirectory": "%USERPROFILE%",
    "hidden": false
}
```

### WSL Ubuntu Profile

```json
{
    "name": "Ubuntu",
    "guid": "{2c4de342-38b7-51cf-b940-2309a097f518}",
    "source": "Windows.Terminal.Wsl",
    "commandline": "wsl.exe -d Ubuntu",
    "icon": "ms-appx:///ProfileIcons/{9acb9455-ca41-5af7-950f-6bca1bc9722f}.png",
    "startingDirectory": "//wsl$/Ubuntu/home/username",
    "hidden": false
}
```

### SSH Profile

```json
{
    "name": "Production Server",
    "commandline": "ssh user@production-server.example.com",
    "icon": "🖥️",
    "tabTitle": "prod-server",
    "startingDirectory": null,
    "hidden": false
}
```

### Git Bash Profile

```json
{
    "name": "Git Bash",
    "commandline": "C:\\Program Files\\Git\\bin\\bash.exe --login -i",
    "icon": "C:\\Program Files\\Git\\mingw64\\share\\git\\git-for-windows.ico",
    "startingDirectory": "%USERPROFILE%",
    "hidden": false
}
```

---

## WSL2 Integration

### Installing WSL2

```powershell
# Install WSL2 with Ubuntu (requires Windows 10 2004+ or Windows 11)
wsl --install

# Install a specific distribution
wsl --install -d Ubuntu-24.04

# List available distributions
wsl --list --online

# List installed distributions
wsl --list --verbose

# Set WSL2 as default version
wsl --set-default-version 2

# Update WSL kernel
wsl --update
```

### WSL2 Configuration

```ini
# %USERPROFILE%\.wslconfig — Global WSL2 settings

[wsl2]
memory=8GB                    # Limit RAM allocation
processors=4                  # Limit CPU cores
swap=4GB                      # Swap file size
localhostForwarding=true      # Access WSL2 services from Windows
nestedVirtualization=true     # Enable nested virtualisation

[experimental]
autoMemoryReclaim=gradual     # Reclaim unused memory
sparseVhd=true               # Compact virtual disk automatically
```

```ini
# /etc/wsl.conf — Per-distribution settings (inside WSL)

[boot]
systemd=true                  # Enable systemd (WSL2 only)

[automount]
enabled=true
root=/mnt/
options="metadata,umask=22,fmask=11"

[network]
generateHosts=true
generateResolvConf=true

[interop]
enabled=true                  # Run Windows executables from WSL
appendWindowsPath=true        # Add Windows PATH to WSL PATH
```

### WSL2 Best Practices

```bash
# Store projects inside WSL filesystem for best performance
# BAD:  /mnt/c/Users/name/projects  (Windows filesystem — slow)
# GOOD: ~/projects                   (WSL filesystem — fast)

# Access WSL files from Windows
# In Windows Explorer: \\wsl$\Ubuntu\home\username

# Access Windows files from WSL
ls /mnt/c/Users/

# Set default WSL user (run in PowerShell)
# ubuntu config --default-user yourusername

# Open Windows Explorer from WSL
explorer.exe .

# Copy between WSL and Windows clipboards
echo "text" | clip.exe              # WSL → Windows clipboard
powershell.exe Get-Clipboard        # Windows clipboard → WSL stdout
```

---

## PowerShell Setup

### Installing PowerShell 7

```powershell
# Via winget
winget install Microsoft.PowerShell

# Via MSI from GitHub releases
# https://github.com/PowerShell/PowerShell/releases

# Verify version
$PSVersionTable.PSVersion
```

### PowerShell Profile Configuration

```powershell
# Check profile path
$PROFILE
# Typically: C:\Users\<name>\Documents\PowerShell\Microsoft.PowerShell_profile.ps1

# Create profile if it doesn't exist
if (!(Test-Path -Path $PROFILE)) {
    New-Item -ItemType File -Path $PROFILE -Force
}

# Edit profile
notepad $PROFILE
# or: nvim $PROFILE
```

```powershell
# Microsoft.PowerShell_profile.ps1

# ─── Oh My Posh (Prompt) ────────────────────────────────────────
# Install: winget install JanDeDobbeleer.OhMyPosh
oh-my-posh init pwsh --config "$env:POSH_THEMES_PATH\catppuccin_mocha.omp.json" | Invoke-Expression

# ─── PSReadLine (Enhanced Readline) ─────────────────────────────
Import-Module PSReadLine
Set-PSReadLineOption -PredictionSource History
Set-PSReadLineOption -PredictionViewStyle ListView
Set-PSReadLineOption -EditMode Emacs
Set-PSReadLineKeyHandler -Key UpArrow -Function HistorySearchBackward
Set-PSReadLineKeyHandler -Key DownArrow -Function HistorySearchForward
Set-PSReadLineKeyHandler -Key Tab -Function MenuComplete

# ─── Aliases ─────────────────────────────────────────────────────
Set-Alias -Name vim -Value nvim
Set-Alias -Name g -Value git
Set-Alias -Name k -Value kubectl
Set-Alias -Name which -Value Get-Command

# ─── Functions ───────────────────────────────────────────────────
function ll { Get-ChildItem -Force @args }
function mkcd { param($dir) New-Item -ItemType Directory -Path $dir -Force; Set-Location $dir }
function touch { param($file) if (Test-Path $file) { (Get-Item $file).LastWriteTime = Get-Date } else { New-Item $file -ItemType File } }
function grep { $input | Out-String -Stream | Select-String $args }
function which { Get-Command @args | Select-Object -ExpandProperty Definition }

# ─── Environment Variables ───────────────────────────────────────
$env:EDITOR = "nvim"
$env:VISUAL = "nvim"

# ─── fzf Integration ────────────────────────────────────────────
# Install: winget install junegunn.fzf
Import-Module PSFzf
Set-PsFzfOption -PSReadlineChordProvider 'Ctrl+t' -PSReadlineChordReverseHistory 'Ctrl+r'

# ─── Terminal Icons ──────────────────────────────────────────────
Import-Module Terminal-Icons
```

### Installing PowerShell Modules

```powershell
# PSReadLine (usually pre-installed with PS7)
Install-Module PSReadLine -Force -SkipPublisherCheck

# Terminal-Icons (file icons in directory listings)
Install-Module Terminal-Icons -Repository PSGallery

# PSFzf (fzf integration)
Install-Module PSFzf -Scope CurrentUser

# posh-git (Git integration for prompt)
Install-Module posh-git -Scope CurrentUser

# z (directory jumping)
Install-Module z -Scope CurrentUser
```

---

## Key Bindings

### Default Key Bindings

| Key Binding | Action |
|-------------|--------|
| `Ctrl+Shift+T` | New tab |
| `Ctrl+Shift+W` | Close tab |
| `Ctrl+Tab` | Next tab |
| `Ctrl+Shift+Tab` | Previous tab |
| `Ctrl+Shift+N` | New window |
| `Alt+Shift+D` | Duplicate pane |
| `Alt+Shift+-` | Split pane horizontally |
| `Alt+Shift++` | Split pane vertically |
| `Alt+Arrow` | Move focus between panes |
| `Alt+Shift+Arrow` | Resize pane |
| `Ctrl+Shift+P` | Open command palette |
| `Ctrl+Shift+,` | Open settings.json |
| `Ctrl+Shift+Space` | Open dropdown (new tab menu) |
| `Ctrl+,` | Open settings UI |

### Custom Key Bindings

```json
{
    "actions": [
        {
            "command": { "action": "splitPane", "split": "horizontal" },
            "keys": "alt+shift+-"
        },
        {
            "command": { "action": "splitPane", "split": "vertical" },
            "keys": "alt+shift+plus"
        },
        {
            "command": { "action": "moveFocus", "direction": "left" },
            "keys": "alt+h"
        },
        {
            "command": { "action": "moveFocus", "direction": "down" },
            "keys": "alt+j"
        },
        {
            "command": { "action": "moveFocus", "direction": "up" },
            "keys": "alt+k"
        },
        {
            "command": { "action": "moveFocus", "direction": "right" },
            "keys": "alt+l"
        },
        {
            "command": { "action": "resizePane", "direction": "left" },
            "keys": "alt+shift+h"
        },
        {
            "command": { "action": "resizePane", "direction": "down" },
            "keys": "alt+shift+j"
        },
        {
            "command": { "action": "resizePane", "direction": "up" },
            "keys": "alt+shift+k"
        },
        {
            "command": { "action": "resizePane", "direction": "right" },
            "keys": "alt+shift+l"
        },
        {
            "command": "togglePaneZoom",
            "keys": "alt+shift+z"
        },
        {
            "command": { "action": "newTab", "profile": "Ubuntu" },
            "keys": "ctrl+shift+u"
        },
        {
            "command": "find",
            "keys": "ctrl+shift+f"
        }
    ]
}
```

---

## Themes and Appearance

### Catppuccin Mocha Colour Scheme

```json
{
    "schemes": [
        {
            "name": "Catppuccin Mocha",
            "background": "#1E1E2E",
            "foreground": "#CDD6F4",
            "cursorColor": "#F5E0DC",
            "selectionBackground": "#585B70",
            "black": "#45475A",
            "red": "#F38BA8",
            "green": "#A6E3A1",
            "yellow": "#F9E2AF",
            "blue": "#89B4FA",
            "purple": "#F5C2E7",
            "cyan": "#94E2D5",
            "white": "#BAC2DE",
            "brightBlack": "#585B70",
            "brightRed": "#F38BA8",
            "brightGreen": "#A6E3A1",
            "brightYellow": "#F9E2AF",
            "brightBlue": "#89B4FA",
            "brightPurple": "#F5C2E7",
            "brightCyan": "#94E2D5",
            "brightWhite": "#A6ADC8"
        }
    ]
}
```

### Other Popular Schemes

| Theme | Style | Source |
|-------|-------|--------|
| **Catppuccin** | Warm pastels | github.com/catppuccin/windows-terminal |
| **Dracula** | Dark purple | github.com/dracula/windows-terminal |
| **Nord** | Arctic cool | github.com/arcticicestudio/nord-windows-terminal |
| **One Half Dark** | Atom-inspired | Built into Windows Terminal |
| **Solarized Dark** | Precise colours | Built into Windows Terminal |
| **Tokyo Night** | Dark blue/purple | github.com/enkia/tokyo-night-vscode-theme |

---

## Font Configuration

### Installing Nerd Fonts

```powershell
# Via scoop
scoop bucket add nerd-fonts
scoop install JetBrainsMono-NF

# Via winget
winget install JetBrainsMono

# Via chocolatey
choco install nerd-fonts-jetbrainsmono

# Manual: Download from https://www.nerdfonts.com/font-downloads
# Extract and install via right-click → "Install for all users"
```

### Font Settings in settings.json

```json
{
    "profiles": {
        "defaults": {
            "font": {
                "face": "JetBrainsMono Nerd Font",
                "size": 12,
                "weight": "normal",
                "features": {
                    "liga": 1,
                    "calt": 1
                }
            }
        }
    }
}
```

### Recommended Nerd Fonts

| Font | Ligatures | Style |
|------|-----------|-------|
| **JetBrains Mono** | ✅ | Clean, purpose-built for code |
| **Fira Code** | ✅ | Popular, wide character support |
| **Cascadia Code** | ✅ | Microsoft's own coding font |
| **Hack** | ❌ | Extremely legible, no frills |
| **Meslo** | ❌ | Customised Apple Menlo |

---

## SSH Integration

### SSH Config

```
# ~/.ssh/config (or %USERPROFILE%\.ssh\config on Windows)

# Global defaults
Host *
    AddKeysToAgent yes
    IdentitiesOnly yes
    ServerAliveInterval 60
    ServerAliveCountMax 3

# Development server
Host dev
    HostName dev.example.com
    User developer
    IdentityFile ~/.ssh/id_ed25519
    ForwardAgent yes

# Production (via bastion)
Host prod
    HostName 10.0.1.50
    User deploy
    IdentityFile ~/.ssh/id_ed25519_prod
    ProxyJump bastion

Host bastion
    HostName bastion.example.com
    User admin
    IdentityFile ~/.ssh/id_ed25519

# Usage: ssh dev, ssh prod, ssh bastion
```

### SSH Agent on Windows

```powershell
# Enable OpenSSH Agent service
Get-Service ssh-agent | Set-Service -StartupType Automatic
Start-Service ssh-agent

# Add SSH key
ssh-add $env:USERPROFILE\.ssh\id_ed25519

# List loaded keys
ssh-add -l

# Generate a new SSH key
ssh-keygen -t ed25519 -C "your.email@example.com"
```

---

## Windows Package Managers

### winget (Windows Package Manager)

```powershell
# Search for packages
winget search neovim

# Install packages
winget install Neovim.Neovim
winget install Git.Git
winget install Microsoft.PowerShell
winget install junegunn.fzf
winget install BurntSushi.ripgrep.MSVC
winget install sharkdp.fd
winget install sharkdp.bat

# Update all packages
winget upgrade --all

# List installed packages
winget list

# Export/import package list
winget export -o packages.json
winget import -i packages.json
```

### Scoop

```powershell
# Install scoop
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser
Invoke-RestMethod -Uri https://get.scoop.sh | Invoke-Expression

# Add extras bucket
scoop bucket add extras
scoop bucket add nerd-fonts

# Install packages
scoop install git neovim fzf ripgrep fd bat eza delta jq
scoop install JetBrainsMono-NF

# Update all packages
scoop update *

# Export/import
scoop export > scoopfile.json
```

### Chocolatey

```powershell
# Install chocolatey (elevated PowerShell)
Set-ExecutionPolicy Bypass -Scope Process -Force
[System.Net.ServicePointManager]::SecurityProtocol = [System.Net.ServicePointManager]::SecurityProtocol -bor 3072
Invoke-Expression ((New-Object System.Net.WebClient).DownloadString('https://community.chocolatey.org/install.ps1'))

# Install packages
choco install git neovim fzf ripgrep fd bat

# Update all
choco upgrade all
```

| Feature | winget | Scoop | Chocolatey |
|---------|--------|-------|------------|
| **Built-in** | ✅ (Win 11) | ❌ | ❌ |
| **No admin required** | Mostly | ✅ | ❌ (needs admin) |
| **Portable installs** | ❌ | ✅ (user-scoped) | ❌ |
| **Package count** | Large | Medium | Very large |
| **GUI apps** | ✅ | ✅ (extras bucket) | ✅ |
| **Best for** | System apps | Dev tools | Everything |

---

## Essential Tools for Windows

```powershell
# Development essentials via winget
winget install Git.Git
winget install Neovim.Neovim
winget install Microsoft.PowerShell
winget install junegunn.fzf
winget install BurntSushi.ripgrep.MSVC
winget install sharkdp.fd
winget install sharkdp.bat
winget install dandavison.delta
winget install ajeetdsouza.zoxide
winget install stedolan.jq
winget install GitHub.cli

# Or via scoop (recommended for dev tools)
scoop install git neovim fzf ripgrep fd bat delta zoxide jq gh eza
```

---

## Next Steps

- **Shell fundamentals** → [01-SHELL-FUNDAMENTALS.md](01-SHELL-FUNDAMENTALS.md) — Configure your shell within WSL2
- **tmux** → [02-TMUX.md](02-TMUX.md) — Terminal multiplexer (essential inside WSL2)
- **Neovim** → [03-NEOVIM.md](03-NEOVIM.md) — Set up Neovim in your Windows Terminal environment
- **Productivity tools** → [07-PRODUCTIVITY-TOOLS.md](07-PRODUCTIVITY-TOOLS.md) — Modern CLI tools for Windows and WSL
- **Best practices** → [08-BEST-PRACTICES.md](08-BEST-PRACTICES.md) — Cross-platform dotfile management

---

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2025 | Initial Windows Terminal documentation |
