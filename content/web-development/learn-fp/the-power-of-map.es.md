+++
title = "Un poco del paradigma funcional en tu javascript: Los poderes de map"
description = "Vamos a ver qué hace a map tan especial"
date = 2020-02-22
lang = "es"
[taxonomies]
tags = ["javascript", "paradigma-funcional", "aprendizaje"]
[extra]
canonical_url = "https://dev.to/vonheikemen/un-poco-del-paradigma-funcional-en-tu-javascript-los-poderes-de-map-473i"
+++

En esta ocasión vamos a dar un vistazo dentro del mundo de los `functors` y descubrir qué los hace tan especiales. Functor es uno de esos términos que aparece cuando la gente a habla del paradigma funcional en la programación pero cuando llega el momento de explicar qué es, lo que ocurre es que mencionan otros términos abstractos o sólo cuantan los detalles necesarios que necesitan saber. Ya que no tengo ningún conocimiento formal de teoría de categorías no voy a fingir qué sé exactamente lo que son, lo que haré será dicerles lo suficiente para que puedan reconocerlos y cómo pueden usarlos.

## ¿Qué es un functor?

Estoy convencido de qué el término es difícil de entender porque se necesita conocimientos de otro tipo de estructura para poder comprenderlos en su totalidad. Otra cosa que contribuye a la confusión es el hecho de que la teoría no sé traduce de la manera más clara en código. Pero bueno, aún así intentaré responder la pregunta, empezando con lo abstracto. 

Pueden pensar en ellos como la relación que existe entre dos conjuntos. Tengan paciencia, esto empezará a tener sentido en un momento. Imaginen dos arreglos.

```js
const favorite_numbers  = [42, 69, 73];
const increased_numbers = [43, 70, 74];
```

Bien, tenemos el conjunto `favorite_numbers` y el conjunto `increased_numbers`, son dos arreglos diferentes almacenados en dos variables separadas pero todos sabemos que hay una conexión entre ellos, lo que debemos tener en cuenta es que podemos expresar esa relación con código. Imaginen que el arreglo `increased_numbers` no existe pero aún necesitamos esos números, para hacer que aparezcan nuevamente sólo necesitamos la ayuda de nuestro viejo amigo `map`.

```js
const increased_numbers = favorite_numbers.map(num => num + 1);
```

`map` va a recorrer todo el arreglo y por cada número va a incrementarlo y colocarlo en nuevo arreglo, lo que trae a `increased_numbers` devuelta. Aunque hemos creado este arreglo nuevamente, este no salió de la nada, nosotros no inventamos los números `43`, `70` y `74`. Lo que hicimos fue describir la relación que hay entre esos números y `favorite_numbers`. 

¿Eso es todo? ¿Un functor es un arreglo? La respuesta a eso es un rotundo no. Los arreglos son simplemente una manera muy conveniente de representar un uso común. Esto deja una pregunta en el aire.

## ¿Cómo los reconocemos?

A menudo veo que otras personas los describen como cajas. No creo que estén totalmente errados porque utilizar un contenedor es una de las maneras más simples en las que se puede implementar un functor. La analogía de la caja es especialmente curiosa en javascript porque podemos usar corchetes para crear un arreglo. Vean.

```js
// Un valor
1;

// Una caja
[];

// Miren, un valor en una caja
[1];
```

Volviendo a la pregunta, ¿Cómo los reconocemos? Okey, resulta pasa y acontece que hay reglas.

### Las reglas

De nuevo usaré arreglos con números sólo por lo conveniente pero estas reglas deben aplicar a todas aquellas estructuras que deseen ser parte del club functor.

#### Identidad

Dada la función `identity`.

```js
function identity(x) {
  return x;
}
```

`value` and `value.map(identity)` deben ser equivalentes.

Por ejemplo.

```js
[1,2,3];               // => [1,2,3]
[1,2,3].map(identity); // => [1,2,3]
```

¿Qué? ¿Qué importancia tiene eso? ¿Qué nos dice?

Buenas preguntas. Esto nos dice que la función `map` debe preservar la forma de la estructura. En nuestro ejemplo si aplicamos `map` a un arreglo de tres elementos debemos recibir un nuevo arreglo con tres elementos. Si fuera un arreglo con cien elementos deberíamos recibir un nuevo arreglo con cien elementos. Ya entienden.

#### Composición

Dadas dos funciones `fx` y `gx` lo siguiente debe ser cierto.

`value.map(fx).map(gx)` y `value.map(arg => gx(fx(arg)))` deben ser equivalentes.

Otro ejemplo.

```js
function add_one(num) {
  return num + 1;
}

function times_two(num) {
  return num * 2;
}

[1].map(add_one).map(times_two);         // => [4]
[1].map(num => times_two(add_one(num))); // => [4]
```

