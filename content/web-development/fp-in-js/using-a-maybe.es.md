+++
title = "Un poco del paradigma funcional en tu javascript: Usando un Maybe"
description = "Veremos cómo afectaría nuestro código si utilizaramos una estructura conocida como Maybe"
date = 2019-10-29
lang = "es"
[taxonomies]
tags = ["javascript", "paradigma-funcional", "aprendizaje"]
[extra]
canonical_url = "https://dev.to/vonheikemen/un-poco-del-paradigma-funcional-en-tu-javascript-usando-un-maybe-4a33"
+++

¿Alguna vez han escuchado de las estructuras llamadas "monads" y lo geniales que son? Tal vez sí, pero aún no las entienden completamente. Bueno... aquí no intentaré definirlas y tampoco les diré si son geniales o no, lo que haré es mostrarles un ejemplo de cómo sería si las usaran en sus javascripts (específicamente el monad `Maybe`).

Haremos algo gracioso, resolveremos un problema trivial usando métodos innecesariamente complicados.

Supongamos que tenemos un diccionario guardado en un archivo .json o en un objeto plano en nuestro script.

```js
{
    "accident": ["An unexpected, unfortunate mishap, failure or loss with the potential for harming human life, property or the environment.", "An event that happens suddenly or by chance without an apparent cause."], 
    "accumulator": ["A rechargeable device for storing electrical energy in the form of chemical energy, consisting of one or more separate secondary cells.\\n(Source: CED)"],
    "acid": ["A compound capable of transferring a hydrogen ion in solution.", "Being harsh or corrosive in tone.", "Having an acid, sharp or tangy taste.", "A powerful hallucinogenic drug manufactured from lysergic acid.", "Having a pH less than 7, or being sour, or having the strength to neutralize  alkalis, or turning a litmus paper red."],
    
     // ... más palabras y significados
    
    "Paris": ["The capital and largest city of France."]
  }
```

Queremos crear un formulario que le permita a un usuario buscar uno de estos términos y luego muestre su signicado. Parece simple ¿Qué podría salir mal?

Y porque todo el mundo adora HTML empezaremos por ahí.

```html
<form id="search_form">
  <label for="search_input">Search a word</label>
  <input id="search_input" type="text">
  <button type="submit">Submit</button>
</form>

<div id="result"></div>
```

En nuestro primer intento sólo intentaremos obtener uno de esos valores basado en la consulta del usuario.

```js
// main.js

// haz magia y tráeme los datos
const entries = data();

function format(results) {
  return results.join('<br>');
}

window.search_form.addEventListener('submit', function(ev) {
  ev.preventDefault();
  let input = ev.target[0];
  window.result.innerHTML = format(entries[input.value]);
});
```

Naturalmente lo primero que haremos es probar con acid. Ahora contemplen los resultados.

> A compound capable of transferring a hydrogen ion in solution.
Being harsh or corrosive in tone.
Having an acid, sharp or tangy taste.
A powerful hallucinogenic drug manufactured from lysergic acid.
Having a pH less than 7, or being sour, or having the strength to neutralize alkalis, or turning a litmus paper red.

Ahora buscaremos "paris", estoy seguro que está ahí. ¿Qué obtuvimos? Nada. No exactamente, tenemos.

> TypeError: results is undefined

Pero tambíen tenemos un botón impredecible que se congela en ocasiones. ¿Pero qué queremos? ¿Qué queremos en realidad? Seguridad, objetos que no hagan estallar nuestra aplicación, queremos objetos confiables.

Entonces lo que haremos será implementar una especie de contenedor que nos permita describir el flujo de ejecución sin tener que preocuparnos por el valor que este contenga. ¿Suena bien, no? Déjenme mostrarles lo que quiero decir con un poco de javascript. Intenten esto.

```js
const is_even = num => num % 2 === 0;

const odd_arr = [1,3,4,5].filter(is_even).map(val => val.toString());
const empty_arr = [].filter(is_even).map(val => val.toString());

console.log({odd_arr, empty_arr});
```

¿Generó un error el arreglo vacío? (si lo hizo díganme). ¿No es genial? ¿No se siente bien saber que los métodos del arreglo harán lo correcto incluso si no tienen nada con qué trabajar? Eso es lo que queremos.

Tal vez se estén preguntando ¿No puedo simplemente poner un `if` y ya? Bueno... sí, ¿pero eso qué tiene de divertido? Todos saben que hacer una cadena de funciones se ve genial, y somos fánaticos de la "programación funcional," así que haremos lo que los conocedores de ese paradigma harían: **esconderemos todo dentro de una función**. 

Entonces lo que haremos será esconder un par de `if`, si el valor que debemos evaluar es indefinido devolveremos un contenedor que sabrá qué hacer sin importar lo que pase.

