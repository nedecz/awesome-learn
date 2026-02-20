# Linux Terminal Setup

A comprehensive guide to setting up a productive terminal environment on Linux — covering popular terminal emulators (Alacritty, kitty, WezTerm, GNOME Terminal, Konsole), GPU-accelerated terminals, font configuration, desktop environment integration, X11 vs Wayland considerations, and dotfile management.

---

## Table of Contents

1. [Overview](#overview)
2. [Terminal Emulator Comparison](#terminal-emulator-comparison)
3. [Alacritty](#alacritty)
4. [kitty](#kitty)
5. [WezTerm](#wezterm)
6. [GNOME Terminal](#gnome-terminal)
7. [Konsole](#konsole)
8. [GPU-Accelerated Terminals](#gpu-accelerated-terminals)
9. [Font Configuration](#font-configuration)
10. [X11 vs Wayland](#x11-vs-wayland)
11. [Desktop Environment Integration](#desktop-environment-integration)
12. [Dotfile Management](#dotfile-management)
13. [Next Steps](#next-steps)
14. [Version History](#version-history)

---

## Overview

Linux offers the widest variety of terminal emulators of any platform, from lightweight GPU-accelerated options to feature-rich desktop-integrated applications. This guide covers the most popular choices and helps you configure a productive terminal environment regardless of your desktop environment.

### Target Audience

- **Linux desktop users** setting up or optimising their terminal environment
- **Developers** choosing a terminal emulator for their Linux workstation
- **System administrators** configuring terminals for efficient remote and local workflows

---

## Terminal Emulator Comparison

| Emulator | GPU Accel | Config Format | Multiplexer | Images | Wayland | Best For |
|----------|-----------|---------------|-------------|--------|---------|----------|
| **Alacritty** | ✅ OpenGL | TOML | ❌ (use tmux) | ❌ | ✅ | Speed and minimalism |
| **kitty** | ✅ OpenGL | kitty.conf | ✅ (tabs, splits) | ✅ (kitty protocol) | ✅ | Power users, image display |
| **WezTerm** | ✅ GPU | Lua | ✅ (built-in) | ✅ (multiple protocols) | ✅ | Cross-platform, Lua config |
| **GNOME Terminal** | ❌ | GUI / dconf | ✅ (tabs) | ❌ | ✅ | GNOME desktop users |
| **Konsole** | ❌ | GUI / config | ✅ (tabs, splits) | ❌ | ✅ | KDE desktop users |
| **foot** | ✅ (Wayland) | foot.ini | ❌ | ✅ (Sixel) | ✅ only | Wayland minimalism |
| **st** | ❌ | C source (patches) | ❌ | ❌ | ✅ | Suckless philosophy |
| **xterm** | ❌ | X resources | ❌ | ❌ | ❌ (X11 only) | Legacy compatibility |

---

## Alacritty

Alacritty is a fast, cross-platform, GPU-accelerated terminal emulator. It is intentionally minimal — no tabs, no splits, no GUI configuration. It is designed to be paired with tmux for session management.

### Installation

```bash
# Debian/Ubuntu
sudo add-apt-repository ppa:aslatter/ppa
sudo apt update && sudo apt install alacritty

# Fedora
sudo dnf install alacritty

# Arch Linux
sudo pacman -S alacritty

# From source (Rust required)
git clone https://github.com/alacritty/alacritty.git
cd alacritty
cargo build --release
sudo cp target/release/alacritty /usr/local/bin/
```

### Configuration

```toml
# ~/.config/alacritty/alacritty.toml

[window]
padding = { x = 8, y = 8 }
decorations = "Full"
opacity = 0.95
startup_mode = "Windowed"
dynamic_title = true

[scrolling]
history = 10000
multiplier = 3

[font]
size = 12.0

[font.normal]
family = "JetBrainsMono Nerd Font"
style = "Regular"

[font.bold]
family = "JetBrainsMono Nerd Font"
style = "Bold"

[font.italic]
family = "JetBrainsMono Nerd Font"
style = "Italic"

[cursor]
style = { shape = "Block", blinking = "On" }
vi_mode_style = { shape = "Block", blinking = "Off" }

[mouse]
hide_when_typing = true

[selection]
save_to_clipboard = true

[terminal]
osc52 = "CopyPaste"

[env]
TERM = "xterm-256color"

# Catppuccin Mocha colour scheme
[colors.primary]
background = "#1E1E2E"
foreground = "#CDD6F4"
dim_foreground = "#CDD6F4"
bright_foreground = "#CDD6F4"

[colors.cursor]
text = "#1E1E2E"
cursor = "#F5E0DC"

[colors.vi_mode_cursor]
text = "#1E1E2E"
cursor = "#B4BEFE"

[colors.selection]
text = "#1E1E2E"
background = "#F5E0DC"

[colors.normal]
black = "#45475A"
red = "#F38BA8"
green = "#A6E3A1"
yellow = "#F9E2AF"
blue = "#89B4FA"
magenta = "#F5C2E7"
cyan = "#94E2D5"
white = "#BAC2DE"

[colors.bright]
black = "#585B70"
red = "#F38BA8"
green = "#A6E3A1"
yellow = "#F9E2AF"
blue = "#89B4FA"
magenta = "#F5C2E7"
cyan = "#94E2D5"
white = "#A6ADC8"

# Key bindings
[[keyboard.bindings]]
key = "N"
mods = "Control|Shift"
action = "CreateNewWindow"

[[keyboard.bindings]]
key = "Return"
mods = "Control|Shift"
action = "ToggleViMode"
```

---

## kitty

kitty is a fast, feature-rich, GPU-accelerated terminal emulator with built-in support for images, ligatures, tabs, splits, and extensibility via "kittens" (Python scripts).

### Installation

```bash
# Official installer (recommended)
curl -L https://sw.kovidgoyal.net/kitty/installer.sh | sh /dev/stdin

# Debian/Ubuntu
sudo apt install kitty

# Fedora
sudo dnf install kitty

# Arch Linux
sudo pacman -S kitty
```

### Configuration

```conf
# ~/.config/kitty/kitty.conf

# ─── Font ────────────────────────────────────────────────────────
font_family      JetBrainsMono Nerd Font
bold_font        auto
italic_font      auto
bold_italic_font auto
font_size        12.0
disable_ligatures never

# ─── Window ─────────────────────────────────────────────────────
window_padding_width 8
background_opacity 0.95
confirm_os_window_close 0
remember_window_size yes
initial_window_width 120c
initial_window_height 35c

# ─── Cursor ─────────────────────────────────────────────────────
cursor_shape block
cursor_blink_interval -1
cursor_stop_blinking_after 15.0

# ─── Scrollback ─────────────────────────────────────────────────
scrollback_lines 10000
scrollback_pager less --chop-long-lines --RAW-CONTROL-CHARS +INPUT_LINE_NUMBER

# ─── Mouse ──────────────────────────────────────────────────────
mouse_hide_wait 3.0
copy_on_select clipboard
strip_trailing_spaces smart

# ─── Terminal ────────────────────────────────────────────────────
term xterm-kitty
shell_integration enabled
enable_audio_bell no

# ─── Tab Bar ─────────────────────────────────────────────────────
tab_bar_edge top
tab_bar_style powerline
tab_powerline_style slanted
active_tab_foreground   #1E1E2E
active_tab_background   #89B4FA
inactive_tab_foreground #CDD6F4
inactive_tab_background #313244

# ─── Layouts ─────────────────────────────────────────────────────
enabled_layouts splits, tall, fat, grid, stack

# ─── Key Bindings ────────────────────────────────────────────────
# Tabs
map ctrl+shift+t new_tab_with_cwd
map ctrl+shift+w close_tab
map ctrl+shift+right next_tab
map ctrl+shift+left previous_tab

# Splits
map ctrl+shift+enter launch --location=split --cwd=current
map ctrl+shift+minus launch --location=hsplit --cwd=current
map ctrl+shift+backslash launch --location=vsplit --cwd=current

# Navigate splits
map ctrl+shift+h neighboring_window left
map ctrl+shift+j neighboring_window down
map ctrl+shift+k neighboring_window up
map ctrl+shift+l neighboring_window right

# Resize splits
map ctrl+shift+alt+h resize_window narrower 3
map ctrl+shift+alt+j resize_window shorter 3
map ctrl+shift+alt+k resize_window taller 3
map ctrl+shift+alt+l resize_window wider 3

# Scrollback
map ctrl+shift+g show_scrollback

# Font size
map ctrl+shift+equal change_font_size all +1.0
map ctrl+shift+minus change_font_size all -1.0
map ctrl+shift+0 change_font_size all 0

# ─── Catppuccin Mocha Theme ─────────────────────────────────────
# Or: include catppuccin-mocha.conf  (from github.com/catppuccin/kitty)
foreground              #CDD6F4
background              #1E1E2E
selection_foreground    #1E1E2E
selection_background    #F5E0DC
cursor                  #F5E0DC
cursor_text_color       #1E1E2E
url_color               #F5E0DC

color0  #45475A
color8  #585B70
color1  #F38BA8
color9  #F38BA8
color2  #A6E3A1
color10 #A6E3A1
color3  #F9E2AF
color11 #F9E2AF
color4  #89B4FA
color12 #89B4FA
color5  #F5C2E7
color13 #F5C2E7
color6  #94E2D5
color14 #94E2D5
color7  #BAC2DE
color15 #A6ADC8
```

### kitty Kittens (Extensions)

```bash
# Display images inline
kitty +kitten icat image.png

# View diffs with syntax highlighting
kitty +kitten diff file1.py file2.py

# Copy text from terminal output
kitty +kitten clipboard

# SSH with automatic terminfo installation
kitty +kitten ssh user@server

# Unicode input
kitty +kitten unicode_input
```

---

## WezTerm

WezTerm is a GPU-accelerated, cross-platform terminal emulator configured with Lua. It includes a built-in multiplexer, image support, and extensive customisation.

### Installation

```bash
# Debian/Ubuntu
curl -fsSL https://apt.fury.io/wez/gpg.key | sudo gpg --yes --dearmor -o /etc/apt/keyrings/wezterm-fury.gpg
echo 'deb [signed-by=/etc/apt/keyrings/wezterm-fury.gpg] https://apt.fury.io/wez/ * *' | sudo tee /etc/apt/sources.list.d/wezterm.list
sudo apt update && sudo apt install wezterm

# Fedora (via COPR)
sudo dnf copr enable wezfurlong/wezterm-nightly
sudo dnf install wezterm

# Arch Linux
sudo pacman -S wezterm

# AppImage
curl -LO https://github.com/wez/wezterm/releases/latest/download/WezTerm-nightly-Ubuntu18.04.AppImage
chmod +x WezTerm-*.AppImage
```

### Configuration

```lua
-- ~/.config/wezterm/wezterm.lua

local wezterm = require("wezterm")
local config = wezterm.config_builder()

-- Font
config.font = wezterm.font("JetBrainsMono Nerd Font")
config.font_size = 12.0
config.harfbuzz_features = { "calt=1", "clig=1", "liga=1" }

-- Window
config.window_padding = { left = 8, right = 8, top = 8, bottom = 8 }
config.window_background_opacity = 0.95
config.window_decorations = "RESIZE"
config.initial_cols = 120
config.initial_rows = 35

-- Tab bar
config.use_fancy_tab_bar = false
config.tab_bar_at_bottom = false
config.hide_tab_bar_if_only_one_tab = true

-- Colour scheme
config.color_scheme = "Catppuccin Mocha"

-- Cursor
config.default_cursor_style = "BlinkingBar"
config.cursor_blink_rate = 500

-- Scrollback
config.scrollback_lines = 10000

-- Key bindings
config.keys = {
    -- Split panes
    { key = "-", mods = "CTRL|SHIFT", action = wezterm.action.SplitVertical({ domain = "CurrentPaneDomain" }) },
    { key = "\\", mods = "CTRL|SHIFT", action = wezterm.action.SplitHorizontal({ domain = "CurrentPaneDomain" }) },

    -- Navigate panes
    { key = "h", mods = "CTRL|SHIFT", action = wezterm.action.ActivatePaneDirection("Left") },
    { key = "j", mods = "CTRL|SHIFT", action = wezterm.action.ActivatePaneDirection("Down") },
    { key = "k", mods = "CTRL|SHIFT", action = wezterm.action.ActivatePaneDirection("Up") },
    { key = "l", mods = "CTRL|SHIFT", action = wezterm.action.ActivatePaneDirection("Right") },

    -- Close pane
    { key = "w", mods = "CTRL|SHIFT", action = wezterm.action.CloseCurrentPane({ confirm = true }) },

    -- Toggle zoom
    { key = "z", mods = "CTRL|SHIFT", action = wezterm.action.TogglePaneZoomState },
}

return config
```

---

## GNOME Terminal

GNOME Terminal is the default terminal for GNOME desktop environments. It is reliable and well-integrated but lacks GPU acceleration.

### Configuration via dconf

```bash
# List all GNOME Terminal profiles
dconf list /org/gnome/terminal/legacy/profiles:/

# Get current profile ID
dconf read /org/gnome/terminal/legacy/profiles:/default

# Set font
dconf write /org/gnome/terminal/legacy/profiles:/:b1dcc9dd-5262-4d8d-a863-c897e6d979b9/font "'JetBrainsMono Nerd Font 12'"
dconf write /org/gnome/terminal/legacy/profiles:/:b1dcc9dd-5262-4d8d-a863-c897e6d979b9/use-system-font false

# Set colour scheme (requires loading a scheme)
# Use gogh for easy colour scheme installation:
bash -c "$(wget -qO- https://git.io/vQgMr)"
```

---

## Konsole

Konsole is the default terminal for KDE Plasma. It offers rich features including split views, profiles, SSH bookmarks, and deep KDE integration.

### Configuration

Konsole stores profiles in `~/.local/share/konsole/`:

```ini
# ~/.local/share/konsole/MyProfile.profile

[Appearance]
ColorScheme=Catppuccin-Mocha
Font=JetBrainsMono Nerd Font,12,-1,5,50,0,0,0,0,0

[General]
Name=MyProfile
Parent=FALLBACK/
Command=/bin/zsh
TerminalColumns=120
TerminalRows=35

[Scrolling]
HistoryMode=2
HistorySize=10000

[Terminal Features]
BlinkingCursorEnabled=true
```

---

## GPU-Accelerated Terminals

### Why GPU Acceleration Matters

```
Traditional rendering (CPU):
  1. Shell outputs text
  2. Terminal parses escape codes
  3. CPU renders each glyph to bitmap
  4. CPU composites bitmaps into framebuffer
  5. Display shows framebuffer

  Problem: Slow for large outputs, high CPU usage

GPU-accelerated rendering:
  1. Shell outputs text
  2. Terminal parses escape codes
  3. CPU shapes text (glyph lookup, layout)
  4. GPU renders glyphs via shaders
  5. GPU composites directly to display

  Benefit: Fast scrolling, low CPU, smooth rendering
```

### Performance Comparison

| Terminal | cat large-file (1M lines) | Memory (idle) | Startup time |
|----------|--------------------------|---------------|--------------|
| **Alacritty** | ~0.8s | ~25 MB | ~50ms |
| **kitty** | ~1.0s | ~40 MB | ~80ms |
| **WezTerm** | ~1.2s | ~60 MB | ~100ms |
| **GNOME Terminal** | ~3.5s | ~45 MB | ~200ms |
| **Konsole** | ~3.0s | ~50 MB | ~200ms |
| **xterm** | ~4.0s | ~15 MB | ~30ms |

*Times are approximate and vary by hardware.*

---

## Font Configuration

### Installing Nerd Fonts

```bash
# Download and install manually
mkdir -p ~/.local/share/fonts
cd ~/.local/share/fonts

# JetBrains Mono Nerd Font
curl -fLO https://github.com/ryanoasis/nerd-fonts/releases/latest/download/JetBrainsMono.tar.xz
tar xf JetBrainsMono.tar.xz
rm JetBrainsMono.tar.xz

# Rebuild font cache
fc-cache -fv

# Verify installation
fc-list | grep -i "JetBrains"
```

### System-Wide Font Configuration

```xml
<!-- ~/.config/fontconfig/fonts.conf -->
<?xml version="1.0"?>
<!DOCTYPE fontconfig SYSTEM "urn:fontconfig:fonts.dtd">
<fontconfig>
    <!-- Default monospace font -->
    <alias>
        <family>monospace</family>
        <prefer>
            <family>JetBrainsMono Nerd Font</family>
            <family>Noto Color Emoji</family>
        </prefer>
    </alias>

    <!-- Enable sub-pixel rendering (for LCD screens) -->
    <match target="font">
        <edit name="rgba" mode="assign">
            <const>rgb</const>
        </edit>
        <edit name="hinting" mode="assign">
            <bool>true</bool>
        </edit>
        <edit name="hintstyle" mode="assign">
            <const>hintslight</const>
        </edit>
        <edit name="antialias" mode="assign">
            <bool>true</bool>
        </edit>
        <edit name="lcdfilter" mode="assign">
            <const>lcddefault</const>
        </edit>
    </match>
</fontconfig>
```

---

## X11 vs Wayland

### Key Differences for Terminal Users

| Feature | X11 | Wayland |
|---------|-----|---------|
| **Clipboard** | `xclip`, `xsel` | `wl-copy`, `wl-paste` |
| **Screenshot** | `scrot`, `xdotool` | `grim`, `slurp` |
| **Display variable** | `DISPLAY=:0` | `WAYLAND_DISPLAY=wayland-0` |
| **DPI scaling** | Per-monitor is complex | Native per-monitor scaling |
| **Screen recording** | `ffmpeg` with x11grab | PipeWire + `wf-recorder` |
| **Terminal choice** | All terminals work | Prefer Wayland-native terminals |

### Clipboard Integration

```bash
# Detect display server
if [ -n "$WAYLAND_DISPLAY" ]; then
    echo "Running on Wayland"
    # Install clipboard tools
    # Debian/Ubuntu: sudo apt install wl-clipboard
    # Fedora: sudo dnf install wl-clipboard
else
    echo "Running on X11"
    # Install clipboard tools
    # Debian/Ubuntu: sudo apt install xclip
    # Fedora: sudo dnf install xclip
fi

# Cross-platform clipboard aliases for shell config
if [ -n "$WAYLAND_DISPLAY" ]; then
    alias pbcopy='wl-copy'
    alias pbpaste='wl-paste'
elif [ -n "$DISPLAY" ]; then
    alias pbcopy='xclip -selection clipboard'
    alias pbpaste='xclip -selection clipboard -o'
fi
```

### tmux Clipboard Configuration

```bash
# ~/.tmux.conf — platform-aware clipboard

# Detect Wayland or X11 and set clipboard command
if-shell '[ -n "$WAYLAND_DISPLAY" ]' \
    'bind -T copy-mode-vi y send-keys -X copy-pipe-and-cancel "wl-copy"' \
    'bind -T copy-mode-vi y send-keys -X copy-pipe-and-cancel "xclip -selection clipboard"'
```

---

## Desktop Environment Integration

### Set Default Terminal (GNOME)

```bash
# Set Alacritty as default terminal in GNOME
sudo update-alternatives --install /usr/bin/x-terminal-emulator \
    x-terminal-emulator /usr/local/bin/alacritty 50
sudo update-alternatives --config x-terminal-emulator

# Or via gsettings
gsettings set org.gnome.desktop.default-applications.terminal exec alacritty
```

### Set Default Terminal (KDE)

```bash
# KDE uses the default terminal from system settings
# System Settings → Applications → Default Applications → Terminal Emulator

# Or via config file
kwriteconfig5 --file kdeglobals --group General --key TerminalApplication alacritty
```

### Keyboard Shortcut to Launch Terminal

```bash
# GNOME: Settings → Keyboard → Shortcuts → Custom Shortcuts
# Add: Name: Terminal, Command: alacritty, Shortcut: Ctrl+Alt+T

# i3/Sway
# In ~/.config/i3/config or ~/.config/sway/config:
# bindsym $mod+Return exec alacritty
```

---

## Dotfile Management

For a detailed guide on dotfile management strategies, see [08-BEST-PRACTICES.md](08-BEST-PRACTICES.md). Here is a Linux-specific quick start:

### GNU Stow (Recommended for Linux)

```bash
# Install stow
sudo apt install stow    # Debian/Ubuntu
sudo dnf install stow    # Fedora
sudo pacman -S stow      # Arch

# Set up dotfiles directory
mkdir -p ~/dotfiles
cd ~/dotfiles

# Create package directories matching home structure
mkdir -p alacritty/.config/alacritty
mkdir -p nvim/.config/nvim
mkdir -p tmux
mkdir -p zsh

# Move existing configs into the structure
mv ~/.config/alacritty/alacritty.toml alacritty/.config/alacritty/
mv ~/.config/nvim/init.lua nvim/.config/nvim/
mv ~/.tmux.conf tmux/.tmux.conf
mv ~/.zshrc zsh/.zshrc

# Create symlinks
cd ~/dotfiles
stow alacritty    # Symlinks alacritty/.config/alacritty/* → ~/.config/alacritty/
stow nvim         # Symlinks nvim/.config/nvim/* → ~/.config/nvim/
stow tmux         # Symlinks tmux/.tmux.conf → ~/.tmux.conf
stow zsh          # Symlinks zsh/.zshrc → ~/.zshrc

# Version control
cd ~/dotfiles
git init
git add .
git commit -m "Initial dotfiles"
```

---

## Next Steps

- **Shell setup** → [01-SHELL-FUNDAMENTALS.md](01-SHELL-FUNDAMENTALS.md) — Configure Bash, Zsh, or Fish
- **tmux** → [02-TMUX.md](02-TMUX.md) — Terminal multiplexer (essential with Alacritty)
- **Neovim** → [03-NEOVIM.md](03-NEOVIM.md) — Terminal-based editor
- **Productivity tools** → [07-PRODUCTIVITY-TOOLS.md](07-PRODUCTIVITY-TOOLS.md) — Modern CLI tools
- **Best practices** → [08-BEST-PRACTICES.md](08-BEST-PRACTICES.md) — Comprehensive dotfile management strategies

---

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2025 | Initial Linux terminal setup documentation |
