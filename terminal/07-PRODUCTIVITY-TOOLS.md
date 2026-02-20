# Modern CLI Productivity Tools

A comprehensive guide to modern command-line tools that replace or enhance traditional Unix utilities — covering fzf, ripgrep, fd, bat, eza, zoxide, delta, tldr, jq, httpie, direnv, and the GitHub CLI.

---

## Table of Contents

1. [Overview](#overview)
2. [Tool Summary](#tool-summary)
3. [fzf — Fuzzy Finder](#fzf--fuzzy-finder)
4. [ripgrep (rg) — Fast Search](#ripgrep-rg--fast-search)
5. [fd — Find Alternative](#fd--find-alternative)
6. [bat — Better cat](#bat--better-cat)
7. [eza — Better ls](#eza--better-ls)
8. [zoxide — Smarter cd](#zoxide--smarter-cd)
9. [delta — Git Diff Viewer](#delta--git-diff-viewer)
10. [tldr — Simplified Man Pages](#tldr--simplified-man-pages)
11. [jq — JSON Processor](#jq--json-processor)
12. [HTTPie — HTTP Client](#httpie--http-client)
13. [direnv — Directory Environment](#direnv--directory-environment)
14. [gh — GitHub CLI](#gh--github-cli)
15. [Installation Cheat Sheet](#installation-cheat-sheet)
16. [Shell Integration](#shell-integration)
17. [Next Steps](#next-steps)
18. [Version History](#version-history)

---

## Overview

Traditional Unix tools (grep, find, cat, ls) were designed decades ago. Modern replacements are faster, more user-friendly, and provide better defaults. This guide covers the most impactful tools and how to integrate them into your terminal workflow.

### Target Audience

- **Any terminal user** who wants to work faster and more efficiently
- **Developers** looking for better search, navigation, and file inspection tools
- **DevOps engineers** who process JSON, make HTTP requests, and manage Git repositories from the terminal

---

## Tool Summary

| Modern Tool | Replaces | Key Improvement |
|-------------|----------|-----------------|
| **fzf** | `Ctrl+R`, manual selection | Fuzzy search for everything (files, history, processes) |
| **ripgrep** (`rg`) | `grep -r` | 10–100× faster, respects `.gitignore`, better defaults |
| **fd** | `find` | Simpler syntax, faster, respects `.gitignore` |
| **bat** | `cat` | Syntax highlighting, line numbers, git integration |
| **eza** | `ls` | Colours, icons, git status, tree view |
| **zoxide** | `cd` | Learns your habits, jump anywhere by partial name |
| **delta** | `diff`, `git diff` | Syntax-highlighted diffs with line numbers |
| **tldr** | `man` | Practical examples instead of exhaustive documentation |
| **jq** | — | Process, filter, and transform JSON from the command line |
| **httpie** | `curl` | Human-friendly HTTP requests with coloured output |
| **direnv** | manual export | Auto-load environment variables per directory |
| **gh** | GitHub web UI | Full GitHub workflow from the terminal |

---

## fzf — Fuzzy Finder

**fzf** is a general-purpose fuzzy finder that can search through any list of items — files, command history, processes, git branches, and more. It is the single most impactful productivity tool you can add to your terminal.

### Installation

```bash
# macOS
brew install fzf

# Debian/Ubuntu
sudo apt install fzf

# Fedora
sudo dnf install fzf

# From source
git clone --depth 1 https://github.com/junegunn/fzf.git ~/.fzf
~/.fzf/install
```

### Shell Integration

```bash
# Bash — add to ~/.bashrc
eval "$(fzf --bash)"

# Zsh — add to ~/.zshrc
source <(fzf --zsh)

# Fish — add to ~/.config/fish/config.fish
fzf --fish | source

# This enables:
# Ctrl+R — Fuzzy search command history
# Ctrl+T — Fuzzy find files and insert path
# Alt+C  — Fuzzy find directories and cd into them
```

### Configuration

```bash
# Add to ~/.bashrc or ~/.zshrc

# Default fzf options
export FZF_DEFAULT_OPTS="
  --height=40%
  --layout=reverse
  --border=rounded
  --info=inline
  --margin=1
  --padding=1
  --color=bg+:#313244,bg:#1e1e2e,spinner:#f5e0dc,hl:#f38ba8
  --color=fg:#cdd6f4,header:#f38ba8,info:#cba6f7,pointer:#f5e0dc
  --color=marker:#f5e0dc,fg+:#cdd6f4,prompt:#cba6f7,hl+:#f38ba8
  --prompt='❯ '
  --pointer='▶'
  --marker='✓'
"

# Use fd for file finding (faster, respects .gitignore)
export FZF_DEFAULT_COMMAND='fd --type f --hidden --follow --exclude .git'
export FZF_CTRL_T_COMMAND="$FZF_DEFAULT_COMMAND"
export FZF_ALT_C_COMMAND='fd --type d --hidden --follow --exclude .git'

# Preview with bat for Ctrl+T
export FZF_CTRL_T_OPTS="
  --preview 'bat --color=always --style=numbers --line-range=:500 {}'
  --bind 'ctrl-/:change-preview-window(down|hidden|)'
"

# Preview with tree for Alt+C
export FZF_ALT_C_OPTS="--preview 'eza --tree --color=always --level=2 {}'"

# History search with preview
export FZF_CTRL_R_OPTS="
  --preview 'echo {}'
  --preview-window up:3:hidden:wrap
  --bind 'ctrl-/:toggle-preview'
"
```

### Practical fzf Recipes

```bash
# Interactive git branch switching
alias gb='git branch | fzf --preview "git log --oneline --graph --color=always {1}" | xargs git checkout'

# Interactive process kill
alias fkill='ps aux | fzf --header-lines=1 --multi | awk "{print \$2}" | xargs kill'

# Interactive file opener
alias fo='fzf --preview "bat --color=always {}" | xargs -r nvim'

# Git commit browser
alias flog='git log --oneline --graph --color=always | fzf --ansi --preview "git show --color=always {1}" | awk "{print \$1}" | xargs git show'

# Docker container selector
alias dsel='docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Image}}" | fzf --header-lines=1 | awk "{print \$1}"'

# SSH host selector (from ~/.ssh/config)
alias fssh='grep "^Host " ~/.ssh/config | awk "{print \$2}" | fzf | xargs ssh'

# Environment variable browser
alias fenv='env | sort | fzf'
```

---

## ripgrep (rg) — Fast Search

**ripgrep** (command: `rg`) is an extremely fast recursive search tool that respects `.gitignore` by default.

### Installation

```bash
# macOS
brew install ripgrep

# Debian/Ubuntu
sudo apt install ripgrep

# Fedora
sudo dnf install ripgrep
```

### Usage

```bash
# Basic search (recursively searches current directory)
rg "pattern"

# Case-insensitive search
rg -i "pattern"

# Search specific file types
rg -t py "import requests"
rg -t js "useState"
rg -t go "func main"

# Search with context (lines before/after match)
rg -C 3 "ERROR" /var/log/

# Search hidden files (normally skipped)
rg --hidden "secret"

# Search files matching a glob
rg "TODO" -g "*.md"
rg "TODO" -g "!*.test.*"   # Exclude test files

# Count matches per file
rg -c "TODO"

# List files containing pattern (no content)
rg -l "pattern"

# Replace text (preview)
rg "old_name" -r "new_name"

# Regex search
rg -e "v\d+\.\d+\.\d+" CHANGELOG.md

# Multiline search
rg -U "func.*\n.*error"

# JSON output (for scripting)
rg --json "pattern" | jq .
```

### ripgrep Configuration

```bash
# ~/.config/ripgrep/config (or set RIPGREP_CONFIG_PATH)
export RIPGREP_CONFIG_PATH="$HOME/.config/ripgrep/config"
```

```
# ~/.config/ripgrep/config

# Smart case (case-insensitive unless uppercase is used)
--smart-case

# Add line numbers
--line-number

# Add column numbers
--column

# Sort results by file path
--sort=path

# Colour output
--color=auto

# Ignore specific directories
--glob=!.git
--glob=!node_modules
--glob=!vendor
--glob=!.venv
```

---

## fd — Find Alternative

**fd** is a fast, user-friendly alternative to `find`.

### Installation

```bash
# macOS
brew install fd

# Debian/Ubuntu (package name is fd-find, binary is fdfind)
sudo apt install fd-find
# Create alias: ln -s $(which fdfind) ~/.local/bin/fd

# Fedora
sudo dnf install fd-find
```

### Usage

```bash
# Find files by name pattern
fd "readme"                    # Case-insensitive by default
fd "\.md$"                     # Regex pattern

# Find specific file types
fd -e py                       # All .py files
fd -e md -e txt                # All .md and .txt files

# Find directories only
fd -t d "config"

# Find files only
fd -t f "test"

# Find executable files
fd -t x

# Include hidden files
fd -H "env"

# Include ignored files (.gitignore)
fd -I "node_modules"

# Exclude patterns
fd -E "*.pyc" -E "__pycache__"

# Execute command on each result
fd -e py -x wc -l              # Count lines in each Python file
fd -e jpg -x convert {} {.}.png  # Convert all JPGs to PNG

# Find and delete
fd -t f "*.bak" -x rm {}

# Max depth
fd -d 2 "config"               # Only search 2 levels deep

# Change directory root
fd "readme" ~/projects
```

---

## bat — Better cat

**bat** is a `cat` replacement with syntax highlighting, line numbers, and Git integration.

### Installation

```bash
# macOS
brew install bat

# Debian/Ubuntu
sudo apt install bat
# Binary may be 'batcat' — alias it:
# alias bat='batcat'

# Fedora
sudo dnf install bat
```

### Usage

```bash
# View file with syntax highlighting
bat README.md
bat src/main.py

# Show specific line range
bat --line-range 10:20 file.py

# Plain output (no decorations — useful for piping)
bat --plain file.txt
bat -p file.txt

# Show non-printable characters
bat --show-all file.txt

# Use as man page viewer
export MANPAGER="sh -c 'col -bx | bat -l man -p'"
export MANROFFOPT="-c"

# Use as a colourised less
alias less='bat --paging=always'

# List supported languages
bat --list-languages

# Force specific language for syntax highlighting
bat --language=json data.txt

# Concatenate files with headers
bat header.md body.md footer.md
```

### Configuration

```bash
# ~/.config/bat/config

# Theme
--theme="Catppuccin Mocha"

# Show line numbers and grid
--style="numbers,changes,header,grid"

# Use italic text
--italic-text=always

# Map file extensions to languages
--map-syntax "*.ino:C++"
--map-syntax ".ignore:Git Ignore"
--map-syntax "*.conf:INI"
```

---

## eza — Better ls

**eza** (formerly `exa`) is a modern replacement for `ls` with colours, icons, and Git integration.

### Installation

```bash
# macOS
brew install eza

# Debian/Ubuntu
sudo apt install eza

# Fedora
sudo dnf install eza

# Cargo (Rust)
cargo install eza
```

### Usage

```bash
# Basic listing (coloured by default)
eza

# Long format with icons
eza -la --icons

# Tree view
eza --tree --level=2

# Tree view with git status
eza --tree --level=2 --git

# Sort by modification time
eza -la --sort=modified

# Sort by size
eza -la --sort=size

# Show only directories
eza -D

# Show git status for each file
eza -la --git

# Group directories first
eza -la --group-directories-first

# Show headers
eza -la --header
```

### Recommended Aliases

```bash
# Add to ~/.bashrc or ~/.zshrc
alias ls='eza --color=always --group-directories-first --icons'
alias ll='eza -la --color=always --group-directories-first --icons --header --git'
alias lt='eza --tree --level=2 --color=always --group-directories-first --icons'
alias la='eza -a --color=always --group-directories-first --icons'
```

---

## zoxide — Smarter cd

**zoxide** is a smarter `cd` command that learns your most-used directories and lets you jump to them with partial names.

### Installation

```bash
# macOS
brew install zoxide

# Debian/Ubuntu
sudo apt install zoxide

# Fedora
sudo dnf install zoxide
```

### Shell Integration

```bash
# Bash — add to ~/.bashrc
eval "$(zoxide init bash)"

# Zsh — add to ~/.zshrc
eval "$(zoxide init zsh)"

# Fish — add to config.fish
zoxide init fish | source

# This creates the 'z' command (and optionally 'zi' for interactive mode)
```

### Usage

```bash
# Jump to a directory by partial name
z projects         # cd to ~/projects (or wherever you go most)
z awesome          # cd to ~/projects/awesome-learn (learns over time)
z api src          # cd to ~/projects/api/src (multiple keywords)

# Interactive selection with fzf
zi

# List tracked directories and their scores
zoxide query -ls

# Add a directory manually
zoxide add /path/to/dir

# Remove a directory
zoxide remove /path/to/dir
```

---

## delta — Git Diff Viewer

**delta** is a syntax-highlighting pager for `git diff`, `git log`, and `git show`.

### Installation

```bash
# macOS
brew install git-delta

# Debian/Ubuntu
sudo apt install git-delta

# Fedora
sudo dnf install git-delta
```

### Git Configuration

```ini
# ~/.gitconfig

[core]
    pager = delta

[interactive]
    diffFilter = delta --color-only

[delta]
    navigate = true          # Use n/N to jump between diff sections
    light = false            # Dark mode
    side-by-side = true      # Side-by-side diff view
    line-numbers = true      # Show line numbers
    syntax-theme = Catppuccin Mocha

[merge]
    conflictstyle = diff3

[diff]
    colorMoved = default
```

### Usage

```bash
# delta is used automatically with git commands:
git diff                     # Syntax-highlighted diff
git log -p                   # Highlighted commit diffs
git show HEAD                # Highlighted commit details
git blame file.py            # Highlighted blame

# Use delta standalone
diff file1.py file2.py | delta
cat file.py | delta --language=python
```

---

## tldr — Simplified Man Pages

**tldr** provides community-maintained, example-focused documentation for commands.

### Installation

```bash
# macOS
brew install tldr

# Debian/Ubuntu (via npm)
npm install -g tldr

# Rust client (faster)
brew install tealdeer    # macOS
cargo install tealdeer   # any platform
```

### Usage

```bash
# Get practical examples for a command
tldr tar
tldr git-rebase
tldr docker-compose
tldr rsync
tldr ffmpeg

# Update the local cache
tldr --update

# Search for a command
tldr --search "compress"
```

---

## jq — JSON Processor

**jq** is a command-line JSON processor — essential for working with APIs, configuration files, and log data.

### Installation

```bash
# macOS
brew install jq

# Debian/Ubuntu
sudo apt install jq

# Fedora
sudo dnf install jq
```

### Usage

```bash
# Pretty-print JSON
echo '{"name":"alice","age":30}' | jq .

# Extract a field
echo '{"name":"alice","age":30}' | jq '.name'
# Output: "alice"

# Extract raw string (no quotes)
echo '{"name":"alice"}' | jq -r '.name'
# Output: alice

# Array operations
echo '[1,2,3,4,5]' | jq '.[]'           # Each element
echo '[1,2,3,4,5]' | jq '.[2]'          # Third element
echo '[1,2,3,4,5]' | jq 'length'        # Array length
echo '[1,2,3,4,5]' | jq 'map(. * 2)'    # Double each element

# Filter objects in array
echo '[{"name":"alice","age":30},{"name":"bob","age":25}]' | jq '.[] | select(.age > 27)'

# Construct new objects
echo '{"first":"alice","last":"smith"}' | jq '{full_name: (.first + " " + .last)}'

# Process API responses
curl -s https://api.github.com/repos/neovim/neovim | jq '{name, stars: .stargazers_count, language}'

# Process JSONL (line-delimited JSON)
cat logs.jsonl | jq -r 'select(.level == "error") | .message'

# Slurp multiple JSON objects into array
cat file1.json file2.json | jq -s '.'

# Convert JSON to CSV
echo '[{"a":1,"b":2},{"a":3,"b":4}]' | jq -r '.[] | [.a, .b] | @csv'
```

---

## HTTPie — HTTP Client

**HTTPie** is a user-friendly HTTP client for the terminal — an alternative to `curl` with coloured output and intuitive syntax.

### Installation

```bash
# macOS
brew install httpie

# Debian/Ubuntu
sudo apt install httpie

# pip
pip install httpie
```

### Usage

```bash
# GET request
http https://api.github.com/users/octocat

# POST with JSON body
http POST https://api.example.com/items name="Widget" price:=9.99

# POST with form data
http -f POST https://example.com/login username=admin password=secret

# Custom headers
http https://api.example.com Authorization:"Bearer token123"

# Download file
http --download https://example.com/file.zip

# PUT request
http PUT https://api.example.com/items/1 name="Updated Widget"

# DELETE request
http DELETE https://api.example.com/items/1

# Verbose output (show request and response headers)
http -v https://example.com

# Follow redirects
http --follow https://example.com

# Output only headers
http --headers https://example.com

# Session persistence (cookies, auth)
http --session=mysession -a user:pass https://api.example.com/login
http --session=mysession https://api.example.com/protected
```

---

## direnv — Directory Environment

**direnv** automatically loads and unloads environment variables based on the current directory, using a `.envrc` file.

### Installation

```bash
# macOS
brew install direnv

# Debian/Ubuntu
sudo apt install direnv

# Shell integration — add to ~/.bashrc, ~/.zshrc, or config.fish
eval "$(direnv hook bash)"     # Bash
eval "$(direnv hook zsh)"      # Zsh
direnv hook fish | source      # Fish
```

### Usage

```bash
# Create .envrc in a project directory
cd ~/projects/myapp
echo 'export DATABASE_URL="postgres://localhost/myapp"' > .envrc
echo 'export API_KEY="dev-key-12345"' >> .envrc

# Allow direnv to load the file
direnv allow

# Now, when you cd into the directory, variables are automatically set
cd ~/projects/myapp
echo $DATABASE_URL
# postgres://localhost/myapp

# When you leave the directory, variables are unloaded
cd ~
echo $DATABASE_URL
# (empty)

# Load a Python virtual environment
# .envrc
layout python3

# Load Node.js version (with nvm)
# .envrc
use nvm 20

# Source another file
# .envrc
dotenv .env
```

---

## gh — GitHub CLI

**gh** is GitHub's official CLI for managing repositories, issues, pull requests, and Actions from the terminal.

### Installation

```bash
# macOS
brew install gh

# Debian/Ubuntu
sudo apt install gh

# Fedora
sudo dnf install gh

# Authenticate
gh auth login
```

### Usage

```bash
# Repository operations
gh repo clone owner/repo
gh repo create my-new-repo --public
gh repo view --web              # Open in browser

# Pull requests
gh pr create --title "Add feature" --body "Description"
gh pr list
gh pr checkout 42               # Check out PR #42
gh pr merge 42 --squash
gh pr review 42 --approve
gh pr diff 42

# Issues
gh issue create --title "Bug report" --body "Details"
gh issue list --label bug
gh issue close 123
gh issue view 123

# GitHub Actions
gh run list
gh run view 12345
gh run watch 12345              # Live-follow a run
gh run rerun 12345

# Gists
gh gist create file.py --public
gh gist list

# Search
gh search repos "terminal tools" --language=rust --sort=stars

# API requests (authenticated)
gh api repos/owner/repo/issues
gh api graphql -f query='{ viewer { login } }'

# Extensions
gh extension install dlvhdr/gh-dash    # Dashboard for PRs/issues
gh extension install seachicken/gh-poi  # Clean merged branches
```

---

## Installation Cheat Sheet

### macOS (Homebrew)

```bash
brew install fzf ripgrep fd bat eza zoxide git-delta tldr jq httpie direnv gh
```

### Debian/Ubuntu

```bash
sudo apt update && sudo apt install fzf ripgrep fd-find bat eza zoxide jq httpie direnv
# Note: bat may be 'batcat', fd may be 'fdfind'
# Install delta and gh from their respective repos
```

### Fedora

```bash
sudo dnf install fzf ripgrep fd-find bat eza zoxide jq httpie direnv
```

### Cargo (Rust — cross-platform)

```bash
cargo install ripgrep fd-find bat eza zoxide git-delta tealdeer
```

---

## Shell Integration

### Complete .zshrc Integration

```bash
# Add to ~/.zshrc

# ─── Modern Tool Aliases ────────────────────────────────────────
# Replace ls with eza
alias ls='eza --color=always --group-directories-first --icons'
alias ll='eza -la --color=always --group-directories-first --icons --header --git'
alias lt='eza --tree --level=2 --color=always --group-directories-first --icons'
alias la='eza -a --color=always --group-directories-first --icons'

# Replace cat with bat
alias cat='bat --paging=never'

# Replace grep with ripgrep
alias grep='rg'

# Replace find with fd
alias find='fd'

# ─── Tool Initialisation ────────────────────────────────────────
# fzf
source <(fzf --zsh)

# zoxide (creates 'z' command)
eval "$(zoxide init zsh)"

# direnv
eval "$(direnv hook zsh)"

# ─── fzf Configuration ──────────────────────────────────────────
export FZF_DEFAULT_COMMAND='fd --type f --hidden --follow --exclude .git'
export FZF_DEFAULT_OPTS='--height=40% --layout=reverse --border=rounded'
export FZF_CTRL_T_COMMAND="$FZF_DEFAULT_COMMAND"
export FZF_CTRL_T_OPTS="--preview 'bat --color=always --style=numbers --line-range=:500 {}'"
export FZF_ALT_C_COMMAND='fd --type d --hidden --follow --exclude .git'
export FZF_ALT_C_OPTS="--preview 'eza --tree --color=always --level=2 {}'"

# ─── bat Configuration ──────────────────────────────────────────
export BAT_THEME="Catppuccin Mocha"
export MANPAGER="sh -c 'col -bx | bat -l man -p'"

# ─── delta Configuration ────────────────────────────────────────
# Configured via ~/.gitconfig (see delta section above)
```

---

## Next Steps

- **Best practices** → [08-BEST-PRACTICES.md](08-BEST-PRACTICES.md) — Manage all your tool configurations with dotfiles
- **Anti-patterns** → [09-ANTI-PATTERNS.md](09-ANTI-PATTERNS.md) — Common mistakes when using (or not using) these tools
- **Shell setup** → [01-SHELL-FUNDAMENTALS.md](01-SHELL-FUNDAMENTALS.md) — Configure your shell to load these tools properly
- **tmux** → [02-TMUX.md](02-TMUX.md) — fzf integrates beautifully with tmux session switching

---

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2025 | Initial productivity tools documentation |
