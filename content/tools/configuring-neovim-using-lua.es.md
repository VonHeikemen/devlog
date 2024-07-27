+++
title = "Todo lo que necesitan saber para configurar neovim usando lua"
description = "Tus primeros pasos hacia una configuración creada en lua"
date = 2021-07-29
updated = 2024-07-27
lang = "es"
[taxonomies]
tags = ["vim", "neovim", "shell"]
[extra]
shared = [
  ["dev.to", "https://dev.to/vonheikemen/todo-lo-que-necesitan-saber-para-configurar-neovim-usando-lua-2fon"],
  ["Hashnode", "https://vonheikemen.hashnode.dev/everything-you-need-to-know-to-configure-neovim-using-lua-es"]
]
+++

Después de mucho tiempo en desarrollo neovim 0.5 por fin fue liberado como una versión estable. Entre las nuevas mejoras tenemos un mejor soporte para lua y la promesa de una api estable para crear nuestra configuración usando este lenguaje. Aprovechando esto hoy voy compartir con ustedes todo lo que aprendí mientras migraba mi configuración de vimscript a lua.

Si neovim es completamente nuevo para ustedes y su objetivo es configurarlo desde cero, les recomiendo leer esto: [Cómo crear tu primera configuración de Neovim usando lua](@/tools/build-your-first-lua-config-for-neovim.es.md).

Aquí voy a hablar de lo que podemos hacer en lua y su interacción con vimscript. Aunque presentaré muchos ejemplos, no les diré qué configuración deben activar o con qué valor. No hablaré de cosas específicas de algún lenguaje, y tampoco abordaré el tema de "convertir neovim en un IDE". Sólo espero darles una buena base para que puedan migrar su propia configuración.

Asumiré que su sistema operativo es linux (o algo parecido) y que su configuración está en la ruta `~/.config/nvim/init.vim`. Todo lo que mencionaré debería funcionar en cada sistema donde pueden instalar neovim, sólo tengan en cuenta que la ruta de `init.vim` puede ser diferente en su caso.

Comencemos.

## Primeros pasos

Lo primero que deben saber es que podemos incorporar código escrito en lua directamente en nuestro `init.vim`. Podemos comenzar la migración de manera gradual desde `init.vim`, y reemplazar `init.vim` por `init.lua` sólo cuando estemos listos.

Empezemos con el "hola mundo" para probar que todo funciona de manera correcta. Pueden colocar esto en su `init.vim`:

```vim
lua <<EOF
print('hola desde lua')
EOF
```

Si ejecutan nuevamente `init.vim` o reinician neovim el mensaje `hola desde lua` debería aparecer justo debajo de su barra de estado. Aquí estamos utilizando algo llamado `lua-heredoc`. Todo lo que está encerrado en `<<EOF ... EOF` es considerado un "script" que será evaluado por el comando `lua`. Resulta útil cuando queremos ejecutar código con múltiples líneas pero no es estrictamente necesario si sólo necesitamos una. También podemos hacer esto.

```vim
lua print('esto también funciona')
```

Pero si vamos a llamar código lua desde vimscript lo que yo recomendaría sería "llamar" un script verdadero. En lua podemos hacer eso con la función `require`. Para que esto funcione necesitamos crear una carpeta llamada `lua` y colocarla en un directorio que se encuentre en el `runtimepath` de neovim.

Lo más conveniente sería usar el mismo directorio desde se encuentra `init.vim`, entonces lo que haremos será crear `~/.config/nvim/lua`, y dentro de esa carpeta colocaremos el script que llamaremos `basic.lua`. Por ahora sólo imprimiremos un mensaje.

```lua
print('hola desde ~/config/nvim/lua/basic.lua')
```

Ahora desde `init.vim` podemos invocarlo de esta manera.

```vim
lua require('basic')
```

Aquí neovim buscará en todos los directorios del `runtimepath` una carpeta llamada `lua` y luego dentro de esa carpeta buscará `basic.lua`. El último script que encuentre que cumpla estas condiciones será ejecutado.

