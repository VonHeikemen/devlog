+++
title = "¿Qué son los applicative functors?" 
description = "Usemos javascript para descubrir qué es un applicative functor"
date = 2020-08-13
lang = "es"
[taxonomies]
tags = ["javascript", "paradigma-funcional"]
+++

Nuestra agenda para hoy será aprender qué son applicative functors (aplicativos de ahora en adelante) usando javascript. Sí, usaremos javascript. No me juzguen, es lo que sé. Al terminar este artículo sabrán cómo crearlos, cómo reconocerlos y les enseñaré un truco que puede o no resultarles útil alguna vez.

Bien, empecemos desde el principio.

## ¿Qué es un functor?

Desde un punto de vista "técnico" podemos decir que son un tipo de contenedor. Verán, la manera más fácil de implementar un functor es "envolviendo" un valor dentro de una estructura. Para interactuar con el valor dentro del functor se provee un método normalmente llamado `map`, este nos permite transformar el valor usando una función (un callback) y luego envuelve el nuevo valor otra vez en una estructura del mismo tipo.

Veamos qué puede hacer `map`. Para familiarizarnos con este tipo de estructuras voy a demostrar sus capacidades usando un tipo de dato común en javascript, los arreglos.

```js
const numbers = [1];
const plus_one = (number) => number + 1;

numbers.map(plus_one);
// [ 2 ]
```

¿Qué pasa aquí?

Tenemos un número dentro de un arreglo, luego usamos `map` para acceder a él y transformarlo usando una función, y después el nuevo valor que obtenemos es puesto nuevamente en un arreglo. Eso es todo. Eso es básicamente el comportamiento que debe tener un functor.

Ahora bien, los arreglos no son los únicos que siguen este patrón, en javascript tenemos otra estructura que actúa de la misma manera, la clase `Promise`. Con las promesas no tenemos un método `map` pero tenemos uno llamado `then`, no son exactamente iguales en cuestión de comportamiento pero se acerca lo suficiente.

```js
const number = Promise.resolve(1);
const plus_one = (number) => number + 1;

number.then(plus_one);
// Promise { <state>: "pending" }
// 2
```

Aquí ocurre la misma cosa, tenemos un valor dentro de una estructura (una promesa), tenemos un método que nos da acceso al valor (`then`) y finalmente el nuevo valor queda atrapado en una nueva instancia de la misma estructura.

Y ese es el patrón. Ya hemos cubierto todo lo que necesitamos saber sobre los functors por ahora. Si desean saber más detalles sobre ellos revisen este artículo: [El Poder de Map](@/web-development/learn-fp/the-power-of-map.es.md).

¿Listos para seguir?

## Applicatives

Resulta que los aplicativos son functors con caraterísticas extras. Ellos nos dan la habilidad de mezclar dos functors. Especificamente, nos permiten aplicar una función dentro de un functor a un valor que también esta dentro de un functor.

Espera... ¿qué? ¿Una función dentro de un functor?

Sí. Algo así.

```js
const plus_one = (number) => number + 1;

// Y luego

[plus_one];

// Ó

Promise.resolve(plus_one);
```

¿Por qué alguien haría eso?

Buena pregunta. La respuesta, nadie lo haría. Si hablamos de patrones comunes en javascript este no es uno de ellos. Eso no significa que los aplicativos no tenga un uso.

Volviendo a nuestra definición. Normalmente, si tenemos un valor y una función somos capaces de aplicar dicha función así: `una_función(un_valor)`. Eso no funcionaría si ambos están encerrados dentro de una estructura. Para "arreglar" eso, los aplicativos tienen un método llamado `ap` (apply abreviado) que se encarga de sacar la función y el valor de sus respectivas estructuras y aplicar la función.

Y es en este punto que me gustaría mostrarles un ejemplo de un tipo de dato que ya sigue las reglas de los aplicativos pero no se me ocurre ninguno. Pero no teman, tomemos esto como una oportunidad para hacer algo más.

## Crear un Aplicativo desde cero

