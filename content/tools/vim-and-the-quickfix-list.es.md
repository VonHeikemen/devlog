+++
title = "vim y el quickfix list: saltar a una ubicación, buscar y reemplazar en múltiples archivos,  y otras curiosidades"
description = "Descifrando el misterio del quickfix list de vim"
date = 2021-05-02
lang = "es"
[taxonomies]
tags = ["vim", "shell"]
+++

En esta ocasión cubriremos una funcionalidad avanzada de vim, la lista de cambios rápidos, A.K.A. *the quickfix list*. Aprenderemos cómo usarla para buscar (y reemplazar) un patrón en múltiples archivos y también para saltar rápidamente a la ubicación de un error generado por un comando externo.

## Quickfix list

Es una modalidad especial de vim en donde se muestra una lista de posiciones, es decir filas y columnas en un archivo. El propósito de esta lista es guardar las posiciones que se encuentran en los mensajes de error que genera un compilador. De esta forma uno puede saltar rápidamente a esta ubicación, arreglar el problema y luego ir al siguiente o intentar compilar nuevamente.

Puede que suene como una funcionalidad limitada, pero no teman, como todo en vim hay más de lo que aparenta a simple vista. Esta lista de cambios puede generarse por varios métodos. Para empezar tenemos algunos comandos que pueden generarla automáticamente, entre ellos está `:make`, `:vimgrep` y `:grep`. Adicionalmente, es posible generarla de manera programática usando la funcion `setqflist`, lo que nos da una flexibilidad increíble.

Debo mencionar que cuando hablo del quickfix list me refiero solamente a la lista de posiciones, deben saber que esta lista puede mostrarse de dos formas. Tenemos el `quickfix window` y el `location list`. Estas son básicamente "ventanas" donde se muestra la lista de posiciones. El `quickfix window` es "global", es decir que sólo podemos tener una en la sesión activa de vim. Por otro lado, podemos tener múltiples `location list` asociados a diferentes ventanas en una misma sesión.

## Saltar a una ubicación

Para comenzar vamos a explorar el comando `:make`, es la manera en la que vim invoca un compilador. ¿El nombre les parece familiar? Eso es porque vim asume que estaremos trabajando con la herramienta `make`. Pueden hacer una prueba si lo tienen instalado en su sistema. Crean un archivo llamado `Makefile` y colocan lo siguiente:

```make
.PHONY: test otra

test:
  @echo 'hola'

otra:
  @echo 'otra'
```

