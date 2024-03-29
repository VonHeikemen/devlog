+++
title = "Cómo crear tu primera configuración de Neovim usando lua"
description = "Donde aprendemos cómo personalizar Neovim y agregar plugins"
date = 2022-07-02
updated = 2024-03-20
lang = "es"
[taxonomies]
tags = ["vim", "neovim", "shell"]
[extra]
shared = [
  ["dev.to", "https://dev.to/vonheikemen/como-crear-tu-primera-configuracion-de-neovim-usando-lua-dah"],
  ["Hashnode", "https://vonheikemen.hashnode.dev/build-your-first-neovim-configuration-in-lua-es"]
]
+++

Neovim es un editor que se caracteriza por ser extensible, con suficiente esfuerzo podemos convertirlo en algo más que un editor de texto. Hoy espero poder enseñarles suficiente sobre `lua` y la api de Neovim para poder construir una configuración que se adapte a sus necesidades.

Lo que haremos será crear un archivo de configuración que llamaremos `init.lua`, agregaremos un par de plugins y les diré cómo crear sus propios comandos.

Este tutorial está pensado para aquellos que son totalmente nuevos en Neovim. Si ustedes ya tienen una configuración escrita en vimscript y desean migrarla a lua, encontrarán todo lo que necesitan aquí: [Todo lo que necesitan saber para configurar neovim usando lua](@/tools/configuring-neovim-using-lua.es.md).

## Recomendaciones

