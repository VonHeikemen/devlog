+++
title = "Observables hechos en casa"
description = "Creando un observable como los de RxJS desde cero"
date = 2019-11-24
lang = "es"
[taxonomies]
tags = ["javascript", "paradigma-reactivo", "aprendizaje"]
[extra]
canonical_url = "https://dev.to/vonheikemen/observables-hechos-en-casa-1bch"
+++

En esta ocasión implementaremos nuestros propios observables. Al terminar espero que tengan un mejor entendimiento de cómo este patrón es usado en librerías como RxJS.

## Los Observables

### ¿Qué son?

Empecemos con **mi** definición de observable.

> Un Observable es una función que sigue una convención y es usada para conectar una fuente que emite datos a un consumidor.

En nuestro caso la fuente puede ser cualquier cosa produzca valores. Y, un consumidor es el que recibe datos.

### Datos curiosos

#### Los Observables no hacen nada por sí solos

Quiero decir que estos no producen ningún efecto o comienzan a trabajar hasta que es absolutamente necesario. No hacen nada hasta que te suscribes a ellos.

#### Pueden emitir datos

Dependendiendo de la fuente, pueden recibir un número infinito de valores.

#### Pueden ser síncronos o asíncronos

Todo dependerá de su implementación. Se puede crear un observable que reciba valores de un evento que puede ocurrir en cualquier momento, también se pueden crear para procesar una colección de datos de manera síncrona.

### Algunas reglas

Ya mencioné que se debe seguir una convención. Bueno, ahora vamos a definir algunas reglas arbitrarias que nuestra implementación va a seguir. Estas son importantes porque crearemos un pequeño ecosistema con nuestros observables.

1. Deberán tener un método `subscribe`.
2. Nuestro "constructor" de observables aceptará un parámetro, será el suscriptor (`subscriber`) el cual será una función.
3. El suscriptor aceptará un parámetro, este será un objeto que llamaremos `observer`.
4. El objeto `observer` puede implementar los siguientes métodos: `next`, `error` y `complete`.

Empecemos.

### El código

### Constructor

```javascript
function Observable(subscriber) {
  return {
    subscribe: observer => subscriber(observer)
  };
}

// Se los juro, esto funciona
```

Tal vez es menos mágico de lo que pensaron. Lo que vemos aquí es que el constructor **Observable** sólo es una forma de posponer el trabajo hasta que se ejecuta `subscribe`. La función `subscriber` es la que hace el trabajo pesado, eso es bueno porque podemos hacer lo que sea ahí, es lo que hace que nuestros observable sean útiles.

Hasta ahora no he explicado el rol de `observer` o `subscriber`. Es mejor explicarlo con un ejemplo.

## Un ejemplo

Digamos que queremos convertir un arreglo en un Observable. ¿Cómo lo hacemos?

Pensemos en lo que sabemos hasta ahora:

* Podemos colocar la lógica dentro de la función `subscriber`.
* Podemos contar con que nuestro objeto `observer` tendrá uno de estos tres métodos `next`, `error` y `complete`

Podemos usar los métodos de `observer` como canales de cómunicación. La función `next` recibirá los valores que nos de la fuente. Utilizaremos `error` cuando algo salga mal, algo así como el método `catch` que tienen las promesas. Por último, utilizaremos `complete` cuando la fuente deje de producir valores.

La función para convertir un arreglo a observable puede ser así.

```javascript
function fromArray(arr) {
  return Observable(function(observer) {
    try {
      arr.forEach(value => observer.next(value));
      observer.complete();
    } catch (e) {
      observer.error(e);
    }
  });
}

// Así la usamos

var arrayStream = fromArray([1, 2, 3, 4]);

arrayStream.subscribe({
  next: value => console.log(value),
  error: err => console.error(err),
  complete: () => console.info('Listo')
});

// Y ahora a ver qué pasa en la cónsola.
```

### Tengan cuidado

Justo ahora nuestros observables básicamente son como un pueblo sin ley, podemos hacer todo tipo de cosas indebidas como seguir enviando valores después de llamar el método `complete`. En un mundo ideal nuestros observables deberían darnos algunas garantías. 

* Los métodos del objeto `observer` deberían ser opcionales.
* Los métodos `complete` y `error` deberían llamar una función para dejar de observar, una función `unsubscribe` (si esta existe).
* Si ejecutas `unsubscribe` yo no podrás ejecutar los demás métodos.
* Si se ejecuta `complete` o `error` se dejarán de recibir valores.

### Un ejemplo interactivo

Ya podemos empezar a hacer cosas interesantes con lo que tenemos hasta ahora. En este ejemplo hice una función que nos permite crear un observable de un evento. 

{{ codepen(id="wxNYPV", title="Observables - basics", load_js=true) }}

## Composición

Ahora que sabemos cómo crearlos veamos cómo podemos manipularlos para extender sus capacidades.

Esta vez lo que haremos será crear funciones complementarias y modificar nuestra implementación.

### Todo está en los operadores

Los operadores son funciones que nos permitirán agregar características a nuestros observables mediante una cadena de funciones. Cada una de estas funciones aceptará un observable como parámetro, lo convertirá en su fuente y devolverá un nuevo observable.

