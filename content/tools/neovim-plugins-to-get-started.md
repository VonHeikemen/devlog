+++
title = "Neovim: Plugins to get started"
description = "Turn Neovim into a fancy editor with these plugins"
date = 2022-09-03
updated = 2026-01-05
lang = "en"
[taxonomies]
tags = ["neovim", "vim", "shell"]
[extra]
shared = [
  ["dev.to", "https://dev.to/vonheikemen/neovim-plugins-to-get-started-5f77"],
]
+++

It happens very often that people start using Neovim but they miss some features that other text editors have. Listing open files in tabs, a file tree explorer, show the current git branch, and so on. So here I want to show you a set of plugins that will help you add those common features to Neovim.

The only thing I will not cover here is **autocompletion**. Smart code completion usually involves integrating with external tools. It means there is a fair amount of details that need to be explained, so there is dedicated post for that: [Getting started with Neovim's LSP client](https://dev.to/vonheikemen/getting-started-with-neovims-native-lsp-client-in-the-year-of-2022-the-easy-way-bp3).

All the configuration shown in this post will be in this repository: [nvim-starter - branch: 02-opinionated](https://github.com/VonHeikemen/nvim-starter/tree/02-opinionated).

## Requirements

If you're completely new to Neovim I recommend learning lua's syntax. If you don't want to learn everything at least have a [reference](https://learnxinyminutes.com/docs/lua/) to know what is valid. All the plugins I'll share here are configured using lua.

If you haven't created a configuration for Neovim, do it now. Here is a guide with everything you need to know: [Build your first Neovim configuration in lua](@/tools/build-your-first-lua-config-for-neovim.md).

You'll need Neovim's latest stable version. You can download it from the [release section](https://github.com/neovim/neovim/releases) in github. From now on I'll assume you are using Neovim v0.9.5 or greater.

## How do we install plugins?

First thing we should know is how to install a plugin manually. Turns out we only need to download them in a specific location and Neovim will be able to use it. We can list all the available directories using this command.

```vim
:set packpath?
```

That will show a comma separated list of paths. 

We can get something better using a little bit of lua.

```vim
:lua vim.tbl_map(print, vim.opt.packpath:get())
```

This one will show the same list but now every path will be in its own line.

In one of those directories we have to create another directory called `pack`. Inside `pack` we create the directory that will hold our plugins. The structure has to be something like this:

```txt
pack
└── plugins-from-github
    ├── opt
    │   ├── [plugin 1]
    │   └── [plugin 2]
    └── start
        └── [plugin 3]
```

Plugins in `opt` will only be loaded if we execute the command `:packadd`. The plugins in `start` will be loaded automatically during Neovim's startup process.

So let's assume we have this path in our `packpath`.

```
/home/dev/.local/share/nvim/site
```

We want to install plugins there, what do we do? Create a `pack` directory. Then create another directory with any name. Let's use the name `github` because why not? So the full path for our plugins will be this.

```
/home/dev/.local/share/nvim/site/pack/github
```

So to install a plugin like [mini.nvim](https://github.com/nvim-mini/mini.nvim) and have it load automatically, we should place it here.

```
/home/dev/.local/share/nvim/site/pack/github/start/mini.nvim
```

And that's it.

To know more about packages in Neovim read the help page.

```vim
:help packages
```

### Plugin manager

But of course we don't have to download plugins manually, we can use a plugin manager that handles everything for us.

At the moment these are the most popular plugin managers in the Neovim ecosystem.

* [lazy.nvim](https://github.com/folke/lazy.nvim)
* [mini.deps](https://nvim-mini.org/mini.nvim/readmes/mini-deps.html)
* [paq.nvim](https://github.com/savq/paq-nvim) 

Remember to read carefully the instructions of the plugin manager you choose.

## Plugins

### Tokyonight

Github: [folke/tokyonight.nvim](https://github.com/folke/tokyonight.nvim)

Because of course the first thing you have to do is change the default theme. We can achieve this by using the command `colorscheme` followed by the name of the theme.

In lua we can call vim commands using `vim.cmd`. So to apply the theme we have to do this.

```lua
vim.cmd('colorscheme name-of-theme')
```

This is the command we need to use tokyonight.

```vim
colorscheme tokyonight
```

### Bufferline

Github: [akinsho/bufferline.nvim](https://github.com/akinsho/bufferline.nvim)

You know how other editors show a tab for each open file? Okay, that's not how it works in Neovim. For starters we call them tabpages, they are like workspaces, inside a tabpage you can open multiple windows. You can even have different working directories per tabpage. But some people prefer the "traditional" behavior and this is what `bufferline` does, it modifies the tabline so it can show the currently opened files.

Here we find ourselves in a situation where we need to enable the plugin explicitly. The way we do it is by executing a function called `.setup()` located in the "main" module of the plugin. For bufferline to work we have to add this line of code to our personal configuration.

```lua
require('bufferline').setup({})
```

This `require` function is the mechanism we use to load a lua module. The string `bufferline` is the name of the module itself. And `.setup()` is a function call, we are telling the lua interpreter we want to execute a function with the name `setup`.

To customize the plugin's settings you provide a "lua table" to the `.setup()` function. How do we figure out what settings are available? By reading the documentation.

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
      {filetype = 'snacks_layout_box'}
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

* `options.mode`: With the value `'buffers'` will tell bufferline that we want to show one tab per file.

* `options.offsets`: Must be a list of filetypes. When one of these filetypes appears bufferline will avoid rendering a tab above the window. Here we are adding the `snacks_layout_box` because it'll be our file explorer. When the explorer window shows up it'll look like a side bar.

* `highlights`: With this we can modify the colors of the components in the tab. Each section (like `buffer_selected`) must be the name of a component. In this example I'm using `italic = false` to tell bufferline I want to disable italic characters. The property `fg` is the one that changes the color of the characters. In here I'm telling it to use the same highlight color my colorscheme uses for functions. If you want to see more details about highlights checkout `:help bufferline-highlights`.

Is worth noting bufferline also offers handful of user commands to navigate between files and some other stuff. But the one I've used the most is `BufferLinePick`. This will allow us to select a visible tab without using the mouse. And we can use it in a keymap for extra convenience.

```lua
vim.keymap.set('n', 'gt', '<cmd>BufferLinePick<cr>', {})
```

### mini.nvim

Github: [nvim-mini/mini.nvim](https://github.com/nvim-mini/mini.nvim)

Website: [nvim-mini.org](https://nvim-mini.org)

`mini.nvim` is a collection of lua modules. This is meant to improve our experience by enhancing Neovim's builtin features or implementing new ones.

There are over 40 modules inside mini.nvim but here I'll show 7 that I find very useful.

* [mini.statusline](https://nvim-mini.org/mini.nvim/doc/mini-statusline.html)

The statusline is a component of Neovim's UI. Is a line located at the bottom of each window. It usually separates the window from the message area. This is where Neovim shows information like the file name and the location of the cursor. And much like the tabline, the statusline can also be modified.

`mini.statusline` is an implementation of the statusline that looks a little bit better than the default. And it can also show information provided by other modules in mini.nvim.

Just like bufferline has a `.setup()` every module in mini.nvim follows the same convention. This is enough to make the plugin work.

```lua
require('mini.statusline').setup({})
```

If you want to know more details about a module in mini.nvim use the `:help` command and the name of the module.

```
:help mini.statusline
```

* [mini.git](https://nvim-mini.org/mini.nvim/doc/mini-git.html)

`mini.git` can track information about the current directory if it's git repository. And also provides a user command (`:Git`) that allows us to use git's cli inside Neovim.

By default `mini.statusline` will show the current git branch if we enable `mini.git`.

```lua
require('mini.git').setup({})
```

The `:Git` command provided by this plugin will try integrate Neovim whenever possible. For example, `:Git diff` will show the output of the diff in a Neovim buffer. `:Git commit` will use the current Neovim instance as the editor for the commit message. To know more details read the documentation.

```
:help mini.git
```

* [mini.diff](https://nvim-mini.org/mini.nvim/doc/mini-diff.html)

`mini.diff` is another "git based" module. This one can show if something was deleted, added or changed in real time.

Let me share with you my personal configuration.

```lua
vim.o.signcolumn = 'yes'
require('mini.diff').setup({
  view = {
    style = 'sign',
    signs = {
      add = '▎',
      change = '▎',
      delete = '➤',
    },
  },
})
```

Here I set the `signcolumn` option to `yes` because I want to reserve a space in the gutter, next to the line numbers. And inside the `mini.diff` settings I change the "style" to `sign` so it can use the signcolumn to show the indicator when there is a change in the code.

* [mini.comment](https://nvim-mini.org/mini.nvim/doc/mini-comment.html)

`mini.comment` provides keymaps to toggle comments in a line of code. By default it provides a `gc` operator which we can use in normal mode. For example `gci{` will toggle comments in all the lines the inside curly braces. And so anything combination that you can do with a regular operator like `d` or `c` you can do with `gc`. The keymap `gcc` will toggle only the current line.

```lua
require('mini.comment').setup({})
```

It is worth noting this feature `mini.comment` provides is now part of Neovim as of version v0.10. But Neovim's implementation is not very configurable, you can't change how it works or extend it in any way. `mini.comment` will work even on Neovim v0.9 and offers a few settings you can change, that includes the function used to provide the comment for the current line.

* [mini.notify](https://nvim-mini.org/mini.nvim/doc/mini-notify.html)

`mini.notify` is a custom implementation for `vim.notify`.

`vim.notify` is a lua function that plugin authors use to display a message to end users. The default implementation Neovim provides uses the message area. `mini.notify` implements a custom function so plugin notifications show up in a floating window on the corner.

```lua
require('mini.notify').setup({})
```

* [mini.clue](https://nvim-mini.org/mini.nvim/doc/mini-clue.html)

`mini.clue` will help you remember (and maybe discover) Neovim key combinations by showing them in a floating window.

Most of mini.nvim's module work just fine with their default settings but here we have to be explicit. We have to specify which key combination will trigger the floating with the clues. Here's a simple example.

```lua
local mini_clue = require('mini.clue')

mini_clue.setup({
  triggers = {
    {mode = 'n', keys = 'g'},
    {mode = 'n', keys = '<leader>'},
  },
  clues = {
    mini_clue.gen_clues.g(),
  },
})
```

In `triggers` we add the prefix of the key combination. And `clues` is where we place the descriptions of the keymaps if we need them.

The cool thing about `mini.clue` is we don't have to do anything special with our custom keymaps. If we create our keymaps with `vim.keymap.set()` and provide a description that will be enough for it to appear on `mini.clue`'s floating window. For example.

```lua
-- Space as a leader key
vim.g.mapleader = ' '

vim.keymap.set('n', '<leader>w', '<cmd>write<cr>', {desc = 'Save file'})
vim.keymap.set('n', '<leader>q', '<cmd>quit<cr>', {desc = 'Close window'})
```

`mini.clue` will be able to show these keymaps in its little floating window, because we specified `<leader>` as a trigger in the `.setup()` function.

For builtin keymaps where there is no description `mini.clue` provides a few functions under [gen_clues](https://nvim-mini.org/mini.nvim/doc/mini-clue.html#miniclue.gen_clues).

A more complete setup would look like this:

```lua
local mini_clue = require('mini.clue')

mini_clue.setup({
  window = {
    delay = 600,
    config = {
      width = 50,
    },
  },
  triggers = {
    {mode = 'n', keys = '['},
    {mode = 'n', keys = ']'},
    {mode = 'n', keys = 'g'},
    {mode = 'x', keys = 'g'},
    {mode = 'n', keys = 'z'},
    {mode = 'x', keys = 'z'},
    {mode = 'n', keys = '<C-w>'},
    {mode = 'i', keys = '<C-x>'},
    {mode = 'n', keys = '<leader>'},
    {mode = 'x', keys = '<leader>'},
  },
  clues = {
    mini_clue.gen_clues.g(),
    mini_clue.gen_clues.z(),
    mini_clue.gen_clues.windows(),
    mini_clue.gen_clues.builtin_completion(),
  },
})
```

Notice here we even have a `window` settings to control the width of the floating window, as well as the time it would take for it to appear on screen.

On to the limitations. One thing you should know is that `mini.clue` does not provide clues for operators. Combinations like `ciw` or `yap` can't be annotated. This is because keymaps like `c`, `y`, `d` are operators. A plugin simply can't hijack these operators without having weird side effects or write a non-trivial amount hacks to make it work.

* [mini.icons](https://nvim-mini.org/mini.nvim/doc/mini-icons.html)

`mini.icons` can be considered a multi-purpose module.

For plugin authors it provides an abstraction they can use to show icons with highlights.

For end users (like us) it provides a way to enable or disable fancy icons whenever possible. Some times a plugin will use icons but they don't offer a way to disable them. `mini.icons` can be used as a mechanism to enable or disable fancy icons.

```lua
require('mini.icons').setup({style = 'glyph'})
```

Here the `style` settings is the one that controls what kind of icons should be used. `glyph` is the default value. So having `mini.icons` enabled with the default settings means we want the fancy icons. Changing the `style` to `ascii` means `mini.icons` will use standard characters instead of symbols that may not be supported by the font you are using at the moment.

### vim-repeat

Github: [tpope/vim-repeat](https://github.com/tpope/vim-repeat)

Adds "dot repeat" support for other plugins. If you don't know, when we press the dot key (`.`) Neovim tries to repeat the last action we did. For example, if we delete a word using `diw` we can repeat that just by pressing `.`. With `vim-repeat` we can repeat actions made by plugins.

### Treesitter

Github: [nvim-treesitter/nvim-treesitter](https://github.com/nvim-treesitter/nvim-treesitter)

So treesitter is a library that was added to Neovim, with it Neovim can read your code in the same way a compiler does, it scans the code and creates an abstract syntax tree. In other words, it turns your code into a data structure Neovim can query.

By itself treesitter doesn't do anything useful, is more like a tool for Neovim mantainers and plugin authors, they will use treesitter to implement the features everyone else will interact with. For example, inside Neovim there is an alternative mechanism to do syntax highlight, it uses treesitter to assign the highlight groups to the text of a buffer.

Now, for treesitter to actually work we need a treesitter parser. The parser is the thing that reads the code. To install a parser we use the command `:TSInstall` followed by the name of the language.

So to install a javascript parser we use this command.

```vim
:TSInstall javascript
```

When it comes to features there isn't a unified interface we can use to "enable treesitter." We have to look up the documentation of the feature we want to use and follow the instructions.

To enable the syntax highlight based on treesitter the documentation recommends using a filetype plugin or an autocommand on the `FileType` event. And then we call the function `vim.treesitter.start()`. Here's an example using an autocommand to enable the syntax highlight in javascript related filetypes.

```lua
vim.api.nvim_create_autocmd('FileType', {
  pattern = {'javascript', 'javascriptreact', 'js', 'jsx'},
  callback = function()
    vim.treesitter.start()
  end,
})
```

### ts-enable.nvim

Github: [VonHeikemen/ts-enable.nvim](https://github.com/VonHeikemen/ts-enable.nvim)

Using `nvim-treesitter` now requires some knowledge about how Neovim works. You have to do some amount of work to enable features. Is not that bad if you are willing to install treesitter parsers manually. But it does get complicated when you want nice quality of life features, like installing parsers on demand. `ts-enable.nvim` is just an abstraction layer, it implements the boilerplate code needed to enable syntax highlight, code folding and indentation based on treesitter.

A basic configuration can look like this:

```lua
-- See :help ts-enable-config
vim.g.ts_enable = {
  parsers = {'lua', 'vim', 'vimdoc', 'c', 'query'},
  auto_install = true,
  highlights = true,
  indents = false,
  folds = false,
}
```

The idea here is that you just specify the names of the treesitter parsers you want to use and the features you want to enable. So you don't have to write any autocommands, `ts-enable.nvim` will take care of the details of each feature and can also install missing parsers using `nvim-treesitter`.

### Snacks.nvim

Github: [folke/snacks.nvim](https://github.com/folke/snacks.nvim)

Snacks.nvim is also a collection of lua modules. But this is different from mini.nvim. Snacks doesn't follow the same design as mini. A module in snacks.nvim doesn't have any self-inflicted limitation. So we could say Snacks is more ambitious than mini.

Most modules in Snacks.nvim can be enabled using a single setup function from the lua module `snacks`.

```lua
local Snacks = require('snacks')

Snacks.setup({
  ---
  -- This is where we write our settings
  ---
})
```

The `snacks` module is the main interface of Snacks.nvim. Everything that we interact with is inside this one lua module.

* [Snacks.input](https://github.com/folke/snacks.nvim/blob/main/docs/input.md)

`Snacks.input` is a custom implementation for `vim.ui.input()`.

`vim.ui.input()` is a lua function that plugin authors call to get some information from the user. For example, in a file explorer when trying to rename a file it makes sense to ask the user what is the new name for the file.

`Snacks.input` implementation shows a floating input in the center of the screen. Instead of using the message area like the default implementation does.

This is what we do to enable it.

```lua
Snacks.setup({
  input = {
    enabled = true,
    icon = '❯',
  },
})
```

* [Snacks.picker](https://github.com/folke/snacks.nvim/blob/main/docs/picker.md)

`Snacks.picker` does three things: 1) provide an interface to fuzzy find items on a list. 2) implements 50+ "pickers" out of the box. 3) provides a custom implementation for `vim.ui.select()`.

Fuzzy finders became popular in the Neovim community because it provided a way to navigate between files in a very fast way. And the cool thing is that "fuzzy finding" is a very general idea, at its core we are just filtering a list of items. Turns out this is useful for many other use cases. That's why Snacks has 50 something pickers already built-in. We have pickers for open files, project files, undo history, keymaps, commands, color schemes... and many others.

Now, `vim.ui.select()` is a lua function that plugin authors use to ask the user to pick one of several options. `Snacks.picker` has a custom implementation that uses its fuzzy find interface.

```lua
Snacks.setup({
  picker = {
    enabled = true,
    ui_select = true,
    prompt = '❯ ',
  },
})
```

Configuring the `picker` settings in the `.setup()` is optional if you don't want to use the custom implementation for `vim.ui.select()`. All the pickers in Snacks can be used even if we don't use the main `.setup()` function. 

How do we use these picker? The most convenient way would be to make them show up by pressing a keymap. You can start with these keymaps:

```lua
-- docs: https://github.com/folke/snacks.nvim/blob/main/docs/picker.md
vim.keymap.set('n', '<leader><space>', function() Snacks.picker('buffers') end, {desc = 'Search open files'})
vim.keymap.set('n', '<leader>ff', function() Snacks.picker('files') end, {desc = 'Search all files'})
vim.keymap.set('n', '<leader>fh', function() Snacks.picker('recent') end, {desc = 'Search file history'})
vim.keymap.set('n', '<leader>fg', function() Snacks.picker('grep') end, {desc = 'Search in project'})
vim.keymap.set('n', '<leader>fd', function() Snacks.picker('diagnostics') end, {desc = 'Search diagnostics'})
vim.keymap.set('n', '<leader>fs', function() Snacks.picker('lines') end, {desc = 'Buffer local search'})
vim.keymap.set('n', '<leader>u', function() Snacks.picker('undo') end, {desc = 'Undo history'})
vim.keymap.set('n', '<leader>/', function() Snacks.picker('pickers') end, {desc = 'Search picker'})
vim.keymap.set('n', '<leader>?', function() Snacks.picker('keymaps') end, {desc = 'Search keymaps'})
```

* [Snacks.indent](https://github.com/folke/snacks.nvim/blob/main/docs/indent.md)

`Snacks.indent` adds indent guides to the buffer. So we can visualize the indentation level of a block code.

```lua
Snacks.setup({
  indent = {
    enabled = true,
    char = '▏',
  },
})
```

By default the current scope where the cursor is located will be highlighted, and this highlight will have an animation. If we want to disable the animation we use a global variable.

```lua
vim.g.snacks_animate = false
```

To be clear, this will only disable the animation. The current scope will still be highlighted.

* [Snacks.bigfile](https://github.com/folke/snacks.nvim/blob/main/docs/bigfile.md)

This module is mostly for older Neovim versions.

We can use `Snacks.bigfile` to disable Neovim features when we open a "big file." Again, this is useful in older Neovim versions because the syntax highlight makes the editor slow to respond. However, in Neovim v0.11 there is an experimental asynchronous highlight enabled, which may eliminate the need for this.

```lua
Snacks.setup({
  bigfile = {
    -- Only use `bigfile` module on older Neovim versions
    enabled = vim.fn.has('nvim-0.11') == 0,
    notify = false,
    size = 1024 * 1024, -- 1MB
    setup = function(ctx)
      vim.cmd('syntax clear')
      vim.opt_local.syntax = 'OFF'
      local buffer = vim.b[ctx.buf]
      if buffer.ts_highlight then
        vim.treesitter.stop(ctx.buf)
      end
    end
  },
})
```

The idea here is that on files that are bigger than `1MB` we disable all syntax highlight. Notice the `enabled` setting will be true if we are using an older version of Neovim below v0.11. So you could omit this module altogether if your Neovim version is recent enough.

* [Snacks.bufdelete](https://github.com/folke/snacks.nvim/blob/main/docs/bufdelete.md)

`Snacks.bufdelete` offers a few functions we can use to delete a buffer without changing the window layout.

```lua
-- docs: https://github.com/folke/snacks.nvim/blob/main/docs/bufdelete.md
vim.keymap.set('n', '<leader>bc', function()
  Snacks.bufdelete()
end, {desc = 'Close buffer'})
```

* [Snacks.terminal](https://github.com/folke/snacks.nvim/blob/main/docs/terminal.md)

Neovim has its own [terminal emulator](https://neovim.io/doc/user/terminal.html). But this is basically a "special buffer," is not a widget or a component. We can't just "toggle" a terminal window with a keymap like in other modern editors.

`Snacks.terminal` provides functions to manage terminal windows with ease. For the most common use case we can just create a keymap to toggle a terminal.

```lua
-- docs: https://github.com/folke/snacks.nvim/blob/main/docs/terminal.md
vim.keymap.set({'n', 't'}, '<C-g>', function()
  Snacks.terminal.toggle()
end, {desc = 'Toggle terminal window'})
```

* [Snacks.explorer](https://github.com/folke/snacks.nvim/blob/main/docs/explorer.md)

`Snacks.explorer` is a file explorer with a tree style view. Under the hood this is actually a "picker" for `Snacks.picker`, so you will find that in the documentation some of the settings should be placed in the `picker` section of the setup.

To start using it we just enable it in the `.setup()` function and make a keymap.

```lua
Snacks.setup({
  explorer = {
    enabled = true,
    replace_netrw = true,
  },
})

vim.keymap.set('n', '<leader>e', function()
  Snacks.explorer()
end, {desc = 'Toggle file explorer'})
```

Now if we put everything together we should have something like this:

```lua
local Snacks = require('snacks')

Snacks.setup({
  indent = {
    enabled = true,
    char = '▏',
  },
  explorer = {
    enabled = true,
    replace_netrw = true,
  },
  input = {
    enabled = true,
    icon = '❯',
  },
  picker = {
    enabled = true,
    ui_select = true,
    prompt = '❯ ',
  },
  bigfile = {
    -- Only use `bigfile` module on older Neovim versions
    enabled = vim.fn.has('nvim-0.11') == 0,
    notify = false,
    size = 1024 * 1024, -- 1MB
    setup = function(ctx)
      vim.cmd('syntax clear')
      vim.opt_local.syntax = 'OFF'
      local buffer = vim.b[ctx.buf]
      if buffer.ts_highlight then
        vim.treesitter.stop(ctx.buf)
      end
    end
  },
})

-- Disable indent guide animation
vim.g.snacks_animate = false

-- docs: https://github.com/folke/snacks.nvim/blob/main/docs/explorer.md
vim.keymap.set('n', '<leader>e', function()
  Snacks.explorer()
end, {desc = 'Toggle file explorer'})

-- docs: https://github.com/folke/snacks.nvim/blob/main/docs/terminal.md
vim.keymap.set({'n', 't'}, '<C-g>', function()
  Snacks.terminal.toggle()
end, {desc = 'Toggle terminal window'})

-- Close while preserving window layout
-- docs: https://github.com/folke/snacks.nvim/blob/main/docs/bufdelete.md
vim.keymap.set('n', '<leader>bc', function()
  Snacks.bufdelete()
end, {desc = 'Close buffer'})

-- Fuzzy finders
-- docs: https://github.com/folke/snacks.nvim/blob/main/docs/picker.md
vim.keymap.set('n', '<leader><space>', function() Snacks.picker('buffers') end, {desc = 'Search open files'})
vim.keymap.set('n', '<leader>ff', function() Snacks.picker('files') end, {desc = 'Search all files'})
vim.keymap.set('n', '<leader>fh', function() Snacks.picker('recent') end, {desc = 'Search file history'})
vim.keymap.set('n', '<leader>fg', function() Snacks.picker('grep') end, {desc = 'Search in project'})
vim.keymap.set('n', '<leader>fd', function() Snacks.picker('diagnostics') end, {desc = 'Search diagnostics'})
vim.keymap.set('n', '<leader>fs', function() Snacks.picker('lines') end, {desc = 'Buffer local search'})
vim.keymap.set('n', '<leader>u', function() Snacks.picker('undo') end, {desc = 'Undo history'})
vim.keymap.set('n', '<leader>/', function() Snacks.picker('pickers') end, {desc = 'Search picker'})
vim.keymap.set('n', '<leader>?', function() Snacks.picker('keymaps') end, {desc = 'Search keymaps'})
```

## What's next?

Next step is to make Neovim really understand our code: have it autocomplete variables, setup jump to definition, rename variables, all that good stuff. To achieve this I recommend using the builtin LSP client, configure it using [nvim-lspconfig](https://github.com/neovim/nvim-lspconfig). I made another guide specifically for this:

* [Getting started with neovim's native LSP client](https://dev.to/vonheikemen/getting-started-with-neovims-native-lsp-client-in-the-year-of-2022-the-easy-way-bp3)

