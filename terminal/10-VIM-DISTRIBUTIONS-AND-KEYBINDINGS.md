# Vim Distributions & Keybindings

A comprehensive guide to Neovim distributions (LazyVim, NvChad, AstroNvim, kickstart.nvim) and using Vim keybindings across editors, browsers, shells, and terminal tools — covering installation, customisation, comparison, and building muscle memory.

---

## Table of Contents

1. [Overview](#overview)
2. [Neovim Distributions](#neovim-distributions)
3. [Learning Vim Keybindings](#learning-vim-keybindings)
4. [Vim Keybindings in Other Tools](#vim-keybindings-in-other-tools)
5. [Essential Vim Motions Cheat Sheet](#essential-vim-motions-cheat-sheet)
6. [Building Muscle Memory](#building-muscle-memory)
7. [Next Steps](#next-steps)
8. [Version History](#version-history)

---

## Overview

Building a Neovim configuration from scratch — as covered in [03-NEOVIM.md](03-NEOVIM.md) — is an excellent way to understand how the editor works. However, pre-configured **Neovim distributions** provide a polished, IDE-like experience out of the box, letting you focus on writing code rather than configuring your editor.

This guide covers two complementary topics:

1. **Neovim distributions** — curated configurations that bundle plugins, keymaps, and sensible defaults into a ready-to-use package
2. **Vim keybindings everywhere** — how to carry your Vim muscle memory into VS Code, JetBrains IDEs, browsers, shells, and terminal tools

### Who This Is For

- **Newcomers** who want a working Neovim IDE without weeks of configuration
- **Experienced Vim users** who want to evaluate distribution options
- **Developers** who want consistent Vim keybindings across their entire workflow

---

## Neovim Distributions

### LazyVim

**LazyVim** is a pre-configured Neovim setup created by folke, the author of lazy.nvim. It uses lazy.nvim under the hood and provides a batteries-included experience with a clean architecture for customisation.

**Link:** [https://www.lazyvim.org/](https://www.lazyvim.org/)

#### Installation

```bash
# Back up existing config
mv ~/.config/nvim{,.bak}
mv ~/.local/share/nvim{,.bak}
mv ~/.local/state/nvim{,.bak}
mv ~/.cache/nvim{,.bak}

# Clone the LazyVim starter template
git clone https://github.com/LazyVim/starter ~/.config/nvim

# Remove the .git directory so you can add your own version control
rm -rf ~/.config/nvim/.git

# Launch Neovim — plugins install automatically
nvim
```

#### Key Features

| Feature | Description |
|---------|-------------|
| lazy.nvim | Plugin manager with lazy-loading and a lockfile |
| LSP | Pre-configured with mason.nvim, lspconfig, and sensible defaults |
| Treesitter | Syntax highlighting, text objects, and incremental selection |
| Telescope / fzf | Fuzzy finding for files, grep, buffers, and more |
| which-key | Popup that shows available keybindings as you type |
| Extras system | Optional plugin bundles you can enable per-language or feature |
| neo-tree | File explorer sidebar |
| mini.nvim | Collection of small, focused modules (pairs, surround, etc.) |

#### Directory Structure

```
~/.config/nvim/
├── init.lua                  # Entry point (do not edit)
├── lazyvim.json              # Enabled extras
├── lazy-lock.json            # Plugin version lockfile
├── stylua.toml               # Lua formatter config
└── lua/
    ├── config/
    │   ├── autocmds.lua      # Custom autocommands
    │   ├── keymaps.lua       # Custom key mappings
    │   ├── lazy.lua          # lazy.nvim bootstrap (do not edit)
    │   └── options.lua       # Custom vim options
    └── plugins/
        ├── example.lua       # Starter example (safe to modify)
        └── ...               # Add your plugin specs here
```

#### Customisation

Add plugins by creating files in `lua/plugins/`:

```lua
-- ~/.config/nvim/lua/plugins/custom.lua

return {
    -- Add a plugin
    {
        "kylechui/nvim-surround",
        version = "*",
        event = "VeryLazy",
        config = true,
    },

    -- Override a LazyVim default
    {
        "nvim-neo-tree/neo-tree.nvim",
        opts = {
            filesystem = {
                filtered_items = { visible = true },
            },
        },
    },

    -- Disable a LazyVim default plugin
    { "echasnovski/mini.pairs", enabled = false },
}
```

Override options and keymaps in the corresponding config files:

```lua
-- ~/.config/nvim/lua/config/options.lua
vim.opt.relativenumber = true
vim.opt.scrolloff = 10
vim.opt.tabstop = 4
vim.opt.shiftwidth = 4
```

#### Default Keybindings

LazyVim uses `<Space>` as the leader key. Press `<Space>` and wait for which-key to show available mappings.

| Mapping | Action |
|---------|--------|
| `<Space>` | Leader key — opens which-key menu |
| `<Space>ff` | Find files (Telescope) |
| `<Space>fg` | Live grep (Telescope) |
| `<Space>fb` | Find buffers |
| `<Space>e` | Toggle file explorer (neo-tree) |
| `<Space>gg` | Open lazygit |
| `<Space>cf` | Format document |
| `<Space>cr` | Rename symbol (LSP) |
| `<Space>ca` | Code actions (LSP) |
| `gd` | Go to definition |
| `gr` | Go to references |
| `K` | Hover documentation |
| `]d` / `[d` | Next / previous diagnostic |

#### LazyVim Extras

Enable language support and additional features via the extras system:

```bash
# In Neovim, open the extras manager
:LazyExtras
```

Popular extras include:

| Extra | What It Adds |
|-------|-------------|
| `lang.typescript` | TypeScript LSP, linting, formatting |
| `lang.python` | pyright, ruff, debugpy |
| `lang.rust` | rust-analyzer, crates.nvim |
| `lang.go` | gopls, gofumpt, delve |
| `lang.docker` | Dockerfile LSP and Compose support |
| `editor.mini-files` | Alternative file explorer |
| `test.core` | Neotest integration |
| `dap.core` | Debug adapter protocol |

---

### NvChad

**NvChad** is a Neovim configuration focused on aesthetics and performance. It features a custom UI layer, a rich theme system, and a structured approach to customisation.

**Link:** [https://nvchad.com/](https://nvchad.com/)

#### Installation

```bash
# Back up existing config
mv ~/.config/nvim{,.bak}
mv ~/.local/share/nvim{,.bak}

# Clone the NvChad starter template
git clone https://github.com/NvChad/starter ~/.config/nvim

# Launch Neovim — plugins install automatically
nvim
```

#### Key Features

| Feature | Description |
|---------|-------------|
| nvchad/ui | Custom UI for statusline, tabufline, and terminal |
| base46 | Theme engine with 50+ built-in colourschemes |
| Mason | LSP/linter/formatter installer integrated by default |
| Telescope | Fuzzy finder for files, grep, themes, and more |
| Treesitter | Syntax highlighting out of the box |
| which-key | Keybinding discovery popup |
| nvterm | Built-in terminal management |

#### Directory Structure

```
~/.config/nvim/
├── init.lua                  # Entry point
├── lazy-lock.json            # Plugin lockfile
└── lua/
    ├── chadrc.lua            # NvChad configuration (theme, UI)
    ├── mappings.lua          # Custom key mappings
    ├── options.lua           # Custom vim options
    └── plugins/
        └── init.lua          # Custom plugin specifications
```

#### Customisation

Configure the theme and UI in `chadrc.lua`:

```lua
-- ~/.config/nvim/lua/chadrc.lua

---@type ChadrcConfig
local M = {}

M.base46 = {
    theme = "onedark",
    transparency = false,
    hl_override = {
        Comment = { italic = true },
    },
}

M.ui = {
    statusline = { theme = "vscode_colored" },
    tabufline = { enabled = true },
}

return M
```

Add custom plugins:

```lua
-- ~/.config/nvim/lua/plugins/init.lua

return {
    {
        "neovim/nvim-lspconfig",
        config = function()
            require("nvchad.configs.lspconfig").defaults()

            local lspconfig = require("lspconfig")
            local servers = { "ts_ls", "pyright", "gopls" }

            for _, lsp in ipairs(servers) do
                lspconfig[lsp].setup({})
            end
        end,
    },

    {
        "nvim-treesitter/nvim-treesitter",
        opts = {
            ensure_installed = {
                "lua", "javascript", "typescript",
                "python", "go", "rust", "html", "css",
            },
        },
    },
}
```

#### Default Keybindings

NvChad uses `<Space>` as the leader key.

| Mapping | Action |
|---------|--------|
| `<Space>ch` | Open cheatsheet (all keybindings) |
| `<Space>th` | Browse and switch themes |
| `<Space>ff` | Find files |
| `<Space>fw` | Live grep (find word) |
| `<Space>fb` | Find buffers |
| `<Space>e` | Toggle NvimTree file explorer |
| `<C-n>` | Toggle NvimTree |
| `<Space>h` / `<Space>v` | Open horizontal / vertical terminal |
| `<Tab>` / `<S-Tab>` | Next / previous buffer |
| `<Space>x` | Close buffer |

#### Theming with base46

```bash
# In Neovim, browse themes interactively
:Telescope themes

# Or press <Space>th to cycle through themes live
```

NvChad's base46 system compiles themes to bytecode for fast loading. Themes affect the entire UI — statusline, tabufline, Telescope, NvimTree, and all syntax highlighting.

---

### AstroNvim

**AstroNvim** is an aesthetic and feature-rich Neovim configuration that emphasises a curated, community-driven experience.

**Link:** [https://astronvim.com/](https://astronvim.com/)

#### Installation

```bash
# Back up existing config
mv ~/.config/nvim{,.bak}
mv ~/.local/share/nvim{,.bak}

# Clone the AstroNvim template
git clone --depth 1 https://github.com/AstroNvim/template ~/.config/nvim

# Remove template git history
rm -rf ~/.config/nvim/.git

# Launch Neovim
nvim
```

#### Key Features

| Feature | Description |
|---------|-------------|
| AstroUI | Unified UI configuration layer |
| AstroLSP | Centralised LSP configuration |
| AstroCommunity | Community plugin repository with pre-configured extras |
| Heirline | Customisable statusline and tabline |
| neo-tree | File explorer |
| Mason | LSP/linter/formatter management |

#### Customisation

AstroNvim uses a layered configuration approach. Add community plugins:

```lua
-- ~/.config/nvim/lua/community.lua

---@type LazySpec
return {
    "AstroNvim/astrocommunity",

    -- Language packs
    { import = "astrocommunity.pack.lua" },
    { import = "astrocommunity.pack.typescript" },
    { import = "astrocommunity.pack.python" },
    { import = "astrocommunity.pack.rust" },

    -- Colourschemes
    { import = "astrocommunity.colorscheme.catppuccin" },

    -- Motion plugins
    { import = "astrocommunity.motion.leap-nvim" },
}
```

#### Default Keybindings

| Mapping | Action |
|---------|--------|
| `<Space>` | Leader key |
| `<Space>ff` | Find files |
| `<Space>fw` | Find word (grep) |
| `<Space>e` | Toggle file explorer |
| `<Space>lf` | Format document |
| `<Space>lr` | Rename symbol |
| `<Space>la` | Code action |

---

### kickstart.nvim

**kickstart.nvim** is a minimal Neovim starting point created by TJ DeVries, a Neovim core contributor. Unlike the distributions above, it is not meant to be used as-is — it is a **learning-focused template** designed to be forked and modified.

**Link:** [https://github.com/nvim-lua/kickstart.nvim](https://github.com/nvim-lua/kickstart.nvim)

#### Installation

```bash
# Back up existing config
mv ~/.config/nvim{,.bak}

# Fork on GitHub first, then clone your fork
git clone https://github.com/YOUR_USERNAME/kickstart.nvim ~/.config/nvim

# Launch Neovim
nvim
```

#### Philosophy

- **Single file:** The entire configuration lives in one `init.lua` (~500 lines)
- **Heavily commented:** Every line is explained so you understand what it does
- **Meant to be modified:** Fork it, read every line, change what you want
- **Minimal but complete:** Includes LSP, Treesitter, Telescope, autocompletion, and a few essentials

#### Best For

- Developers who want to **understand every line** of their configuration
- Users transitioning from a distribution to a custom setup
- Anyone following the approach described in [03-NEOVIM.md](03-NEOVIM.md)

---

### Comparison Table

| Aspect | LazyVim | NvChad | AstroNvim | kickstart.nvim | From Scratch |
|--------|---------|--------|-----------|----------------|--------------|
| **Philosophy** | Batteries included | Beautiful & fast | Community curated | Learn by reading | Full control |
| **Config complexity** | Low (just overrides) | Low–Medium | Medium | Medium (single file) | High |
| **Customisation** | Extras + overrides | chadrc + plugins | Community packs | Edit the file | Unlimited |
| **Performance** | Excellent (lazy-loading) | Excellent (compiled themes) | Very good | Good | Depends on you |
| **Learning curve** | Low | Low | Medium | Medium–High | High |
| **Plugin count** | ~40 (with extras: more) | ~25 | ~35 | ~15 | Your choice |
| **Update mechanism** | `:Lazy sync` | `:Lazy sync` | `:Lazy sync` | `git pull` | Manual |
| **Best for** | Quick IDE setup | Aesthetic configs | Community plugins | Learning Neovim | Full ownership |

### Choosing a Distribution

```
Want a working IDE today with minimal effort?
  → LazyVim

Care deeply about aesthetics and theme customisation?
  → NvChad

Want community-maintained language packs?
  → AstroNvim

Want to understand every line of your config?
  → kickstart.nvim (then evolve it)

Want total control and enjoy the process?
  → Build from scratch (see 03-NEOVIM.md)
```

> **Tip:** You can always start with a distribution and migrate to a custom config later. Many experienced users began with a distribution, learnt what they liked, and then rebuilt from scratch.

---

## Learning Vim Keybindings

### Vim Modes Primer

Vim's modal editing is what makes it uniquely powerful — and what transfers across all Vim-enabled tools.

| Mode | Key to Enter | Purpose |
|------|-------------|---------|
| **Normal** | `<Esc>` | Navigate, delete, copy, paste, run commands |
| **Insert** | `i`, `a`, `o`, `I`, `A`, `O` | Type text |
| **Visual** | `v`, `V`, `<C-v>` | Select text (character, line, block) |
| **Command-Line** | `:` | Run Ex commands (`:w`, `:q`, `:s`, etc.) |
| **Replace** | `R` | Overwrite text character by character |

The key insight: in Normal mode, keys are **commands**, not characters. This is why `d` means "delete", `w` means "word", and `dw` means "delete word".

### Built-in Learning Tools

Neovim and Vim ship with built-in tutorials:

```bash
# Launch the interactive tutorial (30 minutes)
# From the shell:
vimtutor

# Or from within Neovim:
:Tutor
```

The `:help` system is comprehensive and always available:

```vim
" Get help on any topic
:help motions
:help text-objects
:help registers
:help pattern-overview

" Search help topics
:helpgrep treesitter

" Navigate help with Ctrl-] (follow link) and Ctrl-O (go back)
```

### Interactive Learning Resources

| Resource | Type | Description |
|----------|------|-------------|
| **vim-be-good** | Neovim plugin | ThePrimeagen's plugin for practising motions and editing |
| **Vim Adventures** | Browser game | Learn Vim by playing an adventure game |
| **OpenVim** | Web tutorial | Interactive Vim tutorial in the browser |
| **VimGolf** | Challenges | Solve editing tasks in the fewest keystrokes |

#### vim-be-good

```lua
-- Install via lazy.nvim
{
    "ThePrimeagen/vim-be-good",
    cmd = "VimBeGood",
}
```

```vim
" Launch a practice game
:VimBeGood
```

#### Links

- Vim Adventures: [https://vim-adventures.com/](https://vim-adventures.com/)
- OpenVim: [https://www.openvim.com/](https://www.openvim.com/)
- VimGolf: [https://www.vimgolf.com/](https://www.vimgolf.com/)

---

## Vim Keybindings in Other Tools

One of Vim's greatest strengths is that its keybindings can be used almost everywhere. Once you build muscle memory, you carry it across your entire workflow.

### VS Code — VSCodeVim / vscode-neovim

There are two popular extensions for Vim keybindings in VS Code:

| Extension | Approach | Fidelity | Performance |
|-----------|----------|----------|-------------|
| **VSCodeVim** | Vim emulation in JavaScript | Good (most motions and commands) | Slight input lag on large files |
| **vscode-neovim** | Embeds an actual Neovim instance | Excellent (real Neovim) | Better (native processing) |

#### VSCodeVim

```jsonc
// settings.json
{
    "vim.leader": "<space>",
    "vim.hlsearch": true,
    "vim.useSystemClipboard": true,
    "vim.insertModeKeyBindings": [
        { "before": ["j", "k"], "after": ["<Esc>"] }
    ],
    "vim.normalModeKeyBindings": [
        { "before": ["<leader>", "w"], "commands": ["workbench.action.files.save"] },
        { "before": ["<leader>", "e"], "commands": ["workbench.action.toggleSidebarVisibility"] }
    ]
}
```

#### vscode-neovim

```jsonc
// settings.json
{
    "vscode-neovim.neovimExecutablePaths.linux": "/usr/bin/nvim",
    "vscode-neovim.neovimInitVimPaths.linux": "~/.config/nvim-vscode/init.lua"
}
```

```lua
-- ~/.config/nvim-vscode/init.lua
-- Lightweight config for the embedded Neovim in VS Code
vim.g.mapleader = " "

vim.keymap.set("n", "<leader>ff", function()
    require("vscode").call("workbench.action.quickOpen")
end)

vim.keymap.set("n", "<leader>fg", function()
    require("vscode").call("workbench.action.findInFiles")
end)
```

### JetBrains IDEs — IdeaVim

**IdeaVim** adds Vim keybindings to all JetBrains IDEs (IntelliJ, PyCharm, WebStorm, GoLand, etc.).

#### Setup

Install IdeaVim from the JetBrains plugin marketplace, then create `~/.ideavimrc`:

```vim
" ~/.ideavimrc

" Leader key
let mapleader = " "

" Emulated plugins
set surround          " vim-surround
set commentary        " vim-commentary (gc to comment)
set argtextobj        " argument text objects (cia, dia)
set highlightedyank   " briefly highlight yanked text
set nerdtree          " NERDTree emulation
set which-key         " which-key popup
set notimeout         " let which-key display without timeout

" Settings
set number
set relativenumber
set scrolloff=8
set incsearch
set hlsearch
set ignorecase
set smartcase
set ideajoin          " smart line joining with J

" Key mappings
nmap <leader>ff <Action>(GotoFile)
nmap <leader>fg <Action>(FindInPath)
nmap <leader>fb <Action>(RecentFiles)
nmap <leader>e  <Action>(ActivateProjectToolWindow)
nmap <leader>cr <Action>(RenameElement)
nmap <leader>ca <Action>(ShowIntentionActions)
nmap gd         <Action>(GotoDeclaration)
nmap gr         <Action>(FindUsages)
nmap gi         <Action>(GotoImplementation)
nmap K          <Action>(QuickJavaDoc)
nmap ]d         <Action>(GotoNextError)
nmap [d         <Action>(GotoPreviousError)
nmap <leader>cf <Action>(ReformatCode)

" Window navigation
nmap <C-h> <C-w>h
nmap <C-j> <C-w>j
nmap <C-k> <C-w>k
nmap <C-l> <C-w>l
```

### Browser — Vimium / Tridactyl

Navigate web pages entirely with the keyboard using Vim-style keybindings.

#### Vimium (Chrome / Firefox)

| Key | Action |
|-----|--------|
| `f` | Show link hints — type letters to click a link |
| `F` | Open link in new tab |
| `j` / `k` | Scroll down / up |
| `d` / `u` | Scroll half-page down / up |
| `gg` / `G` | Go to top / bottom of page |
| `H` / `L` | Go back / forward in history |
| `J` / `K` | Switch to left / right tab |
| `x` | Close current tab |
| `X` | Restore closed tab |
| `o` | Open URL or search (omnibar) |
| `T` | Search open tabs |
| `/` | Find on page |
| `yy` | Copy current URL |

#### Tridactyl (Firefox)

Tridactyl provides a more advanced experience with Ex-command support:

```vim
" ~/.config/tridactyl/tridactylrc

" Use dark theme
colourscheme dark

" Smooth scrolling
set smoothscroll true

" Search engines
set searchurls.gh https://github.com/search?q=%s
set searchurls.mdn https://developer.mozilla.org/en-US/search?q=%s
```

### Shell — Vim Mode in Bash/Zsh/Fish

Enable Vim keybindings in your shell for command-line editing:

```bash
# Bash — add to ~/.bashrc
set -o vi

# Zsh — add to ~/.zshrc
bindkey -v

# Fish — add to ~/.config/fish/config.fish
fish_vi_key_bindings
```

Once enabled, you edit commands in Normal and Insert modes:

| Key (Normal mode) | Action |
|-------------------|--------|
| `Esc` | Enter Normal mode |
| `i` / `a` | Enter Insert mode (before / after cursor) |
| `0` / `$` | Jump to start / end of line |
| `w` / `b` | Jump forward / backward by word |
| `dw` / `dd` | Delete word / entire line |
| `cc` | Change entire line |
| `v` | Open current command in `$EDITOR` |

### tmux — Vi Copy Mode

See [02-TMUX.md](02-TMUX.md) for full tmux configuration. Enable vi-style copy mode:

```bash
# ~/.tmux.conf
setw -g mode-keys vi

# Vi-style copy mode bindings
bind-key -T copy-mode-vi v send-keys -X begin-selection
bind-key -T copy-mode-vi y send-keys -X copy-selection-and-cancel
bind-key -T copy-mode-vi C-v send-keys -X rectangle-toggle
```

With `mode-keys vi`, you navigate tmux copy mode with familiar Vim motions (`hjkl`, `/` to search, `v` to select, `y` to yank).

### Other Tools with Vim-like Keybindings

Many terminal tools support Vim keybindings by default or via configuration:

| Tool | Category | Vim Keybindings |
|------|----------|-----------------|
| `less` / `man` | Pager | Built-in (`j`/`k`, `/` search, `q` quit) |
| `lazygit` | Git TUI | Built-in navigation with `hjkl` |
| `lazydocker` | Docker TUI | Built-in navigation with `hjkl` |
| `ranger` | File manager | Vi-style navigation by default |
| `lf` | File manager | Vi-style navigation by default |
| `yazi` | File manager | Vi-style navigation, fast and modern |
| `zathura` | PDF viewer | Vi-style navigation (`j`/`k`, `gg`/`G`) |
| readline | Input library | `set editing-mode vi` in `~/.inputrc` |

Enable vi mode globally for all readline-based tools:

```bash
# ~/.inputrc
set editing-mode vi
set show-mode-in-prompt on
set vi-ins-mode-string \1\e[6 q\2
set vi-cmd-mode-string \1\e[2 q\2
```

---

## Essential Vim Motions Cheat Sheet

These motions work across Neovim, VS Code (with Vim extension), JetBrains (IdeaVim), and most Vim-enabled tools.

### Movement

| Key | Motion |
|-----|--------|
| `h` `j` `k` `l` | Left, down, up, right |
| `w` / `W` | Next word / WORD start |
| `b` / `B` | Previous word / WORD start |
| `e` / `E` | Next word / WORD end |
| `0` | Start of line |
| `^` | First non-blank character |
| `$` | End of line |
| `gg` | First line of file |
| `G` | Last line of file |
| `{line}G` | Go to line number |
| `f{char}` | Jump to next `{char}` on line |
| `t{char}` | Jump to just before next `{char}` |
| `F{char}` / `T{char}` | Same as `f`/`t` but backwards |
| `;` / `,` | Repeat last `f`/`t` forward / backward |
| `%` | Jump to matching bracket |
| `{` / `}` | Previous / next paragraph |
| `Ctrl-d` / `Ctrl-u` | Half-page down / up |
| `Ctrl-f` / `Ctrl-b` | Full page down / up |
| `zz` | Centre current line on screen |

### Operators

Operators combine with motions: `{operator}{motion}`.

| Operator | Action |
|----------|--------|
| `d` | Delete |
| `c` | Change (delete + enter Insert mode) |
| `y` | Yank (copy) |
| `>` | Indent right |
| `<` | Indent left |
| `=` | Auto-indent |
| `gU` | Uppercase |
| `gu` | Lowercase |

### Text Objects

Used with operators: `{operator}{a/i}{object}`. `i` = inner (exclude delimiters), `a` = around (include delimiters).

| Text Object | What It Selects |
|-------------|-----------------|
| `iw` / `aw` | Inner word / a word (includes trailing space) |
| `iW` / `aW` | Inner WORD / a WORD |
| `is` / `as` | Inner sentence / a sentence |
| `ip` / `ap` | Inner paragraph / a paragraph |
| `i"` / `a"` | Inside double quotes / around double quotes |
| `i'` / `a'` | Inside single quotes / around single quotes |
| `` i` `` / `` a` `` | Inside backticks / around backticks |
| `i(` / `a(` | Inside parentheses / around parentheses |
| `i[` / `a[` | Inside square brackets / around square brackets |
| `i{` / `a{` | Inside curly braces / around curly braces |
| `it` / `at` | Inside HTML/XML tag / around tag |

### Common Combinations

| Combo | What It Does |
|-------|-------------|
| `ciw` | Change inner word (delete word, enter Insert mode) |
| `di"` | Delete inside double quotes |
| `yap` | Yank a paragraph |
| `ci(` | Change inside parentheses |
| `da{` | Delete around curly braces (including braces) |
| `vit` | Visually select inside HTML tag |
| `>ip` | Indent inner paragraph |
| `gUiw` | Uppercase inner word |
| `dgg` | Delete from cursor to start of file |
| `yG` | Yank from cursor to end of file |
| `ct;` | Change up to next semicolon |
| `df,` | Delete through next comma |
| `=ap` | Auto-indent a paragraph |
| `dd` | Delete entire line |
| `yy` | Yank entire line |
| `cc` | Change entire line |
| `>>` / `<<` | Indent / unindent current line |

---

## Building Muscle Memory

Transitioning to Vim keybindings takes deliberate practice. Here is a practical approach:

### Phase 1 — Basic Navigation (Week 1–2)

- Use `hjkl` instead of arrow keys (disable arrow keys if brave)
- Learn `w`, `b`, `e` for word movement
- Use `i` and `a` to enter Insert mode, `Esc` to return to Normal
- Save with `:w`, quit with `:q`

### Phase 2 — Operators and Motions (Week 3–4)

- Combine `d`, `c`, `y` with motions (`dw`, `cw`, `y$`)
- Use `dd`, `yy`, `cc` for whole-line operations
- Learn `/` and `?` for searching
- Start using `f`/`t` for in-line jumps

### Phase 3 — Text Objects (Week 5–6)

- Use `ciw`, `di"`, `yap` and similar combinations
- Learn visual mode (`v`, `V`, `Ctrl-v`)
- Practise with vim-be-good or VimGolf challenges

### Phase 4 — Expand Everywhere (Ongoing)

- Enable Vim keybindings in your shell (`set -o vi`)
- Install Vimium in your browser
- Configure IdeaVim or VSCodeVim in your IDE
- Use vi copy mode in tmux

### Tips

```
● Don't try to learn everything at once — add one new motion per day
● Use which-key (in Neovim distributions) to discover mappings
● Keep a cheat sheet visible until the motions become automatic
● Practise on real editing tasks, not just tutorials
● When you reach for the mouse, pause and find the Vim way
● Repetition matters more than memorisation
```

---

## Next Steps

- **Build a custom config** → [03-NEOVIM.md](03-NEOVIM.md) — Set up Neovim from scratch with Lua and lazy.nvim
- **Set up tmux** → [02-TMUX.md](02-TMUX.md) — Terminal multiplexer with vi copy mode
- **Productivity tools** → [07-PRODUCTIVITY-TOOLS.md](07-PRODUCTIVITY-TOOLS.md) — fzf, ripgrep, and other CLI tools
- **Best practices** → [08-BEST-PRACTICES.md](08-BEST-PRACTICES.md) — Version-control your dotfiles and configurations

---

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2025 | Initial Vim distributions and keybindings documentation |
