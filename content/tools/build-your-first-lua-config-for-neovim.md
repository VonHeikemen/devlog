+++
title = "Build your first Neovim configuration in lua"
description = "The one where we learn how to customize Neovim and add plugins"
date = 2022-07-04
lang = "en"
[taxonomies]
tags = ["vim", "neovim", "shell"]
[extra]
shared = [
  ["dev.to", "https://dev.to/vonheikemen/build-your-first-neovim-configuration-in-lua-177b"],
]
+++

Neovim is a tool both powerful and extensible. With some effort it can do more than just modify text in a file. Today I hope I can teach you enough about Neovim's `lua` api to be able to build your configuration.

We will create a simple configuration file called `init.lua`, add a couple of plugins and I'll tell you how to make your own commands.

This tutorial is meant for people totally new to Neovim. If you already have a configuration written in vimscript and want to migrate to lua, you might find this other article more useful: [Everything you need to know to configure neovim using lua](@/tools/configuring-neovim-using-lua.md).

## Some advice

Before we start, I suggest you install the latest stable version of Neovim. You can go to the [release page](https://github.com/neovim/neovim/releases) in the github repository and download it from there. From now on I will assume you are using version 0.7.

If you don't feel comfortable using Neovim as an editor, follow the tutorial that comes bundled with it. You can start it using this command in the terminal.

```sh
nvim +Tutor
```

I will assume you know all the features `Tutor` teaches.

## The entry point

First things first, we need to create a configuration file, the famous `init.lua`. And where that might be? Well, it depends on your operating system and also your environment variables. I can tell you a way of creating it using Neovim, that way we don't have to worry about those details.

> Fun fact: Some articles online call the configuration file `vimrc`. That is the name it has in Vim.

For this task we won't be using lua, we'll use the language created specifically for Vim: vimscript.

Let's open Neovim and execute this command.

```vim
:call mkdir(stdpath("config"), "p")
```

It'll create the folder where the configuration file needs to be. If you want to know what folder it created use this.

```vim
:echo stdpath("config")
```

Now we are going to edit the configuration file.

```vim
:exe "edit" stdpath("config") . "/init.lua"
```

After doing that we'll be in a "blank page". At this point the file doesn't exists on the system just yet. We need to save it with this command.

```vim
:write
```

Once the file actually exists we can edit it anytime we want using this.

```vim
:edit $MYVIMRC
```

If you are the kind of person that likes to automate things in scripts you'll be happy to know you can do all of it with one command.

```vim
nvim --headless -c 'call mkdir(stdpath("config"), "p") | exe "edit" stdpath("config") . "/init.lua" | write | quit'
```

## Editor settings

To access the editor's setting we need to use the global variable `vim`. Okay, more than a variable this thing is a module, you'll find all sorts of utilities in there. Right now we are going to focus on the `opt` property, with it we can modify all 351 options Neovim has (in version 0.7).

This is the syntax you should follow.

```lua
vim.opt.option_name = value
```

Where `option_name` can be anything in [this list](https://neovim.io/doc/user/quickref.html#option-list). And `value` must be whatever that option expects.

> You can see the list in Neovim using `:help option-list`.

One thing you should know is that every option has a scope. Some options are global, some only apply in the current window or file. The scope of every option is mentioned in their help page. To navigate to the help page of an option follow this pattern.

```vim
:help 'option_name'
```

### Useful options

* `number`

This option expects a boolean value. This means it can only have two possible values: `true` or `false`. If we assign `true` we enable it, `false` does the opposite.

When we enable `number` Neovim starts showing the line number in the gutter.

```lua
vim.opt.number = true
```

* `mouse`

Neovim (and Vim) can let you use the mouse for some things, like select text or change the size of window. `mouse` expects a data type called a string (a piece of text wrapped in quotes) with a combination of modes. We are not going to worry about those modes now, we can just enable it for every mode.

```lua
vim.opt.mouse = 'a'
```

* `ignorecase`

With this we can tell Neovim to ignore uppercase letters when executing a search. For example, if we search the word `two` the results can contain any variations like `Two`, `tWo` or `two`.

```lua
vim.opt.ignorecase = true
```

* `smartcase`

Makes our search ignore uppercase letters unless the search term has an uppercase letter. Most of the time this is used in combination with `ignorecase`.

```lua
vim.opt.smartcase = true
```

* `hlsearch`

Highlights the results of the previous search. It can get annoying really fast, this is how we disable it.

```lua
vim.opt.hlsearch = false
```

* `wrap`

Makes the text of long lines always visible. Long lines are those that exceed the width of the screen. The default value is `true`.

```lua
vim.opt.wrap = true
```

* `breakindent`

Preserve the indentation of a virtual line. These "virtual lines" are the ones only visible when `wrap` is enabled.

```lua
vim.opt.breakindent = true
```

* `tabstop`

The amount of space on screen a `Tab` character can occupy. The default value is 8. I think 2 is fine.

```lua
vim.opt.tabstop = 2
```

* `shiftwidth`

Amount of characters Neovim will use to indent a line. This option influences the keybindings `<<` and `>>`. The default value is 8. Most of the time we want to set this with same value as `tabstop`.

```lua
vim.opt.shiftwidth = 2
```

* `expandtab`

Controls whether or not Neovim should transform a `Tab` character to spaces. The default value is `false`.

```lua
vim.opt.expandtab = false
```

There are a few other things in the `vim` module we can use to modify variables, but we have other things to do right now. I talk about this topic in more detail here: [Configuring Neovim - Editor Settings](@/tools/configuring-neovim-using-lua.md#editor-settings).

## Keybindings

Because Neovim clearly doesn't have enough, we need to create more. To do it we need to learn about `vim.keymap.set`. Here is a basic usage example.

```lua
vim.keymap.set('n', '<space>w', '<cmd>write<cr>', {desc = 'Save'})
```

After executing this, the sequence `Space` + `w` will call the `write` command. Basically, we can save changes made to a file with `Space` + `w`.

Now let me explain `vim.keymap.set` parameters.

```lua
vim.keymap.set({mode}, {lhs}, {rhs}, {opts})
```

* `{mode}` mode where the keybinding should execute. It can be a list of modes. We need to specify the mode's short name. Here are some of the most common.

  - `n`: Normal mode.
  - `i`: Insert mode.
  - `x`: Visual mode.
  - `s`: Selection mode.
  - `v`: Visual + Selection.
  - `t`: Terminal mode.
  - `o`: Operator-pending.
  - `''`: Yes, an empty string. Is the equivalent of `n` + `v` + `o`.

* `{lhs}` is the key we want to bind.

* `{rhs}` is the action we want to execute. It can be a string with a command or an expression. You can also provide a lua function.

* `{opts}` this must be a lua table. If you don't know what is a "lua table" just think is a way of storing several values in one place. Anyway, it can have these properties.

  - `desc`: A string that describes what the keybinding does. You can write anything you want.
  
  - `remap`: A boolean that determines if our keybinding can be recursive. The default value is `false`. Recursive keybindings can cause some conflicts if used incorrectly. Don't enable it unless you know what you're doing. I will explain this recursive thing later.
  
  - `buffer`: It can be a boolean or a number. If we assign the boolean `true` it means the keybinding will only be effective in the current file. If we assign a number, it needs to be the "id" of an open buffer.

  - `silent`: A boolean. Determines whether or not the keybindings can show a message. The default value is `false`.

  - `expr`: A boolean. If enabled it gives the chance to use vimscript or lua to calculate the value of `{rhs}`. The default value is `false`.

### The leader key

When creating keybindings we can use the special sequence `<leader>` in the `{lhs}` parameter, it'll take the value of the global variable `mapleader`.

So `mapleader` is a global variable in vimscript that can be string. For example.

```lua
vim.g.mapleader = ','
```

After defining it we can use it as a prefix in our keybindings.

```lua
vim.keymap.set('n', '<leader>w', '<cmd>write<cr>')
```

This will make `,` + `w` save the current file.

What happens if we don't define it? The default value is `\`, clearly not the best choice. I can recommend using the space key as leader. Like this.

```lua
vim.g.mapleader = ' '
```

### Mappings

I'll show you just a few keybindings that you might find useful.

* Copy/paste from clipboard

The default behavior in Neovim (and Vim) doesn't take into account the system clipboard. It has its own mechanism to store text. When we copy something using the `y` keybinding that text goes to an internal register. I prefer to keep it that way, and what I do is create dedicated bindings to interact with the clipboard.

Copy to clipboard.

```lua
vim.keymap.set({'n', 'x'}, 'cp', '"+y')
```

Paste from clipboard.

```lua
vim.keymap.set({'n', 'x'}, 'cv', '"+p')
```

* Delete without changing the registers

When we delete text in normal mode or visual mode using `c`, `d` or `x` that text goes to a register. This affects the text we paste with the keybinding `p`. What I do is modify `x` to delete text without changing the internal registers.

```lua
vim.keymap.set({'n', 'x'}, 'x', '"_x')
```

* Select all text in current buffer

```lua
vim.keymap.set('n', '<leader>a', ':keepjumps normal! ggVG<cr>')
```

## Plugin manager

Right now the most popular plugin manager is [packer.nvim](https://github.com/wbthomason/packer.nvim). I'll show some basic usage examples.

First step is to install it from github. It just so happens we can do this using lua. In packer's documentation they show us how to do it. We add this to our config.

```lua
local install_path = vim.fn.stdpath('data') .. '/site/pack/packer/start/packer.nvim'
local install_plugins = false

if vim.fn.empty(vim.fn.glob(install_path)) > 0 then
  print('Installing packer...')
  local packer_url = 'https://github.com/wbthomason/packer.nvim'
  vim.fn.system({'git', 'clone', '--depth', '1', packer_url, install_path})
  print('Done.')

  vim.cmd('packadd packer.nvim')
  install_plugins = true
end
```

Pay attention to `install_path`, this variable controls where we install packer. But more than that it also shows where all our plugins will be installed, you'll want to know where that is. Execute this command.

```vim
:echo stdpath('data') . '/site/pack/packer'
```

If you ever have a problem with the install (or removal) of a plugin, check that folder. Inside you'll find two folders: `opt` and `start`. In the `opt` folder you'll find "optional plugins" and the `start` folder will have all the plugins that Neovim will load at startup. Packer by default install all plugins in the `start` folder.

If you wish to customize the install path of plugins: First read the help page of "packages" `:help packages`. Then read about [packer initialization options](https://github.com/wbthomason/packer.nvim#custom-initialization).

Now, to actually use packer we can follow this pattern.

```lua
require('packer').startup(function(use)

  use 'github-user/repo'

  if install_plugins then
    require('packer').sync()
  end
end)
```

The `.startup` function in packer takes another function as an argument. Inside that function we can list our plugins. We call the function `use` to tell packer what plugins we want. Default behavior of packer makes it easy to download plugins from github, we just have to specify the user and the name of the repository.

That little `if` section we have at the bottom is the one that downloads plugins for the first time. If `install_plugins` has the value `true` packer will do its thing.

If your entire configuration is in one file, meaning you don't have any extra modules, I suggest you stop the execution of the script while plugins are being downloaded the first time. Like this.

```lua
require('packer').startup(function(use)
  ---
  -- List of plugins...
  ---

  if install_plugins then
    require('packer').sync()
  end
end)

if install_plugins then
  return
end
```

Notice the second `if` block has the keyword `return`, that will stop the script and save you the trouble of Neovim trying to configure plugins that aren't installed.

Now let's download a plugin, a colorscheme to make Neovim look better.

```lua
require('packer').startup(function(use)
  -- Package manager
  use 'wbthomason/packer.nvim'

  -- Theme inspired by Atom
  use 'joshdick/onedark.vim'

  if install_plugins then
    require('packer').sync()
  end
end)

if install_plugins then
  return
end
```

> Fun fact: packer can update itself if we add it to the list of plugins.

Here we add `'joshdick/onedark.vim'` to the list. But we are not going to trigger the install process yet, let's add the code to apply the theme first. Add the end of the file put this.

```lua
vim.opt.termguicolors = true

vim.cmd('colorscheme onedark')
```

We enable `termguicolors` so Neovim can show the "best" version of the colorscheme. Each colorscheme can have two versions: one that works for terminals which only support 256 colors and another that specifies colors in hexadecimal codes (has way more colors).

We tell Neovim which theme we want by using the `colorscheme` command. Notice that we are using `vim.cmd`, that allows us to use vimscript inside lua. But why? We still don't have a lua api to apply a theme.

We save these changes and restart Neovim, to trigger the download of packer. When Neovim start it should show a message telling us is cloning packer's repository. After it's done and packer is installed another window will show up, it'll tell us the progress of the plugins download. After plugins are installed we should restart Neovim again to see the changes.

## Plugin configuration

Each plugin author has the freedom to create the configuration method they want. But then how do we know what to do? We have to rely on the documentation the plugin provides, we have no other choice.

Most plugins have a file called `README.md` in their repository, github is kind enough to render that file in the main page. It's the main place you should look for configuration instructions.

If for some reason the README doesn't have the information we are after, look for a folder called `doc`. Inside there should be a `txt` file, this is the help page. We can read it using Neovim executing the command `:help name-of-file.txt`.

### Conventions of lua plugins

Lucky for us a huge amount of plugins written in lua follow a certain pattern. They use a function called `.setup`, and that functions expects a lua table with some options. If there is something you should learn about the syntax of lua is how to create tables.

Let's configure a plugin. For this example I'll use [lualine](https://github.com/nvim-lualine/lualine.nvim), a plugin that can give us a good looking statusline. First step, add it to the list of plugins.

```lua
use 'nvim-lualine/lualine.nvim'
```

Now save the changes, then execute the command to reload the config and install the plugins.

```vim
:source $MYVIMRC | PackerSync
```

After that you can add the configuration.

In lualine's repository we can find a `doc` folder and inside there is a file called `lualine.txt`. We can read it in Neovim using this.

```vim
:help lualine.txt
```

This documentation  shows we can make the plugin work just by calling `setup` on the lualine module. Like this.

```lua
require('lualine').setup()
```

But now lets pretend we want to change some options. For example, say we hate icons, we want them gone. In the documentation there is a section called `lualine-Default-configuration`, in there I can see some code that says `icon_enabled = true`. Great, let's copy all the necessary properties to modify that.

```lua
require('lualine').setup({
  options = {
    icons_enabled = false,
  }
})
```

Looks better already. Now we don't like "component separators", how do we get rid of them? Check the docs again, there is a section `lualine-Disabling-separators`, it shows this.

```lua
options = { section_separators = '', component_separators = '' }
```

Looks promising but we should not copy/paste that code as is. We need to really read it. It shows an `options` property, we already have one of those, what we should do is add these new properties to the thing we have.

```lua
require('lualine').setup({
  options = {
    icons_enabled = false,
    section_separators = '',
    component_separators = ''
  }
})
```

So... what did we learned? To configure some plugins we need know: how to navigate to the documentation and the syntax used to create lua tables.

### Vimscript plugins

There are a lot of useful plugins written in vimscript. Most of them we can configure modifying global variables. In lua we change global variables of vimscript using `vim.g`.

Did you know Neovim has a file explorer? Yeah, it is plugin that comes bundled in Neovim. We can use it with the command `:Lexplore`. It is written in vimscript, so there is no `.setup` function. To know how to configure it we need to check the documentation.

```vim
:help netrw
```

If you check the table of content in the help page, you'll notice a section called `netrw-browser-settings`. Once there we get a list of variables and their descriptions. Let's focus on the ones that start with `g:`.

For example, if we want to hide the help text in the banner we use `netrw_banner`.

```lua
vim.g.netrw_banner = 0
```

Another thing we can do is change the size of the window. For that we need `netrw_winsize`.

```lua
vim.g.netrw_winsize = 30
```

That's it... well no, there are more variables, but like this is the basic stuff you need to know. You check the docs, see the thing you want to change and use `vim.g`.

## Bonus content

### Recursive mappings

If you are familiar with the word "recursive" you might be able to guess what kind of consequences they can have. If not, let me try to explain with an example.

Let's say we have a keybinding that opens the file explorer.

```lua
vim.keymap.set('n', '<F2>', '<cmd>Lexplore<cr>')
```

Now let's add a recursive keybinding that uses `F2`.

```lua
vim.keymap.set('n', '<space><space>', '<F2>', {remap = true})
```

If we press `Space` twice the file explorer will show up. But if we change `remap` to `false` then nothing happens.

With recursive mappings we can use previous keybindings in the `{rhs}` argument. Those keybindings could be created by us or by other plugins. Non recursive mappings only give us access to keybindings defined by Neovim.

Probably the only time when you want a recursive mapping is when you want to use a feature defined by a plugin.

Why is it that recursive mappings can cause conflicts? Consider this.

```lua
vim.keymap.set('n', '*', '*zz')
```

Notice here we are using `*` in `{lhs}` and also `{rhs}`. If we make this recursive we create an endless cycle. Neovim will try to call `*` and never executes `zz`.

### User commands

Yes, we can create our own commands. In lua we use this function.

```lua
vim.api.nvim_create_user_command({name}, {command}, {opts})
```

* `{name}` must be a string. It has to start with an uppercase letter.

* `{command}` if it's a string it must be valid vimscript. Or it can be a lua function.

* `{opts}` must be a lua table. Is not optional. If you don't use any options you provide an empty table.

So we could create a function that "reloads" our configuration.

```lua
vim.api.nvim_create_user_command('ReloadConfig', 'source $MYVIMRC', {})
```

User commands are a fairly advance topic so if you want to know more details you can check the documentation.

```vim
:help nvim_create_user_command()
:help user-commands
```

### Autocommands

With autocommands we can execute actions when Neovim triggers an event. You can check the complete list of events with this command `:help events`.

We can create autocommands with this function.

```lua
vim.api.nvim_create_autocmd({event}, {opts})
```

* `{event}` must be a string with the name of an event.

* `{opts}` must be a lua table, its properties will determine the behavior of the autocommand. These are some of the most useful options.

  - `desc` a string that describes what the autocommand does.

  - `group` it can be a number or a string. If you provide a string it must be the name of an existing group. If you provide a number it must be the "id" of a group.

  - `pattern` can be a lua table or a string. This allows us to control when we want to trigger the autocommand. Its value depends on the event. Check the documentation of the event to know the possible values.

  - `once` it can be a boolean. If enabled the autocommand will only execute once. The default value is `false`.

  - `command` a string. Must be valid vimscript. Is the action we want to execute.

  - `callback` it can be a string or a lua function. If you provide a string it must be the name of a function written in vimscript. This is the action we want to execute. It can't be used with `command`.

Here is an example. I'll create a group called `user_cmds` and add two autocommands to it.

```lua
local augroup = vim.api.nvim_create_augroup('user_cmds', {clear = true})

vim.api.nvim_create_autocmd('FileType', {
  pattern = {'help', 'man'},
  group = augroup,
  desc = 'Use q to close the window',
  command = 'nnoremap <buffer> q <cmd>quit<cr>'
})

vim.api.nvim_create_autocmd('TextYankPost', {
  group = augroup,
  desc = 'Highlight on yank',
  callback = function(event)
    vim.highlight.on_yank({higroup = 'Visual', timeout = 200})
  end
})
```

Creating the group is optional by the way.

The first autocommand will make a keymap `q` to close the current window, but only if the filetype is `help` or `man`. In this example I'm using vimscript but I could have done it with a lua function.

The second autocommand will highlight the text we copy using `y`. If you want to test the effect try copying a line using `yy`.

To know more about autocommands in general check the documentation.

```vim
:help autocmd-intro
```

### User modules

We can use lua modules to split our configuration into smaller pieces.

A common convention is to put every module we create into a single folder. We do this to avoid any potential conflict with a plugin. Lots of people call this module `user` (you can use another name). To make this module we need to create a couple of folders inside our config folder. First create a `lua` folder, and inside that create a `user` folder. You can do it with this command.

```lua
:call mkdir(stdpath("config") . "/lua/user", "p")
```

Inside `/lua/user` we create our lua scripts.

Let's pretend we have one called `settings.lua`. Neovim doesn't know it exists, it won't be executed automatically. We need to call it from `init.lua`.

```lua
require('user.settings')
```

If you want to know more details about `require`'s behavior inside Neovim checkout.

```vim
:help lua-require
```

### The require function

There is something you should know about `require`, it only executes code once. What does mean?

Consider this code.

```lua
require('user.settings')
require('user.settings')
```

In here the script `settings.lua` will only be executed once. If you want to create plugins or create a feature that depends on a plugin, this behavior is good. The bad news is if want to use `:source $MYVIMRC` to reload our config the results might not be what we expect.

There is a very simple hack we can do to make it `source` friendly. We can empty `require`'s cache before using it. Like this.

```lua
local load = function(mod)
  package.loaded[mod] = nil
  require(mod)
end

load('user.settings')
load('user.keymaps')
```

If we do this in `init.lua` the `source` command will be able to execute all the files in our config.

**WARNING**. Be careful with this. Some plugins might act weird if you configure them twice. What do I mean? If we use `source` and call the `.setup` function of a plugin a second time it might have unexpected effects.

## init.lua

If we apply (almost) everything we learned here in a single configuration file this would be the result.

```lua
-- ================================================================== --
-- ===                      EDITOR SETTINGS                       === --
-- ================================================================== --

vim.opt.number = true
vim.opt.mouse = 'a'
vim.opt.ignorecase = true
vim.opt.smartcase = true
vim.opt.hlsearch = false
vim.opt.wrap = true
vim.opt.breakindent = true
vim.opt.tabstop = 2
vim.opt.shiftwidth = 2
vim.opt.expandtab = false

vim.g.netrw_banner = 0
vim.g.netrw_winsize = 30

vim.g.mapleader = ' '

vim.keymap.set({'n', 'x'}, 'cp', '"+y')
vim.keymap.set({'n', 'x'}, 'cv', '"+p')
vim.keymap.set({'n', 'x'}, 'x', '"_x')
vim.keymap.set('n', '<leader>w', '<cmd>write<cr>')
vim.keymap.set('n', '<leader>a', ':keepjumps normal! ggVG<CR>')

local group = vim.api.nvim_create_augroup('user_cmds', {clear = true})

vim.api.nvim_create_autocmd('TextYankPost', {
  desc = 'Highlight on yank',
  group = group,
  callback = function()
    vim.highlight.on_yank({higroup = 'Visual', timeout = 200})
  end,
})

vim.api.nvim_create_autocmd('FileType', {
  pattern = {'help', 'man'},
  group = group,
  command = 'nnoremap <buffer> q <cmd>quit<cr>'
})

-- ================================================================== --
-- ===                          PLUGINS                           === --
-- ================================================================== --

local install_path = vim.fn.stdpath('data') .. '/site/pack/packer/start/packer.nvim'
local install_plugins = false

if vim.fn.empty(vim.fn.glob(install_path)) > 0 then
  print('Installing packer...')
  local packer_url = 'https://github.com/wbthomason/packer.nvim'
  vim.fn.system({'git', 'clone', '--depth', '1', packer_url, install_path})
  print('Done.')

  vim.cmd('packadd packer.nvim')
  install_plugins = true
end

vim.api.nvim_create_user_command(
  'ReloadConfig',
  'source $MYVIMRC | PackerCompile',
  {}
)

require('packer').startup(function(use)
  use 'wbthomason/packer.nvim'
  use 'joshdick/onedark.vim'
  use 'nvim-lualine/lualine.nvim'

  if install_plugins then
    require('packer').sync()
  end
end)

if install_plugins then
  return
end


-- ================================================================== --
-- ===                    PLUGIN CONFIGURATION                    === --
-- ================================================================== --

---
-- Colorscheme
---

vim.opt.termguicolors = true
vim.cmd('colorscheme onedark')


---
-- lualine.nvim (statusline)
---

vim.opt.showmode = false
require('lualine').setup({
  options = {
    icons_enabled = false,
    theme = 'onedark',
    component_separators = '|',
    section_separators = '',
  },
})
```

## What's next?

If you feel prepared to tackle more advanced topics you can read this other tutorial. It teaches how to setup an autocompletion engine and also add support for LSP:

* [Setup nvim-lspconfig + nvim-cmp](@/tools/setup-nvim-lspconfig-plus-nvim-cmp.md)

If you want recommendations for plugins and other configurations I suggest you check these configuration templates.

* [kickstart.nvim](https://github.com/nvim-lua/kickstart.nvim)
* [nvim-basic-ide](https://github.com/LunarVim/nvim-basic-ide)
* [Neovim-from-scratch](https://github.com/LunarVim/Neovim-from-scratch) 

You can also check my personal configuration if you like.

* [neovim config](https://github.com/VonHeikemen/dotfiles/tree/master/my-configs/neovim)

## Conclusion

Now we know how to configure some basic options in Neovim. We learned how to create our very own keybindings. We know how to get plugins from github. We manage to configure a couple of plugins, a lua plugin and one written in vimscript. We took a brief look at some advance topics like recursive mappings, user commands, autocommands and lua modules.

I'd say you have everything you need to start exploring plugins, check other people's config and learn from them.

