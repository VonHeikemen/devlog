+++
title = "Neovim: Plugins to get started"
description = "Turn Neovim into a fancy editor with these plugins"
date = 2022-09-03
updated = 2023-02-15
lang = "en"
[taxonomies]
tags = ["neovim", "vim", "shell"]
[extra]
shared = [
  ["dev.to", "https://dev.to/vonheikemen/neovim-plugins-to-get-started-5f77"],
  ["Hashnode", "https://vonheikemen.hashnode.dev/neovim-plugins-to-get-started"],
]
+++

So you want to customize Neovim but you don't know where to start? Let me help you with that. I'll show you some plugins many people in the Neovim community have been using for a while now.

All the configuration shown in this post will be in this repository: [nvim-starter - branch: 02-opinionated](https://github.com/VonHeikemen/nvim-starter/tree/02-opinionated).

## Requirements

If you're completely new to Neovim I recommend you learn lua's syntax. If you don't want to learn everything at least have a [reference](https://learnxinyminutes.com/docs/lua/) to know what is valid. All the plugins I'll share here are configured using lua.

If you haven't created a configuration for Neovim, do it now. Here is a guide with everything you need to know: [Build your first Neovim configuration in lua](@/tools/build-your-first-lua-config-for-neovim.md).

You'll need Neovim's latest stable version. You can download it from the [release section](https://github.com/neovim/neovim/releases) of github. From now on I'll assume you are using Neovim v0.8 or greater.

## How do we install plugins?

First thing you should know is how to install a plugin manually. Turns out we only need to download them in a specific folder and Neovim will take care of the rest. We can list all the available directories using this command.

```vim
:set packpath?
```

That will show a comma separated list, which I find difficult to read. We can get something better using a little bit of lua.

```vim
:lua vim.tbl_map(print, vim.opt.packpath:get())
```

This one will show the same list but now every path will be in its own line.

In one of those directories we have to create a folder called `pack` and inside pack we must create a "package". A package is a folder that contains several plugins. It must have this structure.

```
package-folder
├── opt
│   ├── [plugin 1]
│   └── [plugin 2]
└── start
    ├── [plugin 3]
    └── [plugin 4]
```

In this example we are creating a folder with two other folders inside: `opt` and `start`. Plugins in `opt` will only be loaded if we execute the command `packadd`. The plugins in `start` will be loaded automatically during the startup process.

So let's assume we have this path in our `packpath`.

```
/home/dev/.local/share/nvim/site
```

We want to install plugins there, what do we do? Create a `pack` folder. Then create a folder with any name you want. Let's name the package `github` because why not? So the full path for our plugins will be this.

```
/home/dev/.local/share/nvim/site/pack/github
```

So to install a plugin like [lualine](https://github.com/nvim-lualine/lualine.nvim) and have it load automatically, we should place it here.

```
/home/dev/.local/share/nvim/site/pack/github/start/lualine.nvim
```

That's it. Well... you need to configure the plugin but that's another story.

To know more about packages in Neovim read the help page.

```vim
:help packages
```

### Plugin manager

But of course we don't have to download plugins manually, we can use a plugin manager that handles everything for us.

At the moment these are the most popular plugin managers in the Neovim ecosystem.

* [lazy.nvim](https://github.com/folke/lazy.nvim)
* [packer.nvim](https://github.com/wbthomason/packer.nvim)
* [paq.nvim](https://github.com/savq/paq-nvim) 

If you prefer minimalism take a look at `paq`. If you want something full of features use `packer` or `lazy.nvim`.

Remember to read carefully the instructions of the plugin manager you choose.

## Plugins

### Tokyonight

Github: [folke/tokyonight.nvim](https://github.com/folke/tokyonight.nvim)

Because of course the first thing you have to do is change the default theme. We can achieve this by using the command `colorscheme` followed by the name of the theme.

In lua we can call vim commands using `vim.cmd`. So to apply the theme we have to do this.

```lua
vim.cmd('colorscheme name-of-theme')
```

Here's how we apply tokyonight.

```vim
colorscheme tokyonight
```

### onedark.vim

Github: [joshdick/onedark.vim](https://github.com/joshdick/onedark.vim)

Port of Atom's default theme.

```vim
colorscheme onedark
```

### darkplus.nvim

Github: [lunarvim/darkplus.nvim](https://github.com/lunarvim/darkplus.nvim)

Port of VSCode's default theme.

```vim
colorscheme darkplus
```

### monokai.nvim

Github: [tanvirtin/monokai.nvim](https://github.com/tanvirtin/monokai.nvim)

Port of Sublime Text's default theme.

```vim
colorscheme monokai
```

### nvim-web-devicons

Github: [kyazdani42/nvim-web-devicons](https://github.com/kyazdani42/nvim-web-devicons)

Collection of functions to display icons. We usually don't interact with this plugin directly, is mostly used by other plugins, to allow them to show icons.

By itself `nvim-web-devicons` is not enough, you need to install a font that supports icons. You can find a good collection in [nerdfonts.com](https://www.nerdfonts.com).

### Lualine

Github: [nvim-lualine/lualine.nvim](https://github.com/nvim-lualine/lualine.nvim)

Lualine can give us a good looking statusline. If you don't know, the statusline is the thing at the bottom showing the filename and the position of the cursor. Lualine can make it pretty and also adds extra information.

To start using lualine we have to call the `setup` function of the `lualine` module.

```lua
require('lualine').setup({})
```

And how do we customize it? In the documentation they show [the default configuration](https://github.com/nvim-lualine/lualine.nvim#default-configuration).

```
:help lualine-Default-configuration
```

All we have to do is add the properties we care about in the argument of `.setup()`.

```lua
vim.opt.showmode = false

require('lualine').setup({
  options = {
    theme = 'onedark',
    icons_enabled = true,
    component_separators = '|',
    section_separators = '',
    disabled_filetypes = {
      statusline = {'NvimTree'}
    }
  },
})
```

The first thing I do in this example is disable `showmode` because lualine already shows the current mode.

Here is what we got in `.setup()`.

* `options.theme`: Is set to `onedark` because is the colorscheme I use. But we can change it to any of the [available themes](https://github.com/nvim-lualine/lualine.nvim/blob/master/THEMES.md) in the plugin.

* `options.icons_enabled`: When set to `true` it shows icons in the filetype section.

* `options.component_separators`: Is the character used between components.

* `options.section_separators`: Is the character used between sections.

* `options.disabled_filetypes.statusline`: Is a list of filetypes in which lualine will be disabled. We add `NvimTree` so the file explorer looks "cleaner".

### Bufferline

Github: [akinsho/bufferline.nvim](https://github.com/akinsho/bufferline.nvim)

You know how other editors show a tab for each open file? Okay, that's not how it works in Neovim. For starters we call them tabpages, they are like workspaces, inside them you can open multiple windows. You can even have different working directories per tabpage. But some people prefer the "traditional" behavior and this is what `bufferline` does, it modifies the tabline so it can show the currently opened files.

To start using this plugin we need to call the `.setup()` function of the `bufferline` module.

```lua
require('bufferline').setup({})
```

You can find a reference to the available options in the help page.

```vim
:help bufferline-configuration
```

Here's an example configuration.

```lua
require('bufferline').setup({
  options = {
    mode = 'buffers',
    offsets = {
      {filetype = 'NvimTree'}
    },
  }
  highlights = {
    buffer_selected = {
      italic = false
    },
    indicator_selected = {
      fg = {attribute = 'fg', highlight = 'Function'},
      italic = false
    }
  }
})
```

* `options.mode`: With the value `'buffers'` we tell bufferline that we want one tab per file.

* `options.offsets`: Must be a list of filetypes. When one of these filetypes appears bufferline will avoid rendering a tab above the window. Here we are adding the `NvimTree` because it'll will be our file explorer. When the nvim-tree window shows up it'll look like a side bar.

* `highlights`: With this we can modify the colors of the components in the tab. Each section (like `buffer_selected`) must be the name of a component. In this example I'm using `italic = false` to tell bufferline I want to disable italic characters. The property `fg` is the one that changes the color of the characters. In here I'm telling it to use the same highlight color my colorscheme uses for functions. If you want to see more details about highlights checkout `:help bufferline-highlights`.

### indent-blankline.nvim

Github: [lukas-reineke/indent-blankline.nvim](https://github.com/lukas-reineke/indent-blankline.nvim)

Adds indent guides in the current file.

This plugin in particular can be configured in several ways. We can tweak its behavior modifying global variables, like this.

```lua
vim.g.indent_blankline_char = '▏'
```

Or if you prefer vimscript.

```vim
let g: indent_blankline_char = '▏'
```

You can find the complete list of variables in the help page.

```vim
:help indent-blankline-variables
```

There is a third option: use the `.setup()` function of the `indent_blankline` module.

```lua
require('indent_blankline').setup({
  char = '▏',
})
```

Notice we don't need the prefix `indent_blankline_` when using `.setup()`.

I prefer to use `.setup()`. And here the options I find interesting.

```lua
require('indent_blankline').setup({
  char = '▏',
  show_trailing_blankline_indent = false,
  show_first_indent_level = false,
  use_treesitter = true,
  show_current_context = false
})
```

* `char`: Is the character that will appear on the screen.

* `show_trailing_blankline_indent`: Show indent guides in blank lines.

* `show_first_indent_level`: Show an indent guide on the first column.

* `use_treesitter`: Use treesitter to determine where the indent guide should be.

* `show_current_context`: Highlight the indent level where the cursor is at.

### Treesitter

Github: [nvim-treesitter/nvim-treesitter](https://github.com/nvim-treesitter/nvim-treesitter)

So treesitter is a library that was added to Neovim, with it Neovim can read your code in the same way a compiler does, it scans the code and creates an abstract syntax tree. In other words, it turns your code into a data structure Neovim can query.

By itself treesitter doesn't do anything useful, its the plugins that use treesitter the ones that add features to Neovim. Enter `nvim-treesitter`, it has modules that enhance Neovim's default features. For example `highlight`, the most popular module, can make the syntax highlight a lot more accurate than the default.

You can find details about nvim-treesitter modules in the help page.

```vim
:help nvim-treesitter-modules
```

To enable a module we must call the `.setup()` function in `nvim-treesitter.configs`. Then we specify the configuration for each module in a lua table.

```lua
require('nvim-treesitter.configs').setup({
  highlight = {
    enable = true,
  },
})
```

All modules in nvim-treesitter are disabled by default, so at the very least we must add `enable = true` to use it.

Now, for treesitter to actually work we need a language parser. The parser is the thing that reads the code. To install a parser we use the command `TSInstall` followed by the name of the language.

So to install a javascript parser we use this command.

```vim
:TSInstall javascript
```

We can also tell nvim-treesitter what parsers we always want installed. For this we use the `ensure_installed` property in the `.setup()` function.

```lua
require('nvim-treesitter.configs').setup({
  highlight = {
    enable = true,
  },
  ensure_installed = {
    'javascript',
    'typescript',
    'tsx',
    'css',
    'json',
    'lua',
  },
})
```

### nvim-treesitter-textobjects

Github: [nvim-treesitter/nvim-treesitter-textobjects](https://github.com/nvim-treesitter/nvim-treesitter-textobjects)

Text objects in Neovim are patterns of text, things like words, paragraphs, xml tags, etc. `nvim-treesitter-textobjects` adds more text objects based on treesitter queries. Functions, classes, conditional statements and loops can be available as text objects if we enable them with `nvim-treesitter-textobjects`.

Recap time: Text objects are used in "operator pending" mode. When you use an operator like `d` (the keyboard shorcut for delete) you enter operator pending mode. In this mode Neovim will wait for you to provide a text object or a motion. So when you press something like `diw`, that `d` is an operator and `iw` is the text object. You can learn more about motions and text objects if you read the help page.

```vim
:help motion.txt
```

Let's go back to that treesitter thing.

nvim-treesitter-textobjects has its own submodules.

```vim
:help nvim-treesitter-textobjects-modules
```

To configure a module we use the same `.setup()` function in `nvim-treesitter.configs`. 

```lua
require('nvim-treesitter.configs').setup({
  highlight = {
    enable = true,
  },
  textobjects = {
    select = {
      enable = true,
      lookahead = true,
      keymaps = {
        ['af'] = '@function.outer',
        ['if'] = '@function.inner',
        ['ac'] = '@class.outer',
        ['ic'] = '@class.inner',
      }
    },
  },
  ensure_installed = {
    --- parsers....
  },
})
```

Here we configure the `select` module in `textobjects`, with it we can add the new text objects. Now inside `select` we have the following options:

* `enable`: Loads the module.

* `lookahead`: Makes the cursor jump to the nearest match.

* `keymaps`: The keybindings for the text objects. In the left hand side we declare the keyboard shortcuts. In the right hand side we have the "capture groups", they represent a treesitter query. You can find the full list of groups here: [Builtin textobjects](https://github.com/nvim-treesitter/nvim-treesitter-textobjects#built-in-textobjects).

### targets.vim

Github: [wellle/targets.vim](https://github.com/wellle/targets.vim)

This plugin creates new text objects. It adds things like tags, pairs, function arguments, etc. I recommend you read the official documentation, you'll find detailed information (with examples) of every new text object: [targets.vim - Overview](https://github.com/wellle/targets.vim#overview).

You can configure some things but its not necessary, you can start using it without adding anything in your Neovim configuration. But if you wish, you can find the available options in the help page.

```vim
:help targets-settings
```

### Comment.nvim

Github: [numToStr/Comment.nvim](https://github.com/numToStr/Comment.nvim)

Adds a new operator to toggle comments in code. By default the operator is bound to the keyboard shorcut `gc`. This means we are allowed to use any combination Neovim can do in operator pending mode. We can comment a word using `gciw`, comment a paragraph with `gcap`, comment an arbitrary number of lines with `gc` + number + `j`. Really the only limitation will be your knowledge of Neovim, like how many motions or text objects you know.

`gc` will also work in visual mode. In visual mode it'll comment the lines selected.

In normal mode use `gcc` to comment individual lines.

The only thing you need to do to make it work is call the `.setup()` function of the `Comment` module.

```lua
require('Comment').setup({})
```

To know what options are available read the documentation on github: [Comment.nvim - Setup](https://github.com/numToStr/Comment.nvim#%EF%B8%8F-setup).

### vim-surround

Github: [tpope/vim-surround](https://github.com/tpope/vim-surround)

This plugin is all about manipulation of surrounding patterns. We can add, delete and change surroundings for a piece of text.

Let me explain.

If we have `'Hello, world'` we can delete the quotes using `ds'`. So `ds` is the keybinding to delete and `'` is the "surrounding pattern" we want to delete.

If we want to add a surrounding we use `ys`. Say we have the word `Hello`, if we want to wrap it in paranthesis we use `ysiw)`. `ys` is the keybinding to add a surrounding, `iw` is the text object for a word and `)` is the pattern we want to add. The result will be `(Hello)`.

If we are in visual mode we can use `S` + a pattern to surround the selected text.

To change a surrounding we use `cs`. Say we have `'Hello, world'`, if we want to change single quotes for double quotes we use `cs'"`. `cs` is the keybinding to change a surrounding, `'` is the thing we want to change and `"` is the new surrounding.

### nvim-tree

Github: [kyazdani42/nvim-tree.lua](https://github.com/kyazdani42/nvim-tree.lua)

File manager plugin. It shows all files in a tree style view, like most other editors do. It has all the usual features: create, delete and rename files.

To start using this plugin we need to call the `.setup()` function of the `nvim-tree` module.

```lua
require('nvim-tree').setup({})
```

All the keybindings available inside the file explorer are listed in the help page.

```vim
:help nvim-tree-default-mappings
```

You can also find a reference to the available options.

```vim
:help nvim-tree-setup
```

Here is an example configuration.

```lua
require('nvim-tree').setup({
  hijack_cursor = false,
  on_attach = function(bufnr)
    local bufmap = function(lhs, rhs, desc)
      vim.keymap.set('n', lhs, rhs, {buffer = bufnr, desc = desc})
    end

    -- See :help nvim-tree.api
    local api = require('nvim-tree.api')
   
    bufmap('L', api.node.open.edit, 'Expand folder or go to file')
    bufmap('H', api.node.navigate.parent_close, 'Close parent folder')
    bufmap('gh', api.tree.toggle_hidden_filter, 'Toggle hidden files')
  end
})

vim.keymap.set('n', '<leader>e', '<cmd>NvimTreeToggle<cr>')
```

Here I'm using `<leader>e` to toggle the file manager. Then in `.setup()` we have the actual configuration.

* `hijack_cursor`: When set to `true` nvim-tree will place the cursor at the beginning of the name in a node. Set it to `false` to disable it.

* `on_attach`: A callback function, it will be executed everytime nvim-tree opens the file manager. It's recommended in the documentation that we set our keybindings here. We use the module `nvim-tree.api` to access nvim-tree's functions. We use `vim.keymap.set` to create each keybinding. We bind `L` to the function `api.node.open.edit` to open a node in the tree. We use `H` close the node below the cursor. We use `gh` to toggle hidden files.

### Telescope

Github: [nvim-telescope/telescope.nvim](https://github.com/nvim-telescope/telescope.nvim)

Telescope's purpose is to provide an interface to filter a list of items. What can we do with this? We can search recently opened files, currently opened files, git commits, command history, keybindings, colorschemes... and 45 other things. Yes, in its current state telescope has 51 commands. Want to know how I know? You can search for a telescope command using telescope.

```vim
:Telescope builtin
```

We can use telescope's commands without any sort of configuration, really. Defaults are actually pretty good. But if you like to know what options are available you can read the help page.

```vim
:help telescope.setup()
```

Now the only thing we need to do is create keybindings for the commands we will like to use frequently.

```lua
vim.keymap.set('n', '<leader><space>', '<cmd>Telescope buffers<cr>')
vim.keymap.set('n', '<leader>?', '<cmd>Telescope oldfiles<cr>')
vim.keymap.set('n', '<leader>ff', '<cmd>Telescope find_files<cr>')
vim.keymap.set('n', '<leader>fg', '<cmd>Telescope live_grep<cr>')
vim.keymap.set('n', '<leader>fd', '<cmd>Telescope diagnostics<cr>')
vim.keymap.set('n', '<leader>fs', '<cmd>Telescope current_buffer_fuzzy_find<cr>')
```

* `<leader><space>`: Search opened files.
* `<leader>?`: Search recently opened files.
* `<leader>ff`: Search files in the current working directory.
* `<leader>fg`: Search for a pattern in all files in the current working directory.
* `<leader>fd`: Search diagnostic messages. A diagnostic can be a error, a warning or a hint.
* `<leader>fs`: Search for a pattern in the current file.

Is worth mention telescope can improve its performance (in some cases) if we install external tools like [fd](https://github.com/sharkdp/fd) and [ripgrep](https://github.com/BurntSushi/ripgrep). 

### telescope-fzf-native

Github: [nvim-telescope/telescope-fzf-native.nvim](https://github.com/nvim-telescope/telescope-fzf-native.nvim)

This is an extension for `telescope`. It allows telescope to use the same search algorithm [fzf](https://github.com/nvim-telescope/telescope-fzf-native.nvim) uses. This means we can use the same syntax in our search queries. And also improves the performance of the search.

This plugin is written in `C` so you will need to compile it before using it. Make sure you have available a `C` compiler and `make` (the build tool).

To use this extension in telescope we need to call the `.load_extension()` function in telescope.

```lua
require('telescope').load_extension('fzf')
```

### Toggleterm

Github: [akinsho/toggleterm.nvim](https://github.com/akinsho/toggleterm.nvim)

The good news is Neovim has an "integrated terminal" and we really don't need any plugin to use it. The bad news is not like a "UI component" we can toggle easily, is more like a special type of buffer. Toggleterm manages windows with terminal buffers, allowing us to toggle them with one keybinding.

Toggleterm has a whole bunch of features I recommend you read the documentation on github: [Toggleterm - roadmap](https://github.com/akinsho/toggleterm.nvim#roadmap).

To start using this plugin we need to call the `.setup()` function of `toggleterm`. Here is an example with the options I find interesting.

```lua
require('toggleterm').setup({
  open_mapping = '<C-g>',
  direction = 'horizontal',
  shade_terminals = true
})
```

* `open_mapping`: Is the keybinding we want to use to toggle the terminal window.

* `direction`: Determines the position of the terminal window. Posible values are `horizontal`,  `vertical` or `float`.

* `shade_terminals`: Darken the background color of the terminal window, to make it different from "normal windows".

### vim-fugitive

Github: [tpope/vim-fugitive](https://github.com/tpope/vim-fugitive)

Provides a graphical interface to manage a git repository inside Neovim. It also wraps the `git` cli so you can invoke any command from inside Neovim.

You can read the help page if you want to know all the details.

```vim
:help fugitive.txt
```

### Gitsigns

Github: [lewis6991/gitsigns.nvim](https://github.com/lewis6991/gitsigns.nvim)

Gitsigns is a plugin you can use to show "signs" in any line that has been changed somehow. It will let us know which lines have been added, where we deleted a line and which ones have been modified. It has a lot more functions, too many to list here. Check the help page if want to learn more.

```vim
:help gitsigns-functions
```

You can find a description of every available option in the help page.

```vim
:help gitsigns-config
```

If you just want to customize the signs text here's how you do it.

```lua
require('gitsigns').setup({
  signs = {
    add = {text = '▎'},
    change = {text = '▎'},
    delete = {text = '➤'},
    topdelete = {text = '➤'},
    changedelete = {text = '▎'},
  }
})
```

* `signs.add.text`: Sign for added lines.
* `signs.change.text`: Sign for modified lines.
* `signs.delete.text`: This sign shows where a line was deleted.
* `signs.topdelete.text`: This sign shows if the first lines of the file have been deleted.
* `signs.changedelete.text`: Sign for modified lines that occupy the same space as a deleted line.

### Plenary

Github: [nvim-lua/plenary.nvim](https://github.com/nvim-lua/plenary.nvim)

Plenary is collection of lua modules. It has bunch of utility functions other plugin authors use to solve common problems.

Telescope uses this plugin internaly, so make sure you install this.

### vim-repeat

Github: [tpope/vim-repeat](https://github.com/tpope/vim-repeat)

Adds "dot repeat" support for other plugins. If you don't know, when we press the dot key (`.`) Neovim tries to repeat the last action we did. For example, if we delete a word using `diw` we can repeat that just by pressing `.`. With `vim-repeat` we can repeat actions made by plugins.

### Editorconfig

Github: [editorconfig/editorconfig-vim](https://github.com/editorconfig/editorconfig-vim)

There is a popular configuration file known as [EditorConfig](https://editorconfig.org/), it concerns itself with style options like indentation, file encoding, line ends, that sort of things. Several editors have support for this file and with this plugin Neovim can be one of them.

### vim-bbye

Github: [moll/vim-bbye](https://github.com/moll/vim-bbye)

Provides commands so we can close a buffer without messing with our layout. Say you have two windows opened, if you close a file with the builtin command `bdelete` you'll close the buffer and the window. With `vim-bbye` we can use the command `Bdelete` to delete a buffer and leave the window open.

All you need to do is make a keybinding to close a buffer with `Bdelete`.

```lua
vim.keymap.set('n', '<leader>bc', '<cmd>Bdelete<CR>')
```

## What's next?

Next step is to make Neovim really understand our code: have it autocomplete variables, setup jump to definition, rename variables, all that good stuff. To achieve this I recommend using the builtin LSP client, configure it using [nvim-lspconfig](https://github.com/neovim/nvim-lspconfig) and then setup autocomplete with [nvim-cmp](https://github.com/hrsh7th/nvim-cmp). But doing that is not exactly easy, I made another guide specifically for this:

* [Setup nvim-lspconfig + nvim-cmp](@/tools/setup-nvim-lspconfig-plus-nvim-cmp.md)

## Where can we find more plugins?

Here are a few resources you can check, where you can find cool things about Neovim.

* [awesome-neovim](https://github.com/rockerBOO/awesome-neovim)
* [neovimcraft](https://neovimcraft.com/)
* [this week in neovim](https://this-week-in-neovim.org/)

## Conclusion

What did we do today? We looked at several colorschemes for Neovim. We learned about plugins we can use to bring features of other editors to Neovim. Like with `bufferline` we can have a tab per open file. Nvim-tree provides a tree-style file manager. We can use `gitsigns` and `fugitive` to integrate with git. We can search for all kinds of things with telescope. We can manipulate text in cool ways with plugins like `nvim-treesitter` and `vim-surround`. With all of this we can have a really good experience with Neovim.

