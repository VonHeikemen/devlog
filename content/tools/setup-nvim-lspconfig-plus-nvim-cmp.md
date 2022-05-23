+++
title = "Setup nvim-lspconfig + nvim-cmp"
description = "Let's configure neovim's builtin LSP client with nvim-lspconfig and nvim-cmp"
date = 2022-05-23
lang = "en"
[taxonomies]
tags = ["vim", "neovim", "shell"]
+++

Neovim includes a lua framework that allows the editor to communicate with a language server. What does that mean for us? Means we can have some IDE-like features such as rename variables, smart jump to definition, list references, etc. But of course we need to configure all of this first.

## Wait, why do we need all these plugins?

Out of the box neovim offers the tools to start and query a language server but it doesn't have any opinion on how we should use them. Enter [nvim-lspconfig](https://github.com/neovim/nvim-lspconfig), with it neovim can scan the "root directory" of a project and choose which language server you configured should be initialiazed.

And then we have [nvim-cmp](https://github.com/hrsh7th/nvim-cmp), the autocompletion plugin. I think the completions provided by neovim are good enough but the thing is it requires a fair amount of manual intervention. Modern code editors have all this automated in a way that feels more intuitive, this is what nvim-cmp offers. We can have that smart autocompletion in neovim.

Now you know the why, let's move on to the how.

## Requirements

* Neovim v0.7 or greater.
* An LSP server.
* Install [nvim-lspconfig](https://github.com/neovim/nvim-lspconfig).
* Install plugins for autocompletion:
  * [nvim-cmp](https://github.com/hrsh7th/nvim-cmp)
  * [cmp-buffer](https://github.com/hrsh7th/cmp-buffer)
  * [cmp-path](https://github.com/hrsh7th/cmp-path)
  * [cmp_luasnip](https://github.com/saadparwaiz1/cmp_luasnip)
  * [cmp-nvim-lsp](https://github.com/hrsh7th/cmp-nvim-lsp)
  * [LuaSnip](https://github.com/L3MON4D3/LuaSnip)
  * [friendly-snippets](https://github.com/rafamadriz/friendly-snippets)

In `nvim-lspconfig` documentation you'll find instructions to install the language servers it supports: [server_configurations.md](https://github.com/neovim/nvim-lspconfig/blob/master/doc/server_configurations.md)

## LSP Config

First thing you would want to do is declare a global configuration, options that you can share between all the servers.

```lua
local lsp_defaults = {
  flags = {
    debounce_text_changes = 150,
  },
  capabilities = require('cmp_nvim_lsp').update_capabilities(
    vim.lsp.protocol.make_client_capabilities()
  ),
  on_attach = function(client, bufnr)
    vim.api.nvim_exec_autocmds('User', {pattern = 'LspAttached'})
  end
}
```

What do we have here?

* `flags.debounce_text_changes`: Amount of miliseconds neovim will wait to send the next document update notification.

* `capabilities`: The data on this option is send to the server, to announce what features the editor can support. We use `vim.lsp.protocol.make_client_capabilities()` build the default capabilities. Since `nvim-cmp` can add features to neovim we need to send an updated version of these capabilities.

* `on_attach`: Callback function that will be executed when a language server is attached to a buffer. It is recommended that we set our keybindings and commands in this function. Personally, I like to use `nvim_exec_autocmds` to trigger an "event", that way I can declare my keybindings anywhere I want.

Now, `lsp_defaults` it's just a variable, there is nothing special about it. We need to merge this `lspconfig`'s global config. We can find it in `util.default_config`. We will be using `vim.tbl_deep_extend` to merge those two variables in a safe way.

```lua
local lspconfig = require('lspconfig')

lspconfig.util.default_config = vim.tbl_deep_extend(
  'force',
  lspconfig.util.default_config,
  lsp_defaults
)
```

Next step is to call the language servers we have installed. They way we do this with `lspconfig` is by calling the `.setup()` of the language server we want to configure.

How do we know which one we have available? Again, lspconfig's documentation has the answer. You can find the list of valid names using `:help lspconfig-server-configurations`.

For the language `lua` we can use a server called `sumneko_lua`. Install it and then call it in your config like this.

```lua
lspconfig.sumneko_lua.setup({})
```

If you need to customize the its behavior you add some keys to `.setup()`'s argument.

```lua
lspconfig.sumneko_lua.setup({
  single_file_support = true,
  on_attach = function(client, bufnr)
    print('hello')
    lspconfig.util.default_config.on_attach(client, bufnr)
  end
})
```

Notice here that the new `on_attach` calls the `on_attach` on `default_config`. We must do this because the options we set on `.setup()` will override the ones on our global configuration. We could also use `lsp_defaults.on_attach` if its in the scope of the function.

At this point to take advantage of some "LSP features" we need to create some keybindings. This particular setup I'm showing allows us to do it with an autocommand. We could declare our keybindings anywhere in our config.

```lua
vim.api.nvim_create_autocmd('User', {
  pattern = 'LspAttached',
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

Here is the complete code to configure `lspconfig`.

```lua
---
-- Global Config
---

local lsp_defaults = {
  flags = {
    debounce_text_changes = 150,
  },
  capabilities = require('cmp_nvim_lsp').update_capabilities(
    vim.lsp.protocol.make_client_capabilities()
  ),
  on_attach = function(client, bufnr)
    vim.api.nvim_exec_autocmds('User', {pattern = 'LspAttached'})
  end
}

local lspconfig = require('lspconfig')

lspconfig.util.default_config = vim.tbl_deep_extend(
  'force',
  lspconfig.util.default_config,
  lsp_defaults
)

---
-- LSP Servers
---

lspconfig.sumneko_lua.setup({})
```

## Snippets

This is a good time to configure our snippet engine. The next bit is not related to language servers, LSP or whatever. We are just going to load the snippets we have installed.

```lua
require('luasnip.loaders.from_vscode').lazy_load()
```

The `.lazy_load()` function will load any snippet we have in our `runtimepath`. And by that I mean the ones available in `friendly-snippets`.

## Autocompletion

Before we start, `nvim-cmp`'s documentation says we should set `completeopt` with the following values:

```lua
vim.opt.completeopt = {'menu', 'menuone', 'noselect'}
```

To configure nvim-cmp we will use two modules `cmp` and `luasnip`.

```lua
local cmp = require('cmp')
local luasnip = require('luasnip')
```

`cmp` is the one we will use to configure `nvim-cmp`. And this `luansnip`? Well, nvim-cmp doesn't "know" how to expand a snippet, that's why we need it.

We are going to spend some time exploring nvim-cmp's options. I want to explain all the things I do in my personal configuration. Now, let's add a setup function.

```lua
local select_opts = {behavior = cmp.SelectBehavior.Select}

cmp.setup({

})
```

### snippet.expand

Callback function, it receives data from a snippet. This is where `nvim-cmp` expect us to provide a thing that can expand snippets, and we do that with `luasnip.lsp_expand`.

```lua
snippet = {
  expand = function(args)
    luasnip.lsp_expand(args.body)
  end
},
```

### sources

Here we can list all the data sources nvim-cmp will use to populate the completion list.

Each "source" is a lua table that must have a `name` property. That `name` is not the name of the plugin, it's the "id" the plugin used when creating the source. Each source should tell what name they have in their documentation.

Other properties you should be aware of are `priority` and `keyword_length`. `priority` allows nvim-cmp to sort the completion list. If you do not set a `priority` then the order of the sources will determine the priority. Now with `keyword_length` you can control how many characters does are necesary to begin querying the source.

In my personal configuration I have the sources setup this way.

```lua
sources = {
  {name = 'path'},
  {name = 'nvim_lsp', keyword_length = 3},
  {name = 'buffer', keyword_length = 3},
  {name = 'luasnip', keyword_length = 2},
},
```

* `path`: Autocomplete file paths.
* `nvim_lsp`: Shows suggestions based on the response of a language server.
* `buffer`: Suggests words found in the current buffer.
* `luasnip`: Shows available snippets and expands them if they are chosen.

### window.documentation

Controls the appearance and settings for the documentation window. To configure this quickly nvim-cmp offers a preset we can use to add some borders.

```lua
window = {
  documentation = cmp.config.window.bordered()
},
```

### formatting.fields

List of strings that determines the order of the elements in an item.

```lua
formatting = {
  fields = {'menu', 'abbr', 'kind'}
},
```

`abbr` is the content of the suggestion. `kind` is the type of data, this can be text, class, function, etc. Finally, `menu` which apparently is empty by default.

### formatting.format

Callback function that allows us to customize the appearance of the completion menu. A simple example I can show: assign an icon to a field based on the source name. 

```lua
formatting = {
  fields = {'menu', 'abbr', 'kind'},
  format = function(entry, item)
    local menu_icon = {
      nvim_lsp = 'λ',
      luasnip = '⋗',
      buffer = 'Ω',
      path = '🖫',
    }

    item.menu = menu_icon[entry.source.name]
    return item
  end,
},
```

### mapping

List of keybindings. For this we need to declare a list of key/value pairs. Where the value is a function of the `cmp` module. Like this.

```lua
mapping = {
  ['<CR>'] = cmp.mapping.confirm({select = true}),
}
```

In this example `['<CR>']` is the key/shortcut we want to bind. The function on the other side of the assignment is the action we want to execute.

Here is a list of common keybindings:

* Move between completion items.

```lua
['<Up>'] = cmp.mapping.select_prev_item(select_opts),
['<Down>'] = cmp.mapping.select_next_item(select_opts),

['<C-p>'] = cmp.mapping.select_prev_item(select_opts),
['<C-n>'] = cmp.mapping.select_next_item(select_opts),
```

* Scroll text in the documentation window.

```lua
['<C-u>'] = cmp.mapping.scroll_docs(-4),
['<C-f>'] = cmp.mapping.scroll_docs(4),
```

* Cancel completions.

```lua
['<C-e>'] = cmp.mapping.abort(),
```

* Confirm selection.

```lua
['<CR>'] = cmp.mapping.confirm({select = true}),
```

* Jump to the next placeholder in the snippet.

```lua
['<C-d>'] = cmp.mapping(function(fallback)
  if luasnip.jumpable(1) then
    luasnip.jump(1)
  else
    fallback()
  end
end, {'i', 's'}),
```

* Jump to the previous placeholder in the snippet.

```lua
['<C-b>'] = cmp.mapping(function(fallback)
  if luasnip.jumpable(-1) then
    luasnip.jump(-1)
  else
    fallback()
  end
end, {'i', 's'}),
```

* Autocomplete with tab.

If the completion menu is visible, move to the next item. If the line is "empty", insert a `Tab` character. If the cursor is inside a word, trigger the completion menu.

```lua
['<Tab>'] = cmp.mapping(function(fallback)
  local col = vim.fn.col('.') - 1

  if cmp.visible() then
    cmp.select_next_item(select_opts)
  elseif col == 0 or vim.fn.getline('.'):sub(col, col):match('%s') then
    fallback()
  else
    cmp.complete()
  end
end, {'i', 's'}),
```

* If the completion menu is visible, move to the previous item.

```lua
['<S-Tab>'] = cmp.mapping(function(fallback)
  if cmp.visible() then
    cmp.select_prev_item(select_opts)
  else
    fallback()
  end
end, {'i', 's'}),
```

### Complete cmp config

Here is the entire configuration for `nvim-cmp` and `luansnip`.

```lua
vim.opt.completeopt = {'menu', 'menuone', 'noselect'}

require('luasnip.loaders.from_vscode').lazy_load()

local cmp = require('cmp')
local luasnip = require('luasnip')

local select_opts = {behavior = cmp.SelectBehavior.Select}

cmp.setup({
  snippet = {
    expand = function(args)
      luasnip.lsp_expand(args.body)
    end
  },
  sources = {
    {name = 'path'},
    {name = 'nvim_lsp', keyword_length = 3},
    {name = 'buffer', keyword_length = 3},
    {name = 'luasnip', keyword_length = 2},
  },
  window = {
    documentation = cmp.config.window.bordered()
  },
  formatting = {
    fields = {'menu', 'abbr', 'kind'},
    format = function(entry, item)
      local menu_icon = {
        nvim_lsp = 'λ',
        luasnip = '⋗',
        buffer = 'Ω',
        path = '🖫',
      }

      item.menu = menu_icon[entry.source.name]
      return item
    end,
  },
  mapping = {
    ['<Up>'] = cmp.mapping.select_prev_item(select_opts),
    ['<Down>'] = cmp.mapping.select_next_item(select_opts),

    ['<C-p>'] = cmp.mapping.select_prev_item(select_opts),
    ['<C-n>'] = cmp.mapping.select_next_item(select_opts),

    ['<C-u>'] = cmp.mapping.scroll_docs(-4),
    ['<C-f>'] = cmp.mapping.scroll_docs(4),

    ['<C-e>'] = cmp.mapping.abort(),
    ['<CR>'] = cmp.mapping.confirm({select = true}),

    ['<C-d>'] = cmp.mapping(function(fallback)
      if luasnip.jumpable(1) then
        luasnip.jump(1)
      else
        fallback()
      end
    end, {'i', 's'}),

    ['<C-b>'] = cmp.mapping(function(fallback)
      if luasnip.jumpable(-1) then
        luasnip.jump(-1)
      else
        fallback()
      end
    end, {'i', 's'}),

    ['<Tab>'] = cmp.mapping(function(fallback)
      local col = vim.fn.col('.') - 1

      if cmp.visible() then
        cmp.select_next_item(select_opts)
      elseif col == 0 or vim.fn.getline('.'):sub(col, col):match('%s') then
        fallback()
      else
        cmp.complete()
      end
    end, {'i', 's'}),

    ['<S-Tab>'] = cmp.mapping(function(fallback)
      if cmp.visible() then
        cmp.select_prev_item(select_opts)
      else
        fallback()
      end
    end, {'i', 's'}),
  },
})
```

## Bonus content

Technically we are ready. All the code shown so far should get us a functional setup. We can now use our language servers and autocompletion. But there are a couple of customizations we can make.

### Change diagnostic icons

We do this using a function called `sign_define`. Keep in mind this not a lua function, it's vimscript. We access it using the prefix `vim.fn`. To know the details of this function check the documentation: `:help sign_define`.

Anyway, this is the code you'll need to change the sign icons.

```lua
local sign = function(opts)
  vim.fn.sign_define(opts.name, {
    texthl = opts.name,
    text = opts.text,
    numhl = ''
  })
end

sign({name = 'DiagnosticSignError', text = '✘'})
sign({name = 'DiagnosticSignWarn', text = '▲'})
sign({name = 'DiagnosticSignHint', text = '⚑'})
sign({name = 'DiagnosticSignInfo', text = ''})
```

### Diagnostics config

This time we must use the `vim.diagnostic` module, specifically the `.config()` function. It takes a lua table as an argument, and these are the defaults.

```lua
{
  virtual_text = true,
  signs = true,
  update_in_insert = false,
  underline = true,
  severity_sort = false,
  float = true,
}
```

* `virtual_text`: Show diagnostic message using virtual text.

* `signs`: Show a sign next to the line with a diagnostic.

* `update_in_insert`: Update diagnostics while editing in insert mode.

* `underline`: Use an underline to show a diagnostic location.

* `severity_sort`: Order diagnostics by severity.

* `float`: Show diagnostic messages in floating windows.

Each one of these option can be either a boolean or a lua table. You can find more details about them in the documentation: `:help vim.diagnostic.config()`.

I prefer less distracting diagnostics. This is the setup I use.

```lua
vim.diagnostic.config({
  virtual_text = false,
  severity_sort = true,
  float = {
    border = 'rounded',
    source = 'always',
    header = '',
    prefix = '',
  },
})
```

### Help windows with borders

There are two lsp methods using floating windows: `vim.lsp.buf.hover()` and `vim.lsp.buf.signature_help()`. By default these windows don't have any style, but we can change by modifying the associated "handler" of each method. To achieve this we need to use `vim.lsp.with()`.

I don't want to bore you with the details so let me just show you the code.

```lua
vim.lsp.handlers['textDocument/hover'] = vim.lsp.with(
  vim.lsp.handlers.hover,
  {border = 'rounded'}
)

vim.lsp.handlers['textDocument/signatureHelp'] = vim.lsp.with(
  vim.lsp.handlers.signature_help,
  {border = 'rounded'}
)
```

### Install language servers "locally"

Sooner or later you are going to find out about this plugin: [nvim-lsp-installer](https://github.com/williamboman/nvim-lsp-installer/). With it you can manage your language servers using neovim. You'll be able to install, update and remove language server.

But the servers you install using this method will not be available globally. They are installed in the "data folder" of neovim. This means you'll have to setup `nvim-lsp-installer` before using `nvim-lspconfig`. You'll use this function.

```lua
require('nvim-lsp-installer').setup({})
```

After you configure `nvim-lsp-installer` you can continue using `lspconfig` like always. Pretend like `nvim-lsp-installer` doesn't even exists.

Here is an example usage.

```lua
require('nvim-lsp-installer').setup({})

local lsp_defaults = {
  flags = {
    debounce_text_changes = 150,
  },
  capabilities = require('cmp_nvim_lsp').update_capabilities(
    vim.lsp.protocol.make_client_capabilities()
  ),
  on_attach = function(client, bufnr)
    vim.api.nvim_exec_autocmds('User', {pattern = 'LspAttached'})
  end
}

local lspconfig = require('lspconfig')

lspconfig.util.default_config = vim.tbl_deep_extend(
  'force',
  lspconfig.util.default_config,
  lsp_defaults
)

lspconfig.sumneko_lua.setup({})
```

Right now this is all I have to say about `nvim-lsp-installer`. If you want to know more about you'll have read its documentation.

## Conclusion

We've learned the things `nvim-lspconfig` can do for us. We figure out how to make a global config for our servers. We setup a handful of lsp functions in our keybindings. Basically, we know everything we need to take advantage of the cool features a language server can provide. 

We had the chance to assemble a configuration for `nvim-cmp` from scratch. We explored some common options step by step.

Finally, we made some additional customizations to diagnostics. Learned a little something about configuring lsp handlers, specifically the ones who use floating windows. And we took a brief look at method to install language servers locally.

You can find all the configuration code here: [nvim-lspconfig + nvim-cmp](https://gist.github.com/VonHeikemen/8fc2aa6da030757a5612393d0ae060bd)

## Sources

* [:help lsp.txt](https://neovim.io/doc/user/lsp.html)
* [:help lspconfig-global-defaults](https://github.com/neovim/nvim-lspconfig/blob/9e6bcf5a8915e8423d5cc7f82c5069c11272184d/doc/lspconfig.txt#L240)
* [:help lspconfig-keybindings](https://github.com/neovim/nvim-lspconfig/blob/9e6bcf5a8915e8423d5cc7f82c5069c11272184d/doc/lspconfig.txt#L481)
* [:help cmp-config](https://github.com/hrsh7th/nvim-cmp/blob/6e1e3865158f340d6cd3936937eb56947b5a90f9/doc/cmp.txt#L360)
* [:help cmp-mapping](https://github.com/hrsh7th/nvim-cmp/blob/6e1e3865158f340d6cd3936937eb56947b5a90f9/doc/cmp.txt#L227)
* [:help vim.diagnostic.config()](https://neovim.io/doc/user/diagnostic.html#vim.diagnostic.config())