Si ya saben como funciona `Array.map` esto debería ser obvio. Aquí se presenta la oportunidad de optimizar el código para el desempeño o legibilidad. En el caso de los arreglos, múltiples llamadas a `map` puede tener un gran impacto en el desempeño a medida que vaya creaciendo el número de elementos en la lista.

Eso es todo. Esas dos reglas son lo único que deben tener en cuenta para reconocer un functor.

## ¿Tiene que ser .map?

Supongo que ahora desean saber qué otro tipo de cosas siguen estas reglas que mencioné. Resulta que hay otra estructura bastante popular que sigue estas reglas y esa es `Promise`. Vean.

```js
// Un valor
1;

// Una caja
Promise.resolve;

// Miren, un valor en una caja
Promise.resolve(1);

// Identidad
Promise.resolve(1).then(identity); // => 1 (eventualmente)

// Composición
Promise.resolve(1).then(add_one).then(times_two);        // => 4
Promise.resolve(1).then(num => times_two(add_one(num))); // => 4
```

Si somos honestos aquí, `Promise.then` se comporta más como `Array.flatMap` y no como `.map` pero ignoremos eso.

Bien, tenemos `Array` y tenemos `Promise` ambos actúan como contenedores y tienen métodos que siguen las reglas. ¿Pero qué pasaría si no existiera `Array.map`? ¿Significa que `Array` no es un functor? ¿Perdemos todos los beneficios?

Vamos a dar un paso atrás. ¿Si `Array.map` no existe `Array` no es un `functor`? No lo sé. ¿Perdemos todos los beneficios? No, aún podemos tratar los arreglos como un functor, lo que perdemos es la conviniencia de la sintaxis `.map`. Aún podemos crear nuestro propio `map` fuera de la estructura.


```js
const List = {
  map(fn, arr) {
    let result = [];
    for (let data of arr) {
      result.push(fn(data));
    }

    return result;
  }
};
```

¿Ven? No está tan mal. Y funciona.

```js
// Identidad
List.map(identity, [1]); // => [1]

// Composición
List.map(times_two, List.map(add_one, [1]));   // => [4]
List.map(num => times_two(add_one(num)), [1]); // => [4]
```

¿Están pensando lo que yo? Probablemente no. Esto es lo que estoy pensando, si podemos crear `map` para los arreglos entonces nada evita que hagamos uno para los objetos, después de todo, los objetos también son un conjunto de valores.

```js
const Obj = {
  map(fn, ob) {
    let result = {};
    for (let [key, value] of Object.entries(ob)) {
      result[key] = fn(value);
    }

    return result;
  }
};

// ¿Por qué solo map? 
// Basado en esto ya pueden ver cómo crear `filter` y `reduce`
```

Vamos a probar.

```js
// Identidad
Obj.map(identity, {some: 1, prop: 2}); // => {some: 1, prop: 2}

// Composición
Obj.map(times_two, Obj.map(add_one, {some: 1, prop: 2})); // => {some: 4, prop: 6}
Obj.map(num => times_two(add_one(num)), {some: 1, prop: 2}); // => {some: 4, prop: 6}
```

## Hazlo tú mismo

