+++
title = "Extendiendo deno cli usando una función" 
description = "Mejorando la interfaz de comandos de deno con un poco de magia"
date = 2021-04-18
lang = "es"
draft =true
[taxonomies]
tags = ["shell", "deno", "hoy-aprendi"]
+++

Estos días he estado usando `deno` con más frecuencia y debo decirles que aún hay un par de cosas que me molestan. Sí logré resolver algunos de esos problemas "agregando" algunos sub-comandos. No, no es magia negra, sólo un pequeño truco que aprendí hace tiempo y hoy voy a decirles cómo funciona.

## ¿Qué necesitamos?

En teoría cualquier intérprete que nos permita crear funciones debería ser suficiente. En este artículo les estaré mostrando los ejemplos usando una sintaxis compatibles con intérpretes que obedecen el estándar POSIX (piensen en "shells" como `zsh`, `bash`, `ash`, etc...).

## ¿Qué queremos lograr?

Personalmente a mí me encantaría que `deno` tuviera algún equivalente nativo a los `npm scripts`, pero parece que no tendremos nada de eso aún. Quisiera poder hacer algo así.

```sh
deno start hello world
```

También me gustaría poder inicializar esos scripts, porque deben existir en algún lado.

```sh
deno init
```

Como ya deben saber los sub-comandos `start` e `init` no existen y vamos a arreglar eso (casi).

## Empecemos

El primer paso sería ubicar el archivo que su intérprete favorito ejecuta cada vez que lo invocan. En `zsh` por ejemplo, pueden usar `~/.zshrc`. En `bash` pueden usar `~/.bashrc`. Esos dos son los únicos que conozco, si usan otro intérprete busquen en su documentación algún equivalente. 

### La Definición

¿Están listos? Ahora deben crear una función en ese archivo.

```sh
deno()
{
  echo "Hola"
}
```

Ahora si reinician el interprete (o "recargan" su configuración) e invocan el comando `deno` debería obtener `Hola` como resultado. Genial, hemos creado un problema, no podemos usar el comando `deno`. No teman, vamos por buen camino.

### Vuelve a mi, deno

Si quisieramos invocar el **comando** `deno` lo único que tenemos que hacer es usar el comando `command` así:

```sh
command deno --version
```

Eso debería mostrarle la información de `deno`.

```
deno x.x.x (release, x86_64-unknown-linux-gnu)
v8 x
typescript x.x
```

Con este nuevo conocimiento podemos resolver nuestro problema. Vamos a hacer una prueba.

```sh
deno()
{
  echo "Estás usando"
  command deno --version
}
```

Y esto es lo que debería ocurrir al invocarlo.

```
$ deno

Estás usando
deno x.x.x (release, x86_64-unknown-linux-gnu)
v8 x
typescript x.x
```

### Volviendo a la  normalidad

Ahora que sabemos cómo llamar a `deno` lo que deberíamos hacer ahora es imitar el comportamiento de `deno` en nuestra función. Debemos hacer esto de una manera que nos deje espacio para agregar nuestros propios comandos, para esto vamos a usar la declaración `case`.

```sh
deno()
{
  local cmd=$1; shift;

  case "$cmd" in
    *)
      command deno $cmd $@
    ;;
  esac
}
```

La primera línea lo que hace es asignar el primer parámetro (`$1`) a la variable `cmd` y luego lo elimina de la lista de argumentos (`$@`). Después lo que hacemos es comparar `cmd` con un patrón. Por ahora el único patrón que tenemos es `*`, que actúa como un comodín que coincide con todo, básicamente es nuestro default. Probemos.

```sh
deno --version
```

Bien, pero ahora si intentamos usar `deno` sin argumentos nos dará un error. Para arreglar eso vamos verificar si el primer argumento está vacío, si ese es el caso sólo llamamos `deno`.

