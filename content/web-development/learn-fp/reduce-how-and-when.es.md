+++
title = "Reduce: cómo y cuando" 
description = "Vamos a identificar cual es el caso ideal para usar reduce y aprenderemos algo en el camino"
date = 2020-04-19
lang = "es"
[taxonomies]
tags = ["javascript", "paradigma-funcional", "aprendizaje"]
+++

Vamos a hablar del elefante rosa en el prototipo `Array`, me refiero al a veces odiado método [reduce](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/Reduce) pero no vamos a discutir sobre si esta función es buena o mala. Vamos a descubrir qué es lo que hace internamente, luego intentaremos descubrir las situaciones en las que puede ser una solución efectiva.

Para asegurarnos de entender su funcionamiento vamos a empezar implementando nuestra versión.

## ¿Cómo funciona?

`reduce` es una función que toma una lista de valores y la transforma en otra cosa. La palabra clave aquí es **transformación**. Esta transformación la determina el "usuario" de nuestra función, son ellos los que deciden qué pasará ¿Qué quiere decir eso? Significa que aparte del arreglo que vamos a procesar necesitamos aceptar una función (un callback) como parámetro. Así que la firma de la función sería esta.

```js
function reduce(arr, callback) {
  // código...
}
```

Ya tenemos algunos valores, y ahora ¿Qué hacemos con ellos? Sabemos que los métodos del prototipo `Array` aplican una función a cada uno de sus elementos. Hagamos eso.

```js
function reduce(arr, callback) {
  for(const valor of arr) {
    callback(valor);
  }
}
```

Todavía no hace lo que queremos pero se acerca. Ahora falta el ingrediente secreto, el acumulador. Esta será una variable que crearemos para recordar el **estado actual** de nuestra transformación. Cada vez que apliquemos la función `callback` a un valor guardamos el resultado en el acumulador. Como bono extra, antes de guardar el nuevo estado en el acumulador le pasamos a `callback` el estado actual para que nuestro "usuario" no tenga que hacer ningún esfuerzo extra.

```diff
  function reduce(arr, callback) {
+   let estado;
    for(const valor of arr) {
-     callback(valor);
+     estado = callback(estado, valor);
    }

+   return estado;
  }
```

Recuerden bien esas líneas que están verde. Por muy complicado que se vea `reduce` en el exterior, sin importar cuantos trucos raros vean por ahí, esas tres líneas son lo único que importa.

Aunque no sea una replica exacta de `Array.reduce` será suficiente para nuestros propósitos. Vamos a probarla. 

```js
const array1 = [1, 2, 3, 4];
const callback = (estado, valor) => {
  if(estado == null) {
    return valor;
  }

  return estado + valor;
};

// 1 + 2 + 3 + 4
reduce(array1, callback);
// valor esperado: 10
```

¿Ven ese `if`? Está ahí porque en la primera iteración `estado` no tiene un valor, eso parece innecesario. Nosotros como autores de `reduce` podemos ayudar a reducir la cantidad de código que necesita `callback`. Al disminuir la carga de responsabilidad que necesita `callback` podemos hacer que `reduce` sea mucho más flexible. Lo que haremos será tomar el primer valor del arreglo y ese se convertirá en el `estado` para nuestra primera iteración.

```diff
  function reduce(arr, callback) {
-   let estado;
-   for(const valor of arr) {
+   let estado = arr[0];
+   let resto = arr.slice(1);
+   for(const valor of resto) {
      estado = callback(estado, valor);
    }

    return estado;
  }
```

Vamos otra vez.

```js
const array1 = [1, 2, 3, 4];
const callback = (estado, valor) => {
  return estado + valor;
};

// 1 + 2 + 3 + 4
reduce(array1, callback);
// valor esperado: 10
```

Si aún les cuesta un poco descifrar lo que está pasando, puedo ayudarles con eso. Si sacamos `callback` de la ecuación esto es lo que sucede.

```js
function reduce(arr) {
  let estado = arr[0];
  let resto = arr.slice(1);
  for(const valor of resto) {
   estado = estado + valor;
  }

  return estado;
}
```

¿Recuerdan las tres líneas verdes?

```diff
  function reduce(arr) {
+   let estado = arr[0];
    let resto = arr.slice(1);
    for(const valor of resto) {
+    estado = estado + valor;
    }

+   return estado;
  }
```