> No cubriré en detalle cómo funciona `make` pero si quieren saber cómo utilizarlo para automatizar tareas pueden leer [este artículo](https://vinta.ws/code/use-makefile-as-the-task-runner-for-arbitrary-projects.html) (en inglés).

Al invocar el comando `:make` deberían obtener algo como esto.

```
hola

(1 de 1): hola
```

Por defecto `make` ejecutará la primera "tarea" que vea en nuestro `Makefile`. Bien, pero entonces ¿Cómo hacemos para ejecutar `otra`? Podemos proveer más argumentos al comando, de esta manera `:make [argumento]`. Si intentan ejecutar `:make otra` deberían ver esto.

```
otra

(1 de 1): otra
```

Interesante, pero esos comandos no generan ningún error. Después de ver estos mensajes no ocurre nada. 

Ya que vim sabe cómo "leer" los errores del compilador `gcc` vamos a ver un ejemplo usando el lenguaje C.

### Un ejemplo en C

Creamos un archivo llamado `hello.c`, en él ponemos el siguiente contenido.

```c
#include <stdio.h>

int main() {
   printf("Hola, Mundo!\n");
   return 0;
}
```

Aquí no hay ningún error. Podemos compilar y ejecutarlo tranquilamente usando `make`. Así que el siguiente paso será crear un `Makefile`.

```make
.PHONY: run-hello

run-hello:
  gcc -Wall -o hello hello.c
  ./hello
  rm ./hello 
```

Todo está en su lugar. Ahora podemos ejecutar el comando `:make --silent run-hello`. Si todo sale bien vim les mostrará su `hola mundo`.

```
Hola, Mundo!

(1 de 1): Hola, Mundo!
```

Si introducimos un error esto es lo que deberíamos obtener.

```
hello.c: In function ‘main’:
hello.c:4:28: error: expected ‘;’ before ‘return’
    printf("Hola, Mundo!\n")
                            ^
                            ;
    return 0;
    ~~~~~~
make: *** [Makefile:7: run-hello] Error 1

(2 de 8): error: expected ‘;’ before ‘return’
```

Luego de ver el mensaje notarán que vim los llevó automáticamente a la ubicación del error (¿no es genial?). Para visualizar el `quickfix window` tenemos que ejecutar el comando `:copen`, el cual debería mostrarle esto.

```
|| hello.c: In function ‘main’:
hello.c|4 col 28| error: expected ‘;’ before ‘return’                  
||     printf("Hola, Mundo!\n")
||                             ^
||                             ;
||     return 0;
||     ~~~~~~
make: *** [Makefile|7| run-hello] Error 1
```

> Para cerrar el `quickfix window` se utiliza el comando `:cclose`.

Presten atención a esta línea. 

```
hello.c|4 col 28| error: expected ‘;’ before ‘return’
```

Aquí vim nos está señalando el archivo, la línea y la columna donde `gcc` dice que está nuestro error, así como también el mensaje de error. Es en este punto donde arreglamos el desperfecto e intentamos compilar nuevamente. En su mayoría ese sería el flujo de trabajo que queremos cuando usamos el quickfix list.

Muy bonito, ¿y qué pasa si no usamos `gcc`? ¿Y si normalmente trabajamos con `Node.js`? ¿vim podría manejar eso también? Sí, con algo de ayuda.

### errorformat

Si empiezan a experimentar con diferentes compiladores e interpretes se darán cuenta rápidamente que vim no puede leer de manera apropiada todos los tipos de errores. Para sobrepasar esta limitación vim nos ofrece algo llamado `errorformat`, una variable con la cual podemos especificar el formato del error que genera un comando externo, de esta forma vim puede reconocerlo cuando se encuentre en el quickfix list.

Ahora intentemos hacer que vim entienda los errores que genera `node`. Vamos a crear un archivo llamado `greeting.js`, en él vamos a tener un "hola mundo" con un error.

```js
console.log(greeting);
const greeting = 'Hola, Mundo!';
```

En nuestro habitual `Makefile` creamos otra tarea.

```make
run-greeting:
  node ./greeting.js
```

Si ejecutamos `:make --silent run-greeting` obtendremos este resultado.

```
console.log(greeting);
            ^

ReferenceError: Cannot access 'greeting' before initialization
    at Object.<anonymous> (/tmp/test/greeting.js:1:13)
    at Module._compile (internal/modules/cjs/loader.js:1063:30)
    at Object.Module._extensions..js (internal/modules/cjs/loader.js:1092:10)
    at Module.load (internal/modules/cjs/loader.js:928:32)
    at Function.Module._load (internal/modules/cjs/loader.js:769:14)
    at Function.executeUserEntryPoint [as runMain] (internal/modules/run_main.js:72:12)
    at internal/main/run_main_module.js:17:47
make: *** [Makefile:4: run-greeting] Error 1
```

vim intentará llevarnos al lugar donde está el error pero probablemente falle miserablemente. En mi caso particular, me lleva a un archivo que no existe.

Para arreglar esto tenemos que decirle a vim cómo leer ese mensaje. Una manera de hacerlo sería esta.

```vim
set errorformat=%E%.%#ReferenceError:\ %m,%Z%.%#%at\ Object.<anonymous>\ (%f:%l:%c)
```

Lo que hacemos es especificar qué "forma" tiene cada linea en el mensaje error. Cada linea tiene su propio formato y debemos separarlos con una coma. Entonces tenemos que.

```
%E%.%#ReferenceError:\ %m
```

Y

```
%Z%.%#%at\ Object.<anonymous>\ (%f:%l:%c)
```

Son dos expresiones distintas que le dicen a vim cómo leer el error. Le estoy diciendo dónde puede encontrar el tipo de error (`ReferenceError`) y dónde está la información sobre la ruta y la ubicación del error. Ya que esas dos piezas de información se encuentran en dos lineas diferentes debo tener dos expresiones separadas por una coma.

Para mejorar la legibilidad también podemos escribirlo de esta manera.

```vim
set errorformat=%E%.%#ReferenceError:\ %m
set errorformat+=%Z%.%#%at\ Object.<anonymous>\ (%f:%l:%c)
```

Noten que así no tenemos que colocar la coma al final de cada expresión. Pero aún tenemos que colocar un `\` antes de cada caracter que vim considere especial (como un espacio en blanco) para no generar un conflicto entre la sintaxis de vim y el formato del error. Aún hay otra forma que también podemos usar.

```vim
let &errorformat = 
  \ '%E%.%#ReferenceError: %m,' .
  \ '%Z%.%#at Object.<anonymous> (%f:%l:%c)'
```

Al usar `let` tenemos la posibilidad de crear nuestro formato usando una cadena de texto, sin generar conflictos entre vim y el `errorformat`. Para mejorar la legibilidad he puesto cada expresión en su propia linea tomando ventaja del operador `.` para concatenar cadenas de texto.

Ahora al intentar ejecutar `:make --silent run-greeting` vim debería llevarnos al lugar exacto donde `node` dice que está el error. Y en el `quickfix window` debería mostrarnos esto.

```
|| /tmp/test/greeting.js:1
|| console.log(greeting);
||             ^
|| 
greeting.js|1 col 13 error| Cannot access 'greeting' before initialization
||     at Module._compile (internal/modules/cjs/loader.js:1063:30)
||     at Object.Module._extensions..js (internal/modules/cjs/loader.js:1092:10)
||     at Module.load (internal/modules/cjs/loader.js:928:32)
||     at Function.Module._load (internal/modules/cjs/loader.js:769:14)
||     at Function.executeUserEntryPoint [as runMain] (internal/modules/run_main.js:72:12)
||     at internal/main/run_main_module.js:17:47
|| make: *** [Makefile:4: run-greeting] Error 1
```

Noten que ya no aparece `ReferenceError`, y tampoco la línea que va debajo. Esa parte específica del mensaje ha sido reemplazada por esto.

```
greeting.js|1 col 13 error| Cannot access 'greeting' before initialization
```

Si les aparece así es porque el formato en el `errorformat` funcionó.

Pero tenemos un problema, y es que esas dos expresiones sólo funcionarían con el tipo de error `ReferenceError`. Si queremos abarcar otros tipos de errores haríamos algo como esto.

```vim
let &errorformat = 
  \ '%E%.%#AssertionError %m,' .
  \ '%E%.%#TypeError: %m,' .
  \ '%E%.%#ReferenceError: %m,' .
  \ '%E%.%#SyntaxError: %m,' .
  \ '%E%.%#RangeError: %m,' .
  \ '%Z%.%#at Object.<anonymous> (%f:%l:%c),' .
  \ '%-G%.%#'
```

#### Tokens especiales

Ahora probablemente quieren saber sobre la sintaxis que he usado en este formato. Específicamente, qué son todos esas cosas con `%`. Vamos a dar un repaso sobre esos "tokens" especiales.

- `%f`: Indica la ubicación de la ruta del archivo donde se encuentra el error.

- `%l`: Indica la línea donde se encuentra el error.

- `%c`: Indica la columna donde se encuentra el error.

- `%m`: Se usa para indicar la ubicación del contenido del mensaje. En nuestro ejemplo lo usamos para asignar la cadena de texto que viene inmediatamente después del tipo de error. 

- `%E`: Indica el inicio de un mensaje de error que abarca múltiples líneas. La letra `E` le dice a vim que esto es un error. También podemos señalar mensajes de advertencia (`%W`), mensajes informativos (`%I`) ó generales (`%G`).

- `%Z`: Indica que es la última línea del mensaje. 

- `%.%#`: Esto es un comodín, basícamente coincide con todo. En otras palabras, significa "cualquier cosa." Por ejemplo, la expresión `%Z%.%#at Object.<anonymous> (%f:%l:%c)` puede leerse de la siguiente forma: La última línea de este mensaje (`%Z`) puede comenzar con cualquier cosa (`%.%#`) seguido de `at Object.<anonymous>` y entre paréntesis el archivo (`%f`), un número de línea (`%l`) y la columna (`%c`).

- `%-`: Le indica a vim que no debe incluir este fragmento en el `quickfix window`. Entonces la expresión `%-G` se puede leer como "no incluyas este mensaje." En nuestro ejemplo usamos `%-G%.%#` que basícamente le dice a vim "ignora todo lo demás." Ya que `%.%#` es una expresión que coincide con todo lo colocamos de último.

Si quieren saber con más detalle lo que pueden hacer con `errorformat` pueden ejecutar el comando `:help errorformat`.

### makeprg

Ya saben que con un poco de esfuerzo podemos hacer que vim lea cualquier tipo de error pero la configuración actual aún nos tiene atados a `make`. No tiene que ser así. Podemos cambiar el comando externo que vim ejecuta cuando invocamos `:make`.

Digamos que queremos usar `node` directamente, para lograr nuestro objetivo sólo tenemos que modificar la variable `makeprg`.

```vim
set makeprg=node
```

O también.

```vim
let &makeprg = 'node'
```

Ahora en lugar de usar `:make --silet run-greeting` podemos ejecutar `:make ./greeting.js` ó `:make %` si estamos editando `greeting.js`.

## Buscar

El quickfix list no solamente es útil para saltar a un error, también puede ser un mecanismo conveniente con el cual podemos explorar un proyecto. Mencioné antes que vim nos ofrece los comandos `:grep` y `:vimgrep`, ellos pueden crear un quickfix list con los resultados de una búsqueda.

### vimgrep

Con este comando podremos hacer uso del mecanismo de búsqueda que viene incorporado en vim. Está diseñado para ser similar a la herramienta [grep](https://linux.die.net/man/1/grep), con la pequeña diferencia que usa el "intérprete de expresiones regulares" de vim. En otras palabras, con `:vimgrep` podemos buscar un patrón (una expresión regular) en múltiples archivos. Se usa de la siguiente manera:

```vim
:vimgrep /<patrón>/ <archivos>
```

Los `/` no son estrictamente obligatorios, pero es una buena idea incluirlos si el patrón que están buscando contiene algún caracter que cause conflicto con la sintaxis de vim. Por ejemplo:

```vim
:vimgrep /create table/ db/**/*.sql
```

Aquí estaríamos realizando una búsqueda del patrón `create table` en un directorio llamado `db`, y seleccionando específicamente todos los archivos que terminen con la extensión `.sql`.

Los delimitadores del patrón de búsqueda no tienen que ser `/`. Pueden ser cualquier caracter que vim no considere un "identificador," pueden encontrar más detalles en la documentación de vim `:help isident`. Esto puede resultar útil si `/` ya se encuentra en nuestro patrón. Si estuviéramos buscando una ruta en nuestro código podríamos hacer algo así.

```
:vimgrep #/home/user/code# scripts/*.sh
```

¿Pero qué pasa si queremos ignorar algún directorio de nuestra búsqueda? Hay algo que pueden hacer. Para empezar, podrían intentar usar la opción `wildignore`. Algo así.

```vim
:set wildignore=*/cache/*,*/tmp/*
```

Hasta donde tengo entendido esto le dice a vim que ignore los directorios `cache` y `tmp` de cualquier expansión de ruta o autocompletado. Por ejemplo, al buscar, si usamos un patrón como este `**/*.js` vim no incluirá ningún directorio que contenga en su ruta `/cache/` o `/tmp/`. Entonces, `:vimgrep` no buscará en esos directorios porque técnicamente nunca los recibe como parámetro. Quiere decir también que esta opción no es específica para `:vimgrep`, también puede afectar otros comandos.

Algunas preguntas en *stackoverflow* sugieren que este método puede que no funcione. En ese caso podríamos intentar con una expresión encerrada en \` (acento grave), las cuales se usan para invocar un comando en nuestro "shell." Podemos hacer cosas interesantes como buscar sólo en los archivos que son manejados por `git`.

```vim
:vimgrep /function/ `git ls-files`
```

En el caso de `:vimgrep` cualquier comando externo que nos devuelva una lista de archivos es válido.

### grep

Ya habrán notado que vim por defecto asume que tenemos a nuestra disposición ciertas herramientas. Vimos que por ejemplo asume que tenemos instalado `make`. Ahora vamos a explorar el comando `:grep`, que es la manera de vim de integrarse con la herramienta de búsqueda `grep`. En su mayoría funciona igual que `:vimgrep`  con la diferencia de que todos los argumentos deben ser compatibles con la sintaxis de `grep`. Por ejemplo.

```vim
:grep -r "create table" db/**/*.sql
```

Noten que no estoy usando `/` sino comillas dobles y le añado el parámetro `-r` (para habilitar la búsqueda recursiva en directorios), esto es porque el comando `:grep` lo que hace es invocar un "shell" no interactivo para ejecutar el programa `grep` y le pasa todos los argumentos tal cual como los escribimos.

Ahora surge la pregunta ¿Cuándo debemos usar `:grep` por encima de `:vimgrep`? Resulta que `:grep` es más rápido y eficiente que `:vimgrep`. Entonces, `:grep` es la mejor opción si tienen que buscar en muchos archivos o archivos que son tienen mucho contenido. 

Por otro lado podríamos preguntarnos ¿Qué ventajas tiene `:vimgrep` sobre `:grep`? No mucho diría yo. Lo primero sería que la sintaxis para las expresiones regulares es la misma que usa vim, si están familiarizados con esa sintaxis pueden tomar ventaja de sus bondades. También, `:vimgrep` funciona de manera consistente en todas las plataformas.

¿Qué pasa si nos encontramos en una plataforma que no tiene instalado `grep`? `:grep` con su configuración por defecto no funcionaría. Pero al igual que `:make`, con `:grep` podemos modificar el comando externo que vim ejecuta, usando la variable `grepprg`. Digamos que no tenemos `grep` sino [ripgrep](https://github.com/BurntSushi/ripgrep) instalado en nuestro sistema, en ese caso podremos hacer esto.

```vim
set grepprg=rg\ --vimgrep\ --smart-case\ --follow
```

Con esto vim usará `rg` con los parámetros especificados para realizar las búsquedas cuando nosotros usemos `:grep`.

### Invitado especial: FZF

La última herramienta de búsqueda que voy a mencionar es [fzf.vim](https://github.com/junegunn/fzf.vim). Este es un plugin que nos proporciona una especie de interfaz en la que podemos realizar búsquedas de manera interactiva. No entraré en detalle de cómo funciona o se usa. Sólo les diré un tip que yo descubrí mucho tiempo después de empezar a usarlo.

Curiosamente, también podemos poblar el quickfix list usando FZF, específicamente con los comandos `Rg` y `Ag`. Después de realizar una búsqueda y se encuentren con su lista de resultados, pueden seleccionar múltiples items (con la tecla `tab` para seleccionar uno o `Alt + a` para seleccionar todos) y presionar `enter`. Eso es todo. Después el quickfix list debería tener todo lo que ustedes seleccionaron. Esto resulta muy útil cuando quieren ejecutar comandos sólo en unas partes del código usando `:cdo`.

## Reemplazar

Hablando de `:cdo`, voy a mostrarles una de las cosas más útiles que podemos hacer ese comando: buscar y reemplazar un patrón en múltiples archivos. Una tarea bastante común, y seguramente se han preguntado cómo pueden hacerlo usando vim. La respuesta a eso es el quickfix list y el comando `:cdo`.

En el caso más simple donde queremos reemplazar cada instancia de un patrón esto es lo que hacemos:

* Paso 1:

Buscamos nuestro patrón usando `:grep`, `:vimgrep` o cualquier otro comando que pueda generar un quickfix list.

* Paso 2 (opcional):

Abrimos el quickfix window usando el comando `:copen`.

* Paso 3:

Usamos el comando `:cdo` para ejecutar una acción sobre cada item en el quickfix list. En este caso la acción que queremos hacer es sustituir y en vim lo hacemos con esta sintaxis `s/{patrón}/{reemplazo}`. 

Imaginemos que en nuestro quickfix list tenemos los resultados de esta búsqueda.

```vim
:vimgrep node **/*.js
```

Una vez que tengamos los resultados lo que haremos será reemplazar `node` con `deno` y para eso usamos este comando.

```vim
:cdo s/node/deno/ | update
```

Lo que va a pasar es que vim ejecutará el comando `s/node/deno | update` en cada item de nuestro quickfix list. En realidad `:cdo` puede ejecutar cualquier comando válido de vim. En nuestro ejemplo en concreto lo usamos para hacer dos cosas, reemplazar el texto `node` y guardar los cambios en el archivo.

### Caso avanzado

Digamos que tenemos los resultados de una búsqueda y por supuesto queremos hacer una substitución, pero también queremos filtrar los resultados. En otras palabras queremos reemplazar un patrón pero no queremos reemplazar todas las instancias de ese patrón. ¿Cómo haríamos? En ese caso lo que podemos hacer es modificar el quickfix list para que refleje sólo los resultados que queremos reemplazar.

Este proceso requiere algo de esfuerzo por nuestra parte. Para empezar necesitamos decirle a vim cómo leer el quickfix list, para que podemos crear versiones modificadas de un quickfix list que existente. Para lograr nuestro objetivo necesitamos agregrar un patrón al `errorformat`. Entonces en su `.vimrc` pueden colocar algo como esto.

```vim
set errorformat+=%f\|%l\ col\ %c\|%m
```

Aún con esto no pueden modificar el quickfix list, vim no los dejará. Cuando quieran eliminar algún resultado deben hacer que el buffer donde está el quickfix list sea modificable. Deben ejecutar este comando.

```vim
:set modifiable
```

Luego eliminan los items que ustedes quieran, pero no ejecuten `:cdo` todavía. Tienen que asegurarse de crear un quickfix list basado en los resultados que quieren. Entonces, luego de hacer sus modificaciones en el buffer ejecutan este comando.

```vim
:cgetbuffer
```

Lo siguiente es asegurarse que están usando la versión actualizada del quickfix list abriendo y cerrando el quickfix window. Creo que este paso es opcional, pero a mí no me gusta arriesgarme. Ejecutan `:cclose` y luego `:copen`.

Por último ejecutan el comando para sustituir.

Aquí les dejo un demo de todo el proceso.

{{ asciinema(id="385145") }}

> Ver en [asciinema](https://asciinema.org/a/385145).

## Mejorando la experiencia

Como habrán notado el uso del quickfix list en ocasiones no es muy intuitivo que digamos. Pero podemos mejorarlo. Puedo darles algunas sugerencias que pueden colocar en su `.vimrc`.

* Un mejor grep.

Lo primero sería crear un comando que ejecute el comando de búsqueda y que luego abra el quickfix window automáticamente. Por suerte para nosotros la documentación nos da una sugerencia, esta de aquí:

```vim
command! -nargs=+ Grep execute 'silent grep! <args>' | copen
```

Pueden cambiar `grep` por `vimgrep` si eso desean. Lo importante aquí es que ahora podrán invocar el comando `:Grep` (con `G` mayúscula) para realizar sus búsquedas.

* Navegar por los resultados.

Podríamos "navegar" por los items del quickfix list sin siquiera abrir el quickfix window de una manera cómoda con estos atajos.

```vim
" Ir a la ubicación anterior
nnoremap [q :cprev<CR>

" Ir a la siguiente ubicación
nnoremap ]q :cnext<CR>
```

* Manejo de la ventana.

Si tienden a usar el quickfix window con frecuencia será mejor que se creen unos atajos para abrir y cerrarlo con más facilidad.

```vim
" Mostrar el quickfix window
nnoremap <Leader>co :copen<CR>

" Ocultar el quickfix window
nnoremap <Leader>cc :cclose<CR>
```

* El errorformat.

Vamos a asegurarnos de que vim siempre pueda leer el formato del quickfix list si queremos actualizarlo.

```vim
augroup quickfix_mapping
    autocmd!
    autocmd filetype qf setlocal errorformat+=%f\|%l\ col\ %c\|%m
augroup END
```

* Atajos

También necesitamos atajos que funcionen sólo en el quickfix window. Ya saben, para hacer más cómoda la navegación y también hacer que el "caso avanzado" que les presenté no sea tan engorroso.

```vim
function! QuickfixMapping()
  " Ir a la ubicación anterior y mantenerse en el quickfix window
  nnoremap <buffer> K :cprev<CR>zz<C-w>w

  " Ir a la siguiente ubicación y mantenerse en el quickfix window
  nnoremap <buffer> J :cnext<CR>zz<C-w>w

  " Haz que el quickfix list sea modificable
  nnoremap <buffer> <leader>u :set modifiable<CR>

  " Actualiza el quickfix window
  nnoremap <buffer> <leader>w :cgetbuffer<CR>:cclose<CR>:copen<CR>

  " Buscar y reemplazar
  nnoremap <buffer> <leader>r :cdo s/// \| update<C-Left><C-Left><Left><Left><Left>
endfunction

augroup quickfix_mapping
    autocmd!
    autocmd filetype qf call QuickfixMapping()
augroup END
```

* Ahora todo junto.

```vim
" Comando Grep que busca y abre el quickfix window
command! -nargs=+ Grep execute 'silent grep! <args>' | copen

" Ir a la ubicación anterior
nnoremap [q :cprev<CR>

" Ir a la siguiente ubicación
nnoremap ]q :cnext<CR>

" Mostrar el quickfix window
nnoremap <Leader>co :copen<CR>

" Ocultar el quickfix window
nnoremap <Leader>cc :cclose<CR>

function! QuickfixMapping()
  " Ir a la ubicación anterior y mantenerse en el quickfix window
  nnoremap <buffer> K :cprev<CR>zz<C-w>w

  " Ir a la siguiente ubicación y mantenerse en el quickfix window
  nnoremap <buffer> J :cnext<CR>zz<C-w>w

  " Haz que el quickfix list sea modificable
  nnoremap <buffer> <leader>u :set modifiable<CR>

  " Actualiza el quickfix window
  nnoremap <buffer> <leader>w :cgetbuffer<CR>:cclose<CR>:copen<CR>

  " Buscar y reemplazar
  nnoremap <buffer> <leader>r :cdo s/// \| update<C-Left><C-Left><Left><Left><Left>
endfunction

augroup quickfix_mapping
  autocmd!
  
  " Asignar los atajos específicos para el quickfix window
  autocmd filetype qf call QuickfixMapping()
  
  " Agregar formato para modificar el quickfix list
  autocmd filetype qf setlocal errorformat+=%f\|%l\ col\ %c\|%m
augroup END
```

### Plugins

Si están dispuestos y saben cómo instalar plugins en vim yo recomendaría estos:

* [vim-qf](https://github.com/romainl/vim-qf)

Este hace que el quickfix window tenga un comportamiento más intuitivo. Por ejemplo, puede hacer que se abra automáticamente cuando ejecutamos `:grep`, `:vimgrep` e incluso `:make` sin tener que crear otros comandos. Pero si queremos disfrutar de algunas de sus bondades tenemos que crear nuestros propios atajos.

* Abrir o cerrar el quickfix window.

```vim
nmap <Leader>cc <Plug>(qf_qf_toggle)
```

* Navegar por los resultados

```vim
" Ir a la ubicación anterior
nmap [q <Plug>(qf_qf_previous)zz

" Ir a la siguiente ubicación
nmap ]q <Plug>(qf_qf_next)zz

function! QuickfixMapping()
  " Ir a la ubicación anterior y mantenerse en el quickfix window
  nmap <buffer> K <Plug>(qf_qf_previous)zz<C-w>w

  " Ir a la siguiente ubicación y mantenerse en el quickfix window
  nmap <buffer> J <Plug>(qf_qf_next)zz<C-w>w
endfunction
augroup quickfix_mapping
    autocmd!
    autocmd filetype qf call QuickfixMapping()
augroup END
```

La diferencia entre estos comandos y los que trae vim por defecto es que los de `vim-qf` no dan error cuando llegamos al final de la lista. Es decir que si estamos visualizando el último item y usamos `]q` para ir al siguiente, en lugar de tener un error este comando nos lleva al primer item del quickfix list.

* [quickfix-reflector](https://github.com/stefandtw/quickfix-reflector.vim)

Este plugin hace que todos los comandos que ejecutamos en el "caso avanzado" queden obsoletos. Nos da la posibilidad de modificar el quickfix window como un buffer normal. Pero eso no es todo, cada cambio que guardemos quedará "reflejado" en el archivo original.

Tomemos el ejemplo que mostré en el demo. Digamos que tenemos esto en el quickfix list.

```
./test dir/a-file.txt|1 col 11| nnoremap <leader>f :FZF
./test dir/a-file.txt|2 col 11| nnoremap <leader>ff :FZF<CR>
./test dir/a-file.txt|3 col 11| nnoremap <leader>fh :History<CR>
./test dir 2/another-file.txt|1 col 11| nnoremap <leader>? :Maps<CR>
./test dir 2/another-file.txt|2 col 11| nnoremap <leader>bb :Buffers<CR>
```

Y digamos que modifico el primer y cuarto item en la lista de manera manual.

```diff
- ./test dir/a-file.txt|1 col 11| nnoremap <leader>f :FZF
+ ./test dir/a-file.txt|1 col 11| nnoremap <AAA>f :FZF
  ./test dir/a-file.txt|2 col 11| nnoremap <leader>ff :FZF<CR>
  ./test dir/a-file.txt|3 col 11| nnoremap <leader>fh :History<CR>
- ./test dir 2/another-file.txt|1 col 11| nnoremap <leader>? :Maps<CR>
+ ./test dir 2/another-file.txt|1 col 11| nnoremap <BBB>? :Maps<CR>
  ./test dir 2/another-file.txt|2 col 11| nnoremap <leader>bb :Buffers<CR>
```

Si guardo estos cambios con el comando `:write` (o `:w` para abreviar) se quedarán reflejados en sus lugares respectivos en cada archivo. Esto resulta sumamente poderoso porque ahora la flexibilidad del quickfix list está limitada a nuestro conocimiento sobre vim.

Cualquier truco que conozcan para modificar varios fragmentos de código deberían funcionar con este plugin. Por ejemplo si queremos replicar el mismo efecto que mostré en el demo anterior lo que tenemos que hacer es esto:

* Borramos las líneas que no queremos modificar.

```
./test dir/a-file.txt|1 col 11| nnoremap <leader>f :FZF
./test dir/a-file.txt|3 col 11| nnoremap <leader>fh :History<CR>
./test dir 2/another-file.txt|1 col 11| nnoremap <leader>? :Maps<CR>
```

* Usamos el comando de substitución "normal."

```vim
:%s/leader/localleader/g
```

Después de eso el quickfix list debería quedar en este estado.

```
./test dir/a-file.txt|1 col 11| nnoremap <localleader>f :FZF
./test dir/a-file.txt|3 col 11| nnoremap <localleader>fh :History<CR>
./test dir 2/another-file.txt|1 col 11| nnoremap <localleader>? :Maps<CR>
```

* Guardamos el cambio.

```vim
:write
```

Y eso es todo.

## Conclusión

Nos hemos familiarizado con el quickfix list y sus utilidades más comunes. 

Sabemos que podemos usar `:make` para invocar cualquier compilador o comando externo que pueda ejecutar nuestro código y darnos mensajes de error. Tenemos las herramientas para hacer que vim entienda un mensaje de error y pueda incorporar la información relevante al quickfix list.

Vimos que podemos usar `:vimgrep` y `:grep` para buscar patrones en múltiples archivos. También pudimos explorar diferentes formas de reemplazar el texto que estamos buscando, con casos simples y otros un poco más complejos. Con estos casos pudimos aprender cómo hacer la sustitución sin usar plugins y también cómo hacerlo con la ayuda de un par de plugins.

Por último aprendimos sobre comandos, métodos y configuraciones que podemos agregar en nuestro `.vimrc` para hacer que la experiencia al usar el quickfix list sea más placentera e intuitiva.

## Fuentes

- [:help quickfix](https://vimhelp.org/quickfix.txt.html)
- [How to edit the vim quickfix list](https://www.reddit.com/r/vim/comments/7dv9as/how_to_edit_the_vim_quickfix_list/)

