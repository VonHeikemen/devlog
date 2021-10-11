+++
title = "Todo lo que necesitan saber para configurar neovim usando lua"
description = "Tus primeros pasos hacia una configuración creada en lua"
date = 2021-07-29
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

Vamos a hablar de lo que podemos hacer en lua y su interacción con vimscript. Aunque presentaré muchos ejemplos, no les diré qué configuración deben activar o con qué valor. No hablaré de cosas específicas de algún lenguaje, y tampoco abordaré el tema de "convertir neovim en un IDE". Sólo espero darles una buena base para que puedan migrar su propia configuración.

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
if vim.fn.has('nvim-0.5') == 1 then
  print('tenemos neovim 0.5')
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

vim.call('plug#begin', '~/.config/nvim/plugged')

-- Los plugins van aquí
-- ....

vim.call('plug#end')
```

Esas tres líneas de código es todo lo que necesitan. Pueden probarlo, esto funciona.

```lua
local Plug = vim.fn['plug#']

vim.call('plug#begin', '~/.config/nvim/plugged')

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

Algunos de ustedes habrán notado que en el último ejemplo usé `vim.cmd` para configurar el tema del editor. Esto es porque aún hay cosas que no podemos hacer en lua. Por los momentos no podemos crear o invocar comandos, y tampoco podemos crear autocomandos.

Para sobrepasar estas limitaciones usamos `vim.cmd`. Esta función es capaz de ejecutar múltiples líneas de vimscript. Significa que puedes hacer múltiples cosas en una sola llamada a `vim.cmd`.

```lua
vim.cmd [[
  syntax enable
  colorscheme rubber

  command! Hola echom "hola!!"
]]
```

Cualquier fragmento de su `init.vim` que no puedan "traducir" a lua pueden colocarlo una cadena de texto y pasarlo a `vim.cmd`.

Ya que podemos ejecutar cualquier comando de vim tengo que mencionar que eso incluye `source`, con esto podemos invocar scripts escritos en vimscript. Por ejemplo, en mi configuración yo lo uso para ejecutar un script que modifica algunos colores del tema.

```lua
vim.cmd 'source ~/.config/nvim/theme.vim'
```

En este caso `theme.vim` crea un autocomando que ejecuta una función que ocurre el evento `ColorScheme`.

```vim
function! MyHighlights() abort
  hi! link Question String
  hi! link NonText LineNr

  hi! link TelescopeMatching Boolean
  hi! link TelescopeSelection CursorLine
endfunction

augroup MyColors
  autocmd!
  autocmd ColorScheme * call MyHighlights()
augroup END
```

Me gusta mantener esta parte separada en su propio script porque es muy probable que siga agregando líneas. Eso y porque no hay manera de traducirlo a lua.

## Atajos de teclado

Aquí nos encontramos en una situación interesante. Actualmente podemos definir nuestros atajos usando lua pero aún no tenemos una api "conveniente". ¿Por qué digo que no es conveniente? La manera en la que se definen los atajos no se parece a lo que estoy acostumbrado en vimscript. No es familiar. Lo otro es que no podemos asignar funciones de lua a un atajo. Sí es posible ejecutar una función de lua con un atajo pero tenemos que hacer trampa (ya les diré cómo).

En fin, estas son las dos funciones que tenemos disponibles.

* `vim.api.nvim_set_keymap`
* `vim.api.nvim_buf_set_keymap`

La primera asigna atajos globales y la segunda asigna atajos sólamente en un archivo específico.

La función `nvim_set_keymap` acepta 4 argumentos:

* Modo en el que tendrá efecto. Usualmente es la forma abreviada del modo. Pueden encontrar una lista completa y detallada [aquí](https://github.com/nanotee/nvim-lua-guide#defining-mappings).
* Atajo que queremos vincular.
* La acción que queremos ejecutar.
* Opciones extra. Estas opciones son las mismas que usaríamos en vimscript (exceptuando la opción `buffer`), pueden encontrar la lista [aquí](https://neovim.io/doc/user/map.html#:map-arguments).

> `nvim_buf_set_keymap` es casi igual, la única diferencia es que su primer argumento es el número (o id) del buffer. Si usan el número `0` neovim asume que queremos asignar el atajo al buffer que estamos editando.

Digamos que queremos trasladar este atajo a lua.

```vim
nnoremap <Leader>w :write<CR>
```

Tendríamos que hacer esto.

```lua
vim.api.nvim_set_keymap('n', '<Leader>w', ':write<CR>', {noremap = true})
```

No es lo más cómodo del mundo pero hay un par de cosas que podemos hacer para mejorar la experiencia.

* Un alias

Si prefieren un enfoque minimalista pueden simplemente asignar esta función a una variable con un nombre más corto.

```lua
local map = vim.api.nvim_set_keymap

map('n', '<Leader>w', ':write<CR>', {noremap = true})
```

* Una función auxiliar

Con este enfoque podríamos aprovecharlo para dejar algunos valores por defecto en las opciones extra. Digo esto porque es considerado una buena práctica que nuestros atajos no sean recursivos, es decir, podríamos dejar `{noremap = true}` por defecto.

```lua
local map = function(key)
  -- extraer opciones
  local opts = {noremap = true}
  for i, v in pairs(key) do
    if type(i) == 'string' then opts[i] = v end
  end

  -- soporte básico para atajos de buffer
  local buffer = opts.buffer
  opts.buffer = nil

  if buffer then
    vim.api.nvim_buf_set_keymap(0, key[1], key[2], key[3], opts)
  else
    vim.api.nvim_set_keymap(key[1], key[2], key[3], opts)
  end
end
```

Uso básico.

```lua
map {'n', '<Leader>w', ':write<CR>'}
```

Lo interesante de esta función es que aprovecha la manera en la que podemos crear tablas en lua. Esto es válido.

```lua
map {noremap = false, 'n', '<Leader>e', '%'}
```

Y también esto.

```lua
map {'n', '<Leader>e', '%', noremap = false}
```

### Invocando funciones de lua

Si aplicamos lo que aprendimos anteriormente sobre cómo llamar código lua desde vimscript podemos hacer lo siguiente:

Asumiendo que tenemos un módulo llamado `usermod` y este módulo tiene una función llamada `unafuncion`.

```lua
local M = {}

M.unafuncion = function()
  print('todo bien')
end

return M
```

Podemos invocar `unafuncion` de esta manera.

```lua
vim.api.nvim_set_keymap(
  'n',
  '<Leader>w',
  "<cmd>lua require('usermod').unafuncion()<CR>",
  {noremap = true}
)
```

Las cosas cambian un poco cuando necesitamos una expresión, en ese caso no podremos usar el comando `<cmd>lua`. Tenemos que usar la variable `v:lua` la cual nos permite invocar funciones globales.

Para ilustrar este proceso haremos un ejemplo común, haremos que la tecla `<Tab>` sea más "inteligente". Cuando el menú de autocompletado sea visible será capaz de navegar entre la lista de opciones, caso contrarío insertará un caracter `<Tab>`.

```lua
local t = function(str)
  return vim.api.nvim_replace_termcodes(str, true, true, true)
end

_G.smart_tab = function()
  if vim.fn.pumvisible() == 1 then
    return t'<C-n>'
  else
    return t'<Tab>'
  end
end

vim.api.nvim_set_keymap(
  'i',
  '<Tab>',
  'v:lua.smart_tab()',
  {noremap = true, expr = true}
)
```

> En lua `_G` es la variable global que contiene todas las variables globales. No es necesario usarla pero de esa manera queda claro que estoy declarando una variable global a propósito.

Si se preguntan porque uso `t'<C-n>'` es porque no necesitamos retornar el texto `<C-n>`, necesitamos retornar el código que representa `<C-n>` y lo mismo ocurre con `<Tab>`.

Si esta api no les parece lo suficientemente buena siempre pueden elegir no migrar sus atajos a lua. Pueden simplemente poner sus atajos en un script `.vim` y llamarlo desde lua.

```lua
vim.cmd 'source ~/.config/nvim/keymap.vim'
```

Para los que quieren huir de vimscript tanto como sea posible puedo recomendarles algunos plugins:

* [astronauta.nvim](https://github.com/tjdevries/astronauta.nvim)
* [mapx.nvim](https://github.com/b0o/mapx.nvim)
* [Vimpeccable](https://github.com/svermeulen/vimpeccable)
* [bex.nvim](https://github.com/bkoropoff/bex.nvim)
* [nest.nvim](https://github.com/LionC/nest.nvim)

No necesitan descargarlos todos. Cada uno tiene una manera diferente de declarar atajos usando lua. Elijan el que tiene el diseño que más les guste.

## Plugin manager

Hablando de plugins. Tal vez quieran usar un plugin manager que este escrito en lua sólo porque sí. Por lo que he visto estas son sus opciones:

* [paq](https://github.com/savq/paq-nvim/)

Es un manejador de plugins rápido y sencillo. No es broma, tiene menos de 300 líneas de código y fue creado para descargar, actualizar y eliminar plugins. Es todo. Si eso es todo lo que necesitan no busquen más, este es el manejador que quieren. 

* [packer](https://github.com/wbthomason/packer.nvim)

Si quieren algo con más funcionalidades `packer` es la alternativa. Aparte de lo básico este manejador ofrece opciones para cargar plugins sólo cuando son necesarios, tiene soporte para declarar y manejar plugins con dependencias, tiene soporte para `luarocks` (este es como un repositorio de paquetes de lua) y también puede manejar "plugins locales". Hace otras cosas pero creo que ya entienden el punto, es bastante completo.

* [vim-packager](https://github.com/kristijanhusak/vim-packager)

No está escrito en lua pero quise agregarlo porque ofrece una interfaz para lua. También quise listarlo porque ofrece más funcionalidades que `paq` pero menos que `packer`, así que si buscan un punto intermedio este puede ser una buena opción.

## Conclusión

Aprendimos cómo usar lua desde vimscript. Sabemos cómo usar vimscript desde lua. Ahora tenemos todas las herramientas para activar, desactivar y modificar cualquier tipo de opción o variable disponible en neovim. Conocemos los métodos para crear nuestros atajos de teclado, sus limitaciones. Sabemos cómo usar un manejador de plugins desde lua ya sea que esté escrito en lua o no. Ya estamos listos.

Para los que quieran ver un ejemplo de la vida real, aquí les dejo un enlace a mi configuración en github: [neovim](https://github.com/VonHeikemen/dotfiles/tree/master/my-configs/neovim).

## Fuentes

* [learn x in y minutes: where X=lua](https://learnxinyminutes.com/docs/lua/)
* [nvim-lua-guide](https://github.com/nanotee/nvim-lua-guide)
* [:help lua-heredoc](https://neovim.io/doc/user/lua.html#:lua-heredoc)
* [:help lua-vim-variables](https://neovim.io/doc/user/lua.html#lua-vim-variables)
* [:help lua-stdlib](https://neovim.io/doc/user/lua.html#lua-stdlib)
* [:help function-list](https://neovim.io/doc/user/usr_41.html#function-list)
* [:help nvim_set_keymap()](https://neovim.io/doc/user/api.html#nvim_set_keymap())
* [curist's bundle.lua](https://github.com/curist/dotvim/blob/98b161f0759d3316fcf6a776d03665d6ab4827ee/bundles.lua)