Sigamos con la temática del arreglo y hagamos un operador **map** que intente imitar el comportamiento del método nativo map que tienen los arreglos. Nuestro operador hará lo siguiente: tomará un valor, aplicará una función sobre ese valor y emitirá el resultado.

Hagamos el intento:

Primer paso, vamos a recibir la función y la fuente de datos, luego devolveremos un observable.

```javascript
function map(transformFn, source$) {
  return Observable(function(observer) {
    // continuará
  });
}
```

Ahora viene lo interesante, la fuente que recibimos es un observable y eso significa que podemos suscribirnos para recibir valores.

```diff
 function map(transformFn, source$) {
   return Observable(function(observer) {
+    return source$.subscribe(function(value) {
+      // continuará
+    });
   });
 }
```

Lo siguiente será pasar el resultado de la transformación a `observer` para que puedan "verlo" cuando se suscriban a este nuevo observable.

```diff
 function map(transformFn, source$) {
   return Observable(function(observer) {
     return source$.subscribe(function(value) {
+      var newValue = transformFn(value);
+      observer.next(newValue);
     });
   });
 }
```

Hay otra forma de hacer esto. Si usamos funciones de una expresión (Arrow functions como se les conoce por ahí) sería algo así.

```javascript
function map(transformFn, source$) {
  return Observable(observer => 
    source$.subscribe(value => observer.next(
      transformFn(value)
    ))
  );
}
```

Ya podemos empezar a usarlo pero justo ahora tendríamos que hacerlo de esta manera.

```javascript
function fromArray(arr) {
  return Observable(function(observer) {
    arr.forEach(value => observer.next(value));
    observer.complete();
  });
}

var thisArray = [1, 2, 3, 4];
var plusOne   = num => num + 1;
var array$    = map(plusOne, fromArray(thisArray));

array$.subscribe(value => console.log(value));
```

Eso no es muy cómodo. Y si queremos seguir usando más funciones `map` tendríamos que "envolverlas", no me parece bien. Nos ocuparemos de eso ahora.

### La cadena
 
Crearemos otro método que nos permitirá usar una cadena de operadores que extenderan un observable fuente. Esta función tomará una lista de funciones, cada función en la lista usará el observable retornado por la anterior.

Primero veamos como podría hacerse esto en una función aislada.

```javascript
function pipe(aFunctionArray, initialSource) {
  var reducerFn = function(source, fn) {
    var result = fn(source);
    return result;
  };

  var finalResult = aFunctionArray.reduce(reducerFn, initialSource);

  return finalResult;
}
```

Aquí usamos `reduce` para recorrer el arreglo de funciones y por cada elemento se ejecuta `reducerFn`. Dentro de `reducerFn` en el primer recorrido `source` tendrá el valor de `initialSource` y en el resto `source` será lo que `reducerFn` retorne. `finalResult` simplemente es el último resultado de `reducerFn`.

Con algunos ajustes a nuestro constructor podemos agregar esta función. También he reducido la implementación del método `pipe` con algo de ayuda.

```javascript
function Observable (subscriber) {
  var observable = {
    subscribe: observer => subscriber(SafeObserver(observer)),
    pipe: function (...fns) {
      return fns.reduce((source, fn) => fn(source), observable);
    }
  }

  return observable; 
}
```

Aún tenemos que hacer una cosa para asegurarnos que los operadores sean compatibles con el método `pipe`. Justo ahora el operador `map` espera tanto `transformFn` como `source`, eso no funcionará cuando usemos `pipe`. Tendremos que dividirlo en dos funciones, una que reciba el parámetro inicial y otra que acepte la fuente. 

Tenemos opciones.

```javascript
// Opción 1
function map(transformFn) {
  // En lugar de devolver el observable
  // regresamos una función que espera `source`
  return source$ => Observable(observer => 
    source$.subscribe(value => observer.next(
      transformFn(value)
    ))
  );
}

// Opción 2
function map(transformFn, source$) {
  if(source$ === undefined) {
    // en caso de no recibir `source` 
    // devolvemos una función una que recuerde `transformFn` 
    // y que espere `source`    
    return placeholder => map(transformFn, placeholder);
  }

  return Observable(observer => 
    source$.subscribe(value => observer.next(
      transformFn(value)
    ))
  );
}
```
Y ya finalmente podemos extender nuestros observables así.

```javascript
var thisArray = [1, 2, 3, 4];
var plusOne   = num => num + 1;
var timesTwo  = num => num * 2;

var array$ = fromArray(thisArray).pipe(
  map(plusOne),
  map(timesTwo),
  map(num => `number: ${num}`),
  // y otros...
);

array$.subscribe(value => console.log(value));
```

Estamos listos para crear más operadores.

## Otro ejercicio

Digamos que tenemos una función que muestra la hora en la cónsola cada segundo, y se detiene después de cinco segundos (sólo porque sí).

