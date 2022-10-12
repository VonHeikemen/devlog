+++
title = "Configurando nvim-lspconfig + nvim-cmp"
description = "Usando nvim-lspconfig y nvim-cmp para configurar el cliente LSP de neovim"
date = 2022-05-21
updated = 2022-07-31
lang = "es"
[taxonomies]
tags = ["vim", "neovim", "shell"]
[extra]
shared = [
  ["dev.to", "https://dev.to/vonheikemen/neovim-lsp-configurando-nvim-lspconfig-nvim-cmp-1m2g"],
  ["Hashnode", "https://vonheikemen.hashnode.dev/setup-nvim-lspconfig-nvim-cmp-es"]
]
+++

Dentro de neovim se encuentra un "framework" que le permite al editor comunicarse con un servidor LSP. ¿Qué significa eso? Quiere decir que, con la configuración correcta, tendremos acceso a funcionalidades como renombrar una variable, saltar a una definición, listar referencias, etc. Características que podrían encontrar en un IDE.

## ¿Por qué necesitamos plugins?

Por defecto neovim nos ofrece las herramientas pero no tiene ninguna opinión de cómo debemos usarlas. Es aquí donde entra [nvim-lspconfig](https://github.com/neovim/nvim-lspconfig), este plugin escanea la "carpeta raíz" de nuestro proyecto e intenta determinar qué servidor LSP necesitamos. Pero claro, primero debemos tener instalados los servidores en nuestro sistema y luego debemos configurarlos dentro de neovim.

Y luego tenemos [nvim-cmp](https://github.com/hrsh7th/nvim-cmp), un plugin de autocompletado. ¿De verdad lo necesitamos? En realidad no. En mi opinión el autocompletado de neovim es bastante capaz. ¿Cuál es el problema? Digamos que requiere de mucha intervención manual. Los editores de código modernos nos tienen acostumbrados a un flujo donde todo ocurre de manera automática. `nvim-cmp` nos permite tener ese autocompletado cómodo e inteligente.

Ahora que sabemos el porqué de las cosas pongamos manos a la obra.

## Requisitos

* Neovim v0.7 o mayor.
* Un servidor LSP.
* Instalar [nvim-lspconfig](https://github.com/neovim/nvim-lspconfig).
* Instalar plugins para el autocompletado:
  * [nvim-cmp](https://github.com/hrsh7th/nvim-cmp)
  * [cmp-buffer](https://github.com/hrsh7th/cmp-buffer)
  * [cmp-path](https://github.com/hrsh7th/cmp-path)
  * [cmp_luasnip](https://github.com/saadparwaiz1/cmp_luasnip)
  * [cmp-nvim-lsp](https://github.com/hrsh7th/cmp-nvim-lsp)
  * [LuaSnip](https://github.com/L3MON4D3/LuaSnip)
  * [friendly-snippets](https://github.com/rafamadriz/friendly-snippets)

En la documentación de `nvim-lspconfig` pueden encontrar instrucciones de cómo instalar algunos servidores: [server_configurations.md](https://github.com/neovim/nvim-lspconfig/blob/master/doc/server_configurations.md)

## Configuración LSP

Lo primero que haremos será crear una configuración global, una serie de opciones que todos nuestros servidores compartirán.

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

¿Qué tenemos aquí?

* `flags.debounce_text_changes`: Es la cantidad de milisegundos que neovim esperará para notificar cambios en el documento al servidor.

* `capabilities`: Esta opción anuncia las funcionalidades que neovim puede manejar. `vim.lsp.protocol.make_client_capabilities()` son las opciones que neovim soporta por defecto. Ya que `nvim-cmp` agrega más funcionalidades al editor necesitamos enviar una versión actualizada de estas "capacidades".

* `on_attach`: Función que se ejecutará cuando un servidor se vincule a un buffer. Por lo general aquí creamos comandos y atajos de teclado vinculados a funciones al servidor LSP. Personalmente me gusta usar `nvim_exec_autocmds` para enviar un "evento", así puedo tener mis atajos de LSP junto a mis atajos normales.

Ahora bien, `lsp_defaults` sólo es una variable, no tiene nada de especial. Necesitamos mezclarlo con las opciones globales de `lspconfig`. Estas opciones globales las podemos encontrar en `util.default_config`. Vamos a usar la función `vim.tbl_deep_extend` para mezclar las dos variables de manera segura.

```lua
local lspconfig = require('lspconfig')

lspconfig.util.default_config = vim.tbl_deep_extend(
  'force',
  lspconfig.util.default_config,
  lsp_defaults
)
```

El siguiente paso será llamar los servidores que tenemos instalados en nuestro sistema. Con `lspconfig` debemos llamar la función `.setup()` del servidor que queremos configurar.

¿Pero cómo sabemos qué servidores tenemos disponibles? En la documentación de `lspconfig` pueden encontrar la lista completa, pueden navegar a ella usando el comando `:help lspconfig-server-configurations`.

Para en lenguaje `lua` tenemos disponible el servidor `sumneko_lua`. Luego de instalarlo en nuestro sistema podemos configurarlo de la siguiente manera.

```lua
lspconfig.sumneko_lua.setup({})
```

Si necesitan agregar más opciones a un servidor deben agregar propiedades al argumento de `.setup()`.

```lua
lspconfig.sumneko_lua.setup({
  single_file_support = true,
  on_attach = function(client, bufnr)
    print('hola')
    lspconfig.util.default_config.on_attach(client, bufnr)
  end
})
```

Noten que en este ejemplo el nuevo `on_attach` está llamando al `on_attach` de `default_config`. Debemos hacer esto porque las opciones que declaramos en `.setup()` sobreescriben lo que tenemos en la configuración global. También podríamos usar `lsp_defaults.on_attach` si está disponible en el ámbito de la función.

Ahora lo más importante, los atajos de teclado. En esta configuración en particular podemos declararlos con un "autocomando". Pueden colocar esto en cualquier parte de su configuración.

```lua
vim.api.nvim_create_autocmd('User', {
  pattern = 'LspAttached',
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

Aquí les muestro la configuración completa necesaria para `lspconfig`.

```lua
---
-- Configuración global
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
-- Servidores LSP
---

lspconfig.sumneko_lua.setup({})
```

## Snippets

Interrumpimos la programación habitual para configurar nuestro motor de snippets. Esta parte no está relacionada con el cliente LSP.

```lua
require('luasnip.loaders.from_vscode').lazy_load()
```

La función `.lazy_load()` carga las colecciones de snippets que tengamos en el `runtimepath`. Es aquí cuando se cargan los snippets de `friendly-snippets`.

## Autocompletado

Para empezar, la documentación de `nvim-cmp` recomienda que configuremos la `completeopt` de la siguiente manera.

```lua
vim.opt.completeopt = {'menu', 'menuone', 'noselect'}
```

Para configurar nvim-cmp necesitamos dos módulos `cmp` y `luasnip`.

```lua
local cmp = require('cmp')
local luasnip = require('luasnip')
```

`cmp` es el módulo de nvim-cmp. ¿Qué hay de `luasnip`? Lo que pasa es que nvim-cmp no "sabe" como expandir un snippet, por eso necesitamos luasnip.

Vamos a pasar un buen rato configurando nvim-cmp. Quiero explicarles las opciones que yo uso en mi configuración, pero primero vamos a agregar la función `cmp.setup`.

```lua
local select_opts = {behavior = cmp.SelectBehavior.Select}

cmp.setup({

})
```

### snippet.expand

Es una función que recibe la información de un snippet. Aquí es donde `nvim-cmp` nos da la oportunidad de expandir los snippets con el plugin que queramos. En nuestro caso tenemos `luasnip`.

```lua
snippet = {
  expand = function(args)
    luasnip.lsp_expand(args.body)
  end
},
```

### sources

Aquí podemos listar los plugins que nvim-cmp usará para obtener datos.

Cada "fuente" es una tabla que debe tener la propiedad `name`, este nombre no se refiere al nombre del plugin sino al nombre con el que se registró la fuente en `nvim-cmp`. La documentación de cada fuente debería mostrarles este nombre.

Otras propiedades que deben tener en cuenta son `priority` y `keyword_length`. Con `priority` podemos elegir el orden en el que aparecen las sugerencias en la lista de autocompletado. Si no se especifica la propiedad `priority` entonces el orden de las fuentes dicta la prioridad. Con `keyword_length` se controla la cantidad de caracteres que se necesitan para empezar a ver datos de la fuente.

En mi configuración personal tengo las fuentes configuradas de esta manera.

```lua
sources = {
  {name = 'path'},
  {name = 'nvim_lsp', keyword_length = 3},
  {name = 'buffer', keyword_length = 3},
  {name = 'luasnip', keyword_length = 2},
},
```

* `path`: Autocompleta rutas de archivos.
* `nvim_lsp`: Muestra sugerencias basadas en la respuesta de un servidor LSP.
* `buffer`: Sugiere palabras que se encuentra en el archivo actual.
* `luasnip`: Muestra los snippets cargados. Si elegimos un snippet lo expande.

### window.documentation

Controla la apariencia de la ventana donde se muestra la documentación de un item. nvim-cmp ofrece una configuración que podemos usar para obtener una ventana con bordes.

```lua
window = {
  documentation = cmp.config.window.bordered()
},
```

### formatting.fields

Lista que controla el orden en el que aparecen los elementos de un item.

```lua
formatting = {
  fields = {'menu', 'abbr', 'kind'}
},
```

`abbr` es el contenido de la sugerencia. `kind` es el tipo de dato, puede ser texto, una clase, una función, etc. `menu` está vacío por defecto.

### formatting.format

Función que determina el formato del item. Un ejemplo simple que puedo mostrarles es cómo asignarle un ícono al "field" `menu` basado en el nombre de la fuente.

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

Lista de atajos de teclado. Aquí necesitamos declarar una lista de pares propiedad/valor. Donde el `valor` será una función del módulo `cmp`.

```lua
mapping = {
  ['<CR>'] = cmp.mapping.confirm({select = true}),
}
```

En este ejemplo `['<CR>']` es el atajo que queremos crear. La función del otro lado de la asignación es la acción que queremos ejecutar.

Estas son algunas acciones comunes:

* Navegar entre las sugerencias.

```lua
['<Up>'] = cmp.mapping.select_prev_item(select_opts),
['<Down>'] = cmp.mapping.select_next_item(select_opts),

['<C-p>'] = cmp.mapping.select_prev_item(select_opts),
['<C-n>'] = cmp.mapping.select_next_item(select_opts),
```

* Desplaza el texto de la ventana de documentación.

```lua
['<C-u>'] = cmp.mapping.scroll_docs(-4),
['<C-f>'] = cmp.mapping.scroll_docs(4),
```

* Cancelar el autocompletado.

```lua
['<C-e>'] = cmp.mapping.abort(),
```

* Confirma la selección.

```lua
['<CR>'] = cmp.mapping.confirm({select = true}),
```

* Salta al próximo placeholder de un snippet.

```lua
['<C-d>'] = cmp.mapping(function(fallback)
  if luasnip.jumpable(1) then
    luasnip.jump(1)
  else
    fallback()
  end
end, {'i', 's'}),
```

* Salta al placeholder anterior de un snippet.

```lua
['<C-b>'] = cmp.mapping(function(fallback)
  if luasnip.jumpable(-1) then
    luasnip.jump(-1)
  else
    fallback()
  end
end, {'i', 's'}),
```

* Autocompletado con tab.

Si la lista de sugerencias es visible, navega al siguiente item. Si la línea está "vacía", inserta el caracter `Tab`. Si el cursor está en medio de una palabra, activa el autocompletado.

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

* Si la lista de sugerencias es visible, navega al item anterior.

```lua
['<S-Tab>'] = cmp.mapping(function(fallback)
  if cmp.visible() then
    cmp.select_prev_item(select_opts)
  else
    fallback()
  end
end, {'i', 's'}),
```

### Configuración cmp

He aquí el código completo de la configuración de `nvim-cmp` y `luasnip`.

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

## Contenido extra

Técnicamente ya están listos. Si agregaron todo el código que les mostré en su configuración ya deben tener el servidor LSP y el autocompletado funcionando. Pero aún cosas que podemos hacer para mejorar nuestra experiencia.

### Personalizando los íconos de diagnóstico

Esto lo logramos con la función `sign_define`. Esta no es una función de lua, es de vimscript, quiere decir que accedemos a ella usando el prefijo `vim.fn`. Para saber los detalles de esta función usen el comando `:help sign_define`.

En fin, con esto podremos cambiar los "signos de diagnóstico".

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

### Configuración global de diagnósticos

Para esto debemos usar el módulo `vim.diagnostic`, específicamente la función `.config()`. Esta función acepta una tabla como argumento, estas son sus propiedades.

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

* `virtual_text`: Muestra mensaje de diagnóstico con un "texto virtual" al final de la línea.

* `signs`: Mostrar un "signo" en la línea donde hay un diagnóstico presente.

* `update_in_insert`: Actualizar los diagnósticos mientras se edita el documento en modo de inserción.

* `underline`: Subrayar la localización de un diagnóstico.

* `severity_sort`: Ordenar los diagnósticos de acuerdo a su prioridad.

* `float`: Habilitar ventanas flotantes para mostrar los mensajes de diagnósticos.

Cada una estas opciones aceptar una tabla de opciones para personalizar aún más su comportamiento. Pueden encontrar todos esos detalles en la documentación: `:help vim.diagnostic.config()`.

Personalmente prefiero diagnósticos menos llamativos. Uso esta configuración.

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

### Bordes en ventanas de ayuda

Hay dos métodos que muestran ventanas flotantes con información: `vim.lsp.buf.hover()` y `vim.lsp.buf.signature_help()`. Por defecto estas ventanas no tienen ningún estilo, pero podemos cambiar eso si modificamos la configuración del "handler" asociado a estas funciones. Para estos casos tenemos que usar `vim.lsp.with()`.

No quiero aburrirlos con los detalles así que aquí les dejo el código.

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

### Instalar servidores LSP de manera "local"

Tarde o temprano se van a encontrar con este plugin: [mason.nvim](https://github.com/williamboman/mason.nvim). Esto les permite manejar completamente sus servidores LSP dentro de neovim. Podrán instalar, actualizar y eliminar servidores usando neovim.

Los servidores que instalen con este método no estarán disponibles de manera global, se instalarán en la carpeta "data" de neovim. Ahora bien, algunos servidores al ser instalados con este método requieren de configuraciones extra, es por eso que también necesitamos usar [mason-lspconfig](https://github.com/williamboman/mason-lspconfig.nvim).

Una vez que tengan todos los plugins necesarios deben configurar mason.nvim de esta manera.

```lua
require('mason').setup()
require('mason-lspconfig').setup()
```

Después de invocar estas funciones pueden usar `lspconfig` como si nada. Pueden fingir que mason.nvim no existe.

Aquí les dejo un ejemplo de uso.

```lua
require('mason').setup()
require('mason-lspconfig').setup()

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

En está ocasión es todo lo que voy a decir sobre `mason.nvim`. Si quieren saber más detalles deben revisar su documentación.

## Conclusión

Han aprendido porque necesitamos nvim-lspconfig. Saben cómo crear una configuración global para todos sus servidores. Conocen las funciones más comúnes para sus atajos de teclado. Todo lo necesario para aprovechar las bondades que puede darles un servidor LSP.

Tuvimos la oportunidad de armar una configuración de `nvim-cmp` desde cero. Explorando paso a paso las opciones más utilizadas.

Finalmente vimos algunas opciones que nos permiten modificar la presentación de los diagnósticos (mensajes de error, advertencia, etc). Aprendimos a modificar los handlers que muestran ventanas flotantes. Y, dimos un vistazo a un método para instalar servidores de manera local.

Pueden encontrar todo el código de configuración aquí: [nvim-lspconfig + nvim-cmp](https://gist.github.com/VonHeikemen/8fc2aa6da030757a5612393d0ae060bd).

Y aquí les dejo una configuración de Neovim completamente funcional: [nvim-starter - branch: 03-lsp](https://github.com/VonHeikemen/nvim-starter/tree/03-lsp).

## Fuentes

* [:help lsp.txt](https://neovim.io/doc/user/lsp.html)
* [:help lspconfig-global-defaults](https://github.com/neovim/nvim-lspconfig/blob/9e6bcf5a8915e8423d5cc7f82c5069c11272184d/doc/lspconfig.txt#L240)
* [:help lspconfig-keybindings](https://github.com/neovim/nvim-lspconfig/blob/9e6bcf5a8915e8423d5cc7f82c5069c11272184d/doc/lspconfig.txt#L481)
* [:help cmp-config](https://github.com/hrsh7th/nvim-cmp/blob/6e1e3865158f340d6cd3936937eb56947b5a90f9/doc/cmp.txt#L360)
* [:help cmp-mapping](https://github.com/hrsh7th/nvim-cmp/blob/6e1e3865158f340d6cd3936937eb56947b5a90f9/doc/cmp.txt#L227)
* [:help vim.diagnostic.config()](https://neovim.io/doc/user/diagnostic.html#vim.diagnostic.config())

