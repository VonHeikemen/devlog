+++
title = "Neovim: Plugins para empezar"
description = "Explorando plugins para mejorar nuestra experiencia en Neovim"
date = 2022-09-03
updated = 2026-02-02
lang = "es"
[taxonomies]
tags = ["neovim", "vim", "shell"]
[extra]
shared = [
  ["dev.to", "https://dev.to/vonheikemen/neovim-plugins-para-empezar-c39"],
  ["Hashnode", "https://vonheikemen.hashnode.dev/neovim-plugins-to-get-started-es"],
]
+++

Suele pasar que una persona quiere usar Neovim para escribir código pero hay ciertas funcionalidades de otros editores que extrañan. Como por ejemplo tener la lista de archivos abiertos en pestañas, un explorador de archivos con una estructura de árbol, un indicador que muestre la rama de git, entre otras cosas. Bueno, aquí quiero mostrarles algunos plugins que pueden usar para implementar esas funcionalidades en Neovim.

Lo único que no cubriré será el autocompletado de código. Configurar un autocompletado inteligente involucra instalar herramientas externas. Esto conlleva una buena cantidad de detalles que deben ser explicados. Para eso les recomiendo leer esto: [Configurando el cliente LSP de neovim](https://dev.to/vonheikemen/configurando-el-cliente-lsp-nativo-de-neovim-en-2022-la-manera-facil-3c17).

Todo el código de configuración que mostraré en esta guía estará este repositorio: [nvim-starter - branch: 02-opinionated](https://github.com/VonHeikemen/nvim-starter/tree/02-opinionated).

## Requisitos

Si ustedes son completamente nuevos en Neovim les recomiendo que aprendan [la sintaxis de lua](https://learnxinyminutes.com/docs/lua/). No es necesario saber todo o dominar el lenguaje, como mínimo tengan a la mano una referencia de la sintaxis para saber qué es válido. Casi todos los plugins que presentaré se configuran en ese lenguaje.

Si todavía no han creado un archivo de configuración para Neovim, háganlo ahora. Aquí les dejo una guía con todo lo que necesitan saber: [Cómo crear tu primera configuración de Neovim usando lua](@/tools/build-your-first-lua-config-for-neovim.es.md).

También sería una buena idea descargar la versión estable más reciente de Neovim. Pueden descargarla desde la [sección releases](https://github.com/neovim/neovim/releases) en github. De aquí en adelante voy a asumir que están usando la versión 0.9.5 o mayor.

## ¿Cómo instalamos un plugin?

Lo primero que deben saber es cómo instalar un plugin de forma manual.

Para instalar un plugin sólo debemos descargarlo en una ubicación específica.

La manera "tradicional" de conocer los directorios disponibles para nuestros plugins es usando este comando.

```vim
:set packpath?
```

Neovim nos muestra una lista separada por comas. A mi parecer este formato resulta difícil de leer. Podemos hacer algo mejor aprovechando las capacidades de lua.

```lua
:lua vim.tbl_map(print, vim.opt.packpath:get())
```

Con esto Neovim les mostrará la misma lista pero cada directorio se muestra en una línea.

En uno de esos directorios debemos crear una carpeta llamada `pack`, y dentro de pack debemos crear el directorio que tendrá nuestros plugins.

La estructura de archivos debe ser algo como esto:

```txt
pack
└── plugins-de-github
    ├── opt
    │   ├── [plugin 1]
    │   └── [plugin 2]
    └── start
        └── [plugin 3]
```

Los plugins en el directorio `opt` son considerados opcionales, Neovim no los usará hasta que nosotros invoquemos el comando `:packadd` con el nombre del plugin. Los plugins en el directorio `start` se cargarán de manera automática durante el proceso de inicialización de Neovim.

Vamos a suponer que tenemos este directorio en nuestro `packpath`.

```
/home/dev/.local/share/nvim/site
```

Entonces el primer paso será crear el directorio `pack`. Luego, dentro de pack, creamos otro directorio. A ese directorio podemos darle cualquier nombre. Vamos a usar el nombre `github` en este ejemplo. Quiere decir que esta es una ubicación válida para nuestros plugins.

```
/home/dev/.local/share/nvim/site/pack/github
```

Para instalar un plugin como [mini.nvim](https://github.com/nvim-mini/mini.nvim) y poder usarlo de manera inmediata debemos descargarlo aquí.

```
/home/dev/.local/share/nvim/site/pack/github/start/mini.nvim
```

Y eso es todo.

Para conocer más detalles de cómo usar plugins desde el directorio `pack` pueden leer la documentación.

```vim
:help packages
```

### Manejador de plugins

Pero claro nadie está obligado a descargar plugins manualmente. Pueden automatizar todo el proceso con un manejador de plugins. 

En la actualidad estos son los más manejadores de plugins más populares en el ecosistema de Neovim.

* [lazy.nvim](https://github.com/folke/lazy.nvim)
* [mini.deps](https://nvim-mini.org/mini.nvim/readmes/mini-deps.html)
* [paq.nvim](https://github.com/savq/paq-nvim) 

Recuerden leer bien la documentación del manejador de plugin que decidan usar.

## Plugins

### Tokyonight

Github: [folke/tokyonight.nvim](https://github.com/folke/tokyonight.nvim)

Para cambiar el tema que Neovim trae por defecto debemos invocar el comando `colorscheme` seguido del nombre del tema.

Ahora bien, en lua para ejecutar comandos usamos `vim.cmd`. Ejemplo:

```lua
vim.cmd('colorscheme nombre-tema')
```

En nuestro caso queremos usar tokyonight, así que nuestro comando sería este.

```vim
colorscheme tokyonight
```

### Bufferline

Github: [akinsho/bufferline.nvim](https://github.com/akinsho/bufferline.nvim)

En Neovim las pestañas son como espacios de trabajo, pueden mostrar varios archivos en una pestaña e incluso cambiar el directorio de trabajo por pestaña. Muchas personas prefieren mostrar un archivo por pestaña, como en otros editores. Eso es exactamente lo que hace `bufferline`, modifica las pestañas para mostrar todos los archivos abiertos.

En este caso nos encontramos con un plugin que debe ser "activado" manualmente. Aquí debemos invocar la función `.setup()` del módulo principal del plugin. Entonces para que bufferline funcione debemos agregar esta línea de código a nuestra configuración.

```lua
require('bufferline').setup({})
```

La función `require` es parte del lenguaje lua, es el mecanismo que usamos para cargar un módulo. La cadena de texto `bufferline` es el nombre del módulo. Y `.setup()` es la función que queremos ejecutar.

Para configurar el plugin debemos colocar nuestras preferencias en una "tabla de lua" dentro de la función `.setup()`. ¿Cómo sabemos qué opciones tenemos disponibles? Debemos leer la documentación del plugin.

Pueden encontrar una referencia con todas las opciones con este comando.

```vim
:help bufferline-configuration
```

Este es un ejemplo.

```lua
require('bufferline').setup({
  options = {
    mode = 'buffers',
    offsets = {
      {filetype = 'snacks_layout_box'}
    },
  }
  highlights = {
    buffer_selected = {
      italic = false
    },
    indicator_selected = {
      fg = {attribute = 'fg', highlight = 'Function'},
      italic = false
    }
  }
})
```

* `options.mode`: Con el valor `'buffers'` indicamos que queremos "pestañas tradicionales" que muestran la lista archivos abiertos.

* `options.offsets`: Es una lista de tipos de archivos. Cuando aparece una ventana de este tipo `bufferline` evita colocar una pestaña sobre ella. Aquí le indicamos el tipo de archivo `snacks_layout_box` porque es el tipo de archivo que tendrá el explorador de archivos que usaremos. Esto hará que se logre el efecto de barra lateral.

* `highlights`: Nos permite modificar los colores en ciertas áreas. Cada sección (como `buffer_selected`) se refiere a un área. En este ejemplo coloco `italic = false` para desactivar las letras cursivas. La opción `fg` cambia el color de las letras. En este ejemplo le indico que debe tomar el mismo color que utiliza mi tema para las funciones. Pueden ver los detalles de la opción highlights usando el comando `:help bufferline-highlights`.

Vale la pena mencionar que bufferline ofrece varios comandos que podemos usar para navegar entre archivos abiertos. Entre esos comandos está `BufferLinePick`. Con esto podremos seleccionar cualquier archivo que esté en una pestaña visible en pantalla. Podemos utilizar este comando en un atajo de teclado.

```lua
vim.keymap.set('n', 'gt', '<cmd>BufferLinePick<cr>', {})
```

### mini.nvim

Github: [nvim-mini/mini.nvim](https://github.com/nvim-mini/mini.nvim)

Website: [nvim-mini.org](https://nvim-mini.org)

`mini.nvim` es una colección de módulos escritos en lua. Este proyecto tiene el objetivo de complementar las capacidades nativas de Neovim, y en algunos casos también busca implementar nuevas funcionalidades.

mini.nvim tiene más de 40 módulos pero aquí sólo les mostraré 7.

* [mini.statusline](https://nvim-mini.org/mini.nvim/doc/mini-statusline.html)

El "statusline" es una parte de la interfaz de Neovim. Es esa línea que se encuentra casi al final de la pantalla, donde se muestra el nombre del archivo actual y la ubicación del cursor. Usualmente se encuentra por encima del área de mensajes.

`mini.statusline` modifica la opción [statusline](https://neovim.io/doc/user/options.html#'statusline') con una implementación que muestra más información. También puede mostrar información que proviene de otros módulos en mini.nvim.

Igual que como ocurre con el plugin bufferline, en mini.nvim debemos habilitar cada módulo de manera explícita. Esto quiere decir que debemos ejecutar la función `.setup()` del módulo que queremos usar.

En el caso de `mini.statusline` esto sería suficiente para hacer que funcione.

```lua
require('mini.statusline').setup({})
```

Si desean saber más detalles de un módulo en mini.nvim usen el comando `:help` seguido del nombre del módulo.

```
:help mini.statusline
```

* [mini.git](https://nvim-mini.org/mini.nvim/doc/mini-git.html)

Si nos encontramos en un repositorio de git `mini.git` puede ayudarnos a obtener información sobre el estado actual del repositorio. También ofrece un comando (`:Git`) con el cual podemos usar git dentro de Neovim.

Por defecto `mini.statusline` puede mostrar la rama de git si activamos `mini.git`.

```lua
require('mini.git').setup({})
```

El comando `:Git` que nos ofrece este plugin intenta tanto como sea posible integrar Neovim con git. Por ejemplo, el comando `:Git diff` muestra el resultado dentro de Neovim. `:Git commit` intentará usar la instancia actual de Neovim para escribir el mensaje del commit. Para conocer más detalles pueden revisar la documentación.

```
:help mini.git
```

* [mini.diff](https://nvim-mini.org/mini.nvim/doc/mini-diff.html)

`mini.diff` es otro módulo que depende de git. En este caso se muestra las modificaciones que se hicieron en el archivo actual. Puede mostrar cuando una línea ha sido modificada, borrada o si se añadió una línea nueva. Todo en tiempo real.

Aquí les comparto mi configuración personal.

```lua
vim.o.signcolumn = 'yes'
require('mini.diff').setup({
  view = {
    style = 'sign',
    signs = {
      add = '▎',
      change = '▎',
      delete = '➤',
    },
  },
})
```

Aquí antes de habilitar el plugin modificamos la opción `signcolumn` con el valor `yes`, esto lo hacemos para reservar un espacio para los íconos de `mini.diff`. Dentro de las opciones para `mini.diff` especificamos el estilo "sign" que es lo que nos permite usar íconos como indicadores.

* [mini.comment](https://nvim-mini.org/mini.nvim/doc/mini-comment.html)

`mini.comment` provee atajos de teclado para comentar o "descomentar" líneas de código. Por defecto utiliza el atajo `gc` como un operador en modo normal. Por ejemplo, la combinación `gci{` puede comentar (o descomentar) todas las líneas de comentario dentro de un bloque de código delimitado por llaves. Cualquier combinación que uno puede hacer con un operador nativo, como `d` o `y`, también lo podemos hacer con el operador `gc` que `mini.comment` provee.

Adicionalmente `mini.comment` crea el atajo `gcc`. Con `gcc` podemos comentar la línea actual.

```lua
require('mini.comment').setup({})
```

Vale la pena mencionar que esta funcionalidad que ofrece `mini.comment` ya forma parte de Neovim. Fue agregada en la versión v0.10 de Neovim. Pero la implementación de Neovim no es muy flexible, no se puede cambiar o extender de ninguna manera. El beneficio de `mini.comment` es que funciona incluso en Neovim v0.9 y nos da la oportunidad de cambiar algunas opciones, e incluso podemos crear nuestra propia implementación para la función que decide la sintaxis del comentario.

* [mini.notify](https://nvim-mini.org/mini.nvim/doc/mini-notify.html)

`mini.notify` es una implementación para la función `vim.notify`.

`vim.notify` es una función de lua que cualquier plugin puede usar para mostrar un mensaje a un usuario final. La implementación nativa de Neovim muestra los mensajes en el área de mensajes. `mini.notify` muestra los mensajes en una ventana flotante en una esquina de la pantalla.

```lua
require('mini.notify').setup({})
```

* [mini.clue](https://nvim-mini.org/mini.nvim/doc/mini-clue.html)

`mini.clue` puede ayudarles a recordar (o tal vez descubrir) atajos de teclado mostrándoles posibles combinaciones en una ventana flotante.

La mayoría de los módulos en mini.nvim tienen una buena configuración por defecto. Quiere decir que sólo debemos invocar la función `.setup()` y usar el plugin. Pero `mini.clue` es una excepción, aquí debemos especificar las teclas que activan la ventana flotante. Este es un ejemplo:

```lua
local mini_clue = require('mini.clue')

mini_clue.setup({
  triggers = {
    {mode = 'n', keys = 'g'},
    {mode = 'n', keys = '<leader>'},
  },
  clues = {
    mini_clue.gen_clues.g(),
  },
})
```

En la opción `triggers` agregamos la combinación que activa la ventana flotante. En `clues` es donde agregamos la descripción de la acción que hace un atajo de teclado en caso de ser necesario.

Lo bueno de `mini.clue` es que se integra muy bien con Neovim. Cuando creamos nuestros atajos de teclado personales. Si creamos nuestros atajos con la función `vim.keymap.set()` y proveemos una descripción, eso será suficiente para que `mini.clue` muestre nuestro atajo en la ventana flotante.

Consideren este código.

```lua
-- Usar tecla Espacio como <leader>
vim.g.mapleader = ' '

vim.keymap.set('n', '<leader>w', '<cmd>write<cr>', {desc = 'Guardar'})
vim.keymap.set('n', '<leader>q', '<cmd>quit<cr>', {desc = 'Cerrar ventana'})
```

Aquí creamos dos atajos de teclado que empiezan con la tecla líder (`<leader>`), que en este caso es `Espacio`. Como ya tenemos `<leader>` en la opción `triggers` de `mini.clue` nuestros atajos aparecerán en la ventana flotante.

Para agregar descripciones a los atajos nativos de Neovim debemos usar la propiedad [gen_clues](https://nvim-mini.org/mini.nvim/doc/mini-clue.html#miniclue.gen_clues).

Una configuración más completa para `mini.clue` sería algo así:

```lua
local mini_clue = require('mini.clue')

mini_clue.setup({
  window = {
    delay = 600,
    config = {
      width = 50,
    },
  },
  triggers = {
    {mode = 'n', keys = '['},
    {mode = 'n', keys = ']'},
    {mode = 'n', keys = 'g'},
    {mode = 'x', keys = 'g'},
    {mode = 'n', keys = 'z'},
    {mode = 'x', keys = 'z'},
    {mode = 'n', keys = '<C-w>'},
    {mode = 'i', keys = '<C-x>'},
    {mode = 'n', keys = '<leader>'},
    {mode = 'x', keys = '<leader>'},
  },
  clues = {
    mini_clue.gen_clues.g(),
    mini_clue.gen_clues.z(),
    mini_clue.gen_clues.windows(),
    mini_clue.gen_clues.builtin_completion(),
  },
})
```

Noten que en este ejemplo tenemos una nueva sección, `windows`. Aquí especificamos las opciones que controlan el comportamiento de la ventana flotante.

Ahora hablemos de limitaciones. Deben saber que `mini.clue` no funciona con operadores en modo normal. No podremos ver la ventana flotante para combinaciones como `ciw` o `dap`. Esto sucede porque los operadores como `c` y `d` no se pueden extender. No hay un mecanismo para ejecutar una acción en medio de un operador. Técnicamente es posible lograr el efecto pero requiere un esfuerzo enorme, y mini.nvim se especializa en módulos "ligeros" donde se busca un balance entre funcionalidad y líneas de código.

* [mini.icons](https://nvim-mini.org/mini.nvim/doc/mini-icons.html)

`mini.icons` podríamos considerarlo como un módulo multi-propósito.

Puede ser usado en plugins para mostrar íconos de manera fácil. Recuerden que Neovim funciona en emuladores de terminal. Para un emulador de terminal todo es texto. Un ícono no es una imagen es un código especial.

Para nosotros los usuarios casuales `mini.icons` nos da la oportunidad de habilitar o deshabilitar íconos. Verán, en ocasiones un plugin puede mostrar íconos en ciertas partes de la interfaz pero no ofrece una manera de ocultarlos o reemplazarlos con caracteres normales. `mini.icons` nos ofrece la posibilidad de desactivar íconos incluso si los plugins que usamos no ofrecen esa opción.

```lua
require('mini.icons').setup({style = 'glyph'})
```

Aquí la opción `style` es la que controla la clase de íconos que queremos usar. `glyph` es el valor por defecto, este hará que `mini.icons` muestre glifos que parecen íconos. Si cambiamos `style` con el valor `ascii` entonces los glifos serán reemplazados con caracteres normales que cualquier terminal puede mostrar.

### vim-repeat

Github: [tpope/vim-repeat](https://github.com/tpope/vim-repeat)

Agrega soporte para repeticiones a comandos creados por plugins. Si no lo saben, si presionamos `.` Neovim repite la última acción que hicimos. Por ejemplo, si borramos una palabra usando `diw` podemos repetir esta acción simplemente presionando `.`. `vim-repeat` permite que las acciones de los plugins también pueda repetirse con `.`.

### Treesitter

Github: [nvim-treesitter/nvim-treesitter](https://github.com/nvim-treesitter/nvim-treesitter)

Treesitter es un componente que se encuentra dentro de Neovim, este le permite a Neovim leer código de la misma manera que un compilador. ¿Cómo es eso? Escanea el código, va recolectando información de cada símbolo y al final genera un árbol de sintaxis. En otras palabras, convierte tu código en una estructura de datos que Neovim puede consultar.

Por sí solo treesitter no nos trae ningún beneficio, es más una herramienta para los desarrolladores de Neovim y los autores de plugins. Son ellos los que usan treesitter para crear las funcionalidades con las que nosotros interactuamos.

Por ejemplo, dentro de Neovim existe un mecanismo alternativo para el resaltado de sintaxis. En ese caso se utiliza treesitter para asignar los "highlight groups" que serán utilizados por el tema del editor.

Ahora bien, para que treesitter funcione necesitamos algo llamado "treesitter parser." Este es el componente que se encarga de leer el código del archivo actual. Cada lenguaje de programación tiene su propio treesitter parser. Para instalar un parser podemos usar el comando `:TSInstall` seguido del nombre del lenguaje.

Si queremos instalar el parser para javascript utilizamos este comando.

```vim
:TSInstall javascript
```

Para funcionalidades como tal, en la actualidad no tenemos una interfaz para "habilitar treesitter." Lo que hacemos es revisar la documentación de la funcionalidad que queremos usar y seguimos las instrucciones.

Si queremos utilizar el resaltado de sintaxis basado en treesitter tenemos la opción de crear un autocomando o un "filetype plugin." Luego debemos invocar la función `vim.treesitter.start()`. Este es un ejemplo que usa un autocomando en los tipos de archivos relacionados con javascript.

```lua
vim.api.nvim_create_autocmd('FileType', {
  pattern = {'javascript', 'javascriptreact', 'js', 'jsx'},
  callback = function()
    vim.treesitter.start()
  end,
})
```

### ts-enable.nvim

Github: [VonHeikemen/ts-enable.nvim](https://github.com/VonHeikemen/ts-enable.nvim)

`nvim-treesitter` requiere de cierta cantidad de conocimiento sobre Neovim. Ciertamente no es nada del otro mundo si estamos dispuestos a instalar los parsers manualmente. Pero la cosa se pone complicada si queremos automatizar todo el proceso. Por eso existe `ts-enable.nvim`, este plugin implementa el código necesario para habilitar algunas funcionalidades basadas en treesitter y también puede instalar parsers cuando es necesario.

La configuración básica puede ser tan simple como esto:

```lua
-- See :help ts-enable-config
vim.g.ts_enable = {
  parsers = {'lua', 'vim', 'vimdoc', 'c', 'query'},
  auto_install = true,
  highlights = true,
  indents = false,
  folds = false,
}
```

La idea aquí es poder especificar los nombres de los parsers que queremos usar y las funcionalidades que queremos habilitar. `ts-enable.nvim` se encarga de crear los autocomandos e invocar las funciones necesarias, para que nosotros no tengamos que preocuparnos por los detalles técnicos.

### Snacks.nvim

Github: [folke/snacks.nvim](https://github.com/folke/snacks.nvim)

Snacks.nvim es una colección de módulos escritos en lua, al igual que mini.nvim. Pero en este caso Snacks.nvim tiene un diseño diferente. Por ejemplo, Snacks.nvim tiene un solo punto de entrada que es el módulo `snacks`, pero en `mini.nvim` cada módulo tiene su propio punto de entrada. Snacks.nvim es más ambicioso con sus funcionalidades, no intenta ser un plugin "ligero."

Para habilitar las funcionalidades en Snacks.nvim usamos la función `.setup()` del módulo `snacks`.

```lua
local Snacks = require('snacks')

Snacks.setup({
  ---
  -- Aquí debemos colocar nuestra configuración
  ---
})
```

Además de la función `.setup()` también podemos acceder a otros módulos a través de `snacks`.

* [Snacks.input](https://github.com/folke/snacks.nvim/blob/main/docs/input.md)

`Snacks.input` es una implementación para la función `vim.ui.input()`.

`vim.ui.input()` es una función que podemos usar en lua, los autores de plugins la usan para pedir información al usuario. Por ejemplo, en un explorador archivos, cuando se intenta renombrar un archivo tiene sentido preguntar el nuevo nombre del archivo. 

La implementación de `Snacks.input` muestra una ventana flotante donde podemos ingresar una cadena de texto. A diferencia de la implementación de Neovim donde ingresamos el texto en el área de mensajes.

Para habilitar el módulo agregamos esta configuración en la función `.setup()`.

```lua
Snacks.setup({
  input = {
    enabled = true,
    icon = '❯',
  },
})
```

* [Snacks.picker](https://github.com/folke/snacks.nvim/blob/main/docs/picker.md)

`Snacks.picker` hace tres cosas: 1) ofrece una interfaz para filtrar items de una lista. 2) implementa más de 50 filtros. 3) provee una implementación para la función `vim.ui.select()`.

Este tipo de plugins han ganado popularidad en la comunidad porque podemos usarlos para navegar rápidamente entre varios archivos. Y lo mejor de todo es que la idea de filtrar items es útil en otros escenarios. Por eso es que Snacks.nvim viene con varios tipos de filtros ya implementados. Podemos buscar archivos, proyectos, historial de cambios, atajos de teclado, temas para el editor... y mucho más.

`vim.ui.select()` es una función que podemos usar en lua para pedir a un usuario que elija un item entre varias opciones. La implementación de `Snacks.picker` usa la interfaz interactiva de `Snacks`.

```lua
Snacks.setup({
  picker = {
    enabled = true,
    ui_select = true,
    prompt = '❯ ',
  },
})
```

Vale la pena mencionar que configurar `picker` es opcional si no queremos usar la implementación de `vim.ui.select()`. Todos los filtros en Snacks funcionan incluso si no habilitamos el módulo de manera explícita en la función `.setup()`.

¿Cómo usamos estos filtros? La manera más conveniente sería invocarlos con un atajo de teclado.

```lua
-- docs: https://github.com/folke/snacks.nvim/blob/main/docs/picker.md
vim.keymap.set('n', '<leader><space>', function() Snacks.picker('buffers') end, {})
vim.keymap.set('n', '<leader>ff', function() Snacks.picker('files') end, {})
vim.keymap.set('n', '<leader>fh', function() Snacks.picker('recent') end, {})
vim.keymap.set('n', '<leader>fg', function() Snacks.picker('grep') end, {})
vim.keymap.set('n', '<leader>fd', function() Snacks.picker('diagnostics') end, {})
vim.keymap.set('n', '<leader>fs', function() Snacks.picker('lines') end, {})
vim.keymap.set('n', '<leader>u', function() Snacks.picker('undo') end, {})
vim.keymap.set('n', '<leader>/', function() Snacks.picker('pickers') end, {})
vim.keymap.set('n', '<leader>?', function() Snacks.picker('keymaps') end, {})
```

  - `<leader><space>`: Muestra la lista de archivos abiertos.
  - `<leader>ff`: Muestra los archivos del directorio de trabajo actual.
  - `<leader>fh`: Muestra el historial de archivos.
  - `<leader>fg`: Ejecuta una búsqueda interactiva en cada línea código de cada archivo en el directorio actual.
  - `<leader>fd`: Muestra la lista de "diagnósticos" del archivo actual. Un diagnóstico puede ser un error de sintaxis, una advertencia o una sugerencia.
  - `<leader>fs`: Ejecuta una búsqueda interactiva en el archivo actual.
  - `<leader>u`: Muestra el historial de cambios.
  - `<leader>/`: Muestra una lista de todos los filtros disponibles en Snacks.
  - `<leader>?`: Muestra una lista de atajos de teclado.

* [Snacks.indent](https://github.com/folke/snacks.nvim/blob/main/docs/indent.md)

`Snacks.indent` añade guías de indentación a todas las líneas del archivo.

```lua
Snacks.setup({
  indent = {
    enabled = true,
    char = '▏',
  },
})
```

Por defecto el bloque de código donde está el cursor tendrá una animación extra. Si queremos desactivarla debemos agregar esta variable global.

```lua
vim.g.snacks_animate = false
```

* [Snacks.bigfile](https://github.com/folke/snacks.nvim/blob/main/docs/bigfile.md)

Este módulo resulta más útil en versiones de Neovim anterior a v0.11.

Podemos usar `Snacks.bigfile` para deshabilitar funcinalidades que hacen que el editor disminuya el tiempo de respuesta o se congele completamente. La mayor parte del tiempo es el resaltado de sintaxis lo que hace al editor lento cuando abrimos archivos grandes. En Neovim v0.11 se ha agregado una funcionalidad de resaltado asíncrono, es por eso que recomiendo este módulo para aquellas personas que utilizan una versión de Neovim más antigua.

```lua
Snacks.setup({
  bigfile = {
    -- Only use `bigfile` module on older Neovim versions
    enabled = vim.fn.has('nvim-0.11') == 0,
    notify = false,
    size = 1024 * 1024, -- 1MB
    setup = function(ctx)
      vim.cmd('syntax clear')
      vim.opt_local.syntax = 'OFF'
      local buffer = vim.b[ctx.buf]
      if buffer.ts_highlight then
        vim.treesitter.stop(ctx.buf)
      end
    end
  },
})
```

La idea aquí es que en archivos que superan `1MB` de tamaño deshabilitamos el resaltado de sintaxis. Noten que en el campo `enabled` usamos una condición dinámica, esta sólo retorna `true` si nuestra versión de Neovim es menor a v0.11.

* [Snacks.bufdelete](https://github.com/folke/snacks.nvim/blob/main/docs/bufdelete.md)

Nos permiten cerrar archivos sin modificar las ventanas abiertas. Por ejemplo, si tienen 2 ventanas abiertas e intentan cerrar un archivo usando el comando `bdelete` Neovim cerrará el archivo y la ventana. Con la función `Snacks.bufdelete()` podemos cerrar el archivo y dejar la ventana abierta.

```lua
-- docs: https://github.com/folke/snacks.nvim/blob/main/docs/bufdelete.md
vim.keymap.set('n', '<leader>bc', function()
  Snacks.bufdelete()
end, {})
```

* [Snacks.terminal](https://github.com/folke/snacks.nvim/blob/main/docs/terminal.md)

La buena noticia es que Neovim tiene una terminal integrada, en teoría no necesitamos ningún plugin para usarla. La mala noticia es que no es "componente" que uno puede ocultar fácilmente, es más como un buffer especial.

`Snacks.terminal` provee funciones que nos permite manejar ventanas con buffers de terminal. La funcionalidad más sencilla que podemos usar es abrir y cerrar una ventana con una terminal.

```lua
-- docs: https://github.com/folke/snacks.nvim/blob/main/docs/terminal.md
vim.keymap.set({'n', 't'}, '<C-g>', function()
  Snacks.terminal.toggle()
end, {})
```

* [Snacks.explorer](https://github.com/folke/snacks.nvim/blob/main/docs/explorer.md)

`Snacks.explorer` es un explorador de archivos. Puede mostrar los archivos en una estructura de árbol como en otros editores. Lo curioso de este explorador es que técnicamente es "filtro" de `Snacks.picker`, en la documentación podrán ver que algunas opciones van en el campo `picker` de la función `.setup()`.

Para empezar a usarlo agregamos esta configuración en la función `.setup()`. Y para hacerlo más conveniente podemos crear un atajo de teclado que muestre el explorador.

```lua
Snacks.setup({
  explorer = {
    enabled = true,
    replace_netrw = true,
  },
})

vim.keymap.set('n', '<leader>e', function()
  Snacks.explorer()
end, {})
```

Ahora si colocamos todo el código relacionado con Snacks.nvim tendremos algo así:

```lua
local Snacks = require('snacks')

Snacks.setup({
  indent = {
    enabled = true,
    char = '▏',
  },
  explorer = {
    enabled = true,
    replace_netrw = true,
  },
  input = {
    enabled = true,
    icon = '❯',
  },
  picker = {
    enabled = true,
    ui_select = true,
    prompt = '❯ ',
  },
  bigfile = {
    -- Only use `bigfile` module on older Neovim versions
    enabled = vim.fn.has('nvim-0.11') == 0,
    notify = false,
    size = 1024 * 1024, -- 1MB
    setup = function(ctx)
      vim.cmd('syntax clear')
      vim.opt_local.syntax = 'OFF'
      local buffer = vim.b[ctx.buf]
      if buffer.ts_highlight then
        vim.treesitter.stop(ctx.buf)
      end
    end
  },
})

-- Disable indent guide animation
vim.g.snacks_animate = false

-- docs: https://github.com/folke/snacks.nvim/blob/main/docs/explorer.md
vim.keymap.set('n', '<leader>e', function()
  Snacks.explorer()
end, {})

-- docs: https://github.com/folke/snacks.nvim/blob/main/docs/terminal.md
vim.keymap.set({'n', 't'}, '<C-g>', function()
  Snacks.terminal.toggle()
end, {})

-- Close while preserving window layout
-- docs: https://github.com/folke/snacks.nvim/blob/main/docs/bufdelete.md
vim.keymap.set('n', '<leader>bc', function()
  Snacks.bufdelete()
end, {})

-- Fuzzy finders
-- docs: https://github.com/folke/snacks.nvim/blob/main/docs/picker.md
vim.keymap.set('n', '<leader><space>', function() Snacks.picker('buffers') end, {})
vim.keymap.set('n', '<leader>ff', function() Snacks.picker('files') end, {})
vim.keymap.set('n', '<leader>fh', function() Snacks.picker('recent') end, {})
vim.keymap.set('n', '<leader>fg', function() Snacks.picker('grep') end, {})
vim.keymap.set('n', '<leader>fd', function() Snacks.picker('diagnostics') end, {})
vim.keymap.set('n', '<leader>fs', function() Snacks.picker('lines') end, {})
vim.keymap.set('n', '<leader>u', function() Snacks.picker('undo') end, {})
vim.keymap.set('n', '<leader>/', function() Snacks.picker('pickers') end, {})
vim.keymap.set('n', '<leader>?', function() Snacks.picker('keymaps') end, {})
```

## ¿Qué sigue?

El siguiente paso sería lograr que Neovim entienda el código de nuestro proyecto: que autocomplete variables, nos permita saltar a la definición de una función, que pueda renombrar una variable, cosas así. Para esto recomiendo el plugin [nvim-lspconfig](https://github.com/neovim/nvim-lspconfig). Como hay muchos detalles que necesitan explicación les recomiendo leer este post:

* [Configurando el cliente LSP de Neovim](https://dev.to/vonheikemen/configurando-el-cliente-lsp-nativo-de-neovim-en-2022-la-manera-facil-3c17)

