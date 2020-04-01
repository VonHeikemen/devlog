+++
title = "Un poco del paradigma funcional en tu javascript: Técnicas de composición"
description = "Una introducción a los patrones comúnes usados en la programación funcional"
date = 2020-03-29
lang = "es"
[taxonomies]
tags = ["javascript", "paradigma-funcional", "aprendizaje"]
+++

Hoy el tema será la composición de funciones. El arte de crear cosas complejas con piezas "simples". Si no saben nada del paradigma funcional en la programación incluso mejor, esta será una introducción a conceptos y patrones de ese paradigma que pueden implementarse en javascript. Lo que voy a presentar no será una fórmula mágica que hará su código más legible, simple y sin errores; así no funcionan las cosas. Sí creo que ayuda en la solución de problemas pero para sacarle el mayor provecho deben tener en cuenta ciertas cosas. Así que antes de mostrar cualquier implementación vamos a hablar de algunos conceptos y filosofía.

## Lo que deben saber

### ¿Qué es la composición de funciones?

Es un mecanismo que nos permite combinar dos o más funciones en una nueva función.

Parece una idea simple, ciertamente todos en algún momento hemos combinado un par de funciones ¿De verdad pensamos en la composición cuando creamos una función? ¿Qué nos ayudaría a crear funciones diseñadas para ser combinadas?

### Filosofía

Repito, la composición de funciones es más efectiva si siguen ciertos principios.

- La función tiene un sólo propósito, una sola responsabilidad.
- Asume que el resultado de la función será consumido por otra.

