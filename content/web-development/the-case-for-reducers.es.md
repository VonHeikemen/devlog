+++
title = "La utilidad de los reducers"
description = "Los reducers son buenos para muchas cosas, vamos a descubrir qué son esas cosas"
date = 2020-11-27
lang = "es"
[taxonomies]
tags = ["javascript"]
[extra]
shared = [
  ["dev.to", "https://dev.to/vonheikemen/la-utilidad-de-los-reducers-4apa"],
  ["Hashnode", "https://vonheikemen.hashnode.dev/the-case-for-reducers-es"]
]
+++

En un [artículo anterior](@/web-development/learn-fp/reduce-how-and-when.md) les hablé del método `.reduce`, cómo funciona y el caso ideal en el que podemos usarlo (en mi opinion). Esta vez voy a mostrar más casos en los que podría ser una buena opción. Ahora bien, no tienen que haber leído ese artículo pero de aquí en adelante voy a asumir que saben cómo funciona el método `Array.reduce`. Al finalizar espero que aprendan a reconocer en qué lugares `.reduce` podría funcionar perfectamente.


## ¿Qué estamos buscando?

Patrones, buscamos patrones. Bueno... sólo uno. Y para saber qué estamos buscando tenemos que ver los requerimientos de un `reducer`. Piensen en ellos por un momento, cuando empiezan a escribir uno que quieren usar con `Array.reduce` tal vez luce así.

```js
function (accumulator, value) {
  /*
    algo de lógica por aquí
  */
  return accumulator;
}
```

Okey, por lo general retornamos una copia modificada de `accumulator` pero eso no es importante, el punto es que retornamos el mismo "tipo" de dato que obtuvimos en el primer parámetro. Entonces tenemos que la **comportamiento de la función** es el siguiente.

```
(Accumulator, Value) -> Accumulator
```

Pero en este caso lo que tenemos aquí es un ejemplo concreto. Yo quiero que vean esto de una forma más abstracta. Lo que en realidad buscamos son funciones con esta forma.

```
(A, B) -> A
```

Eso es básicamente todo lo que necesitan saber. Para que un `reduce` pueda hacer bien su trabajo sólo debe ser capaz de retornar el mismo tipo de dato que recibió en el primer parámetro.

¿Aún están confundidos? No se preocupen, pasaremos lo que queda de este artículo revisando ejemplos donde este patrón puede aparecer.

## Casos de uso

### Acumuladores

Aquí generalmente es la parte donde les muestro una situación donde sumamos un arreglo de números o algo parecido. No hagamos eso. Podemos imaginarnos un escenario más complejo donde un acumulador nos sea útil.

Entonces, vamos a fingir que estamos trabajando en un proyecto que tiene una especie de blog y nos encontramos creando la página del perfil de usuario. Queremos mostrar todos los tags donde el usuario tiene por lo menos un artículo. Quizá quieran extraer ese dato de la base de datos usando una consulta elaborada pero eso llevaría mucho tiempo. Primero hagamos un prototipo.

Antes de hacer las cosas de la manera adecuada lo que vamos a hacer es transformar un arreglo que contiene todos los artículos en un [Set](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Set) con todos los tags, para eso usaremos `Array.reduce`.

```js
// Imaginen que estos objetos son más complejos
const posts = [
  { tags: ["javascript", "discuss"] },
  { tags: ["javascript", "react", "vue-is-better"] },
  { tags: ["discuss"] },
  { tags: ["javascript"] },
];

function dangerously_add_tags(acc, post) {
  for(let value of post.tags) {
    acc.add(value);
  }

  return acc;
}

posts.reduce(dangerously_add_tags, new Set());
```

Este sería el resultado.

```
Set(4) [ "javascript", "discuss", "react", "vue-is-better" ]
```

Ahora piensen en el comportamiento de nuestro `reducer`. Tenemos un `Set` con tags que hace el papel de `Accumulator` y un objeto que representa un post como nuestro `Value`. Podríamos decir que se comporta de la siguiente manera.

```
(Set, Objeto) -> Set
```

Bueno, técnicamente `Objeto` no puede ser cualquier objeto, debe tener una propiedad llamada `tags`. Así que sería algo más como esto.

```
(Set, Artículo) -> Set
```

En fin, este es el patrón del que les hablaba `(A, B) -> A`. La implementación de `dangerously_add_tags` demanda que `B` sea un `Artículo`. Pero para que esta función sea un `reducer` debe ser capaz de retornar el mismo tipo de dato que recibió en el primer parámetro (`Set`), y eso lo logramos retornando `acc`.

### Transformaciones

Probablemente han escuchado que pueden usar `Array.reduce` para suplantar otros métodos del prototipo `Array`, pero aunque esto suene como un dato interesante no tiene mucha utilidad. ¿Por qué harían algo así? No tiene sentido para mí. Sin embargo, aún puede resultar útil si planean "fusionar" las características de varios de esos métodos en uno. ¿Alguna vez han querido filtrar y transformar un arreglo al mismo tiempo? Con `.reduce` eso es posible.