```js
// maybe.js

function Maybe(the_thing) {
  if(the_thing === null 
     || the_thing === undefined 
     || the_thing.is_nothing
  ) {
    return Nothing();
  }

  // No queremos estructuras anidadas.
  if(the_thing.is_just) {
    return the_thing;
  }

  return Just(the_thing);
}

```
Pero estos contenedores no serán los típicos `Maybe` que se ven en un lenguaje propio del paradigma funcional. Nosotros haremos trampa en el nombre de la conveniencia y los efectos secundarios. Sus métodos estaran inspirados por el tipo de dato `Option` que tiene Rust. Aquí es donde está la magia.

```js
// maybe.js

function Just(thing) {
  return {
    map: fun => Maybe(fun(thing)),
    and_then: fun => fun(thing),
    or_else: () => Maybe(thing),
    tap: fun => (fun(thing), Maybe(thing)),
    unwrap_or: () => thing,
    
    filter: predicate_fun => 
      predicate_fun(thing) 
        ? Maybe(thing) 
        : Nothing(),
    
    is_just: true,
    is_nothing: false,
    inspect: () => `Just(${thing})`,
  };
}

function Nothing() {
  return {
    map: Nothing,
    and_then: Nothing,
    or_else: fun => fun(),
    tap: Nothing,
    unwrap_or: arg => arg,

    filter: Nothing,

    is_just: false,
    is_nothing: true,
    inspect: () => `Nothing`,
  };
}
```

¿Qué hacen estos métodos?

- `map`: Aplica la función `fun` a `the_thing` y vuelve a colocarlo en un `Maybe` para mantener la forma del objeto, esto para que podamos encadenar más funciones.
- `and_then`: Este sólo está ahí para los casos de emergencia. Aplica la función `fun` y que el destino decida el resto.
- `or_else`: Este sería el complemento `else` para nuestro `map` y `and_then`. Es el otro camino. El "¿qué pasa si no hay nada ahí?"
- `tap`: Está ahí para cuando necesitemos una función que afecta algo que está fuera de su ámbito (o tal vez es sólo para colocar un `console.log`).
- `filter`: Si la función que proporcionas devuelve `true` o algo parecido entonces "te dejará pasar." 
- `unwrap_or`: Este es el que saca el valor del contenedor. Usarán esto cuando se cansen de encadenar funciones y estén listos para volver al mundo imperativo.

Volvamos a nuestro formulario para aplicar todo esto. Crearemos una función `search` que puede o no devolvernos un resultado a la consulta del usuario. Si lo hace encadenamos otras funciones que se ejecutarán en un "contexto seguro."

```js
// main.js

const search = (data, input) => Maybe(data[input]);

const search_word = word => search(entries, word)
  .map(format)
  .unwrap_or('word not found');
```

Ahora reemplazamos la antigua función.

```diff
 window.search_form.addEventListener('submit', function(ev) {
   ev.preventDefault();
   let input = ev.target[0];
-  window.result.innerHTML = format(entries[input.value]);
+  window.result.innerHTML = search_word(input.value);
 });
```

Probemos. Buscaremos "accident."

> An unexpected, unfortunate mishap, failure or loss with the potential for harming human life, property or the environment.
An event that happens suddenly or by chance without an apparent cause.

Ahora Paris. Busquemos "paris."

> word not found

No congeló el botón, eso es bueno. Pero yo sé que Paris está ahí. Si revisan verán que está "Paris." Sólo tendremos que colocar en mayúscula la primera letra para que el usuario no tenga que hacerlo. Primero intentaremos buscar la palabra exacta y luego intentamos del otro modo.

```js
// main.js

function create_search(data, exact) {
  return input => {
    const word = exact ? input : capitalize(input);
    return Maybe(data[word]);
  }
}

function capitalize(str) {
  return str.charAt(0).toUpperCase() + str.slice(1);
}
```

Modificamos la función `search`.

```diff
- const search = (data, input) => Maybe(data[input]);
+ const search = create_search(entries, true);
+ const search_name = create_search(entries, false);
-
- const search_word = word => search(entries, word)
+ const search_word = word => search(word)
+   .or_else(() => search_name(word))
    .map(format)
    .unwrap_or('word not found');
```

Bien. Esto es lo que tenemos hasta ahora en `main.js` si quieren ver todo el panorama.

```js
// main.js

const entries = data();

function create_search(data, exact) {
  return input => {
    const word = exact ? input : capitalize(input);
    return Maybe(data[word]);
  }
}

function capitalize(str) {
  return str.charAt(0).toUpperCase() + str.slice(1);
}

function format(results) {
  return results.join('<br>');
}

const search = create_search(entries, true);
const search_name = create_search(entries, false);

const search_word = word => search(word)
  .or_else(() => search_name(word))
  .map(format)
  .unwrap_or('word not found');

window.search_form.addEventListener('submit', function(ev) {
  ev.preventDefault();
  let input = ev.target[0];
  window.result.innerHTML = search_word(input.value);
});
```

