+++
title = "Setup nvim-lspconfig + nvim-cmp"
description = "Let's configure Neovim's builtin LSP client with nvim-lspconfig and nvim-cmp"
date = 2022-05-23
updated = 2024-10-05
lang = "en"
[taxonomies]
tags = ["vim", "neovim", "shell"]
[extra]
shared = [
  ["dev.to", "https://dev.to/vonheikemen/neovim-lsp-setup-nvim-lspconfig-nvim-cmp-4k8e"],
  ["Hashnode", "https://vonheikemen.hashnode.dev/setup-nvim-lspconfig-nvim-cmp"]
]
+++

Neovim includes a lua framework that allows the editor to communicate with a language server. What does that mean for us? Means we can have some IDE-like features such as rename variables, smart jump to definition, list references, etc. But of course we need to configure all of this first.

## Wait, why do we need all these plugins?

Out of the box Neovim offers the tools to start and query a language server but it doesn't have any opinion on how we should use them. Enter [nvim-lspconfig](https://github.com/neovim/nvim-lspconfig), with it Neovim can scan the "root directory" of a project and choose which language server you configured should be initialiazed.

And then we have [nvim-cmp](https://github.com/hrsh7th/nvim-cmp), the autocompletion plugin. I think the completions provided by Neovim are good enough but the thing is it requires a fair amount of manual intervention. Modern code editors have all this automated in a way that feels more intuitive, this is what nvim-cmp offers. We can have that smart autocompletion in Neovim.

Now you know the why, let's move on to the how.

## Requirements

* Neovim v0.8 or greater is recommended. v0.7 can work too, with a few tweaks.
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

In `nvim-lspconfig` documentation you'll find instructions to install the language servers it supports: [configs.md](https://github.com/neovim/nvim-lspconfig/blob/master/doc/configs.md)

## LSP Config

What we need to do here is use the module `lspconfig` and call the `.setup()` of the language server we want to configure.

How do we know which language servers are supported? lspconfig's documentation has the answer. You can find the list of valid names using `:help lspconfig-all`.

For the language `lua` we can use a server called `lua_ls`. Install it and then call it in your config like this.

```lua
local lspconfig = require('lspconfig')

lspconfig.lua_ls.setup({})
```

There is one thing you need to keep in mind. We are going to use nvim-cmp to handle the autocompletion. So, we have to modify an option called `capabilities` in every language server we use. This option tells the language server the features Neovim supports.

We are going to tell the language server what features nvim-cmp adds to Neovim. For this we call the module `cmp_nvim_lsp` and get the default capabilities.

```lua
local lsp_capabilities = require('cmp_nvim_lsp').default_capabilities()
```

Once we have this new variable `lsp_capabilities` we can add it to our setup.

```lua
local lspconfig = require('lspconfig')
local lsp_capabilities = require('cmp_nvim_lsp').default_capabilities()

lspconfig.lua_ls.setup({
  capabilities = lsp_capabilities,
})
```

If want to know the available options in `.setup()` checkout the help page using `:help lspconfig-setup`.

At this point to take advantage of some "LSP features" we need to create some keybindings. Neovim will emit the event `LspAttach` each time a language server is attached to a buffer, this will give us the opportunity to create our keybindings. Here is an example.

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
    bufmap('n', 'gs', '<cmd>lua vim.lsp.buf.signature_help()<cr>')

    -- Renames all references to the symbol under the cursor
    bufmap('n', '<F2>', '<cmd>lua vim.lsp.buf.rename()<cr>')

    -- Selects a code action available at the current cursor position
    bufmap('n', '<F4>', '<cmd>lua vim.lsp.buf.code_action()<cr>')

    -- Show diagnostics in a floating window
    bufmap('n', 'gl', '<cmd>lua vim.diagnostic.open_float()<cr>')

    -- Move to the previous diagnostic
    bufmap('n', '[d', '<cmd>lua vim.diagnostic.goto_prev()<cr>')

    -- Move to the next diagnostic
    bufmap('n', ']d', '<cmd>lua vim.diagnostic.goto_next()<cr>')
  end
})
```

This autocommand can live anywhere in our configuration. And if we wanted to, we could write it in vimscript (I'm not doing that here).

> Note: if you are using Neovim v0.7.2 or lower you'll need to create a function and add it to the `on_attach` option of each language server. See `:help lspconfig-keybindings` for an example.

So, here is the complete code to configure one language server using `lspconfig`.

```lua
local lspconfig = require('lspconfig')
local lsp_capabilities = require('cmp_nvim_lsp').default_capabilities()

lspconfig.lua_ls.setup({
  capabilities = lsp_capabilities,
})
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

We are going to spend some time exploring nvim-cmp's options. I want to explain all the things I do in my personal configuration. For now let's just add the setup function.

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

