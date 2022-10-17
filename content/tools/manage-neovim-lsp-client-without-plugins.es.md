+++
title = "Se puede usar el cliente LSP de neovim sin plugins?"
description = "Vamos a descubrir qué necesitamos para usar el cliente LSP de neovim en nuestros proyectos"
date = 2022-06-09
updated = 2022-10-16
lang = "es"
[taxonomies]
tags = ["vim", "neovim", "shell"]
[extra]
shared = [
  ["dev.to", "https://dev.to/vonheikemen/se-puede-usar-el-cliente-lsp-de-neovim-sin-plugins-44ha"],
  ["Hashnode", "https://vonheikemen.hashnode.dev/manage-neovim-lsp-client-without-plugins-es"]
]
+++

Sí se puede. La complejidad de la configuración dependerá del flujo de trabajo que queremos utilizar. Pero si `lua` les resulta fácil de leer podrán entender la "estructura" necesaria para obtener una configuración decente.

## La base

Resulta que sólo necesitamos conocer dos funciones. Una para inicializar el servidor LSP. Una para notificar cada cambio en el texto un archivo.

* `vim.lsp.start_client()`: Crea una instancia de un "cliente" que es la que se encarga de comunicarse con el servidor LSP.

* `vim.lsp.buf_attach_client()`: Envía notificaciones al servidor LSP con cada cambio en el texto.

Este es un ejemplo simple e ineficiente usando un servidor LSP para typescript.

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
  {desc = 'Inicializar tsserver'}
)
```

Con esto creamos el comando `LaunchTsserver`, al ejecutarlo podremos usar el servidor `typescript-language-server` y obtener diagnósticos en el archivo actual.

¿Por qué es ineficiente? Bueno, porque sólo funciona con un solo archivo. Y cada vez que lo ejecutamos estamos creando un nuevo proceso de `typescript-language-server`. Lo ideal sería poder compartir el mismo servidor en todo el proyecto.

## Uso en proyectos

¿Qué nos falta para poder utilizar el servidor LSP de manera eficiente? Necesitamos crear un "autocomando". Debemos indicarle a neovim que queremos vincular una extensión o tipo de archivo con un servidor LSP.

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
  desc = string.format('Vincular servidor: %s', client_name),
  pattern = filetypes,
  callback = buf_attach
})
```

Bien, ahora `nvim_create_autocmd` nos devolverá un id que podremos usar para eliminar el autocomando cuando sea necesario. 

Para eliminar el autocomando necesitaremos la función `nvim_del_autocmd`.

```lua
vim.api.nvim_del_autocmd(autocmd_id)
```

¿Pero en qué momento debemos crear este autocomando? Cuando el servidor termine su proceso de inicialización. Podemos usar la función `on_init` para crearlo y después usamos `on_exit` para borrarlo.

Si aplicamos todo este conocimiento a nuestro ejemplo, la función `launch_tsserver` quedaría de la siguiente manera.

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
      desc = string.format('Vincular servidor: %s', client.name),
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

## ¿Cuándo definimos los atajos?

No podemos olvidarnos de los atajos. ¿Cómo debemos proceder? La documentación dice que podemos usar el evento `LspAttach`, de esta manera podremos definirlos en cualquier lugar de nuestra configuración.

```lua
vim.api.nvim_create_autocmd('LspAttach', {
  desc = 'Acciones LSP',
  callback = function()
    local bufmap = function(mode, lhs, rhs)
      local opts = {buffer = true}
      vim.keymap.set(mode, lhs, rhs, opts)
    end

    -- Muestra información sobre símbolo debajo del cursor
    bufmap('n', 'K', '<cmd>lua vim.lsp.buf.hover()<cr>')

    -- Saltar a definición
    bufmap('n', 'gd', '<cmd>lua vim.lsp.buf.definition()<cr>')

    -- Saltar a declaración
    bufmap('n', 'gD', '<cmd>lua vim.lsp.buf.declaration()<cr>')

    -- Mostrar implementaciones
    bufmap('n', 'gi', '<cmd>lua vim.lsp.buf.implementation()<cr>')

    -- Saltar a definición de tipo
    bufmap('n', 'go', '<cmd>lua vim.lsp.buf.type_definition()<cr>')

    -- Listar referencias
    bufmap('n', 'gr', '<cmd>lua vim.lsp.buf.references()<cr>')

    -- Mostrar argumentos de función
    bufmap('n', '<C-k>', '<cmd>lua vim.lsp.buf.signature_help()<cr>')

    -- Renombrar símbolo
    bufmap('n', '<F2>', '<cmd>lua vim.lsp.buf.rename()<cr>')

    -- Listar "code actions" disponibles en la posición del cursor
    bufmap('n', '<F4>', '<cmd>lua vim.lsp.buf.code_action()<cr>')
    bufmap('x', '<F4>', '<cmd>lua vim.lsp.buf.range_code_action()<cr>')

    -- Mostrar diagnósticos de la línea actual
    bufmap('n', 'gl', '<cmd>lua vim.diagnostic.open_float()<cr>')

    -- Saltar al diagnóstico anterior
    bufmap('n', '[d', '<cmd>lua vim.diagnostic.goto_prev()<cr>')

    -- Saltar al siguiente diagnóstico
    bufmap('n', ']d', '<cmd>lua vim.diagnostic.goto_next()<cr>')
  end
})
```

## Código completo

Si aplicamos todos estos cambios a nuestro ejemplo obtendremos esto.

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
      desc = string.format('Vincular servidor: %s', client.name),
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
  {desc = 'Inicializar tsserver'}
)
```

## ¿Podemos mejorarlo?

Claro. Casi todo el código puede moverse a una función auxiliar, con eso podremos eliminar todo el ruido.

Sólo imaginen algo como esto.

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
  {desc = 'Inicializar tsserver'}
)
```

¿Y qué tiene `make_config`? Bueno... se los dejo de tarea. Tendrán que implementarla ustedes mismos. Ya les mostré todo el código necesario confío en que pueden hacerlo solos.

Si de verdad quieren saber qué haría yo pueden encontrar la respuesta en este repositorio: [VonHeikemen/nvim-lsp-sans-plugins](https://github.com/VonHeikemen/nvim-lsp-sans-plugins)

## Conclusión

Aprendimos suficiente sobre el cliente LSP que viene incluido en neovim para crear una pequeña configuración. Sabemos cómo reutilizar una instancia de servidor LSP en múltiples archivos. Y en el proceso aprendimos un poco sobre los autocomandos. Ya podemos decir que podemos manejar el cliente LSP de neovim.

## Fuentes

* [:help vim.lsp.start_client()](https://neovim.io/doc/user/lsp.html#vim.lsp.start_client()) 
* [:help vim.lsp.buf_attach_client()](https://neovim.io/doc/user/lsp.html#vim.lsp.buf_attach_client()) 
* [What is nvim-lspconfig?](https://github.com/neovim/nvim-lspconfig/wiki) 

