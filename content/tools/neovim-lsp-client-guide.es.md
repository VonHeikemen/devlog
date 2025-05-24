+++
title = "Guía de uso del cliente LSP de Neovim" 
description = "Agrega funcionalidades dignas de un IDE a Neovim sin instalar plugins de terceros"
date = 2024-01-13
updated = 2025-05-23
lang = "es"
[taxonomies]
tags = ["neovim", "shell"]
[extra]
shared = []
+++

El título original para este post era "Cómo agregar funcionalidades como las de un IDE a Neovim sin instalar plugins de terceros." Suena más interesante pero tiene demasiadas palabras. Aún así, es básicamente lo que quiero enseñarles.

En esta ocasión quiero explicar cómo configurar un servidor LSP en Neovim v0.11. Y voy a mostrarles cómo funciona porque este nuevo método es simplemente una abstracción sobre funcionalidades que han estado en Neovim durante mucho tiempo. Así podrán construir su propia configuración incluso en versiones anteriores de Neovim.

## Requerimientos

1. Neovim en su versión v0.8 o mayor
2. Un servidor LSP
3. Paciencia/Energía para escribir un poco de lua por cada servidor LSP que se quiere configurar

Si no conocen el lenguaje lua aquí les dejo un par de recursos donde pueden aprender lo básico:

