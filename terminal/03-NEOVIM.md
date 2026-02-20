# Neovim

A comprehensive guide to Neovim — covering installation, Neovim vs Vim, Lua-based configuration, plugin management with lazy.nvim, essential plugins (Telescope, Treesitter, LSP, nvim-cmp, lualine), key mappings, colourschemes, Neovim as an IDE, project navigation, and terminal integration.

---

## Table of Contents

1. [Overview](#overview)
2. [Installation](#installation)
3. [Neovim vs Vim](#neovim-vs-vim)
4. [Lua Configuration](#lua-configuration)
5. [Plugin Management with lazy.nvim](#plugin-management-with-lazynvim)
6. [Essential Plugins](#essential-plugins)
7. [Key Mappings](#key-mappings)
8. [Colourschemes](#colourschemes)
9. [Neovim as IDE — LSP Setup](#neovim-as-ide--lsp-setup)
10. [Autocompletion with nvim-cmp](#autocompletion-with-nvim-cmp)
11. [Debugging with DAP](#debugging-with-dap)
12. [Project Navigation](#project-navigation)
13. [Terminal Integration](#terminal-integration)
14. [Neovim with tmux](#neovim-with-tmux)
15. [Next Steps](#next-steps)
16. [Version History](#version-history)

---

## Overview

**Neovim** is a modernised fork of Vim that provides first-class Lua scripting, built-in LSP support, Treesitter-based syntax highlighting, and an extensible architecture. It can be configured into a full IDE experience while remaining a fast, terminal-native editor.

### Target Audience

- **Developers** who want a fast, keyboard-driven editor with IDE features
- **Vim users** looking to modernise their configuration with Lua and built-in LSP
- **Terminal-centric engineers** who want their editor integrated with tmux and CLI tools

### Why Neovim

| Feature | Vim | Neovim |
|---------|-----|--------|
| Configuration language | Vimscript | Lua (+ Vimscript) |
| LSP support | Via plugin (coc.nvim, vim-lsp) | Built-in (`vim.lsp`) |
| Treesitter | Limited | Built-in (nvim-treesitter) |
| Async architecture | Limited | First-class async via libuv |
| Terminal emulator | Basic (`:terminal`) | Full terminal with API |
| Plugin ecosystem | Mature (Vimscript-based) | Modern (Lua-based, growing fast) |
| Remote plugins | Via channels | RPC API, headless mode |
| Default config | Conservative | Sensible modern defaults |

---

## Installation

```bash
# macOS
brew install neovim

# Debian/Ubuntu (latest stable)
sudo add-apt-repository ppa:neovim-ppa/stable
sudo apt update && sudo apt install neovim

# Fedora
sudo dnf install neovim

# Arch Linux
sudo pacman -S neovim

# Windows (via scoop)
scoop install neovim

# Windows (via winget)
winget install Neovim.Neovim

# AppImage (Linux, no root needed)
curl -LO https://github.com/neovim/neovim/releases/latest/download/nvim.appimage
chmod u+x nvim.appimage
mv nvim.appimage ~/.local/bin/nvim

# Verify installation
nvim --version
```

### Prerequisites

Before configuring Neovim, install these dependencies:

```bash
# Required for many plugins
# Node.js (for some LSP servers and plugins)
# A C compiler (for Treesitter parser compilation)
# ripgrep (for Telescope live grep)
# fd (for Telescope file finding)
# A Nerd Font (for icons)

# macOS
brew install node ripgrep fd

# Debian/Ubuntu
sudo apt install nodejs npm ripgrep fd-find gcc

# Fedora
sudo dnf install nodejs npm ripgrep fd-find gcc
```

---

## Neovim vs Vim

### Key Differences

```
Vim:
  ● Configuration in ~/.vimrc (Vimscript)
  ● Plugin managers: vim-plug, Vundle, pathogen
  ● LSP via coc.nvim or ALE
  ● Mature, battle-tested, available everywhere
  ● Single-threaded architecture

Neovim:
  ● Configuration in ~/.config/nvim/init.lua (Lua)
  ● Plugin managers: lazy.nvim, packer.nvim
  ● LSP built into the core (vim.lsp)
  ● Treesitter built into the core
  ● Async via libuv (non-blocking)
  ● Modern ecosystem with fast-growing plugins
```

### Migration Path

If you already have a Vim configuration:

```bash
# Neovim can use your existing Vim config as a starting point
# Create a minimal init.lua that sources your vimrc:
mkdir -p ~/.config/nvim
cat > ~/.config/nvim/init.lua << 'EOF'
vim.cmd('source ~/.vimrc')
EOF

# Better approach: gradually migrate to Lua
# Start with a fresh init.lua and port settings incrementally
```

---

## Lua Configuration

### Configuration Directory Structure

```
~/.config/nvim/
├── init.lua                 # Entry point — loads everything else
├── lazy-lock.json           # Plugin version lock file (auto-generated)
└── lua/
    ├── config/
    │   ├── options.lua      # Vim options (set number, etc.)
    │   ├── keymaps.lua      # Key mappings
    │   ├── autocmds.lua     # Autocommands
    │   └── lazy.lua         # lazy.nvim bootstrap and setup
    └── plugins/
        ├── telescope.lua    # Telescope configuration
        ├── treesitter.lua   # Treesitter configuration
        ├── lsp.lua          # LSP configuration
        ├── cmp.lua          # Autocompletion configuration
        ├── lualine.lua      # Status line configuration
        ├── colorscheme.lua  # Colourscheme
        └── ...              # One file per plugin or plugin group
```

### init.lua — Entry Point

```lua
-- ~/.config/nvim/init.lua

-- Load core configuration
require("config.options")
require("config.keymaps")
require("config.autocmds")

-- Bootstrap and load plugins
require("config.lazy")
```

### Options

```lua
-- ~/.config/nvim/lua/config/options.lua

local opt = vim.opt

-- Line numbers
opt.number = true           -- Show line numbers
opt.relativenumber = true   -- Relative line numbers

-- Tabs and indentation
opt.tabstop = 4             -- Tab width
opt.shiftwidth = 4          -- Indent width
opt.expandtab = true        -- Use spaces instead of tabs
opt.autoindent = true       -- Copy indent from current line
opt.smartindent = true      -- Smart autoindenting

-- Search
opt.ignorecase = true       -- Case-insensitive search
opt.smartcase = true        -- Case-sensitive if uppercase present
opt.hlsearch = false        -- Don't highlight all matches
opt.incsearch = true        -- Incremental search

-- Appearance
opt.termguicolors = true    -- True colour support
opt.signcolumn = "yes"      -- Always show sign column
opt.cursorline = true       -- Highlight current line
opt.scrolloff = 8           -- Lines above/below cursor
opt.sidescrolloff = 8       -- Columns left/right of cursor
opt.wrap = false            -- Don't wrap lines
opt.colorcolumn = "100"     -- Column guide at 100 chars

-- Splits
opt.splitright = true       -- Vertical splits open to the right
opt.splitbelow = true       -- Horizontal splits open below

-- Clipboard
opt.clipboard = "unnamedplus"  -- Use system clipboard

-- Undo persistence
opt.undofile = true         -- Save undo history to file
opt.undodir = vim.fn.stdpath("state") .. "/undo"

-- Performance
opt.updatetime = 250        -- Faster CursorHold events
opt.timeoutlen = 300        -- Key sequence timeout

-- Completion
opt.completeopt = "menuone,noselect"  -- Better completion experience

-- File handling
opt.swapfile = false        -- Don't create swap files
opt.backup = false          -- Don't create backup files

-- Mouse
opt.mouse = "a"             -- Enable mouse in all modes

-- Whitespace characters
opt.list = true
opt.listchars = { tab = "» ", trail = "·", nbsp = "␣" }
```

### Keymaps

```lua
-- ~/.config/nvim/lua/config/keymaps.lua

local keymap = vim.keymap.set

-- Set leader key to space
vim.g.mapleader = " "
vim.g.maplocalleader = " "

-- Better escape
keymap("i", "jk", "<Esc>", { desc = "Exit insert mode" })

-- Window navigation
keymap("n", "<C-h>", "<C-w>h", { desc = "Go to left window" })
keymap("n", "<C-j>", "<C-w>j", { desc = "Go to lower window" })
keymap("n", "<C-k>", "<C-w>k", { desc = "Go to upper window" })
keymap("n", "<C-l>", "<C-w>l", { desc = "Go to right window" })

-- Window resize
keymap("n", "<C-Up>", ":resize -2<CR>", { desc = "Decrease window height" })
keymap("n", "<C-Down>", ":resize +2<CR>", { desc = "Increase window height" })
keymap("n", "<C-Left>", ":vertical resize -2<CR>", { desc = "Decrease window width" })
keymap("n", "<C-Right>", ":vertical resize +2<CR>", { desc = "Increase window width" })

-- Buffer navigation
keymap("n", "<S-h>", ":bprevious<CR>", { desc = "Previous buffer" })
keymap("n", "<S-l>", ":bnext<CR>", { desc = "Next buffer" })
keymap("n", "<leader>bd", ":bdelete<CR>", { desc = "Delete buffer" })

-- Move selected lines up/down in visual mode
keymap("v", "J", ":m '>+1<CR>gv=gv", { desc = "Move selection down" })
keymap("v", "K", ":m '<-2<CR>gv=gv", { desc = "Move selection up" })

-- Keep cursor centred when scrolling
keymap("n", "<C-d>", "<C-d>zz", { desc = "Scroll down (centred)" })
keymap("n", "<C-u>", "<C-u>zz", { desc = "Scroll up (centred)" })

-- Keep cursor centred when searching
keymap("n", "n", "nzzzv", { desc = "Next search result (centred)" })
keymap("n", "N", "Nzzzv", { desc = "Previous search result (centred)" })

-- Better paste (don't replace register with deleted text)
keymap("x", "<leader>p", '"_dP', { desc = "Paste without yanking" })

-- Copy to system clipboard
keymap({ "n", "v" }, "<leader>y", '"+y', { desc = "Copy to system clipboard" })

-- Delete without yanking
keymap({ "n", "v" }, "<leader>d", '"_d', { desc = "Delete without yanking" })

-- Quick save
keymap("n", "<leader>w", ":w<CR>", { desc = "Save file" })

-- Quick quit
keymap("n", "<leader>q", ":q<CR>", { desc = "Quit" })

-- Clear search highlights
keymap("n", "<Esc>", ":nohlsearch<CR>", { desc = "Clear search highlights" })

-- File explorer
keymap("n", "<leader>e", ":Explore<CR>", { desc = "Open file explorer" })

-- Diagnostic navigation
keymap("n", "[d", vim.diagnostic.goto_prev, { desc = "Previous diagnostic" })
keymap("n", "]d", vim.diagnostic.goto_next, { desc = "Next diagnostic" })
keymap("n", "<leader>cd", vim.diagnostic.open_float, { desc = "Line diagnostics" })
```

### Autocommands

```lua
-- ~/.config/nvim/lua/config/autocmds.lua

local augroup = vim.api.nvim_create_augroup
local autocmd = vim.api.nvim_create_autocmd

-- Highlight on yank
autocmd("TextYankPost", {
    group = augroup("HighlightYank", { clear = true }),
    callback = function()
        vim.highlight.on_yank({ timeout = 200 })
    end,
})

-- Remove trailing whitespace on save
autocmd("BufWritePre", {
    group = augroup("TrimWhitespace", { clear = true }),
    pattern = "*",
    command = "%s/\\s\\+$//e",
})

-- Return to last cursor position when opening a file
autocmd("BufReadPost", {
    group = augroup("RestoreCursor", { clear = true }),
    callback = function()
        local mark = vim.api.nvim_buf_get_mark(0, '"')
        local line_count = vim.api.nvim_buf_line_count(0)
        if mark[1] > 0 and mark[1] <= line_count then
            pcall(vim.api.nvim_win_set_cursor, 0, mark)
        end
    end,
})

-- Auto-resize splits when window is resized
autocmd("VimResized", {
    group = augroup("AutoResize", { clear = true }),
    command = "tabdo wincmd =",
})
```

---

## Plugin Management with lazy.nvim

### Bootstrap lazy.nvim

```lua
-- ~/.config/nvim/lua/config/lazy.lua

-- Bootstrap lazy.nvim
local lazypath = vim.fn.stdpath("data") .. "/lazy/lazy.nvim"
if not vim.loop.fs_stat(lazypath) then
    vim.fn.system({
        "git", "clone", "--filter=blob:none",
        "https://github.com/folke/lazy.nvim.git",
        "--branch=stable",
        lazypath,
    })
end
vim.opt.rtp:prepend(lazypath)

-- Load plugins from lua/plugins/ directory
require("lazy").setup("plugins", {
    defaults = { lazy = true },
    install = { colorscheme = { "catppuccin", "habamax" } },
    checker = { enabled = true, notify = false },
    change_detection = { notify = false },
    performance = {
        rtp = {
            disabled_plugins = {
                "gzip", "matchit", "matchparen",
                "netrwPlugin", "tarPlugin", "tohtml",
                "tutor", "zipPlugin",
            },
        },
    },
})
```

Each file in `lua/plugins/` returns a plugin spec table. lazy.nvim auto-discovers them.

---

## Essential Plugins

### Telescope — Fuzzy Finder

```lua
-- ~/.config/nvim/lua/plugins/telescope.lua

return {
    "nvim-telescope/telescope.nvim",
    branch = "0.1.x",
    dependencies = {
        "nvim-lua/plenary.nvim",
        { "nvim-telescope/telescope-fzf-native.nvim", build = "make" },
        "nvim-telescope/telescope-ui-select.nvim",
        "nvim-tree/nvim-web-devicons",
    },
    event = "VimEnter",
    config = function()
        local telescope = require("telescope")
        local actions = require("telescope.actions")

        telescope.setup({
            defaults = {
                path_display = { "truncate" },
                sorting_strategy = "ascending",
                layout_config = {
                    horizontal = { prompt_position = "top", preview_width = 0.55 },
                },
                mappings = {
                    i = {
                        ["<C-j>"] = actions.move_selection_next,
                        ["<C-k>"] = actions.move_selection_previous,
                        ["<C-q>"] = actions.send_selected_to_qflist + actions.open_qflist,
                    },
                },
            },
        })

        telescope.load_extension("fzf")
        telescope.load_extension("ui-select")

        -- Key mappings
        local builtin = require("telescope.builtin")
        vim.keymap.set("n", "<leader>ff", builtin.find_files, { desc = "Find files" })
        vim.keymap.set("n", "<leader>fg", builtin.live_grep, { desc = "Live grep" })
        vim.keymap.set("n", "<leader>fb", builtin.buffers, { desc = "Find buffers" })
        vim.keymap.set("n", "<leader>fh", builtin.help_tags, { desc = "Help tags" })
        vim.keymap.set("n", "<leader>fr", builtin.oldfiles, { desc = "Recent files" })
        vim.keymap.set("n", "<leader>fc", builtin.git_commits, { desc = "Git commits" })
        vim.keymap.set("n", "<leader>fs", builtin.git_status, { desc = "Git status" })
        vim.keymap.set("n", "<leader>fd", builtin.diagnostics, { desc = "Diagnostics" })
        vim.keymap.set("n", "<leader>fw", builtin.grep_string, { desc = "Grep word under cursor" })
        vim.keymap.set("n", "<leader>/", builtin.current_buffer_fuzzy_find, { desc = "Fuzzy find in buffer" })
    end,
}
```

### Treesitter — Syntax Highlighting

```lua
-- ~/.config/nvim/lua/plugins/treesitter.lua

return {
    "nvim-treesitter/nvim-treesitter",
    build = ":TSUpdate",
    event = { "BufReadPre", "BufNewFile" },
    dependencies = {
        "nvim-treesitter/nvim-treesitter-textobjects",
    },
    config = function()
        require("nvim-treesitter.configs").setup({
            ensure_installed = {
                "bash", "c", "css", "dockerfile", "go", "html",
                "javascript", "json", "lua", "markdown", "markdown_inline",
                "python", "rust", "toml", "tsx", "typescript", "vim",
                "vimdoc", "yaml",
            },
            auto_install = true,
            highlight = { enable = true },
            indent = { enable = true },
            incremental_selection = {
                enable = true,
                keymaps = {
                    init_selection = "<C-space>",
                    node_incremental = "<C-space>",
                    scope_incremental = false,
                    node_decremental = "<bs>",
                },
            },
            textobjects = {
                select = {
                    enable = true,
                    lookahead = true,
                    keymaps = {
                        ["af"] = "@function.outer",
                        ["if"] = "@function.inner",
                        ["ac"] = "@class.outer",
                        ["ic"] = "@class.inner",
                        ["aa"] = "@parameter.outer",
                        ["ia"] = "@parameter.inner",
                    },
                },
                move = {
                    enable = true,
                    set_jumps = true,
                    goto_next_start = {
                        ["]f"] = "@function.outer",
                        ["]c"] = "@class.outer",
                    },
                    goto_previous_start = {
                        ["[f"] = "@function.outer",
                        ["[c"] = "@class.outer",
                    },
                },
            },
        })
    end,
}
```

### Lualine — Status Line

```lua
-- ~/.config/nvim/lua/plugins/lualine.lua

return {
    "nvim-lualine/lualine.nvim",
    dependencies = { "nvim-tree/nvim-web-devicons" },
    event = "VeryLazy",
    config = function()
        require("lualine").setup({
            options = {
                theme = "catppuccin",
                component_separators = { left = "", right = "" },
                section_separators = { left = "", right = "" },
                globalstatus = true,
            },
            sections = {
                lualine_a = { "mode" },
                lualine_b = { "branch", "diff", "diagnostics" },
                lualine_c = { { "filename", path = 1 } },
                lualine_x = { "encoding", "fileformat", "filetype" },
                lualine_y = { "progress" },
                lualine_z = { "location" },
            },
        })
    end,
}
```

---

## Key Mappings

### Leader Key Namespace

Using `<Space>` as the leader key, here is a recommended namespace organisation:

```
<leader>f  — Find (Telescope)
<leader>g  — Git
<leader>l  — LSP
<leader>b  — Buffer
<leader>w  — Save
<leader>q  — Quit
<leader>c  — Code actions / diagnostics
<leader>d  — Debug (DAP)
<leader>t  — Terminal / Test
<leader>x  — Trouble / Diagnostics list
```

### Which-Key Integration

```lua
-- ~/.config/nvim/lua/plugins/which-key.lua

return {
    "folke/which-key.nvim",
    event = "VeryLazy",
    config = function()
        local wk = require("which-key")
        wk.setup({
            plugins = { spelling = { enabled = true } },
        })
        wk.add({
            { "<leader>f", group = "Find" },
            { "<leader>g", group = "Git" },
            { "<leader>l", group = "LSP" },
            { "<leader>b", group = "Buffer" },
            { "<leader>c", group = "Code" },
            { "<leader>d", group = "Debug" },
            { "<leader>t", group = "Terminal/Test" },
        })
    end,
}
```

---

## Colourschemes

### Catppuccin (Popular Modern Theme)

```lua
-- ~/.config/nvim/lua/plugins/colorscheme.lua

return {
    "catppuccin/nvim",
    name = "catppuccin",
    priority = 1000,
    lazy = false,
    config = function()
        require("catppuccin").setup({
            flavour = "mocha",  -- latte, frappe, macchiato, mocha
            integrations = {
                cmp = true,
                gitsigns = true,
                telescope = { enabled = true },
                treesitter = true,
                which_key = true,
                native_lsp = {
                    enabled = true,
                    underlines = {
                        errors = { "undercurl" },
                        hints = { "undercurl" },
                        warnings = { "undercurl" },
                        information = { "undercurl" },
                    },
                },
            },
        })
        vim.cmd.colorscheme("catppuccin")
    end,
}
```

### Other Popular Colourschemes

| Theme | Style | Plugin |
|-------|-------|--------|
| **Catppuccin** | Pastel, warm | `catppuccin/nvim` |
| **Tokyo Night** | Dark blue/purple | `folke/tokyonight.nvim` |
| **Gruvbox Material** | Retro warm | `sainnhe/gruvbox-material` |
| **Kanagawa** | Dark, inspired by Japanese art | `rebelot/kanagawa.nvim` |
| **Nord** | Arctic, cool blues | `shaunsingh/nord.nvim` |
| **One Dark** | Atom-inspired | `navarasu/onedark.nvim` |
| **Rose Pine** | Muted, elegant | `rose-pine/neovim` |

---

## Neovim as IDE — LSP Setup

### LSP Configuration

```lua
-- ~/.config/nvim/lua/plugins/lsp.lua

return {
    "neovim/nvim-lspconfig",
    dependencies = {
        "mason.nvim",
        "williamboman/mason-lspconfig.nvim",
        "hrsh7th/cmp-nvim-lsp",
    },
    event = { "BufReadPre", "BufNewFile" },
    config = function()
        local lspconfig = require("lspconfig")
        local mason_lspconfig = require("mason-lspconfig")
        local cmp_lsp = require("cmp_nvim_lsp")

        local capabilities = cmp_lsp.default_capabilities()

        -- Keymaps applied when LSP attaches to a buffer
        vim.api.nvim_create_autocmd("LspAttach", {
            group = vim.api.nvim_create_augroup("LspKeymaps", { clear = true }),
            callback = function(event)
                local map = function(keys, func, desc)
                    vim.keymap.set("n", keys, func, { buffer = event.buf, desc = "LSP: " .. desc })
                end

                map("gd", require("telescope.builtin").lsp_definitions, "Go to definition")
                map("gr", require("telescope.builtin").lsp_references, "Go to references")
                map("gI", require("telescope.builtin").lsp_implementations, "Go to implementation")
                map("gy", require("telescope.builtin").lsp_type_definitions, "Go to type definition")
                map("K", vim.lsp.buf.hover, "Hover documentation")
                map("<leader>ca", vim.lsp.buf.code_action, "Code action")
                map("<leader>cr", vim.lsp.buf.rename, "Rename symbol")
                map("<leader>cs", require("telescope.builtin").lsp_document_symbols, "Document symbols")
                map("<leader>cS", require("telescope.builtin").lsp_workspace_symbols, "Workspace symbols")
            end,
        })

        -- Mason (LSP installer)
        mason_lspconfig.setup({
            ensure_installed = {
                "lua_ls",          -- Lua
                "ts_ls",           -- TypeScript/JavaScript
                "pyright",         -- Python
                "gopls",           -- Go
                "rust_analyzer",   -- Rust
                "bashls",          -- Bash
                "dockerls",        -- Dockerfile
                "yamlls",          -- YAML
                "jsonls",          -- JSON
            },
        })

        -- Configure each server
        mason_lspconfig.setup_handlers({
            -- Default handler
            function(server_name)
                lspconfig[server_name].setup({
                    capabilities = capabilities,
                })
            end,

            -- Lua-specific configuration
            ["lua_ls"] = function()
                lspconfig.lua_ls.setup({
                    capabilities = capabilities,
                    settings = {
                        Lua = {
                            runtime = { version = "LuaJIT" },
                            workspace = {
                                checkThirdParty = false,
                                library = { vim.env.VIMRUNTIME },
                            },
                            completion = { callSnippet = "Replace" },
                        },
                    },
                })
            end,
        })
    end,
}
```

### Mason — LSP/Linter/Formatter Installer

```lua
-- ~/.config/nvim/lua/plugins/mason.lua

return {
    "williamboman/mason.nvim",
    cmd = "Mason",
    build = ":MasonUpdate",
    config = function()
        require("mason").setup({
            ui = {
                icons = {
                    package_installed = "✓",
                    package_pending = "➜",
                    package_uninstalled = "✗",
                },
            },
        })
    end,
}
```

---

## Autocompletion with nvim-cmp

```lua
-- ~/.config/nvim/lua/plugins/cmp.lua

return {
    "hrsh7th/nvim-cmp",
    event = "InsertEnter",
    dependencies = {
        "hrsh7th/cmp-nvim-lsp",      -- LSP completions
        "hrsh7th/cmp-buffer",         -- Buffer completions
        "hrsh7th/cmp-path",           -- Path completions
        "L3MON4D3/LuaSnip",          -- Snippet engine
        "saadparwaiz1/cmp_luasnip",   -- Snippet completions
        "rafamadriz/friendly-snippets", -- Snippet collection
    },
    config = function()
        local cmp = require("cmp")
        local luasnip = require("luasnip")

        require("luasnip.loaders.from_vscode").lazy_load()

        cmp.setup({
            snippet = {
                expand = function(args)
                    luasnip.lsp_expand(args.body)
                end,
            },
            completion = { completeopt = "menu,menuone,noinsert" },
            mapping = cmp.mapping.preset.insert({
                ["<C-n>"] = cmp.mapping.select_next_item(),
                ["<C-p>"] = cmp.mapping.select_prev_item(),
                ["<C-b>"] = cmp.mapping.scroll_docs(-4),
                ["<C-f>"] = cmp.mapping.scroll_docs(4),
                ["<C-y>"] = cmp.mapping.confirm({ select = true }),
                ["<C-Space>"] = cmp.mapping.complete(),
                ["<C-e>"] = cmp.mapping.abort(),
                ["<Tab>"] = cmp.mapping(function(fallback)
                    if luasnip.expand_or_locally_jumpable() then
                        luasnip.expand_or_jump()
                    else
                        fallback()
                    end
                end, { "i", "s" }),
                ["<S-Tab>"] = cmp.mapping(function(fallback)
                    if luasnip.locally_jumpable(-1) then
                        luasnip.jump(-1)
                    else
                        fallback()
                    end
                end, { "i", "s" }),
            }),
            sources = cmp.config.sources({
                { name = "nvim_lsp" },
                { name = "luasnip" },
                { name = "path" },
            }, {
                { name = "buffer" },
            }),
        })
    end,
}
```

---

## Debugging with DAP

```lua
-- ~/.config/nvim/lua/plugins/dap.lua

return {
    "mfussenegger/nvim-dap",
    dependencies = {
        "rcarriga/nvim-dap-ui",
        "nvim-neotest/nvim-nio",
        "theHamsta/nvim-dap-virtual-text",
        "williamboman/mason.nvim",
        "jay-babu/mason-nvim-dap.nvim",
    },
    keys = {
        { "<leader>db", function() require("dap").toggle_breakpoint() end, desc = "Toggle breakpoint" },
        { "<leader>dc", function() require("dap").continue() end, desc = "Continue" },
        { "<leader>di", function() require("dap").step_into() end, desc = "Step into" },
        { "<leader>do", function() require("dap").step_over() end, desc = "Step over" },
        { "<leader>dO", function() require("dap").step_out() end, desc = "Step out" },
        { "<leader>dr", function() require("dap").repl.open() end, desc = "Open REPL" },
        { "<leader>du", function() require("dapui").toggle() end, desc = "Toggle DAP UI" },
    },
    config = function()
        local dap = require("dap")
        local dapui = require("dapui")

        require("mason-nvim-dap").setup({
            ensure_installed = { "python", "codelldb", "js" },
            automatic_installation = true,
        })

        dapui.setup()
        require("nvim-dap-virtual-text").setup()

        -- Auto open/close DAP UI
        dap.listeners.after.event_initialized["dapui_config"] = dapui.open
        dap.listeners.before.event_terminated["dapui_config"] = dapui.close
        dap.listeners.before.event_exited["dapui_config"] = dapui.close
    end,
}
```

---

## Project Navigation

### File Explorer with Oil.nvim

```lua
-- ~/.config/nvim/lua/plugins/oil.lua

return {
    "stevearc/oil.nvim",
    dependencies = { "nvim-tree/nvim-web-devicons" },
    config = function()
        require("oil").setup({
            view_options = { show_hidden = true },
            keymaps = {
                ["<C-h>"] = false,  -- Don't conflict with window nav
                ["<C-l>"] = false,
            },
        })
        vim.keymap.set("n", "-", "<CMD>Oil<CR>", { desc = "Open parent directory" })
    end,
}
```

### Harpoon — Quick File Switching

```lua
-- ~/.config/nvim/lua/plugins/harpoon.lua

return {
    "ThePrimeagen/harpoon",
    branch = "harpoon2",
    dependencies = { "nvim-lua/plenary.nvim" },
    config = function()
        local harpoon = require("harpoon")
        harpoon:setup()

        vim.keymap.set("n", "<leader>a", function() harpoon:list():add() end, { desc = "Harpoon add" })
        vim.keymap.set("n", "<C-e>", function() harpoon.ui:toggle_quick_menu(harpoon:list()) end, { desc = "Harpoon menu" })
        vim.keymap.set("n", "<leader>1", function() harpoon:list():select(1) end, { desc = "Harpoon 1" })
        vim.keymap.set("n", "<leader>2", function() harpoon:list():select(2) end, { desc = "Harpoon 2" })
        vim.keymap.set("n", "<leader>3", function() harpoon:list():select(3) end, { desc = "Harpoon 3" })
        vim.keymap.set("n", "<leader>4", function() harpoon:list():select(4) end, { desc = "Harpoon 4" })
    end,
}
```

---

## Terminal Integration

### Built-in Terminal

```lua
-- Open terminal in current window
vim.keymap.set("n", "<leader>tt", ":terminal<CR>", { desc = "Open terminal" })

-- Open terminal in horizontal split
vim.keymap.set("n", "<leader>th", ":split | terminal<CR>", { desc = "Terminal (horizontal)" })

-- Open terminal in vertical split
vim.keymap.set("n", "<leader>tv", ":vsplit | terminal<CR>", { desc = "Terminal (vertical)" })

-- Exit terminal mode with Escape
vim.keymap.set("t", "<Esc><Esc>", "<C-\\><C-n>", { desc = "Exit terminal mode" })

-- Navigate from terminal to other windows
vim.keymap.set("t", "<C-h>", "<C-\\><C-n><C-w>h", { desc = "Go to left window" })
vim.keymap.set("t", "<C-j>", "<C-\\><C-n><C-w>j", { desc = "Go to lower window" })
vim.keymap.set("t", "<C-k>", "<C-\\><C-n><C-w>k", { desc = "Go to upper window" })
vim.keymap.set("t", "<C-l>", "<C-\\><C-n><C-w>l", { desc = "Go to right window" })
```

### Toggleterm — Persistent Terminal

```lua
-- ~/.config/nvim/lua/plugins/toggleterm.lua

return {
    "akinsho/toggleterm.nvim",
    version = "*",
    keys = {
        { "<C-\\>", desc = "Toggle terminal" },
        { "<leader>tg", desc = "Toggle lazygit" },
    },
    config = function()
        require("toggleterm").setup({
            size = function(term)
                if term.direction == "horizontal" then return 15
                elseif term.direction == "vertical" then return vim.o.columns * 0.4
                end
            end,
            open_mapping = "<C-\\>",
            direction = "float",
            float_opts = { border = "curved" },
            shade_terminals = true,
        })

        -- Lazygit terminal
        local Terminal = require("toggleterm.terminal").Terminal
        local lazygit = Terminal:new({
            cmd = "lazygit",
            direction = "float",
            float_opts = { border = "curved" },
            hidden = true,
        })

        vim.keymap.set("n", "<leader>tg", function() lazygit:toggle() end, { desc = "Lazygit" })
    end,
}
```

---

## Neovim with tmux

### vim-tmux-navigator

Seamlessly navigate between tmux panes and Neovim splits with `Ctrl+h/j/k/l`:

```lua
-- ~/.config/nvim/lua/plugins/tmux-navigator.lua

return {
    "christoomey/vim-tmux-navigator",
    event = "VeryLazy",
    cmd = {
        "TmuxNavigateLeft", "TmuxNavigateDown",
        "TmuxNavigateUp", "TmuxNavigateRight",
    },
    keys = {
        { "<C-h>", "<cmd>TmuxNavigateLeft<cr>" },
        { "<C-j>", "<cmd>TmuxNavigateDown<cr>" },
        { "<C-k>", "<cmd>TmuxNavigateUp<cr>" },
        { "<C-l>", "<cmd>TmuxNavigateRight<cr>" },
    },
}
```

Add corresponding config to `~/.tmux.conf`:

```bash
# ~/.tmux.conf — vim-tmux-navigator integration
is_vim="ps -o state= -o comm= -t '#{pane_tty}' \
    | grep -iqE '^[^TXZ ]+ +(\\S+\\/)?g?(view|l?n?vim?x?|fzf)(diff)?$'"
bind-key -n 'C-h' if-shell "$is_vim" 'send-keys C-h'  'select-pane -L'
bind-key -n 'C-j' if-shell "$is_vim" 'send-keys C-j'  'select-pane -D'
bind-key -n 'C-k' if-shell "$is_vim" 'send-keys C-k'  'select-pane -U'
bind-key -n 'C-l' if-shell "$is_vim" 'send-keys C-l'  'select-pane -R'
```

---

## Next Steps

- **Set up tmux** → [02-TMUX.md](02-TMUX.md) — Configure tmux to work seamlessly with Neovim
- **Productivity tools** → [07-PRODUCTIVITY-TOOLS.md](07-PRODUCTIVITY-TOOLS.md) — fzf, ripgrep, and other tools that enhance Neovim
- **Platform setup** → [04-WINDOWS-TERMINAL.md](04-WINDOWS-TERMINAL.md), [05-LINUX-TERMINAL.md](05-LINUX-TERMINAL.md), or [06-MACOS-TERMINAL.md](06-MACOS-TERMINAL.md)
- **Best practices** → [08-BEST-PRACTICES.md](08-BEST-PRACTICES.md) — Version-control your Neovim configuration

---

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2025 | Initial Neovim documentation |
