+++
title = "Se puede usar el cliente LSP de neovim sin plugins?"
description = "Vamos a descubrir qué necesitamos para usar el cliente LSP de neovim en nuestros proyectos"
date = 2022-06-09
lang = "es"
[taxonomies]
tags = ["vim", "neovim", "shell"]
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
  'typescript',
  'javascript',
  'typescriptreact',
  'javascriptreact',
  'typescript.tsx',
  'javascript.jsx'
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
    'typescript',
    'javascript',
    'typescriptreact',
    'javascriptreact',
    'typescript.tsx',
    'javascript.jsx'
  }

  local config = {
    cmd = {'typescript-language-server', '--stdio'},
    name = 'tsserver',
    root_dir = vim.fn.getcwd(),
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

## Configuraciones de servidor

Nuestro ejemplo aún no está completo. La mayoría de los servidores LSP tienen una serie de opciones únicas, para habilitar funcionalidades o modificar algún comportamiento. Justo ahora no le estamos enviando esa información al servidor.

Después de que el servidor esté listo podemos enviarle una "notificación" con los datos que queremos. Entonces, en la función `on_init` agregamos esto.

```lua
if results.offsetEncoding then
  client.offset_encoding = results.offsetEncoding
end

if client.config.settings then
  client.notify('workspace/didChangeConfiguration', {
    settings = client.config.settings
  })
end
```

La documentación de neovim sugiere que debemos configurar la codificación del cliente antes de enviar cualquier petición o notificación. Entonces si el servidor nos devuelve esta información en `results` lo usamos en el cliente.

Si la instancia del cliente tiene la propiedad `settings` nos aseguramos de enviarla al servidor. Vale la pena mencionar que `client.config` es una copia del argumento que le pasamos a la función `vim.lsp.start_client()`.

## ¿Cuándo definimos los atajos?

No podemos olvidarnos de los atajos. ¿Cómo debemos proceder? La documentación nos sugiere que usemos la función `on_attach` para definir los atajos y cualquier otro comando. Pero yo tengo otra sugerencia, yo digo que usemos un "evento de usuario" así podemos definir los atajos en cualquier lugar de nuestra configuración.

```lua
config.on_attach = function(client, bufnr)
  vim.api.nvim_exec_autocmds('User', {pattern = 'LspAttached'})
end
```

Ahora podemos definir nuestros atajos en otro lugar.

```lua
vim.api.nvim_create_autocmd('User', {
  pattern = 'LspAttached',
  desc = 'Acciones LSP',
  callback = function()
    local bufmap = function(mode, lhs, rhs)
      local opts = {buffer = true}
      vim.keymap.set(mode, lhs, rhs, opts)
    end

    -- Configura sugerencias del servidor
    -- se activan presionando Ctrl-x + Ctrl-o en modo de inserción
    vim.bo.omnifunc = 'v:lua.vim.lsp.omnifunc'

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
    'typescript',
    'javascript',
    'typescriptreact',
    'javascriptreact',
    'typescript.tsx',
    'javascript.jsx'
  }

  local config = {
    cmd = {'typescript-language-server', '--stdio'},
    name = 'tsserver',
    root_dir = vim.fn.getcwd(),
    capabilities = vim.lsp.protocol.make_client_capabilities(),
  }

  config.on_attach = function(client, bufnr)
    vim.api.nvim_exec_autocmds('User', {pattern = 'LspAttached'})
  end

  config.on_init = function(client, results)
    if results.offsetEncoding then
      client.offset_encoding = results.offsetEncoding
    end

    if client.config.settings then
      client.notify('workspace/didChangeConfiguration', {
        settings = client.config.settings
      })
    end

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
      'typescript',
      'javascript',
      'typescriptreact',
      'javascriptreact',
      'typescript.tsx',
      'javascript.jsx'
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

