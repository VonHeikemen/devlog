+++
title = "Can we manage neovim's LSP client without plugins?"
description = "Let's learn what does it take to use neovim's LSP client in project"
date = 2022-06-09
updated = 2022-10-16
lang = "en"
[taxonomies]
tags = ["vim", "neovim", "shell"]
[extra]
shared = [
  ["dev.to", "https://dev.to/vonheikemen/can-we-manage-neovims-lsp-client-without-plugins-3mge"],
  ["Hashnode", "https://vonheikemen.hashnode.dev/manage-neovim-lsp-client-without-plugins"]
]
+++

Yes, we can. The complexity of the setup will depend on what you want to achieve in your workflow. But if you know your way around lua you should be able to understand the "boilerplate" needed to get a decent setup.

Let's figure out how to make it work.

## The building blocks

It turns out we only need two functions. One to initialize the language server. One that notifies text changes to the server:

* `vim.lsp.start_client()`: This function creates a "client object" that handles all communications with a language server.

* `vim.lsp.buf_attach_client()`: It notifies all text changes to the language server.

A minimal and inefficient example using typescript's language server would look like this.

```lua
local launch_tsserver = function()
  local config = {
    cmd = {'typescript-language-server', '--stdio'},
    name = 'tsserver',
    root_dir = vim.fn.getcwd(),
    capabilities = vim.lsp.protocol.make_client_capabilities(),
  }

  local client_id = vim.lsp.start_client(config)

  if client_id then
    vim.lsp.buf_attach_client(0, client_id)
  end
end

vim.api.nvim_create_user_command(
  'LaunchTsserver',
  launch_tsserver,
  {desc = 'Start tsserver'}
)
```

Now with the command `LaunchTsserver` we would be able to start a `typescript-language-server` process and get diagnostics in the current file.

Why is this inefficient? Because it would only work on a single file. It'll spawn a new process each time we use it. What you really want is to "share" a single process of `typescript-language-server` for your project.

## Project setup

So what's the missing piece that would make this usable in a project? An autocommand. We need to tell neovim we want to attach the server to a buffer everytime we open a new file of a specific filetype.

```lua
local filetypes = {
  'javascript',
  'javascriptreact',
  'javascript.jsx'
  'typescript',
  'typescriptreact',
  'typescript.tsx',
}

local buf_attach = function()
  vim.lsp.buf_attach_client(0, client_id)
end

autocmd_id = vim.api.nvim_create_autocmd('FileType', {
  desc = string.format('Attach LSP: %s', client_name),
  pattern = filetypes,
  callback = buf_attach
})
```

When we create an autocommand with `nvim_create_autocmd` we get an id, which can be used to delete it later on. To delete the autocommand we would call `nvim_del_autocmd`.

```lua
vim.api.nvim_del_autocmd(autocmd_id)
```

But what would be the best time to create this autocommand? After the server is ready, of course. We can use the `on_init` hook to setup the autocommand and `on_exit` to clean it up.

If we apply all this knowledge to our `launch_tsserver` function we would have this.

```lua
local launch_tsserver = function()
  local autocmd
  local filetypes = {
    'javascript',
    'javascriptreact',
    'javascript.jsx'
    'typescript',
    'typescriptreact',
    'typescript.tsx',
  }

  local config = {
    cmd = {'typescript-language-server', '--stdio'},
    name = 'tsserver',
    root_dir = vim.fn.getcwd(),
    capabilities = vim.lsp.protocol.make_client_capabilities(),
  }

  config.on_init = function(client, results)
    local buf_attach = function()
      vim.lsp.buf_attach_client(0, client.id)
    end

    autocmd = vim.api.nvim_create_autocmd('FileType', {
      desc = string.format('Attach LSP: %s', client.name),
      pattern = filetypes,
      callback = buf_attach
    })

    if vim.v.vim_did_enter == 1 and
      vim.tbl_contains(filetypes, vim.bo.filetype)
    then
      buf_attach()
    end
  end

  config.on_exit = vim.schedule_wrap(function(code, signal, client_id)
    vim.api.nvim_del_autocmd(autocmd)
  end)

  vim.lsp.start_client(config)
end
```

