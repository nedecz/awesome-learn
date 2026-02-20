# Terminal Tools & Workflow Learning Path

A structured, self-paced training guide to mastering terminal tools and workflows — from shell fundamentals and terminal multiplexing to Neovim, platform-specific setup, and a fully-configured productivity environment. Each phase builds on the previous one, progressing from core concepts to advanced terminal mastery.

> **Time Estimate:** 6–8 weeks at ~5 hours/week. Adjust pace to your experience level. Engineers already comfortable with the command line may skip or accelerate Phase 1.

---

## How to Use This Guide

1. **Follow the phases in order** — each one builds directly on prior knowledge; jumping ahead creates gaps that slow you down later
2. **Read the linked documents** — they contain the detailed content, code examples, and reference tables
3. **Complete the exercises** — hands-on practice is how terminal skills become muscle memory; do not skip them
4. **Check yourself** — use the knowledge check items before moving to the next phase; if you cannot answer them confidently, re-read the relevant sections
5. **Build the capstone project** — the final project ties together all five phases and produces a portable terminal configuration you can use daily

---

## Phase 1: Terminal Foundations (Week 1–2)

### Learning Objectives

- Understand what terminals are, the difference between emulators and shells, and how they communicate via PTY
- Choose and configure a shell (Bash, Zsh, or Fish) with proper environment variables, aliases, and functions
- Write basic shell scripts with error handling
- Manage PATH effectively across platforms
- Customise your prompt with Starship, Oh My Zsh, or Powerlevel10k

### Reading

| Order | Document | Focus Areas |
|-------|----------|-------------|
| 1 | [00-OVERVIEW](00-OVERVIEW.md) | Terminal history, emulators vs shells, PTY, ANSI codes |
| 2 | [01-SHELL-FUNDAMENTALS](01-SHELL-FUNDAMENTALS.md) | Shell configuration, env vars, aliases, functions, scripting |

### Exercises

**1. Terminal Architecture Exploration:**

Explore the terminal stack on your system:

```bash
# Identify your terminal emulator
echo $TERM_PROGRAM         # Often set by the emulator
echo $TERM                 # Terminal type for capability detection

# Identify your shell
echo $SHELL                # Login shell
echo $0                    # Current shell

# Explore your PTY
tty                        # Show your PTY device
ls -la $(tty)              # View PTY file details
who                        # Show all active terminal sessions

# Check terminal capabilities
tput colors                # Number of supported colours
tput cols                  # Terminal width in columns
tput lines                 # Terminal height in lines

# Test ANSI colour support
for i in {0..255}; do printf "\033[38;5;${i}m%3d " "$i"; [ $(( (i+1) % 16 )) -eq 0 ] && echo; done
echo -e "\033[0m"
```

Document your findings: What terminal emulator, shell, PTY, and TERM value are you using?

**2. Shell Configuration:**

Set up your shell properly:

```bash
# Measure your current shell startup time
time zsh -i -c exit     # or: time bash -i -c exit
# Record this as your baseline

# Create a modular configuration structure:
mkdir -p ~/.config/shell

# Create each module:
# ~/.config/shell/aliases.sh   — 10+ useful aliases
# ~/.config/shell/functions.sh — 3+ shell functions (mkcd, extract, serve)
# ~/.config/shell/path.sh      — PATH management with duplicate prevention
# ~/.config/shell/prompt.sh    — Prompt configuration (install Starship)

# Source everything from your .zshrc/.bashrc:
# for f in ~/.config/shell/*.sh; do source "$f"; done

# Measure startup time again — it should still be < 200ms
time zsh -i -c exit
```

**3. Basic Shell Scripting:**

Write a script called `setup-project.sh` that:
- Accepts a project name as an argument
- Creates a project directory with `src/`, `tests/`, and `docs/` subdirectories
- Initialises a Git repository
- Creates a basic README.md
- Displays a success message with the full path