* [Lua crash course (video de 11 min)](https://www.youtube.com/watch?v=NneB6GX1Els)
* [Learn X in Y minutes: Where X = lua](https://learnxinyminutes.com/docs/lua/)
* [Guía oficial de lua para Neovim](https://neovim.io/doc/user/lua-guide.html)

## Todo comienza con el servidor LSP

Un servidor LSP es un programa externo que sigue el [Protocolo de Servidor de Lenguajes](https://microsoft.github.io/language-server-protocol/) (Language Server Protocol). En este se definen los parámetros que puede recibir un servidor LSP, y también cómo debe responder. El propósito de todo esto es que [cualquier herramienta que implemente el protocolo](https://microsoft.github.io/language-server-protocol/implementors/tools/) pueda comunicarse con un servidor LSP.

Entonces, el servidor LSP es el programa que va a analizar nuestro código y le dice a nuestro editor qué debe hacer.

*¿Qué servidores LSP tenemos disponibles?*

La página web del protocolo [tiene una lista de servidores](https://microsoft.github.io/language-server-protocol/implementors/servers/).

### En este caso en particular...

Quiero usar [Gleam](https://gleam.run/) y [LuaLS](https://github.com/LuaLS/lua-language-server) como ejemplos prácticos.

Gleam en realidad es un conjunto de herramientas. Incluye un compilador, un formateador de código y un servidor LSP. Las instrucciones de instalación se encuentran en la documentación oficial: [Getting started](https://gleam.run/getting-started/installing/).

LuaLS es un servidor LSP para el lenguaje lua. Pueden descargar la versión más reciente desde su repositorio, en la sección [github releases](https://github.com/LuaLS/lua-language-server/releases/latest). O, pueden [compilarlo con el código fuente](https://luals.github.io/wiki/build/).

Una vez que tienen un servidor LSP instalado recomiendo verificar si Neovim puede encontrar el ejecutable. Para esto pueden usar la función [exepath](https://neovim.io/doc/user/builtin.html#exepath()).

El ejecutable de Gleam es `gleam`.

```vim
:echo exepath('gleam')
```

Y el de LuaLS es `lua-language-server`.

```vim
:echo exepath('lua-language-server')
```

Estos comandos deben mostrar la ubicación del ejecutable en su sistema. De no ser así, significa que algo salió mal durante la instalación del servidor LSP.

## Uso básico

Antes de empezar a escribir código deberíamos aprender cómo usar el servidor LSP. Esta información debería estar en la documentación oficial del servidor que queremos usar.

Si no podemos encontrar instrucciones de uso básico en la documentación, podemos ir al repositorio de [nvim-lspconfig](https://github.com/neovim/nvim-lspconfig) y buscamos el directorio [lsp](https://github.com/neovim/nvim-lspconfig/tree/master/lsp). Ahí podemos encontrar archivos de configuración para un montón de servidores.

Justo ahora estamos interesados en Gleam, entonces deberíamos revisar el contenido de [gleam.lua](https://github.com/neovim/nvim-lspconfig/blob/ac1dfbe3b60e5e23a2cff90e3bd6a3bc88031a57/lsp/gleam.lua#L8).

```lua
return {
    cmd = { 'gleam', 'lsp' },
    filetypes = { 'gleam' },
    root_markers = { 'gleam.toml', '.git' },
}
```

Aquí tenemos la información esencial que necesitamos para poder integrar un servidor LSP en Neovim.

La propiedad `cmd` contiene el comando que necesitamos para inicializar el servidor LSP. Recuerden que un servidor LSP es un programa externo. Quiere decir que `gleam lsp` es un comando válido que podemos ejecutar manualmente en nuestra terminal. Y curiosamente, si hacemos eso nos mostrará este mensaje.

```
This command is intended to be run by language server clients such
as a text editor rather than being run directly in the console.
```

El mensaje dice "este comando fue creado para ser ejecutado por un cliente LSP como por ejemplo un editor de texto, en lugar de ser ejecutado directamente por cónsola."

En la propiedad `filetypes` tenemos la lista de lenguajes que el servidor LSP soporta. Deben tener en cuenta que cada nombre debe ser un "filetype" válido en Neovim.

En `root_markers` tenemos una lista de archivos y directorios. Esta información se usa para determinar la "raíz" de nuestro proyecto. Es decir el directorio principal del proyecto.

## El directorio lsp

A partir de la versión **v0.11** el directorio `lsp` forma parte de lo que llamamos `runtimepath`. Quiere decir que tenemos la posibilidad de crear nuestro propio directorio `lsp`. Ahí podemos colocar la configuración de los servidores LSP que tenemos instalados, sin necesidad de instalar `nvim-lspconfig` como un plugin.

Imaginen por un momento que nuestra configuración personal de Neovim tiene esta estructura:

```txt
nvim
├── init.vim
├── .nvim.lua
└── lsp
    ├── gleam.lua
    └── luals-nvim.lua
```

La configuración dentro de `nvim/lsp/gleam.lua` puede ser exactamente igual a la que tiene `nvim-lspconfig`.

```lua
-- nvim/lsp/gleam.lua

return {
  cmd = {'gleam', 'lsp'},
  filetypes = {'gleam'},
  root_markers = {'gleam.toml', '.git'},
}
```

Ahora bien, cuando configuramos un servidor LSP tenemos ciertas libertades. Entonces, en `luals-nvim` voy a mostrar una configuración de LuaLS específicamente creada para Neovim donde hay un script llamado `.nvim.lua` en el directorio raíz.

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

Dentro del directorio `lsp` los archivos que creamos pueden tener cualquier nombre. Por eso elegí `luals-nvim` en lugar de algo genérico como `lua` o `luals`. Y la configuración que "retornamos" en el script debe contener los mismos parámetros que [vim.lsp.config()](https://neovim.io/doc/user/lsp.html#vim.lsp.config()) espera.

Noten que en este ejemplo tenemos una propiedad llamada `settings`. Esa propiedad está reservada para opciones específicas del servidor LSP. Quiere decir que en Neovim no disponemos de una documentación que nos diga cuáles son las opciones que podemos colocar. Debemos ir a la documentación del servidor LSP y ver qué opciones tenemos disponibles.

Deben saber que los archivos dentro del directorio `lsp` no son ejecutados automáticamente. Debemos decirle a Neovim qué servidores LSP queremos usar. Para usar un servidor LSP debemos ejecutar la función [vim.lsp.enable()](https://neovim.io/doc/user/lsp.html#vim.lsp.enable()).

En este ejemplo el archivo `init.vim` puede tener este código:

```vim
" nvim/init.vim

set exrc

lua vim.lsp.enable('gleam')
```

*¿Por qué usar init.vim?*

Simplemente quería una excusa para mostrar que se puede ejecutar una función de lua en vimscript. He notado que algunos usuarios de Vim creen que para usar lua deben eliminar toda la configuración que tienen en vimscript. No es así. Lua y vimscript pueden coexistir.

En fin, al iniciar Neovim este buscará un archivo en el `runtimepath` que coincida con el patrón `lsp/gleam.lua`. Luego creará un autocomando usando la lista de lenguajes que suministramos en `filetypes`. Entonces si abrimos un archivo de tipo `gleam` Neovim intentará inicializar el servidor LSP.

*¿Y qué pasa con LuaLS?*

La configuración que tenemos para LuaLS sólo es útil para Neovim, lo que nos da una oportunidad para usar una configuración local. En Vim logramos esto usando la opción [exrc](https://neovim.io/doc/user/options.html#'exrc'). Cuando habilitamos los archivos `exrc` Vim/Neovim ejecutará un script que esté en directorio de trabajo actual. En nuestro ejemplo sería `.nvim.lua`. Esto es una espada de doble filo. Por eso en Neovim la opción `exrc` tiene un comportamiento diferente. Cuando se detecta una configuración local Neovim nos preguntará si "confiamos" en el archivo, y si aceptamos se ejecutará. La próxima vez que se detecte la misma configuración local se ejecutará automáticamente si el contenido del archivo no ha cambiado.

Yo recomendaría crear un alias para habilitar la opción `exrc` desde la terminal, en lugar de colocarlo directo en su configuración personal. Algo como esto.

```sh
alias code='nvim --cmd "set exrc"'
```

De vuelta a LuaLS. Luego de Neovim termine de ejecutar el archivo `init.vim` (o `init.lua`) buscará el script `.nvim.lua` en el directorio actual. El que está en nuestra configuración tendrá este código.

```lua
-- nvim/.nvim.lua

vim.lsp.enable('luals-nvim')
```

De esta manera Neovim no intentará usar `lsp/luals-nvim.lua` si no lo abrimos en el directorio donde está nuestra configuración personal. La desventaja de este método es que debemos abrir Neovim exactamente donde está `.nvim.lua`.

## ¿Se puede configurar todo en un archivo?

Si ustedes son la clase de persona que prefiere una configuración simple donde todo está en un sólo archivo, les tengo buenas noticias. No están obligados a usar el directorio `lsp`.

La función [vim.lsp.config()](https://neovim.io/doc/user/lsp.html#vim.lsp.config()) puede usarse para extender las configuraciones que se encuentran en el directorio `lsp`. Pero también la podemos usar para crear configuraciones nuevas, sin necesidad de crear un archivo aparte.

Por ejemplo:

```lua
-- nvim/init.lua

vim.lsp.config('gleam', {
  cmd = {'gleam', 'lsp'},
  filetypes = {'gleam'},
  root_markers = {'gleam.toml', '.git'},
})

vim.lsp.enable('gleam')
```

## ¿Y si algo sale mal?

Si Neovim no fue capaz de iniciar el proceso del servidor LSP pueden revisar el log. Ejecuten este comando en Neovim:

```lua
:lua vim.cmd.edit(vim.lsp.get_log_path())
```

Cuando algo sale mal habrá unas líneas que comienzan con `[ERROR]`. Tal vez ahí pueden encontrar alguna información útil.

Si quieren un log con más detalles pueden ejecutar esta función.

```lua
vim.lsp.set_log_level('debug')
```

Y desde **Neovim v0.10** pueden hacer un "chequeo de salud" usando este comando:

```vim
:checkhealth lsp
```

## Filetypes

En versiones anteriores de Neovim no tenemos disponible `vim.lsp.enable()`, pero podemos replicar su funcionalidad. Ya sabemos lo que necesitamos sólo nos falta juntar las piezas.

Si el servidor LSP que queremos usar sólo soporta un lenguaje de programación entonces podemos crear un "filetype plugin."

Para crear un filetype plugin debemos ir a nuestra configuración de Neovim, y ahí debemos crear un directorio llamado `ftplugin`. Podemos ejecutar este comando en Neovim.

```vim
:call mkdir('./ftplugin', 'p')
```

Luego debemos crear un script con el mismo nombre de un tipo de archivo.

Si queremos crear un filetype plugin para Gleam debemos crear un archivo llamado `gleam.lua`.

```vim
:edit ftplugin/gleam.lua | write
```

En este script vamos a ejecutar la función que "conectará" el servidor LSP con Neovim.

Por otro lado, si el servidor LSP soporta varios tipos de archivos tenemos la opción de crear un autocomando en el evento `FileType`. Algo como esto.

```lua
vim.api.nvim_create_autocmd('FileType', {
  pattern = {'css', 'less', 'sass'},
  callback = function()
    ---
    -- Aquí podemos habilitar el servidor LSP
    ---
  end,
})
```

Aquí la propiedad `pattern` es donde debemos colocar los tipos de archivos que soporta el servidor LSP. Esta es la misma información que colocamos en la propiedad `filetypes` cuando configuramos un servidor LSP en Neovim v0.11.

Dentro del filetype plugin o el autocomando debemos ejecutar la función que va a inicializar el servidor LSP. Pero primero, debemos conocer la lógica detrás de `root_markers`.

## Directorio raíz

Un servidor LSP necesita saber la ruta de nuestro proyecto. La ubicación exacta dentro de nuestro sistema. Este es un problema que debemos resolver en Neovim.

En **Neovim v0.10** podemos usar una función llamada [vim.fs.root()](https://neovim.io/doc/user/lua.html#vim.fs.root()). A esta función le vamos a dar una lista de nombres de archivos y nos devolverá una ruta que podemos usar.

¿Pero qué vamos a buscar? Archivos de configuración, los que usualmente colocamos en la raíz de un proyecto. En proyectos de Gleam siempre hay un archivo `gleam.toml`. En proyectos de javascript usualmente hay un `package.json`. En rust está el `cargo.toml`. Este es el tipo de información que vamos a darle a `vim.fs.root()` para que nos devuelva la ruta correcta.

Entonces en el filetype plugin de Gleam podemos hacer una prueba con `vim.fs.root()`.

```lua
-- nvim/ftplugin/gleam.lua
-- NOTA: vim.fs.root() sólo está disponble en Neovim v0.10 o mayor

local root_markers = {'gleam.toml'}
local root_dir = vim.fs.root(0, root_markers)

print(root_dir)
```

En **Neovim v0.9** o menor debemos recrear el comportamiento de `vim.fs.root()`. En este caso podemos usar [vim.fs.find()](https://neovim.io/doc/user/lua.html#vim.fs.find()).

```lua
-- nvim/ftplugin/gleam.lua
-- NOTA: Este código es para Neovim v0.9.5 o menor

local root_markers = {'gleam.toml'}
local buffer = vim.api.nvim_buf_get_name(0)
local paths = vim.fs.find(root_markers, {
    upward = true,
    path = vim.fn.fnamemodify(buffer, ':p:h'),
})

local root_dir = vim.fs.dirname(paths[1])

print(root_dir)
```

## Creando el cliente

Estamos listos para conectar Neovim con el servidor LSP. Ahora debemos usar la función [vim.lsp.start()](https://neovim.io/doc/user/lsp.html#vim.lsp.start()) para inicializar el servidor LSP. Esta es la misma función que `vim.lsp.enable()` ejecuta por nosotros.

Esto es lo que deben saber, la primera vez que ejecutamos `vim.lsp.start()` Neovim iniciará nuestro servidor LSP como un proceso externo. Cuando se ejecuta nuevamente Neovim simplemente le enviará información al proceso del servidor LSP.

Entonces, el filetype plugin para Gleam debería tener este código:

```lua
-- nvim/ftplugin/gleam.lua
-- NOTA: vim.fs.root() sólo está disponble en Neovim v0.10 o mayor

local root_markers = {'gleam.toml'}
local root_dir = vim.fs.root(0, root_markers)

if root_dir then
  vim.lsp.start({
    cmd = {'gleam', 'lsp'},
    root_dir = root_dir,
  })
end
```

Con esta configuración ya podremos utilizar algunas funcionalidades. Por ejemplo, si editamos un archivo Gleam y este tiene un error, Neovim nos dirá en qué línea se encuentra el error.

## Diagnostics

"Diagnostic" es el término que se usa en Neovim para los errores que se encuentran en nuestro código fuente.

Un diagnóstico puede tener un signo al inicio de la línea para indicarnos que hay un error. Este es el diagnostic sign. El espacio que necesita este signo está escondido por defecto, y cuando Neovim va a mostrarlo toda la pantalla se mueve a la derecha. Este comportamiento puede configurarse.

Si configuramos la opción `signcolumn` con la cadena de texto `yes` Neovim reserva el espacio para el signo. Tendremos un espacio en blanco al inicio de la línea en todo momento.

En `init.vim` pueden configurarlo de esta manera.

```vim
set signcolumn=yes
```

Si usan `init.lua`.

```lua
vim.o.signcolumn = 'yes'
```

Por otro lado si asignamos la cadena de texto `no`, Neovim no mostrará ningún signo. No recomiendo esta opción, especialmente si no están al tanto de las consecuencias. Si quieren esconder los signos de diagnósticos hay una alternativa mejor.

### vim.diagnostic

En lua tenemos acceso al módulo [vim.diagnostic](https://neovim.io/doc/user/diagnostic.html), y dentro de este módulo está una función llamada [.config()](https://neovim.io/doc/user/diagnostic.html#vim.diagnostic.config()). Si con esta función podremos configurar el comportamiento de los diagnósticos.

* Esconder los signos

Esta es la forma segura de deshabilitar los signos de los diagnósticos.

```lua
vim.diagnostic.config({
  signs = false,
})
```

* Texto virtual

Tenemos la opción de mostrar un fragmento del mensaje de error al final de la línea. A esto se le conoce como "virtual text." En **Neovim v0.11** está deshabilitado por defecto. Así lo podemos habilitar.

```lua
vim.diagnostic.config({
  virtual_text = false,
})
```

Si tienen **Neovim v0.10 o mayor** pueden leer el diagnóstico completo usando `<C-w>d` (control + w luego d). Este atajo ejecuta la función [vim.diagnostic.open_float()](https://neovim.io/doc/user/diagnostic.html#vim.diagnostic.open_float()).

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

* En modo normal, el atajo `grn` renombra todas las referencias del símbolo debajo del cursor.

* En modo normal, el atajo `gra` muestra una lista de "code actions" disponibles para la línea actual.

* En modo normal, el atajo `grr` muestra todas las referencias del símbolo debajo del cursor.

* En modo normal, el atajo `gri` muestra la implementación del símbolo debajo del cursor.

* En modo normal, el atajo `gO` muestra lista todos los símbolos que existen en el archivo actual.

* En modo de inserción, el atajo `<Ctrl-s>` (Control + s) muestra documentación de los argumentos de la función debajo del cursor.

## Atajos

En **Neovim v0.11** ya tenemos atajos asignados para las funcionalidades que ofrece el editor. Pero en versiones anteriores debemos crear estos atajos nosotros mismos.

Este es el código que necesitamos en lua.

```lua
-- Pueden agregar esto en su archivo `init.lua`

-- Estos atajos ya forman parte de Neovim v0.10
vim.keymap.set('n', '[d', '<cmd>lua vim.diagnostic.goto_prev()<cr>')
vim.keymap.set('n', ']d', '<cmd>lua vim.diagnostic.goto_next()<cr>')
vim.keymap.set('n', '<C-w>d', '<cmd>lua vim.diagnostic.open_float()<cr>')
vim.keymap.set('n', '<C-w><C-d>', '<cmd>lua vim.diagnostic.open_float()<cr>')

vim.api.nvim_create_autocmd('LspAttach', {
  callback = function(event)
    local bufmap = function(mode, rhs, lhs)
      vim.keymap.set(mode, rhs, lhs, {buffer = event.buf})
    end

    -- Estos atajos ya forman parte de Neovim v0.11
    bufmap('n', 'K', '<cmd>lua vim.lsp.buf.hover()<cr>')
    bufmap('n', 'grr', '<cmd>lua vim.lsp.buf.references()<cr>')
    bufmap('n', 'gri', '<cmd>lua vim.lsp.buf.implementation()<cr>')
    bufmap('n', 'grn', '<cmd>lua vim.lsp.buf.rename()<cr>')
    bufmap('n', 'gra', '<cmd>lua vim.lsp.buf.code_action()<cr>')
    bufmap('n', 'gO', '<cmd>lua vim.lsp.buf.document_symbol()<cr>')
    bufmap({'i', 's'}, '<C-s>', '<cmd>lua vim.lsp.buf.signature_help()<cr>')

    -- Estos son atajos extras
    bufmap('n', 'gd', '<cmd>lua vim.lsp.buf.definition()<cr>')
    bufmap('n', 'grt', '<cmd>lua vim.lsp.buf.type_definition()<cr>')
    bufmap('n', 'grd', '<cmd>lua vim.lsp.buf.declaration()<cr>')
    bufmap({'n', 'x'}, 'gq', '<cmd>lua vim.lsp.buf.format({async = true})<cr>')
  end,
})
```

Y este sería el equivalente en vimscript.

```vim
" Pueden agregar esto en su archivo `init.vim`

" Estos atajos ya forman parte de Neovim v0.10
nnoremap [d <cmd>lua vim.diagnostic.goto_prev()<cr>
nnoremap ]d <cmd>lua vim.diagnostic.goto_next()<cr>
nnoremap <C-w>d <cmd>lua vim.diagnostic.open_float()<cr>
nnoremap <C-w><C-d> <cmd>lua vim.diagnostic.open_float()<cr>

function! LspAttached() abort
  " Estos atajos ya forman parte de Neovim v0.11
  nnoremap <buffer> K <cmd>lua vim.lsp.buf.hover()<cr>
  nnoremap <buffer> grr <cmd>lua vim.lsp.buf.references()<cr>
  nnoremap <buffer> gri <cmd>lua vim.lsp.buf.implementation()<cr>
  nnoremap <buffer> grn <cmd>lua vim.lsp.buf.rename()<cr>
  nnoremap <buffer> gra <cmd>lua vim.lsp.buf.code_action()<cr>
  nnoremap <buffer> gO <cmd>lua vim.lsp.buf.document_symbol()<cr>
  inoremap <buffer> <C-s> <cmd>lua vim.lsp.buf.signature_help()<cr>
  snoremap <buffer> <C-s> <cmd>lua vim.lsp.buf.signature_help()<cr>

  " Estos son atajos extras
  nnoremap <buffer> gd <cmd>lua vim.lsp.buf.definition()<cr>
  nnoremap <buffer> gq <cmd>lua vim.lsp.buf.format({async = true})<cr>
  xnoremap <buffer> gq <cmd>lua vim.lsp.buf.format({async = true})<cr>
  nnoremap <buffer> grd <cmd>lua vim.lsp.buf.declaration()<cr>
  nnoremap <buffer> grt <cmd>lua vim.lsp.buf.type_definition()<cr>
endfunction

autocmd LspAttach * call LspAttached()
```

## Advertencia

No todos los servidores implementan el protocolo LSP en su totalidad. Y no todas las funcionalidades son consistentes entre servidores.

Por ejemplo, Gleam puede mostrar errores en nuestro código en tiempo real, sin necesidad de guardar los cambios en el archivo. Pero [rust-analyzer](https://github.com/rust-lang/rust-analyzer), el servidor LSP para rust, sólo puede mostrar errores una vez que se guardan cambios en el archivo.

Otro ejemplo. [ruff-lsp](https://github.com/astral-sh/ruff-lsp), un servidor LSP para python, en su descripción menciona que su especialidad es reportar errores y formatear código. Hasta donde pude ver este servidor no ofrece completado de código o resaltado de código semántico.

Lo que intento decir es esto: lean la documentación del servidor LSP para que estén informados de las funcionalidades que ofrece.

## Contenido extra

A estas alturas me atrevería a decir que ya tienen el conocimento esencial para usar el cliente LSP de Neovim y ser productivos. Lo que sigue son consejos, ajustes y funcionalidades que pueden implementar agregando un bloque de código a su configuración personal.

### Formatear al guardar

Lo que haremos aquí es ejecutar la función [vim.lsp.buf.format()](https://neovim.io/doc/user/lsp.html#vim.lsp.buf.format()) antes de que Neovim guarde los cambios a un archivo. Y por supuesto, esto sólo lo haremos cuando hay un servidor LSP activo.

Importante: los servidores LSP con capacidades para formatear código tienen su propia configuración. Por ejemplo, puede que nosotros tengamos la indentación configurada a 2 espacios pero el servidor LSP puede formatear el código con 4 espacios. Entonces, siempre es buena idea indagar en la documentación del servidor LSP para saber cómo configurar el estilo del formateo.

```lua
-- Pueden agregar esto en su archivo `init.lua`

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

### Deshabilitar resaltado semántico

La documentación de Neovim sugiere modificar la instancia del cliente LSP. Debemos "eliminar" la propiedad que activa el resaltado semántico.

```lua
-- Pueden agregar esto en su archivo `init.lua`

vim.api.nvim_create_autocmd('LspAttach', {
  desc = 'Deshabilitar resaltado semántico',
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

vim.opt.completeopt = {'menu', 'menuone', 'noinsert', 'noselect'}

vim.api.nvim_create_autocmd('LspAttach', {
  desc = 'Habilitar vim.lsp.completion',
  callback = function(event)
    local client_id = vim.tbl_get(event, 'data', 'client_id')
    if client_id == nil then
      return
    end

    vim.lsp.completion.enable(true, client_id, event.buf, {autotrigger = false})

    -- Activar el menú de sugerencias manualmente
    vim.keymap.set('i', '<C-Space>', '<cmd>lua vim.lsp.completion.trigger()<cr>')
  end
})
```

Noten que el último argumento en `.enable()` tiene la propiedad `autotrigger`. `false` es el valor por defecto entonces yo lo dejo así. Si cambian ese valor a `true` Neovim podrá activar el completado de código cuando encuentre un "trigger character". Por ejemplo, con el servidor LSP de lua Neovim mostrará sugerencias cuando se encuentre con el caracter `.` o `:`.

### Habilitar "inlay hints"

Inlay hints es parecido al texto virtual. Estos son fragmento de texto que Neovim agrega a lo que estamos editando, pero que en realidad no es parte del archivo. En este caso se usa para mostrar el tipo/clase de una variable o argumento de una función.

En este caso debemos usar la función [vim.lsp.inlay_hint.enable()](https://neovim.io/doc/user/lsp.html#vim.lsp.inlay_hint.enable()) en el archivo donde queremos habilitar esta funcionalidad. Como siempre, podemos usar un autocomando con el evento `LspAttach`.

Nota: la mayoría de los servidores que tienen soporte para esta funcionalidad la deshabilitan por defecto. Esto quiere decir que activar los inlay hints en Neovim no es suficiente, también debemos buscar las opciones específicas del servidor LSP y configurarlas en la función `vim.lsp.start()`.

```lua
-- Pueden agregar esto en su archivo `init.lua`

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

Espero haber demostrado que no es tan difícil usar un servidor LSP en Neovim. La parte que realmente requiere esfuerzo es aprender todo el contexto necesario. ¿Qué significa LSP? ¿Qué es un servidor LSP? ¿Filetype plugin? Pero una vez que tienen conocimiento de todas las piezas involucradas todo se hace más fácil.

Una cosa más... por favor no ignoren el "conocimento básico." Dediquen un poco de tiempo para aprender la sintaxis de lua. Aprendan sobre el comando `:help` en Neovim. Investiguen un poco sobre la API de lua que ofrece Neovim. Les aconsejo leer [la guía oficial de lua](https://neovim.io/doc/user/lua-guide.html) que está en la documentación de Neovim, es buen punto de partida.