Other properties you should be aware of are `priority` and `keyword_length`. `priority` allows nvim-cmp to sort the completion list. If you do not set a `priority` then the order of the sources will determine the priority. Now with `keyword_length` you can control how many characters are necesary to begin querying the source.

In my personal configuration I have the sources setup this way.

```lua
sources = {
  {name = 'path'},
  {name = 'nvim_lsp', keyword_length = 1},
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
  ['<CR>'] = cmp.mapping.confirm({select = false}),
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
['<C-d>'] = cmp.mapping.scroll_docs(4),
```

* Cancel completion.

```lua
['<C-e>'] = cmp.mapping.abort(),
```

* Confirm selection.

```lua
['<C-y>'] = cmp.mapping.confirm({select = true}),
['<CR>'] = cmp.mapping.confirm({select = false}),
```

* Jump to the next placeholder in the snippet.

```lua
['<C-f>'] = cmp.mapping(function(fallback)
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
    {name = 'nvim_lsp', keyword_length = 1},
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
    ['<C-d>'] = cmp.mapping.scroll_docs(4),

    ['<C-e>'] = cmp.mapping.abort(),
    ['<C-y>'] = cmp.mapping.confirm({select = true}),
    ['<CR>'] = cmp.mapping.confirm({select = false}),

    ['<C-f>'] = cmp.mapping(function(fallback)
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

Technically we are ready. All the code shown so far should get us a functional setup. We can now use our language servers and autocompletion. But there are still a couple of customizations we can make.

### Change diagnostic icons

We do this using a function called `sign_define`. Keep in mind this is not a lua function, it's vimscript. We access it using the prefix `vim.fn`. To know the details of this function check the documentation: `:help sign_define`.

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
sign({name = 'DiagnosticSignInfo', text = '»'})
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
  },
})
```

### Help windows with borders

There are two LSP methods that use floating windows: `vim.lsp.buf.hover()` and `vim.lsp.buf.signature_help()`. By default these windows don't have any style, but we can change that by modifying the associated "handler" of each method. To achieve this we need to use `vim.lsp.with()`.

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

Sooner or later you are going to find out about this plugin: [mason.nvim](https://github.com/williamboman/mason.nvim). With it you can manage your language servers using Neovim. You'll be able to install, update and remove language server.

But the servers you install using this method will not be available globally. They are installed in the "data folder" of Neovim. Now, some servers work fine in this setup others require extra configurations so it's recommended that you add [mason-lspconfig](https://github.com/williamboman/mason-lspconfig.nvim). 

Once you have both plugins you should setup mason.nvim using these functions.

```lua
require('mason').setup()
require('mason-lspconfig').setup()
```

After doing that you should use `lspconfig` like you usually do. Pretend mason.nvim doesn't even exists.

Here is an example usage.

```lua
require('mason').setup()
require('mason-lspconfig').setup()

local lspconfig = require('lspconfig')
local lsp_capabilities = require('cmp_nvim_lsp').default_capabilities()

lspconfig.lua_ls.setup({
  capabilities = lsp_capabilities,
})
```

Right now this is all I have to say about `mason.nvim`. If you want to know more about it you'll have read their documentation.

## Conclusion

We've learned the things `nvim-lspconfig` can do for us. We figure out how to make a global config for our servers. We setup a handful of LSP functions in our keybindings. Basically, we know everything we need to take advantage of the cool features a language server can provide. 

We had the chance to assemble a configuration for `nvim-cmp` from scratch. We explored some common options step by step.

Finally, we made some additional customizations to diagnostics. Learned a little something about configuring LSP handlers, specifically the ones using floating windows. And we took a brief look at a method to install language servers locally.

You can find all the configuration code here: [nvim-lspconfig + nvim-cmp](https://gist.github.com/VonHeikemen/8fc2aa6da030757a5612393d0ae060bd). 

And here you can find a fully functional Neovim setup: [nvim-starter - branch: 03-lsp](https://github.com/VonHeikemen/nvim-starter/tree/03-lsp).

## Sources

* [:help lsp.txt](https://neovim.io/doc/user/lsp.html)
* [:help lspconfig-global-defaults](https://github.com/neovim/nvim-lspconfig/blob/9e6bcf5a8915e8423d5cc7f82c5069c11272184d/doc/lspconfig.txt#L240)
* [:help lspconfig-keybindings](https://github.com/neovim/nvim-lspconfig/blob/9e6bcf5a8915e8423d5cc7f82c5069c11272184d/doc/lspconfig.txt#L481)
* [:help cmp-config](https://github.com/hrsh7th/nvim-cmp/blob/6e1e3865158f340d6cd3936937eb56947b5a90f9/doc/cmp.txt#L360)
* [:help cmp-mapping](https://github.com/hrsh7th/nvim-cmp/blob/6e1e3865158f340d6cd3936937eb56947b5a90f9/doc/cmp.txt#L227)
* [:help vim.diagnostic.config()](https://neovim.io/doc/user/diagnostic.html#vim.diagnostic.config())