```sh
deno()
{
  if [ -z "$1" ];then
    command deno
    return
  fi

  local cmd=$1; shift;

  case "$cmd" in
    *)
      command deno $cmd $@
    ;;
  esac
}
```

### Invocando sub-comandos

Ya estamos en el lugar que queremos, finalmente podremos agregar los subcomandos que queremos. Primero vamos a hacer una prueba.

```diff
  deno()
  {
    if [ -z "$1" ];then
      command deno
      return
    fi

    local cmd=$1; shift;

    case "$cmd" in
+     hola)
+       command deno eval "console.log('deno dice hola')"
+     ;;
      *)
        command deno $cmd $@
      ;;
    esac
  }
```

Si invocamos el comando hola:

```
$ deno hola

deno dice hola
```

Ahora estamos seguros que funciona.

### deno scripts

Ya podemos comenzar con lo más importante, nuestro reemplazo para los npm scripts. Lo que quiero hacer primero es crear el comando `start`, el cual es el más común que usaría.

```diff
  deno()
  {
    if [ -z "$1" ];then
      command deno
      return
    fi

    local cmd=$1; shift;

    case "$cmd" in
+     start)
+       command deno run --allow-run ./Taskfile.js start $@
+     ;;
      *)
        command deno $cmd $@
      ;;
    esac
  }
```

Esto lo que hará será ejecutar un archivo llamado `Taskfile.js` usando `deno`. `--allow-run` nos permitirá usar `Deno.run` en nuestro código para poder llamar comandos externos dentro de `Taskfile.js`. Vamos a asegurarnos de que funciona.

Creamos `Taskfile.js`.

```js
const cmd = ['echo', 'Taskfile: ', ...Deno.args];
Deno.run({ cmd });
```

Y usamos `start` de esta manera.

```
$ deno start hola

Taskfile start hola
```

