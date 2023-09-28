+++
title = "Neovim: Plugins para empezar"
description = "Explorando plugins para mejorar nuestra experiencia en Neovim"
date = 2022-09-03
updated = 2023-02-15
lang = "es"
[taxonomies]
tags = ["neovim", "vim", "shell"]
[extra]
shared = [
  ["dev.to", "https://dev.to/vonheikemen/neovim-plugins-para-empezar-c39"],
  ["Hashnode", "https://vonheikemen.hashnode.dev/neovim-plugins-to-get-started-es"],
]
+++

¿Quieren personalizar Neovim a su gusto pero no saben por dónde empezar? Puedo ayuderles con eso. Voy compartir con ustedes una lista de plugins que muchas personas en la comunidad de Neovim usan con frecuencia.

Todo el código de configuración que mostraré en esta guía estará este repositorio: [nvim-starter - branch: 02-opinionated](https://github.com/VonHeikemen/nvim-starter/tree/02-opinionated).

## Requisitos

Si ustedes son completamente nuevos en Neovim les recomiendo que aprendan [la sintaxis de lua](https://learnxinyminutes.com/docs/lua/). No es necesario saber todo o dominar el lenguaje, como mínimo tengan a la mano una referencia de la sintaxis para saber qué es válido. Casi todos los plugins que presentaré se configuran en ese lenguaje.

Si todavía no han creado un archivo de configuración para Neovim, háganlo ahora. Aquí les dejo una guía con todo lo que necesitan saber: [Cómo crear tu primera configuración de Neovim usando lua](@/tools/build-your-first-lua-config-for-neovim.es.md).

Les recomiendo descargar la versión estable más reciente de Neovim. Pueden descargarla desde la [sección releases](https://github.com/neovim/neovim/releases) en github. De aquí en adelante voy a asumir que están usando la versión 0.8 o mayor.

## ¿Cómo instalamos un plugin?

Lo primero que deben saber es cómo instalar un plugin de forma manual. Lo único que necesitamos para instalar un plugin es descargarlo en una ubicación específica.

La manera "tradicional" de conocer los directorios disponibles para nuestros plugins es usando este comando.

```vim
:set packpath?
```

Neovim nos muestra una lista separada por comas. A mi parecer este formato resulta difícil de leer. Podemos hacer algo mejor aprovechando las capacidades de lua.

```lua
:lua vim.tbl_map(print, vim.opt.packpath:get())
```

Con esto Neovim les mostrará la misma lista pero cada directorio se muestra en una línea.

En uno de esos directorios debemos crear una carpeta llamada `pack`, y dentro de pack debemos crear un "paquete". Un paquete es una carpeta que contiene varios plugins. Debe tener una estructura como esta.

```
carpeta-paquete
├── opt
│   ├── [plugin 1]
│   └── [plugin 2]
└── start
    ├── [plugin 3]
    └── [plugin 4]
```

En este ejemplo creamos una carpeta que contiene dos carpetas: `opt` y `start`. Todos los plugins en `opt` serán opcionales, Neovim no cargará ningún plugin a menos que nosotros lo invoquemos con el comando `packadd`. Los plugins en `start` se cargarán de manera automática durante el proceso de inicialización de Neovim.

Vamos a suponer que tenemos este directorio en nuestro `packpath`.

```
/home/dev/.local/share/nvim/site
```

Antes de instalar un plugin debemos crear la carpeta `pack`. Y dentro de pack creamos un paquete. Para este ejemplo nuestro paquete se llamará `github`. Entonces, la ruta donde instalaremos los plugins será esta.

```
/home/dev/.local/share/nvim/site/pack/github
```

Ya estamos listos. Ahora digamos que queremos instalar [lualine](https://github.com/nvim-lualine/lualine.nvim), sólo tenemos que descargarlo en esta ruta.

```
/home/dev/.local/share/nvim/site/pack/github/start/lualine.nvim
```

Y eso es todo. Bueno... tenemos que configurar el plugin pero eso ya es otra historia.

Para conocer más detalles sobre los paquetes en Neovim revisen la documentación.

```vim
:help packages
```

### Manejador de plugins

Pero claro que ustedes no tienen que descargar plugins manualmente si no quieren. Pueden automatizar todo el proceso con un manejador de plugins. 

En la actualidad estos son los más manejadores de plugins más populares en el ecosistema de Neovim.

* [lazy.nvim](https://github.com/folke/lazy.nvim)
* [packer.nvim](https://github.com/wbthomason/packer.nvim) 
* [paq.nvim](https://github.com/savq/paq-nvim) 

Si prefieren el minimalismo les recomiendo `paq`. Si quieren algo más "completo" usen `packer.nvim` o `lazy.nvim`.

Recuerden leer bien las instrucciones del manejador que decidan usar.

## Plugins

### Tokyonight

Github: [folke/tokyonight.nvim](https://github.com/folke/tokyonight.nvim)

Porque lo primero que deben hacer cuando instalan Neovim es cambiar el tema que tiene por defecto. Para aplicar un tema debemos usar el comando `colorscheme` seguido del nombre del tema.

Para ejecutar comandos desde lua usamos `vim.cmd`. Entonces si queremos aplicar un tema debemos hacer esto.

```lua
vim.cmd('colorscheme nombre-tema')
```

Para usar tokyonight este es el comando.

```vim
colorscheme tokyonight
```

### onedark.vim

Github: [joshdick/onedark.vim](https://github.com/joshdick/onedark.vim)

Adaptación del tema que trae por defecto el editor Atom.

```vim
colorscheme onedark
```

### darkplus.nvim

Github: [lunarvim/darkplus.nvim](https://github.com/lunarvim/darkplus.nvim)

Adaptación del tema que trae por defecto el editor VSCode.

```vim
colorscheme darkplus
```

### monokai.nvim

Github: [tanvirtin/monokai.nvim](https://github.com/tanvirtin/monokai.nvim)

Adaptación del tema que trae por defecto el editor Sublime Text.

```vim
colorscheme monokai
```

### nvim-web-devicons

Github: [kyazdani42/nvim-web-devicons](https://github.com/kyazdani42/nvim-web-devicons)

Dentro de este plugin se encuentra una colección de funciones que ayudan a mostrar íconos. Por lo general no interactuamos directamente con este plugin. Es una dependencia que otros plugins usan para mostrar íconos.

Por sí sólo no es suficiente para mostrar los íconos en una terminal, deben instalar una fuente que ya tiene soporte para íconos. Pueden encontrar una buena colección de fuentes en [nerdfonts.com](https://www.nerdfonts.com).

### Lualine

Github: [nvim-lualine/lualine.nvim](https://github.com/nvim-lualine/lualine.nvim)

Provee una "línea de estado" atractiva. En la línea de estado es donde vemos el tipo de archivo, ubicación del cursor, ese tipo de cosas. Lualine hace que la línea de estado se vea mejor y también le agrega más información.

Para empezar a usarlo sólo tenemos que llamar la función `.setup()` del módulo `lualine`.

```lua
require('lualine').setup({})
```

Dentro de la documentación hay una sección donde se muestra la [configuración que trae por defecto](https://github.com/nvim-lualine/lualine.nvim#default-configuration). 

```
:help lualine-Default-configuration
```

Ahora bien, si queremos personalizarlo debemos agregar las propiedades que nos interesan en el argumento de la función `.setup()`.

```lua
vim.opt.showmode = false

require('lualine').setup({
  options = {
    theme = 'onedark',
    icons_enabled = true,
    component_separators = '|',
    section_separators = '',
    disabled_filetypes = {
      statusline = {'NvimTree'}
    }
  },
})
```

En este ejemplo lo primero que hacemos es desactivar la opción de Neovim `showmode` porque lualine ya nos muestra en qué modo estamos. 

Y esto es lo que tenemos en `.setup()`.

* `options.theme`: Determina qué tema usará Lualine. Pueden cambiar el valor `onedark` por cualquiera de los [temas disponibles](https://github.com/nvim-lualine/lualine.nvim/blob/master/THEMES.md).

* `options.icons_enabled`: Determina si lualine puede usar íconos. Se usa el valor `true` para activar los íconos, `false` hace lo opuesto.

* `options.component_separators`: Es el caracter que será usado para separar "componentes" de lualine.

* `options.section_separators`: El caracter que será usado para separar secciones.

* `options.disabled_filetypes.statusline`: Es una lista donde debemos colocar los tipos de archivos donde no queremos que apareca lualine. Agregamos `NvimTree` para que la interfaz del explorador de archivos se vea mejor.

### Bufferline

Github: [akinsho/bufferline.nvim](https://github.com/akinsho/bufferline.nvim)

En Neovim las pestañas son como espacios de trabajo, pueden mostrar varios archivos en una pestaña e incluso cambiar el directorio de trabajo por pestaña. Algunos prefieren mostrar un archivo por pestaña, como en otros editores. Eso es exactamente lo que hace `bufferline`, modifica las pestañas para mostrar todos los archivos abiertos.

Para empezar a usarlo sólo debemos llamar la función `.setup()` del módulo `bufferline`.

```lua
require('bufferline').setup({})
```

Pueden encontrar la documentación de las opciones disponibles con este comando.

```vim
:help bufferline-configuration
```

Esta la configuración que recomiendo.

```lua
require('bufferline').setup({
  options = {
    mode = 'buffers',
    offsets = {
      {filetype = 'NvimTree'}
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

* `options.offsets`: Es una lista de tipos de archivos. Cuando aparece una ventana de este tipo `bufferline` evita colocar una pestaña sobre ella. Aquí le indicamos el tipo de archivo `NvimTree` porque ese será el explorador de archivos que usaremos. Esto hará que se logre el efecto de "barra lateral" cuando aparezca la ventana de nvim-tree.

* `highlights`: Nos permite modificar los colores en ciertas áreas. Cada sección (como `buffer_selected`) se refiere a un área. En este ejemplo coloco `italic = false` para desactivar las letras cursivas. La opción `fg` cambia el color de las letras. En este ejemplo le indico que debe tomar el mismo color que utiliza mi tema para las funciones. Pueden ver los detalles de la opción highlights usando el comando `:help bufferline-highlights`.

### indent-blankline.nvim

Github: [lukas-reineke/indent-blankline.nvim](https://github.com/lukas-reineke/indent-blankline.nvim)

Añade guías de indentación a todas las líneas del archivo.

Para configurarlo debemos usar la función `.setup()` del módulo `ibl`.

```lua
require('ibl').setup({
  indent = {
    char = '▏',
  },
})
```

Estas son las opciones más interesantes.

```lua
require('ibl').setup({
  enabled = false,
  scope = {
    enabled = false,
  },
  indent = {
    char = '▏',
  },
})
```

Aquí el valor `true` activa la funcionalidad de una opción y `false` hace lo opuesto.

* `enabled`: Habilita las guías de indentación.

* `scope.enabled`: Resalta el nivel de indentación del ámbito donde se encuentra el cursor.

* `indent.char`: es el caracter que se muestra en pantalla.

### Treesitter

Github: [nvim-treesitter/nvim-treesitter](https://github.com/nvim-treesitter/nvim-treesitter)

Treesitter es un componente que se encuentra dentro de Neovim, este le permite a Neovim leer código de la misma manera que un compilador. ¿Cómo es eso? Escanea el código, va recolectando información de cada símbolo y al final genera un árbol de sintaxis. En otras palabras, convierte tu código en una estructura de datos que Neovim puede consultar.

Por sí sólo treesitter no nos trae ningún beneficio, son los plugins que utilizan treesitter los que agregan funcionalidades a Neovim. Es aquí donde viene `nvim-treesitter`, este trae varios "módulos" que podemos utilizar para aprovechar las bondades de treesitter.

Pueden leer los detalles de los módulos usando este comando.

```vim
:help nvim-treesitter-modules
```

Para habilitar un módulo de `nvim-treesitter` debemos invocar la función `.setup()` del plugin. Y en los argumentos debemos especificar la configuración de cada módulo. Ejemplo:

```lua
require('nvim-treesitter.configs').setup({
  highlight = {
    enable = true,
  },
})
```

Aquí estamos configurando el módulo `highlight`, este se encarga de darnos resaltado de sintaxis. Por defecto todos los módulos en `nvim-treesitter` están deshabilitados. Como mínimo debemos agregar `enable = true` para activarlos.

Para que treesitter pueda funcionar debemos instalar un "parser" por cada lenguaje que queremos utilizar. El parser es el componente que se encarga de leer el código fuente de un archivo. Para instalar un parser debemos usar el comando `TSInstall` seguido del nombre del lenguaje.

Entonces, si queremos instalar el parser para javascript debemos usar este comando.

```vim
:TSInstall javascript
```

También tenemos la posibilidad de decirle a `nvim-treesitter` qué parsers queremos instalar automáticamente. Para esto debemos agregar la opción `ensure_installed` en la función `.setup()`. Ahí colocamos la lista de lenguajes.

```lua
require('nvim-treesitter.configs').setup({
  highlight = {
    enable = true,
  },
  ensure_installed = {
    'javascript',
    'typescript',
    'tsx',
    'css',
    'json',
    'lua',
  },
})
```

### nvim-treesitter-textobjects

Github: [nvim-treesitter/nvim-treesitter-textobjects](https://github.com/nvim-treesitter/nvim-treesitter-textobjects)

Los "text objects" en Neovim corresponden a patrones de texto que pueden encontrarse en un archivo. Por defecto Neovim puede ejecutar acciones sobre palabras, párrafos, etiquetas de xml y algunos otros. `nvim-treesitter-textobjects` añade más patrones basados en consultas de treesitter. Podremos actuar sobre funciones, clases, condicionales y ciclos repetitivos.

Por sí no lo saben, los text objects son usados por Neovim en el modo "operador pendiente". Cuando usamos un operador como `d` (el atajo de teclado para borrar) Neovim entra en modo operador pendiente. Cuando presionamos una combinación como `daw`, esa `d` es el operador y `aw` es el text object. Para saber más sobre el tema revisen la documentación.

```vim
:help motion.txt
```

Volvamos a la programación habitual.

nvim-treesitter-textobjects tiene sus propios submódulos. Pueden navegar a la documentación con este comando.

```vim
:help nvim-treesitter-textobjects-modules
```

Para configurar nvim-treesitter-textobjects deben agregar la propiedad `textobjects` en la función `.setup()` de `nvim-treesitter`.

```lua
require('nvim-treesitter.configs').setup({
  highlight = {
    enable = true,
  },
  textobjects = {
    select = {
      enable = true,
      lookahead = true,
      keymaps = {
        ['af'] = '@function.outer',
        ['if'] = '@function.inner',
        ['ac'] = '@class.outer',
        ['ic'] = '@class.inner',
      }
    },
  },
  ensure_installed = {
    --- parsers....
  },
})
```

Aquí habilitamos el submódulo `select` de `textobjects`, este es el que le permite a Neovim seleccionar patrones básados en consultas de treesitter. Dentro del módulo tenemos estas opciones.

* `enable`: Determina si el módulo debe ser activado.

* `lookahead`: Determina si el cursor debe navegar hacia la ocurrencia más cercana.

* `keymaps`: Son los atajos de teclado. Del lado izquierdo declaramos los atajos que representa el text object. Del lado derecho le asignamos un "grupo". Estos grupos son creados por `nvim-treesitter-textobjects`. Pueden encontrar la lista de grupos aquí: [Builtin textobjects](https://github.com/nvim-treesitter/nvim-treesitter-textobjects#built-in-textobjects).

### targets.vim

Github: [wellle/targets.vim](https://github.com/wellle/targets.vim)

Crea nuevos text objects. Cosas como pares, argumentos, separadores, etc. Les recomiendo revisar documentación, ahí pueden encontrar una explicación detallada (con gráficas) de cada text object: [targets.vim - overview](https://github.com/wellle/targets.vim#overview).

Pueden modificar algunos aspectos de este plugin pero no es necesario. Pueden usarlo inmediatamente sin escribir ninguna configuración. Si desean conocer las opciones disponibles ejecuten este comando.

```vim
:help targets-settings
```

### Comment.nvim

Github: [numToStr/Comment.nvim](https://github.com/numToStr/Comment.nvim)

Agrega un operador para comentar código. Por defecto utiliza el atajo `gc`. Esto significa que tendremos acceso a todas las combinaciones que nos permite Neovim en el modo operador pendiente. Podemos comentar una palabra con `gciw`, podemos comentar un párrafo con `gcap`. Podemos comentar una cantidad arbitraria de líneas con `gc` + un número + `j`. La capacidad de Comment.nvim va a depender de su conocimiento sobre los text objects y movimientos.

El atajo `gc` también funciona en modo visual. Usar `gc` en modo visual comentará todas líneas seleccionadas.

Con el atajo `gcc` podremos comentar una línea en modo normal.

Este plugin no requiere de ninguna configuración manual. Sólo tenemos que llamar la función `.setup()` del módulo `Comment`.

```lua
require('Comment').setup({})
```

Si desean conocer las opciones disponibles revisen la documentación: [Comment.nvim - setup](https://github.com/numToStr/Comment.nvim#%EF%B8%8F-setup).

### vim-surround

Github: [tpope/vim-surround](https://github.com/tpope/vim-surround)

Nos permite ejecutar acciones sobre pedazos de texto que están rodeados por algún patrón. Nos provee atajos para añadir, borrar y reemplazar el patrón que rodea el texto.

Déjenme explicarles con algunos ejemplos.

Si tenemos el texto `'Hola, mundo'` podemos borrar las comillas presionando `ds'`. `ds` es el atajo para borrar y `'` es el patrón que queremos eliminar.

Si queremos agregar un patrón usamos `ys`. Vamos a suponer que tenemos la palabra `Hola`, podemos envolverla en paréntesis presionando `ysiw)`. `ys` es el atajo para agregar, `iw` es el text object que representa la palabra y `)` es el patrón que queremos agregar. El resultado será `(Hola)`.

Si estamos en modo visual podremos envolver la selección usando `S` + el patrón que queremos usar.

Para reemplazar un patrón usamos `cs`. Si tenemos el texto `'Hola, mundo'`, podemos cambiar las comillas simples por dobles presionando `cs'"`. `cs` es el atajo para reemplazar, `'` es lo que queremos reemplazar y `"` es el nuevo patrón.

### nvim-tree

Github: [kyazdani42/nvim-tree.lua](https://github.com/kyazdani42/nvim-tree.lua)

Es un explorador de archivos. Nos muestra la estructura del directorio actual en forma de árbol. También nos permite crear, borrar y renombrar archivos.

Para empezar a usar el plugin sólo deben llamar la función `.setup()` del módulo `nvim-tree`.

```lua
require('nvim-tree').setup({})
```

Para conocer los atajos de teclado disponibles dentro del explorador ejecuten este comando.

```vim
:help nvim-tree-default-mappings
```

Si quieren personalizarlo primero revisen las opciones disponibles.

```vim
:help nvim-tree-setup
```

Estas son las opciones que me parecen más interesantes.

```lua
require('nvim-tree').setup({
  hijack_cursor = false,
  on_attach = function(bufnr)
    local bufmap = function(lhs, rhs, desc)
      vim.keymap.set('n', lhs, rhs, {buffer = bufnr, desc = desc})
    end

    -- See :help nvim-tree.api
    local api = require('nvim-tree.api')
   
    bufmap('L', api.node.open.edit, 'Expand folder or go to file')
    bufmap('H', api.node.navigate.parent_close, 'Close parent folder')
    bufmap('gh', api.tree.toggle_hidden_filter, 'Toggle hidden files')
  end
})

vim.keymap.set('n', '<leader>e', '<cmd>NvimTreeToggle<cr>')
```

Aquí uso el atajo `<leader>e` para abrir y cerrar el explorador. Y en la función `.setup()` tenemos dos opciones.

* `hijack_cursor`: Determina si nvim-tree debe manejar la posición del cursor en la ventana. Si lo cambiamos a `true` nvim-tree coloca el cursor al inicio del nombre de cada "nodo".

* `on_attach`: Es una función que se ejecutará cada vez que nvim-tree abre una ventana. Se recomienda crear los atajos en esta función. Aquí utilizamos el módulo `nvim-tree.api` para tener acceso a la funciones de nvim-tree. Usamos la función de Neovim `vim.keymap.set` para crear los atajos. En el primer atajo asignamos `L` a la función `api.node.open.edit`, que nos permite abrir un nodo. Usamos `H` para cerrar el nodo donde se encuentra el cursor. Usamos `gh` para mostrar/ocultar los archivos ocultos.

### Telescope

Github: [nvim-telescope/telescope.nvim](https://github.com/nvim-telescope/telescope.nvim) 

La funcionalidad principal de telescope es poder filtrar de manera interactiva una lista de items. Por defecto nos permite filtrar listas como el historial de archivos, archivos abiertos, commits de git, errores en el archivo y 47 cosas más. Sí, en la actualidad telescope tiene 51 subcomandos. ¿Cómo lo sé? Porque se puede buscar un comando de telescope usando telescope.

```vim
:Telescope builtin
```

Telescope no necesita ser configurado de manera manual. Podemos invocar cualquiera de sus comandos sin ningún problema. Pero si quieren conocer las opciones disponibles revisen la documentación.

```vim
:help telescope.setup()
```

Entonces lo único que nos queda por hacer es crear atajos para los comandos que queremos usar con más frecuencia.

```lua
vim.keymap.set('n', '<leader><space>', '<cmd>Telescope buffers<cr>')
vim.keymap.set('n', '<leader>?', '<cmd>Telescope oldfiles<cr>')
vim.keymap.set('n', '<leader>ff', '<cmd>Telescope find_files<cr>')
vim.keymap.set('n', '<leader>fg', '<cmd>Telescope live_grep<cr>')
vim.keymap.set('n', '<leader>fd', '<cmd>Telescope diagnostics<cr>')
vim.keymap.set('n', '<leader>fs', '<cmd>Telescope current_buffer_fuzzy_find<cr>')
```

* `<leader><space>`: Muestra la lista de archivos abiertos.
* `<leader>?`: Muestra el historial de archivos.
* `<leader>ff`: Muestra los archivos del directorio de trabajo actual.
* `<leader>fg`: Ejecuta una búsqueda interactiva en cada línea código de cada archivo en el directorio actual.
* `<leader>fd`: Muestra la lista de "diagnósticos" del archivo actual. Un diagnóstico puede ser un error de sintaxis, una advertencia o una sugerencia.
* `<leader>fs`: Ejecuta una búsqueda interactiva en el archivo actual.

Vale la pena mencionar que telescope es capaz de usar herramientas externas como [fd](https://github.com/sharkdp/fd) y [ripgrep](https://github.com/BurntSushi/ripgrep) para incrementar la velocidad de sus búsquedas.

### telescope-fzf-native

Github: [nvim-telescope/telescope-fzf-native.nvim](https://github.com/nvim-telescope/telescope-fzf-native.nvim)

Es una extensión para telescope. Puede hacer que telescope use el mismo algoritmo de búsqueda que [fzf](https://github.com/nvim-telescope/telescope-fzf-native.nvim), con esto podremos usar la misma sintaxis para buscar en `fzf` y telescope. Además de eso también aumenta la velocidad de nuestras búsquedas. 

El código de la extensión está escrito en `C` así que para usarlo deberán asegurarse de tener instalado un compilador de `C` y también `make`.

Para cargar esta extensión debemos usar la función `.load_extension()` de telescope.

```lua
require('telescope').load_extension('fzf')
```

### Toggleterm

Github: [akinsho/toggleterm.nvim](https://github.com/akinsho/toggleterm.nvim)

La buena noticia es que Neovim tiene una terminal integrada, en teoría no necesitamos ningún plugin para usarla. La mala noticia es que no es "componente" que uno puede ocultar fácilmente, es más como un tipo especial de ventana. Toggleterm se encarga de manejar estas ventanas de tal manera que pueden abrir y ocultarse con un atajo de teclado.

Toggleterm tiene un montón de funcionalidades les recomiendo que lean la documentación para conocer más detalles: [Toggleterm.nvim - Roadmap](https://github.com/akinsho/toggleterm.nvim#roadmap).

Para empezar a usar el plugin tenemos que usar la función `.setup()` del módulo `toggleterm`. Les mostraré las opciones que me parecen más interesantes.

```lua
require('toggleterm').setup({
  open_mapping = '<C-g>',
  direction = 'horizontal',
  shade_terminals = true
})
```

* `open_mapping`: Crea el atajo de teclado para abrir/cerrar la ventana con la terminal.

* `direction`: Será la orientación que tendrá por defecto la ventana. Puede ser `horizontal`, `vertical` o `float`.

* `shade_terminals`: Determina si el color de fondo de la ventana con la terminal debe ser más oscuro que las ventanas normales.

### vim-fugitive

Github: [tpope/vim-fugitive](https://github.com/tpope/vim-fugitive)

Provee una interfaz gráfica para manejar un repositorio de git desde Neovim (o Vim). También nos permite ejecutar comandos de git directamente desde Neovim.

Para conocer todos los detalles de este plugin revisen la documentación.

```vim
:help fugitive.txt
```

### Gitsigns

Github: [lewis6991/gitsigns.nvim](https://github.com/lewis6991/gitsigns.nvim)

Su función principal es colocar indicadores al costado de una línea de código, señalando los cambios que se han efectuado en el archivo. Puede decirnos cuales líneas son nuevas, cuales han cambiado o donde borramos una línea. También tiene otras funcionalidades pero son demasiadas para nombrar aquí. Pueden revisar la lista de funciones con este comando.

```vim
:help gitsigns-functions
```

Pueden encontrar una descripción de cada opción disponible para este plugin con este comando.

```vim
:help gitsigns-config
```

Ahora bien, sí sólo quieren personalizar los indicadores estas son las opciones que deben modificar.

```lua
require('gitsigns').setup({
  signs = {
    add = {text = '▎'},
    change = {text = '▎'},
    delete = {text = '➤'},
    topdelete = {text = '➤'},
    changedelete = {text = '▎'},
  }
})
```

* `signs.add.text`: Es el caracter que se muestra al agregar una línea.
* `signs.change.text`: Es el caracter que se muestra al modificar una línea.
* `signs.delete.text`: Es el caracter que se muestra al eliminar una línea.
* `signs.topdelete.text`: Es el caracter que se muestra cuando se elimina una línea al principio del archivo.
* `signs.changedelete.text`: Es el caracter que se muestra cuando la línea actual ha sido modificada y también ocupa el lugar de una línea eliminada.

### Plenary

Github: [nvim-lua/plenary.nvim](https://github.com/nvim-lua/plenary.nvim)

Colección de módulos. No hay una "temática" específica en este plugin, tiene un montón de funciones que otros autores de plugins usan para resolver problemas comunes.

Telescope utiliza este plugin en su código fuente, así que deben tenerlo instalado.

### vim-repeat

Github: [tpope/vim-repeat](https://github.com/tpope/vim-repeat)

Agrega soporte para repeticiones a comandos creados por plugins. Si no lo saben, si presionamos `.` Neovim repite la última acción que hicimos. Por ejemplo, si borramos una palabra usando `diw` podemos repetir esta acción simplemente presionando `.`. `vim-repeat` permite que las acciones de los plugins también pueda repetirse con `.`.

### editorconfig

Github: [editorconfig/editorconfig-vim](https://github.com/editorconfig/editorconfig-vim)

[EditorConfig](https://editorconfig.org/) es como un archivo de configuración pero este se limita a opciones de estilo, cosas como tamaño de la indentación o codificación del archivo. Con `editorconfig-vim` podemos hacer que Neovim pueda leer este formato.

### vim-bbye

Github: [moll/vim-bbye](https://github.com/moll/vim-bbye)

Provee comandos que nos permiten cerrar archivos sin modificar las ventanas abiertas. Por ejemplo, si tienen 2 ventanas abiertas e intentan cerrar un archivo usando el comando `bdelete` Neovim cerrará el archivo y la ventana. Con este plugin podremos usar el comando `Bdelete` para cerrar el archivo y dejar la ventana abierta.

```lua
vim.keymap.set('n', '<leader>bc', '<cmd>Bdelete<CR>')
```

## ¿Qué sigue?

El siguiente paso sería lograr que Neovim entienda el código de nuestro proyecto: que autocomplete variables, nos permita saltar a la definición de una función, que pueda renombrar una variable, cosas así. Para esto recomiendo los plugins [nvim-lspconfig](https://github.com/neovim/nvim-lspconfig) y [nvim-cmp](https://github.com/hrsh7th/nvim-cmp). Configurar esos dos no es tarea fácil, hay muchos detalles que necesitan explicación. He preparado otra guía para este tema en particular:

* [Configurando nvim-lspconfig + nvim-cmp](@/tools/setup-nvim-lspconfig-plus-nvim-cmp.es.md)

## ¿Dónde pueden más encontrar plugins?

Estos son algunos recursos donde podrán encontrar información sobre el ecosistema de plugins para Neovim.

* [awesome-neovim](https://github.com/rockerBOO/awesome-neovim)
* [neovimcraft](https://neovimcraft.com)
* [this week in neovim](https://dotfyle.com/this-week-in-neovim)

## Conclusión

Revisamos varios temas para Neovim. Sabemos que podemos modificar ciertos componentes de la interfaz de Neovim, adaptarlos para que se asemejen a otros editores. Tenemos `bufferline` que puede darnos pestañas por cada archivo abierto. Tenemos `nvim-tree` que nos da un explorador de archivos. Podemos aprovechar funciones de `git` gracias a `fugitive` y `gitsigns`. Podemos buscar todo tipo de cosas con `Telescope`. Y aprendimos sobre una variedad de plugins que pueden extender nuestra capacidad de manipular texto como `nvim-treesitter`, `vim-surround`, etc. Tenemos todo lo necesario para crear una buena experiencia en Neovim.