Para no complicarnos mucho lo que haremos será crear una pequeña extensión de la clase `Promise`. Vamos a hacer que una promesa se comporte más como un functor aplicativo.

¿Por donde comenzamos?

- La meta

Lo que queremos hacer es retrasar ejecución de una promesa. Normalmente cuando se crea una promesa esta ejecuta la "tarea" asignada inmediatamente pero no queremos eso, esta vez queremos controlar cuando se ejecuta la tarea. Para lograr nuestro objetivo crearemos un método llamado `fork`, este se encargará de crear la promesa y preparar las funciones en caso de éxito y error.

```js
function Task(proc) {
  return {
    fork(err, success) {
      const promise = new Promise(proc);
      return promise.then(success).catch(err);
    }
  }
}
```

Genial. Ahora vamos comparar esto con una promesa normal.

```js
let number = 0;
const procedure = function(resolve, reject) {
  const look_ma = () => {
    console.log(`IT WORKED ${++number} times`);
    resolve();
  };

  setTimeout(look_ma, 1000);
};

new Promise(procedure); // Esta se ejecuta inmediatamente

Task(procedure); // Esta no hace nada
Task(procedure)  // Esta sí
  .fork(
    () => console.error('AAHHH!'),
    () => console.log('AWW')
  );
```

Si ejecutan ese código deberían ver estos mensajes después de 1 segundo.

```
IT WORKED 1 times
IT WORKED 2 times
AWW
```

Ahora que tenemos lo que queremos vamos al siguiente paso.

- Haz un functor

Como ya saben los aplicativos son functors, significa que ahora necesitamos un método `map`.

Repasemos una vez más. ¿Cuál es el comportamiento que esperamos de `map`?

1. Debería darnos acceso al valor almacenado internamente a través de una función.
2. Debería retornar un nuevo contenedor del mismo tipo. En nuestro caso una nueva instancia de `Task`.

```diff
  function Task(proc) {
    return {
+     map(fn) {
+       return Task(function(resolve, reject) {
+         const promise = new Promise(proc);
+         promise.then(fn).then(resolve).catch(reject);
+       });
+     },
      fork(err, success) {
        const promise = new Promise(proc);
        return promise.then(success).catch(err);
      }
    }
  }
```

¿Qué pasa en `map`? Bueno, primero recibimos el argumento `fn` esa será una función. Luego, retornamos una instancia de `Task`. Dentro de esa nueva instancia construimos la promesa justo como hacemos en `fork` pero esta vez es más "seguro" porque no se ejecutará inmediatamente. El siguiente paso es colocar las funciones que requiere `promise` en su respectivo orden, primero `fn` la cual transformará el valor, luego `resolve` que marca el "fin" de la tarea actual y finalmente `catch` que recibirá la función `reject` de la tarea actual.

Podemos probar lo que tenemos hasta ahora.

```js
const exclaim = (str) => str + '!!';
const ohh = (value) => (console.log('OOHH'), value);

Task((resolve) => resolve('hello'))
  .map(exclaim)
  .map(ohh)
  .fork(console.error, console.log);
```

Si ejecutan eso tal cual como esta deberían ver esto.

```
OOHH
hello!!
```

Pero si eliminan `fork` deberían tener esto.

```
```

Sí, así es, deberían tener absolutamente nada. Ya hemos terminado con el patrón functor de nuestro `Task`.

- Vamos por Apply

Ya estamos a medio camino. Lo que haremos ahora será que crear `ap`.

Como yo lo veo `ap` es `map` pero con un giro en la trama: la función que queremos aplicar está dentro de una instancia de `Task` [*música dramática suena en el fondo*].

Con esa idea en la mente podemos implementar `ap`.

```diff
  function Task(proc) {
    return {
      map(fn) {
        return Task(function(resolve, reject) {
          const promise = new Promise(proc);
          promise.then(fn).then(resolve).catch(reject);
        });
      },
+     ap(Fn) {
+       return Task(function(resolve, reject) {
+         const promise = new Promise(proc);
+         const success = fn => promise.then(fn);
+         Fn.fork(reject, success).then(resolve);
+       });
+     },
      fork(err, success) {
        const promise = new Promise(proc);
        return promise.then(success).catch(err);
      }
    }
  }
```