Probablemente han escuchado eso en algún otro lado, es parte de la [filosofía unix](https://en.wikipedia.org/wiki/Unix_philosophy#Origin). ¿Alguna vez se han preguntado cómo un lenguage como `bash`, que tiene una sintaxis un tanto extraña y muchas limitaciones, puede ser tan popular? Esos dos principios son parte de la razón. Una gran parte de los programas que se ejecutan en este ambiente están diseñados para ser componentes reusables y cuando "conectas" dos o más, el resultado es un programa que también puede ser conectado con otros programas aún no conocidos.

Para algunos puede parecer tonto o incluso exagerado tener muchas funciones que solo hacen una cosa, especialmente si esas funciones hacen algo que parece inútil, pero puedo demostrarles que cada función puede ser valiosa en el contexto adecuado.

Intentemos ilustrar una situación donde estos principios se ponen en práctica.

> Nota: De antemano me disculpo por el uso indebido de los comandos `cat` y `grep`, esto lo hago para demostrar el valor de la composición.

Digamos que queremos extraer el valor de la variable `HOST` que está en un archivo `.env`, vamos a hacerlo usando `bash`. 

Este sería el archivo.

```
ENV=development
HOST=http://locahost:5000
```

Para mostrar el contenido de ese archivo usamos `cat`.

```sh
cat .env
```

Para filtrar el contenido del archivo y buscar la línea que queremos usamos `grep`, le proveemos el patrón que buscamos y el contenido del archivo.

```sh
cat .env | grep "HOST=.*"
```

Para obtener el valor que queremos usamos `cut`. El comando `cut` va a tomar el resultado de `grep` y lo va a dividir usando un delimitador, luego le decimos qué sección de la cadena queremos.

```sh
cat .env | grep "HOST=.*" | cut --delimiter="=" --fields=2
```

Eso debería mostrarnos.

```
http://locahost:5000
```

Si colocamos esa cadena de comandos en un script o una función en nuestro `.bashrc` efectivamente tendremos un comando que puede ser usado de la misma manera por otros programas que aún no conocemos. Este es el tipo de flexibilidad y poder que queremos lograr.

Espero que en este punto sepan qué tipo de mentalidad debemos tener al momento de crear una función pero aún hay una cosa que deben recordar.

### Las funciones son cosas

Pongamos nuestra atención en javascript. ¿Han escuchado la frase "funciones de primera clase"? Significa que las funciones pueden ser tratadas como cualquier otro valor. Vamos a compararlos con los arreglos.

- Pueden asignarlos a una variable.

```js
const numbers = ['99', '104'];
const repeat_twice = function(str) {
  return str.repeat(2);
};
```

- Pasarlos como argumento a una función.

```js
function map(fn, array) {
  return array.map(fn);
}

map(repeat_twice, numbers);
```

- Pueden ser retornados por una función

```js
function unary(fn) {
  return function(arg) {
    return fn(arg);
  }
}

const safer_parseint = unary(parseInt);

map(safer_parseint, numbers);
```

¿Por qué les muestro esto? Deben estar conscientes de esta característica de javascript porque vamos a usarla crear funciones auxiliares, como `unary`, que manipulan otras funciones. Puede que tome un tiempo acostumbrarse a la idea de tratar las funciones como un dato pero definitivamente vale la pena practicarlo ya que es clave para entender muchos de los patrones que se pueden ver en el paradigma funcional. 

## Composición en la práctica

Vamos a retomar el ejemplo del archivo `.env`. Recrearemos lo que hicimos en `bash`. Primero vamos a intentar un enfoque muy directo, luego exploraremos los defectos de nuestra implementación e intentaremos solucionarlos.

Ya hemos hecho esto antes, sabemos lo que debemos hacer. Empecemos por crear una función por cada paso.

- Extraer el contenido del archivo.

```js
const fs = require('fs');

function get_env() {
  return fs.readFileSync('.env', 'utf-8');
}
```

- Filtrar el contenido basados en un patrón.

```js
function search_host(content) {
  const exp = new RegExp('^HOST=');
  const lines = content.split('\n');

  return lines.find(line => exp.test(line));
}
```

- Extraer el valor.

```js
function get_value(str) {
  return str.split('=')[1];
}
```

Ya estamos listos. Veamos qué podemos hacer para que estas funciones trabajen juntas.

### Composición natural

Mencioné que el primer intento sería un enfoque directo, las funciones ya están listas y lo que queda por hacer es ejecutarlas en secuencia.

```js
get_value(search_host(get_env()));
```

Digamos que este es el escenario perfecto de una composición de funciones, aquí el resultado de una función se convierte en la entrada de la siguiente, es el mismo efecto que tiene el símbolo `|` en `bash`. A diferencia de `bash` aquí el flujo de datos va de derecha a izquierda. 

Ahora imaginemos que tenemos dos funciones más que hacen algo con el valor de `HOST`.

```js
test(ping(get_value(search_host(get_env()))));
```

Las cosas se ponen algo incómodas, todavía esta en un nivel manejable pero la cantidad de paréntesis involucrados ya empieza a molestar. Este sería el momento perfecto para crear una función que agrupe esta cadena en una manera más legible, pero no haremos eso aún, primero buscaremos ayuda.

### Composición automática

Es aquí donde nuestros conocimientos de las funciones empieza a dar frutos. Lo que haremos para resolver el problema de los paréntesis será "automatizar" las llamadas de las funciones. Crearemos una función que acepte una lista de funciones, las ejecute una por una y se asegure de pasar el resultado de la función anterior como parémetro a la siguiente.

```js
function compose(...fns) {
  return function _composed(...args) {
    // Posición de la última función
    let last = fns.length - 1;

    // Se ejecuta la última función
    // con los parámetros de `_composed`
    let current_value = fns[last--](...args);

    // recorremos las funciones restantes en orden inverso
    for (let i = last; i >= 0; i--) {
      current_value = fns[i](current_value);
    }

    return current_value;
  };
}
```

Ahora podremos hacer esto.

```js
const get_host = compose(get_value, search_host, get_env);

// get_host en realidad es `_composed`
get_host();
```

Ya no tenemos el problema de los paréntesis, podemos agregar más funciones de manera más fácil y sin entorpecer la legilibilidad.

```js
const get_host = compose(
  test,
  ping,
  get_value,
  search_host,
  get_env
);

get_host();
```

Como en nuestro primer intento el flujo de ejecución va de derecha a izquierda. Si prefieren invertir el orden sería así.

```js
function pipe(...fns) {
  return function _piped(...args) {
    // Se ejecuta la primera función
    // con los parámetros de `_piped`
    let current_value = fns[0](...args);

    // recorremos las funciones restantes en el orden original
    for (let i = 1; i < fns.length; i++) {
      current_value = fns[i](current_value);
    }

    return current_value;
  };
}
```

Ahora pueden leerlo así.

```js
const get_host = pipe(get_env, search_host, get_value);

get_host();
```

Todo esto es genial, pero como dije antes lo que tenemos aquí es un escenario ideal. Nuestra composición sólo puede manejar funciones que tienen un parámetro de entrada y una sola línea de ejecución (no necesita controlar el flujo de ejecución). Eso no es malo, todos deberíamos diseñar nuestro código para facilitar ese tipo de situaciones pero como todos sabemos...

### No siempre es tan fácil

Incluso en nuestro ejemplo la única razón por la que logramos combinar las funciones fue porque incluimos en el código todos los parámetros necesarios e ignoramos el manejo de errores. Pero no todo está perdido, hay formas de sobrepasar las limitaciones que tenemos.

Antes de continuar modificaremos el ejemplo, haremos que sea más parecido a la implementación en `bash`.

```js
const fs = require('fs');

function cat(filepath) {
  return fs.readFileSync(filepath, 'utf-8');
}

function grep(pattern, content) {
  const exp = new RegExp(pattern);
  const lines = content.split('\n');

  return lines.find(line => exp.test(line));
}

function cut({ delimiter, fields }, str) {
  return str.split(delimiter)[fields - 1];
}
```

No es exactamente lo mismo que sus contrapartes en `bash` pero servirá. Ahora, si quisieramos combinar estas nuevas funciones tendríamos que hacerlo de esta manera.

```js
cut({delimiter: '=', fields: 2}, grep('^HOST=', cat('.env')));
```

Funciona pero yo diría que está al borde de lo aceptable, aún puedo entender lo que está pasando pero no querría agregar otra cosa a esa cadena. Si queremos usar `pipe` tendremos que superar nuestro primer obstáculo.

#### Funciones con múltiples entradas

La solución a esto es **aplicación parcial** y por suerte para nosotros javascript tiene un buen soporte incluido para lo que queremos hacer. Nuestro objetivo es simple, pasarle a una función una parte de sus parámetros sin ejecutarla. Queremos ser capaces de hacer algo así.

```js
const get_host = pipe(
  cat,
  grep('^HOST='), 
  cut({ delimiter: '=', fields: 2 })
);

get_host('.env');
```

Para replicar este resultado tendremos que recurrir a una técnica llamada **currying**, esta consiste en convertir una función de múltiples parámetros en varias funciones de un parámetro. Bien, para lograrlo lo que debemos hacer es aceptar un parámetro a la vez devolviendo una función por cada parámetro que necesitamos. Haremos esto con `grep` y `cut`.

```diff
- function grep(pattern, content) {
+ function grep(pattern) {
+   return function(content) {
      const exp = new RegExp(pattern);
      const lines = content.split('\n');
 
      return lines.find(line => exp.test(line));
+   }
  }

- function cut({ delimiter, fields }, str) {
+ function cut({ delimiter, fields }) {
+   return function(str) {
      return str.split(delimiter)[fields - 1];
+   }
  }
```

En situaciones donde no es posible convertir una función normal a una que soporte currying lo que podemos hacer es usar el método [bind](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Function/bind) que se encuentra en el prototipo de la funciones.

```js
const get_host = pipe(
  cat,
  grep.bind(null, '^HOST='), 
  cut.bind(null, { delimiter: '=', fields: 2 })
);
```

Por último, si todo lo demás parece muy complicado siempre tenemos la opción de crear una función anónima justo en el lugar.

```js
const get_host = pipe(
  cat,
  content => grep('^HOST=', content), 
  str => cut({ delimiter: '=', fields: 2 }, str)
);
```

Eso será suficiente para resolver cualquier tipo de problema con funciones que aceptan múltiples parámetros. Sigamos.

#### Funciones con múltiples salidas

¿Múltiples salidas? Con eso me refiero a funciones que retornan dos (tal vez más) tipos de resultados. Esto pasa en funciones que tienen distintos tipos de respuesta dependiendo de cómo las usemos o el contexto donde las usamos. Tenemos ese tipo de funciones en nuestro ejemplo, veamos `cat`.

```js
function cat(filepath) {
  return fs.readFileSync(filepath, 'utf-8');
}
```

Dentro de `cat` está la función `readFileSync`, es la que se encarga de leer el archivo en nuestro sistema, la cual es una operación que puede fallar por muchas razones. Entonces `cat` puede devolver un `String` si todo sale bien pero también puede arrojar un error si algo malo ocurre. Tenemos que manejar estos dos casos.

Desafortunadamente para nosotros las excepciones no son lo único con lo que tenemos que lidiar, también tenemos que manejar la ausencia de valores. En la función `grep` tenemos esta línea.

```js
lines.find(line => exp.test(line));
```

El método `find` se encarga de evaluar cada línea del contenido del archivo. Como pueden imaginar esta operación también puede fallar, simplemente puede darse el caso de que no encuentre el valor que buscamos. A diferencia de `readFileSync` el método `find` no arroja un error, lo que hace es retornar `undefined`. De por sí `undefined` no es malo, es sólo que no tenemos ninguna utilidad para él. Asumir que el resultado siempre será de tipo `String` es lo que en definitiva causará un error.

¿Cúal es la solución? 

**Functors** && **Monads** (me disculpan las palabrotas). Dar una explicación apropiada de esos conceptos toma tiempo así que sólo vamos a enfocarnos en lo que nos interesa. Por los momentos pueden pensar en ellos como estructuras que siguen ciertas reglas (pueden encontrar algunas de ellas aquí: [Fantasy land](https://github.com/fantasyland/fantasy-land#fantasy-land-specification)).

¿Cómo empezamos? Empecemos con los functors.

- Functors

Vamos a crear una estructura que sea capaz de ejecutar una función en el momento adecuado. Ya se han encontrado con una que puede hacer eso: los arreglos. Intenten esto.

```js
const add_one = num => num + 1;
const number = [41];
const empty = [];

number.map(add_one); // => [42]
empty.map(add_one);  // => []
```

¿Se dieron cuenta? `map` ejecutó `add_one` sólo una vez, con el arreglo `number`. No hizo nada en el arreglo vacío, no detuvo la ejecución del programa arrojando un error, sólo devolvió un arreglo. Ese es el tipo de comportamiento que queremos.

Repliquemos esto por nuestra cuenta. Vamos a crear una estructura llamada `Result`, esta representará una operación que puede o no tener éxito. Tendrá un método `map` que sólo ejecutará la función que recibe como parámetro si la operación resulta exitosa.

```js
const Result = {};

Result.Ok = function(value) {
  return {
    map: fn => Result.Ok(fn(value)),
  };
}

Result.Err = function(value) {
  return {
    map: () => Result.Err(value),
  };
}
```

Tenemos nuestro functor pero ahora se podrían estar preguntando ¿Es todo, cómo nos ayuda eso? Lo estamos haciendo un paso a la vez. Usemos lo que tenemos en `cat`.

```js
function cat(filepath) {
  try {
    return Result.Ok(fs.readFileSync(filepath, 'utf-8'));
  } catch(e) {
    return Result.Err(e);
  }
}
```

¿Qué ganamos? Intenten esto.

```js
cat('.env').map(console.log);
```

Todavía tienen la misma pregunta en su mente, puedo verlo. Ahora intenten incorporar el resto de las funciones.

> Nota: Voy a asumir que pueden usar currying para lograr la aplicación parcial de los parámetros.

```js
cat('.env')
  .map(grep('^HOST='))
  .map(cut({ delimiter: '=', fields: 2 }))
  .map(console.log);
```

¿Vieron? Esa cadena de `map`s se parece mucho a `compose` y `pipe`. Logramos recuperar la composición y le incorporamos manejo de errores (casi).

Quiero hacer algo. Ese patrón que hicimos en el `try/catch` parece útil, podríamos extraerlo a una función.

```js
 Result.make_safe = function(fn) {
  return function(...args) {
    try {
      return Result.Ok(fn(...args));
    } catch(e) {
      return Result.Err(e);
    }
  }
 }
```

Ahora podemos transformar `cat` sin siquiera tocar su código.

```js
const safer_cat = Result.make_safe(cat);

safer_cat('.env')
  .map(grep('^HOST='))
  .map(cut({ delimiter: '=', fields: 2 }))
  .map(console.log);
```

Tal vez quieran hacer algo en caso de error, ¿cierto? Hagamos que sea posible.

```diff
  const Result = {};
 
  Result.Ok = function(value) {
    return {
      map: fn => Result.Ok(fn(value)),
+     catchMap: () => Result.Ok(value),
    };
  }
 
  Result.Err = function(value) {
    return {
      map: () => Result.Err(value),
+     catchMap: fn => Result.Err(fn(value)),
    };
  }
```

Ahora podemos equivocarnos con confianza.

```js
const safer_cat = Result.make_safe(cat);
const show_error = e => console.error(`Whoops:\n${e.message}`);

safer_cat('what?')
  .map(grep('^HOST='))
  .map(cut({ delimiter: '=', fields: 2 }))
  .map(console.log)
  .catchMap(show_error);
```

Sí, lo sé, todo es muy bonito y útil pero en algún momento van a querer sacar el valor del `Result`. Entiendo, javascript no es un lenguaje hecho para este tipo de cosas, van a querer "volver a la normalidad". Agregaremos una función que nos de la libertad de extraer el valor en cualquier caso.

```diff
  const Result = {};
 
  Result.Ok = function(value) {
    return {
      map: fn => Result.Ok(fn(value)),
      catchMap: () => Result.Ok(value),
+     cata: (error, success) => success(value)
    };
  }
 
  Result.Err = function(value) {
    return {
      map: () => Result.Err(value),
      catchMap: fn => Result.Err(fn(value)),
+     cata: (error, success) => error(value)
    };
  }
```

Con esto podremos elegir qué hacer al final de la operación.

```js
const constant = arg => () => arg;
const identity = arg => arg;

const host = safer_cat('what?')
  .map(grep('^HOST='))
  .map(cut({ delimiter: '=', fields: 2 }))
  .cata(constant("This ain't right"), identity)

// ....
```

> Nota: Si se preguntan por qué `cata`, viene de la palabra **catamorfismo**, otro de esos términos de teoría de categoría que algunos usan en el paradigma funcional.

Ahora vamos a crear una estructura que nos permita resolver el problema que tenemos con `grep`. En este caso lo que tenemos que hacer es manejar la ausencia de un valor.

```js
const Maybe = function(value) {
  if(value == null) {
    return Maybe.Nothing();
  }

  return Maybe.Just(value);
}

Maybe.Just = function(value) {
  return {
    map: fn => Maybe.Just(fn(value)),
    catchMap: () => Maybe.Just(value),
    cata: (nothing, just) => just(value)
  };
}

Maybe.Nothing = function() {
  return {
    map: () => Maybe.Nothing(),
    catchMap: fn => fn(),
    cata: (nothing, just) => nothing()
  };
}

Maybe.wrap_fun = function(fn) {
  return function(...args) {
    return Maybe(fn(...args));
  }
}
```

Vamos a envolver `grep` con un `Maybe` y probaremos si funciona usando el `cat` original para extraer el contenido del archivo.

```js
const maybe_host = Maybe.wrap_fun(grep('^HOST='));

maybe_host(cat('.env'))
  .map(console.log)
  .catchMap(() => console.log('Nothing()'));
```

Eso debería mostrar `http://locahost:5000`. Y si cambian el patrón `^HOST=` debería mostrar `Nothing()`.

Tenemos versiones más seguras de `cat` y `grep` pero vean lo que pasa cuando se juntan.

```js
safer_cat('.env')
  .map(maybe_host)
  .map(res => console.log({ res }));
  .catchMap(() => console.log('what?'))
```

Obtienen esto.

```
{
  res: {
    map: [Function: map],
    catchMap: [Function: catchMap],
    cata: [Function: cata]
  }
}
```

¿Qué está pasando? Bueno, hay un `Maybe` atrapado dentro de un `Result`. Tal vez ustedes no esperaban eso pero otras personas sí, y ellas ya tienen las solución.

- Monads

Resulta que los monads son functors con poderes extra. Lo que nos interesa saber por el momento es que resuelven el problema de las estructuras anidadas. Hagamos los ajustes pertinentes.

```diff
  Result.Ok = function(value) {
    return {
      map: fn => Result.Ok(fn(value)),
      catchMap: () => Result.Ok(value),
+     flatMap: fn => fn(value),
      cata: (error, success) => success(value)
    };
  }

  Result.Err = function(value) {
    return {
      map: () => Result.Err(value),
      catchMap: fn => Result.Err(fn(value)),
+     flatMap: () => Result.Err(value),
      cata: (error, success) => error(value)
    };
  }
```

```diff
  Maybe.Just = function(value) {
    return {
      map: fn => Maybe.Just(fn(value)),
      catchMap: () => Maybe.Just(value),
+     flatMap: fn => fn(value),
      cata: (nothing, just) => just(value),
    };
  }

  Maybe.Nothing = function() {
    return {
      map: () => Maybe.Nothing(),
      catchMap: fn => fn(),
+     flatMap: () => Maybe.Nothing(),
      cata: (nothing, just) => nothing(),
    };
  }
```

El método `flatMap` además de comportarse como `map` nos permite deshacernos de "capas" extras que pueden complicar la composición más adelante. Asegúrense de usar `flatMap` sólo con funciones que retornen otros monads ya que esta no es la implementación más segura.

> Nota: Sí, los arreglos también son monads. Tienen los métodos `map` y `flatMap` que siguen todas las leyes.

Probamos otra vez con `maybe_host`.

```js
 safer_cat('.env')
  .flatMap(maybe_host)
  .map(res => console.log({ res }));
  .catchMap(() => console.log('what?'))
```

Eso debería darnos.

```
{ res: 'HOST=http://localhost:5000' }
```

Estamos listos para combinar todo nuevamente.

```js
const safer_cat = Result.make_safe(cat);
const maybe_host = Maybe.wrap_fun(grep('^HOST='));
const get_value = Maybe.wrap_fun(cut({delimiter: '=', fields: 2}));

const host = safer_cat('.env')
  .flatMap(maybe_host)
  .flatMap(get_value)
  .cata(
    () => 'http://127.0.0.1:3000',
    host => host
  );

// ....
```

¿Y cómo sería si quisiéramos usar `pipe` o `compose`?

```js
const chain = fn => m => m.flatMap(fn);
const unwrap_or = fallback => fm => 
  fm.cata(() => fallback, value => value);


const safer_cat = Result.make_safe(cat);
const maybe_host = Maybe.wrap_fun(grep('^HOST='));
const get_value = Maybe.wrap_fun(cut({delimiter: '=', fields: 2}));

const get_host = pipe(
  safer_cat,
  chain(maybe_host),
  chain(get_value),
  unwrap_or('http://127.0.0.1:3000')
);

get_host('.env');
```

Pueden ver todo el código aquí: [link](https://gist.github.com/VonHeikemen/0e6d4950bfe91229ee06eee2e3c74515).

## ¿Todavía quieren saber más?

Hay muchas cosas que no mencioné para no tomar mucho de su tiempo. Si quieren indagar un poco más aquí les dejo más material que he preparado.

- [Aplicacion parcial](@/web-development/fp-in-js/partial-application.es.md)
- [El poder de map (más sobre functors)](@/web-development/fp-in-js/the-power-of-map.es.md)
- [Usando un Maybe](@/web-development/fp-in-js/using-a-maybe.es.md)
- [Funciones puras y efectos](@/web-development/fp-in-js/dealing-with-side-effects-and-pure-functions.es.md)

## Conclusión

Muchas personas hablan de lo lindo que es la composición y cómo hace tu código más declarativo y limpio, pero nunca te muestran el lado difícil. Espero haber logrado eso, enseñarles un poco del lado difícil y cómo se puede superar. Combinar funciones en realidad es un arte, se requiere de práctica y tiempo para acostumbrarse a ciertas cosas (como que las funciones son cosas).

## Fuentes

- [The Power of Composition (video)](https://www.youtube.com/watch?v=vDe-4o8Uwl8)
- [Oh Composable World! (video)](https://www.youtube.com/watch?v=SfWR3dKnFIo)
- [Mary had a little lambda (video)](https://www.youtube.com/watch?v=7BsfMMYvGaU)
- [Functional JavaScript - Functors, Monads, and Promises](https://dev.to/joelnet/functional-javascript---functors-monads-and-promises-1pol)