Antes de empezar, les aconsejo que instalen la versión de Neovim estable más reciente. Pueden visitar la [sección releases](https://github.com/neovim/neovim/releases) del repositorio en github y descargarla de ahí. De aquí en adelante voy a asumir que están usando la version 0.8 o mayor.

Si aún no se sienten cómodos usando Neovim para editar texto, hagan el tutorial interactivo que viene incluido. Pueden acceder a él ejecutando este comando en su terminal.

```sh
nvim +Tutor
```

El tutorial está inglés. Si no tienen un buen dominio del idioma pueden [descargar la versión en español](https://raw.githubusercontent.com/vim/vim/d899e51120798d3fb5420abb1f19dddf3f014d05/runtime/tutor/tutor.es). Guardan ese archivo en algún lugar de su sistema y lo abren con Neovim. Voy a asumir que ya conocen todas las funcionalidades que enseña el tutor.

## El Inicio

Lo primero que debemos hacer es crear nuestro archivo de configuración, el famoso `init.lua`. ¿Dónde? Depende de su sistema operativo, y también de sus variables de entorno. Pero puedo enseñarles cómo crearlo desde Neovim, así no tenemos que preocuparnos por esos detalles.

> Dato curioso: En algunos tutoriales se refieren al archivo de configuración como `vimrc`, porque ese es el nombre que tiene el archivo en Vim.

En esta sección no vamos a usar lua, usaremos el lenguaje que fue creado para Vim: vimscript.

Vamos a abrir Neovim y luego ejecutamos este comando.

```vim
:call mkdir(stdpath("config"), "p")
```

Con esto crearemos la carpeta donde vivirá nuestra configuración. Si quieren saber qué carpeta fue creada ejecuten esto.

```vim
:echo stdpath("config")
```

Ahora vamos con el comando para editar la configuración.

```vim
:exe "edit" stdpath("config") . "/init.lua"
```

Después de ejecutarlo tendremos una página en blanco, pero el archivo aún no existe en el sistema. Para crearlo debemos guardarlo. Ejecuten esto.

```vim
:write
```

Una vez que tenemos el archivo podremos editarlo en cualquier momento con este comando.

```vim
:edit $MYVIMRC
```

Si ustedes son de aquellas personas que les gusta crear scripts para automatizar sus tareas, pueden ejecutar todos estos pasos con un sólo comando.

```sh
nvim --headless -c 'call mkdir(stdpath("config"), "p") | exe "edit" stdpath("config") . "/init.lua" | write | quit'
```

## Opciones del editor

Para acceder a las opciones de Neovim debemos usar la variable global `vim`. Bueno, más que una variable, es un módulo donde podemos encontrar cualquier tipo de utilidades. Pero ahora lo que nos interesa es una propiedad llamada `opt`, con eso podremos modificar cualquiera de las 351 opciones que ofrece Neovim (en su versión 0.7).

Esta es la sintaxis que deben seguir.

```lua
vim.opt.nombre_opcion = valor
```

Donde `nombre_opcion` puede ser cualquiera de [esta lista](https://neovim.io/doc/user/quickref.html#option-list). Y `valor` debe concordar con el tipo de dato que espera la opción.

> Pueden ver la lista de opciones desde Neovim con el comando `:help option-list`.

Deben tener en cuenta que cada opción tiene un "ámbito". Algunas opciones son globales, otras se limitan a la ventana o archivo que están manipulando en su momento. Para conocer estos detalles revisen la documentación de esta manera.

```vim
:help 'nombre_opcion'
```

### Opciones de interés

* `number`

Esta opción es de tipo booleano, quiere decir que sólo tiene dos posibles valores: `true` o `false`. Si le asignamos el valor `true` la habilitamos, `false` hace lo contrario.

Al habilitar `number` Neovim nos muestra los números de cada línea en pantalla.

```lua
vim.opt.number = true
```

* `mouse`

Neovim (y Vim) puede utilizar el ratón para algunas operaciones, cosas como seleccionar texto o cambiar el tamaño de una ventana. `mouse` acepta una cadena de texto (carácteres rodeados por comillas) que contiene una combinación de modos. No vamos a preocuparnos por los modos, podemos habilitarlo para todos así:

```lua
vim.opt.mouse = 'a'
```

* `ignorecase`

Con esto le decimos a Neovim si debe ignorar las letrás mayúsculas cuando realizamos una búsqueda. Por ejemplo, si buscamos la palabra `dos` nuestros resultados pueden incluir diferentes variaciones como `Dos`, `DOS` o `dos`.

```lua
vim.opt.ignorecase = true
```

* `smartcase`

Hace que nuestra búsqueda ignore las letrás mayúsculas a menos que el término que estamos buscando tenga una letra mayúscula. Generalmente se usa en conjunto con `ignorecase`.

```lua
vim.opt.smartcase = true
```

* `hlsearch`

Resalta los resultados de una búsqueda anterior. La mayor parte del tiempo no queremos esto, así que lo deshabilitamos.

```lua
vim.opt.hlsearch = false
```

* `wrap`

Hace que el texto de las líneas largas (las que sobrepasan el ancho de la pantalla) siempre esté visible. Su valor defecto es `true`.

```lua
vim.opt.wrap = true
```

* `breakindent`

Conserva la indentación de las líneas que sólo son visibles cuando `wrap` es `true`.

```lua
vim.opt.breakindent = true
```

* `tabstop`

La cantidad de carácteres que ocupa `Tab`. El valor por defecto es 8. Yo prefiero 2.

```lua
vim.opt.tabstop = 2
```

* `shiftwidth`

El espacio que Neovim usará para indentar una línea. Esta opción afecta los atajos `<<` y `>>`. Su valor por defecto es 8. La convención es tener el mismo valor que `tabstop`.

```lua
vim.opt.shiftwidth = 2
```

* `expandtab`

Determina si Neovim debe transformar el carácter `Tab` en espacios. Su valor por defecto es `false`.

```lua
vim.opt.expandtab = false
```

En lo que se refiere a configuración de variables hay más características que podría mencionar, pero tenemos otras cosas qué hacer. Pueden encontrar más detalles del tema aquí: [Configurando neovim - Opciones del editor](@/tools/configuring-neovim-using-lua.es.md#opciones-del-editor).

## Atajos de teclado

Neovim no tiene suficientes, tenemos que crear más. Para esto debemos aprender a usar la función `vim.keymap.set`. Les mostraré un ejemplo.

```lua
vim.keymap.set('n', '<space>w', '<cmd>write<cr>', {desc = 'Guardar'})
```

Después de ejecutar esta función la combinación `Espacio` + `w` nos permitirá usar el comando `write`. Podremos guardar el archivo actual con `Espacio` + `w`.

Ahora déjenme explicarles qué hace cada parámetro.

```lua
vim.keymap.set({mode}, {lhs}, {rhs}, {opts})
```

* `{mode}` es el modo donde tendrá efecto nuestro atajo (puede ser una lista de modos). Pero no necesitamos nombres, necesitamos la abreviación. Estas son las más comunes.

  - `n`: Modo normal.
  - `i`: Modo de inserción.
  - `x`: Modo visual.
  - `s`: Modo de selección.
  - `v`: Visual y selección.
  - `t`: Modo de terminal.
  - `o`: Modo de espera de operador.
  - `''`: Sí, una cadena de texto vacía. Es el equivalente a `n` + `v` + `o`.

* `{lhs}` es el atajo que queremos crear.

* `{rhs}` es la acción que queremos ejecutar. Puede ser un comando, una expresión o una función de lua.

* `{opts}` este parámetro debe ser una tabla de lua. Si no saben qué es una tabla, sólo piensen que es una manera albergar varios tipos de datos en un lugar. En fin, estas son las propiedades de uso común.

  - `desc`: Cadena de texto que describe qué hace el comando. Aquí podemos escribir cualquier cosa.

  - `remap`: Booleano que controla si nuestro atajo debe ser recursivo. Su valor por defecto es `false`. Los atajos recursivos pueden crear conflictos, así que no lo habiliten si no saben todos los detalles de lo que quieren lograr. Luego les explico con más detalle.

  - `buffer`: Puede ser un Booleano o un número. Si el valor es el booleano `true` quiere decir que al atajo sólo tendrá efecto en el archivo actual. Si es un número deberá ser el "id" de un archivo que tenemos abierto.

  - `silent`: Booleano que controla si el atajo puede mostrar un mensaje. Su valor por defecto es `false`.

  - `expr`: Booleano. Si lo habilitamos tendremos la posibilidad de usar vimscript o lua para generar el valor del parámetro `{rhs}`. Su valor por defecto es `false`.

### La tecla líder

Cuando definimos nuestros atajos podemos usar la secuencia especial `<leader>` en el parámetro `{lhs}`, esta toma el valor que tenemos en la variable global `mapleader`.

`mapleader` es una variable que debe ser una cadena texto. Por ejemplo.

```lua
vim.g.mapleader = ','
```

Con esto podríamos usar la tecla `,` como un prefijo para nuestros atajos.

```lua
vim.keymap.set('n', '<leader>w', '<cmd>write<cr>')
```

Y así la secuencia `,` + `w` guarda el archivo actual.

¿Qué pasa si no definimos `mapleader`? Por defecto tiene el valor `\`, que no es lo mejor del mundo. Mi recomendación para la tecla líder es usar `Espacio`. Pueden hacer esto.

```lua
vim.g.mapleader = ' '
```

### Atajos

Ahora les mostraré algunos atajos que pueden ser útiles para ustedes.

* Copiar y pegar del portapapeles

Por defecto Neovim (y Vim) no interactúa con el portapapeles. Cuando copiamos algún texto con la tecla `y` el contenido se va un registro interno. Yo prefiero conservar esta funcionalidad y crear atajos dedicados para manipular el portapapeles.

Copia al portapapeles.

```lua
vim.keymap.set({'n', 'x'}, 'gy', '"+y')
```

Pegar desde el portapapeles.

```lua
vim.keymap.set({'n', 'x'}, 'gp', '"+p')
```

* Borrar texto sin alterar el registro

Cuando borramos algo en modo normal o visual usando `c`, `d` o `x` ese texto se va un registro. Esto afecta el texto que podemos pegar con la tecla `p`. Lo que quiero hacer es modificar `x` y `X` para poder borrar texto sin afectar el "historial" de copias.

```lua
vim.keymap.set({'n', 'x'}, 'x', '"_x')
vim.keymap.set({'n', 'x'}, 'X', '"_d')
```

Con estos atajos `x` puede borrar un caracter en modo normal, en modo visual va a borrar la selección actual. La `X` va a tener el mismo comportamiento del comando `d`.

* Seleccionar todo el texto

```lua
vim.keymap.set('n', '<leader>a', ':keepjumps normal! ggVG<CR>')
```

## Plugin manager

Vamos a usar [lazy.nvim](https://github.com/folke/lazy.nvim). Me agrada porque podemos crear una configuración simple donde solamente escribimos una lista de plugins, pero también podemos crear módulos para la configuración de plugins. Por el momento mostraré el uso más simple.

Nuestro primer paso será instalar lazy.nvim. En la documentación nos muestran cómo hacerlo usando lua. Vamos a agregar esto.

```lua
local lazy = {}

function lazy.install(path)
  if not vim.loop.fs_stat(path) then
    print('Installing lazy.nvim....')
    vim.fn.system({
      'git',
      'clone',
      '--filter=blob:none',
      'https://github.com/folke/lazy.nvim.git',
      '--branch=stable', -- latest stable release
      path,
    })
  end
end

function lazy.setup(plugins)
  if vim.g.plugins_ready then
    return
  end

  -- Pueden comentar la siguiente línea una vez que lazy.nvim esté instalado
  lazy.install(lazy.path)

  vim.opt.rtp:prepend(lazy.path)

  require('lazy').setup(plugins, lazy.opts)
  vim.g.plugins_ready = true
end
```

Entonces, aquí tenemos dos funciones que están esperando para ser invocadas. Yo prefiero hacerlo de esta manera porque así puedo aislar el código que viene de la documentación.

Ahora tenemos que agregar nuestra configuración. Debemos especificar la ruta donde se instalarán los plugins, usamos la `lazy.path` para eso. Si quieren configurar lazy.nvim utilizan la variable `lazy.opts`, yo por el momento dejo la configuración vacía. Y para terminar vamos a colocar la lista de plugins en la función `lazy.setup`.

```lua
lazy.path = vim.fn.stdpath('data') .. '/lazy/lazy.nvim'
lazy.opts = {}

lazy.setup({
  ---
  -- Lista de plugins
  ---
})
```

Aquí utilizamos `stdpath('data')` para que la ruta de los plugins quede en una carpeta manejada por Neovim. Ustedes pueden cambiar la ruta si quieren. Pero de esta manera no tenemos que preocuparnos por cambiar la ruta dependiendo del sistema operativo donde estamos.

Si quieren saber donde estarán los plugins usen este comando.

```vim
:echo stdpath('data') . '/lazy/lazy.nvim'
```

Ahora vamos a descargar un plugin, un tema para que Neovim se vea mejor. Vamos a agregar esto en `lazy.setup`.

```lua
{'folke/tokyonight.nvim'},
```

Esta es la cantidad mínima de información que lazy.nvim necesita para descargar un plugin de github. Solamente especificamos el usuario de github y el nombre del repositorio.

Su configuración va a quedar de esta manera.

```lua
lazy.path = vim.fn.stdpath('data') .. '/lazy/lazy.nvim'
lazy.opts = {}

lazy.setup({
  {'folke/tokyonight.nvim'},
})
```

Ahora vamos a agregar el código para aplicar el tema.

```lua
vim.opt.termguicolors = true
vim.cmd.colorscheme('tokyonight')
```

Aquí habilitamos la opción `termguicolors` de Neovim para asegurarnos de tener la "mejor versión" del tema. Cada tema puede tener dos versiones, una para terminales que sólo soportan 256 colores y otra que utiliza códigos en hexadecimal (tiene más variedad de colores).

Para decirle a Neovim qué tema queremos usamos el comando `colorscheme`. Okey... técnicamente estamos usando una función de lua, pero esa función esta ejecutando un comando de vim.

```lua
colorscheme tokyonight
```

Si guardamos los cambios y reiniciamos Neovim deberá aparecer un mensaje que nos muestra que se está clonando el repositorio de lazy.nvim. Una vez que lazy.nvim lea nuestra lista de plugins deberá aparecer una ventana extra, esta nos muestra la descarga de los plugins. Deberíamos ver los cambios de manera inmediata luego de que lazy.nvim termine la descarga del tema.

## Configuración de plugins

Cada autor puede crear el método de configuración que mejor le parezca. ¿Cómo sabemos qué debemos hacer? Leemos la documentación, no nos queda de otra.

Cada plugin como mínimo debe tener un archivo llamado `README.md`. Este es el archivo que github nos muestra en la página principal del repositorio. Ahí debemos buscar las instrucciones de configuración.

Si el archivo `README.md` no tiene la información que nos interesa busquen una carpeta llamada `doc`. Por lo general ahí se encuentra un archivo `txt`, esa es la página de ayuda. Podemos lear ese archivo desde github o desde Neovim si usamos el comando `:help nombre-archivo`.

### Convenciones de plugins escritos en lua

Por suerte para nosotros muchos de los plugins escritos en lua siguen un patrón, usan una función llamada `setup` que acepta una tabla de lua como argumento. Si hay algo que deben aprender para configurar plugins en lua es cómo crear tablas.

Vamos a configurar un plugin. Para este ejemplo voy a usar [lualine](https://github.com/nvim-lualine/lualine.nvim), un plugin que modifica la "línea de estado" que se encuentra en el fondo de la pantalla. Para descargarlo agregamos `nvim-lualine/lualine.nvim` a la lista de plugins.

```lua
lazy.setup({
  {'folke/tokyonight.nvim'},
  {'nvim-lualine/lualine.nvim'},
})
```

Podemos empezar la configuración. Si revisan en github notarán que hay un archivo llamado `lualine.txt` en la carpeta `doc`. Podemos leerlo desde Neovim usando.

```vim
:help lualine
```

La documentación nos dice que debemos llamar la función `setup` del módulo `lualine`. Así.

```lua
require('lualine').setup()
```

Eso debería ser suficiente para que funcione. Podemos guardar los cambios y reiniciar Neovim.

Luce bien y todo pero yo quiero modificar algunas opciones. Por ejemplo, no quiero usar íconos, ¿cómo los deshabilito? En la sección `lualine-Default-configuration` puedo ver algo que dice `icons_enabled = true`. Lo que voy a hacer es copiar las propiedades que necesito para desactivarlo.

```lua
require('lualine').setup({
  options = {
    icons_enabled = false,
  }
})
```

Digamos que tenemos un problema con nuestra fuente, no podemos ver los "separadores de componentes". Tenemos que desactivarlos. Si nos vamos a la  sección `lualine-Disabling-separators` nos muestra esto.

```lua
options = { section_separators = '', component_separators = '' }
```

No vamos a copiar/pegar ese código como está. Tenemos que interpretarlo. Nos muestra la propiedad `options`, nosotros ya tenemos una de esas, entonces lo que hacemos es agregar las propiedades nuevas.

```lua
require('lualine').setup({
  options = {
    icons_enabled = false,
    section_separators = '',
    component_separators = ''
  }
})
```

Ahora que tenemos lualine instalado podemos guardar el archivo y ejecutar `:source $MYVIMRC` para ver los cambios.

En resumen, para configurar plugins debemos: saber cómo navegar en la documentación del plugin y también debemos conocer la sintaxis de lua para crear tablas.

### Plugins en vimscript

Aún hay muchos plugins útiles que están escritos en vimscript. En la mayoría de los casos los configuramos usando variables globales. En lua podremos modificar variables globales de vimscript usando `vim.g`.

¿Ya les conté que Neovim viene con un explorador de archivos? Podemos acceder a él con el comando `:Lexplore`. Este explorador es de hecho un plugin escrito en vimscript, entonces no tenemos una función `setup`. Para saber cómo configurarlo debemos buscar en su documentación.

```vim
:help netrw
```

Si revisan la tabla de contenido de esa página de ayuda notarán una sección llamada `netrw-browser-settings`. Ahí nos muestran una lista de variables y sus descripciones. Vamos a fijarnos en las que comienzan con el prefijo `g:`.

Por ejemplo si queremos ocultar el texto de ayuda usamos `netrw_banner`.

```lua
vim.g.netrw_banner = 0
```

Podríamos cambiar el tamaño de la ventana.

```lua
vim.g.netrw_winsize = 30
```

Eso es todo... bueno, hay más variables que pueden modificar pero en general esto es lo más básico que deben saber. Revisan la documentación, identifican la variable que quieren modificar y la cambian con `vim.g`.

## Información extra

### Atajos recursivos y no recursivos

Si ya conocen la palabra "recursividad" tal vez puedan intuir qué consecuencias tienen este tipo de atajos. Si no, dejénme demostrarles con un ejemplo.

Digamos que tenemos este atajo que abre el explorador de archivos.

```lua
vim.keymap.set('n', '<F2>', '<cmd>Lexplore<cr>')
```

Bien, ahora vamos a crear un atajo recursivo que utilice `F2`.

```lua
vim.keymap.set('n', '<space><space>', '<F2>', {remap = true})
```

Si pulsan `Espacio` dos veces seguidas les aparecerá el explorador. Pero si cambian `remap` de `true` a `false`, no aparecerá nada.

Con los atajos recursivos, podremos usar otros atajos definidos por nosotros mismos o por plugins en el parámetro `{rhs}`. Con los atajos "no recursivos" sólo tendremos acceso a los atajos definidos directamente por Neovim.

Por lo general sólo queremos atajos recursivos cuando vamos a usar funcionalidades creadas por plugins.

¿Por qué los atajos recursivos pueden causar conflictos? Consideren esto.

```lua
vim.keymap.set('n', '*', '*zz')
```

Noten que estamos usando `*` en `{lhs}` y también en `{rhs}`. Si este atajo fuera recursivo estaríamos creando un ciclo infinito. Neovim intentará llamar a la funcionalidad atada a `*` y nunca ejecuta `zz`.

### Comandos de usuario

Sí, Neovim nos permite crear nuestros propios comandos. En lua usamos esta función.

```lua
vim.api.nvim_create_user_command({name}, {command}, {opts})
```

* `{name}` es el nombre del comando, debe una cadena de texto y tiene que empezar con una letra mayúscula.

* `{command}` es la acción que queremos ejecutar. Puede una cadena de texto que contiene un fragmento de vimscript o puede ser una función de lua.

* `{opts}` debe ser una tabla de lua. No es opcional. Si no especifican ninguna opción deben colocar una tabla vacía.

Ejemplo, podemos crear un comando dedicado para recargar nuestra configuración.

```lua
vim.api.nvim_create_user_command('ReloadConfig', 'source $MYVIMRC', {})
```

Si quieren saber más detalles de los comandos de usuario revisen la documentación con los comandos.

```vim
:help nvim_create_user_command()
:help user-commands
```

### Autocomandos

Los autocomandos son acciones que Neovim puede ejecutar cuando ocurre un evento. Pueden revisar la lista de eventos con el comando `:help events`.

Podemos crear un autocomando con esta función.

```lua
vim.api.nvim_create_autocmd({event}, {opts})
```

* `{event}` debe ser el nombre de un evento.

* `{opts}` es una tabla de lua. Son las opciones que determinan el comportamiento del autocomando. Estas son algunas propiedades de uso común.

  - `desc` es una cadena de texto que describe lo que hace el autocomando.

  - `group` debe ser un número que representa el `id` de un grupo o una cadena de texto con el nombre de un grupo.

  - `pattern` puede ser una tabla de lua o una cadena de texto. Nos permite filtrar los eventos en los que queremos ejecutar la acción. Su valor depende del evento. Deben revisar la documentación del evento para saber los posibles valores. Esta propiedad es opcional.

  - `once` debe ser un booleano. Si lo habilitamos el autocomando sólo se ejecutará una vez. Su valor por defecto es `false`.

  - `command` es una cadena de texto con un fragmento de vimscript. Es la acción que ejecutará el autocomando.

  - `callback` puede ser una cadena de texto con el nombre de una función de vimscript o una función de lua. Es la acción que ejecutará el autocomando. No puede usarse en combinación con `command`.

Aquí les va un ejemplo. Voy a crear un grupo llamado `user_cmds` y le agregaré dos autocomandos.

```lua
local augroup = vim.api.nvim_create_augroup('user_cmds', {clear = true})

vim.api.nvim_create_autocmd('FileType', {
  pattern = {'help', 'man'},
  group = augroup,
  desc = 'Usar q para cerrar ventana',
  command = 'nnoremap <buffer> q <cmd>quit<cr>'
})

vim.api.nvim_create_autocmd('TextYankPost', {
  group = augroup,
  desc = 'Resaltar texto copiado',
  callback = function(event)
    vim.highlight.on_yank({higroup = 'Visual', timeout = 200})
  end
})
```

Crear el grupo es opcional.

El primer autocomando creará el atajo `q` para cerrar la ventana actual, pero sólo si el archivo actual es de tipo `help` o `man`. En este ejemplo estoy usando vimscript en la propiedad `command`, pero también pude haberlo hecho usando lua con la propiedad `callback`.

El segundo autocomando usa una función de lua en la propiedad `callback`. Esta función lo que hace es resaltar el texto que copiamos. Pueden probar su efecto si copian una línea usando el atajo `yy`.

Para conocer más detalles de los autocomandos revisen la documentación.

```vim
:help autocmd-intro
```

### Módulos de usuario

No hay ninguna regla que nos obligue a tener toda nuestra configuración en un solo archivo. Podemos crear módulos para separar la configuración en piezas más pequeñas.

La convención es colocar todos nuestros módulos en un "espacio único" para evitar conflictos con plugins. Muchas personas crean un módulo llamado `user` (ustedes pueden darle otro nombre si quieren). Para esto debemos ir la carpeta donde está `init.lua`, crear el directorio `lua`, y dentro creamos `user`. Podemos hacer todo eso con un comando.

```vim
:call mkdir(stdpath("config") . "/lua/user", "p")
```

Ya dentro de `/lua/user` podemos crear nuestros scripts de lua. Vamos a suponer que tenemos uno llamado `settings.lua`. Ahora, Neovim no sabe que ese script existe, no lo va a ejecutar, nosotros debemos llamarlo desde `init.lua` así.

```lua
require('user.settings')
```

Si quieren saber con detalle cómo funciona require... revisen la documentación.

```vim
:help lua-require
```

### La función require

Algo que deben saber de `require` es que sólo ejecuta el módulo una vez. ¿Qué quiere decir esto?

Consideren este código.

```lua
require('user.settings')
require('user.settings')
```

Aquí el script `settings.lua` sólo se ejecutará una vez. Si desean crear plugins o alguna funcionalidad que depende de un plugin, esto es bueno. Lo malo es que si quieren usar el comando `:source $MYVIMRC` para "recargar" su configuración no obtendrán el resultado que quieren.

Hay un truco que pueden hacer. Pueden invalidar el caché de la función `require` antes de invocarla con un módulo. Ejemplo.

```lua
local load = function(mod)
  package.loaded[mod] = nil
  require(mod)
end

load('user.settings')
load('user.keymaps')
```

Sí hacemos esto en `init.lua` el comando `source` podrá ejecutar los módulos `settings` y `keymaps` de manera efectiva.

**ADVERTENCIA**. Deben tener en cuenta que algunos plugins pueden actuar de manera extraña si los configuran más de una vez. Es decir, si ustedes usan `:source $MYVIMRC` y provocan que la función `setup` de un plugin se ejecute por segunda vez puede causar efectos inesperados.

## init.lua

Entonces, si aplican (casi) todo lo que les enseñé hoy este sería el resultado.

* [init.lua](https://github.com/VonHeikemen/nvim-starter/blob/01-base/init.lua)

## ¿Qué sigue?

El siguiente paso es crear un ambiente de desarrollo en el que se sientan productivos. Investiguen qué plugins utiliza la comunidad de Neovim.

Este paso puede ser difícil si no saben dónde empezar, por eso he creado una "plantilla" que pueden revisar o incluso usar como base para su propia configuración personal.

* nvim-light: [init.lua](https://github.com/VonHeikemen/nvim-light/blob/main/init.lua) | [github link](https://github.com/VonHeikemen/nvim-light)

Otra buena opción sería ir al proyecto [kickstart.nvim](https://github.com/nvim-lua/kickstart.nvim). El autor de esta configuración tiene un video donde explica cada línea de código. Es un buen recurso para aprender sobre cómo configurar plugins populares.

* [Complete Neovim setup guide](https://www.youtube.com/watch?v=m8C0Cq9Uv9o)

Si quieren, también pueden revisar mi configuración personal.

* [neovim config](https://github.com/VonHeikemen/dotfiles/tree/master/my-configs/neovim)

## Conclusión

Ahora sabemos cómo modificar las opciones básicas de Neovim. Aprendimos cómo crear nuestros propios atajos. Podemos hacer que Neovim descargue plugins desde github. Configuramos un par de plugins, uno escrito en lua, otro escrito en vimscript. Dimos un vistazo a temas como atajos recursivos, comandos de usuario, autocomandos, módulos y algunos trucos.

En este punto ya tenemos todas la herramientas para explorar plugins, ver las configuraciones de otras personas y aprender de ellas.