¿Lo notaron? Eso es todo lo que necesitan recordar. Básicamente,`reduce` nos da la habilidad de transformar una **operación** que actúa sobre dos valores a una que actúa sobre una cantidad variada.

## ¿Cuando es útil?

`reduce` es una de esas funciones que pueden utilizarse en muchas ocasiones pero no en todas es la mejor solución. Ahora que sabemos cómo funciona veamos en qué tipo de situaciones puede ser la mejor opción.

### Un caso ideal

El ejemplo anterior ya debería darles una pista. Nuestra función es más efectiva cuando seguimos ciertos patrones. Pensemos un momento en lo que hace `callback` en nuestro ejemplo. Sabemos que necesita dos números, ejecuta una operación matemática y nos devuelve otro número. Entonces hace esto.

```
Número + Número -> Número
```

 Bien, pero si damos un paso atrás y pensamos en términos más generales lo que tenemos es esto.

```
TipoA + TipoA -> TipoA
```

Hay dos valores del mismo tipo (TipoA) y una operación (el signo +) que nos devuelve otro valor del mismo tipo (TipoA). Cuando lo vemos de esa manera podemos darnos cuenta de un patrón que puede ser útil más allá de las operaciones matemáticas. Hagamos otro ejemplo con números pero esta vez lo que haremos será una comparación.

```js
function max(un_numero, otro_numero) {
  if(un_numero > otro_numero) {
    return un_numero;
  } else {
    return otro_numero;
  }
}
```

`max` es una operación que actúa sobre dos números, los compara y devuelve el mayor. Es muy general y con una capacidad limitada. Si volvemos a pensar en lo abstracto vemos ese patrón otra vez.

```
TipoA + TipoA -> TipoA
```

O si somos más específicos.

```
Número + Número -> Número
```

Ya saben lo que significa, podemos usar `reduce` para ampliar su capacidad.

```js
const array2 = [40, 41, 42, 39, 38];

// 40 > 41 > 42 > 39 > 38
reduce(array2, max);
// valor esperado: 42
```

Resulta que el patrón que hemos estado siguiendo para crear el `callback` que necesita `reduce` tiene un nombre en el paradigma funcional, a este lo llaman **Semigrupo**. En cualquier ocasión en la tengan dos valores de un mismo tipo y puedan combinarlos para crear otra instancia, están en la presencia de un semigrupo. En otras palabras, *dos valores* + *forma de combinarlos* = *Semigrupo*.

Una manera de probar que tienen una operación que sigue las reglas de un semigrupo es asegurarse de que la función cumple con la propiedad asociativa. Nuestra función `max` por ejemplo.

```js
const max_1 = max(max(40, 42), 41); // => 42
const max_2 = max(40, max(42, 41)); // => 42

max_1 === max_2
// valor esperado: true
```

¿Ven? Ejecutarla con el tipo de dato adecuado en diferente orden no afecta su resultado. Esto nos da la garantía de que funcionará si la combinamos con `reduce` y un arreglo de números.

¿Pero podríamos aplicar estas reglas a una estructura más compleja? Claro que sí. En javascript ya tenemos un par que las cumplen. Piensen en los arreglos, en el prototipo `Array` tenemos el método `concat`, este nos permita mezclar dos arreglos y crear uno nuevo con los elementos de ambos.

```js
function concat(uno, otro) {
  return uno.concat(otro);
}
```

Con esto tenemos que.

```
Array + Array -> Array
```

Okey, el segundo parámetro de `concat` no tiene que ser un arreglo pero ignoraremos eso por el momento. Entonces si combinamos `concat` con `reduce`.

```js
const array3 = [[40, 41], [42], [39, 38]];

// [40, 41] + [42] + [39, 38]
reduce(array3, concat);
// valor esperado: [40, 41, 42, 39, 38]
```

Ahora si quisieramos podríamos crear una función que "aplana" un nivel de un arreglo multidimensional, ¿No es genial? Y al igual que con los números, con los arreglos no tenemos porque limitarnos a operaciones que nos provee javascript. Si tenemos alguna función auxiliar que trabaje con dos arreglos y cumpla con la propiedad asociativa podemos combinarla con `reduce`.

Digamos que tenemos una función que une los elementos únicos de dos arreglos.

```js
function union(uno, otro) {
  const set = new Set([...uno, ...otro]);
  return Array.from(set);
}
```

