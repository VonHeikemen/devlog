+++
title = "Cómo sobrevivir sin cursores múltiples en vim" 
description = "Patrones alternativos que podemos usar en vim en lugar de cursores múltiples"
date = 2022-12-31
lang = "es"
draft = true
[taxonomies]
tags = ["vim", "neovim", "shell"]
[extra]
shared = []
+++

Sí, los cursores múltiples son mágicos. Son convenientes, fácil de usar y están presente en muchos editores modernos. Para bien o para mal, vim no los tiene. Pero está bien, podemos ser felices sin ellos. Bueno... yo puedo y voy decirles cómo.

Vamos a repasar las situaciones donde podríamos usar cursores múltiples y veamos qué alternativas tenemos en vim.

## Reemplazar palabra debajo del cursor

En vim debemos comenzar el proceso buscando la palabra debajo del cursor, para esto presionamos `*`. Luego presionamos `cgn` para reemplazar la próxima ocurrencia. Si queremos repetir la acción presionamos `.`, sí, el punto. Si no queremos reemplazar la ocurrencia actual podemos ignorarla y pasar a la siguiente presionando `n`.

Si hay algo que vim hace extremadamente bien es automatizar las teclas que podemos presionar. Podemos hacer este proceso más rápido si creamos un atajo de teclado.

```vim
nnoremap <leader>j *``cgn
```

Con este atajo podremos utilizar la tecla líder + j para reemplazar la palabra actual, luego podremos navegar entre cada ocurrencia de la palabra y usar `.` para reemplazar el texto.

{{ asciinema(id="qxb6feyI4ieLUlFUwO2kXzV4I") }}

