+++
title = "Uniones discriminadas y Fantasy Land" 
description = "Usaremos las uniones discriminadas para explorar una rama de Fantasy Land"
date = 2020-05-24
lang = "es"
[taxonomies]
tags = ["javascript", "paradigma-funcional"]
+++

Vamos a hacer algo divertido, vamos a explorar una de las ramas de la especificación [Fantasy Land](https://github.com/fantasyland/fantasy-land) usando uniones discriminadas como nuestro medio transporte. Para no extendernos más de lo necesario vamos a enfocarnos más que todo en el cómo funcionan las cosas y dejaremos de lado muchos detalles. Entonces, lo que haremos será crear una estructura y ver si podemos seguir las reglas que aparecen en la especificación.

## Uniones Discriminadas

También conocidas como *variantes*, son un tipo de estructura que nos permiten modelar un valor que puede tener diferentes estados. En cualquier punto del tiempo sólo pueden representar uno de sus posibles estados. Otras características importantes incluyen la capacidad de almacenar información sobre ellas mismas así como también una "carga" extra que puede ser cualquier cosa. 

Todo eso suena bien hasta que nos damos cuenta que no tenemos esas cosas en javascript. Si queremos usarlas tendremos que recrearlas nosotros mismos. Por suerte para nosotros no necesitamos una implementación a prueba de balas. Sólo necesitamos un par de cosas, saber el tipo de variante de una variable y también una forma de llevar información. Podemos con eso.

```js
function Union(types) {
  const target = {};
  
  for(const type of types) {
    target[type] = (data) => ({ type, data });
  }

  return target;
}
```

¿Qué tenemos aquí? Pueden pensar en `Union` como una fábrica de constructores. Acepta como argumento una lista de variantes y por cada una creará un constructor. Mejor les muestro. Digamos que queremos modelar los posibles estados de una tarea, usando `Union` podemos crear algo así.

```js
const Status = Union(['Success', 'Failed', 'Pending']);
```

Ahora tenemos una forma de crear variantes de nuestro tipo `Status`.

```js
Status.Success({ some: 'stuff' });
// { "type": "Success", "data": { "some": "stuff" } }
```

Con la propiedad `type` podemos saber con qué variante estamos tratando y en `data` podemos poner cualquier valor que se nos ocurra. Ya habrán notado que sólo usamos el nombre de la variante en `type`, esto puede causar colisiones con otras variantes de diferente tipo, lo mejor sería agregar más información en la función `Union` pero vamos a dejarlo así.

Si este patrón les parece útil y necesitan algo confiable, consideren usar una librería en lugar de hacer su propia implementación. Pueden usar [tagmeme](https://www.npmjs.com/package/tagmeme) o [daggy](https://www.npmjs.com/package/daggy) o cualquier otra.

## Fantasy Land

La descripción en github dice lo siguiente: 

> Especificación de interoperabilidad de estructuras algebraicas comunes en javascript.

¿Estructuras algebraicas? ¿Qué? Ya sé, los entiendo. Y la definición formal tampoco ayuda mucho. Lo mejor que puedo hacer es ofrecerles una definición vaga que los deje con una mínima cantidad de dudas, aquí voy: Las estructuras algebraicas son la combinación de un conjunto de valores y un conjunto de operaciones que siguen ciertas reglas.

En nuestro caso, pueden pensar en las variantes como nuestro "conjunto de valores" y las funciones que crearemos serán nuestras "operaciones," finalmente las reglas que seguiremos serán las de Fantasy Land.

## La Conexión

Bien, sabemos qué son la uniones discriminadas y tenemos una vaga idea para qué sirve Fantasy Land pero queda la pregunta ¿Cómo conectamos esos dos en la práctica? La respuesta a eso es la *búsqueda de patrones* (pattern matching). Aquellos que están familiarizados con el término saben que tampoco tenemos eso en javascript. Lamentablemente en este caso lo mejor que podemos hacer es intentar imitar algunas de sus características.

¿Cómo comenzamos? Vamos a describir lo que queremos. Necesitamos evaluar una variante, poder determinar qué tipo de variante es y por último ejecutar un bloque de instrucciones. Nuestras variantes tienen la propiedad `type` que es de tipo `String`, podríamos simplemente usar un `switch/case`.

```js
switch(status.type) {
  case 'Success':
    // Todo salió bien
    break;

  case 'Failed':
    // Algo salió mal
    break;

  case 'Pending':
    // Esperando
    break;

  default:
    // Nunca debería pasar
    break;
}
```

Esto se acerca bastante a lo que queremos pero hay un problema, no devuelve nada. Queremos hacer lo mismo que hace este `switch/case` pero en una expresión, algo que nos de un resultado. Para recrear este comportamiento en la forma que queremos usaremos objetos y funciones.

```js
function match(value, patterns) {
  const { type = null } = value || {};
  const _match = patterns[type];

  if (typeof _match == 'function') {
    return _match(value.data);
  } else if (typeof patterns._ == 'function') {
    return patterns._();
  }

  return null;
}
```

Aquí nuevamente aprovechamos el hecho de que `type` es de tipo `String` y lo usaremos para "escoger" el patrón que queremos, pero esta vez transportamos nuestros patrones en un objeto. Ahora bien, cada "patrón" será una función asociada a una propiedad del objeto `patterns` y la función `match` devolverá cualquier cosa que retorne nuestro patrón. Finalmente si el patrón de la variante actual no se encuentra buscará una propiedad llamada `_`, eso actuará como el caso `default` del `switch/case` y si todo falla sólo retorna `null`. Con esto podemos ya obtener el comportamiento que queremos.

```js
match(status, {
  Success: ({ some }) => `Some: ${some}`,
  Failed:  () => 'Oops something went wrong',
  Pending: () => 'Wait for it',
  _:       () => 'AAAAHHHH'
});
// "Some: stuff"
```

Con esta función a nuestra disposición podemos seguir adelante.

## La Estructura

Ahora toca crear la estructura que usaremos de aquí en adelante. Lo que haremos será recrear un concepto popular, un posible fallo. Crearemos un tipo de dato con dos variantes `Ok` y `Err`, a este tipo lo llamaremos `Result`. La idea es simple, la variante `Ok` va a representar una operación exitosa y será usada para transportar un valor, todas nuestras operaciones serán basadas en esta variante. Es decir que en caso de que la variante sea de tipo `Err` queremos ignorar cualquier tipo de transformación, lo único que haremos será "propagar el error."

```js
const Result = Union(['Ok', 'Err']);
```

## Las Operaciones

Antes de comenzar a crear nuestras operaciones vamos a crear una función `match` específica para nuestra estructura.

```js
Result.match = function(err, ok, data) {
  return match(data, {Ok: ok, Err: err});
};
```

Ya todo está en su lugar. Como dije antes, solamente nos enfocaremos en una sola rama de la especificación, exploraremos esa que va desde `Functor` hasta `Monad`. Por cada una de estas operaciones vamos a implementar un método estático en nuestro objeto `Result` y además intentaré explicar cómo funciona y para qué sirve. 

La lógica dicta que deberíamos empezar con Functor pero vamos a tomar otro camino.

### Chain

La operación `chain` nos permite interactuar con el valor que se encuentra "dentro" de una estructura y transformarla completamente. ¿Suena fácil, verdad? Nosotros hacemos eso todo el tiempo, pero esta vez debemos seguir unas reglas. Les presento la primera ley del día.

* Asociatividad

```js
Val.chain(Fx).chain(Gx);
// es equivalent a
Val.chain(v => Fx(v).chain(Gx));
```

> Noten que el comentario dice "equivalente a" aunque en muchos casos estas pruebas deben dar resultados idénticos, no necesariamente se trata de una comparación de igualdad, debería interpretarse más como "deben tener el mismo efecto."

Esta ley nos habla del orden de las operaciones. En la primera sentencia se puede ver como una secuencia, va una función detrás de la otra. En la segunda sentencia vemos cómo una operación "envuelve" a la otra. Y esto es interesante ¿ven esto `Fx(value).chain(Gx)`? El segundo `chain` viene directo del resultado de `Fx`. Tanto `Fx` como `Gx` son funciones que retornan estructuras que también siguen esta ley.

Vamos a ver esto en la práctica con una estructura que todos conocemos, los arreglos. Resulta que los arreglos siguen esta ley (algo así). Puede que en la clase `Array` no exista el método `chain` pero sí tiene `flatMap` el cual debería comportarse de igual manera.

```js
const to_uppercase = (str) => str.toUpperCase();
const exclaim      = (str) => str + '!!';

const Val = ['hello'];

const Uppercase = (str) => [to_uppercase(str)];
const Exclaim   = (str) => [exclaim(str)];

const one = Val.flatMap(Uppercase).flatMap(Exclaim);
const two = Val.flatMap(v => Uppercase(v).flatMap(Exclaim));

one.length === two.length;
// true

one[0] === two[0];
// true
```

Entonces `flatMap` nos dejó interactuar con el texto dentro del arreglo y transformarlo usando una función y no importó si el segundo `flatMap` estuviera dentro o fuera del primero, el resultado es el mismo.

Ahora veamos con nuestra estructura. Como mencioné antes, nosotros haremos todas nuestras operaciones con métodos estáticos, así que nuestro ejemplo se verá algo diferente. Esta sería nuestra implementación de `chain`.

```js
Result.chain = Result.match.bind(null, Result.Err);
```

Gracias al poder de la conveniencia `Result.match` ya contiene la lógica que necesitamos, sólo tenemos proveer un valor para el parámetro `err` y lograremos el efecto que queremos. Entonces tenemos que `Result.chain` es una función que espera por el parámetro `ok` y `data`. Si la variante es de tipo `Err` el error quedará envuelto nuevamente en una variante del mismo tipo, como si nada hubiera pasado. Si la variante es de tipo `Ok` ejecutará la función que le pasemos como primer argumento.

```js
const Val = Result.Ok('hello');

const Uppercase = (str) => Result.Ok(to_uppercase(str));
const Exclaim   = (str) => Result.Ok(exclaim(str));

const one = Result.chain(Exclaim, Result.chain(Uppercase, Val));
const two = Result.chain(v => Result.chain(Exclaim, Uppercase(v)), Val);

one.type === two.type
// true

one.data === two.data;
// true
```

Ya que nuestra función cumple con la ley tenemos una forma de crear una composición entre funciones que retornan estructuras de este tipo. Esto resulta especialmente útil cuando se está creando una cadena de funciones donde los argumentos de una función son los resultados de la anterior.

`Result.chain` no sólo sirve para cumplir esta ley, también podemos usarla para construir otras funciones. Comencemos creando una que nos permita "extraer" el valor de nuestra estructura.

```js
const identity = (arg) => arg;

Result.join = Result.chain.bind(null, identity);
```

`Result.join` es una función que sólo espera por el parámetro `data` (este es el milagro de la [aplicación parcial](@/web-development/learn-fp/partial-application.es.md)).

```js
const good_data = Result.Ok('Hello');
Result.join(good_data);
// "Hello"

const bad_data = Result.Err({ message: 'Ooh noes' });
Result.join(bad_data);
// { "type": "Err", "data": { "message": "Ooh noes" } }
```

Esta función se llama `join` porque se supone que debería usarse para "aplanar" una estructura anidada. Algo como en este caso.

```js
const nested_data = Result.Ok(Result.Ok('Hello'));

Result.join(nested_data);
// { "type": "Ok", "data": "Hello" }
```

Pero yo voy a abusar de la naturaleza de esta función para comparar el contenido dentro de las estructuras en nuestras pruebas. Para dejar en claro mis intenciones voy a crear un "alias."

```js
Result.unwrap = Result.join;
```

### Functor

Si han estado leyendo otros artículos sobre el paradigma funcional en javascript el nombre tal vez les parezca familiar. Incluso si no lo conocen es probable que los hayan usado sin saber. Esta especificación es la que introduce a nuestro viejo amigo `.map`. Veamos qué la hace tan especial.

* Identidad

```js
Val.map(v => v);
// es equivalente a
Val;
```

Aunque no lo parezca esta ley es interesante. Esa función que aparece en la primera sentencia, `v => v`, ¿Les parece familiar? Ya usamos una de esas antes, se le conoce como la función identidad (`identity`). Verán, en matemática un elemento identidad es aquel que no tiene ningún efecto sobre una operación, y eso es exactamente lo que hace esta función. Pero lo interesante no es lo que está en la superficie, sino lo que no podemos ver. Si la primera sentencia es igual a la segunda eso quiere decir que `.map(v => v)` retorna otra estructura del mismo tipo, incluso si le pasamos la función más inútil que nos podemos imaginar. Usemos nuevamente un arreglo para ilustrar esta ley.

```js
const identity = (arg) => arg;

const Val = ['hello'];
const Id  = Val.map(identity);

Array.isArray(Val) === Array.isArray(Id);
// true

Val.length === Id.length;
// true

Val[0] === Id[0];
// true
```

¿Pero cómo nos ayuda eso? La parte importante es que `.map` debe "preservar la forma" de nuestra estructura. En el caso de los arreglos, si la ejecutamos en un arreglo de 1 elemento devuelve un arreglo de 1 elemento, si la ejecutamos con un arreglo de 100 elementos devuelve otro arreglo de 100 elementos. Si tenemos la garantía de que el resultado será una estructura del mismo tipo eso nos permite hacer cosas como estas.

```js
Val.map(fx).map(gx).map(hx);
```

Sé lo que están pensando. Usar `.map` de esa manera en un arreglo puede tener un impacto terrible en el desempeño de nuestros programas. No se preocupen, tenemos eso cubierto con nuestra segunda ley.

* Composición

```js
Val.map(v => fx(gx(v)));
// es equivalente a
Val.map(gx).map(fx);
```

Esta ley nos dice que podemos reemplazar llamadas consecutivas a `.map` si combinamos directamente las funciones que usamos como argumentos. Probemos.

```js
const Val = ['hello'];

const one = Val.map(v => exclaim(to_uppercase(v)));
const two = Val.map(to_uppercase).map(exclaim);

one[0] === two[0];
// true
```

`.map` nos da la habilidad de combinar funciones en diferentes formas, esto nos da la oportunidad de optimizar nuestro código para la velocidad o legibilidad. La composición de funciones es un tema muy amplio, me gustaría extenderme y decirles muchas cosas pero no tenemos tiempo para eso ahora. Si sienten curiosidad puede leer este artículo: [técnicas de composición](@/web-development/learn-fp/composition-techniques.es.md).

Es hora de implementar el famoso `.map` para nuestra estructura. Como habrán notado este método tiene muchas similitudes con `.chain`, de hecho es casi igual excepto por una cosa, con `.map` tenemos la garantía de que el resultado será una estructura del mismo tipo.

```js
Result.map = function(fn, data) { 
  return Result.chain(v => Result.Ok(fn(v)), data)
};
```

Si recuerdan, `.chain` sólo ejecutará la función del primer argumento si `data` es una variante de tipo `Ok`, entonces lo único que debemos hacer para mantener la estructura es usar `Result.Ok` en el resultado `fn`.

```js
const Val = Result.Ok('hello');

// Identidad
const Id = Result.map(identity, Val);

Result.unwrap(Val) === Result.unwrap(Id);
// true

// Composición
const one = Result.map(v => exclaim(to_uppercase(v)), Val);
const two = Result.map(exclaim, Result.map(to_uppercase, Val));

Result.unwrap(one) === Result.unwrap(two);
// true
```

### Apply

Esta es difícil, es mejor explicarlo después de entender la ley que rige esta operación.

* Composición

```js
Val.ap(Gx.ap(Fx.map(fx => gx => v => fx(gx(v)))));
// es equivalente a
Val.ap(Gx).ap(Fx);
```

"¿Quééé?"

Sí, yo pensé lo mismo. Esa primera sentencia es la más confusa que hemos visto hasta ahora. Parece que `Fx` y `Gx` no son funciones, son estructuras. `Gx` tiene un método `ap` así que debe ser del mismo tipo que `Val`. Si vemos más allá tenemos que `Fx` tiene un método llamado `map`, eso quiere decir que es un Functor. Entonces `Val`, `Fx` y `Gx` deben implementar la especificación Functor y Apply para que esto funcione. La última pieza es esta `Fx.map(fx => ... fx(...))`, sí hay funciones involucradas en esta ley pero están encerradas dentro de una estructura.

El nombre de la ley y la segunda sentencia nos dice que esto se trata de combinar funciones. Estoy pensando que el comportamiento de esto es el mismo que `.map` pero con un giro en la trama, la función que recibimos como argumento está atrapada dentro un Functor. Ya tenemos suficiente información para intentar implementar nuestro método.

```js
Result.ap = function(res, data) {
  return Result.chain(v => Result.map(fn => fn(v), res), data);
};
```

¿Qué está pasando aquí? Bueno, déjenme explicar. Primero extraemos el valor dentro de `data` si todo sale bien.

```js
Result.chain(v => ..., data);
```

En este punto tenemos un problema, `.chain` no nos da ninguna garantía sobre el resultado, puede retornar cualquier cosa. Pero sabemos que `res` es un Functor, entonces podemos usar `.map` para salvar el día.

```js
Result.map(fn => ..., res);
```

`.map` cumple una doble labor, nos da acceso a la función dentro de `res` y nos ayuda a "preservar la forma de la estructura." Entonces `.chain` va a devolver lo que nos de `.map`, esto nos da la confianza para poder combinar varias llamadas a `.ap`, lo que crea nuestra composición. Por último tenemos esto.

```js
fn(v);
```

Es lo que de verdad queremos de `.ap`. El resultado de esa expresión queda en una variante de tipo `Ok` gracias a `map` y esta va al mundo exterior gracias a `chain`. Ahora vienen las pruebas.

```js
const Val = Result.Ok('hello');

const composition = fx => gx => arg => fx(gx(arg));
const Uppercase   = Result.Ok(to_uppercase);
const Exclaim     = Result.Ok(exclaim);

const one = Result.ap(Result.ap(Result.map(composition, Exclaim), Uppercase), Val);
const two = Result.ap(Exclaim, Result.ap(Uppercase, Val));

Result.unwrap(one) === Result.unwrap(two);
// true
```

Todo eso es genial ¿pero de qué nos sirve? Poner una función dentro de `Result.Ok` no parece algo que ocurre con frecuencia. ¿Por qué alguien haría eso? Todas son preguntas válidas. Luce confuso porque el método `.ap` sólo es la mitad de la historia.

`.ap` con frecuencia se usa para crear una función auxiliar llamada `liftA2`. El objetivo de esta función es tomar una función común y hacer que trabaje con valores que están encerrados dentro de una estructura. Algo como esto.

```js
const Title = Result.Ok('Dr. ');
const Name  = Result.Ok('Acula');

const concat = (one, two) => one.concat(two);

Result.liftA2(concat, Title, Name);
// { "type": "Ok", "data": "Dr. Acula" }
```

Pueden pensar en `liftA2` como la versión extendida de `.map`. Mientras que `.map` trabaja con funciones que sólo aceptan un argumento, `liftA2` trabaja con funciones que aceptan dos argumentos. Pero ahora la pregunta es ¿cómo funciona `liftA2`? La respuesta está en este fragmento.

```js
const composition = fx => gx => arg => fx(gx(arg));
Result.ap(Result.ap(Result.map(composition, Exclaim), Uppercase), Val);
```

Veamos lo que pasa ahí. Todo comienza con `.map`.

```js
Result.map(composition, Exclaim);
```

Esta expresión extrae la función dentro de `Exclaim` y la aplica a `composition`.

```js
fx => gx => arg => fx(gx(arg));
// se transforma en
gx => arg => exclaim(gx(arg));
```

Esa transformación queda en una variante de tipo `Ok` que es lo que `.ap` espera como primer argumento. Entonces lo siguiente que tenemos es esto.

```js
Result.ap(Result.Ok(gx => arg => exclaim(gx(arg))), Uppercase);
```

Ahora que tenemos una función dentro de una variante `.ap` tiene todo lo que necesita para continuar. Aquí básicamente ocurre lo mismo (excepto que nuestro primer argumento ahora es una variante), la función del primer argumento es aplicada al valor dentro de la variante que tenemos como segundo argumento. El resultado es este.

```js
Result.Ok(arg => exclaim(to_uppercase(arg)));
```

¿Ya notaron el patrón? Tenemos otra función dentro una variante, eso es exactamente lo que recibe nuestro último `.ap`.

```js
Result.ap(Result.Ok(arg => exclaim(to_uppercase(arg))), Val);
```

El ciclo se repite nuevamente y finalmente obtenemos.

```js
Result.Ok('HELLO!!');
```

Este es el patrón que `liftA2` sigue. La única diferencia es que en lugar de llevar funciones a un valor, llevamos valores a una función. Ya lo verán.

```js
Result.liftA2 = function(fn, R1, R2) {
  const curried = a => b => fn(a, b);
  return Result.ap(Result.map(curried, R1), R2);
};
```

La probamos otra vez.

```js
const concat = (one, two) => one.concat(two);

Result.liftA2(concat, Result.Ok('Dr. '), Result.Ok('Acula'));
// { "type": "Ok", "data": "Dr. Acula" }
```

¿Quieren hacer un `liftA3`? Ya saben qué hacer.

```js
Result.liftA3 = function(fn, R1, R2, R3) {
  const curried = a => b => c => fn(a, b, c);
  return Result.ap(Result.ap(Result.map(curried, R1), R2), R3);
};
```

Esa es la ley de composición actuando a nuestro favor. Mientras `Result.ap` siga la ley podemos seguir incrementando el número de argumentos que podemos aceptar. Ahora sólo por diversión vamos a crear un `liftN` que pueda aceptar una cantidad arbitraria de argumentos. En esta ocasión necesitaremos ayuda.

```js
function curry(arity, fn, ...args) {
  if(arity <= args.length) {
    return fn(...args);
  }

  return curry.bind(null, arity, fn, ...args);
}

const apply = (arg, fn) => fn(arg);
const pipe  = (fns) => (arg) => fns.reduce(apply, arg);

Result.liftN = function(fn, R1, ...RN) {
  const arity   = RN.length + 1;
  const curried = curry(arity, fn);

  const flipped = data => R => Result.ap(R, data);
  const ap      = pipe(RN.map(flipped));

  return ap(Result.map(curried, R1));
};
```

Esa sería la versión "automatizada" de `liftA3`. Ahora podemos usar todo tipo de funciones.

```js
const concat = (one, ...rest) => one.concat(...rest);

Result.liftN(
  concat,
  Result.Ok('Hello, '),
  Result.Ok('Dr'),
  Result.Ok('. '),
  Result.Ok('Acula'),
  Result.Ok('!!')
);
// { "type": "Ok", "data": "Hello, Dr. Acula!!" }
```

### Applicative

Como habrán notado a estas alturas todo lo que construimos es una especie de extensión de lo anterior, esta no es la excepción. Para que una estructura sea un Applicative primero debe cumplir con la especificación Apply, luego debe agregar un pequeño detalle extra.

El nuevo aporte será un método que nos ayude a construir la unidad más simple de nuestra estructura a partir de un valor. El concepto es similar al de un constructor de una clase, la idea es tener un método que pueda llevar un valor común al "contexto" de nuestra estructura y poder ejecutar cualquier operación de inmediato.

Por ejemplo, con la clase `Promise` podemos hacer esto.

```js
Promise.resolve('hello').then(to_uppercase).then(console.log);
// Promise { <state>: "pending" }
// HELLO
```

Luego de usar `Promise.resolve` nuestro valor `'hello'` queda "dentro" de una promesa y podemos ejecutar sus métodos `then` o `catch` inmediatamente. Si quisiéramos hacer lo mismo usando el constructor tendríamos que hacer esto.

```js
(new Promise((resolve, reject) => { resolve('hello'); }))
  .then(to_uppercase)
  .then(console.log);
// Promise { <state>: "pending" }
// HELLO
```

¿Ven todo el esfuerzo que hay que hacer para lograr el mismo efecto? Es por eso que resulta útil tener un "atajo" para crear una instancia "simple" de nuestra estructura. Es hora de implementarlo en nuestra estructura.

```js
Result.of = Result.Ok;
```

Les aseguro que eso sólo es una coincidencia, no siempre es tan fácil. Pero en serio eso es todo lo que necesitamos y podemos probarlo usando las leyes.

* Identidad

```js
Val.ap(M.of(v => v));
// es equivalente a
Val;
```

Nuestro viejo amigo "identidad" vuelve a presentarse para recordarnos que `.ap` en realidad se parece a `.map`.

```js
const Val = Result.Ok('hello');

const Id = Result.ap(Result.of(identity), Val);

Result.unwrap(Val) === Result.unwrap(Id);
// true
```

* Homomorfismo

```js
M.of(val).ap(M.of(fx));
// es equivalente a
M.of(fx(val));
```

Okey, aquí tenemos un nuevo concepto qué interpretar. Hasta donde pude entender un homomorfismo es una especie de transformación donde se mantiene las capacidades del valor original. Pienso que aquí lo que se quiere probar es que `.of` no tiene ninguna influencia cuando se "aplica" una función a un valor.

```js
const value = 'hello';

const one = Result.ap(Result.of(exclaim), Result.of(value));
const two = Result.of(exclaim(value));

Result.unwrap(one) === Result.unwrap(two);
// true
```

Para recapitular, en la primera sentencia estamos aplicando `exclaim` a `value` mientras ambos están envueltos en nuestra estructura. En la segunda aplicamos `exclaim` a `value` directamente y luego envolvemos el resultado. Ambas sentencias nos dan el mismo resultado. Con esto probamos que `.of` no tiene nada de especial, que sólo está ahí para crear una instancia de nuestra estructura.

* Intercambio

```js
M.of(y).ap(U);
// es equivalente a
U.ap(M.of(fx => fx(y)));
```

Esta es la más difícil de leer. Honestamente no estoy seguro de entender qué se intenta probar aquí. Si tuviera que adivinar diría que no importa de qué lado de la operación `.ap` se encuentre `.of` si podemos tratar su contenido como una constante entonces el resultado será el mismo.

```js
const value   = 'hello';
const Exclaim = Result.Ok(exclaim);

const one = Result.ap(Exclaim, Result.of(value));
const two = Result.ap(Result.of(fn => fn(value)), Exclaim);

Result.unwrap(one) === Result.unwrap(two);
// true
```

### Monad

Para crear un Monad debemos cumplir con la especificación Applicative y Chain. Entonces, lo que debemos hacer ahora es... nada. En serio, ya no hay nada qué hacer. Felicitaciones han creado un Monad ¿Quieren ver unas leyes?

* Identidad - lado izquierdo

```js
M.of(a).chain(f);
// es equivalente a
f(a);
```

Verificamos.

```js
const one = Result.chain(exclaim, Result.of('hello'));
const two = exclaim('hello');

one === two;
// true
```

En este punto se deben estar preguntando ¿No pudimos haber hecho esto después de implementar `.chain` (ya que `.of` es un alias de `Ok`)? La respuesta es sí, pero no sería divertido. Se habrían perdido de todo el contexto.

¿Qué problema resuelve esto? ¿Qué ganamos? Por lo que he visto resuelve un problema muy específico, uno que puede ocurrir con mayor frecuencia si usan Functors, y ese es el de las estructuras anidadas.

Imaginemos que queremos extraer un objeto `config` que está guardado en el `localStorage` de nuestro navegador. Como sabemos que esta operación puede fallar creamos una función que usa nuestra variante `Result`.

```js
function get_config() {
  const config = localStorage.getItem('config');

  return config 
    ? Result.Ok(config)
    : Result.Err({ message: 'Configuración no encontrada' });
}
```

Eso funciona de maravilla. Ahora el problema es que `localStorage.getItem` no devuelve un objeto, la información que queremos está en forma de un `String`.

```js
'{"dark-mode":true}'
```

Por suerte tenemos una función que puede transformar ese texto en un objeto.

```js
function safe_parse(data) {
  try {
    return Result.Ok(JSON.parse(data));
  } catch(e) {
    return Result.Err(e);
  }
}
```

Sabemos que `JSON.parse` puede fallar por eso se nos ocurrió la brillante idea de envolverlo en una "función segura" que también usa nuestra variante `Result`. Ahora intenten unir estas dos funciones usando `.map`.

```js
Result.map(safe_parse, get_config());
// { "type": "Ok", "data": { "type": "Ok", "data": { "dark-mode": true } } }
```

¿No es lo que querían, cierto? Si cerramos los ojos e imaginamos que `get_config` siempre nos da un resultado positivo podríamos reemplazarlo con esto.

```js
Result.of('{"dark-mode":true}');
// { "type": "Ok", "data": "{\"dark-mode\":true}" }
```

Esta ley me dice que si uso `.chain` para aplicar una función a una estructura, es lo mismo que usar dicha función sobre el contenido dentro de la estructura. Aprovechemos eso, ya tenemos la función ideal para este caso.

```js
const one = Result.chain(identity, Result.of('{"dark-mode":true}'));
const two = identity('{"dark-mode":true}');

one === two;
// true
```

Espero que sepan qué haré ahora. Ya lo han visto antes.

```js
Result.join = Result.chain.bind(null, identity);
```

Sí, `.join`. Esto ya empieza a parecerse a una precuela. Vamos a abrir nuestros ojos nuevamente y volvamos a nuestro problema con `.map`.

```js
Result.join(Result.map(safe_parse, get_config()));
// { "type": "Ok", "data": { "dark-mode": true } }
```

Resolvimos nuestro problema. Aquí viene lo gracioso, en teoría podríamos implementar `.chain` usando `.join` y `.map`. Verán, usar `.join` y `.map` en conjunto es un patrón tan común que por eso existe `.chain` (también es la razón por la que algunos lo llaman `flatMap` en lugar de `chain`).

```js
Result.chain(safe_parse, get_config());
// { "type": "Ok", "data": { "dark-mode": true } }
```

¿No es genial cuando todo queda en un bonito ciclo? Pero no se levanten de sus asientos todavía, nos queda la escena post-créditos.

* Identidad - lado derecho

Se veía venir. Bien, ¿Qué dice esta ley?

```js
Val.chain(M.of);
// es equivalente a
Val;
```

Sabemos que podemos cumplirla pero sólo por si acaso, vamos a verificar.

```js
const Val = Result.Ok('hello');

const Id = Result.chain(Result.of, Val);

Result.unwrap(Val) === Result.unwrap(Id);
// true
```

¿Qué podemos hacer con esto? Bueno, lo único que se me ocurre por ahora es hacer una implementación más genérica de `.map`.

```js
Result.map = function(fn, data) {
  return Result.chain(v => Result.of(fn(v)), data);
};
```

Puede que no parezca muy útil en nuestra estructura porque `.of` y `Ok` tienen la misma funcionalidad, pero si nuestro constructor y `.of` tuvieran diferente implementación (como en el caso de la clase `Promise`) esta puede ser una buena manera de simplificar la implementación de `.map`.

Y con esto completamos el ciclo y terminamos nuestro viaje por Fantasy Land.

## Conclusión

Si leyeron todo esto y aún así no pudieron entender todo, no se preocupen, puede ser porque no me expliqué bien. A mí me tomó cerca de dos años acumular el conocimiento necesario para escribir esto. Incluso si les toma un mes entenderlo, van por mejor camino que yo.

Un buen ejercicio que pueden hacer para entender mejor es tratar de cumplir con la especificación usando clases. Debería ser más fácil de esa manera.

Espero que hayan disfrutado la lectura y no les haya dado dolor de cabeza. Hasta la próxima.

## Fuentes
- [Fantasy Land](https://github.com/fantasyland/fantasy-land)
- [Fantas, Eel, and Specification](http://www.tomharding.me/fantasy-land/)
- [Algebraic Structures Explained - Part 1 - Base Definitions](https://dev.to/macsikora/algebraic-structures-explained-part-1-base-definitions-2576)