Bien, tenemos una función que trabaja con dos valores de un mismo tipo ahora veamos si cumple con la propiedad asociativa.

```js
const union_1 = union(union([40, 41], [40, 41, 42]), [39]);
const union_2 = union([40, 41], union([40, 41, 42], [39]));

union_1.join(',') == union_2.join(',');
// valor esperado: true
```

Sí cumple con las reglas, eso quiere decir que es posible procesar una cantidad variada de arreglos si usamos `reduce`.

```js
const array4 = [
  ['hello'],
  ['hello', 'awesome'],
  ['world', '!'],
  ['!!!!', 'world']
];

reduce(array4, union);
// valor esperado: [ "hello", "awesome", "world", "!", "!!" ]
```

### Algo de resistencia

Habrán notado que en todos los ejemplos nuestros arreglos de datos son todos del tipo correcto, esto no siempre es así en el "mundo real". Podemos encontrarnos situaciones en las que el primer elemento de un arreglo no es un dato válido para nuestra operación.

Imaginemos que queremos usar `concat` otra vez pero el arreglo que tenemos para procesar es el siguiente.

```js
const array5 = [40, 41, [42], [39, 38]];
```

Si intentamos usar `reduce`.

```js
reduce(array5, concat);
```

Obtenemos esto.

```
TypeError: uno.concat is not a function
```

Esto ocurre porque en la primera iteración el valor de `uno` es el número `40`, el cual no posee un método `concat`. ¿Qué debemos hacer? Por lo general se considera una buena práctica utilizar un valor inicial fijo para evitar este tipo de errores. Pero tenemos un problema, nuestro `reduce` no acepta un valor inicial, así que deberíamos corregir eso.

```diff
- function reduce(arr, callback) {
-   let estado = arr[0];
-   let resto = arr.slice(1);
+ function reduce(arr, ...args) {
+   if(args.length === 1) {
+     var [callback] = args;
+     var estado = arr[0];
+     var resto = arr.slice(1);
+   } else if(args.length >= 2) {
+     var [estado, callback] = args;
+     var resto = arr;
+   }
    for(const valor of resto) {
     estado = callback(estado, valor);
    }

    return estado;
  }
```

Ahora para evitar el error anterior lo que haremos será pasarle a `reduce` un arreglo vacío como valor inicial.

```js
reduce(array5, [], concat);
// valor esperado: [ 40, 41, 42, 39, 38 ]
```

Ya no hay error y pudimos obtener el arreglo que queríamos. Pero fíjense en una cosa, el arreglo vacío no sólo logró evitar el error sino que también dejó intacto el resultado de la operación. Al igual que con los números, con los arreglos tenemos la noción de un elemento vacío que podemos usar en nuestras operaciones sin causar un error en nuestro programa.

El arreglo vacío puede considerarse como un **elemento identidad**, un valor neutro que al aplicarlo a una operación no tiene ningún efecto en el resultado final. Adivinen qué, este comportamiento también tiene un nombre en el paradigma funcional, se le conoce como **Monoid**. Cuando tenemos un semigrupo con un elemento identidad estamos en presencia de un monoid. Entonces, *semigrupo* + *elemento identidad* = *Monoid*.

Podemos probar que los arreglos siguen las reglas de un monoid para nuestras operaciones.

```js
// Concat
const concat_1 = concat([], ['hello']) // => ["hello"]
const concat_2 = concat(['hello'], []) // => ["hello"]

concat_1.join(',') == concat_2.join(',');
// valor esperado: true

// Union
const union_3 = union([], ['hello']); // => ["hello"]
const union_4 = union(['hello'], []); // => ["hello"]

union_3.join(',') == union_4.join(',');
// valor esperado: true
```

¿Por qué es importante? Piensen en esto: ¿Cuantas veces han tenido que escribir un `if` para resguardar una operación de un valor `null` o `undefined`? Si podemos representar un "valor vacío" de una manera más segura podemos eliminar toda una categoría de errores en nuestros programas.

Otra situación donde los monoids son útiles es cuando queremos ejecutar una operación "insegura" sobre un valor. Podríamos aplicar esa operación sobre una referencia a un valor vacío y así dejar el resto de los elementos intactos.

Imaginen que tienen fragmentos de información esparcidos en varios objectos y queremos unirlos.

```js
const array6 = [
  {name: 'Harold'},
  {lastname: 'Cooper'},
  {state: 'wrong'}
];
```

