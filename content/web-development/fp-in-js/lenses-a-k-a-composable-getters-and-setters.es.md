+++
title = "Lenses o mejor dicho getters y setters combinables"
description = "Vamos implementar lenses desde cero"
date = 2019-11-27
lang = "es"
[taxonomies]
tags = ["javascript", "paradigma-funcional"]
[extra]
canonical_url = "https://dev.to/vonheikemen/lenses-o-mejor-dicho-getters-y-setters-combinables-kfo"
+++

Esta vez vamos a descubrir qué son lenses (lentes en inglés), cómo se ven en javascript y espero que al final de todo esto podamos crear una implementación casi adecuada.

Pero primero vamos a retroceder un poco y vamos a preguntarnos.

## ¿Qué son getter y setter?

Son funciones que deben cumplir un propósito, extraer o asignar un valor. Pero claro eso no es lo único que pueden hacer. En la mayoría de los casos (que yo he visto) se usan para observar los cambios a una variable y causar algún efecto o para colocar validaciones que impidan algún comportamiento no deseado.

En javascript pueden ser explícitos.

```javascript
function Some() {
  let thing = 'stuff';

  return {
    get_thing() {
      // puedes hacer lo que sea aquí
      return thing;
    },
    set_thing(value) {
      // igual aquí
      thing = value;
    }
  }
}

let obj = Some();
obj.get_thing(); // => 'stuff'

obj.set_thing('other stuff');

obj.get_thing(); // => 'other stuff'
```

O pueden ser implícitos.

```javascript
let some = {};

Object.defineProperty(some, 'thing', {
  get() {
    return 'thing';
  },
  set(value) {
    console.log("no pasarás");
  }
});

some.thing // => 'thing'
some.thing = 'what?';

//
// no pasarás
//

some.thing // => 'thing'
```

¿Pero qué tiene eso de malo que algunas personas sienten la necesidad de usar alternativas como lenses?

Comencemos con el segundo ejemplo. Puedo decirles a algunas personas no les gustan las cosas mágicas, el sólo hecho de tener una función que se ha estado ejecutando sin su conocimiento es suficiente para evitarlos.

El primer ejemplo es más interesante. Vamos a verlo otra vez.

```javascript
obj.get_thing(); // => 'stuff'

obj.set_thing('other stuff');

obj.get_thing(); // => 'other stuff'
```

Se ejecuta `get_thing` el resultado es `stuff`, hasta ahora todo bien. Pero aquí viene el problema, cuando lo usas otra vez y de la misma manera obtienes `other stuff`. Tienes que rastrear la última llamada a `set_thing` para saber lo que obtendrás. No tienes la capacidad de predicir el resultado de `get_thing`, no puedes estar 100% seguro sin mirar (o saber) otras partes del código.

## ¿Hay una alternativa mejor?

No diría mejor. Intentemos crear estos lenses, después pueden decidir si les gusta o no.

¿Qué necesitamos? Lenses son un concepto que se encuentra en el paradigma de la programación funcional, entonces lo primero que haremos será crear unas funciones auxiliares. Estas serán nuestra primera versión de getter y setter.

```javascript
// Getter
function prop(key) {
  return obj => obj[key];
}

// Setter
function assoc(key) {
  return (val, obj) => Object.assign({}, obj, {[key]: val});
}
```

Ahora el "constructor."

```javascript
function Lens(getter, setter) {
  return { getter, setter };
}

// Eso es todo.
```

Notarán que `Lens` no hace absolutamente nada, esto es a propósito. Desde ya pueden darse cuenta que la mayor parte del trabajo está en `getter` y `setter`. El resultado será tan eficiente como lo sean sus implementaciones de `getter` y `setter`.

Ahora, para hacer que un `lens` haga algo útil crearemos tres funciones.

`view`: Extrae un valor.

```javascript
function view(lens, obj) {
   return lens.getter(obj);
}
```

`over`: transforma un valor usando un callback.

```javascript
function over(lens, fn, obj) {
  return lens.setter(
    fn(lens.getter(obj)),
    obj
  );
}
```

`set`: reemplaza un valor

```javascript
function always(val) {
  return () => val;
}

function set(lens, val, obj) {
  // no es genial? Ya estamos reusando funciones
  return over(lens, always(val), obj);
}
```

Es momento de crear unas pruebas.

