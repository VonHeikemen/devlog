+++
title = "Neovim: Install plugins without a plugin manager"
description = "Learn how to install Neovim plugins just using git clone"
date = 2024-07-27
lang = "en"
[taxonomies]
tags = ["vim", "neovim", "shell"]
+++

Yes, and all we have to do is download them in the right path.

## How does it work?

Neovim and vim have this feature called [packages](https://neovim.io/doc/user/repeat.html#_using-vim-packages). A package is a directory that can contain multiple plugins.

The file structure of package can look like this:

```
some-package
├── colors
│   ├── a-colorscheme.vim
│   └── some-colorscheme.lua
├── doc
│   └── my-help-page.txt
├── lua
│   └── some-module
│       └── some-file.lua
└── plugin
    ├── a-file.vim
    └── some-file.lua
```

According to Neovim's documentation, a [plugin](https://neovim.io/doc/user/usr_05.html#_global-plugins) is literally a script located in the plugin directory. They call it a "global plugin". So in our example, `plugin/a-file.vim` is one plugin and `plugin/some-file.lua` is another plugin.

Notice something interesting? This is the same file structure all Neovim plugins can have. So what we consider to be one plugin, Neovim's documentation calls it a `package`.

But where should we add these things?

## The pack directory

Every directory in the `packpath` option can contain a directory called `pack`. Inside `pack` we can group our packages.

For example:

```
pack
└── vendor
    ├── opt
    │   └── package-one
    └── start
        ├── package-two
        └── package-three
```

`vendor` is just a random name I chose for this group. You can pick any name you want.

Within our group we can have `opt` packages and `start` packages.

Packages in the `start` directory will be available in the `runtimepath` during Neovim startup process.

A package in the `opt` directory will only be added to the `runtimepath` after we call the command `:packadd`. This is the way we can lazy load plugins.

## The download path

First, inspect the value of the option `packpath`. Execute this command in Neovim.

```vim
:set packpath?
```

If you find that unreadable, you can try to print each path using lua. Execute this command.

```lua
:lua vim.tbl_map(print, vim.opt.packpath:get())
```

That's the list of directories where you can have a `pack` directory.

Example time:

By default Neovim's configuration directory is a part of the packpath, so that's a perfectly valid place to create a `pack` directory.

Let's pretend we are using linux and want to download [mini.nvim](https://github.com/nvim-mini/mini.nvim). We could execute this command the terminal.

```
git clone https://github.com/nvim-mini/mini.nvim \
  ~/.config/nvim/pack/vendor/start/mini.nvim
```

Note that after we download a plugin we need to generate the help tags using this command inside Neovim.

```vim
:helptags ALL
```

After we create the help tags we can use the `:help` command to navigate to the documentation of the plugin.

Now that `mini.nvim` is available in Neovim's runtimepath we can test it. We can try for example the module `mini.files` in our Neovim configuration.

Let's create a keyboard shortcut to toggle `mini.files` explorer.

```lua
local mini_files = require('mini.files')

mini_files.setup({})

vim.keymap.set('n', '<F2>', function()
  if mini_files.close() then
    return
  end

  mini_files.open()
end)
```

## A little command

If you want, you can have a Neovim command that uses `git clone` to download plugins from github.

This should work just fine:

```lua
vim.api.nvim_create_user_command(
  'GitPlugin',
  function(input)
    local repo = input.fargs
    local url = 'https://github.com/%s/%s.git'
    local plugin_dir = vim.fn.stdpath('config') 
      .. '/pack/vendor/start/%s'

    if repo[1] == nil or repo[2] == nil then
      local msg = 'Must provide user name and repository'
      vim.notify(msg, vim.log.levels.WARN)
      return
    end

    local full_url = url:format(repo[1], repo[2])
    local command = {'git', 'clone', full_url, plugin_dir:format(repo[2])}

    local on_done = function()
      vim.cmd('packloadall! | helptags ALL')
      vim.notify('Done.')
    end

    vim.notify('Cloning repository...')
    vim.fn.jobstart(command, {on_exit = on_done})
  end,
  {nargs = '+'}
)
```

Going back to our previous example, now we could download `mini.nvim` using this command inside Neovim:

```
GitPlugin echasnovski mini.nvim
```

## Conclusion

You are ready to enjoy Neovim plugins available on github without a plugin manager.

I want to clarify, I'm not saying plugin managers are useless. You probably still need one. Neovim doesn't have a mechanism to update plugins, or remove them. The package feature only loads plugins into Neovim's runtime. You still need to manage your plugins somehow.

