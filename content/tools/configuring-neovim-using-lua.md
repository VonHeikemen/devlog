+++
title = "Everything you need to know to configure neovim using lua"
description = "Your first steps into a lua configuration"
date = 2021-08-01
updated = 2023-02-15
lang = "en"
[taxonomies]
tags = ["vim", "neovim", "shell"]
[extra]
shared = [
  ["dev.to", "https://dev.to/vonheikemen/everything-you-need-to-know-to-configure-neovim-using-lua-3h58"],
  ["Hashnode", "https://vonheikemen.hashnode.dev/everything-you-need-to-know-to-configure-neovim-using-lua"]
]
+++

After a long time in development neovim 0.5 was finally released as a stable version. Among the new exciting features we have better lua support and the promise of a stable api to create our configuration using this language. So today I'm going to share with you everything I learned while I was migrating my own configuration from vimscript to lua.

If you are new to neovim and want to start your configuration from scratch, I recommend you read this: [Build your first Neovim configuration in lua](@/tools/build-your-first-lua-config-for-neovim.md).

Here I'm going to talk about the things we can do with lua and its interaction with vimscript. I will be showing a lot of examples but I will not tell you what options you should set with what value. Also, this won't be a tutorial on "how to turn neovim into an IDE", I'll avoid anything that is language specific. What I want to do is teach you enough about lua and the neovim api so you can migrate your own configuration.

I will assume your operating system is linux (or something close to it) and that your configuration is located at `~/.config/nvim/init.vim`. Everything that I will show should work on every system in which neovim can be installed, just keep in mind that the path to the `init.vim` file might be different in your case.

Let us begin.

## First steps

The first thing you need to know is that we can embed lua code directly in `init.vim`. So we can migrate our config piece by piece and only change from `init.vim` to `init.lua` when we are ready.

Let's do the "hello world" to test that everything works as expected. Try this in your `init.vim`.

```vim
lua <<EOF
print('hello from lua')
EOF
```

After "sourcing" the file or restarting neovim the message `hello from lua` should appear right below your statusline. In here we are using something called `lua-heredoc`, so everything that's between `<<EOF ... EOF` will be considered a lua script and will be executed by the `lua` command. This is useful when we want to execute multiple lines of code but it's not necessary when we only need one. This is valid.

```vim
lua print('this also works')
```

But if we are going to call lua code from vimscript I say we use a real script. In lua we can do this by using the `require` function. For this to work we need to create a `lua` folder somewhere in the `runtimepath` of neovim.

You'll probably want to use the same folder where `init.vim` is located, so we will create `~/.config/nvim/lua`, and inside that we'll create a script called `basic.lua`. For now we will only print a message.

```lua
print('hello from ~/config/nvim/lua/basic.lua')
```

Now from our `init.vim` we can call it like this.

```vim
lua require('basic')
```

When we do this neovim will search in every directory in the `runtimepath` for a folder called `lua` and inside that it'll look for `basic.lua`. Neovim will run the last script that meets those conditions.

If you go around and check other people's code you'll notice that they use a `.` as a path separator. For example, let's say they have the file `~/.config/nvim/lua/usermod/settings.lua`. If they want to call `settings.lua` they do this.

```lua
require('usermod.settings')
```

Is a very common convention. Just remember that the dot is a path separator.

With all this knowledge we are ready to begin our configuration using lua.

## Editor settings

Each option in neovim is available to us in the global variable called `vim`... well more than just a variable try think of this as a global module. With `vim` we have access to the editor's settings, we also have the neovim api and even a set of helper functions (a standard library if you will). For now, we only need to care about something they call "meta-accessors", is what we'll use to access all the options we need.

### Scopes

Just like in vimscript, in lua we have different scopes for each option. We have global settings, window settings, buffer settings and a few others. Each one has its own namespace inside the `vim` module.

* vim.o

Gets or sets general settings.

```lua
vim.o.background = 'light'
```

* vim.wo

Gets or sets window-scoped options.

```lua
vim.wo.colorcolumn = '80'
```

* vim.bo

Gets or sets buffer-scoped options.

```lua
vim.bo.filetype = 'lua'
```

* vim.g

Gets or sets global variables. This is usually the namespace where you'll find variables set by plugins. The only one I know isn't tied to a plugin is the leader key.

```lua
-- use space as a the leader key
vim.g.mapleader = ' '
```

You should know that some variable names in vimscript are not valid in lua. We still have access to them but we can't use the dot notation. For example, [vim-zoom](https://github.com/dhruvasagar/vim-zoom) has a variable called `zoom#statustext` and in vimscript we use it like this.