Digamos que tenemos un objeto llamado `alice`.

```javascript
const alice = {
  name: 'Alice Jones',
  address: ['22 Walnut St', 'San Francisco', 'CA'],
  pets: { dog: 'joker', cat: 'batman' }
};
```

Empecemos con algo simple, vamos a inspeccionar un valor. Tendríamos que hacer esto.

```javascript
const result = view(
  Lens(prop('name'), assoc('name')),
  alice
);

result // => "Alice Jones"
```

Veo que no están impresionados y eso está bien. Acabo de escribir un montón de cosas sólo para ver un nombre. Pero este es el asunto, todo eso son funciones aisladas. Siempre tenemos la opción de combinarlas y crear nuevas. Empecemos con `Lens(prop, assoc)`, vamos usarlo con mucha frecuencia.

```javascript
function Lprop(key) {
  return Lens(prop(key), assoc(key));
}
```

Y ahora...

```javascript
const result = view(Lprop('name'), alice);

result // => "Alice Jones"
```

Pueden incluso ir más allá y crear una función que sólo acepte el objeto que contiene los datos.

```javascript
const get_name = obj => view(Lprop('name'), obj);

// o con aplicación parcial

const get_name = view.bind(null, Lprop('name'));

// o usando una dependencia.
// view = curry(view);

const get_name = view(Lprop('name'));

// y lo mismo aplica para `set` y `over`
```

Suficiente. Volvamos a nuestras pruebas. Vamos con `over`, vamos a transformar el texto a mayúsculas.

```javascript
const upper = str => str.toUpperCase();
const uppercase_alice = over(Lprop('name'), upper, alice);

// vieron lo que hice?
get_name(uppercase_alice) // => "ALICE JONES"

// por si acaso
get_name(alice)           // => "Alice Jones"
```

Es el turno de `set`.

```javascript
const alice_smith = set(Lprop('name'), 'Alice smith', alice);

get_name(alice_smith) // => "Alice smith"

// por si acaso
get_name(alice)       // => "Alice Jones"
```

Todo muy bonito pero `name` es sólo una propiedad, ¿Qué pasa con los objetos anidados o los arreglos? Bueno, es ahí donde nuestra implementación se vuelve algo incómoda. Justo ahora tendríamos que hacer algo así.

```javascript
let dog = Lens(
  obj => prop('dog')(prop('pets')(obj)),
  obj => assoc('dog')(assoc('pets')(obj))
);

view(dog, alice); // => "joker"

// o traemos una dependencia, `compose`

dog = Lens(
  compose(prop("dog"), prop("pets")),
  compose(assoc("dog"), assoc("pets"))
);

view(dog, alice); // => "joker"
```

Los escucho. No se preocupen, no los dejaría escribir cosas así. Es por cosas como esta que algunos van y dicen "usa [Ramda](https://ramdajs.com/) y ya" (y tienen razón) ¿Pero qué hace ramda que lo hace tan especial?

## El toque especial

Si van a la documentación de ramda y buscan "lens" verán que tienen una función llamada `lensProp` que basicamente hace lo mismo que `Lprop`. Y si van al código fuente verán esto.

```javascript
function lensProp(k) {
  return lens(prop(k), assoc(k));
}
```

Miren eso. Ahora bien, los comentarios en el código y la documentación sugieren que trabaja con una sola propiedad. Volvamos a nuestra búsqueda en su documentación. Ahora prestemos atención a esa curiosa función llamada `lensPath`. Parece que hace exactamente lo que queremos. Una vez más vemos el código fuente y ¿qué vemos?

```javascript
function lensPath(p) {
  return lens(path(p), assocPath(p));
}

// Bienvenidos al paradigma funcional
``` 

El secreto está en otras funciones que no tienen ningún vinculo específico con `lenses`. ¿No es genial?

¿Qué hay en esa función `path`? Vamos a revisar. Voy a mostrarles una versión ligeramente diferente, pero el comportamiento es el mismo. 

```javascript
function path(keys, obj) {
  if (arguments.length === 1) {
    // esto es para imitar la dependencia `curry`
    // esto es lo que pasa
    // retornan una función que recuerda `keys`
    // y espera el argumento `obj`
    return path.bind(this, keys);
  }

  var result = obj;
  var idx = 0;
  while (idx < keys.length) {
    // no nos agrada null
    if (result == null) {
      return;
    }
    
    // así obtenemos los objetos anidados
    result = result[keys[idx]];
    idx += 1;
  }

  return result;
}
```