Use `set -euo pipefail`, proper argument validation, and a `die()` function for errors.

### Knowledge Check

- [ ] Can you explain the data flow from keystroke to command execution (terminal emulator → PTY → shell → kernel)?
- [ ] What is the difference between `.bash_profile` and `.bashrc`? Between `.zshenv` and `.zshrc`?
- [ ] Can you write a shell function that accepts arguments and handles errors?
- [ ] What does `set -euo pipefail` do in a script?
- [ ] Is your shell startup time under 200ms?

---

## Phase 2: tmux Mastery (Week 3)

### Learning Objectives

- Install and configure tmux with an ergonomic prefix key and key bindings
- Manage sessions, windows, and panes efficiently
- Use copy mode with vim keybindings for text selection and clipboard integration
- Set up tmux plugins with TPM (resurrect, continuum, vim-tmux-navigator)
- Write tmux scripts for automated project layouts

### Reading

| Order | Document | Focus Areas |
|-------|----------|-------------|
| 1 | [02-TMUX](02-TMUX.md) | All sections — core concepts through scripting |
| 2 | [08-BEST-PRACTICES](08-BEST-PRACTICES.md) | Terminal multiplexer workflows, ergonomic key bindings |

### Exercises

**1. tmux Configuration:**

Create a complete `~/.tmux.conf` with:
- Prefix key changed to `Ctrl+a`
- Mouse support enabled
- True colour support
- Vi-style copy mode with system clipboard integration
- Intuitive split keys (`|` for vertical, `-` for horizontal)
- Vim-style pane navigation (`h/j/k/l`)
- Status bar with session name, window list, and time

Test each setting by reloading the config (`<prefix> r`).

**2. Session Management:**

Practice the following workflow:

```bash
# Create three named sessions
tmux new -s project-a -d
tmux new -s project-b -d
tmux new -s notes -d

# In project-a: Create 3 windows (editor, server, shell)
# In project-b: Create 2 windows with split panes
# In notes: Single window

# Practice switching between sessions with <prefix> s
# Practice detaching and reattaching

# Kill all sessions and verify they're gone
tmux kill-server
tmux ls  # Should show "no server running"
```

**3. tmux Startup Script:**

Write a `tmux-dev.sh` script that creates a development session with:
- Window 0 "editor": Starts Neovim (or your editor)
- Window 1 "shell": A general-purpose shell
- Window 2 "git": Two vertical panes (git log on left, git status on right)
- Window 3 "server": For running a development server

The script should check if the session already exists and attach to it instead of creating a duplicate.

**4. tmux Plugin Setup:**

Install TPM and configure these plugins:
- tmux-resurrect (save/restore sessions)
- tmux-continuum (automatic saving)
- tmux-yank (clipboard integration)

Test session persistence: save with `<prefix> Ctrl+s`, kill tmux, restart, and restore with `<prefix> Ctrl+r`.

### Knowledge Check

- [ ] Can you create sessions, windows, and panes without looking up commands?
- [ ] Can you use copy mode to select and copy text to system clipboard?
- [ ] Does your tmux configuration include all recommended ergonomic settings?
- [ ] Can you write a tmux script that automates a multi-window layout?
- [ ] Are tmux-resurrect and tmux-continuum working correctly?

---

## Phase 3: Neovim Setup (Week 4–5)

### Learning Objectives

- Install Neovim and understand its configuration directory structure
- Write a Lua-based configuration with proper module organisation
- Set up lazy.nvim for plugin management
- Configure essential plugins: Telescope, Treesitter, LSP, nvim-cmp, lualine
- Use Neovim as an IDE with code completion, diagnostics, and navigation
- Navigate between Neovim splits and tmux panes seamlessly
- Evaluate Neovim distributions (LazyVim, NvChad, AstroNvim, kickstart.nvim) as alternatives to building from scratch
- Begin building Vim keybinding muscle memory across tools

### Reading

