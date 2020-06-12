+++
title = "Tagged unions and Fantasy Land" 
description = "Let's use tagged unions to explore the Fantasy Land specification"
date = 2020-06-06
lang = "en"
draft = false
[taxonomies]
tags = ["javascript", "functional-programming"]
+++

Let's do something fun, let's explore one branch of the [Fantasy Land](https://github.com/fantasyland/fantasy-land) specification using tagged unions. In order to keep this as short as possible I'll mostly focus on how things work and leave out a lot of details. So, what we'll do is create a data type and see if we can follow the rules on the specification.

## Tagged Unions

Also known as *variants*, is a data type that can represent diferent states of a single value. At any given time it can only be in one of those states. Other important features include the ability to carry information about themselves as well as an extra "payload" that can hold anything.

It sounds cool until we realise we don't have those things in javascript. If we want to use them we'll have to recreate them. Fortunately for us we don't need a bulletproof implementation. We just need to deal with a couple of things, the type of the variant and the payload they should carry. We can handle that. 

```js
function Union(types) {
  const target = {};
  
  for(const type of types) {
    target[type] = (data) => ({ type, data });
  }

  return target;
}
```

What do we have here? You can think of `Union` as a factory of constructor functions. It takes a list of variants and for each one it will create a constructor. It looks better in an example. Let's say we want to model the states of a task, using `Union` we could create this.

```js
const Status = Union(['Success', 'Failed', 'Pending']);
```

Now we can create our `Status` variants.

```js
Status.Success({ some: 'stuff' });
// { "type": "Success", "data": { "some": "stuff" } }
```

As you can see here, we have a function that gives us a plain object. In this object we have a `type` key were we store name of our variant and in the `data` key we can store anything we can think of. You might think that storing just the name of the variant isn't enough, because it may cause colisions with other variants of different types and you would be right. We are only going to use the one data type so this is not an issue for us.

If you find this pattern useful and want to use it you need something reliable, consider using a library like [tagmeme](https://www.npmjs.com/package/tagmeme) or [daggy](https://www.npmjs.com/package/daggy) or something else.

## Fantasy Land

The github description says the following.

> Specification for interoperability of common algebraic structures in JavaScript

Algebraic structures? What? I know. The wikipedia definition for that doesn't help much either. Best I can offer is a vague sentence that leaves you with the least amount of questions, here I go: A set of values that have some operations associated with them that follow certain rules.

In our case, you can think the variants is our "set of values" and the functions that we will create will be the "operations," as far as the rules goes we follow the Fantasy Land specification.

## The Link

So, we know about tagged unions and we have a vague idea about this Fantasy Land thing but know the question remains, how do we connect those two? The answer is *pattern matching*. Those familiar with the term also know that we don't have that in javascript. Sadly, in this case we can only mimic certain features.

How do we start? Let's just describe what we need. We need to evaluate a variant, be able to determine which type we have and execute a block of code. We already have the `type` key which is a `String`, with that we could just use a `switch/case`.

```js
switch(status.type) {
  case 'Success':
    // Everything went well
    break;

  case 'Failed':
    // Something went wrong
    break;

  case 'Pending':
    // Waiting...
    break;

  default:
    // Should never happen
    break;
}
```

This actually gets pretty close to what we want but there is a problem, it doesn't return anything. We want to do the same this `switch/case` does but inside an expression, something that yields a result. To recreate this behavior in the way we want we'll use objects and functions.

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

Once again we take advantage of the fact that `type` is a `String` and use it to "choose" the pattern that we want, but this time our patterns are inside an object. Now, each "pattern" will be asociated with a method on the `patterns` object and our function `match` will return what the choosen pattern returns. If it can't find the pattern it will try to call a method with the name `_`, this will mimic the `default` keyword on the `switch/case` and if that fails it just returns `null`. With this we can have the behavior we want. 

```js
match(status, {
  Success: ({ some }) => `Some: ${some}`,
  Failed:  () => 'Oops something went wrong',
  Pending: () => 'Wait for it',
  _:       () => 'AAAAHHHH'
});
// "Some: stuff"
```

With this function at our disposal we can now move on.


## The Data Type

This is the part where we create the thing we're going to work with. We are going to model a fairly popular concept, an action that might fail. To do this we'll create a union with two variants `Ok` and `Err`, we will call it `Result`. The idea is simple, `Ok` will represent a success and will be use to carry the "expected" value, all our operations will be based on this variant. On the other hand if we get a variant of type `Err` all we want to do is propagate the error, this means that we'll ignore any kind of transformation on this variant.

```js
const Result = Union(['Ok', 'Err']);
```

## The Operations

Before we move on let's just do one more thing, let's create a `match` function specific to our data type.

```js
Result.match = function(err, ok, data) {
  return match(data, {Ok: ok, Err: err});
};
```

Okay, now everything is in place. So like I said before, we will focus on just one branch of the Fantasy Land specification and that will be the one that goes from `Functor` to `Monad`. For each operation we will implement a static method in our `Result` object and I'll try to explain how does it work and why is useful.

Logic dictates that we start with Functor but we are going to take another road.

### Chain

The `chain` operation lets us interact with the value that is inside our structure and apply a transformation. Sounds easy, right? We do that all the time, but this time we have rules. May I present to you the first law of the day.

* Associativity

```js
Val.chain(Fx).chain(Gx);
// is equivalent to
Val.chain(v => Fx(v).chain(Gx));
```

> Notice that the comment says "equivalent to" although in most cases it should have identical results, this doesn't necessarily means that it should be read like equality, it should read more like "it has the same effect as."

This law is about the order of the operations. In the first statement notice that it reads like a sequence, it goes one after the other. In the second statement it's like one operation wraps around the other. And this part is interesting, see this `Fx(value).chain(Gx)`? The second `chain` comes directly from `Fx`. We can tell that `Fx` and `Gx` are functions that return a data type that also follows this law.

Let's see this in practice with another data type that everyone is familiar with, arrays. It turns out that arrays follow this law (sorta). I know there is no `chain` in the `Array` prototype but there is a `flatMap` which behaves just the same.

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

So `flatMap` let us interact with the `String` inside the array and transform it using a function and it didn't mattered that the second `flatMap` was inside or outside of the first, we got the same result.

Now let's do the same with our data type. Our implementation will be a static method (just for fun), so our examples will look a tiny bit different. This is how we do it.

```js
Result.chain = Result.match.bind(null, Result.Err);
```

Thanks to the power of convenience `Result.match` has all the logic that we need, the only thing we need to do is provide a value for the `err` parameter and just like that we achieve the effect that we want. So `Result.chain` then is a function that expects the `ok` and the `data` parameters. If the variant is of type `err` the error will just be wrapped again in a variant of the same type, like nothing happened. If the variant is of type `Ok` it will call the function we pass in the first argument. 

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

Since our function follows law we know have a way to compose that return other values of the same type. This is specially useful when creating a function composition where arguments of a function are the result of the last.

`Result.chain` can also be use to create other utility functions. Let's start be creating one that allows us to "extract" a value from the wrapper structure.

```js
const identity = (arg) => arg;

Result.join = Result.chain.bind(null, identity);
```

So, we get `Result.join` a function that only waits for the `data` (this is the power of [partial application](@/web-development/learn-fp/partial-application.md)). Let's see it in action.

```js
const good_data = Result.Ok('Hello');
Result.join(good_data);
// "Hello"

const bad_data = Result.Err({ message: 'Ooh noes' });
Result.join(bad_data);
// { "type": "Err", "data": { "message": "Ooh noes" } }
```

We called `join` because we should only use it to "flatten" a nested structure. Like in this case.

```js
const nested_data = Result.Ok(Result.Ok('Hello'));

Result.join(nested_data);
// { "type": "Ok", "data": "Hello" }
```

I'm going to abuse the nature of this function in future tests, to compare the content inside our structures. To make my intentions clear I'm going to create an "alias."

```js
Result.unwrap = Result.join;
```

### Functor

If you've been reading other articles about functional programming in javascript that name may sound familiar. Even if you don't recognize it you've probably use it before. This part of the specification is the one that introduces our old friend `.map`. Let's see what makes it so special.

* Identity

```js
Val.map(v => v);
// is equivalent to
Val;
```

It might not look interesting but it is. Pay attention to that function on the first statement, `v => v`, you know this one, right? We've used it before, it is known as the `identity` function. In math an identity element is a neutral value that has no effect on the result of the operation, in this is exactly what this function does. But the interesting part is not on the surface, is what we can't see. If the first statement has the same effect as the second then that means that `.map(v => v)` returns another value of the same type, even if we give it the most useless function we could possibly imagine. Let's show this again using arrays as an example.


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

That's nice but how does that help us? The important part to understand here is that `.map` should "preserve the shape" of our structure. In this case with arrays, if we call it with an array with one item we get back another array with one item, if we call it with an array with a hundred items we get back another array with a hundred items. Knowing that the result will always have the same type allows us to do stuff like this.

```js
Val.map(fx).map(gx).map(hx);
```

I know what you're thinking. Using `.map` that way with arrays can have a big impact on performance. Shall not be worried, the second law has that covered.

* Composition

```js
Val.map(v => fx(gx(v)));
// is equivalent to
Val.map(gx).map(fx);
```

This law tells us that we can replace several calls to `.map` if we compose directly the functions we use as arguments. Let's try it.

```js
const Val = ['hello'];

const one = Val.map(v => exclaim(to_uppercase(v)));
const two = Val.map(to_uppercase).map(exclaim);

one[0] === two[0];
// true
```

So `.map` gave us the ability to combine those functions in different ways, this gives us the oportunity to optimize for speed or readability. Function composition is a very complex subject and would like to say more, but we don't have time for that right now. If you're curious about it you can read this: [composition techniques](@/web-development/learn-fp/composition-techniques.es.md). 

Now is the time to implement the famous `.map` in our structure. You might have notice that this method is very similar to `.chain`, it has almost the same behavior except for one thing, with `.map` we should the guarantee that the result should be a value of the same type.

```js
Result.map = function(fn, data) { 
  return Result.chain(v => Result.Ok(fn(v)), data)
};
```

If you remember `.chain` only executes the callback function if `data` is a variant of type `Ok`, so the only thing we need to do to keep our structure is wrap the result from  `fn` with `Result.Ok`. 

```js
const Val = Result.Ok('hello');

// Identity
const Id = Result.map(identity, Val);

Result.unwrap(Val) === Result.unwrap(Id);
// true

// Composition
const one = Result.map(v => exclaim(to_uppercase(v)), Val);
const two = Result.map(exclaim, Result.map(to_uppercase, Val));

Result.unwrap(one) === Result.unwrap(two);
// true
```

### Apply

This is a tough one, I better try to explain after I show you the law.

* Composition

```js
Val.ap(Gx.ap(Fx.map(fx => gx => v => fx(gx(v)))));
// is equivalent to
Val.ap(Gx).ap(Fx);
```

"Whaaat?"

Yes, my thoughts exactly. That first statement is the most confusing thing we've seen so far. This time it looks like `Fx` and `Gx` are not functions, they are data structures. `Gx` has an `.ap` method so it must be the same type as `Val`. And if go further we can tell that `Fx` has a `map` method, that means is a Functor. So for this to work `Val`, `Fx` and `Gx` must implement the Functor and Apply specification. The last piece of the puzzle is this `Fx.map(fx => ... fx(...))`, there are functions involve but they are inside a data structure.

The name of this law and that second statement suggest this is about function composition. So I'm thinking that this should behave just like `.map` but with a plot twist, the callback we get it's trapped inside a Functor. With this we have enough information to make our method. 

```js
Result.ap = function(res, data) {
  return Result.chain(v => Result.map(fn => fn(v), res), data);
};
```

What's going on in here? Well, let me explain. First we get the value inside `data` if everything goes well.

```js
Result.chain(v => ..., data);
```

At this point we have problem, `.chain` doesn't give us any guarantee about the result, it can return anything. But we know that `res` is a Functor, so we can use `.map` to save the day.

```js
Result.map(fn => ..., res);
```

In here `.map` has two jobs, it give us access to the function inside `res` and helps us preserve the shape of our structure. So, `.chain` will return anything that `.map` gives it, with this in place we can now have the confidence to call `.ap` multiple times and that is what creates the composition. 

The last stop in our trip is this.

```js
fn(v);
```

That is what we actually want from `.ap`. Thanks to `.map` the result of that expression gets wrapped inside another variant which in turn goes back to the outside world thanks to `.chain`. We can test it now.

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

Fine, but what is it good for? Putting a function inside a `Result.Ok` doesn't seem like common thing, why would somebody do that? All fair questions. I believe this is all confusing because `.ap` is only half of the story.

`.ap` can be used to create a helper function called `liftA2`. The goal of this function is to make another function work with values that are wrapped in a structure. Something like this.

```js
const Title = Result.Ok('Dr. ');
const Name  = Result.Ok('Acula');

const concat = (one, two) => one.concat(two);

Result.liftA2(concat, Title, Name);
// { "type": "Ok", "data": "Dr. Acula" }
```

You can think of it as the extended version of `.map`. While `.map` is meant to work with callbacks that take one argument, `liftA2` is designed to work with a function that takes two arguments. Now the question is how does it work? The answer is in this piece of code.

```js
const composition = fx => gx => arg => fx(gx(arg));
Result.ap(Result.ap(Result.map(composition, Exclaim), Uppercase), Val);
```

Let's see what happens here. It all begins with `.map`.

```js
Result.map(composition, Exclaim)
```

In this expression we extract the function inside `Exclaim` and we apply it to `composition`.

```js
fx => gx => arg => fx(gx(arg))
// becomes
gx => arg => exclaim(gx(arg))
```

That second statement gets wrapped inside an `Ok` variant which is exactly what `.ap` expects as the first argument. So, after `.map` gets evaluated we get this.

```js
Result.ap(Result.Ok(gx => arg => exclaim(gx(arg))), Uppercase);
```

And now that we have a function inside a variant `.ap` has all it needs to continue. In here we basically have more of the same, the function inside the second argument is applied to the function in the first. So we get this.

```js
Result.Ok(arg => exclaim(to_uppercase(arg)));
```

Notice the pattern now? We have yet another function inside a variant, and this is exactly what our last `.ap` gets.

```js
Result.ap(Result.Ok(arg => exclaim(to_uppercase(arg))), Val);
```

The cycle repeats again and finally we get.

```js
Result.Ok('HELLO!!');
```

This is basically the pattern that `liftA2` follows. The only difference is that instead of taking functions to a value, we take values to a function. You'll see.

```js
Result.liftA2 = function(fn, R1, R2) {
  const curried = a => b => fn(a, b);
  return Result.ap(Result.map(curried, R1), R2);
};
```

We test again.

```js
const concat = (one, two) => one.concat(two);

Result.liftA2(concat, Result.Ok('Dr. '), Result.Ok('Acula'));
// { "type": "Ok", "data": "Dr. Acula" }
```

And what if you want to make a `liftA3`? You know what to do.

```js
Result.liftA3 = function(fn, R1, R2, R3) {
  const curried = a => b => c => fn(a, b, c);
  return Result.ap(Result.ap(Result.map(curried, R1), R2), R3);
};
```

And now that is the law of composition acting in our favor. As long a s `Result.ap` follows the law we can keep adding arguments with little effort. Now just for fun let's create a `liftN` function that can take any number of arguments. This time we will need a little help.

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

That is the "automated" version of `liftA3`. Now we can use all kinds of functions.

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

<!--
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
// is equivalent to
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
// is equivalent to
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
// is equivalent to
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
// is equivalent to
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
// is equivalent to
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

-->