Vamos a reusar nuestra variable `posts` aquí también.

```js
const posts = [
  {
    category: "javascript",
    tags: ["javascript", "discuss"]
  },
  {
    category: "frameworks",
    tags: ["javascript", "react", "vue-is-better"]
  },
  {
    category: "watercooler",
    tags: ["discuss"]
  },
  {
    category: "functional programming",
    tags: ["javascript"]
  },
];
```

Esta vez lo que queremos hacer es filtrar los que tengan el tag `discuss`, y por cada uno que pase la prueba queremos extraer la categoría y capitalizar el valor. ¿Cómo haríamos eso?

```js
function capitalize(str) {
  return str[0].toUpperCase() + str.slice(1);
}

function filter_map_posts(acc, post) {
  // aquí estamos filtrando
  if(post.tags.includes('discuss')) {
    return acc.concat(
      // esta es la transformación
      capitalize(post.category)
    );
  }
	
  return acc;
}

posts.reduce(filter_map_posts, []);
```

Aquí tenemos nuestro resultado.

```
Array [ "Javascript", "Watercooler" ]
```

¿Por qué funciona? Si revisan el comportamiento de `filter_map_posts` tenemos esto.

```
(Arreglo, Artículo) -> Arreglo
```

### Coordinación

Si han indagado un poco en librerías enfocadas en el paradigma funcional hay una alta probabilidad de que se hayan topado con una función llamada `pipe`. Con esta función podemos combinar una cantidad arbitraria de funciones. Esta es la manera en la que se usa.

```js
pipe(
  una_funcion,
  otra,
  proceso_serio,
  efectos_adelante,
);
```

La idea detrás de esto se centra en transportar el resultado de una función hacia la siguiente en la lista. En efecto lo que hacemos aquí es coordinar llamadas de funciones. En este caso, el fragmento anterior es equivalente a esto:

```js
function pipe(arg) {
  return efectos_adelante(proceso_serio(otra(una_funcion(arg))));
}
```

Si se preguntan por qué les digo esto, es porque podemos implementar `pipe` usando `.reduce`. Si se fijan bien notarán que lo único que hacemos en esa función es aplicar funciones a un argumento. Eso es todo. No hay nada más.

¿Y qué?

¡Es una operación binaria! Podemos transformar eso en una función.

```js
function apply(arg, fn) {
  return fn(arg);
}
```

¿Y saben qué funciona bien con operaciones binarias? Nuestro amigo `.reduce`.

```js
function pipe(...fns) {
  return function(some_arg) {
    return fns.reduce(apply, some_arg);
  };
}
```

Lo primero que hacemos en `pipe` es recolectar la lista de funciones que usaremos y convertirla en un arreglo. El segundo paso es retornar una función que desencadenará las llamadas a las funciones en nuestro arreglo, también en este paso obtenemos nuestro argumento inicial. Al final de eso, con todo en su lugar, `.reduce` se encarga del resto. Pueden probarlo ustedes mismos.

```js
const post = { 
  category: "javascript",
  tags: ["javascript", "discuss"] 
}

function capitalize(str) {
  return str[0].toUpperCase() + str.slice(1);
}

function get_prop(key) {
  return function(obj) {
    return obj[key];
  }
}

function exclaim(str) {
  return str + "!!";
}

const exciting_category = pipe(
  get_prop("category"),
  capitalize,
  exclaim
);

exciting_category(post);
// => Javascript!!
```

Okey, okey. Ahora bien, ¿Cómo es que `apply` sigue el patrón?

Ah, buena pregunta. Es algo raro pero aún podemos darle sentido. Mírenlo de esta manera.

```
(Algo, Función) -> Algo
```

Si tienen una unidad de lo que sea (literalmente cualquier cosa) y una función, `apply` hará su trabajo. Pero tengan presente que aquí no hay garantía de que su función no explote, esa sería responsabilidad de ustedes.

### Cambios de estado a través del tiempo

Este bonus track es para todos aquellos desarrolladores frontend allí afuera.

