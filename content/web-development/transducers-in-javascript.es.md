+++
title = "Transductores en javascript" 
description = "Vamos a adentrarnos un poco en el mundo de los transductores (en javascript)"
date = 2020-12-27
lang = "es"
[taxonomies]
tags = ["javascript", "paradigma-funcional"]
+++

¿Qué pasaría si pudiéramos extraer la idea detrás de operaciones como `map` y `filter` y aplicarlas a otro tipo de colecciones más allá de los arreglos? ¿Y si les digo que puedo implementar `filter` una sola vez y reusar ese mismo código en diferentes tipos de colecciones? Esa es la premisa de los transductores. Hoy vamos a aprender qué son, cómo funcionan y cómo se usan.

## Requerimientos

Antes de comenzar hay un par de cosas que necesitan saber:

* [Cómo funciona Array.reduce](@/web-development/learn-fp/reduce-how-and-when.es.md)
* [Qué es un reducer](@/web-development/the-case-for-reducers.es.md)

También es recomendado que tengan familiaridad con los siguientes conceptos:

* Funciones de primera clase
* Funciones de orden superior
* Cierres (closures)

Y si no están al tanto de qué significa todo eso, no se preocupen. Sólo deben saber que en javascript podemos tratar a las funciones como cualquier otro tipo de dato.

Comencemos.

## ¿Qué son los transductores?

La palabra transductor tiene una larga historia. Si buscan su definición se van a encontrar con algo como esto:

> Un transductor es un dispositivo capaz de transformar o convertir una determinada manifestación de energía de entrada, en otra diferente de salida...
-- [Wikipedia](https://es.wikipedia.org/wiki/Transductor)

Definitivamente no estamos hablando de dispositivos físicos en este artículo. Pero sí se acerca a lo que queremos, el objetivo principal de un transductor (en nuestro contexto) será procesar los datos de una colección y potencialmente convertir esa colección de un tipo de dato a otro. 

Para nuestros propósitos una definición más cercana a lo que queremos sería esta:

> Transformaciones algorítmicas combinables.

Ya sé, no parece que esa tampoco ayude mucho. Bueno, la idea aquí es básicamente combinar procesos de una manera declarativa, y también que sea reusable en diferentes estructuras. Eso es todo. Pero claro es más fácil decirlo que hacerlo.

¿Cómo logramos todo eso?

Buena pregunta. Esto será todo un viaje, mejor empecemos con pasos pequeños. Primero preguntemos...

## ¿Por qué?

Usemos un ejemplo para responder eso. Imaginemos un escenario común. Digamos que tenemos un arreglo y queremos filtrarlo. ¿Cómo lo hacemos? Usamos el método `.filter`.

```js
const is_even = number => number % 2 === 0;
const data = [1, 2, 3];

data.filter(is_even);
// Array [ 2 ]
```

Todo se ve bien. Ahora nos llega otro requerimiento, tenemos que transformar los valores que pasan la prueba de la función `is_even`. No hay problema porque podemos usar `.map`.

```js
const is_even = number => number % 2 === 0;
const add_message = number => `The number is: ${number}`;

const data = [1, 2, 3];

data.filter(is_even).map(add_message);
// Array [ "The number is: 2" ]
```

Genial. Todo funciona bien hasta que un día, por razones que no vamos discutir, nos vemos obligados a convertir `data` en un [Set](https://developer.mozilla.org/es/docs/Web/JavaScript/Referencia/Objetos_globales/Set). Después de hacer el cambio nos topamos con este mensaje.

```
Uncaught TypeError: data.filter is not a function
```

¿Cómo podemos resolver esto? Una forma sería usar el ciclo `for..of`.

```js
const is_even = number => number % 2 === 0;
const add_message = number => `The number is: ${number}`;

const data = new Set([1, 2, 3]);
const filtered = new Set();

for(let number of data) {
  if(is_even(number)) {
    filtered.add(add_message(number));
  }
}

filtered;
// Set [ "The number is: 2" ]
```

La buena noticia es que esto funciona con cualquier estructura que implemente el [protocolo de iteración](https://developer.mozilla.org/es/docs/Web/JavaScript/Referencia/Iteration_protocols). La mala noticia es que para agregar otra "operación" tenemos que modificar el código de nuestro `for`.

¿Por qué modificar el código sería un problema?

Hagamos una comparación. Digamos que tenemos nuestro ciclo en su lugar.

```js
for(let number of data) {

}
```

¿Qué hacemos cuando queremos filtrar? Agregamos código dentro del bloque.

```diff
  for(let number of data) {
+   if(is_even(number)) {
+     filtered.add(number);
+   }
  }
```

¿Qué hacemos cuando queremos transformar? Agregamos código dentro del bloque. 

```diff
  for(let number of data) {
    if(is_even(number)) {
-     filtered.add(number);
+     filtered.add(add_message(number));
    }
  }
```

Eso va ocurrir cada vez que queramos agregar alguna funcionalidad a nuestro ciclo. ¿Alguna vez han escuchado la frase "abierto para extensión, cerrado para modificación"? Es básicamente lo que quiero ilustrar aquí. Para extender el ciclo `for` necesitamos modificarlo, no es que sea una terrible idea, es sólo que hay una forma más "elegante" de lograr nuestro objetivo.

Revisemos nuevamente nuestra primera versión, la que tenía `data` como un `Array`. ¿Qué hacemos cuando necesitamos filtrar? Agregamos una función.

```js
data.filter(is_even);
```

¿Qué hacemos cuando queremos transformar? Agregamos una función.

```diff
- data.filter(is_even);
+ data.filter(is_even).map(add_message);
```

¿Ven a donde quiero llegar? No voy a decir que es mejor, sólo digamos que es más "expresivo". En este caso, para extender nuestro proceso lo que hacemos es combinar funciones.

Pero no todo es color de rosas. Ya nos topamos con un problema: no todas las colecciones implementan estos métodos. Y otro problema que podríamos enfrentar tiene que ver con el desempeño, porque cada método es el equivalente a un ciclo `for`. Así que tal vez no sea una buena idea hacer una larga cadena de `filter`s y `map`s. 

Aquí es donde entran los transductores, con ellos podemos construir una cadena de operaciones de una manera declarativa y eficiente. Aunque no serán tan rápidos como un ciclo `for`, puede ser una manera de aumentar el desempeño cuando tienen una larga cadena de operaciones actuando sobre una colección con muchos (muchos) elementos.

Otra cosa en la que destacan sobre los métodos tradicionales en el prototipo `Array` es que podemos reusar la misma operación en distintas estructuras. Podemos por ejemplo implementar `filter` como un transductor una vez y reusamos ese mismo código para los arreglos, `Set`s, [generadores](https://developer.mozilla.org/es/docs/Web/JavaScript/Referencia/Sentencias/function*) u otro tipo de colecciones. Suena genial, ¿cierto?

## ¿Cómo funcionan?

La magia detrás de los transductores se encuentra dentro de un concepto que mencioné en la  sección de requirimientos: `reducer`. Especificamente `reducer`s de orden superior. (Se los advertí).

Tomen un momento para digerir esta frase: "reducer de orden superior."

¿Están listos?

Por ahora podemos pensar en un transductor como una función que puede recibir un `reducer` como argumento y retorna otro `reducer`. Resulta que (con un poco de magia) podemos combinar `reducer`s usando composición de funciones. Esta propiedad es la que nos va a permitir armar cadenas de operaciones como en nuestro ejemplo anterior, donde llamamos al método `.filter` y luego `.map`. Pero hay una gran diferencia, la manera en la que se combinan de hecho debería ser algo así.

```js
compose(filter(is_even), map(add_message));
```

Antes de que pregunten, la magia no ocurre en `compose`. Esa función es bastante genérica. Lo único que hace es pasar el resultado de una función a la otra. Podemos implementarla nosotros mismos.

```js
function compose(...fns) {
  const apply = (arg, fn) => fn(arg);
  return (initial) => fns.reduceRight(apply, initial);
}
```

Ahora, cuando combinamos varios transductores con `compose` lo que obtenemos es otro transductor. Pero ese no es el fin de la historia, como ya mencioné un transductor nos devuelve un `reducer`, ¿Y qué función conocen ustedes que necesite un `reducer`? Por supuesto, nuestro amigo `reduce`. `reduce` será como el "protocolo" que usaremos para recorrer los valores de la colección y hacer algo con ellos.

Creo que ya es suficiente de teorías, vamos a la práctica. Para empezar vamos a crear un transductor con la misma funcionalidad de `filter`.

### Creando un transductor

#### Paso 1: Reunir los argumentos necesarios

Primero lo primero, creamos una función y obtenemos todo lo que necesitamos con los parámetros. ¿Qué necesitamos en este caso? Una función que debería retornar `true` o `false`, un predicado.

```js
function filter(predicate) {

}
```

Un buen comienzo pero no es suficiente. Sabemos que en algún momento vamos a combinar este transductor con otro. Lo que necesitamos ahora es aceptar otro `reducer`, que vendría siendo el siguiente "paso" en la composición. Vamos a agregar eso.

```js
function filter(predicate, next) {
  
}
```

Si aún no está claro, recordemos nuestro ejemplo anterior.

```js
compose(filter(is_even), map(add_message));
```

Aquí lo que va a pasar es que `map(add_message)` nos devolverá un `reducer`. Ese `reducer` se convertirá en nuestro parámetro `next`.

Ya sé lo que piensan, sólo estoy enviando el argumento `is_even`. ¿Cómo vamos a obtener `next`? Vamos a lidiar con eso después. Sigamos. 

#### Paso 2: Retornar un reducer

En la práctica un `reducer` no es más que una función binaria. Sólo necesitamos retornar eso.

```js
function filter(predicate, next) {
  return function reducer(state, value) {
    // ???
  };
}
```

#### Paso 3: Implementa el resto

Bien, ya (casi) terminamos con la estructura del transductor. Lo que viene ahora es la lógica que queremos implementar. En este caso, lo que queremos hacer es replicar el comportamiento de `Array.filter`.

```js
function filter(predicate, next) {
  return function reducer(state, value) {
    if(predicate(value)) {
      return next(state, value);
    }

    return state;
  };
}
```

Aquí tomamos el predicado, lo evaluamos, y decidimos si vamos seguir con el siguiente paso o no hacemos nada.

#### Paso 4: Aplicación parcial

Aquí viene la magia. Sabemos cómo queremos usar `filter` pero justo ahora no va a funcionar. Necesitamos que `filter` sea lo suficientemente inteligente para saber cuando tiene que ejecutarse, ¿Cuándo es eso? Cuando tenga todos sus argumentos.

```js
function filter(predicate, next) {
  if(arguments.length === 1) {
    return (_next) => filter(predicate, _next);
  }

  return function reducer(state, value) {
    if(predicate(value)) {
      return next(state, value);
    }

    return state;
  };
}
```

Esta es sólo una forma de lograr la aplicación parcial. No tiene que ser de esta manera.

### Usando un transductor

Ya tenemos algo que en teoría debería funcionar. Ahora necesitamos una función `reduce`. Por suerte para nosotros el prototipo `Array` tiene una que podemos usar. Empecemos usando un solo transductor.

```js
const is_even = number => number % 2 === 0;

const data = [1, 2, 3];

const combine = (state, value) => (state.push(value), state);

data.reduce(filter(is_even, combine), []);
// Array [ 2 ]
```

¡Genial, de verdad funciona! Ahora vamos a expandir el conjunto de datos. Digamos que ahora `data` tendrá números negativos, pero tampoco queremos esos, vamos a crear otro filtro que deje pasar sólo los números positivos. Aquí es donde la composición entra en escena.

```js
const is_even = number => number % 2 === 0;
const is_positive = number => number > 0;

const data = [-2, -1, 0, 1, 2, 3];

const combine = (state, value) => (state.push(value), state);

const transducer = compose(filter(is_positive), filter(is_even));

data.reduce(transducer(combine), []);
// Array [ 2 ]
```

¿Vieron? Obtuvimos el mismo resultado. Ahora hagamos algo mejor, vamos a añadir otra "operación."

```js
function map(transform, next) {
  if(arguments.length === 1) {
    return (_next) => map(transform, _next);
  }

  return function reducer(state, value) {
    return next(state, transform(value));
  };
}
```

El comportamiento es el mismo que esperarían de `Array.map`. Aquí el valor es transformado antes de ir al siguiente paso. Ahora vamos a incorporarlo en el ejemplo.

```js
const data = [-2, -1, 0, 1, 2, 3];

const transducer = compose(
  filter(is_positive),
  filter(is_even),
  map(add_message)
);

data.reduce(transducer(combine), []);
// Array [ "The number is: 2" ]
```

Esto es bueno, muy bueno. Hay un detalle que necesitamos atender, la compatibilidad. Les mencioné que los transductores deberían funcionar con otros tipos de colecciones aparte de `Array`, pero aquí usamos `Array.reduce`. El asunto es que para completar el panorama tenemos que controlar la función `reduce`, así que haremos una.

Ya que javascript nos ofrece el protocolo de iteración, vamos a usar eso para ahorrarnos muchas molestias en nuestro propio `reduce`, con esto haremos que nuestros transductores sean compatibles con más tipos de colecciones.

```js
function reduce(reducer, initial, collection) {
  let state = initial;

  for(let value of collection) {
    state = reducer(state, value);
  }

  return state;
}
```

Para probar esto cambiaremos nuestro ejemplo, `data` pasará de ser un arreglo a un `Set`. Cambiaremos la función `combine`, para que ahora esté al tanto de cómo armar un `Set`. También cambiaremos nuestro valor inicial en `reduce` a un `Set`. Lo demás seguirá igual.

```js
const data = new Set([-2, -1, 0, 1, 2, 3]);

const combine = (state, value) => state.add(value);

const transducer = compose(
  filter(is_positive),
  filter(is_even),
  map(add_message)
);

reduce(transducer(combine), new Set(), data);
// Set [ "The number is: 2" ]
```

Noten que el resultado no tiene porque ser un `Set`, podemos transformar `data` a un `Array` si eso deseamos. Para cambiar de un tipo de colección a otro, sólo tenemos que intercambiar el valor inicial en `reduce` y cambiar la función `combine`.

Todo funciona bien pero hay una cosa más que podemos hacer para crear una "experiencia" más agradable. Hagamos una función auxiliar, `transduce`, para que se encargue de algunos detalles por nosotros.

```js
function transduce(combine, initial, transducer, collection) {
  return reduce(transducer(combine), initial, collection);
}
```

No parece una gran mejora pero esto nos permite aumentar nuestro control sobre `reduce`, ahora podríamos tener varias implementaciones para diferentes estructuras y decidir cual queremos usar basados en el tipo de dato de `collection`. Pero por el momento sólo usaremos la función `reduce` que creamos anteriormente.

Ahora lo que haremos será encargarnos de algunos detalles antes de tiempo. Crearemos funciones que tengan la misma funcionalidad de `combine`, para acumular los valores finales y asociamos eso con el valor inicial correcto.

```js
function curry(arity, fn, ...rest) {
  if (arity <= rest.length) {
    return fn(...rest);
  }

  return curry.bind(null, arity, fn, ...rest);
}

const Into = {
  array: curry(2, function(transducer, collection) {
    const combine = (state, value) => (state.push(value), state);
    return transduce(combine, [], transducer, collection);
  }),
  string: curry(2, function(transducer, collection) {
    const combine = (state, value) => state.concat(value);
    return transduce(combine, "", transducer, collection)
  }),
  set: curry(2, function(transducer, collection) {
    const combine = (state, value) => state.add(value);
    return transduce(combine, new Set(), transducer, collection);
  }),
};
```

Ahora podemos usar aplicación parcial en los argumentos. En esta ocasión logramos ese efecto con la función `curry`. Vamos a probar.

```js
const data = [-2, -1, 0, 1, 2, 3];

const transducer = compose(
  filter(is_positive),
  filter(is_even),
  map(add_message)
);

Into.array(transducer, data);
// Array [ "The number is: 2" ]
```

También podemos hacer esto.

```js
const some_process = Into.array(compose(
  filter(is_positive),
  filter(is_even),
  map(add_message)
));

some_process(data);
// Array [ "The number is: 2" ]
```

> Pueden visualizar todo el código de este ejemplo [aquí](https://gist.github.com/VonHeikemen/a6b2b2e27ea999e87ebc30cf9c039295)

Ahora poseemos "operaciones" reusables. No tuvimos que implementar un `filter` especial para el `Array` y otra para el `Set`. En este ejemplo no parece gran cosa, pero imagínense tener un arsenal de operaciones como [RxJS](https://rxjs.dev/api), y poder usarlas en diferentes estructuras. Lo único que deben hacer es una función `reduce`. Además, la manera en la que combinamos estas operaciones nos invita a resolver nuestros problemas con una función a la vez.

Hay una cosa más que deben saber.

## Esta no es su forma final

Hasta ahora he estado presentando los transductores como funciones que retornan un `reducer`, pero sólo era para ilustrar su funcionamiento. El problema es que nuestros transductores son limitados. Hay un par de cosas que nuestra implementación no soporta:

* Mecanismo de inicialización: Una manera de que un transductor pueda producir el valor inicial para el proceso.

* Interrupción temprana: Un transductor debe ser capaz de interrumpir todo el proceso y devolver el resultado que se ha procesado hasta el momento. Algo así como el `break` de un ciclo `for`.

* Una función "final": Básicamente proveer un mecanismo para ejecutar una función al final del proceso. Esto podría ser útil para ejecutar procesos de "limpieza".

Es por cosas como esas que muchos artículos que hablan sobre transductores recomiendan encarecidamente que usen una librería.

Librerías que tienen soporte para transductores solo conozco:

* [transducers-js](https://github.com/cognitect-labs/transducers-js)
* [ramda](https://ramdajs.com/docs/)

## Siguiendo el protocolo

Ya sabemos cómo funcionan los transductores a grandes rasgos, ahora vamos a descubrir cómo implementar uno de la manera correcta. Para esto vamos a seguir el [protocolo](https://github.com/cognitect-labs/transducers-js#the-transducer-protocol) establecido en la librería *transducers-js*.

Las reglas dicen que un transductor debe ser un objeto con la siguiente forma.

```js
const transducer = {
  '@@transducer/init': function() {
    return /* ???? */;
  },
  '@@transducer/result': function(state) {
    return state;
  },
  '@@transducer/step': function(state, value) {
    // ???
  }
};
```

* **@@transducer/init**: Será la función que nos da la oportunidad de retornar un valor inicial si por alguna razón necesitamos uno. El comportamiento "por defecto" es delegar sus funciones al siguiente transductor de la composición, con suerte alguno tendrá que devolver algo útil.

* **@@transducer/result**: Será la función que se ejecute al final del proceso, es decir cuando ya no haya más valores para procesar. Al igual que `@@transducer/init`, el comportamiento que se espera por defecto es delegar sus funciones al siguiente transductor en la composición.

* **@@transducer/step**: Aquí es donde reside la lógica para nuestro transductor, es decir la "operación" que queremos ejecutar. Básicamente esta función será nuestro `reducer`. 

Aún no hemos terminado, también necesitamos una manera de señalar que el proceso será interrumpido y regresar el resultado que se tiene en ese momento. Para esto el protocolo indica la existencia de un objeto especial que llama `reduced` (reducido). La idea es que cuando la función `reduce` detecte este objeto se de por terminado el proceso. Este objeto debe tener la siguiente forma.

```js
const reduced = {
  '@@transducer/reduced': true,
  '@@transducer/value': algo // el valor procesado hasta el momento
};
```

### Un verdadero transducer

Es momento de aplicar todo lo que hemos aprendido, vamos a reimplementar `filter` de la manera correcta. Podemos hacerlo, la mayor parte va a ser igual. 

Empezamos con una función que retorna un objeto.

```js
function filter(predicate, next) {
  return {

  };
}
```

Ahora la inicialización, ¿Qué necesitamos hacer? Nada, en realidad. Entonces lo que haremos será delegar.

```diff
  function filter(predicate, next) {
    return {
+     '@@transducer/init': function() {
+       return next['@@transducer/init']();
+     },
    };
  }
```

Al finalizar, ¿Qué necesitamos hacer? Nada. Ya saben el procedimiento.

```diff
  function filter(predicate, next) {
    return {
      '@@transducer/init': function() {
        return next['@@transducer/init']();
      },
+     '@@transducer/result': function(state) {
+       return next['@@transducer/result'](state);
+     },
    };
  }
```

Ahora para el gran final, la operación en sí.

```diff
  function filter(predicate, next) {
    return {
      '@@transducer/init': function() {
        return next['@@transducer/init']();
      },
      '@@transducer/result': function(state) {
        return next['@@transducer/result'](state);
      },
+     '@@transducer/step': function(state, value) {
+       if(predicate(value)) {
+         return next['@@transducer/step'](state, value);
+       }
+
+       return state;
+     },
    };
  }
```

Y que no se les olvide el toque mágico.

```diff
  function filter(predicate, next) {
+   if(arguments.length === 1) {
+     return (_next) => filter(predicate, _next);
+   }

    return {
      '@@transducer/init': function() {
        return next['@@transducer/init']();
      },
      '@@transducer/result': function(state) {
        return next['@@transducer/result'](state);
      },
      '@@transducer/step': function(state, value) {
        if(predicate(value)) {
          return next['@@transducer/step'](state, value);
        }
 
        return state;
      },
    };
  }
```

Ya tenemos el transductor, pero ahora tenemos un problema: no tenemos una función `reduce` capaz de utilizarlo.

### reduce mejorado

Ahora nos toca hacerle unos ajustes a nuestro `reduce`.

Recuerdan esto.

```js
function reduce(reducer, initial, collection) {
  let state = initial;

  for(let value of collection) {
    state = reducer(state, value);
  }

  return state;
}
```

Primero vamos a manejar la inicialización.

```diff
- function reduce(reducer, initial, collection) {
+ function reduce(transducer, initial, collection) {
+   if(arguments.length === 2) {
+     collection = initial;
+     initial = transducer['@@transducer/init']();
+   }
+
    let state = initial;

    for(let value of collection) {
      state = reducer(state, value);
    }

    return state;
  }
```

Cuando la función reciba dos argumentos la colección estará en `initial` y `collection` será `undefined`, así que lo que hacemos es asignar `initial` a `collection` y darle la oportunidad a nuestro transductor de generar el estado inicial del proceso. 

Ahora veremos cómo ejecutar el `reducer` que como saben ahora está situado en `@@transducer/step`.

```diff
  function reduce(transducer, initial, collection) {
    if(arguments.length === 2) {
      collection = initial;
      initial = transducer['@@transducer/init']();
    }
 
    let state = initial;

    for(let value of collection) {
-     state = reducer(state, value);
+     state = transducer['@@transducer/step'](state, value);
    }

    return state;
  }
```

Lo siguiente será evaluar el resultado del `reducer` y determinar si debemos seguir con el proceso.

```diff
  function reduce(transducer, initial, collection) {
    if(arguments.length === 2) {
      collection = initial;
      initial = transducer['@@transducer/init']();
    }
 
    let state = initial;

    for(let value of collection) {
      state = transducer['@@transducer/step'](state, value);
+
+     if(state != null && state['@@transducer/reduced']) {
+       state = state['@@transducer/value'];
+       break;
+     }
    }

    return state;
  }
```

Por último debemos asegurarnos que todas las operaciones sepan que el proceso ha terminado.

```diff
  function reduce(transducer, initial, collection) {
    if(arguments.length === 2) {
      collection = initial;
      initial = transducer['@@transducer/init']();
    }
 
    let state = initial;

    for(let value of collection) {
      state = transducer['@@transducer/step'](state, value);
 
      if(state != null && state['@@transducer/reduced']) {
        state = state['@@transducer/value'];
        break;
      }
    }

-   return state;
+   return transducer['@@transducer/result'](state);
  }
```

Hay un paso extra que me gustaría hacer. Tal vez notaron que renombré `reducer` a `transducer`, pero me gustaría que siguiera funcionando con `reducer`s normales, como los que se usa con `Array.reduce`. Entonces lo que haremos será crear un transductor que pueda transformar un `reducer` en un transductor.

```js
function to_transducer(reducer) {
  if(typeof reducer['@@transducer/step'] == 'function') {
    return reducer;
  }

  return {
    '@@transducer/init': function() {
      throw new Error('Method not implemented');
    },
    '@@transducer/result': function(state) {
      return state;
    },
    '@@transducer/step': function(state, value) {
      return reducer(state, value);
    }
  };
}
```

Ahora podemos usarla en `reduce`.

```diff
  function reduce(transducer, initial, collection) {
+   transducer = to_transducer(transducer);
+
    if(arguments.length === 2) {
      collection = initial;
      initial = transducer['@@transducer/init']();
    }
 
    let state = initial;

    for(let value of collection) {
      state = transducer['@@transducer/step'](state, value);
 
      if(state != null && state['@@transducer/reduced']) {
        state = state['@@transducer/value'];
        break;
      }
    }

    return transducer['@@transducer/result'](state);
  }
```

Es momento de probar todo el arduo trabajo.

```js
const is_positive = number => number > 0;

const data = [-2, -1, 0, 1, 2, 3];
const combine = (state, value) => (state.push(value), state);

reduce(filter(is_positive, to_transducer(combine)), [], data);
// Array(3) [ 1, 2, 3 ]
```

Bien, todo funciona. Pero es mucho trabajo usar `reduce`. Es por eso que tenemos la función `transduce`, pero justo ahora le falta algo, tenemos que agregarle `to_transducer`.

```js
function transduce(combine, initial, transducer, collection) {
  return reduce(
    transducer(to_transducer(combine)),
    initial,
    collection
  );
}
```

Vamos de nuevo.

```js
const is_positive = number => number > 0;

const data = [-2, -1, 0, 1, 2, 3];
const combine = (state, value) => (state.push(value), state);

transduce(combine, [], filter(is_positive), data);
// Array(3) [ 1, 2, 3 ]
```

Ahora vamos a probar la composición.

```js
const is_even = number => number % 2 === 0;
const is_positive = number => number > 0;

const data = [-2, -1, 0, 1, 2, 3];
const combine = (state, value) => (state.push(value), state);

const transducer = compose(filter(is_positive), filter(is_even));

transduce(combine, [], transducer, data);
// Array [ 2 ]
```

> Pueden visualizar todo el código de este ejemplo [aquí](https://gist.github.com/VonHeikemen/a70479feeb59a26cd5f217ea752cf115)

Oficialmente hemos terminado. No hay nada más qué hacer. Creo que ya tienen suficiente información para crear sus propios transductores.

## Conclusión

¡Lo lograron! Llegaron al final del artículo. Debo felicitarlos, especialmente si entendieron todo en el primer intento, este no fue nada fácil. Celebren, se lo merecen.

En fin, hoy aprendimos que los transductores (en javascript) son transformaciones que pueden operar en diferentes tipos de colecciones, siempre y cuando estas provean una función `reduce` que sea compatible. También tienen algunas propiedades sumamente útiles como interrupción temprana (como la de un ciclo `for`), mecanismos para señalar la finalización e inicio de un proceso y pueden ser combinadas usando composición de funciones. Y por último, también deberían ser eficientes, pero no son más rápidos que un ciclo `for`. Aunque no sean la solución más eficiente en cuestión de desempeño su nivel de compatibilidad con diferentes colecciones y la forma declarativa para combinar operaciones hacen que sea una herramienta poderosa.

## Fuentes

- [Functional-Light JavaScript | Appendix A: Transducing](https://github.com/getify/Functional-Light-JS/blob/master/manuscript/apA.md/#appendix-a-transducing)
- [Transducers: Supercharge your functional JavaScript](https://www.jeremydaly.com/transducers-supercharge-functional-javascript/)
- [Magical, Mystical JavaScript Transducers](https://jrsinclair.com/articles/2019/magical-mystical-js-transducers/)
- [Transducers: Efficient Data Processing Pipelines in JavaScript](https://medium.com/javascript-scene/transducers-efficient-data-processing-pipelines-in-javascript-7985330fe73d)
- ["Transducers" by Rich Hickey (video)](https://www.youtube.com/watch?v=6mTbuzafcII)
- [transducers-js](https://github.com/cognitect-labs/transducers-js)