```javascript
function startTimer() {
  var time = 0;
  var interval = setInterval(function() {
    time = time + 1;

    var minutes = Math.floor((time / 60) % 60).toString().padStart(2, '0');
    var seconds = Math.floor(time % 60).toString().padStart(2, '0');
    var timeString = minutes + ':' + seconds;

    console.log(timeString);

    if(timeString === '00:05') {
      clearInterval(interval);
    }
  }, 1000);
}
```

Ahora bien, esa función no tiene nada de malo. Digo, hace su trabajo, es predecible y todo lo que necesitas saber está a plena vista. Pero recien aprendimos algo nuevo y queremos aplicarlo. Convertiremos esto en un observable.

Primero lo primero, vamos a extraer la lógica que maneja el formateo y el cálculo del tiempo.

```javascript
function paddedNumber(num) {
  return num.toString().padStart(2, '0');
}

function readableTime(time) {
  var minutes = Math.floor((time / 60) % 60);
  var seconds = Math.floor(time % 60);
 
  return paddedNumber(minutes) + ':' + paddedNumber(seconds);
}
```

Veamos qué hacemos con el tiempo. `setInterval` es un buen candidato para convertirse una fuente, recibe un "callback" en el cual podemos producir valores y también tiene un mecanismo de "limpieza". Es un buen observable.

```javascript
function interval(delay) {
  return Observable(function(observer) {
    var counter   = 0;
    var callback  = () => observer.next(counter++);
    var _interval = setInterval(callback, delay);
    
    observer.setUnsubscribe(() => clearInterval(_interval));
    
    return observer.unsubscribe;
  });
}
```

Tenemos una forma reusable de crear y destruir un `interval`.

Puede que hayan notado que le pasamos un número a `observer`, no lo llamamos "segundos" porque `delay` puede ser cualquier número. Aquí no estamos siguiendo el tiempo, estamos contando las veces que `callback` es ejecutado. ¿Por qué? Porque queremos que nuestros constructores sean genéricos. Siempre podremos modificar su comportamiento con operadores.

Así usamos nuestro nuevo constructor.

```javascript
// fingiremos que las demás funciones están por aquí

var time$ = interval(1000).pipe(
  map(plusOne),
  map(readableTime)
);

var unsubscribe = time$.subscribe(function(timeString) {
  console.log(timeString);
  
  if(timeString === '00:05') {
    unsubscribe();
  }
});
```

Está mejor. Pero ese `if` me molesta. Como que no debería estar ahí. ¿Saben que podemos hacer? Crear otro operador, uno que cancele la suscripción después de que `interval` emita cinco valores.

```javascript

function take(total) {
  return source$ => Observable(function(observer) {
    // tendremos nuestro propio contador porque no confío
    // en los valores que emiten otros observables
    var count = 0;
    var unsubscribeSource = source$.subscribe(function(value) {
      count++;
      // pasamos cada valor a `observer`
      // la función subscribe aún recibirá cada valor original
      observer.next(value);
      
      if (count === total) {
        // indicamos que el flujo a terminado y lo "destruimos"
        observer.complete();
        unsubscribeSource();
      }
    });
  });
}
``` 

Ya tenemos un contador que se autodestruye. Finalmente.

```javascript
// las otras funciones siguen ahí

var time$ = interval(1000).pipe(
  map(plusOne),
  map(readableTime),
  take(5)
);

time$.subscribe({
  next: timeString => console.log(timeString),
  complete: () => console.info("Time's up")
});
```

## Patio de juegos

Hice un par de ejemplos en codepen para poder hacer experimentos con estas cosas. [Este de aquí](https://codepen.io/VonHeikemen/pen/OwQYxG) contiene todos el código relacionado con `Observable` y algo más.

Y este de aquí es el del ejercicio.

{{ codepen(id="VGZXZa", title="Observables - boring timer") }}

## Conclusión

Los Observables nos permiten hacer muchas cosas y con un algo de creatividad puedes convertir lo que sea en un observable. En serio, una promesa (Promise), una petición AJAX, un evento en el DOM, un arreglo... otro observable. Todo lo que se pueden imaginar puede ser una fuente de datos que pueden envolver en un observable. También nos dan habilidad de ensamblar soluciones utilizando funciones genéricas y otras más específicas.

Aún así no son la solución perfecta para todo. Tendrán que decidir si la complejidad que traen vale la pena. Como en el ejemplo del intervalo, perdimos la simplicidad de `startTimer` por la "flexibilidad" de los observables.

## Fuentes

* [Learning Observable By Building Observable](https://medium.com/@benlesh/learning-observable-by-building-observable-d5da57405d87)
* [Observables, just powerful functions?](https://medium.com/@kevinkreuzer/observables-just-powerful-functions-a033c355b22c)
* [Who’s Afraid of Observables?](https://netbasal.com/whos-afraid-of-observables-bde0dc4f48cc)
* [Understanding mergeMap and switchMap in RxJS](https://netbasal.com/understanding-mergemap-and-switchmap-in-rxjs-13cf9c57c885)
* [JavaScript — Observables Under The Hood](https://netbasal.com/javascript-observables-under-the-hood-2423f760584)
* [Github repository - zen-observable](https://github.com/zenparsing/zen-observable)
* [Understanding Observables](https://dev.to/supermanitu/understanding-observables)
