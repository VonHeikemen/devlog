+++
title = "Simple Neovim config"
description = "Learn the basics of Neovim configuration in lua"
date = 2024-09-12
updated = 2026-01-05
lang = "en"
[taxonomies]
tags = ["vim", "neovim", "shell"]
+++

Can I offer you a nice Neovim configuration in this trying time?

40 lines of code. That's all you need to have a solid starting point. Believe it or not, even those IDE-like features people talk about online can be enabled using just **1 plugin** and a few keymaps.

You can try [this simple config](#init-lua) and then add new things to it based on the problems you encounter along the way. Or just take this as an opportunity to learn about the basics of Neovim configuration.

## Installing Neovim

We are going to need Neovim v0.9 or greater. But be aware a lot of plugins written in lua only guarantee support for the latest stable version. Right now that's v0.11.4 which was released on `August 31, 2025`.

If you are on linux, pay attention to the version your package manager has available.

If you know how to install precompiled executables you can download the latest version from the [github releases](https://github.com/neovim/neovim/releases).

There is also a Neovim version manager called `bob`. You can try it: [MordechaiHadad/bob](https://github.com/MordechaiHadad/bob)

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

Ideally, we would do something like this.

```lua
vim.cmd.colorscheme('retrobox')
```

There is nothing wrong with the example above. That is the correct way to change the theme. However, `retrobox` is a recent a addition to Neovim. If you have Neovim v0.9.5 or lower the command will fail. For the sake of backwards compatibility I will recommend using a "protective call," this way you can setup a backup theme just in case.

```lua
local ok_theme = pcall(vim.cmd.colorscheme, 'retrobox')
if not ok_theme then
  vim.cmd.colorscheme('habamax')
end
```

## Clipboard interaction

This is where we learn about [vim.keymap.set()](https://neovim.io/doc/user/lua-guide.html#_mappings), the function we use to create new keymaps.

By default when you copy some text using the `y` keymap Neovim will store that in a register. The `p` keymap, which we use to paste, also uses the same register as `y`. Basically, Neovim will ignore the system clipboard. We can still use it but we have to be explicit.

Now, I like to keep the default behavior and what I do is create new keymaps to use the system clipboard.

```lua
vim.keymap.set({'n', 'x'}, 'gy', '"+y', {desc = 'Copy to clipboard'})
vim.keymap.set({'n', 'x'}, 'gp', '"+p', {desc = 'Paste clipboard text'})
```

With this `gy` will be the keymap to copy content to the clipboard, and `gp` will paste content from the clipboard to Neovim.

Side note: `{'n', 'x'}` is the list of vim modes where we want our keymap to be created. `n` is normal mode and `x` is visual mode.

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

Yes, that's a string with a blank space. `vim.keycode` was added in Neovim v0.10, so before that people just did that thing with the blank space. So if you have it, then you can make it more explicit.

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

So Neovim is very close to have [its own plugin manager](https://neovim.io/doc/user/pack.html#_plugin-manager). But, it will take a few years before is included in a stable release available in every operating system. Right now only those who download Neovim's "nightly version" will be able to use it in its current state.

Interestingly enough, Neovim's plugin manager is based on a plugin called [mini.deps](https://nvim-mini.org/mini.nvim/doc/mini-deps.html). So while the Neovim team works on the new plugin manager, we can use `mini.deps`. In the future it should be fairly simple to migrate from `mini.deps` to the native solution.

The question is, *how does one install a plugin without a plugin manager?*

There is a feature in **Vim** called [packages](@/tools/installing-neovim-plugins-without-a-plugin-manager.md). Long story short, we can put plugins in a specific location in they should work.

Let's use Neovim in headless mode again, this time to create the directory were `mini.deps` should be.

```sh
nvim --headless -c 'call mkdir(stdpath("data") . "/site/pack/deps/start/", "p")' -c 'quit'
```

This other command will show the full path of the directory we just created.

```sh
nvim --headless -c 'echo stdpath("data") . "/site/pack/deps/start/"' -c 'quit'
```

Now navigate to that location using your terminal. You will want to execute the command `git clone` inside that directory.

* Install [mini.nvim](https://github.com/nvim-mini/mini.nvim) 

`mini.deps` is part of a project called `mini.nvim`, which is a collection of lua modules aimed to solve common problems and enhance some of Neovim builtin features. So we want to download `mini.nvim` using this command.

```
git clone --filter=blob:none https://github.com/nvim-mini/mini.nvim
```

After the install is complete we should generate the help tags.

```sh
nvim --headless -c 'helptags ALL' -c 'quit'
```

To use the plugin we must load the module `mini.deps`. But here we want to use in a way that is "safe." Because what if we copy our configuration to a new machine but we forget to download `mini.nvim`? If we do it the normal way we get hit by a scary wall of red text. We can make the experience less jarring by adding a few lines code.

```lua
local ok, MiniDeps = pcall(require, 'mini.deps')
if not ok then
  vim.notify('[WARN] mini.deps module not found', vim.log.levels.WARN)
  return
end
```

Here if something goes wrong Neovim will output a single line message then it will stop the execution of the script.

If everything goes according to plan every function of the `mini.deps` module will be inside the variable `MiniDeps`.

Is important to note all features of `mini.nvim` are opt-in, meaning we have to do something in our configuration to enable a module. For `mini.deps` we have to use a function called `.setup()`. So we add this after we load the module.

```lua
MiniDeps.setup({})
```

* Install [nvim-lspconfig](https://github.com/neovim/nvim-lspconfig)

We will use this plugin to help us enable IDE-like features.

At this point we have `mini.deps` available so is easier to install a new plugin. This is the minimum amount of code we need.

```lua
MiniDeps.add('neovim/nvim-lspconfig')
```

The `.add()` function will make sure the plugin is installed and loaded. And is also the reason I recommend using `mini.deps` while we wait for Neovim's plugin manager. In the future we will be able to add plugins like this:

```lua
vim.pack.add({'https://github.com/neovim/nvim-lspconfig'})
```

There are some sutle differences between `mini.deps` and `vim.pack` but for the most part those two functions do the same thing.

For now let's focus on `mini.deps` because that's the thing we are going to use.

The moment we need to add more information about the plugin we want to download we should replace the string with a lua table. For example, let's say we want to download a previous version of `nvim-lspconfig`. This is what we do.

```lua
MiniDeps.add({
  source = 'neovim/nvim-lspconfig',
  checkout = 'v1.8.0'
})
```

Now the first argument to `.add()` is a lua table with some properties. `source` would be the link to the plugin. In `mini.deps` github links get a special treatment because is very popular, so here we can just specify the github user and the name of the repository. `checkout` can be a commit hash, tag or branch name. In this case [v1.8.0](https://github.com/neovim/nvim-lspconfig/releases/tag/v1.8.0) is a tag on the `nvim-lspconfig` repository, it is the last version that supports **Neovim v0.9**.

To know more details about `mini.deps` read the official documentation: [mini.nvim/doc/mini-deps.html](https://nvim-mini.org/mini.nvim/doc/mini-deps.html).

## File explorer

You should know that Neovim already has a file explorer, [it's called netrw](@/tools/using-netrw-vim-builtin-file-explorer.md). We are not going to use it here because it has a few quirks I dislike. Still, I think you should know it exists.

Since we have `mini.nvim` available let's use the file explorer it provides: [mini.files](https://github.com/nvim-mini/mini.nvim/blob/main/readmes/mini-files.md).

First, we call `mini.files` setup function.

```lua
require('mini.files').setup({})
```

This is a common convention of lua plugins. Sometimes **you have to** execute a function called `.setup()` to enable the plugin. Notice I said is a convention, it is not a rule. Some plugins execute their setup automatically.

How do we use `mini.files`?

We make a keymap to open the file explorer.

```lua
vim.keymap.set('n', '<leader>e', '<cmd>lua MiniFiles.open()<cr>', {desc = 'File explorer'})
```

After we open the file explorer press `g` then `?`, this will show the keymaps available.

What if we want to change the settings in `mini.files`?

Any custom setting should be inside the `{}` of the setup function. Here's an example.

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

That took me to [the help page of mini.files](https://github.com/nvim-mini/mini.nvim/blob/main/doc/mini-files.txt).

Remember when we installed `mini.nvim`? We generated some called help tags. These tags allow you to navigate to the "offline" documentation of the plugin: the help page.

The tab complete feature of the `:help` command is really good too. You can type `:help mini` and then press the `Tab` key, Neovim will show you all the help tags that start with `mini`.

One more thing, if the font on your terminal doesn't support "icons" then you can add the [mini.icons](https://github.com/nvim-mini/mini.nvim/blob/main/readmes/mini-icons.md) module. Set the `style` property to `ascii`.

```lua
require('mini.icons').setup({style = 'ascii'})
```

## Navigating between files

For educational purposes only, I will tell you what I do when I don't have plugins installed: I use the `files` command to list the open files, then use the `buffer` command to navigate the file I want. And to make it convenient I make a keymap.

```lua
vim.keymap.set('n', '<leader><space>', '<cmd>files<cr>:buffer ', {desc = 'Search open files'})
```

In the `:buffer` command you can tab complete the name of the file or use the "id" of the buffer.

But why do that when we have `mini.nvim` installed? We can use the module [mini.pick](https://github.com/nvim-mini/mini.nvim/blob/main/readmes/mini-pick.md). This one provides a fancy interactive filter. And it also gives us commands we can use to search files.

If you use the module [mini.extra](https://github.com/nvim-mini/mini.nvim/blob/main/readmes/mini-extra.md) you will have access to more search commands in `mini.pick`.

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

For this we will use [mini.completion](https://github.com/nvim-mini/mini.nvim/blob/main/readmes/mini-completion.md) and [mini.snippets](https://github.com/nvim-mini/mini.nvim/blob/main/readmes/mini-snippets.md).

```lua
require('mini.snippets').setup({})
require('mini.completion').setup({})
```

`mini.completion` is kind of a wrapper around Neovim's builtin completion mechanism. Neovim makes you press specific keybindings depending on the type of completion you want to get. And what `mini.completion` can do is take that burden away from you, by triggering the completion automatically as you type. In other words, it adds true autocompletion to Neovim.

By default you just get suggestions based on words of the current file. But whenever possible it'll try to use the LSP client to provide smart code completion.

To control the completion menu we use Neovim's default keybindings:

* `<Down>`: Select the next item on the list.

* `<Up>`:  Select previous item on the list.

* `<Ctrl-n>`: Select and insert text of the next item on the list.

* `<Ctrl-p>`: Select and insert text of the previous item on the list.

* `<Ctrl-y>`: Confirm selected item.

* `<Ctrl-e>`: Cancel the completion.

* `<Enter>`: If item was selected using `<Up>` or `<Down>` it confirms selection. If no item is selected, hides completion menu. Else, inserts a newline character.

Unfortunately, Neovim's builtin completion mechanism does not offer any support for snippets by default. That's why we have `mini.snippets`. But to be fair, newer versions of Neovim do have some support for snippets... but even with that, it still doesn't offer as many features as `mini.snippets`.

## The LSP client

Neovim's LSP client is the thing that enables the nice IDE-like features. Things like go to definition, rename variable, inspect function signature. You get the idea. But this "client" doesn't work on its own, it needs a server. In this case a "server" is an external program that implements the [LSP specification](https://microsoft.github.io/language-server-protocol/overviews/lsp/overview/). If you want to know more details about LSP and language servers, watch this video by TJ DeVries: [LSP explained (5 min)](https://www.youtube.com/watch?v=LaS32vctfOY).

What do we need to do here? We follow these 3 steps.

* Install a language server
* Configure that language server
* Create some keymaps (optional)

### Step 1: Install a language server

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

### Step 2: Configure a language server

Now you have to "enable" each language server you have installed in your system.

On **Neovim v0.11** you can use the function [vim.lsp.enable()](https://neovim.io/doc/user/lsp.html#vim.lsp.enable()).

```lua
vim.lsp.enable('example_server')
```

On **Neovim v0.10** or lower you must use the "legacy setup" function of `nvim-lspconfig`.

```lua
require('lspconfig').example_server.setup({})
```

If you have the language server for `go` and `rust` you would do this.

```lua
vim.lsp.enable({'gopls', 'rust_analyzer'})
```

Or this.

```lua
require('lspconfig').gopls.setup({})
require('lspconfig').rust_analyzer.setup({})
```

If you want your config to be backwards compatible you can write a function that uses the correct method.

```lua
-- This function will use the "legacy setup" on older Neovim version.
-- The new api is only available on Neovim v0.11 or greater.
local function lsp_setup(server, opts)
  if vim.fn.has('nvim-0.11') == 0 then
    require('lspconfig')[server].setup(opts)
    return
  end

  if not vim.tbl_isempty(opts) then
    vim.lsp.config(server, opts)
  end

  vim.lsp.enable(server)
end
```

Then you can write the setup like this:

```lua
lsp_setup('gopls', {})
lsp_setup('rust_analyzer', {})
```

Do note there are a handful of language servers that can't be configured with `vim.lsp.enable()`. The list of missing servers is in this github issue: [nvim-lspconfig/issues/3705](https://github.com/neovim/nvim-lspconfig/issues/3705).

### Step 3: Create keymaps

After a language server is active in the current buffer we can use the functions under the module [vim.lsp.buf](https://neovim.io/doc/user/lsp.html#lsp-buf). On **Neovim v0.11** we have a handful of keymaps that trigger functions on this module. But on older versions we have to create the keymaps ourselves.

```lua
-- NOTE: These keymaps are already part of Neovim v0.11.
-- You do not have to add them to your personal configuration
-- if you are using Neovim v0.11 or greater.

-- Displays a function's signature information
vim.keymap.set('i', '<C-s>', '<cmd>lua vim.lsp.buf.signature_help()<cr>')

-- Lists all symbols in the current buffer
vim.keymap.set('n', 'gO', '<cmd>lua vim.lsp.buf.document_symbol()<cr>')

-- Lists all the implementations for the symbol under the cursor
vim.keymap.set('n', 'gri', '<cmd>lua vim.lsp.buf.implementation()<cr>')

-- Jumps to the definition of the type symbol
vim.keymap.set('n', 'grt', '<cmd>lua vim.lsp.buf.type_definition()<cr>')

-- Lists all the references
vim.keymap.set('n', 'grr', '<cmd>lua vim.lsp.buf.references()<cr>')

-- Renames all references to the symbol under the cursor
vim.keymap.set('n', 'grn', '<cmd>lua vim.lsp.buf.rename()<cr>')

-- Selects a code action available at the current cursor position
vim.keymap.set('n', 'gra', '<cmd>lua vim.lsp.buf.code_action()<cr>')

-- Show diagnostics in a floating window
vim.keymap.set('n', '<C-w>d', '<cmd>lua vim.diagnostic.open_float()<cr>')
vim.keymap.set('n', '<C-w><C-d>', '<cmd>lua vim.diagnostic.open_float()<cr>')

-- Jump next/previous diagnostic in the current buffer
vim.keymap.set('n', '[d', '<cmd>lua vim.diagnostic.goto_prev()<cr>')
vim.keymap.set('n', ']d', '<cmd>lua vim.diagnostic.goto_next()<cr>')

vim.api.nvim_create_autocmd('LspAttach', {
  callback = function(event)
    -- NOTE: By default `K` uses the program specified in 'keywordprg'
    -- Here we force it to use the LSP client

    -- Display documentation of the symbol under the cursor
    vim.keymap.set('n', 'K', '<cmd>lua vim.lsp.buf.hover()<cr>', {buffer = event.buf})
  end,
})
```

### Keep in mind

Every language server is an independent project. A language server can have its own unique configuration method outside of Neovim. They can have bugs, limitations and quirks.

Some language servers were created for VS Code, the fact that Neovim and other editors can use them is just a happy accident. I believe the language servers for `html` and `css` fit in this category. They can have features that only work on VS Code. For example, Neovim can't enable the language server for `css` inside an `html` style tag (see [issue 26783](https://github.com/neovim/neovim/issues/26783)). That probably works fine in VS Code.

## Honorable mention

[nvim-treesitter](https://github.com/nvim-treesitter/nvim-treesitter) is a plugin that has been around since 2020. A lot of people will say it's essential to your Neovim experience, and I do agree to some extend. It is incredibly useful... as a dependency for other plugins. It allows Neovim to gather more information about source code of the current file, and plugin authors can do really cool things with that. Treesitter became known in the community because it could be used to enhance the syntax highlight of many programming languages.

There are things you need to be aware of:

* It needs a `C` compiler.
  - Not a problem on linux or mac but it is a problem on windows.
* It only supports the latest stable version of Neovim.
  - It can drop the support for previous versions of Neovim really fast.
* Is considered experimental.
  - It can introduce breaking changes at any point in time.
  - They do use `git tags` to [track versions](https://github.com/nvim-treesitter/nvim-treesitter/tags), in case you need a specific version.

To know more about `tree-sitter` watch this video: [tree-sitter explained (15 min)](https://www.youtube.com/watch?v=09-9LltqWLY).

If you decide to try it, you can use the command `:TSInstall` to download a treesitter parser for the language you want, and then you can enable the enhanced syntax highlight using an autocommand.

For example, you can download the treesitter parser for javascript using this command.

```vim
:TSInstall javascript
```

And then create an autocommand that will call the function `vim.treesitter.start()` in any filetype that uses javascript syntax. Like this.

```lua
vim.api.nvim_create_autocmd('FileType', {
  pattern = {'javascript', 'javascriptreact', 'js', 'jsx'},
  callback = function()
    vim.treesitter.start()
  end,
})
```

## init.lua

And this is it. This is all you need to get started if you are using Neovim v0.11. If you need backwards compatibility with Neovim v0.9, [use this other configuration instead](#backwards-compatible-config).

```lua
-- NOTE: This configuration is meant for Neovim v0.11 or greater

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
vim.keymap.set({'n', 'x'}, 'gy', '"+y', {desc = 'Copy to clipboard'})
vim.keymap.set({'n', 'x'}, 'gp', '"+p', {desc = 'Paste clipboard text'})

-- Command shortcuts
vim.keymap.set('n', '<leader>w', '<cmd>write<cr>', {desc = 'Save file'})
vim.keymap.set('n', '<leader>q', '<cmd>quitall<cr>', {desc = 'Exit vim'})

vim.cmd.colorscheme('retrobox')

local ok, MiniDeps = pcall(require, 'mini.deps')
if not ok then
  vim.notify('[WARN] mini.deps module not found', vim.log.levels.WARN)
  return
end

MiniDeps.setup({})
MiniDeps.add('neovim/nvim-lspconfig')

require('mini.snippets').setup({})
require('mini.completion').setup({})

require('mini.files').setup({})
vim.keymap.set('n', '<leader>e', '<cmd>lua MiniFiles.open()<cr>', {desc = 'File explorer'})

require('mini.pick').setup({})
vim.keymap.set('n', '<leader><space>', '<cmd>Pick buffers<cr>', {desc = 'Search open files'})
vim.keymap.set('n', '<leader>ff', '<cmd>Pick files<cr>', {desc = 'Search all files'})
vim.keymap.set('n', '<leader>fh', '<cmd>Pick help<cr>', {desc = 'Search help tags'})

-- List of compatible language servers is here:
-- https://github.com/neovim/nvim-lspconfig/blob/master/doc/configs.md
vim.lsp.enable({'gopls', 'rust_analyzer'})
```

## Backwards compatible config

For those of you that are using Ubuntu or Debian based operating systems, you have to be aware of the version available in the official repo of your system. In Ubuntu 24.04 you'll have `v0.9.5`. And any system based on Debian 13 will have `v0.10.4` available.

You can still have a good experience using older versions of Neovim but the configuration will be a little bit more complex.

```lua
-- NOTE: This is meant to be backwards compatible with Neovim v0.9.5

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
vim.g.mapleader = ' '

-- Basic clipboard interaction
vim.keymap.set({'n', 'x'}, 'gy', '"+y', {desc = 'Copy to clipboard'})
vim.keymap.set({'n', 'x'}, 'gp', '"+p', {desc = 'Paste clipboard text'})

-- Command shortcuts
vim.keymap.set('n', '<leader>w', '<cmd>write<cr>', {desc = 'Save file'})
vim.keymap.set('n', '<leader>q', '<cmd>quitall<cr>', {desc = 'Exit vim'})

local ok_theme = pcall(vim.cmd.colorscheme, 'retrobox')
if not ok_theme then
  vim.cmd.colorscheme('habamax')
end

local ok, MiniDeps = pcall(require, 'mini.deps')
if not ok then
  vim.notify('[WARN] mini.deps module not found', vim.log.levels.WARN)
  return
end

MiniDeps.setup({})

if vim.fn.has('nvim-0.11') == 1 then
  MiniDeps.add('neovim/nvim-lspconfig')
else
  MiniDeps.add({
    source = 'neovim/nvim-lspconfig',
    checkout = 'v1.8.0'
  })
end

require('mini.snippets').setup({})
require('mini.completion').setup({})

require('mini.files').setup({})
vim.keymap.set('n', '<leader>e', '<cmd>lua MiniFiles.open()<cr>', {desc = 'File explorer'})

require('mini.pick').setup({})
vim.keymap.set('n', '<leader><space>', '<cmd>Pick buffers<cr>', {desc = 'Search open files'})
vim.keymap.set('n', '<leader>ff', '<cmd>Pick files<cr>', {desc = 'Search all files'})
vim.keymap.set('n', '<leader>fh', '<cmd>Pick help<cr>', {desc = 'Search help tags'})

-- Displays a function's signature information
vim.keymap.set('i', '<C-s>', '<cmd>lua vim.lsp.buf.signature_help()<cr>')

-- Lists all symbols in the current buffer
vim.keymap.set('n', 'gO', '<cmd>lua vim.lsp.buf.document_symbol()<cr>')

-- Lists all the implementations for the symbol under the cursor
vim.keymap.set('n', 'gri', '<cmd>lua vim.lsp.buf.implementation()<cr>')

-- Jumps to the definition of the type symbol
vim.keymap.set('n', 'grt', '<cmd>lua vim.lsp.buf.type_definition()<cr>')

-- Lists all the references
vim.keymap.set('n', 'grr', '<cmd>lua vim.lsp.buf.references()<cr>')

-- Renames all references to the symbol under the cursor
vim.keymap.set('n', 'grn', '<cmd>lua vim.lsp.buf.rename()<cr>')

-- Selects a code action available at the current cursor position
vim.keymap.set('n', 'gra', '<cmd>lua vim.lsp.buf.code_action()<cr>')

-- Show diagnostics in a floating window
vim.keymap.set('n', '<C-w>d', '<cmd>lua vim.diagnostic.open_float()<cr>')
vim.keymap.set('n', '<C-w><C-d>', '<cmd>lua vim.diagnostic.open_float()<cr>')

if vim.fn.has('nvim-0.11') == 0 then
  -- Jump next/previous diagnostic in the current buffer
  vim.keymap.set('n', '[d', '<cmd>lua vim.diagnostic.goto_prev()<cr>')
  vim.keymap.set('n', ']d', '<cmd>lua vim.diagnostic.goto_next()<cr>')
end

vim.api.nvim_create_autocmd('LspAttach', {
  callback = function(event)
    -- NOTE: By default `K` uses the program specified in the 'keywordprg' option.
    -- Here we override the default behavior, we force it to use the LSP client

    -- Display documentation of the symbol under the cursor
    vim.keymap.set('n', 'K', '<cmd>lua vim.lsp.buf.hover()<cr>', {buffer = event.buf})
  end,
})

-- This function will use the "legacy setup" on older Neovim version.
-- The new api is only available on Neovim v0.11 or greater.
local function lsp_setup(server, opts)
  if vim.fn.has('nvim-0.11') == 0 then
    require('lspconfig')[server].setup(opts)
    return
  end

  if not vim.tbl_isempty(opts) then
    vim.lsp.config(server, opts)
  end

  vim.lsp.enable(server)
end

-- List of compatible language servers is here:
-- https://github.com/neovim/nvim-lspconfig/blob/master/doc/configs.md
lsp_setup('gopls', {})
lsp_setup('rust_analyzer', {})
```