¿Notan la diferencia con `map`? No se preocupen igual les diré, la diferencia es que para aplicar la función en `Fn` usamos `fork` en lugar de interactuar con una promesa normal. Eso es todo. Veamos si funciona.

```js
const to_uppercase = (str) => str.toUpperCase();
const exclaim = (str) => str + '!!';

const Uppercase = Task((resolve) => resolve(to_uppercase));
const Exclaim = Task((resolve) => resolve(exclaim));
const Hello = Task((resolve) => resolve('hello'));

Hello.ap(Uppercase).ap(Exclaim)
  .fork(console.error, console.log);
```

¡Lo logramos! Ahora podemos mezclar funciones que se encuentren dentro de aplicativos. Pero `Task` aún no puede entrar en el club de los aplicativos. Tenemos que ocuparnos de otra cosa primero.

- El ingrediente olvidado

Los aplicativos deben ser capaces de colocar cualquier valor dentro de la unidad más simple de su estructura.

La clase `Promise` tiene algo así. En lugar de hacer esto.

```js
new Promise((resolve) => resolve('hello'));
```

Usualmente hacemos esto.

```js
Promise.resolve('hello');
```

Luego de usar `Promise.resolve` podemos empezar a usar métodos como `then` y `catch`. Eso es lo que le hace falta a nuestro `Task`.

Para esta implementar esto necesitaremos un método estático. Para esto existen varios nombres, algunos lo llaman "pure" otros lo llaman "unit" y también están quienes lo llaman "of".


```js
Task.of = function(value) {
  return Task((resolve) => resolve(value));
};
```

Y ahora sí, finalmente podemos decir que tenemos un aplicativo.

## Algo que puedes usar en tu desarrollo diario

Poder crear tu propio tipo de dato es genial, ¿pero no sería mejor si pudiéramos aplicar estos patrones con las estructuras que ya existen?

Tengo buenas y malas noticias. La buena es que definitivamente podemos. La mala es que a veces puede ser incómodo.

Sigamos con el ejemplo de `Task` que hemos usado hasta ahora. Pero ahora digamos que queremos usar `map` y `ap` pero no queremos crear una nueva estructura. ¿Qué hacemos? Un par de funciones serán suficientes.

Si ya están familiarizados con los patrones que buscan, escribirlos en unas funciones estáticas bastará. Así luciría nuestro `Task` como funciones simples.

```js
const Task = {
  of(value) {
    return Promise.resolve(value);
  },
  map(fn, data) {
    return data.then(fn);
  },
  ap(Fn, data) {
    return Fn.then(fn => data.then(value => fn(value)));
  }
};
```

Para usar `map` sería así.

```js
const to_uppercase = (str) => str.toUpperCase();

Task.map(to_uppercase, Task.of('hello'))
  .then(console.log);
```

Y `ap` funciona de la misma manera.

```js
const exclaim = (str) => str + '!!';

Task.ap(Task.of(exclaim), Task.of('hello'))
  .then(console.log);
```

Puedo percibir su escepticismo desde aquí. Sean pacientes. Ahora, `map` parece medio útil pero `ap` no tanto. No se preocupen, aún podemos usar `ap` para un bien mayor. ¿Y si les digo que podemos tener una versión "mejorada" de `map`? Nuestro `map` sólo trabaja con funciones que reciben un argumento y eso es bueno pero puede haber ocasiones en que necesitemos más que eso.

Digamos que tenemos una función que recibe dos argumentos pero en su mayoría los argumentos casi siempre vienen de dos promesas diferentes. Así que, en nuestra situación imaginaria tenemos estas funciones.

```js
function get_username() {
  return new Promise((resolve) => {
    const fetch_data = () => resolve('john doe'); 
    setTimeout(fetch_data, 1000);
  });
}

function get_location() {
  return new Promise((resolve) => {
    const fetch_data = () => resolve('some place'); 
    setTimeout(fetch_data, 500);
  });
}

function format_message(name, place) {
  return `name: ${name} | place: ${place}`;
}
```

