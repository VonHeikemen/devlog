+++
title = "Un poco del paradigma funcional en tu javascript: Aplicación parcial"
description = "Resolviendo el misterio de la función como primer parámetro"
date = 2020-03-10
lang = "es"
[taxonomies]
tags = ["javascript", "paradigma-funcional", "aprendizaje"]
[extra]
canonical_url = "https://dev.to/vonheikemen/un-poco-del-paradigma-funcional-en-tu-javascript-aplicacion-parcial-41kl"
shared = [
  ["dev.to", "https://dev.to/vonheikemen/un-poco-del-paradigma-funcional-en-tu-javascript-aplicacion-parcial-41kl"],
  ["Hashnode", "https://vonheikemen.hashnode.dev/partial-application-es"]
]
+++

Hoy vamos a resolver un misterio, el misterio de porque algunas personas crean funciones que aceptan una (otra) función como primer parámetro. Ya deben estar pensando que la respuesta es aplicación parcial y tienen razón en cierta parte, pero la aplicación parcial sólo es el medio para un fin, la verdadera razón de esto es para hacer posible una "mejor" composición de funciones. Pero antes de adentrarnos en los detalles técnicos de la aplicación parcial vamos a explorar la manera en la que hacemos las cosas actualmente.

## Como hacemos las cosas

Cuando creamos una función usualmente ordenamos los parámetros basados en un sistema de prioridad/importancia, donde el más importante va primero. Como resultado, cuando trabajamos sobre un dato este es el primero en la lista, le siguen los parámetros de configuración y por último dejamos los parámetros opcionales que podemos omitir.

Pongamos a prueba esa teoría. Digamos que queremos crear una función que pueda extraer unas propiedades específicas de un objeto plano. Pensemos en lo que necesitamos. ¿El objeto, eso en lo primero que pensaron? Es natural, no queremos omitirlo por accidente cuando ejecutemos la función. Eso deja la lista de propiedades como último parámetro. 


```js
function pick(obj, keys) {
  let result = {};
  
  for(key of keys) {
    result[key] = obj[key];
  }
  
  return result;
}
```