Una particularidad de lua que van a encontrar es que podemos usar el `.` como un separador de ruta. Por ejemplo, imaginemos que tenemos el archivo `~/.config/nvim/lua/usermod/settings.lua`. Si queremos llamar al archivo `settings.lua` podemos hacerlo de la siguiente manera.

```lua
require('usermod.settings')
```

Es una convención común que van a encontrar si revisan el código de otras personas. Sólo recuerden que el punto es un separador de ruta.

Con este conocimiento ya están preparados para empezar su configuración en lua.

## Opciones del editor

Cada opción en neovim está disponible para nosotros en la variable global llamada `vim`... bueno, es más que una variable, es un módulo. Con `vim` tenemos acceso a opciones, a la api de neovim e incluso un conjunto de funciones auxiliares (una librería estándar). Por ahora lo que nos interesa es algo que llaman "meta-accessors", es lo que usaremos para acceder a las opciones del editor.

### Ámbitos

Al igual que en vimscript, en lua tenemos diferentes ámbitos para cada opción. Tenemos opciones que son globales, opciones que actúan sólo en una ventana, otras que aplican sólo para los archivos abiertos, etc. Cada uno tiene su propio espacio dentro del módulo `vim`.

* vim.o

Sirve para leer o modificar opciones generales del editor.

```lua
vim.o.background = 'light'
```

* vim.wo

Lee o modifica valores específicos para una ventana.

```lua
vim.wo.colorcolumn = '80'
```

* vim.bo

Lee o modifica valores específicos para un archivo (buffer).

```lua
vim.bo.filetype = 'lua'
```

* vim.g

Lee o modifica valores globales. Aquí más que todo encontrarán valores usados por plugins. La única opción que conozco que no está necesariamente ligada a un plugin es la tecla líder.

```lua
-- espacio como tecla líder
vim.g.mapleader = ' '
```

