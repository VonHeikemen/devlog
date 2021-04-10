+++
title = "vim en modo fácil: usando vim como un editor de texto 'convencional'" 
description = "Explorar el 'modo fácil' que provee vim de forma nativa"
date = 2021-04-10
lang = "es"
[taxonomies]
tags = ["vim", "shell", "hoy-aprendi"]
+++

Resulta que vim tiene incorporado de manera nativa un "modo fácil." ¿Quién lo diría? Hoy vamos a aprender de qué se trata, cómo podemos habilitarlo, veremos por qué nadie le presta atención y por último les enseñaré algunas cosas que pueden hacer para mejorarlo.

## Fácil, ¿qué quiere decir eso?

De acuerdo a la documentación, su propósito es ayudar a las personas que no usan vim con frecuencia y que no conocen sus comandos. Hace que vim se comporte como un editor de texto convencional, ya saben, como uno de esos donde podemos insertar texto de manera inmediata.

### ¿Qué nos brinda esta modalidad?

Conveniencia. Esa sería la respuesta. Pero ya en serio, lo primero que notarán es que podremos insertar y editar texto de una manera convencional. Las personas que han entrado en `vim` por accidente sin tener conocimiento de cómo funciona sabrán que ya esto es una gran mejora. También tendremos acceso a atajos de teclado comunes que se encuentran en otros editores, tales como:

* `Ctrl + s`: guarda los cambios en el archivo.
* `Ctrl + z`: revierte el cambio más reciente.
* `Ctrl + y`: rehace el cambio más reciente.
* `Ctrl + c`: copia el texto seleccionado al portapapeles.
* `Ctrl + x`: corta el texto seleccionado, y lo pone en el portapapeles.
* `Ctrl + v`: pega el texto del portapapeles.
* `Ctrl + a`: selecciona todo el texto.

Incluso habilita algunas opciones que hacen la experiencia más intuitiva, como por ejemplo el soporte para seleccionar texto con el ratón.

### ¿Cómo funciona?

Bien, en realidad este "modo fácil" es un grupo configuraciones que se encuentran en un archivo llamado [evim.vim](https://github.com/vim/vim/blob/314dd79cac2adc10304212d1980d23ecf6782cfc/runtime/evim.vim). Si quieren inspeccionarlo pueden ejecutar este comando `vim -R -c 'edit $VIMRUNTIME/evim.vim'` (para salir pulsen `ZZ` o cierren la terminal). Entonces, cuando activamos esta modalidad vim lo que hace es leer ese archivo y ejecutarlo.

### ¿Cómo lo usamos?

Le pasamos a `vim` el argumento `-y`, así `vim -y`. Otra forma de usarlo es invocando el comando `evim` en nuestra terminal.

## El gran fallo en el plan

¿Por qué esta modalidad no tiene más adopción? Porque, irónicamente, esta modalidad hace que sea más difícil salir de vim. Sí, es una tragedia si me preguntan a mí. En fin, para salir de vim tendrían ir al "modo normal" y luego usar el comando para salir. Estos serían los pasos: pulsan `Ctrl + o`, luego `:`, escriben `q` y presionan `Enter`.

*¿Por qué eres así vim?*

Parece que este modo fácil fue diseñado para ser usado en una interfaz gráfica, es decir, vim en una interfaz gráfica. Entonces en teoría deberían cerrar esa interfaz para salir de vim.

## Soluciones

Los expertos leyendo esto deben estar pensando "usen `nano` y ya." Bueno, sí, esa es una solución válida. Pero si quieren darle una oportunidad a `vim` en modo fácil, sigan leyendo (no presten atención a esos expertos).

### La interfaz gráfica

Si lo que necesitamos es una interfaz gráfica, entonces usemos una. Podemos crear alias que use nuestra terminal preferida para abrir vim.

```sh
alias evim='xterm -e vim -y'
```

O tal vez prefieran un script.

```sh
#! /usr/bin/env sh

xterm -e vim -y $@
```

> Pero cambien el comando `xterm` con su terminal favorita.

De esta forma cuando usen el comando `evim` o invoquen su script vim se abrirá en una ventana aparte. Cuando terminen de editar simplemente cierran esa nueva ventana donde está vim.

### La pieza faltante

Por otro lado, si lo que hace falta es un atajo para salir de vim, entonces creemos uno. Podemos hacer esto.

```sh
vim -y -c "inoremap <c-q> <c-o>:q<cr>"
```

Este comando abrirá vim en modo fácil y también creará el atajo `Ctrl + q` para salir. De nuevo, pueden colocar eso en un alias o un script.

### Fácil por defecto

Hay una forma de habilitar este modo sin tener que recurrir al argumento `-y`. En nuestra carpeta de usuario (nuestro "home") podemos crear un archivo llamado `.vimrc`, y colocamos el siguiente contenido.

```vim
" habilita el 'modo fácil'
source $VIMRUNTIME/evim.vim

" Ctrl + q para salir de vim
inoremap <c-q> <c-o>:q<cr>
```

Una vez que tenemos nuestra configuración al invocar `vim` (sólo `vim`) ya estaremos en el modo fácil, y lo más importante podremos salir fácilmente usando `Ctrl + q`.

Esto es lo que yo recomendaría para las personas que no tienen ninguna intención de usar vim de manera frecuente (o incluso si nunca quieren usarlo). Incluso si entran en `vim` por accidente podrán usarlo como un editor convencional o como mínimo podrán salir de él rápidamente.

#### Ya que estamos aquí

Ahora si así lo desean, pueden mejorar un poco su configuración. En el archivo `.vimrc` pueden colocar algunas de estas configuraciones.

* Evita crear archivos temporales.

```vim
set noswapfile
set nobackup
```

* Resalta la línea donde se encuentra el cursor.

```vim
set cursorline
```

* Activa el resaltado de sintaxis y mejora los colores.

```vim
if (has("termguicolors"))
  set termguicolors
endif

syntax enable
```

* Crear atajos para buscar y reemplazar.

```vim
" Ctrl + f para iniciar una búsqueda
inoremap <c-f> <c-o>/

" Ctl + f (x2) para dejar de resaltar los resultados de una búsqueda
inoremap <c-f>f <c-o>:nohlsearch<cr>
inoremap <c-f><c-f> <c-o>:nohlsearch<cr>

" F3 para ir al resultado anterior
inoremap <F3> <c-o>:normal Nzz<cr>

" F4 para ir al siguiente resultado de la búsqueda
inoremap <F4> <c-o>:normal nzz<cr>

" Ctrl + h para iniciar la búsqueda y reemplazo
inoremap <c-h> <c-o>:%s///gc<Left><Left><Left><Left>
```

Este último atajo se usa de la siguiente manera.

```vim
s/{patrón de busqueda}/{el reemplazo}/gc
```

Ejemplo:

```vim
s/Left/Right/gc
```

Esto significa que `vim` buscará todas las ocurrencias de `Left`, nos preguntará si queremos reemplazarlas, y si le decimos que sí lo reemplazará con el valor `Right`.

## Conclusión

Ahora tienen todas las herramientas necesarias para "obligar" a vim a comportarse como un editor convencional. De ahora en adelante su experiencia con vim no tiene porque ser tan terrible.

## Fuentes

* [:help easy](https://vimhelp.org/starting.txt.html#easy)
* [:help evim-keys](https://vimhelp.org/starting.txt.html#evim-keys)