> Nota: No somos lo únicos que pensamos de esta manera, echen un vistazo a [pick de lodash](https://lodash.com/docs/#pick)

Ahora digamos que tenemos un objeto `user` y queremos esconder cualquier información "sensible". Lo haríamos de esta manera.

```js
const user = {
  id: 7,
  name: "Tom",
  lastname: "Keen",
  email: "noreply@example.com",
  password: "hudson"
};

pick(user, ['name', 'lastname']); 

// { name: "Tom", lastname: "Keen" }
```

Funciona bien, ¿Pero qué pasa cuando necesitamos trabajar con un arreglo de usuarios?

```js
const users = [
  {
    id: 7,
    name: "Tom",
    lastname: "Keen",
    email: "noreply@example.com",
    password: "hudson"
  },
  {
    id: 30,
    name: "Smokey",
    lastname: "Putnum",
    email: "noreply@example.com",
    password: "carnival"
  },
  {
    id: 69,
    name: "Lady",
    lastname: "Luck",
    email: "noreply@example.com",
    password: "norestforthewicked"
  }
];
```

Nos vemos forzados a recorrer el arreglo y llamar la función.

```js
users.map(function(user) {
  return pick(user, ['name', 'lastname']);
});

/*
[
  {"name": "Tom", "lastname": "Keen"},
  {"name": "Smokey", "lastname": "Putnum"},
  {"name": "Lady", "lastname": "Luck"}
]
*/
```

No está tan mal. ¿Saben qué? Esa función parece útil. Vamos ponerla en otro lugar y le daremos un nombre.

```js
function public_info(user) {
  return pick(user, ['name', 'lastname']);
}

users.map(public_info);
```

¿Qué está pasando en realidad? Lo que estamos haciendo es vincular el segundo parámetro de la función con el valor `['name', 'lastname']` y obligamos a `pick` a esperar por el objeto `user` para ser ejecutado.

Llevemos este ejemplo más allá. Vamos a fingir que `Async/Await` no existe y que el arreglo `users` viene de una promesa (de una instancia de `Promise`) tal vez de una petición http usando `fetch`. ¿Qué hacemos?

```js
fetch(url).then(function(users) {
  users.map(function(user) {
    return pick(user, ['name', 'lastname']);
  })
});
```

Eso sí se ve mal. Tal vez una función con flechas puedan mejorar la situación.

```js
fetch(url).then(users => users.map(user => pick(user, ['name', 'lastname'])));
```

¿Está mejor? Una pregunta para otro día. Pero ya nos preparamos para esto, tenemos la función `public_info`, vamos a usarla. 

```js
fetch(url).then(users => users.map(public_info));
```

Es aceptable, me gusta. Y si queremos podemos crear otra función que vincule `public_info` con `.map`.

```js
function user_list(users) {
  return users.map(public_info);
}
```

Ahora tenemos.

```js
fetch(url).then(user_list);
```

Veamos cómo llegamos a este punto.

```js
function pick(obj, keys) {
  // código...
}

function public_info(user) {
  return pick(user, ['name', 'lastname']);
}

function user_list(users) {
  return users.map(public_info);
}

fetch(url).then(user_list);
```

¿Y si les digo que hay otra manera de crear `public_info` y `user_list`? ¿Y si se pudiera crear así?

```js
const public_info = pick(['name', 'lastname']);
const user_list = map(public_info);

fetch(url).then(user_list);
```

O poner todo en una línea si eso prefieren.

```js
fetch(url).then(map(pick(['name', 'lastname'])));
```

Podemos hacerlo pero primero tenemos que cambiar ligeramente la forma en la que pensamos en las funciones.

## Pensando diferente

En lugar de pensar en prioridades deberíamos empezar a pensar en dependencias y datos. Al crear una función pensemos ¿qué parámetro es el que cambia con más frecuencia? Ese debería ser el último parámetro. 

Hagamos una función que tome los primeros elementos de algo. ¿Qué necesitamos? Necesitamos ese "algo" y también el necesitamos el número de elementos que vamos a tomar. De esos dos ¿cuál cambia con más frecuencia? Es el dato, ese "algo".

```js
function take(count, data) {
  return data.slice(0, count);
}
```

En una situación normal esta es la forma de usarla.

```js
take(2, ['first', 'second', 'rest']);

// ["first", "second"]
```

Pero con un poco de magia (la cual será revelada pronto) podemos reusarla de la siguiente manera.

```js
const first_two = take(2);

first_two(['first', 'second', 'rest']);
```

Este patrón se vuelve más conveniente cuando hay funciones (callbacks) involucradas. Vamos a "revertir" los parámetros de `Array.filter` y veamos qué podemos hacer.

```js
function filter(func, data) {
  return data.filter(func);
}
```

Hagamos algo sencillo, vamos a excluir de un arreglo todos los valores que puedan ser interpretados como falsos.

```js
filter(Boolean, [true, '', null, 'that']);

// => [ true, "that" ]
```

Se ve bien, y puede ser incluso mejor se le añadimos algo de contexto.

```js
const exclude_falsey = filter(Boolean);

exclude_falsey([true, '', null, 'that']);
```

Espero que a estas alturas puedan ver las posibilidades que este patrón puede ofrecer. Existen librerías (como [Ramda](https://ramdajs.com/docs/)) que usan esta técnica para construir funciones complejas usando como bases funciones pequeñas de un sólo propósito.

Ya basta de hablar, ahora veamos como podemos lograr implementar esto.

## Este es el camino

Como todo en javascript hay mil maneras de lograr la misma meta, algunas son más convenientes que otras, y en ocasiones se requiere de magia para implementarlo. Empecemos.

### El vínculo mágico de bind

Resulta que no necesitamos hacer nada extraordinario para vincular valores a los parámetros de una función porque cada una ya cuenta con el método [bind](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Function/bind). Puede que la sintaxis no parezca tan conveniente como la mostré en los ejemplo pero está bastante cerca. Sólo se debe tener en cuenta que el primer parámetro de `Function.bind` es el "contexto", es decir el valor que tiene la palabra clave `this` dentro de una función. Este es su uso básico. 

```js
const exclude_falsey = filter.bind(null, Boolean);

exclude_falsey([true, '', null, 'that']);
```

### La magia interior

Este requiere de un poco de trabajo e involucra otra palabra clave, `arguments`. Lo que haremos será aprovechar el hecho de que `arguments` es una estructura parecida a un arreglo que tiene una propiedad `.length` con la cual podremos contar el número de parámetros que la función ha recibido, si es menos de los que necesitamos devolveremos nuevamente la función. ¿Suena confuso?

```js
function filter(func, data) {

  // Aquí empezamos a contar.
  if(arguments.length === 1) {
    // si .length es 1 eso significa que tenemos `func`
    // también significa que no tenemos `data`
    // asi que devolvemos una función que
    // recuerda el valor de `func` y espera por `data`
    return arg => filter(func, arg);
  }

  return data.filter(func);
}
```

Ahora es posible hacer esto.

```js
const exclude_falsey = filter(Boolean);

exclude_falsey([true, '', null, 'that']);
```

Y también.

```js
filter(Boolean, [true, '', null, 'that']);
```

¿No es genial?

### ¿Un enfoque simple?

Y por supuesto que siempre tenemos la opción de implementar `bind` nosotros mismos. Con la ayuda del operador de propagación (los `...`) podemos recoger los argumentos por pasos y simplemente aplicarlos a la función cuando sea momento de llamarla.

```js
function bind(func, ...first_args) {
  return (...rest) => func(...first_args, ...rest);
}
```

El primer paso es obtener la función y recoger una lista de parámetros, luego devolvemos una función que recolecta otra lista de parámetros y finalmente llamamos la función `func` con todo lo que tenemos.

```js
const exclude_falsey = bind(filter, Boolean);

exclude_falsey([true, '', null, 'that']);
```

Lo interesante de esto es que si revierten el orden de `first_args` con `rest` pueden crear una función que vincula los argumentos en el orden opuesto.

### No más magia

Con este pueda que tenga sentimientos encontrados pero la verdad es que esta la forma más simple.

```js
function filter(func) {
  return function(data) {
    return data.filter(func);
  }
}
```

Lo que es equivalente a esto.

```js
const filter = func => data => data.filter(func);
```

La idea es tomar un parámetro a la vez en funciones separadas. Basicamente, sigan devolviendo funciones hasta que tengan todos los parámetros que necesitan. Esto es lo que algunos llaman "currying". ¿Cómo se usa?

```js
const exclude_falsey = filter(Boolean);

exclude_falsey([true, '', null, 'that']);
```

Ese es un caso. Este es el otro.

```js
filter (Boolean) ([true, '', null, 'that']);
```

¿Ven ese par de paréntesis extra? Esa es la segunda función. Necesitan colocar un par por cada parámetro que tenga la función.

### Curry automático

Volviendo al tema de la magia, pueden "automatizar" el proceso de curry usando una función.

```js
function curry(fn, arity, ...rest) {
  if (arguments.length === 1) {
    // Adivina cuantos argumentos se necesitan
    // Esto no funciona todo el tiempo.
    arity = fn.length;
  }

  // ¿Tenemos lo que necesitamos?
  if (arity <= rest.length) {
    return fn(...rest);
  }

  // Ejecuta `curry.bind` con `fn`, `arity` y `rest` como argumentos
  // retorna una función que espera el resto
  return curry.bind(null, fn, arity, ...rest);
}
```

Con esto ya pueden transformar funciones ya existentes o crear nuevas que soporten el "curry" desde el inicio.

```js
const curried_filter = curry(filter);

const exclude_falsey = curried_filter(Boolean);

exclude_falsey([true, '', null, 'that']);
```

Ó.

```js
const filter = curry(function(func, data) {
  return data.filter(func); 
});
```

Eso es todo, amigos. Espero hayan disfrutado la lectura.

## Fuentes
- [Hey Underscore, You're Doing It Wrong! (video)](https://www.youtube.com/watch?v=m3svKOdZijA)
- [Partial Application in JavaScript](http://benalman.com/news/2012/09/partial-application-in-javascript/)