Toda esta charla de arreglos y objetos es útil pero ahora pienso que sabemos lo suficiente para crear nuestro propio functor, las reglas parecen ser bastante sencillas. Vamos a hacer algo vagamente útil. ¿Alguna vez han escuchado de los Observables? Bien, vamos a hacer algo parecido. Vamos a crear una versión más simple de [mithril-stream](https://mithril.js.org/stream.html), será divertido.

Lo que queremos hacer es manejar un flujo de datos a través del tiempo. La interfaz de nuestra función será esta.

```js
// Crear instancia con valor inicial
const num_stream = Stream(0);

// Crear un flujo dependendiente
const increased = num_stream.map(add_one);

// Obtener el valor actual
num_stream(); // => 0

// Colocar un nuevo valor en el flujo
num_stream(42); // => 42

// La fuente se actualiza
num_stream(); // => 42

// El dependiente se actualiza
increased(); // => 43
```

Empecemos con la función que obtiene y actualiza el valor.

```js
function Stream(state) {
  let stream = function(value) {
    // Si tenemos un parametro actualizamos el estado
    if(arguments.length > 0) {
      state = value;
    }

    // retorna el estado actual
    return state;
  }

  return stream;
}
```

Ahora esto debería funcionar.

```js
// Inicializamos
const num_stream = Stream(42);

// Obtenemos el valor
num_stream(); // => 42

// Actualizamos
num_stream(73);

// Revisamos
num_stream(); // => 73
```

Ya sabemos que queremos un método `map` pero ¿Cuál es el efecto que debe tener? Lo que queremos es que la función (el callback) escuche los cambios de la fuente. Empecemos con eso, lo que haremos será almacenar las funciones proporcionadas a `map` en un arreglo y las ejecutaremos justo después de que se produzca el cambio.

```diff
  function Stream(state) {
+   let listeners = [];
+
    let stream = function(value) {
      if(arguments.length > 0) {
        state = value;
+       listeners.forEach(fn => fn(value));
      }

      return state;
    }

    return stream;
  }
```

Ahora creamos el método `map`, pero no debe ser un método cualquiera, debemos seguir las reglas.

- Identidad: Cuando `map` es ejecutado necesita preservar la forma de la estructura. Esto significa que debemos retornar otro `stream`.

- Composición: Ejecutar `map` varias veces debe ser equivalente a la composición de funciones proporciondas a esas llamadas.

```js
function Stream(state) {
  let listeners = [];

  let stream = function(value) {
    if(arguments.length > 0) {
      state = value;
      listeners.forEach(fn => fn(value));
    }

    return state;
  }

  stream.map = function(fn) {
    // Crea una nueva instancia con el valor transformado.
    // Esto ejecutara `fn` cuando se llame a `map`
    // esto no siempre será lo mejor si `fn` tiene algún 
    // efecto fuera de su ámbito. Tengan cuidado.
    let target = Stream(fn(state));

    // Transforma el valor y actualiza el nuevo flujo
    const listener = value => target(fn(value));

    // Actualiza los dependientes de la fuente
    listeners.push(listener);

    return target;
  }

  return stream;
}
```

Probemos las reglas. Comenzamos con identidad.

```js
// Los `Stream` son como una cascada
// el primero es el más importante
// este es el que activa los demás
const num_stream = Stream(0);

// Crea el dependendiente
const identity_stream = num_stream.map(identity); 

// Actualiza la fuente
num_stream(42);

// Revisa
num_stream();      // => 42
identity_stream(); // => 42
```

Ahora la composición.

```js
// Crea la fuente
const num_stream = Stream(0);

// Crea los dependientes
const map_stream = num_stream.map(add_one).map(times_two);
const composed_stream = num_stream.map(num => times_two(add_one(num)));

// Actualiza
num_stream(1);

// Revisa
map_stream();      // => 4
composed_stream(); // => 4
```

Nuestro trabajo está hecho. ¿Pero de verdad sirve? ¿Se puede hacer algo con eso? Bueno, sí, pueden usarlo para manejar eventos. Así.

{{ codepen(id="dyoMJRw", title="an fmap example", load_js=true) }}

### Más ejemplos

Ahora ya deben tener un buen entendimiento de los functors, pero si quieren seguir viendo más pueden revisar estos artículos.

- [Manejar ausencia de valores](@/web-development/learn-fp/using-a-maybe.es.md)
- [Manejo de efectos secundarios](https://jrsinclair.com/articles/2018/how-to-deal-with-dirty-side-effects-in-your-pure-functional-javascript/) (inglés)

## Conclusión

Lo único que queda por responser es "¿Qué beneficios tienen los functors?"

- Este patrón nos permite enfocarnos en un problema a la vez. La función `map` se encarga de obtener los datos necesarios y en el `callback` nos podemos enfocar en cómo procesarlos.

- Reutilización. Este estilo de programación promueve el uso y creación de funciones de generales que sólo se encargan de una tarea, en muchos casos estas pueden ser compartidas incluso entre proyectos.

- Extensión a través de la composición. Hay gente que tiene sentimientos encontrados en este caso, especialmente si hablamos de aplicarlo a los arreglos. Pero lo que quiero decir es que los functors promueven el uso de cadenas de funciones para implementar un procedimiento.

## Fuentes
- [Why is map called map?](https://dev.to/techgirl1908/why-is-map-called-map-2l03)
- [Fantasy land](https://github.com/fantasyland/fantasy-land)
- [Static land](https://github.com/fantasyland/static-land)
- [funcadelic.js](https://github.com/thefrontside/funcadelic.js)
- [How to deal with dirty side effects in your pure functional JavaScript](https://jrsinclair.com/articles/2018/how-to-deal-with-dirty-side-effects-in-your-pure-functional-javascript/)
- [What’s more fantastic than fantasy land? An Introduction to Static land](https://jrsinclair.com/articles/2020/whats-more-fantastic-than-fantasy-land-static-land/)
- [Your easy guide to Monads, Applicatives, & Functors](https://medium.com/@lettier/your-easy-guide-to-monads-applicatives-functors-862048d61610)

