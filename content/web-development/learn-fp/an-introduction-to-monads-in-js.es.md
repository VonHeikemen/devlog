+++
title = "Una introducción a las mónadas (en javascript)" 
description = "Donde intentamos usar javascript para explicar que son los mónadas"
date = 2020-09-26
lang = "es"
[taxonomies]
tags = ["javascript", "paradigma-funcional"]
[extra]
shared = [
  ["dev.to", "https://dev.to/vonheikemen/una-introduccion-a-las-monadas-en-javascript-4hg0"],
  ["Hashnode", "https://vonheikemen.hashnode.dev/an-introduction-to-monads-in-js-es"]
]
+++

Las infames mónadas. Innombrables en el mundo javascript. Hoy hablaremos de ellas, para ser más específico lo que haremos será "revisar" una definición de mónadas que leí por ahí, la única que no hace que mi cerebro explote. Para mantener nuestra cordura intacta sólo vamos a explorar los aspectos que podemos modelar fácilmente usando javascript. ¿Todo el mundo listo? Comencemos.

Aquí vamos. Esta será fácil, se los juro. Las mónadas son...

> functores puntiagudos que pueden aplanarse.

Dijeron que estaban listos. En fin, podemos con esto. Sólo tienen que conocer cuál es el comportamiento de un functor y los demás será pan comido.

## Presentando a los Functores

Si hablamos de javascript, la forma más común de implementar un functor es creando una especie contenedor con una característica especial: debe permitirnos transformar el valor interno en cualquier forma que nosotros queramos sin tener que dejar el contenedor. 

¿Acaso no suena interesante? ¿Cómo se vería eso en código? Intentemos creando el functor más simple que podamos imaginar.

### La Caja

```js
function Caja(data) {
  return {
    map(fn) {
      return Caja(fn(data));
    }
  }
}
```

Muy bien ¿qué ocurre aquí? Bueno, tenemos una `Caja` diseñada específicamente para almacenar un valor que llamamos `data` y la única manera de llegar a ese valor es a través del método `map`. En esta instancia `map` recibe una función `fn` (un callback) como argumento, aplica esta función a `data` y coloca el resultado de la función en una nueva `Caja`. No todos los functores lucen así, pero en general todos siguen este patrón. Ahora vamos a usarlo.

```js
const xbox = Caja('x');
const to_uppercase = (str) => str.toUpperCase();

xbox.map(to_uppercase).map(console.log);
// => X
// => Object { map: map() }
```

Entonces, tenemos esta `Caja` que es um... totalmente inútil. Sip, y eso es a propósito. Verán, lo que tenemos aquí es el functor `Identidad`. Su utilidad en el "mundo real" es debatible pero para ilustrar el patrón de los functors con fines educativos funciona de maravilla.

Muy bonito todo ¿Pero cuáles son los beneficios que nos traen estas cosas, los functores? Al agregar esta pequeña abstracción obtenemos la habilidad de separar un "efecto" de una computación pura. Para aclarar un poco mi punto vamos a darle un vistazo a un functor que sí tiene un propósito.

### Un rostro familiar

No sé si están al tanto o no pero les diré de todas formas, los arreglos siguen el patrón que les acabo de describir. Prueben esto.

```js
const xbox = ['x'];
const to_uppercase = (str) => str.toUpperCase();

xbox.map(to_uppercase);
// => Array [ "X" ]
```

El arreglo es un contenedor, tiene un método `map` el cual nos permite transformar el contenido del arreglo, y los nuevos valores que se originan de la función son puestos nuevamente en un arreglo.

Bien, pero ahora ¿Cuál es el "efecto" de un arreglo? Ellos nos permiten almacenar múltiples valores en una sola estructura, eso es lo que hacen. `Array.map` en particular se asegura de aplicar una función a cada elemento del arreglo. No importa si tienen un arreglo con 100 elementos o uno que esté vacío, `.map` se encarga de la lógica que dicta **cuando** debe ejecutarse la función para que ustedes se concentren en **qué** deben hacer con el elemento dentro de la estructura.

Y por supuesto los functores se pueden usar para muchas otras cosas, como el manejo de errores o validar la ausencia de valores e incluso para procesos asíncronos. Me gustaría seguir hablando de este tema pero debemos seguir con la definición de mónada.

## La parte puntiaguda

Necesitamos que nuestros functores sean "puntiagudos". Esta es una manera graciosa de decirnos que necesitamos una función auxiliar que pueda colocar cualquier valor ordinario dentro de la unidad más simple de nuestra estructura. Esta función es conocida como "pure", otros nombres también incluyen "unit" y "of".

Volvamos a visitar a nuestro viejo amigo `Array`. ¿Si colocamos un valor dentro de la unidad más simple de una arreglo, qué tenemos? Sí, un arreglo con un solo elemento. Curiosamente hay una función que puede hacer eso por nosotros.

```js
Array.of('¿en serio?');
// => Array [ "¿en serio?" ]

Array.of(42);
// => Array [ 42 ]

Array.of(null);
// => Array [ null ]
```

Algo como esto puede ser especialmente útil si la forma normal de crear un functor es complicada. Con esta función podríamos envolver cualquier valor que queramos y empezar a usar `.map` inmediatamente. Podría contarles más sobre esta función pero esa es básicamente la idea. Sigamos.