```vim
let g:zoom#statustext = 'Z'
```

In lua we would have to do this.

```lua
vim.g['zoom#statustext'] = 'Z'
```

As you might have guessed this also gives us an oportunity to access properties which have the name of keywords. You may find yourselves in a situation where you need to access a property called `for`, `do` or `end` which are reserved keywords, in those cases remember this bracket syntax.

* vim.env

Gets or sets environment variables.

```lua
vim.env.FZF_DEFAULT_OPTS = '--layout=reverse'
```

As far as I know if you make a change in an environment variables the change will only apply in the active neovim session.

But now how do we know which "scope" we need to use when we're writting our config? Don't worry about that, think of `vim.o` and company just as a way to read values. When it's time set values we can use another method.

### vim.opt

With `vim.opt` we can set global, window and buffer settings.

```lua
-- buffer-scoped
vim.opt.autoindent = true

-- window-scoped
vim.opt.cursorline = true

-- global scope
vim.opt.autowrite = true
```

When we use it like this `vim.opt` acts like the `:set` command in vimscript, it give us a consistent way to modify neovim's options.

A funny thing you can do is assign `vim.opt` to a variable called `set`.

Say we have this piece of vimscript.

```vim
" Set the behavior of tab
set tabstop=2
set shiftwidth=2
set softtabstop=2
set expandtab
```

We could translate this easily in lua like this.

```lua
local set = vim.opt

-- Set the behavior of tab
set.tabstop = 2
set.shiftwidth = 2
set.softtabstop = 2
set.expandtab = true
```

> When you declare a variable do not forget the `local` keyword. In lua variables are global by default (that includes functions).

Anyway, what about global variables or the environment variables? For those you should keep using `vim.g` and `vim.env` respectively

What's interesting about `vim.opt` is that each property is a kind of special object, they are "meta-tables". It means that these objects implement their own behavior for certain common operations.

In the first example we had something like this: `vim.opt.autoindent = true`, and now you might think you can inspect the current value by doing this.

```lua
print(vim.opt.autoindent)
```

You won't get the value you expect, `print` will tell you `vim.opt.autoindent` is a table. If you want to know the value of an option you'll need to use the `:get()` method.

```lua
print(vim.opt.autoindent:get())
```

If you really, really want to know what's inside `vim.out.autoindent` you need to use `vim.inspect`.

```lua
print(vim.inspect(vim.opt.autoindent))
```

Or even better, if you have version v0.7 you can inspect the value using the command `lua =`.

```vim
:lua = vim.opt.autoindent
```

Now that will show you the internal state of this property.

### Types of data

Even when we assign a value inside `vim.opt` there is a little bit of magic going on in the background. I think is important to know how `vim.opt` can handle different types of data and compare it with vimscript.

* Booleans

These might not seem like a big deal but there is still a difference that is worth mention.

In vimscript we can enable or disable an option like this.

```vim
set cursorline
set nocursorline
```

This is the equivalent in lua.

```lua
vim.opt.cursorline = true
vim.opt.cursorline = false
```

* Lists

For some options neovim expects a comma separated list. In this case we could provide it as a string ourselves.

```lua
vim.opt.wildignore = '*/cache/*,*/tmp/*'
```

Or we could use a table.

```lua
vim.opt.wildignore = {'*/cache/*', '*/tmp/*'}
```

If you check the content of `vim.o.wildignore` you'll notice is the thing we want `*/cache/*,*/tmp/*`. If you really want to be sure you can check with this command.

```vim
:set wildignore?
```

You'll get the same result.

But the magic does not end there. Sometimes we don't need to override the list, sometimes we need to add an item or maybe delete it. To makes things easier `vim.opt` offers support for the following operations:

**Add an item to the end of the list**

Let's take `errorformat` as an example. If we want to add to this list using vimscript we do this.

```vim
set errorformat+=%f\|%l\ col\ %c\|%m
```

In lua we have a couple of ways to achieve the same goal:

Using the `+` operator.

```lua
vim.opt.errorformat = vim.opt.errorformat + '%f|%l col %c|%m'
```

Or the `:append` method.

```lua
vim.opt.errorformat:append('%f|%l col %c|%m')
```

**Add to the beginning**

In vimscript:

```vim
set errorformat^=%f\|%l\ col\ %c\|%m
```

Lua:

