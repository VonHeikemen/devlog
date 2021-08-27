+++
title = "Cómo combinar efectos y funciones puras en javascript"
description = "algunas ideas de cómo usar las funciones puras en el mundo real"
date = 2020-01-10
lang = "es"
[taxonomies]
tags = ["javascript", "paradigma-funcional"]
[extra]
canonical_url = "https://dev.to/vonheikemen/como-combinar-efectos-y-funciones-puras-en-javascript-38go"
shared = [
  ["dev.to", "https://dev.to/vonheikemen/como-combinar-efectos-y-funciones-puras-en-javascript-38go"],
  ["Hashnode", "https://vonheikemen.hashnode.dev/dealing-with-side-effects-and-pure-functions-in-javascript-es"]
]
+++

¿Alguna vez han escuchado el término "función pura"? ¿Y "efecto secundario"? Si la respuesta es sí entonces probablemente les han dicho que los efectos secundarios son malos y deben evitarlos a toda costa. Este es el problema, si están usando javascript es muy probable que quieran causar esos efectos (especialmente si les pagan por usar javascript) Entonces la solución no es evitar estos efectos sino controlarlos. Voy a mostrarles algunas maneras en las que pueden combinar los efectos secundarios y las funciones puras.

Antes de empezar vamos a repasar algunos conceptos, para que todos estén al tanto.

## Conceptos

### Función pura

Para no extenderme mucho diré que una función pura es aquella cuyo resultado es determinado por sus parámetros y no tiene ningún efecto observable fuera de su ámbito. El mejor beneficio que proveen es la predictibilidad, dado un conjunto de valores de entrada siempre devolverán el mismo resultado. Veámos algunos ejemplos.

Esta es una función pura.

```js
function increment(number) {
  return number + 1;
}
```

Esta no.

```js
Math.random();
```

Y estas son debatibles.

```js
const A_CONSTANT = 1;

function increment(number) {
  return number + A_CONSTANT;
}

module.exports ={
  increment
};
```

```js
function a_constant() {
  return 1;
}

function increment(number) {
  return number + a_constant();
}
```

### Efecto secundario

Llamaremos efecto secundario a cualquier cosa que afecte la "pureza" de una función. La lista incluye pero no está limitada a:

 - Cambiar (mutar) una variable externa en cualquier forma posible.
 - Mostrar cosas en la pantalla.
 - Modificar un archivo.
 - Hacer una petición http.
 - Crear un proceso.
 - Guardar datos en una base de datos.
 - Ejecutar funciones con efectos secundarios.
 - Cambiar el DOM.
 - Aleatoriedad.

Entonces, cualquier cosa que afecte el "estado del mundo exterior" es un efecto secundario.

## ¿Cómo combinamos esas cosas?

Apuesto a que todavía están pensando en esa lista de efectos, incluye básicamente todo lo que hace que javascript sea útil y aún así hay personas que dicen debes evitarlos cómo sea. No tengan miedo, yo les tengo algunas sugerencias.

### Composición de funciones

Otra forma de describir lo que voy decir sería esta: separación de responsabilidades. Este es la manera más simple. Si tienen la oportunidad de separar un cálculo/transformación de un efecto entonces trasladen esa transformación a una función y usen el resultado en el bloque que contiene el efecto.

En ocasiones puede ser tan simple como este caso.

```js
function some_process() {
  const data = get_data_somehow();
  const clean_data = computation(data);
  const result = save(clean_data);

  return result;
}
```

