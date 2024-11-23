+++
title = "Simple Neovim config"
description = "Learn the basics of Neovim configuration in lua"
date = 2024-09-12
updated = 2024-10-05
lang = "en"
[taxonomies]
tags = ["vim", "neovim", "shell"]
+++

Can I offer you a nice Neovim configuration in this trying time?

40 lines of code. That's all you need to have a solid starting point. Believe it or not, even those IDE-like features people talk about online can be enabled using just **1 plugin** and a few keymaps.

You can try [this simple config](#init-lua) and then add new things to it based on the problems you encounter along the way. Or just take this as an oportunity to learn about the basics of Neovim configuration.

## Installing Neovim

We are going to need Neovim v0.10 or greater. v0.10 was released on `May 16 2024` so it's still somewhat recent. Not every package manager will have that version available.

If you know how to install precompiled binaries you can download the latest version from the [github releases](https://github.com/neovim/neovim/releases).

There is also a Neovim version manager called [bob](https://github.com/MordechaiHadad/bob). You can try that.

## The config file

The entry point for our Neovim configuration will be a file called `init.lua`. The location of this file can change depending on the operating system, so to save ourselves some trouble we will create it using Neovim in headless mode.

Execute this command on the terminal.

```sh
nvim --headless -c 'exe "write ++p" stdpath("config") . "/init.lua"' -c 'quit'
```

Once the configuration file exists we can check where it is using this command in the terminal:

```sh
nvim --headless -c 'echo $MYVIMRC' -c 'quit'
```

Now we can start adding some code. But first, if you are not familiar with lua I suggest taking some time to understand the syntax of the language.

* [Lua crash course (12 min video)](https://www.youtube.com/watch?v=NneB6GX1Els)
* [Learn X in Y minutes: Where X = lua](https://learnxinyminutes.com/docs/lua/) 

There is no need to memorize everything or become a lua expert, just review the syntax and have a reference at hand.

## Editor settings

Neovim's defaults are actually good but there are still a few options that need to be enabled/changed. To do this in lua [vim.o](https://neovim.io/doc/user/lua-guide.html#_vim.o) is the [recommended method](https://github.com/neovim/neovim/issues/30383#issuecomment-2351519326). We access a property through `vim.o` and assign a value to it. Like this.

```lua
vim.o.option_name = value
```

Where `option_name` can be anything on this list: [Quickref - option list](https://neovim.io/doc/user/quickref.html#option-list).

Here's some basic settings that you may find useful:

* Enable line numbers.

```lua
vim.o.number = true
```

* Disable line wrapping

```lua
vim.o.wrap = false
```

* Adjust the width of the tab character.

```lua
vim.o.tabstop = 2
vim.o.shiftwidth = 2
```

* Ignore case when the search pattern is all lowercase

```lua
vim.o.smartcase = true
vim.o.ignorecase = true
```

* Clear search highlights after submit

```lua
vim.o.hlsearch = false
```

* Reserve a space in the gutter for signs. Some plugins use this to show icons.

```lua
vim.o.signcolumn = 'yes'
```

## Changing Neovim's theme

The way we do this in lua is using [vim.cmd](https://neovim.io/doc/user/lua-guide.html#_vim-commands). Use `vim.cmd` to execute the command `colorscheme`.

```lua
vim.cmd.colorscheme('retrobox')
```

## Clipboard interaction

This is where we learn about [vim.keymap.set()](https://neovim.io/doc/user/lua-guide.html#_mappings), the function we use to create new keymaps.

By default when you copy some text using the `y` keymap Neovim will store that in a register. The `p` keymap, which we use to paste, also uses the same register as `y`. Basically, Neovim will ignore the system clipboard. We can still use it but we have to be explicit.

Now, I like to keep the default behavior and what I do is create new keymaps to use the system clipboard.

```lua
vim.keymap.set({'n', 'x', 'o'}, 'gy', '"+y', {desc = 'Copy to clipboard'})
vim.keymap.set({'n', 'x', 'o'}, 'gp', '"+p', {desc = 'Paste clipboard text'})
```

With this `gy` will be the keymap to copy content to the clipboard, and `gp` will paste content from the clipboard to Neovim.

Side note: `{'n', 'x', 'o'}` is the list of vim modes where we want our keymap to be created. `n` is normal mode, `x` is visual mode and `o` is operator pending mode.

## The leader key

Usually we don't go around making keymaps that override the defaults like we did in the previous section. Most of the time we prefix our custom keymaps with the special string `<leader>`. This way we don't override an important feature or cause some sort of conflict.

But what's `<leader>`?

By default is the `\` key. So, if we create a keymap like this:

```lua
vim.keymap.set('n', '<leader>h', '<cmd>echo "hello there"<cr>')
```

Then `\ + h` will print the message `hello there`.

But we can change that using the **vim global variable** `mapleader`.

```lua
vim.g.mapleader = ','
```

With this your leader key will change from `\` to `,`. But note that if you want to use a special sequence like `Control + k` you'll have to use its keycode.

```lua
vim.g.mapleader = vim.keycode('<C-k>')
```

Most people use the `Space key` as their leader key.

```lua
vim.g.mapleader = ' '
```

Yes, that's a string with a blank space. `vim.keycode` is a recent addition to Neovim, before that function was added people just did that thing with the blank space. But you, you can be explicit if you want.

```lua
vim.g.mapleader = vim.keycode('<Space>')
```

Is important to note order matters, you **must** to define `mapleader` before you use `<leader>` in your keymaps.

## Command shorcuts

Now let's make keymaps that can execute [Ex-commands](https://neovim.io/doc/user/vimindex.html#_6.-ex-commands).

There are two ways of doing it, and this is my favorite:

```
<cmd>...<cr>
```

`<cmd>` is a special sequence that tells Neovim that you want to execute a vim expression. `<cr>` represents the enter key, it marks the end of the expression you want to execute.

Here are a couple of examples:

* Save current file.

```lua
vim.keymap.set('n', '<leader>w', '<cmd>write<cr>', {desc = 'Save file'})
```

* Close all files and exit Neovim.

```lua
vim.keymap.set('n', '<leader>q', '<cmd>quitall<cr>', {desc = 'Exit vim'})
```

## Installing plugins

In Neovim there is something called [vim packages](@/tools/installing-neovim-plugins-without-a-plugin-manager.md). Long story short, we can put plugins in a folder in they should work.

We are going to keep it simple here. 2 plugins is all we want to get started. And because of this I will say we can download the plugins manually.

I going to recommend that you `git clone` plugins in your Neovim config folder (for now). Of course, this means **you** will be in charge of updating and removing plugins manually. But at this point I think the focus should be using Neovim for coding. If you decide that you actually like Neovim then you search for a plugin manager.

Okay. Where exactly do we download plugins?

Let's use Neovim in headless mode again, this time to create the folder were we can download plugins.

This command will create the folder we need.

```sh
nvim --headless -c 'call mkdir(stdpath("config") . "/pack/vendor/start/", "p")' -c 'quit'
```

And this other command will show the location.

```sh
nvim --headless -c 'echo stdpath("config") . "/pack/vendor/start/"' -c 'quit'
```

Now navigate to that location using your terminal. You will want to execute the command `git clone` inside that folder.

* Install [nvim-lspconfig](https://github.com/neovim/nvim-lspconfig)

We will use this plugin to help us enable IDE-like features.

```
git clone https://github.com/neovim/nvim-lspconfig
```

* Install [mini.nvim](https://github.com/echasnovski/mini.nvim) 

`mini.nvim` is a collection of lua modules aim to solve common problems and enhance some of Neovim builtin features.

```
git clone --filter=blob:none https://github.com/echasnovski/mini.nvim
```

After we install a plugin we have to generate the help tags.

```sh
nvim --headless -c 'helptags ALL' -c 'quit'
```

Now Neovim's config folder should have this structure.

```
nvim
├── init.lua
└── pack
    └── vendor
        └── start
            ├── mini.nvim
            └── nvim-lspconfig
```

Remember to delete the `pack` folder when you start using a plugin manager. You'll want to re-install those plugins with the plugin manager.

## File explorer

You should know that Neovim already has a file explorer, [it's called netrw](@/tools/using-netrw-vim-builtin-file-explorer.md). We are not going to use it here because it has a few quirks I dislike. Still, I think you should know it exists.

Since we have `mini.nvim` available let's use the file explorer it provides: [mini.files](https://github.com/echasnovski/mini.nvim/blob/main/readmes/mini-files.md).

First, we call `mini.files` setup function.

```lua
require('mini.files').setup({})
```

This is a common convention of lua plugins. Sometimes **you have to** call a function called `.setup()` to enable the plugin. Notice I said is a convention, it is not a rule. Some plugins execute their setup automatically, without any user intervention.

How do we use `mini.files`?

We make a keymap to open the file explorer.

```lua
vim.keymap.set('n', '<leader>e', '<cmd>lua MiniFiles.open()<cr>', {desc = 'File explorer'})
```

After you open the file explorer press `g` then `?`, this will show the keymaps you can use.

What if we want to change the settings in `mini.files`?

You'll want to add your custom settings inside the `{}` of the setup function. Here's an example.

```lua
require('mini.files').setup({
  mappings = {
    show_help = 'gh',
  },
})
```

This will change the keymap that shows the help window. Instead of `g?` now is `gh` (get help).

These custom settings are in a "lua table." If there is piece of syntax that you should learn is that one. Knowing the correct syntax of a lua table will prevent so many headaches.

And how did I know those settings exists?

In Neovim's command-line I executed this command.

```vim
:help mini.files
```

That took me to [the help page of mini.files](https://github.com/echasnovski/mini.nvim/blob/main/doc/mini-files.txt).

Remember when we installed the plugins? We generated some called help tags. These tags allow you to navigate to the "offline" documentation of the plugin: the help page.

The tab complete feature of the `:help` command is really good too. You can type `:help mini` and then press the `Tab` key, Neovim will show you all the help tags that start with `mini`.

## Navigating between files

For educational purposes only, I will tell you what I do when I don't have plugins installed: I use the `files` command to list the open files, then use the `buffer` command to navigate the file I want. And to make it convenient I make a keymap.

```lua
vim.keymap.set('n', '<leader><space>', '<cmd>files<cr>:buffer ', {desc = 'Search open files'})
```

In the `:buffer` command you can tab complete the name of the file or use the "id" of the buffer.

But why do that when we have `mini.nvim` installed? We can use the module [mini.pick](https://github.com/echasnovski/mini.nvim/blob/main/readmes/mini-pick.md). This one provides a fancy interactive filter. And it also gives us commands we can use to search files.

If you use the module [mini.extra](https://github.com/echasnovski/mini.nvim/blob/main/readmes/mini-extra.md) you will have access to more search commands in `mini.pick`.

Anyway, enable `mini.pick` module.

```lua
require('mini.pick').setup({})
```

Then create keymaps to start the search between files.

```lua
vim.keymap.set('n', '<leader><space>', '<cmd>Pick buffers<cr>', {desc = 'Search open files'})
vim.keymap.set('n', '<leader>ff', '<cmd>Pick files<cr>', {desc = 'Search all files'})
vim.keymap.set('n', '<leader>fh', '<cmd>Pick help<cr>', {desc = 'Search help tags'})
```

## Code completion

For this we use the [mini.completion](https://github.com/echasnovski/mini.nvim/blob/main/readmes/mini-completion.md) module.

```lua
require('mini.completion').setup({})
```

This is kind of a wrapper around Neovim's builtin completion mechanism. Neovim makes you press specific keybindings depending on the type of completion you want to get. And what `mini.completion` can do is take that burden away from you, by triggering the completion automatically as you type. In other words, it adds true autocompletion to Neovim.

By default you just get suggestions based on words of the current file. But whenever possible it'll try to use the LSP client to provide smart code completion.

To control the completion menu we use Neovim's default keybindings:

* `<Down>`: Select the next item on the list.

* `<Up>`:  Select previous item on the list.

* `<Ctrl-n>`: Select and insert text of the next item on the list.

* `<Ctrl-p>`: Select and insert text of the previous item on the list.

* `<Ctrl-y>`: Confirm selected item.

* `<Ctrl-e>`: Cancel the completion.

* `<Enter>`: If item was selected using `<Up>` or `<Down>` it confirms selection. If no item is selected, hides completion menu. Else, inserts a newline character.

## The LSP client

Neovim's LSP client is the thing that enables the nice IDE-like features people like so much. Things like go to definition, rename variable, inspect function signature. You get the idea. But this "client" doesn't work on its own, it needs a server. In this case a "server" is an external program that implements the [LSP specification](https://microsoft.github.io/language-server-protocol/overviews/lsp/overview/). If you want to know more details about LSP and language servers, watch this video by TJ DeVries: [LSP explained (5 min)](https://www.youtube.com/watch?v=LaS32vctfOY).

What do we need to do here? We follow these 3 steps.

* Create some keymaps
* Install a language server
* Configure that language server

### Step 1: keymaps

Some of the keymaps I'm about to show are based on Neovim's defaults, they do a similar thing as the default keymap but use the LSP client. And the rest will use the prefix `<leader>l`.

We create these keymaps on the autocommand `LspAttach` because we want to follow the same pattern Neovim has, that is enable these features only when there is a language server active in the file.

```lua
vim.api.nvim_create_autocmd('LspAttach', {
  desc = 'LSP actions',
  callback = function(event)
    local opts = {buffer = event.buf}

    -- Display documentation of the symbol under the cursor
    vim.keymap.set('n', 'K', '<cmd>lua vim.lsp.buf.hover()<cr>', opts)

    -- Jump to the definition
    vim.keymap.set('n', 'gd', '<cmd>lua vim.lsp.buf.definition()<cr>', opts)

    -- Format current file
    vim.keymap.set({'n', 'x'}, 'gq', '<cmd>lua vim.lsp.buf.format({async = true})<cr>', opts)

    -- Displays a function's signature information
    vim.keymap.set('i', '<C-s>', '<cmd>lua vim.lsp.buf.signature_help()<cr>', opts)

    -- Jump to declaration
    vim.keymap.set('n', '<leader>ld', '<cmd>lua vim.lsp.buf.declaration()<cr>', opts)

    -- Lists all the implementations for the symbol under the cursor
    vim.keymap.set('n', '<leader>li', '<cmd>lua vim.lsp.buf.implementation()<cr>', opts)

    -- Jumps to the definition of the type symbol
    vim.keymap.set('n', '<leader>lt', '<cmd>lua vim.lsp.buf.type_definition()<cr>', opts)

    -- Lists all the references
    vim.keymap.set('n', '<leader>lr', '<cmd>lua vim.lsp.buf.references()<cr>', opts)

    -- Renames all references to the symbol under the cursor
    vim.keymap.set('n', '<leader>ln', '<cmd>lua vim.lsp.buf.rename()<cr>', opts)

    -- Selects a code action available at the current cursor position
    vim.keymap.set('n', '<leader>la', '<cmd>lua vim.lsp.buf.code_action()<cr>', opts)
  end,
})
```

### Step 2: Install a language server

The Neovim community has created a resource where we can find a list of language servers. This list is in `nvim-lspconfig` documentation: [configs.md](https://github.com/neovim/nvim-lspconfig/blob/master/doc/configs.md).

Since I need some examples for the next step I'm going to show the install commands for the `go` language server and the `rust` language server.

If you have the toolchain for the [go programming language](https://go.dev/) you can download its language server (`gopls`) using this command.

```
go install golang.org/x/tools/gopls@latest
```

If you have [rustup](https://rust-lang.github.io/rustup/) installed you can download `rust_analyzer` using this command.

```
rustup component add rust-analyzer
```

### Step 3: Configure a language server

For each language server you have installed in your system you have to add their setup function using this syntax.

```lua
require('lspconfig').example_server.setup({})
```

Where `example_server` is the name of the language server we have installed.

If you have the language server for `go` and `rust` you would do this.

```lua
require('lspconfig').gopls.setup({})
require('lspconfig').rust_analyzer.setup({})
```

I do recommend that you read the documentation of the language server you want to use. That's usually the place were you find installation instructions, configuration settings and other stuff.

### Keep in mind

Every language server is an independent project. A language server can have its own unique configuration method outside of Neovim. They can have bugs, limitations and quirks.

Some language servers were created for VS Code, the fact that Neovim and other editors can use them is just a happy accident. I believe the language servers for `html` and `css` fit in this category. They can have features that only work on VS Code. For example, Neovim can't enable the language server for `css` inside an `html` style tag (see [issue 26783](https://github.com/neovim/neovim/issues/26783)). That probably works fine in VS Code.

## Honorable mention

[nvim-treesitter](https://github.com/nvim-treesitter/nvim-treesitter) is a plugin that has been around for 4 years now. A lot of people will say this is essential to your Neovim experience, and I do agree to some extend. It is incredibly useful... as a dependency for other plugins. It allows Neovim to gather more information about source code of the current file, and plugin authors can do really cool things with that. For "normal users" like you and me there are [some modules](https://github.com/nvim-treesitter/nvim-treesitter?tab=readme-ov-file#available-modules) we can enable. One of them can be used to enhance the syntax highlight of many programming languages. That I think **the** feature `nvim-treesitter` is known for.

There are things you need to be aware of:

* It needs a `C` compiler.
  - Not a problem on linux or mac but it is a problem on windows.
* It only supports the latest stable version of Neovim.
  - It can drop the support for previous versions of Neovim really fast.
* Is considered experimental.
  - It can introduce breaking changes at any point in time.
  - They do use `git tags` to [track versions](https://github.com/nvim-treesitter/nvim-treesitter/tags), in case you need a specific version.

To know more about `tree-sitter` watch this video: [tree-sitter explained (15 min)](https://www.youtube.com/watch?v=09-9LltqWLY).

If you decide to try it, here's how you use it to enable the enhanced syntax highlight.

```lua
require('nvim-treesitter.configs').setup({
  auto_install = true,
  highlight = {
    enable = true,
  },
})
```

## init.lua

And this is it. I believe that's all you need to know to get started. 

```lua
-- Learn about Neovim's lua api
-- https://neovim.io/doc/user/lua-guide.html

vim.o.number = true
vim.o.tabstop = 2
vim.o.shiftwidth = 2
vim.o.smartcase = true
vim.o.ignorecase = true
vim.o.wrap = false
vim.o.hlsearch = false
vim.o.signcolumn = 'yes'

-- Space as the leader key
vim.g.mapleader = vim.keycode('<Space>')

-- Basic clipboard interaction
vim.keymap.set({'n', 'x', 'o'}, 'gy', '"+y', {desc = 'Copy to clipboard'})
vim.keymap.set({'n', 'x', 'o'}, 'gp', '"+p', {desc = 'Paste clipboard text'})

-- Command shortcuts
vim.keymap.set('n', '<leader>w', '<cmd>write<cr>', {desc = 'Save file'})
vim.keymap.set('n', '<leader>q', '<cmd>quitall<cr>', {desc = 'Exit vim'})

vim.cmd.colorscheme('retrobox')

require('mini.completion').setup({})

require('mini.files').setup({})
vim.keymap.set('n', '<leader>e', '<cmd>lua MiniFiles.open()<cr>', {desc = 'File explorer'})

require('mini.pick').setup({})
vim.keymap.set('n', '<leader><space>', '<cmd>Pick buffers<cr>', {desc = 'Search open files'})
vim.keymap.set('n', '<leader>ff', '<cmd>Pick files<cr>', {desc = 'Search all files'})
vim.keymap.set('n', '<leader>fh', '<cmd>Pick help<cr>', {desc = 'Search help tags'})

-- List of compatible language servers is here:
-- https://github.com/neovim/nvim-lspconfig/blob/master/doc/configs.md
require('lspconfig').gopls.setup({})
require('lspconfig').rust_analyzer.setup({})

vim.api.nvim_create_autocmd('LspAttach', {
  desc = 'LSP actions',
  callback = function(event)
    local opts = {buffer = event.buf}

    -- Display documentation of the symbol under the cursor
    vim.keymap.set('n', 'K', '<cmd>lua vim.lsp.buf.hover()<cr>', opts)

    -- Jump to the definition
    vim.keymap.set('n', 'gd', '<cmd>lua vim.lsp.buf.definition()<cr>', opts)

    -- Format current file
    vim.keymap.set({'n', 'x'}, 'gq', '<cmd>lua vim.lsp.buf.format({async = true})<cr>', opts)

    -- Displays a function's signature information
    vim.keymap.set('i', '<C-s>', '<cmd>lua vim.lsp.buf.signature_help()<cr>', opts)

    -- Jump to declaration
    vim.keymap.set('n', '<leader>ld', '<cmd>lua vim.lsp.buf.declaration()<cr>', opts)

    -- Lists all the implementations for the symbol under the cursor
    vim.keymap.set('n', '<leader>li', '<cmd>lua vim.lsp.buf.implementation()<cr>', opts)

    -- Jumps to the definition of the type symbol
    vim.keymap.set('n', '<leader>lt', '<cmd>lua vim.lsp.buf.type_definition()<cr>', opts)

    -- Lists all the references
    vim.keymap.set('n', '<leader>lr', '<cmd>lua vim.lsp.buf.references()<cr>', opts)

    -- Renames all references to the symbol under the cursor
    vim.keymap.set('n', '<leader>ln', '<cmd>lua vim.lsp.buf.rename()<cr>', opts)

    -- Selects a code action available at the current cursor position
    vim.keymap.set('n', '<leader>la', '<cmd>lua vim.lsp.buf.code_action()<cr>', opts)
  end,
})
```

