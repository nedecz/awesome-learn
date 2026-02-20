# Terminal Tools & Workflow Learning Resources

A comprehensive guide to terminal tools and workflows — from shell fundamentals and terminal emulators to tmux, Neovim, platform-specific setup, modern CLI tools, and productivity-optimised configurations.

## 📚 Documentation Structure

| Document | Description | When to Read |
|----------|-------------|--------------|
| [00-OVERVIEW](00-OVERVIEW.md) | What terminals are, emulators vs shells, history (TTY/PTY), ANSI codes | **Start here** |
| [01-SHELL-FUNDAMENTALS](01-SHELL-FUNDAMENTALS.md) | Bash, Zsh, Fish shells, configuration, scripting, prompt customisation | When setting up your shell environment |
| [02-TMUX](02-TMUX.md) | Terminal multiplexer: sessions, windows, panes, plugins, scripting | When managing multiple terminal sessions |
| [03-NEOVIM](03-NEOVIM.md) | Neovim as IDE: Lua config, plugins, LSP, debugging, navigation | When setting up a terminal-based editor |
| [04-WINDOWS-TERMINAL](04-WINDOWS-TERMINAL.md) | Windows Terminal, WSL2, PowerShell, settings.json, Windows tools | When working on Windows |
| [05-LINUX-TERMINAL](05-LINUX-TERMINAL.md) | Alacritty, kitty, WezTerm, GPU terminals, X11/Wayland, dotfiles | When working on Linux |
| [06-MACOS-TERMINAL](06-MACOS-TERMINAL.md) | iTerm2, Alacritty, Homebrew, macOS shell config, key bindings | When working on macOS |
| [07-PRODUCTIVITY-TOOLS](07-PRODUCTIVITY-TOOLS.md) | fzf, ripgrep, fd, bat, eza, zoxide, delta, jq, httpie, gh CLI | **Essential — modern CLI replacements** |
| [08-BEST-PRACTICES](08-BEST-PRACTICES.md) | Dotfile management, portable configs, ergonomic key bindings | **Essential — workflow optimisation** |
| [09-ANTI-PATTERNS](09-ANTI-PATTERNS.md) | Common terminal mistakes and how to avoid them | **Essential — what NOT to do** |
| [LEARNING-PATH](LEARNING-PATH.md) | Structured learning guide with exercises and capstone project | **Start here** after the Overview |

## 🚀 Quick Start

### For Beginners

1. **Read the Overview** ([00-OVERVIEW](00-OVERVIEW.md))
   - Understand what terminals are and how they evolved from TTYs
   - Learn the difference between terminal emulators and shells
   - Explore ANSI escape codes and terminal protocols

2. **Set Up Your Shell** ([01-SHELL-FUNDAMENTALS](01-SHELL-FUNDAMENTALS.md))
   - Choose and configure your shell (Bash, Zsh, or Fish)
   - Set up environment variables, aliases, and functions
   - Customise your prompt with Starship or Oh My Zsh

3. **Learn tmux** ([02-TMUX](02-TMUX.md))
   - Install and configure your first tmux session
   - Master sessions, windows, and panes
   - Set up persistent sessions with resurrect/continuum

4. **Follow the Learning Path** ([LEARNING-PATH](LEARNING-PATH.md))
   - Structured curriculum with hands-on exercises
   - Progressive skill building from basics to advanced workflows

### For Experienced Users

1. **Review Best Practices** ([08-BEST-PRACTICES](08-BEST-PRACTICES.md))
   - Dotfile management with chezmoi or GNU stow
   - Portable configurations across platforms
   - Ergonomic key bindings and efficient navigation

2. **Avoid Anti-Patterns** ([09-ANTI-PATTERNS](09-ANTI-PATTERNS.md))
   - Common terminal workflow mistakes
   - Platform-specific pitfalls and their solutions

3. **Set Up Neovim as IDE** ([03-NEOVIM](03-NEOVIM.md))
   - Lua-based configuration with lazy.nvim
   - LSP, Treesitter, Telescope, and debugging with DAP
   - Terminal integration within Neovim

4. **Upgrade CLI Tools** ([07-PRODUCTIVITY-TOOLS](07-PRODUCTIVITY-TOOLS.md))
   - Modern replacements for traditional Unix commands
   - fzf, ripgrep, bat, eza, zoxide, delta, and more

## 🏗️ Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                        User Input                                │
│          Keyboard → Terminal Emulator → Shell → Kernel           │
└───────────────────────────┬─────────────────────────────────────┘
                            │
┌───────────────────────────▼─────────────────────────────────────┐
│                    Terminal Emulator Layer                        │
│   ┌─────────────────┐  ┌──────────────┐  ┌───────────────────┐  │
│   │ Windows Terminal│  │   iTerm2     │  │    Alacritty      │  │
│   │ (Windows)       │  │   (macOS)    │  │  (Cross-platform) │  │
│   └────────┬────────┘  └──────┬───────┘  └────────┬──────────┘  │
│            │                  │                    │             │
│            └──────────────────┼────────────────────┘             │
│                               │                                  │
│                    ┌──────────▼──────────┐                       │
│                    │   PTY (Pseudo-TTY)  │                       │
│                    │   Kernel Interface  │                       │
│                    └──────────┬──────────┘                       │
└───────────────────────────────│──────────────────────────────────┘
                                │
