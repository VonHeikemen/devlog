+++
title = "Usando Netrw, el navegador de archivos nativo de vim" 
description = "Donde aprenderemos a usar y personalizar Netrw"
date = 2021-01-10
lang = "es"
[taxonomies]
tags = ["vim", "shell"]
+++

¿Sabían que vim tiene un navegador de archivos? Sí, es un plugin que ya viene instalado con vim. Se llama Netrw... y no es muy popular, al menos no en comparación con otros plugins como NERDtree. La razón para esto es: 1) no es muy intuitivo. 2) tiene algunas limitaciones que son molestas. 3) se ve feo. Hoy vamos a aprender a usarlo y en el proceso también veremos cómo sobrepasar algunas de sus limitaciones y convertirlo en un navegador de archivos más intuitivo.

## Presentación

Para empezar a conocer Netrw podemos intentar abrir un directorio usando vim (algo así: `vim .`). Asumiendo que su `vimrc` no está fuertemente configurado Netrw debería verse así.

![netrw en pantalla completa](https://res.cloudinary.com/vonheikemen/image/upload/v1609938838/devlog/using-vim-netrw/netrw-full-screen_2021-01-06_09-13-11.png)

Lo primero que vemos es un banner que tiene información sobre el directorio. Esto es lo que nos muestra.

- Información de Netrw, nombre y versión (v156).
- La ruta del directorio. 
- El criterio que usa para ordenar los archivos y directorios, en este caso es por nombre.
- Otro criterio de orden. Esta vez es una secuencia que usará para dar prioridad a los archivos de acuerdo al sufijo.
- "Ayuda rápida". Aquí se listan algunas pistas sobre funcionalidades que ofrece Netrw.

Lo curioso de este banner es que pueden interactuar con algunas de sus "opciones". Por ejemplo, si colocan el cursor sobre la línea donde dice "sorted" y presionan `Enter` verán cómo cambia el orden de los archivos según otros criterios. Podemos cambiar el order según la última actualización, el tamaño o extensión del archivo. En la ayuda rápida, al presionar `Enter`, te mostrará distintos atajos para tareas comunes.

Luego del banner tenemos nuestros archivos y directorios. `../` representa el directorio padre y `./` es el directorio actual. Finalmente, justo debajo tenemos nuestros archivos perfectamente ordenados.

## Uso

Ahora que sabemos cómo luce Netrw vamos a cubrir las funcionalidades básicas.

### Invocación

Nuestra primera parada será el comando `:Explore`. Al invocar este comando sin argumentos, nos mostrará el directorio del archivo que estamos editando actualmente. Si no queremos eso, podemos proveerle la ruta a un directorio de nuestra elección. Ahora, el comportamiento de `:Explore` varía de acuerdo a la configuración de vim, específicamente la opción `hidden`. Si está deshabilitada (así es por defecto) y el archivo que estamos editando no tiene cambios sin guardar `:Explore` abrirá Netrw en pantalla completa. Si resulta que tenemos cambios sin guardar Netrw creará una división horizontal y mostrará el directorio en la ventana superior.

![netrw en media pantalla](https://res.cloudinary.com/vonheikemen/image/upload/v1609951259/devlog/using-vim-netrw/netrw-half-screen_2021-01-06_12-39-50.png)

> Si queremos que la división sea vertical utilizamos el comando `:Explore!`. 

Si tenemos la opción `hidden` habilitada Netrw siempre ocupará toda pantalla, no importa si tenemos cambios sin guardar en un archivo.

Ahora hablemos de las variantes de `:Explore` que tenemos disponibles.

- Hexplore: Creará una división horizontal y mostrará el directorio en la pantalla inferior. La variante con `!` muestra el directorio en el lado opuesto.

- Vexplore: Creará una división vertical y mostrará el directorio a la izquierda. La variante con `!` muestra el directorio en el lado opuesto.

- Sexplore: Creará una división horizontal y mostrará el directorio en la pantalla superior. La variante con `!` crea una división vertical y muestra el directorio del lado izquierdo.

- Texplore: Creará un nuevo tabpage para mostrar el directorio.

- Lexplore: Funciona de una manera similar a `Vexplore` con la diferencia de que cuando abren un archivo este aparecerá en la ventana de donde abrieron Netrw. También tiene la particularidad de poder abrir y cerrar una ventana de Netrw. Pueden verlo en acción en este demo.

{{ asciinema(id="Fa9y0AieDUImMHZjUbjKzjlwn") }}

> Ver en [asciinema](https://asciinema.org/a/Fa9y0AieDUImMHZjUbjKzjlwn).

### Navegando

Si queremos movernos entre directorios y archivos estos son los atajos que necesitamos conocer:

- `Enter`: Entra en un directorio o abre un archivo.
- `-`: Regresa al directorio padre.
- `u`: Regresa al directorio anterior (en el historial).
- `gb`: Salta al directorio más reciente en el registro de "Bookmarks". Para crear un bookmark se utiliza `mb`.

Vamos a repasar. Si quieren ir "hacia abajo" un directorio presionan `Enter`. Si quieren ir "hacia arriba" presionan `-`. Si quieren ir "hacia atrás", `u`. Y si quieren "saltar" a un directorio cualquiera, primero deben marcar su destino con un bookmark (usando `mb`) y luego pueden usar `gb` para ir a ese directorio.

### Operaciones sobre archivos

Ahora vamos a darle un vistazo a los atajos involucrados en las tareas más comunes cuando queremos gestionar nuestros archivos.

- `p`: Abre una vista previa del archivo.

- `<C-w>z`: `Ctrl + w` y luego `z`. Cierra la ventana de vista previa.

- `gh`: Habilita y deshabilita la visualización de archivos ocultos.

- `%`: Crea un archivo. Bueno... en realidad no crea nada, te da la oportunidad de crearlo. Esto es lo que pasa, cuando se presiona `%` vim pregunta por el nombre del archivo. Luego de que ingresas el nombre del archivo, te deja editarlo (pero aún no existe). Después de que editas el contenido y lo guardas (usando el comando `:write`) el archivo es creado.

- `R`: Es para renombrar un archivo.

- `mt`: Asigna el "directorio destino" para las acciones de copiar y mover.

- `mf`: Marca un archivo o directorio. Esto es importante porque las otras operaciones dependen de estas marcas. Si quieren copiar, mover o eliminar un archivo, este debe estar marcado.

- `mc`: Copia los archivos marcados en el directorio destino.

- `mm`: Mueve los archivos marcados al directorio destino.

- `mx`: Ejecuta un comando externo sobre los archivos marcados.

- `D`: Elimina un archivo o un directorio vacío. vim no nos dejará eliminar directorios con archivos dentro. Más adelante les diré cómo sobrepasar esta restricción.

- `d`: Crea un directorio.

### Ejecutando una operación sobre varios archivos

Después de leer sobre los atajos anteriores tal vez se estén preguntando cuáles son los pasos a seguir cuando queremos copiar o mover un archivo. Vamos a hacer un ejemplo moviendo un par de archivos a un directorio.

Este es un proceso de tres pasos:

- Asignar el directorio destino.

- Marcar los archivos que queremos mover.

- Ejecutar el comando correspondiente, en nuestro caso: `mm`.

Aquí les dejo un demo para ilustrar el proceso.

{{ asciinema(id="YkvegGilPQpSbABrOcFZYhN7W") }}

> Ver en [asciinema](https://asciinema.org/a/YkvegGilPQpSbABrOcFZYhN7W).

Esto es lo que pasa:

- *00:00-00:17* Se usa el comando `:Explore` para abrir Netrw. Y luego se verifica el contenido del directorio `test dir`.

- *00:18* Se asigna `test dir` como directorio destino. Noten que ahora el banner se actualiza y nos indica el directorio destino. Se agrega esta línea.

```
"   Copy/Move Tgt: /tmp/vim/test dir/ (local)
```

- *00:20-00:25* Se marcan los archivos `a-file.txt` y `another-file.txt`. Para indicar que están marcados los nombres ahora aparecen en negrita.

- *00:25-00:27* Se presiona `mm` para mover los archivos, y estos desaparecen de la pantalla actual.

- *00:29* Se puede observar que los archivos marcados ahora están dentro de `test dir`.

Y ese básicamente el proceso para las operaciones de copiar y mover. Para ejecutar comandos externos y eliminar es casi igual, con la diferencia de que no necesitamos un directorio destino (un paso menos).

## Limitaciones de Netrw

- Al mover archivos.

Esto ocurre en linux y tal vez en macOS sea igual. En nuestro ejemplo anterior mover `a-file.txt` al directorio `test dir` funciona de maravilla, pero si quieren devolver `a-file.txt` al directorio padre se van a encontrar con este error. 

```
**error** (netrw) tried using g:netrw_localmovecmd<mv>; it doesn't work!
```

> Esto no ocurre cuando van a copiar un archivo.

Hasta donde pude indagar esto tiene que ver con el directorio donde abrimos Netrw y el directorio que estamos visualizando, cuando no son iguales ocurre este error. Para arreglarlo pueden cambiar la variable global `g:netrw_keepdir` a cero.

```vim
let g:netrw_keepdir = 0
```

- Al ejecutar operaciones en archivos marcados.

Cuando van a ejecutar un comando sobre archivos marcados este sólo afecta a los que están listados en la pantalla actual.

Digamos que tenemos esta estructura en nuestro directorio.

```
vim
├── mini-plugins
│   ├── better-netrw.vim
│   ├── guess-indentation.vim
│   └── project-buffers.vim
├── test dir
│   ├── a-file.txt *
│   ├── another-file.txt *
│   └── text.txt
├── custom-commands.vim
└── init.vim *
```

Los archivos que tienen un `*` son los que tenemos marcados. Si nos encontramos en el directorio `vim` y hacemos el proceso para mover los archivos nos daremos cuenta que sólo se ha movido `init.vim` al directorio destino. En teoría esto es bueno porque siempre tendremos a la vista los archivos sobre los cuales estamos operando.

- Netrw no permite borrar directorios con archivos usando `D`.

La respuesta a esto es: usar un comando externo. Por suerte Netrw tiene algo que puede ayudarnos. Si prestaron atención en secciones pasadas sabrán que el atajo `mx` nos deja hacer precisamente eso. Pueden verlo en acción.

{{ asciinema(id="YbscBomZSa752kXnEASUnaxlx") }}

> Ver en [asciinema](https://asciinema.org/a/YbscBomZSa752kXnEASUnaxlx).

Entonces, la solución: marcar los directorios con `mf`, usar `mx` y escribir el comando que necesitamos (en el demo `rm -r`). Es todo. ¿Pero podemos hacer este proceso más conveniente? Sí, y vamos discutirlo en la siguiente sección.

## Personalización

Si han decidido darle una oportunidad a Netrw tal vez quieran modificarlo un poco para hacer que sea más agradable.

### Configuración recomendada

Sincronizar directorio actual y el directorio que está mostrando Netrw. Esto ayuda con el error cuando se intenta mover archivos.

```vim
let g:netrw_keepdir = 0
```

Configurar el porcentaje que ocupa Netrw cuando crea una división. 30% esta bien.

```vim
let g:netrw_winsize = 30
```

Ocultar el banner (Si quieren). Para mostrarlo temporalmente sólo deben presionar `I` en Netrw.

```vim
let g:netrw_banner = 0
```

Ocultar archivos que comiencen con un punto.

```vim
let g:netrw_list_hide = '\(^\|\s\s\)\zs\.\S\+'
```

Modificar el comando para copiar archivos. Para permitir copiar directorios de manera recursiva.

```vim
let g:netrw_localcopydircmd = 'cp -r'
```

Resaltar los archivos marcados de la misma manera en la que se resalta las coincidencias en una búsqueda.

```vim
hi! link netrwMarkFile Search
```

> Esta es la manera más fácil que se me ocurrió para resaltar más las marcas. Esto puede crear confusión si buscan una cadena de texto y tienen archivos marcados. Si desean aplicar otros colores investiguen sobre el comando `highlight`.

### Keymaps

Ahora que Netrw se ve mejor, hagamos que sea más fácil de usar. 

#### Invocando Netrw

Podemos usar el comando `:Lexplore` en un atajo para mostrarlo y ocultarlo cuando queramos.

```vim
nnoremap <leader>dd :Lexplore %:p:h<CR>
nnoremap <Leader>da :Lexplore<CR>
```

- `Leader dd`: Será para abrir Netrw en el directorio del archivo actual.
- `Leader da`: Lo abrirá en el directorio de trabajo actual.

#### Navegación y visualización

Ahora bien, desafortunadamente no hay una manera directa de crear atajos exclusivos para Netrw. Es posible hacerlo pero necesitamos hacer algunas cosas primero.

Netrw es plugin que define su propio tipo de archivo, así que vamos a usar eso para crear una función que pueda asignar los atajos que nosotros queramos. Para ser específico, lo que haremos será crear un `autocommand` que llame una función cada vez que vim abra un archivo de tipo `netrw`.

```vim
function! NetrwMapping()
endfunction

augroup netrw_mapping
  autocmd!
  autocmd filetype netrw call NetrwMapping()
augroup END
```

Con esto en nuestra configuración lo que queda por hacer es colocar nuestros atajos en la función `NetrwMapping`. Así.

```vim
function! NetrwMapping()
  nmap <buffer> H u
  nmap <buffer> h -^
  nmap <buffer> l <CR>

  nmap <buffer> . gh
  nmap <buffer> P <C-w>z

  nmap <buffer> L <CR>:Lexplore<CR>
  nmap <buffer> <Leader>dd :Lexplore<CR>
endfunction
```

Ya que no tenemos acceso a los comandos que usa Netrw internamente (al menos no todos), usamos `nmap` para hacer una especie de redirección. Por ejemplo, aquí `H` será equivalente a presionar `u`, que a su vez activará la función que queremos. Entonces tenemos que:

- `H`: Ahora será para "ir atrás" en el historial.
- `h`: Será para "ir arriba" un directorio.
- `l`: Será para abrir un directorio o archivo.
- `.`: Controlará si se muestran los archivos ocultos.
- `P`: Para ocultar la ventana de la vista previa.
- `L`: Abrirá un archivo y cerrará la ventana de Netrw.
- `Leader dd`: Sólo cerrará la ventana de Netrw.

Con sólo agregar estos atajos (más la configuración recomendada) Netrw se convierte en un navegador decente. Pero esperen, aún podemos mejorar.

#### Marcas

Podemos encontrar una manera más cómoda de manejar las marcas de los archivos. Yo sugiero usar `<Tab>` para esto.

```vim
nmap <buffer> <TAB> mf
nmap <buffer> <S-TAB> mF
nmap <buffer> <Leader><TAB> mu
```

- `Tab`: marcará (o quitará una marca) de un archivo o directorio.
- `Shift Tab`: Quitará las marcas de todos los archivos listados en la pantalla actual.
- `Leader Tab`: Quitará todas las marcas.

#### Gestión de archivos

Ya que hay un montón de comandos relacionados con los archivos vamos a usar la letra `f` como prefijo para agruparlos.

```vim
nmap <buffer> ff %:w<CR>:buffer #<CR>
nmap <buffer> fe R
nmap <buffer> fc mc
nmap <buffer> fC mtmc
nmap <buffer> fx mm
nmap <buffer> fX mtmm
nmap <buffer> f; mx
```

- `ff`: Para crear un archivo. Crearlo de verdad. En esta ocasión, después de `%` usamos `:w<CR>` para guardar el archivo vacío y `:buffer #<CR>` para volver a Netrw.
- `fe`: Para renombrar un archivo.
- `fc`: Para copiar los archivos marcados.
- `fC`: Lo usaremos para ahorranos un paso. Con esto podremos situarnos sobre un directorio y copiar los archivos marcados directamente.
- `fx`: Para mover los archivos marcados.
- `fX`: Tiene la misma mecánica que `fC` pero esta vez será para mover archivos.
- `f;`: Lo usaremos para ejecutar comandos externos sobre los archivos marcados. 

Aún podemos hacer un par de cosas, si no les importa usar algunas variables internas de Netrw.

Ver la lista de archivos marcados.

```vim
nmap <buffer> fl :echo join(netrw#Expose("netrwmarkfilelist"), "\n")<CR>
```

Mostrar el directorio destino, en caso de que queramos evitar mostrar el banner.

```vim
nmap <buffer> fq :echo 'Target:' . netrw#Expose("netrwmftgt")<CR>
```

Ahora podemos usar el atajo anterior en conjunto con `mt` para asignar el directorio destino. De nuevo, esto sólo es útil si queremos evitar a toda costa mostrar el banner.

```vim
nmap <buffer> fd mtfq
```

#### Bookmarks

En el mismo estilo en el que agrupamos la gestión de archivos podemos hacerlo con los "Bookmarks".

```vim
nmap <buffer> bb mb
nmap <buffer> bd mB
nmap <buffer> bl gb
```

- `bb`: Para crear un bookmark.
- `bd`: Para eliminar el bookmark más reciente.
- `bl`: Para saltar al directorio del bookmark más reciente.

#### Eliminando archivos de manera recursiva

Lo último que haremos será "automatizar" el proceso para eliminar un directorio no vacío. Para esto necesitaremos una función.

```vim
function! NetrwRemoveRecursive()
  if &filetype ==# 'netrw'
    cnoremap <buffer> <CR> rm -r<CR>
    normal mu
    normal mf
    
    try
      normal mx
    catch
      echo "Canceled"
    endtry

    cunmap <buffer> <CR>
  endif
endfunction
```

En esta función lo primero que hacemos es asegurarnos de que estamos dentro de un archivo de tipo `netrw`. Lo siguiente que hacemos es preparar el comando para eliminar. Nos aprovechamos del hecho de que vim nos obliga a entrar en modo de comando y creamos un atajo (`<CR>`) que escribirá el comando para eliminar archivos por nosotros. Luego con `normal mu` limpiamos la lista de archivos marcados (no queremos borrar nada por accidente). Marcamos el directorio que está debajo del cursor con `normal mf`. Aquí viene lo gracioso, `normal mx` nos preguntará qué comando queremos ejecutar, es en este punto donde podemos abortar el proceso usando `ctrl + c` o presionar `Enter` lo que causará que vim escriba `rm -r` y ejecute el comando. Por último desechamos el atajo que creamos al principio de la función, porque sería muy inconveniente tenerlo de forma permanente.

¿Y cómo la usamos?

Dentro de `NetrwMapping` creamos otro atajo.

```vim
nmap <buffer> FF :call NetrwRemoveRecursive()<CR>
```

Pueden visualizar todas las opciones y funciones presentadas en este artículo: [aquí](https://gist.github.com/VonHeikemen/fa6f7c7f114bc36326cda2c964cb52c7).

## Conclusión

Netrw puede que no sea el mejor gestor de archivos en el ecosistema de vim pero con un poco de configuración podemos por lo menos convertirlo navegador de archivos intuitivo. Incluso si deciden no adoptar Netrw en su rutina diaria no viene mal aprender lo básico para poder navegar fácilmente entre directorios. Nunca saben cuando estarán varados en servidor remoto usando vim sin sus plugins favoritos a mano. 

## Fuente

[:help netrw](https://vimhelp.org/pi_netrw.txt.html)

