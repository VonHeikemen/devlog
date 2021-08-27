+++
title = "Usando 'abbreviations' en vim" 
description = "Buscando usos interesantes para abbreviations en vim"
date = 2020-11-14
lang = "es"
[taxonomies]
tags = ["vim", "shell"]
[extra]
shared = [
  ["dev.to", "https://dev.to/vonheikemen/usando-abbreviations-en-vim-569i"],
  ["Hashnode", "https://vonheikemen.hashnode.dev/using-vim-abbreviations-es"]
]
+++

Una de las cosas que vim hace extremadamente bien es la automatización. Con esto en mente hoy exploraremos una funcionalidad llamada "abbreviations" (abreviación) y cómo podemos usarlas para automatizar cosas en el modo de inserción (se puede usar en otros modos pero sólo nos enfocaremos en el modo de inserción). Primero veremos unos casos básicos y luego nos moveremos hacia áreas más complejas. Pero no se preocupen, haremos todo de manera progresiva, paso a paso.

## Paso 0

Para crear una abreviación podemos usar el comando `:ab[breviate]`. Pero ya que sólo queremos que estén disponibles en el modo de inserción usaremos el comando `:iabbrev`. Esta es la sintaxis. 

```
:iabbrev [<expr>] [<buffer>] {abreviación} {expansión}
```

En este comando todo lo que se encuentre dentro de un par de `[]` es opcional. El argumento `<expr>` es para indicar que queremos usar una "expresión" (lo que se considere una expresión en vimscript) para crear la expansión. `<buffer>` es para indicar que queremos que esta abreviación sólo este disponible para el buffer actual. `abreviación` es el argumento que uno ingresa en modo de inserción y que será remplazado por `expansión`. 