Normalmente usarían la [sintaxis de extensión](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Spread_syntax) para mezclar todo eso, pero digamos que vivimos en un mundo donde eso no es posible. No teman, tenemos una función que puede hacer el trabajo.

```js
Object.assign;
```

Si lo piensan `Object.assign` también sigue el patrón.

```
TipoA + TipoA -> TipoA
```

Si le pasamos dos objetos nos devuelve un nuevo objecto. Pero hay algo que deben saber, `Object.assign` modifica el objeto que le pasamos como primer parámetro. Entonces si hacemos esto.

```js
reduce(array6, Object.assign);
// Valor esperado: { "name": "Harold", "lastname": "Cooper", "state": "wrong" }
```

Parecería que todo está bien, pero no es así. Si revisan `array6[0]` verán que ha cambiado, definitivamente no quieren eso. Por suerte para nosotros los objetos en javascript se comportan como monoids, así que podemos usar un "valor vacío". Entonces la manera correcta de usar `reduce` en este caso sería esta. 

```js
reduce(array6, {}, Object.assign);
// Valor esperado: { "name": "Harold", "lastname": "Cooper", "state": "wrong" }

array6
// Valor esperado: [ { "name": "Harold" }, { "lastname": "Cooper" }, { "state": "wrong" } ]
```

Podemos decir que cuando trabajamos con un arreglo de estructuras que siguen las reglas de los monoid podemos estar seguros que `reduce` será una buena opción para procesarlo.

## Más allá de los arreglos

Si nosotros pudimos implementar una versión de `reduce` para los arreglos entonces no sería del todo extraño pensar que otras personas hayan incorporado algo similar a otras estructuras. Saber cómo funciona `reduce` puede ser muy útil si utilizan una librería que tenga un método parecido.

Por ejemplo, la librería [mithril-stream](https://mithril.js.org/stream.html) tiene un método llamado `scan` que tiene la siguiente forma.

```
Stream.scan(fn, accumulator, stream)
```

Esa variable `fn` debe ser una función que debe tener la siguiente firma.

```
(accumulator, value) -> result | SKIP
```

¿Reconocen eso? Espero que sí. Son los mismos requerimientos de `reduce`. ¿Pero qué hace esa función? Bueno, ejecuta la función `fn` cuando la fuente (`stream`) produce un nuevo dato. Cuando la función `fn` es ejecutada recibe como parámetro el estado actual del acumulador y el nuevo dato producido, luego el resultado retornado por `fn` se convierte en el nuevo estado del acumulador. ¿Les suena familiar ese comportamiento?

Pueden probar el método `scan` con nuestra función `union` y ver cómo se comporta.

```js
import Stream from 'https://cdn.pika.dev/mithril-stream@^2.0.0';

function union(one, another) {
  const set = new Set([...one, ...another]);
  return Array.from(set);
}

const list = Stream(['node', 'js']);

const state = Stream.scan(union, [], list);
state.map(console.log);

list(['node']);
list(['js', 'deno']);
list(['node', 'javascript']);
```

Deberían observar cómo la lista sólo agrega elementos que no han sido agregados antes.

Pueden ver en acción una versión modificada de ese fragmento en codepen.

{{ codepen(id="NWGrozo", title="A different reduce", load_js=true) }}

¿Vieron? nuestro conocimiento de `reduce` (y tal vez algo de semigrupos y monoids) nos puede ayudar a crear funciones auxiliares que podemos reusar con diferentes estructuras. ¿No es genial?

## Conclusión

Aunque no mencioné todas las cosas que pueden hacer con `reduce` ahora tienen las herramientas para poder identificar los casos en lo que puede ser utilizado efectivamente, incluso si no están seguros pueden hacer las pruebas necesarias para garantizar que la operación que quieren ejecutar tiene las características adecuadas.

## Fuentes

- [Practical Category Theory: Monoids (video)](https://www.youtube.com/watch?v=Qnkn4612ZIQ)
- [Funcadelic.js](https://github.com/thefrontside/funcadelic.js)
- [Functional JavaScript: How to use array reduce for more than just numbers](https://jrsinclair.com/articles/2019/functional-js-do-more-with-reduce/)
- [Array.prototype.reduce (MDN)](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/Reduce)
- [Fantasy Land](https://github.com/fantasyland/fantasy-land#fantasy-land-specification)