> Ver en [asciinema](https://asciinema.org/a/540516). 

## Cambiar una variable

Puede ser que la palabra que quieren reemplazar es una variable en su código, en este caso sería ideal que el editor sólo reemplace las referencias que son válidas. Aquí la solución puede ser un poco más complicada. Como saben vim no es un IDE, este tipo de funcionalidad no está habilitada por defecto. Pero no es imposible, existen plugins que nos permiten usar servidores LSP. Resulta que renombrar variables es una de esas funcionalidades que podemos obtener con un servidor LSP.

Personalmente uso Neovim y no vim como tal. En mi caso sólo necesito algo como esto.

```vim
lua require('lspconfig').tsserver.setup({})

nnoremap <F2> <cmd>lua vim.lsp.buf.rename()<cr>
```

Aquí uso [nvim-lspconfig](https://github.com/neovim/nvim-lspconfig) para configurar [tsserver](https://github.com/theia-ide/typescript-language-server). Y creo el atajo `<F2>` para renombrar variables.

{{ asciinema(id="217Yk9e9HtmuPi0t8HsNPHOAN") }}

> Ver en [asciinema](https://asciinema.org/a/540539).

Si ustedes usan vim pueden probar estos plugins:

* [vim-lsp](https://github.com/prabirshrestha/vim-lsp)
* [coc.nvim](https://github.com/neoclide/coc.nvim)

## Reemplazar selección

Tal vez el texto que queremos reemplazar no es una palabra, tal vez es una oración o una etiqueta de html. En este caso no tenemos una funcionalidad nativa que nos ayude pero podemos implementar nuestra propia solución.

Lo que haremos será colocar la selección actual en el "registro de búsqueda" de vim.

```vim
let @/=escape(@", '/')
```

Aquí tomamos el registro `"` que contiene la selección más reciente, nos aseguramos de escapar el caracter `/` para que no ocurra un conflicto en nuestra búsqueda, finalmente asignamos el resultado al registro `/`.

Luego debemos borrar toda la selección e ir al modo de inserción. Esto lo logramos con esta secuencia.

```
"_cgn
```

Con `"_` le decimos a vim que queremos usar el registro `_` para nuestra siguiente operación. `cgn` es para reemplazar la primera ocurrencia de una búsqueda.

Si juntamos estas piezas en un atajo tendremos esto.

```vim
xnoremap <leader>j y<cmd>let @/=escape(@", '/')<cr>"_cgn
```

Pero esta solución sólo funciona con búsquedas de una línea. Si por alguna razón quieren reemplazar una selección de múltiples líneas deben hacer esto.

```vim
xnoremap <leader>j y<cmd>substitute(escape(@", '/'), '\n', '\\n', 'g')<cr>"_cgn
```

Aquí usamos la función `substitute` para reemplazar el salto de línea con `\n`, así técnicamente nuestra búsqueda siempre será de una línea.

¿Cómo usamos esto? De la misma forma que usamos el atajo `<leader>j` en la sección "reemplazar palabra debajo del cursor". La diferencia es que en este caso primero debemos ir a modo visual para seleccionar el texto. Luego podremos repetir la acción usando `.` o podemos movernos a la siguiente ocurrencia usando `n`.

## Añadir texto al inicio de una lista

Digamos que tenemos una lista de palabras y queremos convertirla en una lista ordenada en markdown.

Queremos transformar esto.

```
volar
html
cssls
```

En esto.

```
1. volar
1. html
1. cssls
```

En vim tenemos un modo llamado "visual block", cuando estamos en este modo tenemos la posibilidad de agregar texto a cada línea de la selección si vamos a modo de inserción usando `I` o `A`. Luego de insertar el texto en la primera línea vim procede a repetir esta acción en el resto de las líneas seleccionadas.

Vamos a revisar paso a paso lo que debemos hacer.

{{ asciinema(id="tjCikrHM5d1QsoSk00u9iUQMa") }}

> Ver en [asciinema](https://asciinema.org/a/539909).

1. Vamos al primer caracter de la primera línea.
1. Presionamos `Ctrl + v` para ir al modo "visual block".
1. Seleccionamos las líneas que queremos cambiar.
1. Presionamos `I` para colocar el cursor al inicio de la selección.
1. Agregamos el texto que queremos.
1. Presionamos `Esc`.

## Añadir texto al final de una lista

Sí. También podemos modificar el final de la línea. El proceso es casi idéntico, con la diferencia de aquí debemos expandir la selección hacia el final de la línea.

Digamos que tenemos nuestra lista ordenada pero ahora queremos agregar el texto `(is supported)` al final de cada línea.

{{ asciinema(id="eF1Pdf34IY4Sj5PixzjCqPIxz") }}

> Ver en [asciinema](https://asciinema.org/a/539912).

1. Vamos al primer caracter de la primera línea.
1. Presionamos `Ctrl + v` para ir al modo "visual block".
1. Seleccionamos las líneas que queremos cambiar.
1. Expandimos la selección hacia el final de la línea presionando `$`.
1. Presionamos `A` para colocar el cursor al final de la primera línea.
1. Agregamos el texto que queremos.
1. Presionamos `Esc`.

## Repetir un movimientos 

El modo `visual block` puede ser útil en muchas situaciones pero es limitado. Sólo puede agregar texto en un lugar. ¿Qué hacemos cuando queremos hacer modificaciones complejas? La respuesta: `macros`. Un macro es una cadena de texto que describe una secuencia de teclas. Podemos "grabar" un macro y repetirlo tantas veces sea necesario.

¿Cómo usamos un macro? Primero debemos elegir un registro. Algo así como una etiqueta para nuestro macro. Presionamos `q` seguido de la letra de nuestro registro. Luego ejecutamos las modificaciones que queremos. Finalmente presionamos `q` para terminar de grabar el macro. Para repetir el macro debemos presionar `@` seguido de la letra que elegimos en el primer paso.

Vamos con un ejemplo.

Tenemos esta lista.

```
volar
html
cssls
eslint
```

Y queremos convertirlo en una lista ordenada de enlaces.

```
1. [volar](http://localhost/how-to-configure-volar-lsp)
1. [html](http://localhost/how-to-configure-html-lsp)
1. [cssls](http://localhost/how-to-configure-cssls-lsp)
1. [eslint](http://localhost/how-to-configure-eslint-lsp)
```

Noten que aquí necesitamos añadir texto al inicio y final de cada línea. También tenemos que copiar el nombre de cada item y colocarlo en el enlace.

Lo que haremos será convertir el primer item y repetir el proceso usando un macro. Estos son los pasos.

1. Grabamos el macro en el registro `i`. Presionamos `qi`.
1. Modificamos el primer item.
1. Finalizamos la grabación del macro presionando `q` nuevamente.
1. Repetimos el macro tres veces presionando `3@i`.

{{ asciinema(id="85qQ7TtkOclesgMJNBDcKk5BK") }}

> Ver en [asciinema](https://asciinema.org/a/540581).

Cuando aplicamos un macro de esta manera, con repeticiones automáticas, debemos tener en cuenta la posición del cursor. En este caso particular el primer atajo que utilicé al comenzar a grabar el macro fue `0` y terminé presionando `j`. Con esto me aseguro de que en cada repetición el cursor esté en el primer caracter de la línea, y justo antes de terminar el macro muevo el cursor a la línea que está debajo.

### Aplicar un macro en líneas específicas

Otra forma interesante de aplicar un macro es con el comando `g`. Este comando ejecuta una búsqueda y nos da la oportunidad de ejecutar otro comando en las líneas donde hay una ocurrencia. En nuestro caso lo que queremos hacer después de la búsqueda es ejecutar un macro, para eso usamos el comando `normal @i` (donde `i` puede ser cualquier registro).

Digamos que queremos buscar todas las líneas con la palabra `vim` y ejecutar un macro. Podemos hacer algo así.

```vim
:g/vim/normal @i
```

Si quieren asegurarse de que su búsqueda tiene los resultados esperados pueden omitir la sección que va después de la búsqueda.

```vim
:g/vim/
```

Si todo está bien agregamos `normal @i`.

### Aplicar un macro en una selección

No estamos obligados a usar el comando `g`, podemos utilizar el comando `normal @i` sobre una selección ejecutando este comando.

```vim
'<,'>normal @i
```

> Nota: No se preocupen por recordar la sintaxis `'<,'>`, vim lo agregará automáticamente cuando pasamos de modo visual a modo de comando.

Esto ejecutará el macro en `i` sobre cada línea de la selección. Deben tener en cuenta que cada macro comenzará con el cursor al principio de la línea.

### Buscar patrón y ejecutar un macro

Esta es una alternativa al comando `g`. Porque tal vez no se sienten cómodos usando expresiones regulares. Los entiendo. La mayoría de las veces lo que quiero hacer es seleccionar un pedazo de texto y aplicar un macro sobre algunas ocurrencias. Ya vimos cómo buscar una selección, podemos aplicar este conocimiento para manipular la posición del cursor.

¿Recuerdan esto?

```vim
y<cmd>let @/=substitute(escape(@", '/'), '\n', '\\n', 'g')<cr>
```

Esta es la secuencia que usamos para colocar la selección actual en el registro de búsqueda. Todo lo que tenemos que hacer es iniciar la grabación del macro después de esta función. Así.

```vim
gvqi
```

Con `gv` vim vuelve a resaltar la selección anterior, porque recuerden que perdemos la selección luego de presionar `y`. Después con `qi` comienza la grabación del macro en el registro `i`.

Si juntamos todo esto en un atajo obtenemos:

```vim
xnoremap <leader>i y<cmd>let @/=substitute(escape(@", '/'), '\n', '\\n', 'g')<cr>gvqi
```

La historia no termina aquí, ahora tenemos que aplicar el macro en las ocurrencias de nuestra selección. Usaremos `gn` para seleccionar la siguiente ocurrencia y ejecutar el macro.

```vim
nnoremap <F8> gn@i
```

Vamos con un ejemplo de la vida real.

Hace unos meses estaba probando [packer.nvim](https://github.com/wbthomason/packer.nvim), un manejador de plugins. En mi configuración tenía este código.

```lua
require('packer').startup(function(use)
  use({
    'nvim-lualine/lualine.nvim',
    config = function() require('plugins.lualine') end,
  })
  use({
    'akinsho/bufferline.nvim',
    config = function() require('plugins.bufferline') end,
  })
  use({
    'lukas-reineke/indent-blankline.nvim',
    config = function() require('plugins.indent-blankline') end,
  })
end)
```

Me molestaba tener que repetir `function() require...` por cada plugin. Busqué por internet y encontré la manera de reducir el código repetido. Escribí esta función.

```lua
local function load(name)
  return string.format([[pcall(require, 'plugins.%s')]], name)
end
```

Con esta función puedo reducir el código necesario. Ahora utilizar `config` de esta manera.

```lua
config = load('lualine')
```

Lo que queda es reemplazar la propiedad `config` de cada plugin.

{{ asciinema(id="OUlPjhimpPKIDxEqPJgoxHcYt") }}

> Ver en [asciinema](https://asciinema.org/a/OUlPjhimpPKIDxEqPJgoxHcYt).

1. Seleccionamos el patrón que vamos a buscar. En este caso `config =`.
1. Comenzamos a grabar el macro usando el atajo `<leader>i`.
1. Hacemos las operaciones necesarias.
1. Terminamos de grabar el macro presionando `q`.
1. Presionamos `n` para ir al siguiente punto.
1. Repetimos el macro con el atajo `<F8>`.

## Búsqueda y reemplazo

Porque muchas veces no necesitamos herramientas complicadas.

En vim podemos usar el comando `substitute`. Esta es la sintaxis.

```vim
%s/<patrón>/<reemplazo>/g
```

Aquí `%` es el "rango" de la búsqueda, si usamos `%` quiere decir que queremos buscar en todo el archivo. `s` es el comando, no tenemos que escribir el nombre completo `substitute`. `<patrón>` es la expresión regular que queremos buscar. `<reemplazo>` es el texto queremos usar. `g` es un "flag" que le indica a vim que debe buscar en toda la línea. Noten que cada una de estas secciones está separada por `/`, pero podemos usar otros caracteres.

Vamos con un ejemplo. Digamos que queremos cambiar la palabra `config` por `setup`. No necesitamos conocer nada de expresiones regulares, sólo colocamos las palabras en el lugar correcto.

```vim
%s/config/setup/g
```

### El Kirby tuerto

Okey. Si hay algo que les recomiendo aprender de expresiones regulares en vim son los "grupos". Incluso un pequeño truco puede traer muchos beneficios.

¿Qué es eso del kirby tuerto? Es la forma de recordar esta sintaxis.

```
\(.*\)
```

> Este truco lo aprendí de [ThePrimeagen](https://github.com/ThePrimeagen).

Con este patrón podemos capturar un pedazo de texto y crear un "grupo". Una vez que tenemos un grupo podremos utilizar el texto en nuestro texto de reemplazo.

Consideren este comando.

```
%s/`\(.*\)`/[\1](#how-to-configure-\1-lsp)
```

Aquí capturamos todo el texto que se encuentre entre comillas. Luego utilizamos el texto del grupo con el patrón `\1` en el texto de reemplazo. Aquí tienen un demo.

{{ asciinema(id="odouPFLqH5SSHkJuSVhp28yjR") }}

> Ver en [asciinema](https://asciinema.org/a/542501).

En este demo se puede observar en tiempo real el efecto del comando mientras escribo. Tengo entendido que esto es único de neovim (puedo estar equivocado). Pero sí, con este efecto en tiempo real el comando `substitute` puede ser una buena alternativa (en algunos casos) a los cursores múltiples.

Si les preocupa no poder recordar la sintaxis del Kirby tuerto consideren crear un atajo.

```vim
cnoremap <F2> \(.*\)
```

De esta manera sólo tienen que presionar `<F2>` para crear el grupo.

## Conclusión

Ya están listos. Pueden salir al mundo y ser productivos con vim. No puedo garantizar que serán felices pero sobrevivirán. Está bien si todavía consideran que los cursores múltiples es la mejor herramienta. No importa, pueden vivir sin ellos mientras están en vim.

