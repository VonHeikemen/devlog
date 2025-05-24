+++
title = "A guide on Neovim's LSP client" 
description = "Enable IDE-like features in Neovim without installing any additional plugins"
date = 2023-12-25
updated = 2025-05-23
lang = "en"
[taxonomies]
tags = ["neovim", "shell"]
[extra]
shared = []
+++

Maybe I should have called this "How to enable IDE-like features without third party plugins." Sounds interesting, right? That's basically what I want to show you here.

I'm going to explain how to use the new configuration method that was introduced in Neovim v0.11. And I want to show how it works because it's basically a layer on top of existing features. This way you can make your own setup even on older Neovim versions.

## Requirements

1. Neovim v0.8 or greater.
2. A language server.
3. Patience/Energy to write some lua code for each language server.

If you don't know anything about lua here are a couple of links that can help you learn the basics:

* [Lua crash course (11 min video)](https://www.youtube.com/watch?v=NneB6GX1Els)
* [Learn X in Y minutes: Where X = lua](https://learnxinyminutes.com/docs/lua/)
* [Neovim's official lua guide](https://neovim.io/doc/user/lua-guide.html)

## Let's start with the language server

A language server is an external program that follows the [Language Server Protocol](https://microsoft.github.io/language-server-protocol/). The LSP specification defines what type of messages a language server can receive and also how it should respond. The idea here is that [any tool that follows the LSP specification](https://microsoft.github.io/language-server-protocol/implementors/tools/) can communicate with a language server.

And so the language server is the thing that analyzes our source code and it can tell the editor what to do.

*Where can we find these language servers?*

The website for the LSP specification [has a list](https://microsoft.github.io/language-server-protocol/implementors/servers/).

### In this particular case...

I want to use [Gleam](https://gleam.run/) and [LuaLS](https://github.com/LuaLS/lua-language-server) as examples.

Gleam is a toolchain. It has a compiler, a formatter and a language server. Install instructions are in the official documentation: [Getting started](https://gleam.run/getting-started/installing/).

LuaLS is a language server for lua. You can download it from [github releases](https://github.com/LuaLS/lua-language-server/releases/latest) or [build it from source](https://luals.github.io/wiki/build/).

Once you have a language server installed it's a good idea to check if Neovim "knows" where it is. You can check that using the function [exepath()](https://neovim.io/doc/user/builtin.html#exepath()).

For Gleam you want to search for the `gleam` executable.

```vim
:echo exepath('gleam')
```

LuaLS' executable is called `lua-language-server`.

```vim
:echo exepath('lua-language-server')
```

This should show you the path to the executable. If it doesn't, it means something went wrong during the installation.

## Basic Usage

Before we write any code we should learn how to use the language server. This should be in the official documentation of said server.

If we can't find the basic usage in the documentation we can go to [nvim-lspconfig](https://github.com/neovim/nvim-lspconfig)'s github repository. In there we look for a folder called [lsp](https://github.com/neovim/nvim-lspconfig/tree/master/lsp), this contains configuration files for a bunch of language servers.

Say we are interested in Gleam, we should go and inspect the contents of [gleam.lua](https://github.com/neovim/nvim-lspconfig/blob/ac1dfbe3b60e5e23a2cff90e3bd6a3bc88031a57/lsp/gleam.lua#L8).

```lua
return {
  cmd = { 'gleam', 'lsp' },
  filetypes = { 'gleam' },
  root_markers = { 'gleam.toml', '.git' },
}
```

This shows the minimum amount of information we need to make a language server work.

The `cmd` property has the command we need to start the language server. Remember, language servers are external programs so `gleam lsp` is a valid command you can type on the terminal. And funny enough, if you do that it'll give you this message.

```
This command is intended to be run by language server clients such
as a text editor rather than being run directly in the console.
```

`filetypes` is the list of languages the server can handle. These must be valid filetype names that Neovim supports.

`root_markers` should be a list of files or folders. We will use this to determine the root of the project.

## The lsp folder

This `lsp` folder is not something unique only `nvim-lspconfig` can have. Since **Neovim v0.11** it is now part of the `runtimepath`. This means we can have an `lsp` folder inside our own personal configuration.

Imagine a Neovim config folder with this structure:

```
nvim
├── init.vim
├── .nvim.lua
└── lsp
    ├── gleam.lua
    └── luals-nvim.lua
```

The configuration inside `nvim/lsp/gleam.lua` can be the exact same thing `nvim-lspconfig` has.

```lua
-- nvim/lsp/gleam.lua

return {
  cmd = {'gleam', 'lsp'},
  filetypes = {'gleam'},
  root_markers = {'gleam.toml', '.git'},
}
```

We do have some amount of freedom here. So in `luals-nvim` I want to have a configuration made specifically for a Neovim config with an `.nvim.lua` script at the root.

```lua
-- nvim/lsp/luals-nvim.lua

return {
  cmd = {'lua-language-server'},
  filetypes = {'lua'},
  root_markers = {'.nvim.lua'},
  settings = {
    Lua = {
      runtime = {
        version = 'LuaJIT',
        path = {'lua/?.lua', 'lua/?/init.lua'},
      },
      diagnostics = {
        globals = {'vim'},
      },
      telemetry = {
        enable = false,
      },
      workspace = {
        checkThirdParty = false,
        library = {
          vim.env.VIMRUNTIME,
        },
      },
    },
  },
}
```

Inside the `lsp` folder the files can have any name. So I made this `luals-nvim` instead of something generic like `lua` or `luals`. And the return value can be anything that the [vim.lsp.config()](https://neovim.io/doc/user/lsp.html#vim.lsp.config()) function expects.

Notice in this one we have a `settings` property. That's reserved for server specific options. You'll have to search the documentation of the language server to know what options you can add there.

Now, having a configuration in the `lsp` folder doesn't mean Neovim will use it. We have to be explicit. To use a language server we must call the function [vim.lsp.enable()](https://neovim.io/doc/user/lsp.html#vim.lsp.enable()).

So in this example the `init.vim` looks like this.

```vim
" nvim/init.vim

set exrc

lua vim.lsp.enable('gleam')
```

*But why init.vim?*

I just wanted an excuse to show is possible to execute lua code inside vimscript. Some Vim users think they have to delete their vimscript config to use lua. That's just not true.

Anyway, next time we open Neovim it will look for a file that matches the pattern `lsp/gleam.lua` inside the `runtimepath`. Then it will create an autocommand using the list we provided in the `filetypes` property. So whenever we open a file with the type `gleam` Neovim will try to enable the language server.

*What about LuaLS?*

The config we have for LuaLS only makes sense for Neovim. So I think this is a good oportunity to use an [exrc](https://neovim.io/doc/user/options.html#'exrc') file. Vim's version of a project local config. When the `exrc` option is enabled Vim/Neovim will execute a script located in the current working directory. This is both convenient and dangerous. So Neovim's `exrc` is slightly different from Vim. Neovim will ask you if you "trust" the file, and if you say yes it'll be executed. If the content of the file remains the same Neovim won't ask again, it'll just execute it automatically.

Personally, I would create an alias to enable `exrc` instead of having it in the `init` file. Something like this.

```sh
alias code='nvim --cmd "set exrc"'
```

Let's go back to LuaLS. After Neovim is done executing the `init` file it will search for an `.nvim.lua` in the current working directory. The one in our Neovim folder will have this.

```lua
-- nvim/.nvim.lua

vim.lsp.enable('luals-nvim')
```

So Neovim will not even try to use `lsp/luals-nvim.lua` if we are not inside the config folder. The downside of this approach (right now) is that we have to open Neovim in the exact location where `.nvim.lua` is stored.

## Single file setup?

If you are the kind of person that has a very simple config that fits in one file, I have good news for you. You are not forced to use the `lsp` folder to configure a language server.

The configurations inside the `lsp` folder can be extended using the function [vim.lsp.config()](https://neovim.io/doc/user/lsp.html#vim.lsp.config()). But this can also be used to create an entire new config, without needing to create a new file.

Here's an example:

```lua
-- nvim/init.lua

vim.lsp.config('gleam', {
  cmd = {'gleam', 'lsp'},
  filetypes = {'gleam'},
  root_markers = {'gleam.toml', '.git'},
})

vim.lsp.enable('gleam')
```

## Where to look if something goes wrong?

If Neovim wasn't able to start the language server, you can take a look at the log file, execute this command inside Neovim:

```lua
:lua vim.cmd.edit(vim.lsp.get_log_path())
```

Look for the lines that start with `[ERROR]`. Maybe there is an error message with some useful information.

If you want the logs to have more details, increase the log level using this function.

```lua
vim.lsp.set_log_level('debug')
```

And since **Neovim v0.10** you can do a "health check" with this command:

```vim
:checkhealth lsp
```

## Execute on filetype

Turns out `vim.lsp.enable()` is actually a layer on top of things we already had. We can still use a language server without plugins on older Neovim versions, we just need to know how to put the pieces together.

If the server we want to use only supports one language it makes sense to use a "filetype plugin."

We create a filetype plugin in our Neovim configuration simply by adding a script in the folder `ftplugin`. Note that the name of the script needs to be the same as a valid filetype.

We can navigate to Neovim's configuration folder, open Neovim and then create the `ftplugin` folder.

```vim
:call mkdir('./ftplugin', 'p')
```

If we want a filetype plugin for Gleam we create a new file called `gleam.lua`.

```vim
:edit ftplugin/gleam.lua | write
```

On the other hand, if we have a language server that can work on multiple filetypes we can use an autocommand on the event `FileType`.

```lua
vim.api.nvim_create_autocmd('FileType', {
  pattern = {'css', 'less', 'sass'},
  callback = function()
    ---
    -- In here you can do whatever you want
    ---
  end,
})
```

The `pattern` of the `FileType` event is a list of filetypes. The same thing you would provide in the `filetypes` property of an lsp config file.

Inside the filetype plugin or the autocommand we are going to execute the function that enables the language server. But first, we need to know how recreate the logic behind `root_markers`.

## Root directory

Language servers need to know what is the path of our project folder and this is a problem Neovim needs to solve. 

On **Neovim v0.10** we can use a function called [vim.fs.root()](https://neovim.io/doc/user/lua.html#vim.fs.root()). We will give it a list of files and it will return the parent directory of the first match.

What do we look for? We search for common configuration files that projects have in the root folder. So, in a Gleam project there is always a `gleam.toml` file. Javascript projects usually have a `package.json`. Rust projects have a `cargo.toml`. We feed this information to `vim.fs.root()` and it should give us a path we can use.

We can make a test already by adding this piece of code in a `gleam` filetype plugin.

```lua
-- nvim/ftplugin/gleam.lua
-- NOTE: vim.fs.root() is only available on Neovim v0.10 or greater

local root_markers = {'gleam.toml'}
local root_dir = vim.fs.root(0, root_markers)

print(root_dir)
```

On **Neovim v0.9** or lower we would have to recreate the behavior of `vim.fs.root()`. For this we can use [vim.fs.find()](https://neovim.io/doc/user/lua.html#vim.fs.find()).

```lua
-- nvim/ftplugin/gleam.lua
-- NOTE: this code is for Neovim v0.9.5 or lower

local root_markers = {'gleam.toml'}
local buffer = vim.api.nvim_buf_get_name(0)
local paths = vim.fs.find(root_markers, {
  upward = true,
  path = vim.fn.fnamemodify(buffer, ':p:h'),
})

local root_dir = vim.fs.dirname(paths[1])

print(root_dir)
```

## Start the client

We are ready to enable the language server. Now we call the function [vim.lsp.start()](https://neovim.io/doc/user/lsp.html#vim.lsp.start()), which is the same thing `vim.lsp.enable()` uses under the hood.

The first time this is executed it will launch the language server as an external process. When called again with the same root directory it will only send information to the existing process.

Our gleam filetype plugin can look like this.

```lua
-- nvim/ftplugin/gleam.lua
-- NOTE: vim.fs.root() is only available on Neovim v0.10 or greater

local root_markers = {'gleam.toml'}
local root_dir = vim.fs.root(0, root_markers)

if root_dir then
  vim.lsp.start({
    cmd = {'gleam', 'lsp'},
    root_dir = root_dir,
  })
end
```

With this setup we should get "diagnostics" out the box. If there is an error in a gleam file Neovim will show what line has the error.

## About the diagnostics

So the diagnostics signs, the thing Neovim uses to tell us there is an error in our source code... by default the space needed to render that sign is hidden and when there is a sign the whole screen shifts to the right. That behavior can be configured.

If you set the option `signcolumn` to the string `yes` Neovim will reserve the space for the sign. You will have a whitespace reserved for any type of signs in the gutter.

In your `init.vim` you can have this.

```vim
set signcolumn=yes
```

Or, if you have an `init.lua`.

```lua
vim.o.signcolumn = 'yes'
```

If you set `signcolumn` to the string `no`, Neovim will hide the column altogether. Don't do that unless you are fully aware of the consequences. There is a better way to hide the diagnostic signs.

### vim.diagnostic

There is a lua module dedicated specifically to diagnostics: [vim.diagnostic](https://neovim.io/doc/user/diagnostic.html). This has a [.config()](https://neovim.io/doc/user/diagnostic.html#vim.diagnostic.config()) function we can use to configure the interface of the diagnostics.

* Hide diagnostic signs

This is the safe way to disable the diagnostics sign.

```lua
vim.diagnostic.config({
  signs = false,
})
```

* Virtual text

There is an option to show the error message inline. This is called "virtual text." This used to be enabled by default, but now on **Neovim v0.11** is disabled.

```lua
vim.diagnostic.config({
  virtual_text = true,
})
```

If you have **Neovim v0.10 or greater** you can read the diagnostic message under the cursor with the keybinding `<C-w>d` (control + w then d). This will trigger the function [vim.diagnostic.open_float()](https://neovim.io/doc/user/diagnostic.html#vim.diagnostic.open_float()).

When using **Neovim v0.9.5 or lower** you'll have to create that keybinding yourself.

```lua
vim.keymap.set('n', '<C-w>d', '<cmd>lua vim.diagnostic.open_float()<cr>')
```

Now, I would love to explain all the options `vim.diagnostic.config()` supports but we don't have time for that. If you want to know more you can [read the documentation](https://neovim.io/doc/user/diagnostic.html#vim.diagnostic.config()).

## What else do we get for free?

These are things Neovim does when a language server active in the buffer.

**Since Neovim v0.8**

* There is an LSP powered `tagfunc`. That means we can jump to the definition of a function or class using the keybinding `<C-]>` (control + ]). And can jump back to where we were using `<C-t>`.

* The `formatexpr` option is set to a function that uses the language server. This means the `gq` operator can format a piece of code. We can enter visual mode, select a piece of text, press `gq` and Neovim will request the language server to format the code.

* The `omnifunc` option is also set. This one enables smart code completions. In insert mode the keybinding `<C-x><C-o>` (control + x then control + o) will trigger the completion menu with suggestions that the language server think are relevant.

**Since Neovim v0.9**

* Semantic highlight is supported. Some language servers can provide information about the tokens in the source code, this allows for a more accurate syntax highlight.

**Since Neovim v0.10**

* In normal mode, if we don't have a custom keybinding for `K` then it will display the available documentation for the symbol under the cursor.

* In normal mode, `<C-w>d` opens a floating window showing the diagnostics in the line under the cursor.

* In normal mode, `[d` and `]d` can be used to move the cursor to the previous and next diagnostic of the current file.

**Since Neovim v0.11**

* In normal mode, `grn` renames all references of the symbol under the cursor.

* In normal mode, `gra` shows a list of code actions available in the line under the cursor.

* In normal mode, `grr` lists all the references of the symbol under the cursor.

* In normal mode, `gri` lists all the implementations for the symbol under the cursor.

* In normal mode, `gO` lists all symbols in the current buffer.

* In insert mode, `<Ctrl-s>` displays the function signature of the symbol under the cursor.

## LSP keymaps

Neovim v0.11 has defaults keymaps for almost everything now. The good news is that older versions of Neovim have the same features as v0.11, so the only thing we have to do is create keymaps for them.

So here's the code you can add in your personal configuration.

```lua
-- you can add this in your init.lua

-- These keymaps are the defaults in Neovim v0.10
vim.keymap.set('n', '[d', '<cmd>lua vim.diagnostic.goto_prev()<cr>')
vim.keymap.set('n', ']d', '<cmd>lua vim.diagnostic.goto_next()<cr>')
vim.keymap.set('n', '<C-w>d', '<cmd>lua vim.diagnostic.open_float()<cr>')
vim.keymap.set('n', '<C-w><C-d>', '<cmd>lua vim.diagnostic.open_float()<cr>')

vim.api.nvim_create_autocmd('LspAttach', {
  callback = function(event)
    local bufmap = function(mode, rhs, lhs)
      vim.keymap.set(mode, rhs, lhs, {buffer = event.buf})
    end

    -- These keymaps are the defaults in Neovim v0.11
    bufmap('n', 'K', '<cmd>lua vim.lsp.buf.hover()<cr>')
    bufmap('n', 'grr', '<cmd>lua vim.lsp.buf.references()<cr>')
    bufmap('n', 'gri', '<cmd>lua vim.lsp.buf.implementation()<cr>')
    bufmap('n', 'grn', '<cmd>lua vim.lsp.buf.rename()<cr>')
    bufmap('n', 'gra', '<cmd>lua vim.lsp.buf.code_action()<cr>')
    bufmap('n', 'gO', '<cmd>lua vim.lsp.buf.document_symbol()<cr>')
    bufmap({'i', 's'}, '<C-s>', '<cmd>lua vim.lsp.buf.signature_help()<cr>')

    -- These are custom keymaps
    bufmap('n', 'gd', '<cmd>lua vim.lsp.buf.definition()<cr>')
    bufmap('n', 'grt', '<cmd>lua vim.lsp.buf.type_definition()<cr>')
    bufmap('n', 'grd', '<cmd>lua vim.lsp.buf.declaration()<cr>')
    bufmap({'n', 'x'}, 'gq', '<cmd>lua vim.lsp.buf.format({async = true})<cr>')
  end,
})
```

And here is the vimscript equivalent:

```vim
" you can add this in your init.vim

" These keymaps are the defaults in Neovim v0.10
nnoremap [d <cmd>lua vim.diagnostic.goto_prev()<cr>
nnoremap ]d <cmd>lua vim.diagnostic.goto_next()<cr>
nnoremap <C-w>d <cmd>lua vim.diagnostic.open_float()<cr>
nnoremap <C-w><C-d> <cmd>lua vim.diagnostic.open_float()<cr>

function! LspAttached() abort
  " These keymaps are the defaults in Neovim v0.11
  nnoremap <buffer> K <cmd>lua vim.lsp.buf.hover()<cr>
  nnoremap <buffer> grr <cmd>lua vim.lsp.buf.references()<cr>
  nnoremap <buffer> gri <cmd>lua vim.lsp.buf.implementation()<cr>
  nnoremap <buffer> grn <cmd>lua vim.lsp.buf.rename()<cr>
  nnoremap <buffer> gra <cmd>lua vim.lsp.buf.code_action()<cr>
  nnoremap <buffer> gO <cmd>lua vim.lsp.buf.document_symbol()<cr>
  inoremap <buffer> <C-s> <cmd>lua vim.lsp.buf.signature_help()<cr>
  snoremap <buffer> <C-s> <cmd>lua vim.lsp.buf.signature_help()<cr>

  " These are custom keymaps
  nnoremap <buffer> gd <cmd>lua vim.lsp.buf.definition()<cr>
  nnoremap <buffer> gq <cmd>lua vim.lsp.buf.format({async = true})<cr>
  xnoremap <buffer> gq <cmd>lua vim.lsp.buf.format({async = true})<cr>
  nnoremap <buffer> grd <cmd>lua vim.lsp.buf.declaration()<cr>
  nnoremap <buffer> grt <cmd>lua vim.lsp.buf.type_definition()<cr>
endfunction

autocmd LspAttach * call LspAttached()
```

## Fair warning

Not every language server implements the entire LSP specification. The features may not be consistent between servers.

For example, Gleam can show diagnostics in real time, there is no need to save the file to get new diagnostics. But [rust-analyzer](https://github.com/rust-lang/rust-analyzer), the language server for rust, can only update diagnostics after saving the file.

Here's another example: [ruff-lsp](https://github.com/astral-sh/ruff-lsp), a language server for python. It describes itself as a linter and code formatter. As far as I can tell `ruff-lsp` does not provide code completions or semantic highlights.

What I want say is this: read the documentation of the language server so you know what it can do.

## Bonus content

At this point I'd say you have all the essential knowledge needed to be productive. What follows are tips, configurations, and features you can implement by adding some boilerplate code in your Neovim configuration.

### Format on save

What we do here is trigger the function [vim.lsp.buf.format()](https://neovim.io/doc/user/lsp.html#vim.lsp.buf.format()) before Neovim saves a file. And of course, we only do it when there is an active language server.

Important note: most language servers with formatting capabilities have their own style settings. For example, we can have 2 space indent in our Neovim config but maybe the language server formats the code with 4 space indent. So it's a good idea to check the documentation of the language server to see how to configure that.

```lua
-- You can add this in your init.lua
-- or a global plugin

local fmt_group = vim.api.nvim_create_augroup('autoformat_cmds', {clear = true})

local function setup_autoformat(event)
  local id = vim.tbl_get(event, 'data', 'client_id')
  local client = id and vim.lsp.get_client_by_id(id)
  if client == nil then
    return
  end

  vim.api.nvim_clear_autocmds({group = fmt_group, buffer = event.buf})

  local buf_format = function(e)
    vim.lsp.buf.format({
      bufnr = e.buf,
      async = false,
      timeout_ms = 10000,
    })
  end

  vim.api.nvim_create_autocmd('BufWritePre', {
    buffer = event.buf,
    group = fmt_group,
    desc = 'Format current buffer',
    callback = buf_format,
  })
end

vim.api.nvim_create_autocmd('LspAttach', {
  desc = 'Setup format on save',
  callback = setup_autoformat,
})
```

### Change diagnostics sign text

When using **Neovim v0.9.5 or lower** we need to call the function [vim.fn.sign_define()](https://neovim.io/doc/user/builtin.html#sign_define()).

```lua
-- You can add this in your init.lua
-- or a global plugin

local function sign_define(args)
  vim.fn.sign_define(args.name, {
    texthl = args.name,
    text = args.text,
    numhl = ''
  })
end

sign_define({name = 'DiagnosticSignError', text = '✘'})
sign_define({name = 'DiagnosticSignWarn', text = '▲'})
sign_define({name = 'DiagnosticSignHint', text = '⚑'})
sign_define({name = 'DiagnosticSignInfo', text = '»'})
```

When using **Neovim v0.10 or greater** we should do this with `vim.diagnostic.config()`.

```lua
-- You can add this in your init.lua
-- or a global plugin

vim.diagnostic.config({
  signs = {
    text = {
      [vim.diagnostic.severity.ERROR] = '✘',
      [vim.diagnostic.severity.WARN] = '▲',
      [vim.diagnostic.severity.HINT] = '⚑',
      [vim.diagnostic.severity.INFO] = '»',
    },
  },
})
```

### Disable semantic highlights

To opt-out of this feature Neovim's documentation suggest that we "delete" a property from the LSP client instance.

```lua
-- You can add this in your init.lua
-- or a global plugin

vim.api.nvim_create_autocmd('LspAttach', {
  desc = 'Disable LSP semantic highlights',
  callback = function(event)
    local id = vim.tbl_get(event, 'data', 'client_id')
    local client = id and vim.lsp.get_client_by_id(id)
    if client == nil then
      return
    end

    client.server_capabilities.semanticTokensProvider = nil
  end,
})
```

### Highlight symbol under cursor

What we want to do here is call the function [vim.lsp.buf.document_highlight()](https://neovim.io/doc/user/lsp.html#vim.lsp.buf.document_highlight()) when the cursor spends some amount of time on top of a symbol. And then clear the highlight when the cursor moves.

Note, for this to work properly the colorscheme needs to support the following highlight groups:

* LspReferenceRead
* LspReferenceText
* LspReferenceWrite

If the colorscheme does not support these highlight groups, we can "link" them to an existing group. Here's an example using the `Search` highlight group.

```lua
vim.api.nvim_set_hl(0, 'LspReferenceRead', {link = 'Search'})
vim.api.nvim_set_hl(0, 'LspReferenceText', {link = 'Search'})
vim.api.nvim_set_hl(0, 'LspReferenceWrite', {link = 'Search'})
```

```lua
-- You can add this in your init.lua
-- or a global plugin

-- time it takes to trigger the `CursorHold` event
vim.opt.updatetime = 400

local function highlight_symbol(event)
  local id = vim.tbl_get(event, 'data', 'client_id')
  local client = id and vim.lsp.get_client_by_id(id)
  if client == nil or not client.supports_method('textDocument/documentHighlight') then
    return
  end

  local group = vim.api.nvim_create_augroup('highlight_symbol', {clear = false})

  vim.api.nvim_clear_autocmds({buffer = event.buf, group = group})

  vim.api.nvim_create_autocmd({'CursorHold', 'CursorHoldI'}, {
    group = group,
    buffer = event.buf,
    callback = vim.lsp.buf.document_highlight,
  })

  vim.api.nvim_create_autocmd({'CursorMoved', 'CursorMovedI'}, {
    group = group,
    buffer = event.buf,
    callback = vim.lsp.buf.clear_references,
  })
end

vim.api.nvim_create_autocmd('LspAttach', {
  desc = 'Setup highlight symbol',
  callback = highlight_symbol,
})
```

### Simple tab complete

In this one we will use the `Tab` (and shift tab) key to navigate between the items in the completion menu. When the completion menu is not visible and the cursor is in a whitespace character, it will insert a tab character. Else, it will trigger the completion menu.

When the language server can provide code completion it'll use that. Otherwise, it will try to suggest words found in the current buffer.

Note that you can use the Enter key or `<C-y>` to confirm the current item in the completion menu

```lua
-- You can add this in your init.lua
-- or a global plugin

vim.opt.completeopt = {'menu', 'menuone', 'noselect', 'noinsert'}
vim.opt.shortmess:append('c')

local function tab_complete()
  if vim.fn.pumvisible() == 1 then
    -- navigate to next item in completion menu
    return '<Down>'
  end

  local c = vim.fn.col('.') - 1
  local is_whitespace = c == 0 or vim.fn.getline('.'):sub(c, c):match('%s')

  if is_whitespace then
    -- insert tab
    return '<Tab>'
  end

  local lsp_completion = vim.bo.omnifunc == 'v:lua.vim.lsp.omnifunc'

  if lsp_completion then
    -- trigger lsp code completion
    return '<C-x><C-o>'
  end

  -- suggest words in current buffer
  return '<C-x><C-n>'
end

local function tab_prev()
  if vim.fn.pumvisible() == 1 then
    -- navigate to previous item in completion menu
    return '<Up>'
  end

  -- insert tab
  return '<Tab>'
end

vim.keymap.set('i', '<Tab>', tab_complete, {expr = true})
vim.keymap.set('i', '<S-Tab>', tab_prev, {expr = true})
```

### Expand snippets

> Neovim v0.11 or greater is required.

For this we use a new module called [vim.lsp.completion](https://neovim.io/doc/user/lsp.html#lsp-completion), this will extend the behavior of the builtin completion so it can support "additional text edits" a language server can provide. Additional edits can be things like adding missing import statements or expanding code snippets.

Right now you have to opt-in to the features `vim.lsp.completion` provides. So, when a language server is active you have to call the [vim.lsp.completion.enable()](https://neovim.io/doc/user/lsp.html#vim.lsp.completion.enable()) function.

```lua
-- You can add this in your init.lua
-- or a global plugin
vim.opt.completeopt = {'menu', 'menuone', 'noinsert', 'noselect'}

vim.api.nvim_create_autocmd('LspAttach', {
  desc = 'Enable vim.lsp.completion',
  callback = function(event)
    local client_id = vim.tbl_get(event, 'data', 'client_id')
    if client_id == nil then
      return
    end

    vim.lsp.completion.enable(true, client_id, event.buf, {autotrigger = false})

    -- Trigger lsp completion manually using Ctrl + Space
    vim.keymap.set('i', '<C-Space>', '<cmd>lua vim.lsp.completion.trigger()<cr>')
  end
})
```

Notice in the last argument to `.enable()` there is a property called `autotrigger`. `false` is the default value so I just leave it like that. If you set it to `true` Neovim will trigger the completion menu when it finds a trigger character. Trigger characters change depending on the language server. In lua for example, the completion will be triggered automatically after a `.` or `:` character.

### Enable inlay hints

> Neovim v0.10 or greater is required.

For this we can use the function [vim.lsp.inlay_hint.enable()](https://neovim.io/doc/user/lsp.html#vim.lsp.inlay_hint.enable()).

Note that some language servers may have inlay hints disabled by default. The settings needed to enable hints should be in the documentation of the language server.

```lua
-- You can add this in your init.lua
-- or a global plugin

vim.api.nvim_create_autocmd('LspAttach', {
  desc = 'Enable inlay hints',
  callback = function(event)
    local id = vim.tbl_get(event, 'data', 'client_id')
    local client = id and vim.lsp.get_client_by_id(id)
    if client == nil or not client.supports_method('textDocument/inlayHint') then
      return
    end

    vim.lsp.inlay_hint.enable(true, {bufnr = event.buf})
  end,
})
```

## Conclusion

Hopefully I showed is not difficult to "connect" a language server with Neovim. The hard part is gathering all the context inside your head. What does LSP even mean? What's a language server? Filetype plugin? I hardly know her. But once you know about the moving pieces and where to find the information you need, it gets easier.

One last thing, don't ignore the basics. Take your time and learn lua, read [Neovim's lua guide](https://neovim.io/doc/user/lua-guide.html) and learn how to navigate Neovim's documentation with the `:help` command.

