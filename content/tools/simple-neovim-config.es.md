+++
title = "Una configuración simple para Neovim"
description = "Aprende lo básico para configurar Neovim usando lua"
date = 2024-11-23
updated = 2025-05-15
lang = "es"
[taxonomies]
tags = ["vim", "neovim", "shell"]
+++

50 líneas de código es suficiente para tener una buena configuración en Neovim. Y puede que no me crean esto pero sólo necesitamos **1 plugin** para tener acceso a ciertas funcionalidades que generalmente encuentran en un IDE.

Pueden usar [esta configuración](#init-lua) como un punto de partida y luego van agregando cosas nuevas cuando encuentren algún problema. O, simplemente veanlo como una oportunidad para aprender los conceptos básicos de la configuración de Neovim.

## Instalando Neovim

Para la configuración que les voy a mostrar necesitamos Neovim en su versión v0.9 o mayor. Pero deben tener en cuenta que muchos plugins escritos en lua sólo garantizan soporte para la versión estable actual. En este momento sería v0.11.1, que fue publicada el `26 de abril del 2025`.

Si su sistema operativo está basado en linux deben prestar atención a la versión de Neovim que está disponible en su manejador de paquetes.

Si saben cómo instalar ejecutables pre-compilados pueden ir a la sección [releases](https://github.com/neovim/neovim/releases) del repositorio de Neovim y descargan la versión estable actual.

Otra alternativa sería usar el manejador de versiones llamado `bob`. Pueden encontrarlo aquí: [MordechaiHadad/bob](https://github.com/MordechaiHadad/bob).

## El punto de entrada

Ahora vamos a crear el archivo de configuración `init.lua`. La ubicación de este archivo depende del sistema operativo que estamos usando. Entonces, para ahorranos problemas vamos a crearlo usando Neovim en modo "headless" desde la terminal.

Ejecuten este comando en su terminal.

```sh
nvim --headless -c 'exe "write ++p" stdpath("config") . "/init.lua"' -c 'quit'
```

Después de crear el archivo podemos revisar su ubicación usando este comando.

```sh
nvim --headless -c 'echo $MYVIMRC' -c 'quit'
```

Ya podemos empezar a escribir algo de código. Pero primero, les recomiendo que dediquen un poco de tiempo a aprender `lua`. Como mínimo tengan a la mano una referencia de la sintaxis. Aquí les dejo unos recursos (en inglés):

* [Lua crash course (video 12 min)](https://www.youtube.com/watch?v=NneB6GX1Els)
* [Learn X in Y minutes: Where X = lua](https://learnxinyminutes.com/docs/lua/)

## Opciones del editor

Para modificar la configuración básica del editor en lua [se recomienda](https://github.com/neovim/neovim/issues/30383#issuecomment-2351519326) usar [vim.o](https://neovim.io/doc/user/lua-guide.html#_vim.o). Esta es la sintaxis que debemos seguir.

```lua
vim.o.nombre_opcion = valor
```

Donde `nombre_opcion` debe ser una opción válida de Neovim. Y `valor` debe ser lo que esa opción espera. Pueden encontrar la lista de opciones en la documentación de Neovim: [Quickref - option list](https://neovim.io/doc/user/quickref.html#option-list).

Estas son algunas opciones que pueden interesarles:

* Habilitar números de líneas.

```lua
vim.o.number = true
```

* Habilita el scroll horizontal.

```lua
vim.o.wrap = false
```

* Ajustar el ancho del caracter `Tab`.

```lua
vim.o.tabstop = 2
vim.o.shiftwidth = 2
```

* Ignorar mayúsculas y minúsculas cuando el patrón de búsqueda esté todo en minúsculas.

```lua
vim.o.smartcase = true
vim.o.ignorecase = true
```

* Deshacer el resaltado de una búsqueda luego de terminar el proceso.

```lua
vim.o.hlsearch = false
```

## Tema del editor

Para cambiar el tema del editor debemos usar el comando `colorscheme`. Y para invocar un comando en lua usamos [vim.cmd](https://neovim.io/doc/user/lua-guide.html#_vim-commands). Entonces en nuestra configuración podemos hacer esto:

```lua
vim.cmd.colorscheme('retrobox')
```

El problema es que `retrobox` fue agregado a Neovim recientemente. Si tienen Neovim v0.9.5 o una versión menor, el comando va a fallar. Entonces, para garantizar la compatiblidad recomiendo usar la función `pcall`. Así evitamos que un posible error interrumpa la ejecución de nuestra configuración.

```lua
local ok_theme = pcall(vim.cmd.colorscheme, 'retrobox')
if not ok_theme then
  vim.cmd.colorscheme('habamax')
end
```

## Portapapeles

Cuando copiamos un pedazo de texto usando el atajo `y` Neovim lo guarda en un registro interno. Para pegar usamos el atajo `p`, este utiliza el mismo registro que `y`. En otras palabras, Neovim por defecto ignora el portapapeles del sistema. Podemos usar el portapapeles pero debemos hacerlo de manera explícita.

Lo que hago en mi configuración personal es crear atajos específicos para interactuar con el portapapeles usando la función [vim.keymap.set()](https://neovim.io/doc/user/lua-guide.html#_mappings).

```lua
vim.keymap.set({'n', 'x'}, 'gy', '"+y', {desc = 'Copiar al portapapel'})
vim.keymap.set({'n', 'x'}, 'gp', '"+p', {desc = 'Pegar del portapapel'})
```

Con esto el atajo `gy` copia texto de Neovim al portapapeles del sistema, y `gp` pega texto del portapapel a Neovim.

Nota: `{'n', 'x'` es la lista de modos de Neovim donde podemos ejecutar el atajo. `n` es modo normal y `x` es modo visual.

## Tecla líder

Por lo general no se recomienda crear atajos que modifiquen el comportamiento que tiene Neovim por defecto, como lo hicimos en sección anterior. La mayor parte del tiempo colocamos el prefijo `<leader>` para nuestros atajos personales. De esta forma evitamos cualquier tipo de conflicto con las funcionalidades de Neovim.

¿Pero qué significa `<leader>`?

Es una especie de variable. Por defecto es la tecla `\`. Por ejemplo:

```lua
vim.keymap.set('n', '<leader>h', '<cmd>echo "hola"<cr>')
```

Este código crea el atajo `\ + h`, y este a su vez muestra el mensaje `hola`.

Para cambiar el valor de `<leader>` usamos la **variable global** `mapleader`.

```lua
vim.g.mapleader = ','
```

Con esto cambiamos la tecla líder de `\` a `,`. Pero en caso de querer usar una secuencia especial como `control + k` debemos utilizar el "keycode" de esa secuencia. Por ejemplo.

```lua
vim.g.mapleader = vim.keycode('<C-k>')
```

La mayoría de las configuraciones que he visto usan la tecla `Espacio` como líder. Hacen algo como esto.

```lua
vim.g.mapleader = ' '
```

Sí, colocan un espacio en blanco. Esto es lo que la gente hacía antes de que Neovim agregara la función `vim.keycode`. Pero si tienen Neovim v0.10 pueden hacer esto:

```lua
vim.g.mapleader = vim.keycode('<Space>')
```

Ahora bien, para que la variable `mapleader` tenga efecto en nuestros atajos deben definirla **antes** de crear sus atajos personales.

## Ejecutando comandos

Ya estamos listos para crear atajos que ejecuten comandos de vim, lo que se conoce como [Ex-commands](https://neovim.io/doc/user/vimindex.html#_6.-ex-commands). 

Podemos hacer esto de varias maneras pero esta es mi favorita:

```
<cmd>...<cr>
```

`<cmd>` es una secuencia especial que le indica a Neovim que queremos ejecutar una expresión. Y `<cr>` representa la tecla `Enter`, lo que marca el fin de la expresión que queremos ejecutar.

Por ejemplo:

* Guardar el archivo actual.

```lua
vim.keymap.set('n', '<leader>w', '<cmd>write<cr>', {desc = 'Guardar'})
```

* Cerrar todos los archivos y salir del editor.

```lua
vim.keymap.set('n', '<leader>q', '<cmd>quitall<cr>', {desc = 'Salir de vim'})
```

## Instalando plugins

Neovim tiene una funcionalidad llamada [vim packages](@/tools/installing-neovim-plugins-without-a-plugin-manager.es.md). Esto quiere decir que para instalar un plugin lo único que debemos hacer es descargarlo en la ubicación correcta.

Para no complicarnos mucho sólo vamos a descargar 2 plugins. Y ya que vamos por el camino de la simplicidad, haremos el proceso de instalación de forma manual.

Podemos usar el comando `git clone` para descargar plugins de Neovim que se encuentran alojados en github. Tengan en cuenta que el proceso de actualización también es manual. **Ustedes** serán los encargados de actualizar cada plugin. Pero en este punto yo digo que es mejor tener una configuación básica para probar cómo es editar código en Neovim. Luego si deciden que de verdad les gusta pueden eligir un plugin manager.

Bien. ¿Cómo se descarga un plugin?

Vamos a usar Neovim en modo headless otra vez. En la terminal vamos a ejecutar el comando para crear la carpeta donde debemos descargar los plugins.

```sh
nvim --headless -c 'call mkdir(stdpath("config") . "/pack/vendor/start/", "p")' -c 'quit'
```

Para revisar la ubicación de la carpeta que acaban de crear pueden usar este comando.

```sh
nvim --headless -c 'echo stdpath("config") . "/pack/vendor/start/"' -c 'quit'
```

Ahora deben navegar a esa carpeta, ahí es donde debemos ejecutar el comando `git clone`.

* Instalar [nvim-lspconfig](https://github.com/neovim/nvim-lspconfig)

Este es el plugin que nos ayudará a tener esas funcionalidades como las de un IDE.

```
git clone https://github.com/neovim/nvim-lspconfig
```

**Importante**: `nvim-lspconfig` ya no tiene soporte para versiones anteriores de Neovim. Si están usando Neovim v0.9 cambien al [tag v1.8.0](https://github.com/neovim/nvim-lspconfig/releases/tag/v1.8.0).

```
git switch --detach v1.8.0
```

* Instalar [mini.nvim](https://github.com/echasnovski/mini.nvim)

`mini.nvim` es una colección de módulos de lua. Cada módulo es independiente y funciona como un plugin. La idea de esto es resolver problemas comunes y mejorar donde sea posible las capacidades nativas de Neovim.

```
git clone --filter=blob:none https://github.com/echasnovski/mini.nvim
```

Luego de instalar los plugins debemos generar los tags para la documentación.

```sh
nvim --headless -c 'helptags ALL' -c 'quit'
```

Después de ejecutar todos esos comandos la carpeta de configuración de Neovim debe tener esta estructura.

```
nvim
├── init.lua
└── pack
    └── vendor
        └── start
            ├── mini.nvim
            └── nvim-lspconfig
```

## Explorador de archivos

Neovim ya tiene un explorador archivos, se [llama netrw](@/tools/using-netrw-vim-builtin-file-explorer.es.md). Yo no lo uso porque tiene algunas peculiaridades que no me agradan. Aquí voy a recomendarles [mini.files](https://github.com/echasnovski/mini.nvim/blob/main/readmes/mini-files.md), el explorador de archivos de `mini.nvim`.

Para usar `mini.files` debemos invocar la función `setup()` del módulo correspondiente.

```lua
require('mini.files').setup({})
```

Esta es una convención que muchos plugins creados con lua siguen. Algunas veces **deben** invocar la función `.setup()` para habilitar el plugin. Debo enfatizar que es una convención, no es una regla. Hay algunos plugins que activan sus funcionalidades automáticamente.

Ahora bien ¿Cómo se usa `mini.files`?

Debemos crear un atajo de teclado para abrir el explorador.

```lua
vim.keymap.set('n', '<leader>e', '<cmd>lua MiniFiles.open()<cr>', {desc = 'Explorador de archivos'})
```

Luego de abrir el explorador pueden usar el atajo `g?`, este abre una ventana de ayuda que les mostrará todos los atajos disponibles dentro del explorador.

¿Y cómo se configura `mini.files`?

Tienen que agregar su configuración dentro de las llaves `{}`. Por ejemplo.

```lua
require('mini.files').setup({
  mappings = {
    show_help = 'gh',
  },
})
```

Esto cambia el atajo que muestra la ventana de ayuda. En lugar de ser `g?` ahora es `gh`.

La configuración como tal está dentro de una "tabla." Si hay una parte de la sintaxis de lua que deben aprender es esa. Saber cómo se crea y modifica una tabla les ahorrará muchos dolores de cabeza.

¿Cómo sabe uno qué opciones tiene disponible un plugin?

Podemos navegar a la documentación "offline" de un plugin usando el comando `:help`.

```lua
:help mini.files
```

Ese comando los llevará a [la página de ayuda de mini.files](https://github.com/echasnovski/mini.nvim/blob/main/doc/mini-files.txt).

Vale la pena mencionar que pueden usar la tecla `Tab` para autocompletar el argumento del comando `:help`. Por ejemplo, si escriben `:help mini` y luego presionan `Tab` aparecerá una lista de todos los tags que contienen la palabra `mini`. 

Si la fuente (tipo de letra) que tienen configurada en su terminal no contiene "íconos" pueden usar el módulo [mini.icons](https://github.com/echasnovski/mini.nvim/blob/main/readmes/mini-icons.md). En la propiedad `style` colocan el valor `ascii`.

```lua
require('mini.icons').setup({style = 'ascii'})
```

## Navegando entre archivos

Aquí hablaremos de los métodos que podemos usar para navegar entre archivos abiertos.

Cuando no tengo plugins instalados uso el comando `:files` para listar los archivos abiertos. Luego uso el comando `:buffer` para ir al archivo que quiero. Y para que este proceso sea conveniente creo un atajo de teclado.

```lua
vim.keymap.set('n', '<leader><space>', '<cmd>files<cr>:buffer ', {desc = 'Listar archivos'})
```

El comando `:buffer` acepta como argumento el nombre del archivo o el "id" del buffer. También pueden autocompletar el nombre usando la tecla `Tab`, así que no tienen escribir el nombre completo del archivo.

Pero ya que tenemos `mini.nvim` instalado podemos usar el módulo [mini.pick](https://github.com/echasnovski/mini.nvim/blob/main/readmes/mini-pick.md), este provee una especie de buscador interactivo que hace este proceso aún más cómodo.

También tienen la opción de usar el módulo [mini.extra](https://github.com/echasnovski/mini.nvim/blob/main/readmes/mini-extra.md) para expandir las capacidades de `mini.pick`.

En fin, vamos a habilitar `mini.pick`.

```lua
require('mini.pick').setup({})
```

Luego de esto podemos crear los atajos para navegar entre archivos.

```lua
vim.keymap.set('n', '<leader><space>', '<cmd>Pick buffers<cr>', {desc = 'Listar archivos abiertos'})
vim.keymap.set('n', '<leader>ff', '<cmd>Pick files<cr>', {desc = 'Listar todos los archivos'})
vim.keymap.set('n', '<leader>fh', '<cmd>Pick help<cr>', {desc = 'Listar help tags'})
```

## Autocompletado de código

Para esto podemos usar [mini.completion](https://github.com/echasnovski/mini.nvim/blob/main/readmes/mini-completion.md) y [mini.snippets](https://github.com/echasnovski/mini.nvim/blob/main/readmes/mini-snippets.md). 

```lua
require('mini.snippets').setup({})
require('mini.completion').setup({})
```

`mini.completion` utiliza el mecanismo nativo de Neovim y le hace algunas mejoras. Por defecto debemos presionar atajos específicos dependiendo del tipo de completado que queremos activar. `mini.completion` nos libera de esa carga, activando el completado de código mientras escribimos. En otras palabras, `mini.completion` hace posible tener autocompletado real dentro de Neovim.

Por defecto `mini.completion` activa las sugerencias basadas en las palabras que se encuentren en el archivo actual. Pero de ser posible utiliza el cliente LSP de Neovim para mostrar sugerencias más relevantes.

Para controlar el menú de sugerencias utilizamos los atajos nativos de Neovim:

* `<Down>`: Selecciona el siguiente item en la lista.

* `<Up>`: Selecciona el item anterior en la lista.

* `<Ctrl-n>`: Selecciona e inserta el contenido del siguiente item en la lista.

* `<Ctrl-p>`: Selecciona e inserta el contenido del item anterior.

* `<Ctrl-y>`: Confirma el item seleccionado.

* `<Ctrl-e>`: Cancela el proceso de completado y esconde el menú.

* `<Enter>`: Si el item fue seleccionado con `<Up>` o `<Down>` confirma la selección. Si no hay ningún item seleccionado, esconde el menú. Si no, inserta un salto de línea.

Desafortunadamente, el menú de completado de código en Neovim no ofrece ningún suporte para "snippets" de código. Para eso tenemos `mini.snippets`. Vale la pena mencionar que las nuevas versiones de Neovim sí tienen un suporte básico para snippets... pero incluso la versión más reciente no tiene tantas funcionalidades como `mini.snippets`.

## Cliente LSP

El cliente LSP es lo que hace posible que Neovim obtenga funcionalidades que antes sólo podías encuentrar en un IDE. Cosas como saltar a la definición de una clase, renombrar una variable, inspeccionar los argumentos de una función. Ustedes entienden. Pero este "cliente" no funciona por sí solo, necesitamos un servidor. En este caso un "servidor" debe ser un programa externo que implementa el [protocolo LSP](https://microsoft.github.io/language-server-protocol/overviews/lsp/overview/). Si desean conocer más del protocolo LSP y los servidores LSP pueden ver este video: [LSP explained (5 min)](https://www.youtube.com/watch?v=LaS32vctfOY).  

Para usar el cliente LSP vamos a seguir estos pasos:

* Crear atajos de teclado.
* Instalar un servidor LSP.
* Configurar el servidor LSP.

### Paso 1: los atajos

Algunos de estos atajos están basados en funcionalidades que trae Neovim por defecto, la diferencia es que estos utilizan el cliente LSP. En este caso se reemplaza el atajo original con el nuestro. El resto utilizará el prefijo `<leader>l`.

La convención que se aplica para estos atajos es crearlos si existe un servidor LSP activo en el archivo que estamos editando. Entonces lo que hacemos es crear un autocomando en el evento `LspAttach`.

```lua
vim.api.nvim_create_autocmd('LspAttach', {
  callback = function(event)
    local opts = {buffer = event.buf}

    -- Muestra información sobre símbolo debajo del cursor
    vim.keymap.set('n', 'K', '<cmd>lua vim.lsp.buf.hover()<cr>', opts)

    -- Saltar a definición
    vim.keymap.set('n', 'gd', '<cmd>lua vim.lsp.buf.definition()<cr>', opts)

    -- Formatear código
    vim.keymap.set({'n', 'x'}, 'gq', '<cmd>lua vim.lsp.buf.format({async = true})<cr>', opts)

    -- Mostrar argumentos de función
    vim.keymap.set('i', '<C-s>', '<cmd>lua vim.lsp.buf.signature_help()<cr>', opts)

    -- Saltar a declaración
    vim.keymap.set('n', '<leader>ld', '<cmd>lua vim.lsp.buf.declaration()<cr>', opts)

    -- Mostrar implementaciones
    vim.keymap.set('n', '<leader>li', '<cmd>lua vim.lsp.buf.implementation()<cr>', opts)

    -- Saltar a definición de tipo
    vim.keymap.set('n', '<leader>lt', '<cmd>lua vim.lsp.buf.type_definition()<cr>', opts)

    -- Listar referencias
    vim.keymap.set('n', '<leader>lr', '<cmd>lua vim.lsp.buf.references()<cr>', opts)

    -- Renombrar símbolo
    vim.keymap.set('n', '<leader>ln', '<cmd>lua vim.lsp.buf.rename()<cr>', opts)

    -- Listar "code actions" disponibles en la posición del cursor
    vim.keymap.set('n', '<leader>la', '<cmd>lua vim.lsp.buf.code_action()<cr>', opts)
  end,
})
```

### Paso 2: Instalar un servidor LSP

La comunidad de Neovim ha creado un recurso donde podemos encontrar una lista de servidores LSP. Esta se encuentra en la documentación del plugin `nvim-lspconfig`, en el archivo [doc/configs.md](https://github.com/neovim/nvim-lspconfig/blob/master/doc/configs.md).

Me gusta proveer ejemplos prácticos que ustedes pueden probar, así que voy a mostrarles los comandos para instalar el servidor LSP de los lenguajes `go` y `rust`.

Si ya tienen en su sistema la [herramienta para go](https://go.dev/doc/install), pueden instalar el servidor LSP ejecutando este comando en su terminal.

```
go install golang.org/x/tools/gopls@latest
```

Para `rust`, si ya tienen [rustup](https://rust-lang.github.io/rustup/) pueden instalar el servidor LSP `rust_analyzer`.

```
rustup component add rust-analyzer
```

### Paso 3: Configurar un servidor LSP

Ahora deben "habilitar" cada servidor LSP instalado en el sistema.

En **Neovim v0.11** pueden usar la función [vim.lsp.enable()](https://neovim.io/doc/user/lsp.html#vim.lsp.enable()).

```lua
vim.lsp.enable('example_server')
```

Pero en **Neovim v0.10** o alguna versión anterior, deben usar la función `.setup()` del módulo `lspconfig`.

```lua
require('lspconfig').example_server.setup({})
```

En estos ejemplos `example_server` debe ser el nombre del servidor.

Para utilizar los servidores LSP de `go` y `rust` podemos usar este método.

```lua
vim.lsp.enable({'gopls', 'rust_analyzer'})
```

O este.

```lua
require('lspconfig').gopls.setup({})
require('lspconfig').rust_analyzer.setup({})
```

Si quieren que su configuración sea compatible en diferentes versiones de Neovim pueden crear una función que invoque el método correcto de acuerdo a la versión de Neovim que están usando.

```lua
-- Esta función utiliza el módulo lspconfig en versiones anteriores de Neovim.
-- vim.lsp.enable() sólo está disponible en Neovim v0.11
local function lsp_setup(server, opts)
  if vim.fn.has('nvim-0.11') == 0 then
    require('lspconfig')[server].setup(opts)
    return
  end

  if not vim.tbl_isempty(opts) then
    vim.lsp.config(server, opts)
  end

  vim.lsp.enable(server)
end
```

Ahora pueden configurar sus servidores de esta manera:

```lua
lsp_setup('gopls', {})
lsp_setup('rust_analyzer', {})
```

Bien. Por los momentos hay un puñado de servidores LSP que no pueden ser habilitados usando `vim.lsp.enable()`. La lista de servidores que aún no pueden usar el nuevo método está aquí: [nvim-lspconfig/issues/3705](https://github.com/neovim/nvim-lspconfig/issues/3705).

### Deben tener en cuenta...

Cada servidor LSP es parte de un proyecto independiente. Es decir, un servidor LSP tiene su propio equipo de desarrollo que no está relacionado con Neovim. Tienen sus propios defectos (bugs), limitaciones y en ocasiones tienen sus propios archivos de configuración.

Algunos servidores LSP fueron creados exclusivamente para VS Code. El hecho de que Neovim (u otros editores) puedan usarlos es mera coincidencia. Tengo entendido que los servidores LSP para `html` y `css` entran en esta categoría. Estos pueden tener funcionalidades que sólo VS Code puede usar, o que tienen mejor desempeño en VS Code.

## Mención honorífica

[nvim-treesitter](https://github.com/nvim-treesitter/nvim-treesitter) es un plugin que fue creado en 2020. Mucha gente en la comunidad de Neovim dirá que es indispensable, y yo concuerdo hasta cierto punto. Es un plugin increíble... cuando otro plugin lo utiliza como dependencia. Verán, `tree-sitter` le permite a Neovim recopilar información sobre el archivo actual, y los autores de plugins pueden hacer cosas grandiosas con esa información. Para los "usuarios casuales" como nosotros existen algunos módulos que podemos habilitar. Uno de esos módulos puede mejorar el resaltado de código de muchos lenguajes de programación. Esa es la característica principal por la que `nvim-treesitter` se volvió famoso.

Pero hay cosas que deben saber:

* Necesita un compilador de `C`.
  - Por lo general no es problema en `linux` o `mac` pero en `windows` es un tema complicado.
* Sólo garantiza soporte para la versión estable actual de Neovim.
  - Puede dejar el soporte para versiones previas en cualquier momento.
* Se considera como un plugin experimental.
  - Puede introducir cambios drásticos en cualquier momento.
  - Usan `git tags` por si necesitan descargar una [versión previa](https://github.com/nvim-treesitter/nvim-treesitter/tags).

Para conocer más detalles de `tree-sitter` pueden ver este video: [tree-sitter explained (15 min)](https://www.youtube.com/watch?v=09-9LltqWLY).

Si quieren probar nvim-treesitter, este es el código para habilitar el módulo de resaltado de código.

```lua
require('nvim-treesitter.configs').setup({
  auto_install = true,
  highlight = {
    enable = true,
  },
})
```

## init.lua

```lua
-- Guia oficial de lua en Neovim:
-- https://neovim.io/doc/user/lua-guide.html

vim.o.number = true
vim.o.tabstop = 2
vim.o.shiftwidth = 2
vim.o.smartcase = true
vim.o.ignorecase = true
vim.o.wrap = false
vim.o.hlsearch = false
vim.o.signcolumn = 'yes'

-- Tecla Espacio como <leader>
vim.g.mapleader = ' '

-- Interación con el portapapeles
vim.keymap.set({'n', 'x', 'o'}, 'gy', '"+y', {desc = 'Copiar al portapapel'})
vim.keymap.set({'n', 'x', 'o'}, 'gp', '"+p', {desc = 'Pegar del portapapel'})

-- Comandos
vim.keymap.set('n', '<leader>w', '<cmd>write<cr>', {desc = 'Guardar'})
vim.keymap.set('n', '<leader>q', '<cmd>quitall<cr>', {desc = 'Salir de vim'})

-- Tema del editor
local ok_theme = pcall(vim.cmd.colorscheme, 'retrobox')
if not ok_theme then
  vim.cmd.colorscheme('habamax')
end

require('mini.snippets').setup({})
require('mini.completion').setup({})

require('mini.files').setup({})
vim.keymap.set('n', '<leader>e', '<cmd>lua MiniFiles.open()<cr>', {desc = 'Explorador de archivos'})

require('mini.pick').setup({})
vim.keymap.set('n', '<leader><space>', '<cmd>Pick buffers<cr>', {desc = 'Listar archivos abiertos'})
vim.keymap.set('n', '<leader>ff', '<cmd>Pick files<cr>', {desc = 'Listar todos los archivos'})
vim.keymap.set('n', '<leader>fh', '<cmd>Pick help<cr>', {desc = 'Listar help tags'})

vim.api.nvim_create_autocmd('LspAttach', {
  callback = function(event)
    local opts = {buffer = event.buf}

    -- Muestra información sobre símbolo debajo del cursor
    vim.keymap.set('n', 'K', '<cmd>lua vim.lsp.buf.hover()<cr>', opts)

    -- Saltar a definición
    vim.keymap.set('n', 'gd', '<cmd>lua vim.lsp.buf.definition()<cr>', opts)

    -- Formatear código
    vim.keymap.set({'n', 'x'}, 'gq', '<cmd>lua vim.lsp.buf.format({async = true})<cr>', opts)

    -- Mostrar argumentos de función
    vim.keymap.set('i', '<C-s>', '<cmd>lua vim.lsp.buf.signature_help()<cr>', opts)

    -- Saltar a declaración
    vim.keymap.set('n', '<leader>ld', '<cmd>lua vim.lsp.buf.declaration()<cr>', opts)

    -- Mostrar implementaciones
    vim.keymap.set('n', '<leader>li', '<cmd>lua vim.lsp.buf.implementation()<cr>', opts)

    -- Saltar a definición de tipo
    vim.keymap.set('n', '<leader>lt', '<cmd>lua vim.lsp.buf.type_definition()<cr>', opts)

    -- Listar referencias
    vim.keymap.set('n', '<leader>lr', '<cmd>lua vim.lsp.buf.references()<cr>', opts)

    -- Renombrar símbolo
    vim.keymap.set('n', '<leader>ln', '<cmd>lua vim.lsp.buf.rename()<cr>', opts)

    -- Listar "code actions" disponibles en la posición del cursor
    vim.keymap.set('n', '<leader>la', '<cmd>lua vim.lsp.buf.code_action()<cr>', opts)
  end,
})

-- Esta función utiliza el módulo lspconfig en versiones anteriores de Neovim.
-- vim.lsp.enable() sólo está disponible en Neovim v0.11
local function lsp_setup(server, opts)
  if vim.fn.has('nvim-0.11') == 0 then
    require('lspconfig')[server].setup(opts)
    return
  end

  if not vim.tbl_isempty(opts) then
    vim.lsp.config(server, opts)
  end

  vim.lsp.enable(server)
end

-- Lista de servidores LSP:
-- https://github.com/neovim/nvim-lspconfig/blob/master/doc/configs.md
lsp_setup('gopls', {})
lsp_setup('rust_analyzer', {})
```