## A Planilandia

Ya estamos llegando al corazón del problema. Esperen... ¿cuál es exactamente el problema?

Imaginen esta situación, tenemos un número en una `Caja` y queremos usar `.map` para aplicar una función que llamaremos `accion`. Algo así.

```js
const numero = Caja(41);
const accion = (numero) => Caja(numero + 1);

const resultado = numero.map(accion);
```

Todo parece estar bien hasta que nos damos cuenta que `accion` nos regresa otra `Caja`. Entonces `resultado` es de hecho una `Caja` dentro de otra `Caja`: `Caja(Caja(42))`. Ahora para acceder al valor tendríamos que hacer esto.

```js
resultado.map((caja) => caja.map((valor) => {/* código */}));
```

Eso no está bien. Nadie quiere lidiar con una estructura así. Aquí es donde las mónadas pueden ayudarnos. Ellas nos dan la "habilidad" de fusionar estas capas innecesarias que crean una estructura anidada. En nuestro caso puede transformar `Caja(Caja(42))` en `Caja(42)`. ¿Cómo? Con la ayuda de un método llamado `join`.

Así sería la implementación en nuestra `Caja`.

```diff
  function Caja(data) {
    return {
      map(fn) {
        return Caja(fn(data));
      },
+     join() {
+       return data;
+     }
    }
  }
```

Ya sé lo que están pensando, no parece que esté fusionando nada. Quizá hasta estén pensando en cambiarle el nombre al método y ponerle "extract". Sólo esperen un momento. Volvamos a nuestro ejemplo con `accion`, vamos a arreglarlo.

```js
const resultado = numero.map(accion).join();
```

Ahora sí tenemos una `Caja(42)`, con esto podemos acceder al valor que queremos usando un solo `.map`. ¿Qué? ¿Por qué me miran así? Bien, digamos que le cambio el nombre. Ahora es así.

```js
const resultado = numero.map(accion).extract();
```

Este es el problema, si leo esa línea por sí sola yo asumiría que `resultado` es un valor ordinario, algo que pueda usar libremente, me voy a molestar un poco cuando descubra que en realidad tengo una `Caja`. Por otra parte, si veo `join` sé que `resultado` aún es una mónada y puedo prepararme para ello.

Ahora pueden estar pensando "Bien, ya entendí ¿Pero sabes qué? Yo uso javascript, simplemente voy a ignorar totalmente los functores y no necesitaré esas mónadas". Totalmente válido, pueden hacer eso. La mala noticia es que **los arreglos son functores** así que no pueden escapar de ellos. La buena noticia es que **los arreglos son mónadas** así que cuando se encuentren con ese problema de estructuras anidadas (y lo harán) pueden arreglarlo fácilmente.

Los arreglos no tienen un método `join`... bueno, sí lo tienen pero se llama `flat`. Contemplen.

```js
[[41], [42]].flat();
// => Array [ 41, 42 ]
```

Y ahí lo tienen, después de llamar a `flat` pueden seguir con sus vidas sin tener que preocuparse por "capas" innecesarias entorpeciendo su camino. Eso es todo, en la práctica esto es básicamente el problema que las mónadas resuelven.

Pero antes de irme quiero decirles una cosa más.

## Mónadas en secuencia

Resulta que esta combinación de `map/join` es tan común que hay un método que combina las características de esos dos. Este también tiene varios nombres: "chain", "flatMap", "bind", ">>=" (en haskell). Los arreglos lo llaman `flatMap`.

```js
const split = str => str.split('/');

['some/stuff', 'another/thing'].flatMap(split);
// => Array(4) [ "some", "stuff", "another", "thing" ]
```

¿Acaso no es genial? En lugar de tener dos arreglos anidados sólo tenemos un gran arreglo. Esto es mucho más fácil de manejar que una estructura anidada.

Pero esto no sólo es para ahorrar unos cuantos caracteres, también fomenta la composición de funciones de la misma forma que `.map` lo hace. Podrían hacer algo como esto.

```js
monad.flatMap(action)
  .map(another)
  .map(cool)
  .flatMap(getItNow);
```

No estoy diciendo que hagan esto con los arreglos. Les estoy diciendo que si crean sus propias mónadas pueden combinar funciones de esta manera. Sólo tienen que recordar si su función retorna una mónada usan `flatMap`, si no usan `map`.

## Conclusión

Aprendimos que las mónadas son functores con características extras. En otras palabras son contenedores mágicos que... ¿no les gusta tener otros contenedores internamente? Intentemos nuevamente: son como cebollas mágicas que... no importa, son mágicos, dejémoslo así.

Podemos usarlos para añadir un "efecto" a cualquier valor ordinario. Podemos usarlos para el manejo de errores, operaciones asíncronas, controlar efectos secundarios, y un montón de cosas más.

También aprendimos que a las mónadas se les quiere o se les tiene un odio irracional, y no hay ningún punto medio.

## Fuentes
- [Professor Frisby's Mostly Adequate Guide to Functional Programming. Chapter 9: Monadic Onions](https://mostly-adequate.gitbooks.io/mostly-adequate-guide/content/ch09.html)
- [Funcadelic.js](https://github.com/thefrontside/funcadelic.js)
- [Fantasy Land](https://github.com/fantasyland/fantasy-land)