```lua
vim.opt.errorformat = vim.opt.errorformat ^ '%f|%l col %c|%m'

-- or try the equivalent

vim.opt.errorformat:prepend('%f|%l col %c|%m')
```

**Delete an item**

Vimscript:

```vim
set errorformat-=%f\|%l\ col\ %c\|%m
```

Lua:

```lua
vim.opt.errorformat = vim.opt.errorformat - '%f|%l col %c|%m'

-- or the equivalent

vim.opt.errorformat:remove('%f|%l col %c|%m')
```

* Pairs

Some options expect a list of key-value pairs. To ilustrate this we'll use `listchars`.

```vim
set listchars=tab:▸\ ,eol:↲,trail:·
```

In lua we can use tables for this too.

```lua
vim.opt.listchars = {eol = '↲', tab = '▸ ', trail = '·'}
```

> Note: to actually see this on your screen you need to enable the `list` option. See [:help listchars](https://neovim.io/doc/user/options.html#'listchars').

Since we are still using tables this option also supports the same operations mentioned in the previous section.

## Calling vim functions

Vimscript like any other programming language it has its own built-in functions ([many functions](https://neovim.io/doc/user/usr_41.html#function-list)) and thanks to the `vim` module we can call them throught `vim.fn`. Just like `vim.opt`, `vim.fn` is a meta-table,  but this one is meant to provide a convenient syntax for us to call vim functions. We use it to call built-in functions, user defined functions and even functions of plugins that are not written in lua.

We could for example check the neovim version like this:

```lua
if vim.fn.has('nvim-0.7') == 1 then
  print('we got neovim 0.7')
end
```

Wait, hold up, why are we comparing the result of `has` with a `1`? Ah, well, it turns out vimscript only included booleans in the `7.4.1154` version. So functions like `has` return `0` or `1`, and in lua both are truthy.

I've already mentioned that vimscript can have variable names that are not valid in lua, in that case you know you can use square brackets like this.

```lua
vim.fn['fzf#vim#files']('~/projects', false)
```

Another way we can solve this is by using `vim.call`.

```lua
vim.call('fzf#vim#files', '~/projects', false)
```

Those two do the exact same thing. In practice `vim.fn.somefunction()` and `vim.call('somefunction')` have the same effect.

Now let me show you something cool. In this particular case the lua-vimscript integration is so good we can use a plugin manager without any special adapters.

### vim-plug in lua

I know there is a lot of people out there who use [vim-plug](https://github.com/junegunn/vim-plug/), you might think you need to migrate to a plugin manager that is written in lua, but that's not the case. We can use `vim.fn` and `vim.call` to bring vim-plug to lua.

```lua
local Plug = vim.fn['plug#']

vim.call('plug#begin', '~/.config/nvim/plugged')

-- List of plugins goes here
-- ....

vim.call('plug#end')
```

Those 3 lines of code are the only thing you need. You can try it, this works.

```lua
local Plug = vim.fn['plug#']

vim.call('plug#begin', '~/.config/nvim/plugged')

Plug 'wellle/targets.vim'
Plug 'tpope/vim-surround'
Plug 'tpope/vim-repeat'

vim.call('plug#end')
```

Before you say anything, yes, all of that is valid lua. If a function only receives a single argument, and that argument is a string or a table, you can omit the parenthesis.

If you use the second argument of `Plug` you'll need the parenthesis and the second argument must be a table. Let me show you. If you have this in vimscript.

```vim
Plug 'scrooloose/nerdtree', {'on': 'NERDTreeToggle'}
```

In lua you'll need to do this.

```lua
Plug('scrooloose/nerdtree', {on = 'NERDTreeToggle'})
```

Unfortunately `vim-plug` has a couple of options that will cause an error if we use this syntax, those are `for` and `do`. In this case we need to wrap the key in quotes and square brackets.

```lua
Plug('junegunn/goyo.vim', {['for'] = 'markdown'})
```

You might know that the `do` option takes a string or a function which will be executed when the plugin is updated or installed. But what you might not know is that we are not forced to use a "vim function", we can use lua function and it'll work just fine.

```lua
Plug('VonHeikemen/rubber-themes.vim', {
  ['do'] = function()
    vim.opt.termguicolors = true
    vim.cmd('colorscheme rubber')
  end
})
```

There you have it. You don't need to use a plugin manager written in lua if you don't want to.

## Vimscript is still our friend

You might have notice in that last example I used `vim.cmd` to set the color scheme, this is because there are still things we can't do with lua. And so what we do is use `vim.cmd` to call vimscript commands (or expressions) that don't have a lua equivalent.

One interestings thing you should know is `vim.cmd` can execute multiple lines of vimscript. It means that we can do lots of things in a single call.

```lua
vim.cmd [[
  syntax enable
  colorscheme rubber
]]
```

So anything that you can't "translate" to lua you can put it in a string and pass that to `vim.cmd`.

I told you we can execute any vim command, right? I feel compelled to tell you this includes the `source` command. For those who don't know, `source` allows us to call other files written in vimscript. For example, if you have lots of keybindings writing in vimscript but don't wish to convert them to lua right now, you can create a script called `keymaps.vim` and call it.

```lua
vim.cmd 'source ~/.config/nvim/keymaps.vim'
```

## Keybindings

No, we don't need vimscript for this. We can do it in lua.

We can create our keybindings using `vim.keymap.set`. This functions takes 4 arguments.

* Mode. But not the name of the mode, we need the abbreviation. You can find a list of valid options [here](https://github.com/nanotee/nvim-lua-guide#defining-mappings).
* Key we want to bind.
* Action we want to execute.
* Extra arguments. These are the same options we would use in vimscript, you can find the list [here](https://neovim.io/doc/user/map.html#:map-arguments). Also, check the documentation [:help vim.keymap.set()](https://neovim.io/doc/user/lua.html#vim.keymap.set()).

So if we wanted to translate this to lua.

```vim
nnoremap <Leader>w :write<CR>
```

We would have to do this.

```lua
vim.keymap.set('n', '<Leader>w', ':write<CR>')
```

By default our keybindings will be "non recursive", it means that we don't have to worry about infinite loops. This opens the possibility to create alternatives of built-in keybindings. For example, we could center the screen after making a search with `*`.

```lua
vim.keymap.set('n', '*', '*zz', {desc = 'Search and center screen'})
```

In here we are using `*` in the key and the action, and we won't have any conflict.

But sometimes we do need recursive bindings, most of the time when the action we want to execute was created by a plugin. In this case we can use `remap = true` in the last parameter.

```lua
vim.keymap.set('n', '<leader>e', '%', {remap = true, desc = 'Go to matching pair'})
```

Now the best part, `vim.keymap.set` can use a lua function as the action.

```lua
vim.keymap.set('n', 'Q', function()
  print('Hello')
end, {desc = 'Say hello'})
```

Let's talk about the `desc` option. This allows us to add a description to our keybindings. We can read these description if we use the command `:map <keybinding>`.

So executing `:map *` will show us.

```
n  *           * *zz
                 Search and center screen
```

It becomes specially useful when using binding lua functions. If we inspect our `Q` mapping with `:map Q` we will get.

```
n  Q           * <Lua function 50>
                 Say hello
```

Notice we can't see the source for our function but at least we see the description and know what it can do. It is worth mention that plugins can read this property and show it with a nice presentation.

## User commands

Since version v0.7 we can create our own "ex-commands" using lua, with the function `vim.api.nvim_create_user_command`.

`nvim_create_user_command` expects three arguments:

* Name of the command. It **must** begin with an uppercase letter.
* Command. Can be a string or a lua function.
* Attributes. A lua table with the options for our command. You can find details in the documentation [:help nvim_create_user_command()](https://neovim.io/doc/user/api.html#nvim_create_user_command()) and also [:help command-attributes](https://neovim.io/doc/user/map.html#command-attributes).

Say we have this command in vimscript.

```vim
command! -bang ProjectFiles call fzf#vim#files('~/projects', <bang>0)
```

Using lua we have the possibility of using the vimscript command in a string.

```lua
vim.api.nvim_create_user_command(
  'ProjectFiles',
  "call fzf#vim#files('~/projects', <bang>0)",
  {bang = true}
)
```

Or, we could use a lua function.

```lua
vim.api.nvim_create_user_command(
  'ProjectFiles',
  function(input)
    vim.call('fzf#vim#files', '~/projects', input.bang)
  end,
  {bang = true, desc = 'Search projects folder'}
)
```

Yes, we can also add a description to user commands. We read these descriptions using the command `:command <name>`. But they will only appear if our command is bound to a lua function. So the command `:command ProjectFiles` will show this.

```
    Name              Args Address Complete    Definition
!   ProjectFiles      0                        Search projects folder
```

If the command had been vimscript expression the that expression will appear in the `Definition` column.

## Autocommands

With autocommands we execute functions and expressions on certain events. See [:help events](https://neovim.io/doc/user/autocmd.html#events) for a list of built-in events.

If we wanted to modify the `rubber` colorscheme a little bit we would do something like this.

```vim
augroup highlight_cmds
  autocmd!
  autocmd ColorScheme rubber highlight String guifg=#FFEB95
augroup END
```

> This block should be executed before calling the `colorscheme` command.

This would be the equivalent in lua.

```lua
local augroup = vim.api.nvim_create_augroup('highlight_cmds', {clear = true})

vim.api.nvim_create_autocmd('ColorScheme', {
  pattern = 'rubber',
  group = augroup,
  command = 'highlight String guifg=#FFEB95'
})
```

Notice here we are using an option called `command`, this can only execute vimscript. If we wanted to use a lua function we need to replace `command` with an option called `callback`.

```lua
local augroup = vim.api.nvim_create_augroup('highlight_cmds', {clear = true})

vim.api.nvim_create_autocmd('ColorScheme', {
  pattern = 'rubber',
  group = augroup,
  desc = 'Change string highlight',
  callback = function()
    vim.api.nvim_set_hl(0, 'String', {fg = '#FFEB95'})
  end
})
```

Just like the previous apis we can add a description for autocommands. To inspect an autocommand use the command `:autocmd <event> <pattern>`. But just like user commands they will only be available for inspection when using a lua function. If we check with `:autocmd ColorScheme rubber` will get.

```
ColorScheme
    rubber    Change string highlight
```

To know more details about autocommands checkout the documentation, see [:help nvim_create_autocmd()](https://neovim.io/doc/user/api.html#nvim_create_autocmd()) and [:help autocmd](https://neovim.io/doc/user/autocmd.html#autocommand).

## Plugin manager

You might want a plugin manager that is written in lua, just because. It appears that right now these are your options:

* [paq](https://github.com/savq/paq-nvim/)

A plugin manager that is simple and fast. I'm serious, this thing has less than 300 lines of code. It was created to download, update and remove plugins. That's it. If you don't need anything else, look no further, this is the plugin manager you want.

* [packer](https://github.com/wbthomason/packer.nvim)

It has the basic features you would expect, it also offers lazy-loading capabilities, has support for `luarocks` (which is like a package manager for lua), it can handle "local plugins". And does other things I don't understand, but the point is that is a feature complete plugin manager.

* [lazy.nvim](https://github.com/folke/lazy.nvim)

Has emerged as an alternative to `packer.nvim`. It advertises automatic lazy-loading that allows fast startup times. It lets you manage your plugins with a very nice interface. You can also split your plugins into modules where you can specify dependencies, configurations, version and other things.

* [vim-packager](https://github.com/kristijanhusak/vim-packager)

This one is not written in lua but I want to add it because it does offer a lua api. It offers more features than `paq` but not as much a `packer` or `lazy.nvim`, so if you are looking for a middle ground between those, this might be a good choice.

## Conclusion

Recap time. We learned how to use lua from vimscript. We now know how to use vimscript from lua. We have the tools to activate, deactivate and modify all sorts of options and variables in neovim. We got to know the methods we have available to create our keymaps. We learned about commands and autocommands. We figure out how to use plugin managers that aren't written in lua, and saw a few alternatives that are written in lua. I say we are ready to use lua in neovim.

For those who want to see a real world usage or whatever, I'll share a link to my current config in github:

* [neovim (v0.6)](https://github.com/VonHeikemen/dotfiles/tree/057f4d604e31ad121c4b4b36fd7e4148483d50c8/my-configs/neovim)
* [neovim (v0.7)](https://github.com/VonHeikemen/dotfiles/tree/18fb9ab01d171a3d862d2438cd0f8fd2612a3e52/my-configs/neovim)
* [neovim (v0.8+)](https://github.com/VonHeikemen/dotfiles/tree/master/my-configs/neovim)

## Sources

* [learn x in y minutes: where X=lua](https://learnxinyminutes.com/docs/lua/)
* [nvim-lua-guide](https://github.com/nanotee/nvim-lua-guide)
* [:help lua-heredoc](https://neovim.io/doc/user/lua.html#:lua-heredoc)
* [:help lua-vim-variables](https://neovim.io/doc/user/lua.html#lua-vim-variables)
* [:help lua-stdlib](https://neovim.io/doc/user/lua.html#lua-stdlib)
* [:help function-list](https://neovim.io/doc/user/usr_41.html#function-list)
* [curist's bundle.lua](https://github.com/curist/dotvim/blob/98b161f0759d3316fcf6a776d03665d6ab4827ee/bundles.lua)