El comando anterior puede usarse en conjunto con un [autocommand](https://learnvimscriptthehardway.stevelosh.com/chapters/12.html) para crear abreviaciones que sólo apliquen a un tipo específico de archivo. Algo así.

```
:autocmd FileType html,javascript,typescript,vue
  \ :iabbrev <buffer> una-abreviación una-expansión
```

En este ejemplo `una-abreviación` estará disponible en el buffer que sea considerado de tipo `html`, `javascript`, `typescript` o `vue`. Usando eso en su `.vimrc` pueden crear sus abreviaciones automáticamente.

## Hola munod

Errores tipográficos. ¿Acaso no les molestan? ¿No les a pasado por lo menos que su código no funciona porque escribieron `heigth` en lugar de `height`? Bueno, ya no más. Ahora tenemos las herramientas para arreglar este tipo de errores automáticamente.

```
:iabbrev heigth height
```

Después de ejecutar ese comando, cuando escriban `heigth` y presionen `espacio` se convertirá en `height`.

![arreglando un error](https://res.cloudinary.com/vonheikemen/image/upload/v1604437302/devlog/using-vim-abbreviations/1-typos_03-11-2020_16-51.gif)

Pero la diversión no termina ahí.

## No sólo es para las letras

Resulta que también podemos usar secuencias especiales como `<CR>` (la tecla `enter`), `<Up>`, `<Tab>` y muchas otras más. Esto nos da la posibilidad de mover la posición del cursor y crear "snippets" más elaborados.

Intentemos con algo común. Una de las cosas que hago en javascript con mucha frecuencia es escribir `console.log()`, y no sólo soy yo, esto es tan común que algunos editores ya cuentan con un atajo para generarlo. Vim no tiene un atajo para esto pero nosotros podemos hacer uno.

```
:iabbrev <buffer> con@ console.log();<Left><Left>
```

![un snippet simple](https://res.cloudinary.com/vonheikemen/image/upload/v1604438822/devlog/using-vim-abbreviations/2-simple-snippet_03-11-2020_17-26.gif)

El símbolo `@` no tiene importancia en este caso, es sólo una convención que yo sigo en mi configuración, sólo para recordar que será reemplazado por otra cosa. Uso `<buffer>` porque no quiero que esta abreviación esté presente en cada buffer, sólo en los que pueda usar javascript. Generalmente lo uso con un autocommand con el que mostré anteriormente. 

Notarán que hay un espacio entre los paréntesis en el método `log`, eso es porque elegí expandir la abreviación con la tecla `espacio`. Vim comienza la expansión cuando presiono un "non-keyword character" y ese caracter es agregado al final de la expansión. Por si preguntan, todas las letras y unos caracteres especiales son considerados "keyword character", y cualquier cosa que no sea parte de ese conjunto comenzará la expansión. Si quieren saber más detalles revisen la documentación con `:help 'iskeyword'`. 

En fin, si no quieren ese espacio extra pueden comenzar la expansión con la secuencia `<C-]>` (`control + corchete de cierre`) ó con la tecla `<Esc>`.

## Espera... ¿alguien dijo snippet?

Sí, claro que sí. Con esta funcionalidad podemos crear snippets complejos como los que encontrarías en un IDE súper avanzado.

¿Se encuentran editando un archivo de javascript y quieren crear una expresión de función ejecutada inmediatamente (un [iife](https://developer.mozilla.org/es/docs/Glossary/IIFE))? No hay problema.

```
:iabbrev <buffer> iife@ (async function() {})();<Left><Left><Left><Left><Left><CR><CR><Up>
```

![iife snippet](https://res.cloudinary.com/vonheikemen/image/upload/v1604442411/devlog/using-vim-abbreviations/3-another-snippet_03-11-2020_18-25.gif)

Ah, no, ¿saben qué? Sí hay un problema. Tantos `<Left>` molestan un poco. ¿Pueden imaginarse hacer algo así pero con un snippet más complejo, como `switch/case`? Escribirlo con ese estilo sería tedioso, funcionaría pero sería tedioso. Pero no se preocupen hay una solución para esto.

### Un escape del modo de inserción

Ya que básicamente estamos automatizando pulsaciones de teclas podemos hacer algo gracioso, podemos "presionar" escape (`<Esc>`) para ir al modo normal y tomar ventaja de todas las bondades que trae vim en ese modo. Podríamos reescribir el snippet anterior así.

```
:iabbrev <buffer> iife@ (async function() {})();<Esc>4hi<CR><CR><Up>
```

Con esto tenemos el mismo snippet pero con menos ruido. Reemplazamos todos los `<Left>` con `<Esc>4hi`, es más enigmático pero más corto. Aunque me atrevo a decir que si han usado vim por un tiempo ya deben saber qué pasa si presionan `4hi` manualmente. Y ya que básicamente sólo entramos en modo normal para ejecutar un comando (`4h`) y volver nuevamente al modo de inserción, podemos simplificar aún más esa abreviación.

```
:iabbrev <buffer> iife@ (async function() {})();<C-o>4h<CR><CR><Up>
```

En esta ocasión usamos `<C-o>` (`control` + o) para ir al modo normal, también hace que vim entre en modo de inserción luego de ejecutar un comando.

## Ahora a las extravagancias

¿Qué tal si llevamos esto un paso más adelante? Hagamos un snippet que deje el cursor en un lugar conveniente sin necesidad de usar direcciones explicitas como `<Left>` y `<Right>`.

Podría mostrarles otra versión de la abreviación `iife@` pero quiero darles más ideas. Hagamos un ciclo `for` "tradicional". Ya saben, como este.


```js
for(let i = 0; i < {PLACEHOLDER}; i++) {

}
```

Al final de la expansión quiero que el cursor quede donde dice `{PLACEHOLDER}`, pero sin tanta ceremonia, sólo quiero llegar ahí. ¿Cómo lo logramos? Con una búsqueda. Sí, también podemos iniciar una búsqueda, ya sea con `/` o con `?`.

```
:iabbrev <buffer> forii@ for(let i = 0; i <z; i++) {<CR><CR>}<Esc>?z<CR>xi
```

¿Notaron la `z`? Ese sería mi placeholder. Luego de que todo ya está escrito presiono `<Esc>`, luego `?` (una búsqueda en reverso), buscamos la `z`, presionamos `enter` y eso me lleva a donde quiero, finalmente presionamos `xi` para borrar la `z` e ir nuevamente al modo de inserción.

![un bonito ciclo for](https://res.cloudinary.com/vonheikemen/image/upload/v1604449379/devlog/using-vim-abbreviations/4-with-placeholder_03-11-2020_20-17.gif)

## Más Poder

¿Qué más podemos hacer? Podríamos usar argumentos. Usar una palabra como si fuera el argumento de una función. Suena interesante. Sé exactamente dónde necesito algo como eso. Empecemos.

¿Alguno de ustedes conoce [vue-vscode-snippets](https://github.com/sdras/vue-vscode-snippets)? ¿No? Está bien, igual les diré. Esos son snippets de vue para vscode. Uno de esos snippets se llama `vimport-export`, que es responsable de crear este código.

```js
import Name from '@/components/Name.vue';

export default {
  component: {
    Name,
  }
};
```

La parte genial de ese snippet es que usa la funcionalidad de cursores múltiples para remplazar `Name` con el nombre del componente. Tengo que ser sincero, estoy celoso, quisiera tener esa funcionalidad en vim. Ya sé que existe un plugin, pero quiero que sea una funcionalidad nativa. En fin, no necesitamos múltiples cursores, sólo tenemos que usar la misma palabra en lugares diferentes.

Esto es lo que va a pasar: voy a poner una palabra (el nombre de mi componente) y luego la abreviación. Así.

```
MyComponent vcomp@
```

Cuando la expansión comience lo que haré será poner `MyComponent` en un registro y simplemente lo pegaré en los lugares que necesite. ¿Están listos? Empecemos con el `import`.

```
:iabbrev <buffer> vcomp@ <Esc>bvediimport <C-o>P from '@/components/<C-o>P.vue';
```

El componente principal de esto es `<Esc>bved`. `<Esc>` nos lleva al modo normal, `b` toma el cursor y lo coloca al principio de la palabra anterior (el nombre del componente) y `ved` seleccionará toda la palabra y la borrará. Gracias a la manera en la que `d` funciona tendremos el nombre del componente en un registro listo para ser copiado cuando sea necesario.

Okey, esa es la primera parte del problema, y para ser justos podríamos resolver la otra parte de la misma manera pero vamos a tomar otro rumbo. El caso común para esto sería cuando vamos a registrar un componente dentro de otro componente, en este caso ya debería existir `export default` y lo que queremos es agregarlo en la propiedad `components`. Lo que haremos será automatizar este proceso:

![registrando un componente](https://res.cloudinary.com/vonheikemen/image/upload/v1604454412/devlog/using-vim-abbreviations/5-vexport_03-11-2020_21-40.gif)

Esta segunda fase comienza con `<Esc>`, luego usamos `/` para buscar la cadena `components: {`, terminamos la búsqueda y entramos en modo de inserción en la linea que está debajo usando `o`, luego usamos `<Tab>` para llegar al lugar correcto y pegar el nombre del componente (que aún tenemos en un registro) y ponemos una coma. Podríamos terminar en ese punto pero quisiera preservar el mismo orden de los `import`. Entonces, lo que hacemos es tomar toda la linea donde está el componente con `<Esc>dd`, luego vamos al final de la linea con `k$`. Para ir al fondo del objeto usamos `%`, una vez ahí vamos a la última linea con `k` y creamos otra linea con `o`. Ya estamos en el lugar que queremos, presionamos `<Esc>` para ir al modo normal y pegamos la linea con el componente usando `P` (`p` mayúscula). Ahora todo está listo pero queda una linea extra, la borramos usando `jdd`. Finalmente nos devolvemos al lugar del import con `2<C-o>` (necesitamos hacer dos saltos, por eso el 2). Entonces... necesitamos esto:

```
<Esc>/components: {<CR>o<Tab><Esc>pa,<Esc>ddk$%ko<Esc>Pjdd2<C-o>a
```

Y todo junto es así.

```
:iabbrev <buffer> vcomp@ <Esc>bvediimport <C-o>P from '@/components/<C-o>P.vue';<Esc>/components: {<CR>o<Tab><Esc>pa,<Esc>ddk$%ko<Esc>Pjdd2<C-o>a
```

![registrar un componente](https://res.cloudinary.com/vonheikemen/image/upload/v1604456390/devlog/using-vim-abbreviations/6-vcomp_03-11-2020_22-15.gif)

Si tienen tiempo para un ejercicio intenten crear una abreviación que transforme esto.

```js
Example vvcomp@

export default {

};
```

En esto.

```js
import Example from '@/components/Example.vue';

export default {
  components: {
    Example,
  },
};
```

### La cereza del pastel

Las abreviaciones funcionan incluso dentro un macro. Intenten grabar este macro y úsenlo varias veces.

```
0ea vcomp@^]^]j
```

`^]` es la tecla `<Esc>`.

![macros](https://res.cloudinary.com/vonheikemen/image/upload/v1604457566/devlog/using-vim-abbreviations/7-macro_03-11-2020_22-34.gif)


## Conclusión

Eso es todo. Es todo lo que tengo por ahora. Espero que hayan encontrado algo útil o al menos algo gracioso que jamás utilizarán.

Si necesitan más ideas pueden revisar [mis snippets](https://github.com/VonHeikemen/dotfiles/blob/master/my-configs/vim/snippets.vim). Si ustedes tienen alguna idea interesante, díganme.

## Fuentes
- [:help abbreviations](https://vimhelp.org/map.txt.html#Abbreviations)
- [Learn Vimscript the Hard Way: Abbreviations](https://learnvimscriptthehardway.stevelosh.com/chapters/08.html)