Cuando usamos `format_message` sus argumentos vienen de las otras dos funciones `get_username` y `get_location`. Esas últimas dos son asíncronas, entonces tal vez se vean tentados a usar las palabras clave `Async/Await` pero esa no sería una buena idea. Verán, esas funciones no depende la una de la otra, estaríamos desperdiciando tiempo si hacemos que se ejecuten en secuencia cuando deberían ejecutarse de manera concurrente. Una solución puede encontrarse en la forma de `Promise.all` y se ve así.

```js
Promise.all([get_username(), get_location()])
  .then(([name, place]) => format_message(name, place))
  .then(console.log);
```

Ahí lo tienen. Eso funciona. Pero nosotros podemos hacer algo mejor, ya que tenemos los aplicativos de nuestro lado. Además, ya tenemos ese objeto `Task`. Ahora sólo vamos a agregar una función más, esta hará lo mismo que está haciendo `Promise.all`.

```js
Task.liftA2 = function(fn, A1, A2) {
  const curried = a => b => fn(a, b);
  return Task.ap(Task.map(curried, A1), A2);
};
```

Ya les explico el nombre después. Ahora veamos cómo se usa.

```js
Task.liftA2(format_message, get_username(), get_location())
  .then(console.log);
```

¿No les parece que esto es un poco mejor?

Y sí, es cierto que pueden presentar argumentos contra la implementación de `liftA2` e incluso todo el objeto `Task`, pero todos los patrones que he mostrado aquí deberían funcionar para los aplicativos que puedan encontrarse por ahí.

Como un ejercicio pueden intentar implementar `map` y `ap` para la clase [Set](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Set). Vean que clase de cosas graciosas descubren en el proceso. 

En fin, el nombre `liftA2`. En el paradigma funcional cuando logramos que una función trabaje con un tipo de contenedor como lo son los functors se dice que "levantamos" (`lift` en inglés) esa función al contexto de ese contenedor. ¿Qué significa eso de contexto? Bueno, en el mundo de los arreglos la función que le proveen a `map` puede ejecutarse muchas veces (o ninguna), en el contexto de una promesa la función que suministran a `then` sólo se ejecuta cuando la promesa culmina su tarea de manera exitosa. ¿Ya ven lo que digo? Bien. ¿Y el `A2`? Ya saben, es porque recibe sólo dos argumentos.

Hay otro truco que se puede hacer con los aplicativos pero aún no entiendo completamente cómo funciona así que será en otra ocasión.

## Conclusión

¿Qué aprendimos hoy, clase?

- Aprendimos de los functors:
  * Qué hacen.
  * Qué patrones deben seguir.
- Aprendimos de los aplicativos
  * Qué son.
  * Qué hacen.
  * Cómo crear uno desde cero.
  * Cómo hacer un método `ap` aún si la estructura con la trabajamos no tiene soporte para el patrón de los aplicativos.
  * Y esa cosa `liftA2` que se ve genial.

¿Aprendieron todo eso? Dios santo. Ustedes son los mejores.

Bueno, mi trabajo aquí ha terminado.

## Fuentes

- [Fantasy Land](https://github.com/fantasyland/fantasy-land)
- [Static Land](https://github.com/fantasyland/static-land)
- [Fantas, Eel, and Specification 8: Apply](http://www.tomharding.me/2017/04/10/fantas-eel-and-specification-8/)
- [Fantas, Eel, and Specification 9: Applicative](http://www.tomharding.me/2017/04/17/fantas-eel-and-specification-9/)
- [Professor Frisby's Mostly Adecuate Guide to Functional Programming. Chapter 10: Applicative Functors](https://mostly-adequate.gitbooks.io/mostly-adequate-guide/ch10.html)
- [Learn you a Haskell: Functors, Applicative Functors and Monoids](http://learnyouahaskell.com/functors-applicative-functors-and-monoids#applicative-functors)