Ahora bien, `some_process` sigue siendo una función impura pero eso está bien, esto es javascript, no necesitamos que todo sea puro, lo que queremos es mantener la cordura. Al separar los efectos de un cálculo puro hemos creados tres funciones independientes que resuelven un problema a la vez. Pueden incluso ir más allá y utilizar una función como [pipe](https://ramdajs.com/docs/#pipe) para eliminar esos valores intermedios y crear una composición más directa.

```js
const some_process = pipe(get_data_somehow, computation, save);
```

Pero ahora hemos creado otro problema, ¿Qué pasa si queremos insertar un efecto en medio de esa cadena? ¿Qué hacemos? Bueno, si una función nos metió en este problema yo digo que usemos otra para salir. Esto servirá.

```js
function tap(fn) {
  return function (arg) {
    fn(arg);
    return arg;
  }
}
```

Esta función nos permitirá colocar un efecto en nuestra cadena sin afectar la composición.

```js
const some_process = pipe(
  get_data_somehow,
  tap(console.log),
  computation,
  tap(a_side_effect),
  save
);
```

Algunos dirán que este tipo de cosas hacen que la lógica de la función esté esparcida por todos lados y ahora tienen que buscar más de lo necesario para saber qué hace la función. A mí no me molesta mucho, es asunto de preferencias. Suficiente de eso, hablemos de los argumentos de la función `tap`, mirénlo `tap(fn)` acepta una función cómo parámetro, vamos a ver cómo podemos usar eso para otras cosas.

### Haz que otro se encargue del problema

Como todos sabemos la vida no siempre es tan simple, habrá ocasiones en las que simplemente no podemos hacer esa bonita cadena de funciones. A veces necesitamos colocar un efecto en medio de un proceso y cuando eso pasa siempre podemos hacer trampa. Javascript nos permite usar las funciones como si fuera un valor cómun (como un número) y esto nos da la oportunidad de hacer algo gracioso como usar una función como parámetro de otra función (lo que llaman callback). De esta forma una función "pura" puede mantener su predictibilidad y al mismo tiempo proveer la flexibilidad de ejecutar un efecto cuando sea conveniente. 

Digamos por ejemplo que tenemos una función que ya es pura que transforma los valores de una colección pero por alguna razón ahora necesitamos escribir en un log el valor original y el nuevo pero justo después de la transformación. Lo que podemos hacer es añadir una función como parámetro y llamarla en el momento justo.

```js
function transform(onchange, data) {
  let result = Array.isArray(data) ? [] : {};
  for(let key in data) {
    result[key] = data[key] + 1;
    onchange(data[key], result[key]);
  }

  return result;
}
```

Esto técnicamente cumple los requisitos de una función pura, el resultado (y comportamiento) de la función está determinado por sus parámetros, sólo que da la casualidad que uno de esos parámetros es una función que puede tener un efecto secundario. De nuevo, la meta no es pelear contra la naturaleza de javascript hacer que todo sea 100% puro, lo que queremos es controlar estos efectos, en este caso quien controla si se debe tener un efecto es quien llama a nuestra función y provee los parámetros. Un beneficio extra que tenemos de esto es que podemos reusar la función en pruebas unitarias sin tener que instalar una librería extra, lo único que tenemos que hacer suministrar parámetros y evaluar el resultado.

Tal vez se estén preguntando por qué pongo el callback como primer parámetro, es cuestión de preferencia. Si ponen el valor que cambia con más frecuencia en la última posición se les hace más fácil aplicar parcialmente los argumentos, con esto me refiero a vincular parámetros a una función sin ejecutarla. Pueden por ejemplo usar `transform.bind` para crear una función especializada que ya tenga el valor `onchange` y que sólo espere el argumento `data`.

### Efecto tardío

La idea aquí es retrasar lo inevitable. En lugar de ejecutar un efecto en seguida lo que queremos hacer es darle la oportunidad a quien usa nuestra función de decidir cuándo se debe ejecutar el efecto. Podemos hacer de varias maneras.

#### Devolviendo funciones

Como mencioné antes, en javascript podemos tratar a las funciones como un valor y una cosa que hacemos con frecuencia es devolver valores de funciones. Estoy hablando de funciones que devuelven funciones, ya vimos lo útil que puede ser y no es tan inusual si lo piensan bien, ¿Cuántas veces han visto algo como esto? 

```js
function Stuff(thing) {
  
  // preparar datos

  return {
    some_method() {
      // código...
    },
    other() {
      // código...
    }
  }
}
```

Esto es una especie de constructor. Antes, en la era del ES5 esta era una de las maneras en las que se podía imitar el comportamiento de una clase. Es una función normal que devuelve un objeto, y como todos sabemos los objetos pueden tener métodos. Lo que queremos hacer es muy parecido, queremos convertir un bloque que contiene un efecto y devolverlo.

```js
function some_process(config) {

  /*
   * Hacemos algo con `config`
   */

  return function _effect() {
   /*
    * aquí podemos tener cualquier cosa
    */ 
  }
}
```

Así es como le damos la oportunidad a quien llama nuestra función de usar el efecto cuando quieran, y pueden incluso pasarlo a otras funciones o usarla en una cadena (como la que hicimos antes). Este patrón no es muy común, tal vez es porque podemos usar otros métodos para lograr la misma meta.

#### Usando estructuras

Otra forma de retrasar un efecto es envolverlo en una estructura. Lo que queremos hacer es tratar un efecto como un valor cualquiera, tener la habilidad de manipularlo e incluso combinarlo con otros efectos de una manera "segura," es decir sin ejecutarlos. Probablemente ya han visto este patrón antes, un ejemplo que puedo dar es con lo que llaman "Observables." Vean este ejemplo que utiliza rxjs.

```js
// extraído de:
// https://www.learnrxjs.io/operators/creation/create.html

/*
  Incrementa el valor cada segundo, emite valores de los números pares
*/
const evenNumbers = Observable.create(function(observer) {
  let value = 0;
  const interval = setInterval(() => {
    if (value % 2 === 0) {
      observer.next(value);
    }
    value++;
  }, 1000);

  return () => clearInterval(interval);
});
```

El resultado de `Observable.create` no sólo retrasa la ejecución de `setInterval` sino que también nos da la oportunidad de usar `evenNumber.pipe` para crear una cadena de observables que también pueden contener otros efectos. Claro que los Observables y rxjs no son la única manera, nosotros podemos crear nuestro propia estructura para los efectos. Si queremos crear nuestros propios efectos lo único que necesitamos es una función para ejecutarlos y otra para combinarlos.

```js
function Effect(effect) {
  return {
    run(...args) {
      return effect(...args);
    },
    map(fn) {
      return Effect(arg => fn(effect(arg)));
    }
  };
}

```

Puede que no sea mucho pero esto es suficiente para tener algo útil. Con esto ya pueden empezar a combinar efectos sin causar cambios en su ambiente. Por ejemplo.

```js
const persist = (data) => {
  console.log(`guardando ${data} en la base de datos...`);
  return data.length ? true : false;
};
const show_message = result => result 
  ? console.log('todo bien') 
  : console.log('no estamos bien');

const save = Effect(persist).map(show_message);

save.run('algo');
// guardando algo en la base de datos...
// todo bien

save.run('');
// guardando  en la base de datos....
// no estamos bien
```

Si alguna vez han usado `Array.map` para transformar datos de un arreglo se sentirán como en casa usando `Effect`, todo lo que tienen que hacer es suministrar los efectos y al final de la cadena tendrán una función que sabrá qué hacer cuando estén listos para ejecutarla.

Esta es sólo una muestra de lo que pueden hacer con `Effect`, si quieren aprender un poco más busquen por ahí el término `functor` y `IO monad`, ahí tienen diversión para un buen rato.

## ¿Ahora qué?

Ahora espero que puedan echarle un vistazo al enlace que está al final, es un articulo en inglés que explica con mejor detalle todo esto que yo describí aquí.

Espero que ahora tengan el conocimiento y la confianza para empezar a escribir funciones puras en su código y poder combinarlas con los efectos prácticos que pueden hacer con javascript.

## Fuente
- [How to deal with dirty side effects in your pure functional JavaScript](https://jrsinclair.com/articles/2018/how-to-deal-with-dirty-side-effects-in-your-pure-functional-javascript/)