┌───────────────────────────────▼──────────────────────────────────┐
│                         Shell Layer                               │
│   ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌───────────────┐   │
│   │   Bash   │  │   Zsh    │  │   Fish   │  │  PowerShell   │   │
│   │  (.bashrc│  │ (.zshrc) │  │(config.  │  │  (profile.ps1)│   │
│   │)        │  │          │  │  fish)   │  │               │   │
│   └────┬─────┘  └────┬─────┘  └────┬─────┘  └──────┬────────┘   │
│        └──────────────┼─────────────┼───────────────┘            │
│                       │             │                             │
│              ┌────────▼─────────────▼────────┐                   │
│              │     Terminal Multiplexer       │                   │
│              │          (tmux)                │                   │
│              │  Sessions → Windows → Panes   │                   │
│              └───────────────┬────────────────┘                   │
└──────────────────────────────│───────────────────────────────────┘
                               │
┌──────────────────────────────▼───────────────────────────────────┐
│                      Application Layer                            │
│   ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌───────────────┐   │
│   │  Neovim  │  │   fzf    │  │  ripgrep │  │   git / gh    │   │
│   │  (Editor)│  │ (Fuzzy   │  │ (Search) │  │   (Version    │   │
│   │          │  │  Finder) │  │          │  │    Control)   │   │
│   └──────────┘  └──────────┘  └──────────┘  └───────────────┘   │
└──────────────────────────────────────────────────────────────────┘
```

## 🔑 Key Concepts

```
Terminal Fundamentals
─────────────────────
Terminal Emulator → GUI application that emulates a hardware terminal (e.g., Alacritty, iTerm2)
Shell            → Command-line interpreter that processes user commands (e.g., Bash, Zsh, Fish)
PTY              → Pseudo-terminal; kernel interface connecting emulator to shell
ANSI Codes       → Escape sequences for controlling terminal output (colours, cursor, formatting)

Terminal Multiplexer
────────────────────
Session  → A named collection of windows, persists across disconnections
Window   → A full-screen view within a session, like a tab
Pane     → A split within a window, showing its own shell instance

Editor (Neovim)
───────────────
LSP        → Language Server Protocol; provides code intelligence (completion, diagnostics)
Treesitter → Incremental parser for syntax highlighting and code analysis
lazy.nvim  → Modern plugin manager for Neovim with lazy-loading support
DAP        → Debug Adapter Protocol; enables debugging within Neovim

Productivity Tools
──────────────────
fzf     → Fuzzy finder for files, commands, history, and more
ripgrep → Ultra-fast regex search tool (replaces grep)
bat     → cat with syntax highlighting and line numbers
eza     → Modern ls replacement with colours, icons, and git status
zoxide  → Smarter cd that learns your most-visited directories
delta   → Syntax-highlighting pager for git diffs
```

## 📋 Topics Covered

- **Foundations** — Terminal history, emulators vs shells, TTY/PTY, ANSI escape codes
- **Shell Fundamentals** — Bash, Zsh, Fish configuration, scripting, prompt customisation
- **tmux** — Sessions, windows, panes, key bindings, plugins, session persistence
- **Neovim** — Lua configuration, plugin management, LSP, debugging, navigation
- **Windows Terminal** — Settings, WSL2 integration, PowerShell, Windows package managers
- **Linux Terminal** — GPU-accelerated emulators, X11/Wayland, font and dotfile management
- **macOS Terminal** — iTerm2, Homebrew, macOS-specific configuration and key bindings
- **Productivity Tools** — fzf, ripgrep, fd, bat, eza, zoxide, delta, jq, gh CLI
- **Best Practices** — Dotfile management, portable configs, ergonomic workflows
- **Anti-Patterns** — Common terminal workflow mistakes and how to avoid them

## 🤝 Contributing

This is a living collection of learning resources. Contributions are welcome — see the repository [CONTRIBUTING.md](../CONTRIBUTING.md) for guidelines.

## 🏁 Next Steps

**New to terminals?** → Start with [00-OVERVIEW.md](00-OVERVIEW.md) then follow [LEARNING-PATH.md](LEARNING-PATH.md)

**Already comfortable with the terminal?** → Jump to [02-TMUX.md](02-TMUX.md) or [03-NEOVIM.md](03-NEOVIM.md)

**Setting up a new machine?** → Read the platform-specific guide ([04-WINDOWS-TERMINAL.md](04-WINDOWS-TERMINAL.md), [05-LINUX-TERMINAL.md](05-LINUX-TERMINAL.md), or [06-MACOS-TERMINAL.md](06-MACOS-TERMINAL.md))

**Want a structured path?** → Follow the [LEARNING-PATH.md](LEARNING-PATH.md) — progressive exercises from basics to a fully-configured terminal environment