Una cosa que deben tener en cuenta es que algunos nombres de variables en vimscript no son válidos en lua. Aún podemos acceder a ellos pero debemos usar una sintaxis diferente. Por ejemplo [vim-zoom](https://github.com/dhruvasagar/vim-zoom) tiene una variable llamada `zoom#statustext` y en vimscript podemos modificarla usando `let`, así:

```vim
let g:zoom#statustext = 'Z'
```

En lua tendríamos que hacer esto:

```lua
vim.g['zoom#statustext'] = 'Z'
```

Esta sintaxis también nos sirve para acceder a propiedades que tienen el nombre de una palabra reservada. Por ejemplo `for`, `do` y `end` son palabras reservadas; entonces si tenemos alguna variable con esas propiedades podemos usar esta sintaxis para evitar un error.

* vim.env

Lee o modifica variables de entorno.

```lua
vim.env.FZF_DEFAULT_OPTS = '--layout=reverse'
```

Tengo entendido que los cambios a estas variables sólo tendrán efecto en la sesión activa del editor.

Pero entonces ¿cómo sabemos qué "ámbito" usar cuando vamos a crear nuestra configuración? No se preocupen por eso, pueden pensar en `vim.o` y compañía como una especie de acceso rápido a una variable, es mejor usarlo para leer valores. Para realizar modificaciones tenemos otra propiedad.

### vim.opt

Con `vim.opt` podremos modificar opciones generales, de ventana y de archivo.

```lua
-- opción de buffer
vim.opt.autoindent = true

-- opción de ventana
vim.opt.cursorline = true

-- opción general
vim.opt.autowrite = true
```

En este sentido `vim.opt` actúa como el comando `:set` en vimscript, nos da una manera consistente de declarar nuestros valores.

Puedo decirles que asignar `vim.opt` a una variable llamada `set` funciona a la perfección. Por ejemplo, imaginemos que tenemos este fragmento en vimscript.

```vim
" comportamiento de la tecla tab
set tabstop=2
set shiftwidth=2
set softtabstop=2
set expandtab
```

Podemos migrar sin esfuerzo a lua de esta manera.

```lua
local set = vim.opt

-- comportamiento de la tecla tab
set.tabstop = 2
set.shiftwidth = 2
set.softtabstop = 2
set.expandtab = true
```

> Cuando declaran variables en lua no olviden la palabra clave `local`. En lua las variables son globales por defecto (esto incluye las funciones).

¿Qué pasa con las variables globales o las variables de entorno? Para eso deberían seguir usando `vim.g` y `vim.env` respectivamente.

Lo interesante de `vim.opt` es que cada propiedad es como un objeto especial, son lo que llaman "meta-tabla". Quiere decir que son objetos que implementan sus propias funciones para operaciones comunes.

En el primer ejemplo teníamos esto: `vim.opt.autoindent = true`, tal vez estén pensando que también pueden inspeccionar su valor de manera normal, así:

```lua
print(vim.opt.autoindent)
```

No obtendrán el valor que esperan, `print` les dirá que `vim.opt.autoindent` es una tabla. Si quieren acceder a su valor deben usar el método `:get()`.

```lua
print(vim.opt.autoindent:get())
```

Si realmente quieren saber qué hay dentro de `vim.opt.autoindent` pueden usar `vim.inspect`.

```lua
print(vim.inspect(vim.opt.autoindent))
```

O incluso mejor, si tienen la version 0.7 pueden user el comando `:lua =` para inspeccionar su valor desde el modo de comandos.

```vim
:lua = vim.opt.autoindent
```

Eso les mostrará el estado interno de la propiedad.

### Tipos de datos

Incluso cuando asignamos un valor a una propiedad de `vim.opt` hay algo de magia en el fondo. Ahora tienen que saber cómo `vim.opt` maneja los valores en comparación con vimscript.

* Booleanos

Puede que no parezca muy especial pero aún creo que vale la pena mencionarlos.

En vimscript para activar o desactivar alguna opción hacemos esto.

```vim
set cursorline
set nocursorline
```

Este es el equivalente en lua.

```lua
vim.opt.cursorline = true
vim.opt.cursorline = false
```

* Listas

Para algunas opciones se espera una lista separada por comas. En este caso podríamos proveer la cadena texto nosotros mismos.

```lua
vim.opt.wildignore = '*/cache/*,*/tmp/*'
```

Ó podríamos usar una tabla.

```lua
vim.opt.wildignore = {'*/cache/*', '*/tmp/*'}
```

Si revisan el contenido de `vim.o.wildignore` notarán que es la cadena de texto `*/cache/*,*/tmp/*`, eso significa que funcionó. Y si quieren estar muy seguros de que funcionó, revisen con este comando dentro de neovim.

```vim
:set wildignore?
```

Obtandrán el mismo resultado.

La magia no termina ahí. En ocasiones no necesitamos sobreescribir los valores de la lista, a veces queremos agregar un elemento o tal vez necesitamos eliminarlo. Para facilitar estas tareas `vim.opt` tiene soporte para las siguientes operaciones:

**Añadir elemento al final de la lista**

Tomemos la opción `errorformat` como ejemplo. Si queremos añadir algo usando vimscript utilizamos este comando.

```vim
set errorformat+=%f\|%l\ col\ %c\|%m
```

En lua podemos lograr el mismo efecto de dos maneras:

Usando el operador `+`.

```lua
vim.opt.errorformat = vim.opt.errorformat + '%f|%l col %c|%m'
```

O la función `append`.

```lua
vim.opt.errorformat:append('%f|%l col %c|%m')
```

**Añadir al inicio**

En vimscript:

```vim
set errorformat^=%f\|%l\ col\ %c\|%m
```

En lua:

```lua
vim.opt.errorformat = vim.opt.errorformat ^ '%f|%l col %c|%m'

-- o su equivalente

vim.opt.errorformat:prepend('%f|%l col %c|%m')
```

**Eliminar un elemento**

En vimscript:

```vim
set errorformat-=%f\|%l\ col\ %c\|%m
```

En lua:

```lua
vim.opt.errorformat = vim.opt.errorformat - '%f|%l col %c|%m'

-- o su equivalente

vim.opt.errorformat:remove('%f|%l col %c|%m')
```

* Pares

Algunas opciones tienen un formato donde se debe especificar una propiedad y el valor para esa propiedad. Como ejemplo tenemos `listchars`.

```vim
set listchars=tab:▸\ ,eol:↲,trail:·
```

En lua podemos usar tablas para representar esta opción.

```lua
vim.opt.listchars = {eol = '↲', tab = '▸ ', trail = '·'}
```

> Nota: para que la opción `listchars` tenga efecto deben activar la opción `list`. Ver [:help listchars](https://neovim.io/doc/user/options.html#'listchars')

Ya que esto también es una tabla podemos hacer las mismas operaciones que mencioné en la sección anterior.

## Invocando funciones de vim

Vimscript como cualquier otro lenguaje de programación tiene sus propias funciones nativas ([muchas funciones](https://neovim.io/doc/user/usr_41.html#function-list)) y gracias al módulo `vim` podemos acceder a ellas usando `vim.fn`. Al igual que `vim.opt`, `vim.fn` es una meta-tabla, pero en este caso nos permite tener una sintaxis conveniente para llamar funciones de vim. Podemos usar `vim.fn` para invocar funciones nativas, funciones creadas por nosotros mismos e incluso funciones de plugins que no están en escritos en lua.

Podríamos por ejemplo validar la versión de neovim de esta manera:

```lua
if vim.fn.has('nvim-0.7') == 1 then
  print('tenemos neovim 0.7')
end
```

¿Por qué estoy comparando el resultado de `has` con un `1`? Vimscript no siempre ha tenido booleanos, estos fueron agregados a partir de la versión `7.4.1154`. Entonces funciones como `has` devuelven `0` o `1` y en lua cualquiera de esos valores pasa la evaluación de un `if`.

Hay casos donde el nombre de la función no es válido en lua. Ya saben que podemos usar corchetes así:

```lua
vim.fn['fzf#vim#files']('~/projects', false)
```

Pero también podemos usar la función `vim.call`.

```lua
vim.call('fzf#vim#files', '~/projects', false)
```

En la práctica `vim.fn.unafuncion()` y `vim.call('unafuncion')` tienen exactamente el mismo efecto.

Ahora déjenme mostrarle algo genial. La integración lua-vimscript es tan buena que podríamos utilizar un "plugin manager" sin necesidad de adaptaciones especiales.

### vim-plug en lua

Sé que hay un montón de gente que utiliza [vim-plug](https://github.com/junegunn/vim-plug/) y tal vez se estén preguntando si tienen que migrar a un plugin manager que esté escrito en lua. No tienen que hacerlo, `vim.fn` y su acompañante `vim.call` son suficientes para usarlo desde lua.

```lua
local Plug = vim.fn['plug#']

vim.call('plug#begin')

-- Los plugins van aquí
-- ....

vim.call('plug#end')
```

Esas tres líneas de código es todo lo que necesitan. Pueden probarlo, esto funciona.

```lua
local Plug = vim.fn['plug#']

vim.call('plug#begin')

Plug 'wellle/targets.vim'
Plug 'tpope/vim-surround'
Plug 'tpope/vim-repeat'

vim.call('plug#end')
```

Todo eso es válido en lua. Si una función sólo recibe un argumento y ese argumento es una cadena de texto o una tabla, pueden omitir los paréntesis.

Si necesitan usar el segundo argumento de `Plug` deben usar los paréntesis y el segundo argumento debe ser una tabla. Vamos a comparar. Si en vimscript tenemos esto:

```vim
Plug 'scrooloose/nerdtree', {'on': 'NERDTreeToggle'}
```

En lua debemos representarlo así:

```lua
Plug('scrooloose/nerdtree', {on = 'NERDTreeToggle'})
```

Desafortunadamente `vim-plug` tiene opciones llamadas `for` y `do` que como ya mencioné son palabras reservadas, para estos casos debemos envolver el nombre de la propiedad con corchetes y comillas.

```lua
Plug('junegunn/goyo.vim', {['for'] = 'markdown'})
```

Una última cosa, la opción `do` se usa para ejecutar una acción cuando se instala o actualiza un plugin. Esta opción acepta una cadena de texto o una función. Si queremos usar una función no estamos obligados a pasar una "función de vim", podemos usar una función de lua sin ningún problema.

```lua
Plug('VonHeikemen/rubber-themes.vim', {
  ['do'] = function()
    vim.opt.termguicolors = true
    vim.cmd('colorscheme rubber')
  end
})
```

Ahora ya saben, no tienen que preocuparse si su plugin manager no está escrito en lua. Siempre y cuando exponga alguna función podremos usarlo en lua.

## Vimscript aún es nuestro amigo

Algunos de ustedes habrán notado que en el último ejemplo usé `vim.cmd` para configurar el tema del editor. Esto es porque aún hay cosas que no podemos hacer en lua. Con `vim.cmd` podemos ejecutar expresiones escritas en vimscript. Esto nos permite invocar comandos que no tienen un equivalente en lua.

`vim.cmd` también es capaz de ejecutar múltiples líneas de vimscript. Significa que podemos hacer múltiples cosas en una sola llamada a `vim.cmd`.

```lua
vim.cmd [[
  syntax enable
  colorscheme rubber
]]
```

Cualquier fragmento de su `init.vim` que no puedan "traducir" a lua pueden colocarlo una cadena de texto y pasarlo a `vim.cmd`.

Ya que podemos ejecutar cualquier comando de vim tengo que mencionar que eso incluye `source`, con esto podemos invocar scripts escritos en vimscript. Por ejemplo, digamos que estamos migrando nuestra configuración pero aún no estamos listos para migrar nuestros atajos de teclado. Podemos crear un archivo `keymaps.vim` con nuestros atajos y ejecutarlo desde lua.

```lua
vim.cmd 'source ~/.config/nvim/keymaps.vim'
```

## Atajos de teclado

No, no necesitamos vimscript. Podemos crearlos usando lua.

Para estos casos tenemos la función `vim.keymap.set` (introducida en la versión v0.7). Esta función acepta 4 argumentos.

* Modo (o una lista de modos) en el que tendrá efecto nuestro atajo. Pero no podemos usar el nombre del modo, necesitamos usar su forma abreviada. Pueden encontrar una lista completa y detallada [aquí](https://github.com/nanotee/nvim-lua-guide#defining-mappings).
* Atajo que queremos vincular.
* La acción que queremos ejecutar.
* Opciones extra. Estas opciones son las mismas que usaríamos en vimscript, pueden encontrar la lista [aquí](https://neovim.io/doc/user/map.html#:map-arguments). Pero también acepta un par de nuevas opciones, pueden encontrar los detalles en la documentación [:help vim.keymap.set()](https://neovim.io/doc/user/lua.html#vim.keymap.set()).

Digamos que queremos trasladar este atajo a lua.

```vim
nnoremap <Leader>w :write<CR>
```

Tendríamos que hacer esto.

```lua
vim.keymap.set('n', '<Leader>w', ':write<CR>')
```

Por defecto los atajos serán "no recursivos", lo que significa que no debemos preocuparnos por ocasionar ciclos infinitos. Podríamos crear versiones alternativas de algún atajo. Por ejemplo, si queremos centrar la pantalla después de realizar una búsqueda con `*`.

```lua
vim.keymap.set('n', '*', '*zz', {desc = 'Buscar y centrar pantalla'})
```

Podemos usar `*` en el atajo y la acción, esto no creará ningún conflicto.

Hay ocasiones donde necesitamos un atajo recursivo, generalmente cuando la acción que queremos ejecutar fue creada por un plugin. En esta situación podemos proveer la opción `remap = true` en el último argumento.

```lua
vim.keymap.set('n', '<leader>e', '%', {remap = true, {desc = 'Ir al par correspondiente'}})
```

Un beneficio extra de esta función es que nos permite usar funciones de lua como nuestra "acción".

```lua
vim.keymap.set('n', 'Q', function()
  print('Hola')
end, {desc = 'Saludar'})
```

Hablemos un poco de la opción `desc`. Esta nos permite agregar una descripción a nuestros atajos. Podemos leer estas descripciones con el comando `:map <atajo>`.

Entonces, ejecutar `:map *` nos mostrará:

```
n  *           * *zz
                 Buscar y centrar pantalla
```

Querrán utilizar estas descripciones cuando su atajo ejecute una función de lua. Por qué? Si revisamos `Q` con el comando `:map Q` obtenemos:

```
n  Q           * <Lua function 50>
                 Saludar
```

No podemos leer el código de la función, entonces el único indicio que tenemos de su funcionalidad es la descripción. Tambien vale la pena mencionar que plugins pueden extraer esta descripción y mostrarlas de una manera más amigable.

## Comandos de usuario

A partir de la version v0.7 neovim nos permite crear nuestros propios "ex-commands" usando lua, con la función `vim.api.nvim_create_user_command`.

`nvim_create_user_command` espera tres argumentos:

* Nombre del comando. El nombre **debe** empezar con una letra mayúscula.
* Comando. Puede ser una cadena de texto o una función de lua.
* Atributos. Una tabla con las características opcionales que puede tener nuestro comando. Pueden encontrar los detalles en la documentación [:help nvim_create_user_command()](https://neovim.io/doc/user/api.html#nvim_create_user_command()) y [:help command-attributes](https://neovim.io/doc/user/map.html#command-attributes).

Digamos que tenemos este comando en vimscript.

```vim
command! -bang ProjectFiles call fzf#vim#files('~/projects', <bang>0)
```

En lua tenemos la posibilidad de usar vimscript in una cadena de texto.

```lua
vim.api.nvim_create_user_command(
  'ProjectFiles',
  "call fzf#vim#files('~/projects', <bang>0)",
  {bang = true}
)
```

O podemos usar una función de lua.

```lua
vim.api.nvim_create_user_command(
  'ProjectFiles',
  function(input)
    vim.call('fzf#vim#files', '~/projects', input.bang)
  end,
  {bang = true, desc = 'Buscar en directorio projects'}
)
```

Sí, podemos agregar descripciones a nuestros comandos. Pero en esta ocasión sólo estarán disponibles si el comando ejecuta una función de lua. Para inspeccionar un comando podemos ejecutar `:command <nombre>`. Entonces, el comando `:command ProjectFiles` nos mostrará:

```
    Name              Args Address Complete    Definition
!   ProjectFiles      0                        Buscar en directorio projects
```

Si nuestro comando ejecuta una expresión de vimscript nos mostrará ese código en la columna `Definition`.

## Autocomandos

Los autocomandos nos permiten ejecutar funciones y comandos cuando neovim emite un evento. Pueden encontrar la lista de eventos en la documentación: [:help events](https://neovim.io/doc/user/autocmd.html#events).

Digamos que tenemos un autocomando que modifica ligeramente un tema del editor, específicamente el tema `rubber`.

```vim
augroup highlight_cmds
  autocmd!
  autocmd ColorScheme rubber highlight String guifg=#FFEB95
augroup END
```

> Este bloque debe ejecutarse antes de invocar el comando `colorscheme`.

Este es el equivalente en lua.

```lua
local augroup = vim.api.nvim_create_augroup('highlight_cmds', {clear = true})

vim.api.nvim_create_autocmd('ColorScheme', {
  pattern = 'rubber',
  group = augroup,
  command = 'highlight String guifg=#FFEB95'
})
```

Noten que aquí estamos usando una opción llamada `command`, esta nos permite utilizar únicamente expresiones escritas en vimscript. También podemos usar una función de lua pero no con la propiedad `command`, tenemos que usar `callback`.

```lua
local augroup = vim.api.nvim_create_augroup('highlight_cmds', {clear = true})

vim.api.nvim_create_autocmd('ColorScheme', {
  pattern = 'rubber',
  group = augroup,
  desc = 'Cambiar color de cadenas de texto'
  callback = function()
    vim.api.nvim_set_hl(0, 'String', {fg = '#FFEB95'})
  end
})
```

Los autocomandos también pueden tener una descripción. Pero al igual que los comandos de usuario sólo estarán disponibles si están vinculadas a una función de lua. Si queremos revisar los autocomandos de un evento podemos ejecutar el comando `:autocmd <evento> <patrón>`. El comando `:autocmd ColorScheme rubber` debería mostrarnos:

```
ColorScheme
    rubber    Cambiar color de cadenas de texto
```

Para conocer más detalles de los autocomandos pueden leer la documentación, [:help nvim_create_autocmd()](https://neovim.io/doc/user/api.html#nvim_create_autocmd()) y [:help autocmd](https://neovim.io/doc/user/autocmd.html#autocommand).

## Plugin manager

Tal vez quieran usar un plugin manager que este escrito en lua sólo porque sí. Por lo que he visto estas son sus opciones:

* [paq-nvim](https://github.com/savq/paq-nvim/)

Es un manejador de plugins rápido y sencillo. No es broma, tiene alrededor de 500 líneas de código y fue creado para descargar, actualizar y eliminar plugins. Es todo. Si eso es lo único que necesitan no busquen más, este es el manejador que quieren.

* [lazy.nvim](https://github.com/folke/lazy.nvim)

Este se caracteriza por optimizar el tiempo de carga de nuestros plugins. Por defecto buscará cargar nuestros plugins sólo cuando sea necesario. Ofrece una interfaz agradable para actualizar, inspeccionar y eliminar plugins. También nos permite declarar nuestros plugins usando módulos de lua. Podemos especificar versión, dependencias, configuración, entre otras cosas.

* [mini.deps](https://github.com/echasnovski/mini.deps)

Ofrece un punto intermedio entre `paq-nvim` y `lazy.nvim`. Tiene funcionalidades útiles que no se encuentran en `paq.nvim`, por ejemplo, la habilidad de devolver un plugin a una version anterior. Pero `mini.deps` no tiene opciones avanzadas como `lazy.nvim`.

## Conclusión

Aprendimos cómo usar lua desde vimscript. Sabemos cómo usar vimscript desde lua. Ahora tenemos todas las herramientas para activar, desactivar y modificar cualquier tipo de opción o variable disponible en neovim. Conocemos los métodos para crear nuestros atajos de teclado. Aprendimos sobre comandos y autocomandos. Sabemos cómo usar un manejador de plugins desde lua ya sea que esté escrito en lua o no. Ya estamos listos.

Para los que quieren ver un ejemplo de la vida real, aquí les algunos recursos.

Esta es una "plantilla" que pueden copiar y modificar a su gusto:

* nvim-light: [init.lua](https://github.com/VonHeikemen/nvim-light/blob/main/init.lua) | [github link](https://github.com/VonHeikemen/nvim-light)

Y esta es mi configuración personal en github:

* [neovim (v0.10)](https://github.com/VonHeikemen/dotfiles/tree/master/my-configs/neovim)

## Fuentes

* [learn x in y minutes: where X=lua](https://learnxinyminutes.com/docs/lua/)
* [nvim-lua-guide](https://github.com/nanotee/nvim-lua-guide)
* [:help lua-heredoc](https://neovim.io/doc/user/lua.html#:lua-heredoc)
* [:help lua-vim-variables](https://neovim.io/doc/user/lua.html#lua-vim-variables)
* [:help lua-stdlib](https://neovim.io/doc/user/lua.html#lua-stdlib)
* [:help function-list](https://neovim.io/doc/user/usr_41.html#function-list)
* [curist's bundle.lua](https://github.com/curist/dotvim/blob/98b161f0759d3316fcf6a776d03665d6ab4827ee/bundles.lua)

