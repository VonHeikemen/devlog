+++
title = "Guía de uso del cliente LSP de Neovim" 
description = "Agrega funcionalidades dignas de un IDE a Neovim sin instalar plugins de terceros"
date = 2024-01-13
updated = 2024-07-27
lang = "es"
[taxonomies]
tags = ["neovim", "shell"]
[extra]
shared = []
+++

El título original para este post era "Cómo agregar funcionalidades como las de un IDE a Neovim sin instalar plugins de terceros." Suena más interesante pero tiene demasiadas palabras. Aún así, es básicamente lo que quiero enseñarles.

Okey, no necesitamos plugins de terceros. Pero entonces ¿qué debemos tener?

1. Neovim en su versión v0.8 o mayor
2. Un servidor LSP
3. Paciencia/Energía para escribir un poco de lua por cada servidor LSP que se quiere configurar

Si quieren implementar lo que voy a mostrar aquí en su configuración personal, consideren dedicar un poco de tiempo para aprender lua. Aquí les dejo algunos recursos.

* [Lua crash course (video de 11 min)](https://www.youtube.com/watch?v=NneB6GX1Els)
* [Learn X in Y minutes: Where X = lua](https://learnxinyminutes.com/docs/lua/)
* [Guía oficial de lua para Neovim](https://neovim.io/doc/user/lua-guide.html)

## Todo comienza con el servidor LSP

Un servidor LSP es un programa externo que sigue el [Protocolo de Servidor de Lenguajes](https://microsoft.github.io/language-server-protocol/) (Language Server Protocol). En este se definen los parámetros que puede recibir un servidor LSP, y también cómo debe responder. El propósito de todo esto es que [cualquier herramienta que implemente el protocolo](https://microsoft.github.io/language-server-protocol/implementors/tools/) pueda comunicarse con un servidor LSP.

Entonces, el servidor LSP es el programa que va a analizar nuestro código y le dice a nuestro editor qué debe hacer.

*¿Qué servidores LSP tenemos disponibles?*

La página web del protocolo [tiene una lista de servidores](https://microsoft.github.io/language-server-protocol/implementors/servers/).

### En este caso en particular...

Voy a mostrarles cómo configurar [intelephense](https://intelephense.com/), pero tengan en cuenta que sólo es un ejemplo. Todos los pasos/consejos que compartiré pueden aplicarse a cualquier servidor.

Si quieren usar intelephense deben instalar [NodeJS](https://nodejs.org/en). Si ya tienen NodeJS pueden instalar intelephense usando este comando en su terminal. 

```sh
npm install -g intelephense
```

Para estar seguros de que todo está bien revisen si Neovim reconoce la ruta donde esta instalado intelephense. Ejecuten este comando en Neovim.

```vim
:echo exepath('intelephense')
```

Debería mostrarles la ruta donde está el ejecutable de intelephense. Si no muestra nada quiere decir que algo salió mal durante la instalación.

Si llegaron aquí y no le dieron click al enlace de intelephense: deben saber que es un servidor LSP para php. Si sólo quieren probar el código que voy a mostrar no necesitan instalar el intérprete de php. Sólo necesitan un proyecto de php. Pueden clonar repositorio de [minicli](https://github.com/minicli/minicli). Tiene una buena cantidad de código y no tiene dependencias.

## Uso básico

Antes de empezar a escribir código deberíamos aprender cómo usar el servidor LSP. Lo primero que debemos saber es cómo ejecutarlo. Es decir, cuál es el comando que inicia el servidor. Esta información debería estar en la documentación oficial del servidor.

Si no podemos encontrar la información básica en la documentación del servidor, podemos ir al [repositorio de nvim-lspconfig](https://github.com/neovim/nvim-lspconfig) y buscamos la carpeta que se llama [server_configurations](https://github.com/neovim/nvim-lspconfig/tree/master/lua/lspconfig/server_configurations). Ahí podemos encontrar archivos de lua que contienen la configuración para muchos servidores.

En este momento queremos configurar intelephense, entonces deberíamos revisar el contenido de [intelephense.lua](https://github.com/neovim/nvim-lspconfig/blob/4a69ad6646eaad752901413c9a0dced73dfb562d/lua/lspconfig/server_configurations/intelephense.lua). El código que estamos buscando se encuentra en una propiedad llamada `default_config`. Esto:

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

Aquí la propiedad `cmd` es el comando que necesitamos para iniciar el servidor. `filetypes` es la lista de lenguajes que el servidor puede analizar. Y en un momento les cuento sobre `root_dir`.

## ¿Cúando debe ejecutarse?

En el caso de intelephense sólo lo necesitamos en archivos php, esta es la oportunidad perfecta para usar algo llamado "filetype plugin," que es básicamente un script que Neovim ejecuta después de que determina qué tipo de archivo estamos editando.

¿Cómo creamos un filetype plugin? Primero debemos ir a la carpeta de configuración de Neovim, y ahí debemos crear una carpeta llamada `ftplugin`. Podemos ejecutar este comando en Neovim.

```vim
:call mkdir('./ftplugin', 'p')
```

Dentro de esta carpeta debemos crear un script con el mismo nombre de un tipo de archivo.

Necesitamos un filetype plugin para php entonces debemos crear un archivo llamado `php.lua`.

```vim
:edit ftplugin/php.lua | write
```

En este script es donde vamos a ejecutar la función que "conectará" intelephense con Neovim.

## Directorio raíz

La última pieza del rompecabezas es el `root_dir`. Simplemente debemos decirle al servidor LSP en qué ruta se encuentra el código fuente de nuestro proyecto.

Ahora bien, en nuestro filetype plugin vamos a usar una función llamada [vim.fs.find()](https://neovim.io/doc/user/lua.html#vim.fs.find()). A esta función le vamos a dar una lista de nombres de archivos y nos devolverá una ruta que podemos usar.

¿Pero qué vamos a buscar? Archivos de configuración, los que usualmente colocamos en la carpeta raíz de un proyecto. En proyectos de php normalmente hay un `composer.json`. En proyectos de javascript siempre hay un `package.json`. En rust está el `cargo.toml`. Este es el tipo de información que vamos a darle a `vim.fs.find()` para que nos devuelva la ruta correcta.

Ya podemos empezar a escribir un poco de código. En `php.lua` podemos hacer una prueba con `vim.fs.find()`.

```lua
-- ftplugin/php.lua

local root_files = {'composer.json'}
local paths = vim.fs.find(root_files, {stop = vim.env.HOME})

print(vim.fs.dirname(paths[1]))
```

Por defecto `vim.fs.find()` comenzará a buscar en la carpeta actual y luego comenzará a buscar en la carpeta padre, y seguirá buscando "hacia arriba" hasta encontrar algo. Con el argumento `stop` le decimos que debe detener la búsqueda cuando llegue a nuestra carpeta de usuario (no queremos que el servidor empieze a analizar esta carpeta ni por accidente).

Debido a que `vim.fs.find()` retorna una lista nosotros debemos elegir el primer elemento. Y para asegurarnos de obtener la ruta a una carpeta usamos la función `vim.fs.dirname()`.

Si quieren probar este código pueden navegar a un proyecto de php con un archivo `composer.json` y revisar la ruta que detecta.

## Creando el cliente

Estamos listos para conectar Neovim con el servidor LSP. Ya tenemos todo lo necesario para usar la función [vim.lsp.start()](https://neovim.io/doc/user/lsp.html#vim.lsp.start()).

Esto es lo que deben saber, la primera vez que ejecutamos `vim.lsp.start()` Neovim iniciará nuestro servidor LSP como un proceso externo. Cuando se ejecuta nuevamente Neovim simplemente le enviará información al proceso del servidor LSP.

Entonces, nuestro filetype plugin para php debería tener este código.

```lua
-- ftplugin/php.lua

local root_files = {'composer.json'}
local paths = vim.fs.find(root_files, {stop = vim.env.HOME})
local root_dir = vim.fs.dirname(paths[1])

if root_dir then
  vim.lsp.start({
    cmd = {'intelephense', '--stdio'},
    root_dir = root_dir,
  })
end
```

Con esta configuración ya podremos utilizar algunas funcionalidades. Por ejemplo, si editamos un archivo php y este tiene un error, Neovim nos dirá en qué línea se encuentra el error. Debería mostrarnos algo como esto.

![Fragmento de código php que usa el framework slimphp](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/xcdxdx9kj3xh90o2369u.png)

Aquí la letra `E` que está en la línea 8 es un "diagnostic sign" que nos indica dónde está el error. El texto que se encuentra después del símbolo `■` es un "texto virtual" que nos muestra el mensaje del error.

### La configuración del servidor

Cada servidor LSP puede tener parámetros únicos y específicos, este tipo de parámetros debe colocarse en una propiedad llamada `settings` en la función `vim.lsp.start()`. Ahora bien, algunas veces nos encontramos que la documentación del servidor nos enseña ejemplos en este formato:

```
intelephense.files.maxSize: 1000000
```

Para que funcione con el cliente LSP de Neovim debemos adaptar este formato a algo que sea compatible con lua. Les mostraré cómo se hace en este caso.

```lua
vim.lsp.start({
  cmd = {'intelephense', '--stdio'},
  root_dir = root_dir,
  settings = {
    intelephense = {
      files = {
        maxSize = 1000000,
      },
    }
  }
})
```

Básicamente cada punto es una "tabla de lua" anidada.

Y si hay otra configuración con el mismo prefijo `intelephense.files` simplemente la añadimos a la tabla que ya existe.

```lua
vim.lsp.start({
  cmd = {'intelephense', '--stdio'},
  root_dir = root_dir,
  settings = {
    intelephense = {
      files = {
        maxSize = 1000000,
        otroEjemplo = false,
      },
    }
  }
})
```

### ¿Dónde empezamos a buscar si algo sale mal?

Si Neovim no fue capaz de iniciar el proceso del servidor LSP pueden revisar el log. Ejecuten este comando en Neovim:

```lua
:lua vim.cmd.edit(vim.lsp.get_log_path())
```

Cuando algo sale mal habrá unas líneas que comienzan con `[ERROR]`. Tal vez ahí pueden encontrar alguna información útil.

Si quieren un log con más detalles pueden agregar esto a su archivo `init.lua`.

```lua
vim.lsp.set_log_level('debug')
```

## Diagnostics

"Diagnostic" es el término que se usa en Neovim para los errores que se encuentran en nuestro código fuente.

Un diagnóstico puede tener un signo al inicio de la línea para indicarnos que hay un error. Este es el diagnostic sign. El espacio que necesita este signo está escondido por defecto, y cuando Neovim va a mostrarlo toda la "pantalla" se mueve a la derecha. Este comportamiento puede configurarse en nuestro `init.lua`.

Si configuramos la opción `signcolumn` con la cadena de texto `yes`, Neovim reserva el espacio para el signo. Tendremos un espacio en blanco al inicio de la línea en todo momento.

```lua
vim.opt.signcolumn = 'yes'
```

Por otro lado si asignamos la cadena de texto `no`, Neovim no mostrará ningún signo. No recomiendo esta opción, especialmente si no están al tanto de las consecuencias. Si quieren esconder los signos de diagnósticos hay una alternativa mejor.

### vim.diagnostic

En lua tenemos acceso al módulo [vim.diagnostic](https://neovim.io/doc/user/diagnostic.html), y dentro de este módulo está una función llamada [.config()](https://neovim.io/doc/user/diagnostic.html#vim.diagnostic.config()). Si usamos esta función en nuestro archivo `init.lua` podremos configurar el comportamiento de los diagnósticos.

* Esconder los signos

Esta es la forma segura de deshabilitar los signos de los diagnósticos.

```lua
vim.diagnostic.config({
  signs = false,
})
```

* Deshabilitar texto virtual

```lua
vim.diagnostic.config({
  virtual_text = false,
})
```

Si tienen **Neovim v0.10 o mayor** pueden leer el diagnóstico de la línea actual usando `<C-w>d` (control + w luego d). Este atajo ejecuta la función [vim.diagnostic.open_float()](https://neovim.io/doc/user/diagnostic.html#vim.diagnostic.open_float()).

Si están usando **Neovim v0.9.5 o menor** tendrán que crear un atajo de teclado.

```lua
vim.keymap.set('n', '<C-w>d', '<cmd>lua vim.diagnostic.open_float()<cr>')
```

Ahora bien, me gustaría poder explicarles con más detalle cada opción en `vim.diagnostic.config()` pero no tenemos tiempo. Si quieren explorar las demás opciones pueden [leer la documentación](https://neovim.io/doc/user/diagnostic.html#vim.diagnostic.config()).

## Funcionalidades gratis

Hablemos de las funcionalidades que Neovim nos ofrece con la configuración que tenemos hasta ahora, que es básicamente lo mínimo.

Cuando un servidor LSP está activo en un archivo Neovim habilita ciertas funcionalidades.

**Desde Neovim v0.8**

* A la opción `tagfunc` se le asigna una función que usa el servidor LSP activo. En otras palabras, podemos saltar a la definición de una función o clase usando el atajo `<C-]>` (Control + ]). Y podemos volver a la posición anterior usando `<C-t>` (Control + t).

* La opción `formatexpr` se le asigna una función que usa el servidor LSP activo. Esto quiere decir que podemos usar el operador `gq` para formatear un fragmento de código. Ejemplo: podemos ir al modo visual, seleccionamos un pedazo de código, presionamos `gq` y Neovim le pedirá al servidor LSP activo que formatee el código.

* La opción `omnifunc` también es configurada. Esta nos permite obtener completado de código. En modo de inserción el atajo `<C-x><C-o>` (control + x y luego control + o) nos presentará sugerencias que el servidor LSP cree que son relevantes.

**Desde Neovim v0.9**

* Se agrega soporte para el "resaltado semántico." Resulta que algunos servidores proporcionan información acerca de los símbolos que se encuentran en el archivo actual, y Neovim puede usar esta información para crear un resaltado de código más preciso.

**Desde Neovim v0.10**

* Si estamos en modo normal y no tenemos ningún atajo vinculado a `K`, podremos usarlo para leer la documentación del símbolo que está debajo del cursor.

* En modo normal, el atajo `<C-w>d` muestra los diagnósticos de la línea debajo del cursor.

* En modo normal, el atajo `]d` mueve el cursor hacia siguiente diagnóstico más cercano.

* En modo normal, el atajo `[d` mueve el cursor hacia diagnóstico previo.

**Desde Neovim v0.11**

> v0.11 es la versión que está en desarrollo actualmente. La información en esta sección puede cambiar en cualquier momento.

* En modo normal, el atajo `grn` renombra todas las referencias del símbolo debajo del cursor.

* En modo normal, el atajo `gra` muestra una lista de "code actions" disponibles para la línea actual.

* En modo normal, el atajo `grr` muestra todas las referencias del símbolo debajo del cursor.

* En modo de inserción, el atajo `<Ctrl-s>` (Control + s) muestra documentación de los argumentos de la función debajo del cursor.

## Funciones disponibles

Ahora voy a mostrarles una lista de funcionalidades que podremos usar sin mucho esfuerzo, lo único que debemos hacer es ejecutar una función de lua.

* Saltar a declaración
* Mostrar implementaciones
* Saltar a definición de tipo
* Listar referencias
* Mostrar argumentos de función
* Renombrar referencias del símbolo debajo del cursor
* Formatear archivo (o rango seleccionado)
* Listar "code actions" disponibles

Para aprovechar estas funciones podemos vincularlas a atajos de teclado.

### Atajos

Para los atajos de teclado es muy común seguir el mismo patrón que Neovim usa en otras funcionalidades, es decir configurar funciones sólo cuando hay un servidor LSP activo. Para lograr esto podemos usar un autocomando con el evento `LspAttach`. Si no están al tanto, Neovim emite el evento `LspAttach` cada vez que activa un servidor LSP en un archivo. Y con la función `nvim_create_autocmd` podremos ejecutar una función de lua cada vez que Neovim emite ese evento.

> Pueden encontrar más detalles sobre autocomandos aquí: [lua-guide-autocommands](https://neovim.io/doc/user/lua-guide.html#lua-guide-autocommands).

Aquí les muestro una manera de crear atajos de teclado para las funcionalidades vinculadas a un servidor LSP.

```lua
-- pueden agregar esto en su archivo `init.lua`

-- Mostrar diagnósticos de la línea actual
vim.keymap.set('n', 'gl', '<cmd>lua vim.diagnostic.open_float()<cr>')

-- Saltar al diagnóstico anterior
vim.keymap.set('n', '[d', '<cmd>lua vim.diagnostic.goto_prev()<cr>')

-- Saltar al siguiente diagnóstico
vim.keymap.set('n', ']d', '<cmd>lua vim.diagnostic.goto_next()<cr>')

vim.api.nvim_create_autocmd('LspAttach', {
  desc = 'Acciones LSP',
  callback = function(event)
    local bufmap = function(mode, lhs, rhs)
      local opts = {buffer = event.buf}
      vim.keymap.set(mode, lhs, rhs, opts)
    end

    -- Para más información de estas funciones usen el comando :help
    -- ver por ejemplo, :help vim.lsp.buf.hover()

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
  end
})
```

## Advertencia

No todos los servidores implementan el protocolo LSP en su totalidad. Y no todas las funcionalidades son consistentes entre servidores.

Por ejemplo, intelephense puede mostrar errores en nuestro código en tiempo real, sin necesidad de guardar los cambios en el archivo. Pero [rust-analyzer](https://github.com/rust-lang/rust-analyzer), el servidor LSP para rust, sólo puede mostrar errores una vez que se guardan cambios en el archivo.

Otro ejemplo. [ruff-lsp](https://github.com/astral-sh/ruff-lsp), un servidor LSP para python, en su descripción menciona que su especialidad es reportar errores y formatear código. Hasta donde pude ver este servidor no ofrece completado de código o resaltado de código semántico.

Lo que intento decir es esto: lean la documentación del servidor LSP para que estén informados de las funcionalidades que ofrece.

## Contenido extra

A estas alturas me atrevería a decir que ya tienen el conocimento esencial para usar el cliente LSP de Neovim y ser productivos. Lo que sigue son consejos, ajustes y funcionalidades que pueden implementar agregando un bloque de código a su configuración personal.

### Configurar un servidor LSP para múltiples tipos de archivo

Para esto aún podríamos usar un filetype plugin pero existe una alternativa mejor. Tenemos la opción de crear un "plugin global" y configurar el servidor usando un autocomando con el evento `FileType`.

En esta ocasión vamos a utilizar [tsserver](https://github.com/typescript-language-server/typescript-language-server) como ejemplo. Y si no lo saben, `tsserver` es el servidor LSP para typescript y javascript.

Si quieren probar el código que les voy a mostrar, deben instalar tsserver usando este comando.

```sh
npm install -g typescript typescript-language-server
```

Para crear un plugin global simplemente tenemos que crear un archivo en el lugar correcto. Vamos a la carpeta de configuración de Neovim, abrimos Neovim, y luego creamos una carpeta llamada `plugin`.

```vim
:call mkdir('./plugin', 'p')
```

Dentro de esta carpeta podemos crear un script de lua. A este script podemos darle cualquier nombre. Pero en este caso vamos a llamarlo `tsserver.lua`.

```vim
:edit plugin/tsserver.lua | write
```

En este script vamos a adaptar la configuración que se encuentra en el [repositorio de nvim-lspconfig](https://github.com/neovim/nvim-lspconfig/blob/3f6d120721e3a2b2812af43e6a8ba5522aa421c5/lua/lspconfig/server_configurations/tsserver.lua).

```lua
-- plugin/tsserver.lua

local function start_tsserver()
  local root_files = {'package.json', 'tsconfig.json', 'jsconfig.json'}
  local paths = vim.fs.find(root_files, {stop = vim.env.HOME})
  local root_dir = vim.fs.dirname(paths[1])

  if root_dir == nil then
    return
  end

  vim.lsp.start({
    name = 'tsserver',
    cmd = {'typescript-language-server', '--stdio'},
    root_dir = root_dir,
    init_options = {hostInfo = 'neovim'},
  })
end

vim.api.nvim_create_autocmd('FileType', {
  pattern = {'javascript', 'javascriptreact', 'typescript', 'typescriptreact'},
  desc = 'Iniciar tsserver',
  callback = start_tsserver,
})
```

Este autocomando funciona de manera similar a un filetype plugin, la diferencia es que aquí sólo ejecutamos una función de lua y no un archivo completo. La ventaja de esto es que podemos listar varios tipos de archivos en el argumento `pattern`.

Antes de que se me olvide, este ejemplo no tiene que estar en la carpeta plugin. Podemos colocar este código en el archivo `init.lua`.

### Agregar bordes a ventanas flotantes

Desafortunadamente no hay forma de colocar bordes a todas las ventanas flotantes que crea Neovim. Esto es algo que debemos configurar en cada funcionalidad que queremos usar.

Podemos colocar los siguientes ejemplos en el archivo `init.lua`.

* Detalles de diagnósticos

Para la ventana que muestra el mensaje de error de un diagnóstico.

```lua
vim.diagnostic.config({
  float = {
    border = 'rounded',
  },
})
```

* Documentación de símbolo

Para la ventana que desplega la función [vim.lsp.buf.hover()](https://neovim.io/doc/user/lsp.html#vim.lsp.buf.hover()).

```lua
vim.lsp.handlers['textDocument/hover'] = vim.lsp.with(
  vim.lsp.handlers.hover,
  {border = 'rounded'}
)
```

* Documentación de funciones

Para la ventana que desplega la función [vim.lsp.buf.signature_help()](https://neovim.io/doc/user/lsp.html#vim.lsp.buf.signature_help()).

```lua
vim.lsp.handlers['textDocument/signatureHelp'] = vim.lsp.with(
  vim.lsp.handlers.signature_help,
  {border = 'rounded'}
)
```

### Formatear al guardar

Lo que haremos aquí es ejecutar la función [vim.lsp.buf.format()](https://neovim.io/doc/user/lsp.html#vim.lsp.buf.format()) antes de que Neovim guarde los cambios a un archivo. Y por supuesto, esto sólo lo haremos cuando hay un servidor LSP activo.

Importante: los servidores LSP con capacidades para formatear código tienen su propia configuración. Por ejemplo, puede que nosotros tengamos la indentación configurada a 2 espacios pero el servidor LSP puede formatear el código con 4 espacios. Entonces, siempre es buena idea indagar en la documentación del servidor LSP para saber cómo configurar el estilo del formateo.

```lua
-- Pueden agregar esto en su archivo `init.lua`
-- o un script en la carpeta plugin

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
    desc = 'Formatear archivo',
    callback = buf_format,
  })
end

vim.api.nvim_create_autocmd('LspAttach', {
  desc = 'Configurar formatear al guardar',
  callback = setup_autoformat,
})
```

### Cambiar texto de signos de diagnósticos

Si estamos usando Neovim en su **versión v0.9.5 o menor**, debemos usar la función [vim.fn.sign_define()](https://neovim.io/doc/user/builtin.html#sign_define()).

```lua
-- Pueden agregar esto en su archivo `init.lua`
-- o un script en la carpeta plugin

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

Si estamos usando Neovim en su **versión v0.10 o mayor**, debemos usar la función `vim.diagnostic.config()`.

```lua
-- Pueden agregar esto en su archivo `init.lua`
-- o un script en la carpeta plugin

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

### Deshabilitar diagnósticos al entrar en modo de inserción

Neovim ya lo hace por defecto pero hay problema... no un problema como tal, es un detalle: los diagnósticos sólo desaparecen cuando empezamos a editar el código.

Esta es la configuración que **yo uso** para deshabilitar immediatamente los diagnósticos al entrar en modo de inserción (o modo de selección).

```lua
-- Pueden agregar esto en su archivo `init.lua`
-- o un script en la carpeta plugin

vim.api.nvim_create_autocmd('ModeChanged', {
  pattern = {'n:i', 'v:s'},
  desc = 'Deshabilitar diagnósticos',
  callback = function(e) vim.diagnostic.disable(e.buf) end
})

vim.api.nvim_create_autocmd('ModeChanged', {
  pattern = 'i:n',
  desc = 'Habilitar diagnósticos',
  callback = function(e) vim.diagnostic.enable(e.buf) end
})
```

### Deshabilitar resaltado semántico

La documentación de Neovim sugiere "despejar" los grupos de resaltado con el prefijo `@lsp`. Entonces, deberíamos hacer algo como esto.

```lua
-- Pueden agregar esto en su archivo `init.lua`
-- este código debe ejecutarse antes de configurar el tema del editor

vim.api.nvim_create_autocmd('ColorScheme', {
  desc = 'Deshabilitar resaltado semántico',
  callback = function()
    for _, group in ipairs(vim.fn.getcompletion('@lsp', 'highlight')) do
      vim.api.nvim_set_hl(0, group, {})
    end
  end,
})
```

Debemos crear el autocomando `ColorScheme` antes de configurar el tema del editor. De esta manera Neovim puede deshabilitar el resaltado incluso si cambiamos de tema en medio de una sesión.

### Resaltar símbolo debajo del cursor

Para esto vamos a usar la función [vim.lsp.buf.document_highlight()](https://neovim.io/doc/user/lsp.html#vim.lsp.buf.document_highlight()). Cuando el cursor pase una cantidad de tiempo sobre un símbolo vamos a ejecutar esa función. Y luego vamos desactivar el resaltado cuando el cursor se mueva de lugar.

Para que esto funcione de manera apropiada el tema del editor debe soportar los siguientes grupos:

* LspReferenceRead
* LspReferenceText
* LspReferenceWrite

Si el tema que tenemos no define ningún color para esos grupos podemos hacer un "vinculo" con un grupo que ya existe. Este es un ejemplo que muestra cómo vincular al grupo `Search`.

```lua
vim.api.nvim_set_hl(0, 'LspReferenceRead', {link = 'Search'})
vim.api.nvim_set_hl(0, 'LspReferenceText', {link = 'Search'})
vim.api.nvim_set_hl(0, 'LspReferenceWrite', {link = 'Search'})
```

```lua
-- Pueden agregar esto en su archivo `init.lua`
-- o un script en la carpeta plugin

-- tiempo en milisegundos para emitir el event `CursorHold`
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
  desc = 'Resaltar símbolo debajo del cursor',
  callback = highlight_symbol,
})
```

### Tab para completar

Para mantener una implementación simple vamos a hacer lo siguiente: Si el menú de sugerencias es visible, usaremos Tab y Shift + Tab para navegar entre las sugerencias. Si el cursor está sobre un espacio en blanco, insertaremos el caracter `<Tab>`. Si no, mostraremos el menú de sugerencias.

Ahora bien, si tenemos un servidor LSP activo que puede proveer sugerencias usamos eso. De lo contrario le pediremos a Neovim que sugiera palabras que se encuentran en el archivo actual.

Nota: recuerden que pueden confirmar una sugerencia usando `Enter` o `<C-y>` (Control + y).

```lua
-- Pueden agregar esto en su archivo `init.lua`
-- o un script en la carpeta plugin

vim.opt.completeopt = {'menu', 'menuone', 'noselect', 'noinsert'}
vim.opt.shortmess:append('c')

local function tab_complete()
  if vim.fn.pumvisible() == 1 then
    -- navega al siguiente elemento en la lista
    return '<Down>'
  end

  local c = vim.fn.col('.') - 1
  local is_whitespace = c == 0 or vim.fn.getline('.'):sub(c, c):match('%s')

  if is_whitespace then
    -- inserta el caracter tab
    return '<Tab>'
  end

  local lsp_completion = vim.bo.omnifunc == 'v:lua.vim.lsp.omnifunc'

  if lsp_completion then
    -- activa el completado con el servidor LSP
    return '<C-x><C-o>'
  end

  -- sugiere palabras en el archivo actual
  return '<C-x><C-n>'
end

local function tab_prev()
  if vim.fn.pumvisible() == 1 then
    -- navega al elemento anterior en la lista
    return '<Up>'
  end

  -- inserta el caracter tab
  return '<Tab>'
end

vim.keymap.set('i', '<Tab>', tab_complete, {expr = true})
vim.keymap.set('i', '<S-Tab>', tab_prev, {expr = true})
```

### Expandir snippets

En **Neovim v0.11** tenemos acceso a un módulo llamado [vim.lsp.completion](https://neovim.io/doc/user/lsp.html#lsp-completion), con esto podremos extender las funcionalidades del mecanismo de completado de código. Para ser más específico, Neovim podrá responder a más "comandos de edición" que retorne un servidor LSP. Por ejemplo, podremos expandir snippets de código.

Por el momento las funcionalidades de `vim.lsp.completion` son opcionales, debemos usar la función [vim.lsp.completion.enable()](https://neovim.io/doc/user/lsp.html#vim.lsp.completion.enable()) para habilitarlas.

```lua
-- Pueden agregar esto en su archivo `init.lua`
-- o un script en la carpeta plugin

vim.opt.completeopt = {'menu', 'menuone', 'noinsert', 'noselect'}

vim.api.nvim_create_autocmd('LspAttach', {
  desc = 'Habilitar vim.lsp.completion',
  callback = function(event)
    local client_id = vim.tbl_get(event, 'data', 'client_id')
    if client_id == nil then
      return
    end

    -- advertencia: api inestable
    vim.lsp.completion.enable(true, client_id, event.buf, {autotrigger = false})

    -- advertencia: api inestable
    -- Activar el menú de sugerencias manualmente
    vim.keymap.set('i', '<C-Space>', '<cmd>lua vim.lsp.completion.trigger()<cr>')
  end
})
```

Noten que el último argumento en `.enable()` tiene la propiedad `autotrigger`. `false` es el valor por defecto entonces yo lo dejo así. Si cambian ese valor a `true` Neovim podrá activar el completado de código cuando encuentre un "trigger character". Por ejemplo, con el servidor LSP de lua Neovim mostrará sugerencias cuando se encuentre con el caracter `.` o `:`.

Si están usando **Neovim v0.10** no tendrán acceso al módulo `vim.lsp.completion`, pero sí podrań usar un módulo llamado [vim.snippet](https://neovim.io/doc/user/lua.html#vim.snippet). Con esto podrán implementar un autocomando para expandir snippets.

```lua
-- Pueden agregar esto en su archivo `init.lua`
-- o un script en la carpeta plugin

-- Nota: esta funcion sólo expande snippets
-- no maneja "comandos de edición extra"
local function expand_snippet(event)
  local comp = vim.v.completed_item
  local kind = vim.lsp.protocol.CompletionItemKind
  local item = vim.tbl_get(comp, 'user_data', 'nvim', 'lsp', 'completion_item')

  -- Verifica que el item es un snippet
  if (
    not item
    or not item.insertTextFormat
    or item.insertTextFormat == 1
    or not (
      item.kind == kind.Snippet
      or item.kind == kind.Keyword
    )
  ) then
    return
  end

  -- Eliminar texto inicial
  local cursor = vim.api.nvim_win_get_cursor(0)
  local line = vim.api.nvim_get_current_line()
  local lnum = cursor[1] - 1
  local start_col = cursor[2] - #comp.word

  if start_col < 0 then
    return
  end

  local set_text = vim.api.nvim_buf_set_text
  local ok = pcall(set_text, bufnr, lnum, start_col, lnum, #line, {''})

  if not ok then
    return
  end

  -- Insertar snippet
  local snip_text = vim.tbl_get(item, 'textEdit', 'newText') or item.insertText

  assert(snip_text, "El servidor LSP dice que nos ha proporcionado un snippet, pero el campo donde debe estar el texto está vacío")

  vim.snippet.expand(snip_text)
end

vim.api.nvim_create_autocmd('CompleteDone', {
  desc = 'Expandir snippet',
  callback = expand_snippet
})
```

Con el módulo `vim.snippet` también podemos saltar entre placeholders dentro del snippet. En **Neovim v0.11** podrán usar `<Tab>` y `<Shift-Tab>` para navegar entre placeholders.

Si están usando **Neovim v0.10**, tendrán que crear sus propios atajos. Aquí les dejo una sugerencia:

```lua
-- Pueden agregar esto en su archivo `init.lua`

-- Control + f: Saltar al siguiente placeholder
vim.keymap.set({'i', 's'}, '<C-f>', function()
  if vim.snippet.active({direction = 1}) then
    return '<cmd>lua vim.snippet.jump(1)<cr>'
  else
    return '<C-f>'
  end
end, {expr = true})

-- Control + b: Saltar al placeholder anterior
vim.keymap.set({'i', 's'}, '<C-b>', function()
  if vim.snippet.active({direction = -1}) then
    return '<cmd>lua vim.snippet.jump(-1)<cr>'
  else
    return '<C-b>'
  end
end, {expr = true})

-- Control + l: "Salir" del snippet activo
vim.keymap.set({'i', 's'}, '<C-l>', function()
  if vim.snippet.active() then
    return '<cmd>lua vim.snippet.exit()<cr>'
  else
    return '<C-l>'
  end
end, {expr = true})
```

### Habilitar "inlay hints"

Inlay hints es parecido al texto virtual. Estos son fragmento de texto que Neovim agrega a lo que estamos editando, pero que en realidad no es parte del archivo. En este caso se usa para mostrar el tipo/clase de una variable o argumento de una función.

En este caso debemos usar la función [vim.lsp.inlay_hint.enable()](https://neovim.io/doc/user/lsp.html#vim.lsp.inlay_hint.enable()) en el archivo donde queremos habilitar esta funcionalidad. Como siempre, podemos usar un autocomando con el evento `LspAttach`.

Nota: la mayoría de los servidores que tienen soporte para esta funcionalidad la deshabilitan por defecto. Esto quiere decir que activar los inlay hints en Neovim no es suficiente, también debemos buscar las opciones específicas del servidor LSP y configurarlas en la función `vim.lsp.start()`.

```lua
-- Pueden agregar esto en su archivo `init.lua`
-- o un script en la carpeta plugin

vim.api.nvim_create_autocmd('LspAttach', {
  desc = 'Habilitar inlay hints',
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

## Conclusión

Les mostré cómo "conectar" un servidor LSP con Neovim. Tuvimos que ejecutar 1 comando en la terminal y agregar 9 líneas de código a un archivo de lua. ¿No fue tan difícil, verdad?

La parte que realmente requiere esfuerzo es aprender todo el contexto necesario. ¿Qué significa LSP? ¿Qué es un servidor LSP? ¿Filetype plugin? Pero una vez que tienen conocimiento de todas las piezas involucradas todo se hace más fácil.

Una cosa más... por favor no ignoren el "conocimento básico." Dediquen un poco de tiempo para aprender la sintaxis de lua. Aprendan sobre el comando `:help` en Neovim. Investiguen un poco sobre la API de lua que ofrece Neovim. Les aconsejo leer [la guía oficial de lua](https://neovim.io/doc/user/lua-guide.html) que está en la documentación de Neovim, es buen punto de partida.