Genial. El siguiente paso sería crear un `Taskfile.js` "más inteligente," que sea capaz de ejecutar varias tareas. Ya he hecho algo así en el pasado (Pueden encontrar los detalles [aquí](https://dev.to/vonheikemen/a-simple-way-to-replace-npm-scripts-in-deno-4j0g)), así que les mostraré el código que yo usaría.

```js
const entrypoint = "./src/main.js";

run(Deno.args, {
  start(...args) {
    exec(["deno", "run", entrypoint, ...args]);
  },
  list() {
    console.log('Available tasks: ');
    Object.keys(this).forEach((k) => console.log(`* ${k}`));
  },
});

function run([name, ...args], tasks) {
  if(tasks[name]) {
    tasks[name](...args);
  } else {
    console.log(`Task "${name}" not found\n`);
    tasks.list();
  }
}

async function exec(args) {
  const proc = await Deno.run({ cmd: args }).status();

  if (proc.success == false) {
    Deno.exit(proc.code);
  }

  return proc;
}
```

Con esto podríamos ejecutar el archivo `./src/main.js` al invocar `deno start`. Ahora tenemos otro problema, no queremos escribir todo eso cada vez que queremos iniciar un proyecto. Lo que haremos será crear un comando `init` que copie esta plantilla en la carpeta de nuestro proyecto.

```diff
  deno()
  {
    if [ -z "$1" ];then
      command deno
      return
    fi

    local cmd=$1; shift;

    case "$cmd" in
      start)
        command deno run --allow-run ./Taskfile.js start $@
      ;;
+     init)
+       cp /path/to/template/Taskfile.js ./
+       echo "Taskfile.js created"
+     ;;
      *)
        command deno $cmd $@
      ;;
    esac
  }
```

Ya tiene buena forma. La última cosa que nos quedaría por hacer es manejar otras tareas además de `start`. Con `npm` podemos invocar `npm run` para elegir qué script queremos ejecutar, nuestro equivalente sólo se llamará `x`.

```diff
  deno()
  {
    if [ -z "$1" ];then
      command deno
      return
    fi

    local cmd=$1; shift;

    case "$cmd" in
+     x)
+       command deno run --allow-run ./Taskfile.js $@
+     ;;
      start)
        command deno run --allow-run ./Taskfile.js start $@
      ;;
      init)
        cp /path/to/template/Taskfile.js ./
        echo "Taskfile.js created"
      ;;
      *)
        command deno $cmd $@
      ;;
    esac
  }
```

Ahora podríamos tener una "tarea" llamada `test:api` y llamarla de esta manera.

```sh
deno x test:api
```

### Un poco más de conveniencia

Lo último que quiero hacer es crear una forma de llamar a un script el cual pueda llamar las librerías que yo uso con más frecuencia sin usar su URL. Podemos hacer esto con el apoyo de un [import-map](https://deno.land/manual@v1.9.0/linking_to_external_code/import_maps), un archivo `.json` en el cual podemos vincular un "alias" a una URL.

Yo uso uno como este.

```json
{
  "imports": {
    "@std/": "https://deno.land/std@0.93.0/",
    "@npm/": "https://jspm.dev/",

    "ansi-colors": "https://jspm.dev/ansi-colors@4.1.1",
    "arg": "https://jspm.dev/arg@5.0.0",
    "cheerio": "https://jspm.dev/cheerio@1.0.0-rc.5",
    "exec": "https://deno.land/x/exec@0.0.5/mod.ts",
    "ramda": "https://jspm.dev/ramda@0.27.1",

    "@utils/": "/path/to/deno/utils/"
  }
}
```

`deno` puede leerlo si le pasamos el argumento `--import-map`. Agregemos el comando.
 
```diff
  deno()
  {
    if [ -z "$1" ];then
      command deno
      return
    fi

    local cmd=$1; shift;

    case "$cmd" in
      x)
        command deno run --allow-run ./Taskfile.js $@
      ;;
      start)
        command deno run --allow-run ./Taskfile.js start $@
      ;;
      init)
        cp /path/to/template/Taskfile.js ./
        echo "Taskfile.js created"
      ;;
+     s|script)
+       command deno run --import-map="/path/to/deno/import-map.json" $@
+     ;;
      *)
        command deno $cmd $@
      ;;
    esac
  }
```

Como ya es costumbre, tenemos que probar esto. Vamos a crear un script de prueba con este contenido.

```js
import dayjs from '@npm/dayjs';
import c from 'ansi-colors';

c.enabled = !Deno.noColor;

const date = dayjs().format('{YYYY} MM-DDTHH:mm:ss SSS [Z] A');

console.log(c.green(date));
```

> Tal vez quieran especificar la versión de la librería que están usando con `@npm`, en ese caso harían esto `@npm/dayjs@1.10.4`

Lo ejecutamos.

```
$ deno script ./test.js

Download https://jspm.dev/dayjs@1.10.4
Download https://jspm.dev/npm:dayjs@1.10.4!cjs
{2021} 04-18T11:28:05 929 Z AM
```

¡Jaja! Ahora tengo todo lo que quiero.

### El resultado final

Después de todo este proceso la función `deno` debería quedar así.

```sh
deno()
{
  if [ -z "$1" ];then
    command deno
    return
  fi

  local cmd=$1; shift

  case "$cmd" in
    x)
      command deno run --allow-run ./Taskfile.js $@
    ;;
    start)
      command deno run --allow-run ./Taskfile.js start $@
    ;;
    init)
      cp /path/to/template/Taskfile.js ./
      echo "Taskfile.js created"
    ;;
    s|script)
      command deno run --import-map="/path/to/deno/import-map.json" $@
    ;;
    *)
      command deno $cmd $@
    ;;
  esac
}
```

## Conclusión

Entonces, ¿qué hicimos hoy? "Escondimos" el comando `deno` detrás de una función para poder agregar "sub-comandos" creados por nosotros. Logramos crear una especie de equivalente de `npm run` para `deno` y finalmente usamos import maps para simplificar la declaración de dependencias en un script. No está mal para un día de trabajo.