¿Pero es todo lo que queremos? No, claro que no, también queremos encontrar el amor, pero ya que javascript no puede hacer eso, nos conformaremos con agregar una funcionalidad de "sugerencia." Quiero que cuando escriba "accu" y presione el botón, que salga un dialogo que me diga "Did you mean accumulator?" (en inglés porque no me pagan lo suficiente para traducir los mensajes del sistema) 

Para esto necesitaremos ayuda, instalaremos una dependencia, una que encuentre resultados similares: [fuzzy-search](https://github.com/wouter2203/fuzzy-search#readme). Agreguemos lo siguiente.


```js
// main.js

import FuzzySearch from 'https://unpkg.com/fuzzy-search@3.0.1/src/FuzzySearch.js';

const fzf = new FuzzySearch(
  Object.keys(entries),
  [],
  {caseSensitive: false, sort: true}
);
```

Pero volvemos a la misma situación, esta no sería una operación segura porque en el momento que intentemos sacar un resultado de un arreglo vacío todo se cae. ¿Entonces qué hacemos? Escondemos todo debajo de una función.


```js
// main.js

function suggest(word) {
  const matches = fzf.search(word);
  return Maybe(matches[0]);
}
```

FuzzySearch está listo, ahora agregaremos un grandioso dialogo de confirmación.

```js
// main.js

function confirm_word(value) {
  if(value && confirm(`Did you mean ${value}`)) {
    return value;
  }
}
```

Combinemos las nuevas funciones con `search`.

```js
// main.js

const suggest_word = value => () => suggest(value)
  .map(confirm_word)
  .map(search);
```

Agregamos la nueva funcionalidad a `search_word`.

```diff
 const search_word = word => search(word)
   .or_else(() => search_name(word))
+  .or_else(suggest_word(word))
   .map(format)
   .unwrap_or('word not found');
```

Funciona. Pero ahora digamos que somos alérgicos a los `if`, sin mencionar que es de mala educación devolver `undefined` de una función. Podemos ser mejores.

```diff
 function confirm_word(value) {
-  if(value && confirm(`Did you mean ${value}`)) {
-    return value;
-  }
+  return confirm(`Did you mean ${value}`);
 }
```

```diff
 const suggest_word = value => () => suggest(value)
-  .map(confirm_word)
+  .filter(confirm_word)
   .map(search);
```

Algo me molesta. Cuando busco "accu," el dialogo aparece, confirmo la sugerencia y el resultado aparece. Pero "accu" sigue ahí en el formulario, es incómodo. Haremos el formulario se actualice con la palabra correcta.

```js
const update_input = val => window.search_form[0].value = val;
```
```diff
 const suggest_word = value => () => suggest(value)
   .filter(confirm_word)
+  .tap(update_input)
   .map(search);
```

¿Quieren verlo en acción? Aquí tienen.

{{ codepen(id="JjjNvLE", title="Maybe I got your word", load_js=true) }}

## Bonus track

> *Advertencia*: El objetivo de todo esto ya fue logrado, que vieran ese ejemplo en codepen. Lo que sigue es un experimento para ver si podía agregar soporte de operaciones asíncronas en la función `Maybe`. Si ya están cansados vayan directo al final y vean el último ejemplo.

Ahora quizá estén pensando: muy bonito y todo pero en el "mundo real" hacemos peticiones a servidores, consultamos bases de datos, hacemos todo tipo de cosas asíncronas, ¿puedo usar eso en este contexto? 

Bien. Entiendo. La implementación actual sólo contempla tareas normales. Tendrían que romper la cadena de `Maybe`s en el momento que aparezca una promesa (`Promise`)

Podemos crear un nuevo `Just` que esté consciente de que contiene una promesa. Es perfectamente posible, ¿un `AsyncJust`? ¿`JustAsync`? Suena horrible. 

Por si no lo saben, una promesa en javascript (me refiero a una instancia de la clase `Promise`) es un tipo de dato que se utiliza para coordinar eventos futuros. Lo hace usando un método llamado `then` el cual acepta una función (lo que llaman callback) y también tiene un método `catch` para cuando las cosas salen mal. Pero si controlamos lo que va dentro del `then` podemos mantener la misma interface del `Maybe`.

¿Qué tan buenos son siguiendo un montón de callbacks?

Aquí está. Lo llamaré `Future`.

```js
// no me juzguen

function Future(promise_thing) { 
  return {
    map: fun => Future(promise_thing.then(map_future(fun))),
    and_then: fun => Future(promise_thing.then(map_future(fun))),
    or_else: fun => Future(promise_thing.catch(fun)),
    tap: fun => Future(promise_thing.then(val => (fun(val), val))),
    unwrap_or: arg => promise_thing.catch(val => arg),

    filter: fun => Future(promise_thing.then(filter_future(fun))), 

    is_just: false,
    is_nothing: false,
    is_future: true,
    inspect: () => `<Promise>`
  };
}
```

Si apartamos todo el ruido tal vez se pueda entender mejor.

```js

{
  map: fun => promise.then(fun),
  and_then: fun => promise.then(fun),
  or_else: fun => promise.catch(fun),
  tap: fun => promise.then(val => (fun(val), val))),
  unwrap_or: arg => promise.catch(val => arg),

  filter: fun => promise.then(fun), 
}
```

- `map`/`and_then`: estos son iguales porque no puedes escaparte de una promesa.
- `or_else`: toma la función proporcionada y la pasa al método `catch`, esto para imitar el comportamiento de un `else`.
- `tap`: usa el método `then` para "echarle un vistazo" al valor dentro de la promesa. Este método es conviniente para colocar esas funciones "impuras" que tienen efecto sobre el mundo exterior.
- `unwrap_or`: Esto devuelve la promesa para que puedan usar `await`. Si todo sale bien obtendrán el valor original de la promesa, sino devolverá el primer parámetro que fue proporcionado.
- `filter`: este es un caso especial de `map`, es por eso que existe `filter_future`. 
- Casi todos estos métodos devuelven un nuevo `Future` porque `promise.then` siempre devuelve una nueva promesa.

Pero lo que hace que `Future` sea raro es lo que pasa dentro de `map`. ¿Recuerdan `map_future`?

```js
function map_future(fun) { // `fun` es el callback proporcionado
  return val => {
    /* Evaluemos el valor original de la promesa */

    let promise_content = val;

    // Necesitamos decidir si podemos confiar 
    // en el valor original
    if(Maybe(promise_content).is_nothing) {
      Promise.reject();
      return;
    }

    // Si es un Just obtenemos su contenido
    if(promise_content.is_just) {
      promise_content = val.unwrap_or();
    }

    /* Evaluemos el valor que devuelve el callback */

    // Usaremos Maybe otra vez 
    // porque tengo problemas de confianza.
    const result = Maybe(fun(promise_content));

    if(result.is_just) {
      // Si llegamos hasta aquí todo está bien.
      return result.unwrap_or();
    }

    // en este punto debería revisar si result
    // tiene un Future pero de ser así
    // lo están usando mal, así que por ahora
    // no hago nada.

    // Algo anda muy mal.
    return Promise.reject();
  }
}
```

Ahora `filter_future`.

```js
function filter_future(predicate_fun) {
  return val => {
    const result = predicate_fun(val);

    // ¿Acaso devolviste una promesa?
    if(result.then) {
      // Lo hiciste. Es por eso que no te pasan cosas buenas.
 
      // veamos dentro de la promesa.
      const return_result = the_real_result => the_real_result 
        ? val
        : Promise.reject();

      // mantenemos la cadena viva.
      return result.then(return_result);
    }

    return result ? val : Promise.reject();
  }
}
```

Lo último que me gustaría hacer es crear una función que convierta un valor regular en un `Future`.

```js
Future.from_val = function(val) {
  return Future(Promise.resolve(val));
}
```

Ahora lo que tenemos que hacer para agregar soporte dentro de `Maybe` es esto.

```diff
 function Maybe(the_thing) {
   if(the_thing === null 
     || the_thing === undefined 
     || the_thing.is_nothing
   ) {
     return Nothing();
   }
-
-  if(the_thing.is_just) {
+  if(the_thing.is_future || the_thing.is_just) {
     return the_thing;
    }

    return Just(the_thing);
 }
```

Pero la pregunta del millón sigue ahí. ¿Funciona?

Hice una  "[versión para terminal](https://github.com/VonHeikemen/maybe-type-in-js)" de esta aplicación. También modifiqué el ejemplo de codepen: agregué las funciones relacionadas con `Future`, el dialogo de confirmación ahora sí es un dialogo ([este](https://github.hubspot.com/vex/)) y la función del evento 'submit' la marqué con `async` para poder usar `await`.

{{ codepen(id="oNNwagG", title="Maybe I will promise you a word") }}

### Bonus bonus edit
Antes mencioné que haríamos trampa con esta implementación. [Así sería](https://codepen.io/VonHeikemen/pen/QWWYJwZ) con una implementación más apegada a las ideas del paradigma funcional.