## What about some keybindings?

Right? I mean, you probably want to use some of those LSP features with just a few keystrokes. How do we proceed? The documentation says we can use an autocommand with the event `LspAttach`. So you can define your keybindings anywhere you want.

```lua
vim.api.nvim_create_autocmd('LspAttach', {
  desc = 'LSP actions',
  callback = function()
    local bufmap = function(mode, lhs, rhs)
      local opts = {buffer = true}
      vim.keymap.set(mode, lhs, rhs, opts)
    end

    -- Displays hover information about the symbol under the cursor
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

    -- Selects a code action available at the current cursor position
    bufmap('n', '<F4>', '<cmd>lua vim.lsp.buf.code_action()<cr>')
    bufmap('x', '<F4>', '<cmd>lua vim.lsp.buf.range_code_action()<cr>')

    -- Show diagnostics in a floating window
    bufmap('n', 'gl', '<cmd>lua vim.diagnostic.open_float()<cr>')

    -- Move to the previous diagnostic
    bufmap('n', '[d', '<cmd>lua vim.diagnostic.goto_prev()<cr>')

    -- Move to the next diagnostic
    bufmap('n', ']d', '<cmd>lua vim.diagnostic.goto_next()<cr>')
  end
})
```

## Complete code

And now if we apply the new changes to our example this would be it.

```lua
local launch_tsserver = function()
  local autocmd
  local filetypes = {
    'javascript',
    'javascriptreact',
    'javascript.jsx'
    'typescript',
    'typescriptreact',
    'typescript.tsx',
  }

  local config = {
    cmd = {'typescript-language-server', '--stdio'},
    name = 'tsserver',
    root_dir = vim.fn.getcwd(),
    capabilities = vim.lsp.protocol.make_client_capabilities(),
  }

  config.on_init = function(client, results)
    local buf_attach = function()
      vim.lsp.buf_attach_client(0, client.id)
    end

    autocmd = vim.api.nvim_create_autocmd('FileType', {
      desc = string.format('Attach LSP: %s', client.name),
      pattern = filetypes,
      callback = buf_attach
    })

    if vim.v.vim_did_enter == 1 and
      vim.tbl_contains(filetypes, vim.bo.filetype)
    then
      buf_attach()
    end
  end

  config.on_exit = vim.schedule_wrap(function(code, signal, client_id)
    vim.api.nvim_del_autocmd(autocmd)
  end)

  vim.lsp.start_client(config)
end

vim.api.nvim_create_user_command(
  'LaunchTsserver',
  launch_tsserver,
  {desc = 'Start tsserver'}
)
```

## Can we do better?

Sure. I mean, you can hide almost all the boilerplate inside another function and reduce the noise.

Just imagine this.

```lua
local launch_tsserver = function()
  local config = make_config({
    cmd = {'typescript-language-server', '--stdio'},
    name = 'tsserver',
    filetypes = {
      'javascript',
      'javascriptreact',
      'javascript.jsx'
      'typescript',
      'typescriptreact',
      'typescript.tsx',
    }
  })

  vim.lsp.start_client(config)
end

vim.api.nvim_create_user_command(
  'LaunchTsserver',
  launch_tsserver,
  {desc = 'Start tsserver'}
)
```

How does `make_config` look like? Well... that's homework, my friend. You can implement it anyway you want. I've already showed you all the code you need to make it possible.

You really want to know how I would do it? The answer is in this github repository: [VonHeikemen/nvim-lsp-sans-plugins](https://github.com/VonHeikemen/nvim-lsp-sans-plugins)

## Conclusion

We learned enough about neovim's builtin LSP client to create our own little setup. We know how to initialize the language server. We can "share" the same server across multiple files. In the process we worked with autocommands, now we know how to create and delete them. We could totally manage some LSP servers without plugins.

## Sources

* [:help vim.lsp.start_client()](https://neovim.io/doc/user/lsp.html#vim.lsp.start_client()) 
* [:help vim.lsp.buf_attach_client()](https://neovim.io/doc/user/lsp.html#vim.lsp.buf_attach_client()) 
* [What is nvim-lspconfig?](https://github.com/neovim/nvim-lspconfig/wiki) 

