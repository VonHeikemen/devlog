+++
title = "Tagged unions and Fantasy Land" 
description = "Let's use tagged unions to explore the Fantasy Land specification"
date = 2020-06-12
lang = "en"
draft = false
[taxonomies]
tags = ["javascript", "functional-programming"]
+++

Let's do something fun, let's explore one branch of the [Fantasy Land](https://github.com/fantasyland/fantasy-land) specification using tagged unions. In order to keep this as short as possible I'll mostly focus on how things work and leave out a lot of details. So, what we'll do is create a data structure and see if we can follow the rules on the specification.

## Tagged Unions

Also known as *variants*, is a data structure that can represent different states of a single type. At any given time it can only be in one of those states. Other important features include the ability to carry information about themselves as well as an extra "payload" that can hold anything.

It sounds cool until we realize we don't have those things in javascript. If we want to use them we'll have to recreate them. Fortunately for us we don't need a bulletproof implementation. We just need to deal with a couple of things, the type of the variant and the payload they should carry. We can handle that. 

```js
function Union(types) {
  const target = {};
  
  for(const type of types) {
    target[type] = (data) => ({ type, data });
  }

  return target;
}
```

What do we have here? You can think of `Union` as a factory of constructor functions. It takes a list of variants and for each it will create a constructor. It looks better in an example. Let's say we want to model the states of a task, using `Union` we could create this.

```js
const Status = Union(['Success', 'Failed', 'Pending']);
```

Now we can create our `Status` variants.

```js
Status.Success({ some: 'stuff' });
// { "type": "Success", "data": { "some": "stuff" } }
```

As you can see here we have a function that returns a plain object. In this object we have a `type` key where we store the name of our variant. The `data` key will hold anything we can think of. You might think that storing just the name of the variant isn't enough, because it may cause collisions with other variants of different types and you would be right. Since we are only going to create one data type this is not an issue for us.

If you find this pattern useful and want to use it, you'll need something reliable, consider using a library like [tagmeme](https://www.npmjs.com/package/tagmeme) or [daggy](https://www.npmjs.com/package/daggy) or something else.

## Fantasy Land

The github description says the following.

> Specification for interoperability of common algebraic structures in JavaScript

Algebraic structures? What? I know. The wikipedia definition for that doesn't help much either. Best I can offer is a vague sentence that leaves you with the least amount of questions, here I go: A set of values that have some operations associated with them that follow certain rules.

In our case, you can think the variants as our "set of values" and the functions that we create will be the "operations," as far as the rules goes we follow the Fantasy Land specification.

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

Once again we take advantage of the fact that `type` is a `String` and use it to "choose" the pattern that we want. This time around our patterns are inside an object. Now, each "pattern" will be associated with a method on the `patterns` object and our function `match` will return what the chosen pattern returns. If it can't find the pattern it will try to call a method with the name `_`, this will mimic the `default` keyword on the `switch/case` and if that fails it just returns `null`. With this we can have the behavior we want. 

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


## The Data Structure

This is the part where we create the thing we're going to work with. We are going to model a fairly popular concept, an action that might fail. To do this we'll create a union with two variants `Ok` and `Err`, we will call it `Result`. The idea is simple, `Ok` will represent a success and we'll use it to carry the "expected" value, all our operations will be based on this variant. On the other hand if we get a variant of type `Err` all we want to do is propagate the error, this means that we'll ignore any kind of transformation on this variant.

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

The `chain` operation lets us interact with the value that's inside our structure and apply a transformation. Sounds easy, right? We do that all the time, but this time we have rules. I present to you the first law of the day.

* Associativity

```js
Val.chain(Fx).chain(Gx);
// is equivalent to
Val.chain(v => Fx(v).chain(Gx));
```

> Notice that the comment says "equivalent to" although in most cases it should have identical results, this doesn't necessarily means that it should be read like equality, it is more like "it has the same effect as."

This law is about the order of the operations. In the first statement notice that it reads like a sequence, it goes one after the other. In the second statement it's like one operation wraps around the other. And this part is interesting, `Fx(value).chain(Gx)`. That second `chain` comes directly from `Fx`. We can tell that `Fx` and `Gx` are functions that return a data type that also follows this law.

Let's see this in practice with another data type everyone is familiar with, arrays. It turns out that arrays follow this law (sorta). I know there is no `chain` in the `Array` prototype but there is a `flatMap` which behaves just like it.

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

Thanks to the power of convenience `Result.match` has all the logic we need, the only thing we need to do is provide a value for the `err` parameter and just like that we achieve the effect we want. So `Result.chain` is a function that expects the `ok` and the `data` parameters. If the variant is of type `err` the error will just be wrapped again in a variant of the same type, like nothing happened. If the variant is of type `Ok` it will call the function we pass in the first argument. 

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

Since our function follows law we now have a way to compose functions that return other values of the same type. This is specially useful when creating a function composition where arguments of a function are the result of a previous function call.

`Result.chain` can also be use to create other utility functions. Let's start by creating one that allows us to "extract" a value from the wrapper structure.

```js
const identity = (arg) => arg;

Result.join = Result.chain.bind(null, identity);
```

So with this we get `Result.join` a function that only waits for the `data` parameter (this is the power of [partial application](@/web-development/learn-fp/partial-application.md)). Let's see it in action.

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

It might not look interesting but it is. Pay attention to that function on the first statement, `v => v`, you know this one, right? We've used it before, it is known as the `identity` function. So, in math an identity element is a neutral value that has no effect on the result of the operation and this is exactly what this function does (nothing). But the interesting part is not on the surface, is what we can't see. If the first statement has the same effect as the second, then it means that `.map(v => v)` returns another value of the same type, it does that even if we give it the most useless function we could possibly imagine. Let's show this again using arrays as an example.


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

That's nice but how does that help us? The important part to understand here is that `.map` should "preserve the shape" of our structure. In this case with arrays, if we call it with an array with one item we get back another array with one item, if we call it with an array with one hundred items we get back another array with one hundred items. Knowing that the result will always have the same type allows us to do stuff like this.

```js
Val.map(fx).map(gx).map(hx);
```

I know what you're thinking, using `.map` like that way with arrays can have a big impact on performance. Shall not be worried, the second law has that covered.

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

So `.map` gave us the ability to combine those functions in different ways, this gives us the opportunity to optimize for speed or readability. Function composition is a very complex subject and I would like to say more but we don't have time for that right now. If you're curious about it you can read this: [composition techniques](@/web-development/learn-fp/composition-techniques.es.md). 

Now is the time to implement the famous `.map` in our structure. You might have notice that this method is very similar to `.chain`, it has almost the same behavior except for one thing, with `.map` we should the guarantee that the result should be a value of the same type.

```js
Result.map = function(fn, data) { 
  return Result.chain(v => Result.Ok(fn(v)), data)
};
```

If you remember the behavior of `.chain` it only executes the callback function if `data` is a variant of type `Ok`, so the only thing we need to do to keep our structure is wrap the result from  `fn` with `Result.Ok`. 

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

The name of this law and that second statement suggest this is about function composition. I'm thinking that this should behave just like `.map` but with a plot twist, the callback we get it's trapped inside a Functor. With this we have enough information to make our method. 

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

In here `.map` has two jobs, it give us access to the function inside `res` and helps us preserve the shape of our structure. So, `.chain` will return anything that `.map` gives it, with this in place we can now have the confidence to call `.ap` multiple times. 

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

`.ap` can be used to create a helper function called `liftA2`, the goal of this function is to make another function work with values that are wrapped in a structure. Something like this.

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

This is basically the pattern that `liftA2` follows, the only difference is that instead of taking functions to a value, we take values to a function. You'll see.

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

You might have notice that everything we have built is some sort of extension of the previous methods, this will not be the exception. In order for our data structure to be an applicative it must first implement the Apply specification and then it must add one tiny detail. 

The new contribution will be a method that can help us take a value and convert it into the simplest unit of our data structure. It's kinda like a constructor method in a class, the idea is to take any regular value and take to the "context" of our structure so we can start making any kind of operation.

You've probably used something like this before. With the `Promise` class we can do this.

```js
Promise.resolve('hello').then(to_uppercase).then(console.log);
// Promise { <state>: "pending" }
// HELLO
```

After we call `Promise.resolve` our `'hello'` is "inside" a promise and we can immediately call methods like `then` or `catch`. If we wanted to do the same using the constructor we would have to do this.

```js
(new Promise((resolve, reject) => { resolve('hello'); }))
  .then(to_uppercase)
  .then(console.log);
// Promise { <state>: "pending" }
// HELLO
```

All that extra effort doesn't look very clean, right? This is why a "shortcut" is useful, we can make a "simple" unit of our data structure without extra steps. It's time to make this for `Result`.

```js
Result.of = Result.Ok;
```

I can assure you that is a coincidence, it's not always this easy. But really this is everything we need and we can prove that if we check the laws.

* Identity

```js
Val.ap(M.of(v => v));
// is equivalent to
Val;
```

Our old friend "identity" comes back to remind us that `.ap` really does behave like `.map`.

```js
const Val = Result.Ok('hello');

const Id = Result.ap(Result.of(identity), Val);

Result.unwrap(Val) === Result.unwrap(Id);
// true
```

* Homomorphism

```js
M.of(val).ap(M.of(fx));
// is equivalent to
M.of(fx(val));
```

Okay, so here we have a new concept we should learn. As far as I can tell an Homomorphism is some kind of transformation where we keep some of the "abilities" of the original value. I think this law tells us that `.of` doesn't have any effect when you "apply" a function to a value.

```js
const value = 'hello';

const one = Result.ap(Result.of(exclaim), Result.of(value));
const two = Result.of(exclaim(value));

Result.unwrap(one) === Result.unwrap(two);
// true
```

To recap, in the first statement we apply `exclaim` to `value` while both of them are wrapped inside a variant. In the second statement we apply `exclaim` to `value` directly. In both cases we get the same result. With this we prove that there is nothing special about `.of`, it's just there to create a unit of our data structure.

* Interchange

```js
M.of(y).ap(U);
// is equivalent to
U.ap(M.of(fx => fx(y)));
```

This is a tough one. Honestly, I'm not sure I understand what's trying to prove here. If I had to guess I would say that it doesn't matter which side of `.ap` we have the `.of` method, if can treat its content as a constant the result will be the same.

```js
const value   = 'hello';
const Exclaim = Result.Ok(exclaim);

const one = Result.ap(Exclaim, Result.of(value));
const two = Result.ap(Result.of(fn => fn(value)), Exclaim);

Result.unwrap(one) === Result.unwrap(two);
// true
```

### Monad

To create a Monad we must implement the Applicative and Chain specifications. So, what we have to do now is... nothing. Really, there is nothing left to do. You have created a Monad, congrats! Want to read some laws?

* Identity - left side

```js
M.of(a).chain(f);
// is equivalent to
f(a);
```

We check.

```js
const one = Result.chain(exclaim, Result.of('hello'));
const two = exclaim('hello');

one === two;
// true
```

At this point you may be wondering, couldn't we have done this after implementing `.chain` (since` .of` is an alias for `Ok`)? The answer is yes, but that wouldn't be fun.

So, what problems does this solve? What do we gain? This solves a very specific problem, one that could happen very often if we use Functors and that is nested structures.

Say we want to retrieve a `config` object that we have in `localStorage`. We know this action can fail that's why we created a function that uses our `Result` variant. 

```js
function get_config() {
  const config = localStorage.getItem('config');

  return config 
    ? Result.Ok(config)
    : Result.Err({ message: 'Config not found' });
}
```

This works wonders. Now the problem is `localStorage.getItem` doesn't return an object, the data we want is in a `String`.

```js
'{"dark-mode":true}'
```

We anticipated this so we created a function that can transform this into an object.

```js
function safe_parse(data) {
  try {
    return Result.Ok(JSON.parse(data));
  } catch(e) {
    return Result.Err(e);
  }
}
```

We know that `JSON.parse` can also fail, that's why we figure we could wrap it in a "safe function" that also uses our variant. Now try to use those two together using `.map`.

```js
Result.map(safe_parse, get_config());
// { "type": "Ok", "data": { "type": "Ok", "data": { "dark-mode": true } } }
```

Is that what you expected? If we close our eyes and pretend that `get_config` is always successful we could replace it with this.

```js
Result.of('{"dark-mode":true}');
// { "type": "Ok", "data": "{\"dark-mode\":true}" }
```

This law tells me that if I use `.chain` to apply the function to a structure it's the same as applying that function to the data inside the structure. Let's use that, we have the perfect function for this situation. 

```js
const one = Result.chain(identity, Result.of('{"dark-mode":true}'));
const two = identity('{"dark-mode":true}');

one === two;
// true
```

I hope by now you know what I'm going to do. You've seen it before.

```js
Result.join = Result.chain.bind(null, identity);
```

Yes, it's `.join`. This is starting to look like a prequel. Let's open our eyes now and go back to our problem with `.map`.

```js
Result.join(Result.map(safe_parse, get_config()));
// { "type": "Ok", "data": { "dark-mode": true } }
```

We solved our problem. Now here comes the funny thing, in theory we could implement `.chain` using `.join` and `.map`. Using `.join` and `.map` together is so common that `.chain` was created (also, that's why some people call it `.flatMap`). Let's use it.

```js
Result.chain(safe_parse, get_config());
// { "type": "Ok", "data": { "dark-mode": true } }
```

Isn't it great when everything is wrap in a nice cycle? But don't get up your seats just yet, we still have a post-credit scene.

* Identity - right side

So predictable. Alright, what does it say?

```js
Val.chain(M.of);
// is equivalent to
Val;
```

We know we can do this but let's check anyway.

```js
const Val = Result.Ok('hello');

const Id = Result.chain(Result.of, Val);

Result.unwrap(Val) === Result.unwrap(Id);
// true
```

Nice, what can we do with this? Well, the only thing I can think of right now is making a more generic version of `.map`.

```js
Result.map = function(fn, data) {
  return Result.chain(v => Result.of(fn(v)), data);
};
```

It may not look like much because `.of` and `Ok` are the same thing, but if our constructor was a bit more complex (like `Promise`) this could be a nice way to simplify the implementation of `.map`.

And with this we close the cycle and end our journey through Fantasy Land.

## Conclusion

If you read all of this but couldn't understand all of it, don't worry you can blame me, maybe I didn't explain as well as I thought. It took me like two year to gather the knowledge to write this. Even if it takes you like a month to get it, you are already doing better than me.

A nice way to try understand how this methods work is to follow the specification using regular class instances, that should be easier.

I hope you enjoyed the reading and I hope I didn't cause you a headache. Until next time.

## Sources
- [Fantasy Land](https://github.com/fantasyland/fantasy-land)
- [Fantas, Eel, and Specification](http://www.tomharding.me/fantasy-land/)
- [Algebraic Structures Explained - Part 1 - Base Definitions](https://dev.to/macsikora/algebraic-structures-explained-part-1-base-definitions-2576)