Haré lo mismo con `assocPath`. En este caso en ramda usan algunas funciones internas pero en esencia esto es lo que pasa.

```javascript
function assocPath(path, value, obj) {
  // otra vez esto
  // por eso tienen la función `curry`
  if (arguments.length === 1) {
    return assocPath.bind(this, path);
  } else if (arguments.length === 2) {
    return assocPath.bind(this, path, value);
  }

  // revisamos si está vacío
  if (path.length === 0) {
    return value;
  }

  var index = path[0];

  // Cuidado: recursividad adelante
  if (path.length > 1) {
    var is_empty =
      typeof obj !== 'object' || obj === null || !obj.hasOwnProperty(index);

    // si el objeto actual está "vacío"
    // tenemos que crear otro
    // de lo contrario usamos el valor en `index`
    var next = is_empty
      ? typeof path[1] === 'number'
        ? []
        : {}
      : obj[index];

    // empecemos otra vez
    // pero ahora con un `path` reducido
    // y `next` es el nuevo `obj`
    value = assocPath(Array.prototype.slice.call(path, 1), value, next);
  }

  // el caso base
  // o copiamos un arreglo o un objeto
  if (typeof index === 'number' && Array.isArray(obj)) {
    // 'copiamos' el arreglo
    var arr = [].concat(obj);

    arr[index] = value;
    return arr;
  } else {
    // una copia como las de antes
    var result = {};
    for (var p in obj) {
      result[p] = obj[p];
    }

    result[index] = value;
    return result;
  }
}
```

Con nuestro nuevo conocimiento podemos crear `Lpath` y mejorar `Lprop`.

```javascript
function Lpath(keys) {
  return Lens(path(keys), assocPath(keys));
}

function Lprop(key) {
  return Lens(path([key]), assocPath([key]));
}
```

Ahora podemos hacer otras cosas, como manipular la propiedad `pets` de `alice`.

```javascript
const dog_lens = Lpath(['pets', 'dog']);

view(dog_lens, alice);     // => 'joker'

let new_alice = over(dog_lens, upper, alice);
view(dog_lens, new_alice); // => 'JOKER'

new_alice = set(dog_lens, 'Joker', alice);
view(dog_lens, new_alice); // => 'Joker'
```

Todo funciona de maravilla pero hay un pequeño detalle, nuestro constructor `Lens` no produce "instancias" combinables. Imaginen que tenemos lenses en varios lugares y queremos combinarlos de la siguiente manera.

```javascript
compose(pet_lens, imaginary_lens, dragon_lens);
```

Eso no funcionaría porque `compose` espera una lista de funciones y lo que tenemos ahora son objetos. Pero podemos cambiar eso (de una forma muy curiosa) con algunos trucos propios de la programación funcional. 