| Order | Document | Focus Areas |
|-------|----------|-------------|
| 1 | [03-NEOVIM](03-NEOVIM.md) | All sections — configuration through tmux integration |
| 2 | [10-VIM-DISTRIBUTIONS-AND-KEYBINDINGS](10-VIM-DISTRIBUTIONS-AND-KEYBINDINGS.md) | Neovim distributions, Vim keybinding learning resources, motions cheat sheet |
| 3 | [02-TMUX](02-TMUX.md) | vim-tmux-navigator section |

### Exercises

**1. Neovim Configuration from Scratch:**

Build your Neovim configuration step by step:

```
Step 1: Create the directory structure
  ~/.config/nvim/
  ├── init.lua
  └── lua/
      ├── config/
      │   ├── options.lua
      │   ├── keymaps.lua
      │   ├── autocmds.lua
      │   └── lazy.lua
      └── plugins/
          └── (empty for now)

Step 2: Configure options.lua (line numbers, tabs, search, clipboard)
Step 3: Configure keymaps.lua (leader key, window nav, buffer nav)
Step 4: Configure autocmds.lua (highlight yank, trim whitespace)
Step 5: Bootstrap lazy.nvim in lazy.lua
```

Verify each step works before proceeding.

**2. Plugin Installation:**

Add plugins one at a time, testing each before adding the next:

```
1. Colourscheme (catppuccin) — verify colours change
2. Lualine — verify status bar appears
3. Telescope — verify <leader>ff finds files
4. Treesitter — verify syntax highlighting improves
5. LSP (mason + nvim-lspconfig) — verify code intelligence works
6. nvim-cmp — verify autocompletion appears
7. which-key — verify leader key hints appear
8. vim-tmux-navigator — verify Ctrl+h/j/k/l works across tmux/nvim
```

**3. Language-Specific LSP:**

Configure at least two language servers for languages you use:

```
For each language:
1. Install the LSP server via Mason (:Mason)
2. Verify hover documentation (K)
3. Verify go-to-definition (gd)
4. Verify find references (gr)
5. Verify code actions (<leader>ca)
6. Verify diagnostics appear inline
```

**4. Workflow Integration:**

Practice these tasks entirely in Neovim:
- Find a file by name (Telescope `<leader>ff`)
- Search for text across the project (Telescope `<leader>fg`)
- Navigate to a function definition and back
- Rename a symbol across files
- Run a shell command from Neovim's built-in terminal

### Knowledge Check

- [ ] Can you explain the Neovim configuration directory structure?
- [ ] Can you add a new plugin to your lazy.nvim configuration?
- [ ] Can you navigate code with LSP (go to definition, find references)?
- [ ] Does autocompletion work with LSP sources?
- [ ] Can you seamlessly navigate between tmux panes and Neovim splits?

---

## Phase 4: Platform-Specific Setup (Week 5–6)

### Learning Objectives

- Configure your terminal emulator for your specific platform (Windows, Linux, or macOS)
- Install and configure a Nerd Font
- Set up platform-specific tools and package managers
- Ensure clipboard integration works across terminal, tmux, and Neovim
- Make your configuration portable with platform detection

### Reading

| Order | Document | Focus Areas |
|-------|----------|-------------|
| 1 | Your platform guide: [04-WINDOWS-TERMINAL](04-WINDOWS-TERMINAL.md), [05-LINUX-TERMINAL](05-LINUX-TERMINAL.md), or [06-MACOS-TERMINAL](06-MACOS-TERMINAL.md) | All sections |
| 2 | [08-BEST-PRACTICES](08-BEST-PRACTICES.md) | Portable configurations, clipboard integration |

### Exercises

**1. Terminal Emulator Configuration:**

Configure your terminal emulator of choice:
- Install and set a Nerd Font (JetBrains Mono recommended)
- Apply a colour scheme (Catppuccin Mocha recommended for consistency)
- Configure opacity/transparency (if desired)
- Set up keyboard shortcuts for tabs/splits (if your emulator supports them)
- Ensure true colour support works:

```bash
# Test true colour
printf "\x1b[38;2;255;100;0mTrue colour test\x1b[0m\n"
```

**2. Package Manager Setup:**

Set up your platform's package manager and install all tools:

```bash
# macOS: Homebrew
brew install fzf ripgrep fd bat eza zoxide git-delta jq gh tmux neovim starship

# Linux (Debian/Ubuntu):
sudo apt install fzf ripgrep fd-find bat eza zoxide tmux neovim

# Windows: Scoop
scoop install fzf ripgrep fd bat eza zoxide delta jq gh neovim
```

Create a reproducible package list (Brewfile, apt list, or scoop export).

**3. Clipboard Integration Test:**

Verify clipboard works end-to-end:

```bash
# Shell → clipboard
echo "test from shell" | clip    # (use your platform's clip alias)
paste                              # Should output "test from shell"

# tmux copy mode → clipboard
# Enter copy mode, select text, yank — verify it's in system clipboard

# Neovim → clipboard
# Yank text in Neovim — verify it's in system clipboard (Ctrl+V outside terminal)
```

**4. Cross-Platform Config:**

Add platform detection to your shell configuration:
- Conditional Homebrew loading (macOS only)
- Conditional clipboard aliases
- Conditional tool alternatives (e.g., `batcat` vs `bat` on Ubuntu)

### Knowledge Check

- [ ] Is your terminal emulator configured with a Nerd Font and colour scheme?
- [ ] Does true colour work in your terminal, tmux, and Neovim?
- [ ] Can you copy text from tmux/Neovim to the system clipboard?
- [ ] Do you have a reproducible package list for your platform?
- [ ] Does your shell configuration work on both your primary and secondary machines?

---

## Phase 5: Productivity Tools & Polish (Week 6–7)

### Learning Objectives

- Install and configure fzf, ripgrep, fd, bat, eza, zoxide, and delta
- Integrate fzf with shell history, file finding, and directory jumping
- Configure delta as the Git diff viewer
- Set up direnv for per-project environment variables
- Master the gh CLI for GitHub workflows from the terminal

### Reading

| Order | Document | Focus Areas |
|-------|----------|-------------|
| 1 | [07-PRODUCTIVITY-TOOLS](07-PRODUCTIVITY-TOOLS.md) | All tools — installation, configuration, integration |
| 2 | [09-ANTI-PATTERNS](09-ANTI-PATTERNS.md) | All anti-patterns — self-assessment |

### Exercises

**1. fzf Mastery:**

Configure fzf with all integrations and practice:

```bash
# Shell integration
source <(fzf --zsh)

# Configure FZF_DEFAULT_COMMAND, FZF_CTRL_T_OPTS, FZF_ALT_C_OPTS
# (see 07-PRODUCTIVITY-TOOLS.md for full configuration)

# Practice:
# Ctrl+R — search history for a command from last week
# Ctrl+T — find a deeply nested file and insert its path
# Alt+C  — jump to a project directory

# Create 3 custom fzf functions:
# 1. Interactive git branch switcher
# 2. Interactive process killer
# 3. Interactive file opener with bat preview
```

**2. Git Integration:**

Configure delta and practice:

```bash
# Add to ~/.gitconfig:
# [core] pager = delta
# [delta] side-by-side = true, line-numbers = true

# Practice:
git diff                    # Should show syntax-highlighted side-by-side diff
git log -p -5               # Should show highlighted commit diffs
git blame some-file.py      # Should show highlighted blame
```

**3. Anti-Pattern Self-Assessment:**

Go through [09-ANTI-PATTERNS.md](09-ANTI-PATTERNS.md) and score yourself:
- For each of the 12 anti-patterns, rate yourself: ✅ Fixed, ⚠️ Partial, ❌ Need to fix
- Create an action plan for any ❌ items