Si han pasado cualquier cantidad de tiempo investigando sobre librerías para manejar el estado de una aplicación tal vez hayan escuchado de una cosa llamada [redux](https://redux.js.org/). Esta librería tiene un enfoque interesante porque espera que el usuario (el desarrollador) suministre un `reducer` que sea capaz de manejar los cambios en el estado de la aplicación. A algunos les parece genial, a otros no. Pero bien sea que estén de acuerdo con esto o no, su enfoque tiene mucho sentido. Déjenme mostrarles. 

Empecemos con el `reducer`. En esta ocasión necesitamos uno con este comportamiento.

```
(Estado, Acción) -> Estado
```

`Estado` y `Acción` son objetos. Aquí no hay nada extravagante. La "forma" de nuestro `Estado` depende de la aplicación en la que trabajamos, los desarrolladores pueden hacer lo que quieran con él. La `Acción` por otro lado debe tener una propiedad `type`, y `redux` se asegura de esto.

Entonces, vamos a fingir que este es el estado de una aplicación imaginaria en la que estamos trabajando.

```js
const state = {
  count: 40,
  flag: false
};
```

Ah, sí. Un milagro de la ingeniería.

Ahora que sabemos cómo luce el `Estado`, y también sabemos qué necesita una `Acción`, podemos comenzar a escribir nuestro `reducer`.

```js
function reducer(state, action) {
  switch(action.type) {
    case 'add':
      return {
        ...state,
        count: state.count + 1,
      };
    case 'subtract':
      return {
        ...state,
        count: state.count - 1,
      };
    case 'toggle_flag':
      return {
        ...state,
        flag: !state.flag,
      };
    default:
      return state;
  }
}
```

Aquí viene la parte curiosa: no necesitamos `redux` para probar nuestro `reducer`. Es un `reducer` genérico, bien podríamos usarlo con `Array.reduce` para ver qué puede hacer.

```js
const actions = [
  { type: 'add' },
  { type: 'add' },
  { type: 'subtract' },
  { type: 'add' },
  { type: 'subtract' },
  { type: 'add' },
  { type: 'toggle_flag' }
];

actions.reduce(reducer, state);
```

`actions.reduce` debería devolvernos otra "instancia" de nuestro estado. En nuestro caso, después de aplicar todas esas acciones, tendríamos el siguiente resultado.

```js
{
  count: 42,
  flag: true
}
```

Y allí lo tienen, la funcionalidad principal de `redux` sin `redux`.

Vamos a dar un paso adelante en nuestro proceso e introduzcamos el concepto de tiempo. Para esto vamos a agregar una tienda "falsa" de `redux`. Bueno... la tienda será "real" pero será una imitación barata. Empecemos.

```js
function Store(reducer, state) {
  let _listener = null;

  const get_state = function() {
    return state;
  };

  const subscribe = function(listener) {
    _listener = listener;
  };

  const dispatch = function(action) {
    state = reducer(state, action);
    _listener && _listener();

    return action;
  };

  return { get_state, dispatch, subscribe };
}
```

¿Todo bien? ¿Saben lo que ocurre ahí? La parte que realmente nos interesa es la del `dispatch`. Esto de aquí.

```js
const dispatch = function(action) {
  state = reducer(state, action);
  _listener && _listener();

  return action;
};
```

Esta función se encarga de reemplazar el `Estado` actual. Como mencioné antes, el `reducer` se encarga de la lógica que dice **cómo** actualizar el `Estado`. La tienda (`Store`) se encarga de la lógica que dice **cuando** debe actualizarse. Suficiente palabrería, vamos a probarlo.

```js
function delay(ms) {
  return new Promise(resolve => setTimeout(resolve, ms));
}

const store = Store(reducer, state);
store.subscribe(function() {
  console.log(store.get_state());
});

(async function() {
  store.dispatch({ type: 'add' });
  await delay(500);

  store.dispatch({ type: 'add' });
  await delay(500);

  store.dispatch({ type: 'subtract' });
  await delay(700);

  store.dispatch({ type: 'add' });
  await delay(400);

  store.dispatch({ type: 'subtract' });
  await delay(800);

  store.dispatch({ type: 'add' });
  await delay(100);

  store.dispatch({ type: 'toggle_flag' });
})();
```

Si ejecutan eso notarán cómo los mensajes aparecen en pantalla (o la cónsola del navegador) con una pequeña demora entre cada uno.

```
- { count: 41, flag: false }
- { count: 42, flag: false }
- { count: 41, flag: false }
- { count: 42, flag: false }
- { count: 41, flag: false }
- { count: 42, flag: false }
- { count: 42, flag: true }
```

¿Se dieron cuenta que el resultado final es el mismo que nos dio `Array.reduce`? ¿No es genial?

Si quieren jugar con el verdadero `redux` aquí les dejo un ejemplo en codepen.

{{ codepen(id="zYBgObJ", title="Using redux", load_js=true) }}

## Conclusión

Espero que en este punto los `reducers` no parezcan tan misteriosos y aterradores. Sólo recuerden que se trata de una función con este comportamiento.

```
(A, B) -> A
```

Eso es todo. No hay ninguna magia extraña detrás de eso. Si pueden hacer que una función tenga esas características pueden estar seguros que funcionará de maravilla con cualquier cosa que actue como `.reduce`.

## Fuentes
- [Array.prototype.reduce()](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/Reduce)
- [Reduce: how and when](@/web-development/learn-fp/reduce-how-and-when.md)
- [Redux: Store](https://redux.js.org/api/store)

