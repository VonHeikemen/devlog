+++
title = "A guide on Neovim's LSP client" 
description = "Enable IDE-like features in Neovim without installing any additional plugins"
date = 2023-12-25
lang = "en"
[taxonomies]
tags = ["neovim", "shell"]
[extra]
shared = []
+++

Maybe I should have called this "How to enable IDE-like features without third party plugins." Sounds interesting, right? That's basically what I want to show you here.

We don't need any third party plugins, but we do need this:

1. Neovim v0.8 or greater.
2. A language server.
3. Patience/Energy to write some lua code for each language server.

If you want to implement any of this stuff in your own configuration, consider dedicating a little bit of time to learn the basics of lua. Here are a couple of links to help you with that:

* [Lua crash course (11 min video)](https://www.youtube.com/watch?v=NneB6GX1Els)
* [Learn X in Y minutes: Where X = lua](https://learnxinyminutes.com/docs/lua/)
* And... there is also [Neovim's official lua guide](https://neovim.io/doc/user/lua-guide.html)

## Let's start with the language server

A language server is an external program that follows the [Language Server Protocol](https://microsoft.github.io/language-server-protocol/). The LSP specification defines what type of messages a language server can receive, and also how it should respond. The idea here is that [any tool that follows the LSP specification](https://microsoft.github.io/language-server-protocol/implementors/tools/) can communicate with a language server.

And so the language server is the thing that analyzes our source code and it can tell the editor what to do.

*Where can we find these language servers?*

The website for the LSP specification [has a list](https://microsoft.github.io/language-server-protocol/implementors/servers/).

### In this particular case...

I'm going to use [intelephense](https://intelephense.com/) to show the minimal configuration needed to setup a language server in Neovim.

If you want to test intelephense you need to install [NodeJS](https://nodejs.org/en). And then you can install the server running this command in the terminal.

```sh
npm install -g intelephense
```

Once you have a language server installed it's a good idea to check if Neovim "knows" where it is. You can execute this command inside Neovim.

```vim
:echo exepath('intelephense')
```

This should show you the path to the language server executable. If it doesn't, it means something went wrong during the installation.

In case you didn't click on the link to intelephense, you should know that is a language server for php. If you just want to test the code I show here, you don't need the php interpreter installed, just the source code of a php project. You can use this repository: [minicli](https://github.com/minicli/minicli), is a decent size codebase and doesn't depend on any other php libraries.

## Basic Usage

Before we write any code we should learn how to use the language server. The first piece of information we need is the command that starts the server. This should be in the official documentation of said server.

If we can't find the basic usage in the documentation we can go to [nvim-lspconfig](https://github.com/neovim/nvim-lspconfig)'s github repository. In there we look for a folder called [server_configurations](https://github.com/neovim/nvim-lspconfig/tree/master/lua/lspconfig/server_configurations), this contains configuration files for a bunch of language servers.

Right now we are interested in intelephense, so we should inspect the contents of [intelephense.lua](https://github.com/neovim/nvim-lspconfig/blob/4a69ad6646eaad752901413c9a0dced73dfb562d/lua/lspconfig/server_configurations/intelephense.lua). The thing we are looking for is in a property called `default_config`. This piece of code right here:

```lua
default_config = {
  cmd = { 'intelephense', '--stdio' },
  filetypes = { 'php' },
  root_dir = function(pattern)
    local cwd = vim.loop.cwd()
    local root = util.root_pattern('composer.json', '.git')(pattern)

    -- prefer cwd if root is a descendant
    return util.path.is_descendant(cwd, root) and cwd or root
  end,
},
```

The `cmd` property has the command we need to start the language server. `filetypes` is the list of languages the server can handle. And I'm going to talk about the `root_dir` in a little while.

## Execute on filetype

Since we only need intelephense in php files we can use something called a filetype plugin. That's a script that gets executed after Neovim assigns a filetype to a buffer.

We create a filetype plugin in our Neovim configuration simply by adding a script in the folder `ftplugin`. Note that the name of the script needs to be the same as a valid filetype.

We can navigate to Neovim's configuration folder, open Neovim and then create the `ftplugin` folder.

```vim
:call mkdir('./ftplugin', 'p')
```

We want a filetype plugin for php, so we create a new file called `php.lua`.

```vim
:edit ftplugin/php.lua | write
```

Inside this new file we are going to execute the function that enables intelephense.

## Root directory

The last piece of information we need is the root directory. We just have to tell the language server where is our project folder.

In our filetype plugin we are going to use a function called [vim.fs.find()](https://neovim.io/doc/user/lua.html#vim.fs.find()). We will give it a list of files and it will return the path of the first match it finds.

What do we look for? We search for common configuration files that projects have in their root folder. So, in php is very common to have a `composer.json` file. Javascript projects usually have a `package.json`. Rust projects have a `cargo.toml`. We feed this information to `vim.fs.find()` and it should give us a path we can use.

We can make a test already by adding this piece of code in the newly created `php.lua`.

```lua
-- ftplugin/php.lua

local root_files = {'composer.json'}
local paths = vim.fs.find(root_files, {stop = vim.env.HOME})

print(vim.fs.dirname(paths[1]))
```

By default `vim.fs.find()` will look in the current folder and then the parent folders. The `stop` argument tells the function it should stop the search if it hits the `home` folder (you don't want your language server to analyze your home folder by accident).

Since `vim.fs.find()` returns a list we just pick the first item. And to make sure we get the path to a folder we use `vim.fs.dirname()`.

We can navigate to a php project with a composer.json file and check that the project path is detected correctly.

## Start the client

We are ready to enable the language server. Now we call the function [vim.lsp.start()](https://neovim.io/doc/user/lsp.html#vim.lsp.start()). The first time this is executed it will launch the language server as an external process. When called again with the same root directory it will only send information to the existing process.

Our php filetype plugin should look like this.

```lua
-- ftplugin/php.lua

local root_files = {'composer.json'}
local paths = vim.fs.find(root_files, {stop = vim.env.HOME})

vim.lsp.start({
  cmd = {'intelephense', '--stdio'},
  root_dir = vim.fs.dirname(paths[1]),
})
```

With this setup we should get "diagnostics" out the box. If there is an error in a php file Neovim will show what line has the error, and also the message. Something like this.

![Example code from slimphp framework showing an error](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/xcdxdx9kj3xh90o2369u.png)

The `E` in line 8 is a diagnostic sign, it indicates there is an error. And the thing after the `■` symbol is "virtual text" showing the error message.

### Server settings

Now, some server specific configuration should be placed in a property called `settings` in the `vim.lsp.start()` function. But here's the thing, you may find the documentation of some language servers shows them in this format:

```
intelephense.files.maxSize: 1000000
```

We would need to adapt this so it works with Neovim's LSP client. Let me show you how it should be.

```lua
vim.lsp.start({
  cmd = {'intelephense', '--stdio'},
  root_dir = vim.fs.dirname(paths[1]),
  settings = {
    intelephense = {
      files = {
        maxSize = 1000000,
      },
    }
  }
})
```

Basically, each dot is a nested "lua table" we need to add.

If there was another setting with the same namespace `intelephense.files`, we just add it to the existing table.

```lua
vim.lsp.start({
  cmd = {'intelephense', '--stdio'},
  root_dir = vim.fs.dirname(paths[1]),
  settings = {
    intelephense = {
      files = {
        maxSize = 1000000,
        anotherOptionExample = false,
      },
    }
  }
})
```

### Where to look if something goes wrong?

If Neovim wasn't able to start the language server, you can take a look at the log file, execute this command inside Neovim:

```lua
:lua vim.cmd.edit(vim.lsp.get_log_path())
```

Look for the lines that start with `[ERROR]`. Maybe there is an error message with some useful information.

If you want the logs to have more details, increase the log level. Add this in your `init.lua` file.

```lua
vim.lsp.set_log_level('debug')
```

## About the diagnostics

So the diagnostics signs, the thing Neovim uses to tell us there is an error in our source code... by default the space needed to render that sign is hidden, and when there is a sign the whole screen shifts to the right. That behavior can be configured in the `init.lua` file.

If you set the option `signcolumn` to the string `yes`, Neovim will reserve the space for the sign. You will have a whitespace reserved for any type of signs in the gutter.

```lua
vim.opt.signcolumn = 'yes'
```

If you set `signcolumn` to the string `no`, Neovim will hide the column altogether. Don't do that unless you are fully aware of the consequences. There is a better way to hide the diagnostic signs.

### vim.diagnostic

There is a lua module dedicated specifically to diagnostics: [vim.diagnostic](https://neovim.io/doc/user/diagnostic.html). This has a [.config()](https://neovim.io/doc/user/diagnostic.html#vim.diagnostic.config()) function we can use to configure the interface of the diagnostics.

We can add the following to our `init.lua`.

* Hide diagnostic signs

This is the "safe" way to disable the diagnostics sign.

```lua
vim.diagnostic.config({
  signs = false,
})
```

* Disable virtual text

Yes, there's also an option to hide the virtual text that contains the error message.

```lua
vim.diagnostic.config({
  virtual_text = false,
})
```

To read the diagnostic message of the line under the cursor we can use the function [vim.diagnostic.open_float()](https://neovim.io/doc/user/diagnostic.html#vim.diagnostic.open_float()) in a keybinding.

```lua
vim.keymap.set('n', 'gl', '<cmd>lua vim.diagnostic.open_float()<cr>')
```

I would love to explain all the options `vim.diagnostic.config()` supports but we don't have time for that. If you want to know more you can [read the documentation](https://neovim.io/doc/user/diagnostic.html#vim.diagnostic.config()).

## What else do we get for free?

These are things Neovim does when a language server active in the buffer.

**Since Neovim v0.8**

* There is an LSP powered `tagfunc`. That means we can jump to the definition of a function or class using the keybinding `<C-]>` (Control + ]). And can jump back to where we were using `<C-t>`.

* The `formatexpr` option is set to a function that uses the language server. This means the `gq` operator can format a piece of code. We can enter visual mode, select a piece of text, press `gq` and Neovim will request the language server to format the code.

* The `omnifunc` option is also set. This one enables smart code completions. In insert mode the keybinding `<C-x><C-o>` (control + x then control + o) will trigger the completion menu with suggestions that the language server think are relevant.

**Since Neovim v0.9**

* Semantic highlight is supported. Some language servers can provide information about the tokens in the source code, this allows for a more accurate syntax highlight.

**Since Neovim v0.10**

* In normal mode, if we don't have a custom keybinding for `K` then it will display the available documentation for the symbol under the cursor.

## Let's get some easy wins

Here's a list of features we can use without too much effort, pretty much the only thing we have to do is call a lua function.

* Jump to declaration
* Lists implementations
* Jump to type definition
* List all references 
* Display a function's signature information
* Rename all references to the symbol under the cursor
* Format current file (or selected range)
* List and execute a code action
* Show diagnostic details
* Move between diagnostics

All of these can be used in custom keybindings.

### Custom keybindings

Is very common for people to follow the same patterns Neovim's defaults have, that is only enable features when the language server is active. So let me introduce you to the `LspAttach` event, this is triggered everytime Neovim enables a language server in a buffer. And with the function `nvim_create_autocmd` we can tell Neovim we want to execute a callback function everytime this event happens.

> You can find more details about autocommands here: [lua-guide-autocommands](https://neovim.io/doc/user/lua-guide.html#lua-guide-autocommands).

So this is one way to setup custom keybindings.

```lua
-- you can add this in your init.lua
-- (note: diagnostics are not exclusive to LSP)

-- Show diagnostics in a floating window
vim.keymap.set('n', 'gl', '<cmd>lua vim.diagnostic.open_float()<cr>')

-- Move to the previous diagnostic
vim.keymap.set('n', '[d', '<cmd>lua vim.diagnostic.goto_prev()<cr>')

-- Move to the next diagnostic
vim.keymap.set('n', ']d', '<cmd>lua vim.diagnostic.goto_next()<cr>')

vim.api.nvim_create_autocmd('LspAttach', {
  desc = 'LSP actions',
  callback = function(event)
    local bufmap = function(mode, lhs, rhs)
      local opts = {buffer = event.buf}
      vim.keymap.set(mode, lhs, rhs, opts)
    end

    -- You can find details of these function in the help page
    -- see for example, :help vim.lsp.buf.hover()

    -- Trigger code completion
    bufmap('i', '<C-Space>', '<C-x><C-o>')

    -- Display documentation of the symbol under the cursor
    bufmap('n', 'K', '<cmd>lua vim.lsp.buf.hover()<cr>')

    -- Jump to the definition
    bufmap('n', 'gd', '<cmd>lua vim.lsp.buf.definition()<cr>')

    -- Jump to declaration
    bufmap('n', 'gD', '<cmd>lua vim.lsp.buf.declaration()<cr>')

    -- Lists all the implementations for the symbol under the cursor
    bufmap('n', 'gi', '<cmd>lua vim.lsp.buf.implementation()<cr>')

    -- Jumps to the definition of the type symbol
    bufmap('n', 'go', '<cmd>lua vim.lsp.buf.type_definition()<cr>')

    -- Lists all the references 
    bufmap('n', 'gr', '<cmd>lua vim.lsp.buf.references()<cr>')

    -- Displays a function's signature information
    bufmap('n', '<C-k>', '<cmd>lua vim.lsp.buf.signature_help()<cr>')

    -- Renames all references to the symbol under the cursor
    bufmap('n', '<F2>', '<cmd>lua vim.lsp.buf.rename()<cr>')

    -- Format current file
    bufmap('n', '<F3>', '<cmd>lua vim.lsp.buf.format()<cr>')

    -- Selects a code action available at the current cursor position
    bufmap('n', '<F4>', '<cmd>lua vim.lsp.buf.code_action()<cr>')
  end
})
```

## Fair warning

Not every language server implements the entire LSP specification. The features they offer depend on the implementation.

For example, intelephense can show diagnostics in real time, there is no need to save the file to get new diagnostics. But [rust-analyzer](https://github.com/rust-lang/rust-analyzer), the language server for rust, can only update diagnostics after saving the file.

Here's another example: [ruff-lsp](https://github.com/astral-sh/ruff-lsp), a language server for python. It describes itself as a linter and code formatter. As far as I can tell `ruff-lsp` does not provide code completions or semantic highlights.

What I want say is this: read the documentation of the language server so you know what it can do.

## Bonus content

At this point I'd say you have all the essential knowledge needed to be productive. What follows are tips, configurations, and features you can implement by adding some boilerplate code in your Neovim configuration.

### Configure a language server for multiple filetypes

Sometimes a language server can support multiple filetypes. An example of this is [tsserver](https://github.com/typescript-language-server/typescript-language-server), the language server for javascript and typescript. In this case a filetype plugin can still work but there is an easier way to go about it.

One option to consider is a "plugin script." In there we can configure the language server in the callback function of a `FileType` autocommand.

If you want follow along, install tsserver using this command in the terminal.

```sh
npm install -g typescript typescript-language-server
```

For this we need to create a `plugin` folder inside Neovim's configuration folder. So, we navigate to Neovim's config folder, open Neovim, then execute this command to create the plugin folder.

```vim
:call mkdir('./plugin', 'p')
```

Next we create a lua script, it can have any name we want. We can call it `tsserver.lua`.

```vim
:edit plugin/tsserver.lua | write
```

In `tsserver.lua` we are going to adapt the configuration in [nvim-lspconfig's source code](https://github.com/neovim/nvim-lspconfig/blob/3f6d120721e3a2b2812af43e6a8ba5522aa421c5/lua/lspconfig/server_configurations/tsserver.lua).

```lua
-- plugin/tsserver.lua

local function start_tsserver()
  local root_files = {'package.json', 'tsconfig.json', 'jsconfig.json'}
  local paths = vim.fs.find(root_files, {stop = vim.env.HOME})

  vim.lsp.start({
    name = 'tsserver',
    cmd = {'typescript-language-server', '--stdio'},
    root_dir = vim.fs.dirname(paths[1]),
    init_options = {hostInfo = 'neovim'},
  })
end

vim.api.nvim_create_autocmd('FileType', {
  pattern = {'javascript', 'javascriptreact', 'typescript', 'typescriptreact'},
  desc = 'Start typescript LSP',
  callback = start_tsserver,
})
```

This will work exactly like a filetype plugin, except here we are executing one lua function and not an entire file. The advantage of the autocommand is we can define multipe filetypes in the `pattern` property.

By the way, this doesn't have to be a plugin script, we can setup the autocommand in the `init.lua` file.

### Add borders to floating windows

Sadly, there is no way to add borders to all floating windows, this means we have to enable it for each feature.

We can add the following to our `init.lua` file.

* Diagnostic details

```lua
vim.diagnostic.config({
  float = {
    border = 'rounded',
  },
})
```

* Documentation window 

The one used by the function [vim.lsp.buf.hover()](https://neovim.io/doc/user/lsp.html#vim.lsp.buf.hover()).

```lua
vim.lsp.handlers['textDocument/hover'] = vim.lsp.with(
  vim.lsp.handlers.hover,
  {border = 'rounded'}
)
```

* Signature help

The one used by the function [vim.lsp.buf.signature_help()](https://neovim.io/doc/user/lsp.html#vim.lsp.buf.signature_help()).

```lua
vim.lsp.handlers['textDocument/signatureHelp'] = vim.lsp.with(
  vim.lsp.handlers.signature_help,
  {border = 'rounded'}
)
```

### Format on save

The only thing we will do here is trigger the function [vim.lsp.buf.format()](https://neovim.io/doc/user/lsp.html#vim.lsp.buf.format()) before Neovim saves a file. And of course, we only do it when there is an active language server.

Important note: most language servers with formatting capabilities have their own style settings. For example, we can have 2 space indent in our Neovim config, but maybe the language server formats the code with 4 space indent. So it's a good idea to check the documentation of the language server to see how to configure that.

```lua
-- You can add this in your init.lua
-- or a plugin script

local fmt_group = vim.api.nvim_create_augroup('autoformat_cmds', {clear = true})

local function setup_autoformat(event)
  vim.api.nvim_clear_autocmds({group = fmt_group, buffer = event.buf})

  local id = vim.tbl_get(event, 'data', 'client_id')
  local client = id and vim.lsp.get_client_by_id(id)
  if client == nil then
    return
  end

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

When using **Neovim v0.9 or lower** we need to call the function [vim.fn.sign_define()](https://neovim.io/doc/user/builtin.html#sign_define()).

```lua
-- You can add this in your init.lua
-- or a plugin script

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
-- or a plugin script

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

### Disable diagnostics in insert mode

This is already the default behavior but there is a problem... not really a problem, just a minor detail: the diagnostics only disappear after we start typing something.

This is the code **I use** to disable diagnostics right after going into insert mode (or select mode).

```lua
-- You can add this in your init.lua
-- or a plugin script

vim.api.nvim_create_autocmd('ModeChanged', {
  pattern = {'n:i', 'v:s'},
  desc = 'Disable diagnostics in insert and select mode',
  callback = function(e) vim.diagnostic.disable(e.buf) end
})

vim.api.nvim_create_autocmd('ModeChanged', {
  pattern = 'i:n',
  desc = 'Enable diagnostics when leaving insert mode',
  callback = function(e) vim.diagnostic.enable(e.buf) end
})
```

### Disable semantic highlights

Neovim's documentation suggest that we "clear" the highlight groups of the `@lsp` namespace. So, we can do this.

```lua
-- You can add this in your init.lua
-- this should be executed before setting the colorscheme

local function hide_semantic_highlights()
  for _, group in ipairs(vim.fn.getcompletion('@lsp', 'highlight')) do
    vim.api.nvim_set_hl(0, group, {})
  end
end

vim.api.nvim_create_autocmd('ColorScheme', {
  desc = 'Clear LSP highlight groups',
  callback = hide_semantic_highlights,
})
```

The `ColorScheme` autocommand should be created before setting up the colorscheme. This way Neovim can clear the highlights even if we change the colorscheme in the middle of a coding session.

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
-- or a plugin script

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

When the language server can provide code completion, it'll use that. Otherwise, it will try to suggest words found in the current buffer.

Note that you can use the Enter key or `<C-y>` to confirm the current item in the completion menu

```lua
-- You can add this in your init.lua
-- or a plugin script

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

> Unstable feature. Neovim v0.10 or greater is required.

Some language servers can provide snippets in their code completions, but right now Neovim doesn't have a mechanism to expand them automatically. Neovim's developers are making progress on this front though, since Neovim v0.10 there is a new module called [vim.snippet](https://neovim.io/doc/user/lua.html#vim.snippet). For the moment it doesn't have any "deep integration" with the completion menu, but in it's current state we can use it to implement our own expand autocommand.

```lua
-- You can add this in your init.lua
-- or a plugin script

local function expand_snippet(event)
  local comp = vim.v.completed_item
  local item = vim.tbl_get(comp, 'user_data', 'nvim', 'lsp', 'completion_item')

  -- Check that we were given a snippet
  if (
    not item
    or not item.insertTextFormat
    or item.insertTextFormat == 1
    or not (item.kind == vim.lsp.protocol.CompletionItemKind.Snippet)
  ) then
    return
  end

  -- Remove the inserted text
  local cursor = vim.api.nvim_win_get_cursor(0)
  local line = vim.api.nvim_get_current_line()
  local lnum = cursor[1] - 1
  local start_char = cursor[2] - #comp.word
  vim.api.nvim_buf_set_text(event.buf, lnum, start_char, lnum, #line, {''})

  -- Insert snippet
  local snip_text = vim.tbl_get(item, 'textEdit', 'newText') or item.insertText

  assert(snip_text, "Language server indicated it had a snippet, but no snippet text could be found!")

  -- warning: this api is not stable yet
  vim.snippet.expand(snip_text)
end

vim.api.nvim_create_autocmd('CompleteDone', {
  desc = 'Expand LSP snippet',
  callback = expand_snippet
})
```

`vim.snippet` also supports snippet placeholders. This means we can jump to different places in the current active snippet. Here are some keybindings that we can use.

```lua
-- You can add this in your init.lua

-- Control + f: Jump to next snippet placeholder
vim.keymap.set({'i', 's'}, '<C-f>', function()
  -- warning: this api is not stable yet
  if vim.snippet.jumpable(1) then
    return '<cmd>lua vim.snippet.jump(1)<cr>'
  else
    return '<C-f>'
  end
end, {expr = true})

-- Control + b: Jump to previous snippet placeholder
vim.keymap.set({'i', 's'}, '<C-b>', function()
  -- warning: this api is not stable yet
  if vim.snippet.jumpable(-1) then
    return '<cmd>lua vim.snippet.jump(-1)<cr>'
  else
    return '<C-b>'
  end
end, {expr = true})

-- Control + l: Exit current snippet
vim.keymap.set({'i', 's'}, '<C-l>', function()
  -- warning: this api is not stable yet
  if vim.snippet.active() then
    return '<cmd>lua vim.snippet.exit()<cr>'
  else
    return '<C-l>'
  end
end, {expr = true})
```

### Enable inlay hints

> Unstable feature. Neovim v0.10 or greater is required.

For this we can use the function [vim.lsp.inlay_hint.enable()](https://neovim.io/doc/user/lsp.html#vim.lsp.inlay_hint.enable()). This one needs to be called on a per file basis. But that's not a problem, we can always use the good old `LspAttach` autocommand.

Note that some language servers may have inlay hints disabled by default. The settings needed to enable hints should be in the documentation of the language server.

```lua
-- You can add this in your init.lua
-- or a plugin script

vim.api.nvim_create_autocmd('LspAttach', {
  desc = 'Enable inlay hints',
  callback = function(event)
    local id = vim.tbl_get(event, 'data', 'client_id')
    local client = id and vim.lsp.get_client_by_id(id)
    if client == nil or not client.supports_method('textDocument/inlayHint') then
      return
    end

    -- warning: this api is not stable yet
    vim.lsp.inlay_hint.enable(event.buf, true)
  end,
})
```

## Conclusion

Hopefully I showed is not difficult to "connect" a language server with Neovim. Think about it, 1 shell commmand and 6 lines of lua code is all it takes to get intelephense working in Neovim.

The hard part is gathering all the context inside your head. What does LSP even mean? What's a language server? Filetype plugin? hardly know her. But once you know about the moving pieces and where to find the information you need, it gets easier.

One last thing, don't ignore the basics. Take your time and learn lua, read [Neovim's lua guide](https://neovim.io/doc/user/lua-guide.html), learn how to navigate Neovim's documentation with the `:help` command.