**4. Full Workflow Practice:**

Complete a realistic development task using only your terminal (no GUI IDE):
- Clone a repository
- Create a feature branch
- Navigate the codebase with Telescope
- Edit code with Neovim (with LSP assistance)
- Run tests from a tmux pane
- Review your changes with delta (`git diff`)
- Commit and push
- Create a pull request with `gh pr create`

### Knowledge Check

- [ ] Can you use fzf Ctrl+R, Ctrl+T, and Alt+C without thinking?
- [ ] Is delta configured as your Git diff viewer?
- [ ] Have you addressed all critical anti-patterns from the self-assessment?
- [ ] Can you complete a full development workflow entirely in the terminal?
- [ ] Are all your modern tools (rg, fd, bat, eza) configured with aliases?

---

## Capstone Project: Portable Terminal Environment

### Objective

Create a fully-configured, version-controlled, cross-platform terminal environment that you can deploy to any new machine in under 5 minutes.

### Requirements

1. **Dotfile Repository** — All configuration files managed with chezmoi (or GNU Stow / bare git)
2. **Shell Configuration** — Modular `.zshrc` with platform detection, lazy loading, and < 200ms startup
3. **tmux Configuration** — Ergonomic keybindings, vim-tmux-navigator, plugins (resurrect, continuum)
4. **Neovim Configuration** — Lua-based config with LSP, Treesitter, Telescope, autocompletion
5. **Terminal Emulator Config** — Colour scheme, Nerd Font, true colour, keyboard settings
6. **Bootstrap Script** — Automated setup script that installs all tools and applies dotfiles
7. **Documentation** — README.md in your dotfiles repo explaining the setup

### Acceptance Criteria

- [ ] Running the bootstrap script on a fresh machine installs all tools and applies all configs
- [ ] Shell starts in < 200ms
- [ ] tmux sessions persist across restarts (resurrect + continuum)
- [ ] Neovim has working LSP for at least 2 languages
- [ ] Clipboard works: shell → system, tmux → system, Neovim → system
- [ ] Ctrl+h/j/k/l navigates seamlessly between tmux panes and Neovim splits
- [ ] fzf is integrated with shell history (Ctrl+R), file finding (Ctrl+T), and directory jumping (Alt+C)
- [ ] All modern CLI tools are installed and aliased (rg, fd, bat, eza, zoxide, delta)
- [ ] Configuration is portable: the same dotfile repo works on macOS and Linux (or your primary platforms)
- [ ] Dotfile repo is pushed to GitHub with a clear README

### Stretch Goals

- [ ] Git hooks that lint your shell scripts before committing
- [ ] GitHub Actions CI that validates your dotfiles (shellcheck for scripts, lua-check for Neovim config)
- [ ] Screenshot or recording demonstrating your complete workflow
- [ ] Template support in chezmoi for machine-specific differences

---

## Recommended Reading Order by Role

```
Developer:
  00-OVERVIEW → 01-SHELL-FUNDAMENTALS → 02-TMUX → 03-NEOVIM → 10-VIM-DISTRIBUTIONS-AND-KEYBINDINGS → 07-PRODUCTIVITY-TOOLS

DevOps / SRE:
  00-OVERVIEW → 01-SHELL-FUNDAMENTALS → 02-TMUX → 07-PRODUCTIVITY-TOOLS → 08-BEST-PRACTICES

New to Terminal:
  00-OVERVIEW → 01-SHELL-FUNDAMENTALS → Platform Guide (04/05/06) → 02-TMUX → LEARNING-PATH exercises

Experienced User (Optimising):
  09-ANTI-PATTERNS → 08-BEST-PRACTICES → 07-PRODUCTIVITY-TOOLS → 03-NEOVIM → 10-VIM-DISTRIBUTIONS-AND-KEYBINDINGS
```

---

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2025 | Initial terminal learning path documentation |