Empecemos con el constructor. En lugar de devolver un objeto vamos retornar una función, una que reciba "por partes" un callback, un objeto y que devuelva un **Functor** (eso es una cosa que tiene un método `map` que sigue [estas reglas](https://github.com/fantasyland/fantasy-land#functor))

```javascript
function Lens(getter, setter) {
  return fn => obj => {
    const apply = focus => setter(focus, obj);
    const functor = fn(getter(obj));
    return functor.map(apply);
  };
}
```

¿Y eso de `fn => obj => `qué? Eso nos va ayudar con el problema que tenemos con `compose`. Después de que le proporcionas `getter` y `setter` te devuelve una función que es compatible con `compose`.

¿Y `functor.map`? Eso es para asegurarnos que podamos usar un lens como una unidad (como `Lprop('pets')`) y también como parte de una cadena usando `compose`.

En caso de que se pregunten qué diferencia hay con lo que hace ramda, ellos usan su propia implementación de la función `map`.

Ahora modificamos `view` y `over`. Empezando con `view`.

```javascript
function view(lens, obj) {
  const constant = value => ({ value, map: () => constant(value) });
  return lens(constant)(obj).value;
}
```

Esa función `constant` puede que parezca innecariamente compleja pero tiene su propósito. Las cosas se pueden enredar mucho cuando usas `compose`, esa estructura se asegura que el valor que queremos se mantenga intacto.

¿Y `over`? Es casi igual, excepto que en ese caso sí utilizamos la función `setter`.

```javascript
function over(lens, fn, obj) {
  const identity = value => ({ value, map: setter => identity(setter(value)) });
  const apply = val => identity(fn(val));
  return lens(apply)(obj).value;
}
```

Y ahora deberíamos tener una implementación casi adecuada. Esto es lo que tenemos sin contar las dependencias (`path` and `assocPath`).

```javascript
function Lens(getter, setter) {
  return fn => obj => {
    const apply = focus => setter(focus, obj);
    const functor = fn(getter(obj));
    return functor.map(apply);
  };
}

function view(lens, obj) {
  const constant = value => ({ value, map: () => constant(value) });
  return lens(constant)(obj).value;
}

function over(lens, fn, obj) {
  const identity = value => ({ value, map: setter => identity(setter(value)) });
  const apply = val => identity(fn(val));
  return lens(apply)(obj).value;
}

function set(lens, val, obj) {
  return over(lens, always(val), obj);
}

function Lprop(key) {
  return Lens(path([key]), assocPath([key]));
}

function Lpath(keys) {
  return Lens(path(keys), assocPath(keys));
}

function always(val) {
  return () => val;
}
```

¿Me creerían si les digo que funciona? No deberían. Hagamos unas pruebas. Volvamos con `alice` y vamos añadirle otro objeto, `calie`.

```javascript
const alice = {
  name: "Alice Jones",
  address: ["22 Walnut St", "San Francisco", "CA"],
  pets: { dog: "joker", cat: "batman", imaginary: { dragon: "harley" } }
};

const calie = {
  name: "calie Jones",
  address: ["22 Walnut St", "San Francisco", "CA"],
  pets: { dog: "riddler", cat: "ivy", imaginary: { dragon: "hush" } },
  friend: [alice]
};
```

Y porque teniamos todo planeado desde antes, ya tenemos unos lenses disponibles.

```javascript
// uno genérico
const head_lens = Lprop(0);

// otros específicos
const bff_lens = compose(Lprop('friend'), head_lens); 
const imaginary_lens = Lpath(['pets', 'imaginary']);
```

Supongamos que queremos manipular la propiedad `dragon` de cada una, todo lo que tenemos que hacer es combinar.

```javascript
const dragon_lens = compose(imaginary_lens, Lprop('dragon'));

// sólo porque sí
const bff_dragon_lens = compose(bff_lens, dragon_lens); 

// demo
const upper = str => str.toUpperCase();

// view
view(dragon_lens, calie);         // => "hush"
view(bff_dragon_lens, calie);     // => "harley"

// over
let new_calie = over(dragon_lens, upper, calie);
view(dragon_lens, new_calie);     // => "HUSH"

new_calie = over(bff_dragon_lens, upper, calie);
view(bff_dragon_lens, new_calie); // => "HARLEY"

// set
new_calie = set(dragon_lens, 'fluffykins', calie);
view(dragon_lens, new_calie);     // => "fluffykins"

new_calie = set(bff_dragon_lens, 'pumpkin', calie);
view(bff_dragon_lens, new_calie); // => "pumpkin"
```

Así que acabamos de manipular un objeto anidado en varios niveles combinando lenses. Resolvimos un problema combinando funciones. Si no les parece genial no sé qué más decirles. 

Estas cosas son difíciles de vender porque requieren de un estilo particular para poder aprovecharlos al máximo. Y para los que usan javascript, probablemente existe una librería que resuelve el mismo problema pero de una manera más conveniente o por lo menos que se ajuste a su estilo.

En fin, si aún están interesado en cómo funcionarían estos lenses en un contexto más complejo revisen [este repositorio](https://github.com/kwasniew/hyperapp-realworld-example-app), es un [ejemplo de "real world app"](https://github.com/gothinkster/realworld) (algo así como un clon de medium.com) usa hyperapp para manejar la interfaz. El autor quiso usar lenses para manejar el estado de la aplicación.

## Fuentes

- [ramda - docs](https://ramdajs.com/docs/)
- [fp-lenses.js](https://gist.github.com/branneman/f06bd451f74e5bc1725db23be682d4fe)
- [Lambda World 2018 - Functional Lenses in JavaScript (video)](https://www.youtube.com/watch?v=IoVaArsh6tM)
