+++
title = "Reduce: how and when" 
description = "Let's identify the best usecase for reduce and maybe learn something else in the way"
date = 2020-04-21
lang = "en"
[taxonomies]
tags = ["javascript", "functional-programming", "learning"]
+++

Let's talk about the elephant in the `Array` prototype, the not so loved [reduce](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/Reduce) method but we're not going to discuss whether if it's good or not, let's not do that. We'll talk about how it works internally, then we'll try to figure out under what situation it can be an effective solution.

To make sure everyone here knows how it works we're going to make our own implementation.

## How it works

`reduce` is a function that takes a list of values and transform it into something else. The key here is the word **transformation**. The "user" of our function is the one that determines what's going to happen. What does that mean? It means that apart from the array that we're going to process we need to take a callback function as a parameter. So the function signature will be this. 

```js
function reduce(arr, callback) {
  // code...
}
```

We got ourselves some values, now what? What do we do with them? Usually the `Array` methods apply the function to every element in it. Let's do that.

```js
function reduce(arr, callback) {
  for(const value of arr) {
    callback(value);
  }
}
```

It's still not what we want but we're getting there. Now for the secret ingredient, the accumulator. We will create a variable that remembers the **current state** of our transformation. Every time we apply the `callback` function to a value we save the result in the accumulator. As a bonus before we save the new state we will pass the current state to the `callback` function so our "user" doesn't have to make any effort.

```diff
  function reduce(arr, callback) {
+   let state;
    for(const value of arr) {
-     callback(value);
+     state = callback(state, value);
    }

+   return state;
  }
```

Keep those green lines in your mind at all times. No matter how complex `reduce` looks on the outside, no matter how many weird tricks you see in the wild, those three lines are the only thing that matters.

That may not be an exact replica of `Array.reduce` but it'll do for now. Let's test it.

```js
const array1 = [1, 2, 3, 4];
const callback = (state, value) => {
  if(state == null) {
    return value;
  }

  return state + value;
};

// 1 + 2 + 3 + 4
reduce(array1, callback);
// Expected output: 10
```

See that `if`? It's there because `state` doesn't have a value in the first iteration of the loop, it's something unnecessary. As authors of `reduce` we can help reduce the amount of code that `callback` needs. If we take some of the responsibility out of the `callback` we can make `reduce` a lot more flexible. What we'll do is take the first element in the array and make that our initial state.

```diff
  function reduce(arr, callback) {
-   let state;
-   for(const value of arr) {
+   let state = arr[0];
+   let rest = arr.slice(1);
+   for(const value of rest) {
      state = callback(state, value);
    }

    return state;
  }
```

Let's do it again.

```js
const array1 = [1, 2, 3, 4];
const callback = (state, value) => {
  return state + value;
};

// 1 + 2 + 3 + 4
reduce(array1, callback);
// Expected output: 10
```

If you're still having a hard time trying to figure out what's happening then let me see if I can help. If we take `callback` out of the picture this is what happens.

```js
function reduce(arr) {
  let state = arr[0];
  let rest = arr.slice(1);
  for(const value of rest) {
   state = state + value;
  }

  return state;
}
```

Remember the green lines?

```diff
  function reduce(arr) {
+   let state = arr[0];
    let rest = arr.slice(1);
    for(const value of rest) {
+    state = state + value;
    }

+   return state;
  }
```

See that? That's the only thing you need to remember. As we can see `reduce` give us the ability increase the "capacity" of a binary **operation**, to make it process a lot more values. 

## When can I use this?

So `reduce` is one of those functions that can be used in many different situations but it's not always the best solution, still there is a time and place for it and now that we know how it works we can figure out what is the best use case.

### An ideal use case

The previous example should have give you a clue. Our function is more effective when we follow a certain pattern. Let's think about the `callback` in that example. We know it needs two numbers, runs a math operation and returns a number. Basically this.

```
Number + Number -> Number
```

That's nice, but if we take a step back and think in more general terms this is what we got.

```
TypeA + TypeA -> TypeA
```

There are two values of the same type (TypeA) and an operation (the + sign) that returns another instance of the same type (TypeA). When we look at it in that way we can see a pattern that we can apply beyond math. Let's do another example with some numbers, this time we'll do a comparison.

```js
function max(number, another_one) {
  if(number > another_one) {
    return number;
  } else {
    return another_one;
  }
}
```

`max` is a function that takes two numbers, compares them and returns the largest. It's a very general function and a bit limited. Now, if we think again in abstract terms we see that pattern again.

```
TypeA + TypeA -> TypeA
```

If we want to be more specific.

```
Number + Number -> Number
```

You know what it means, we can use `reduce` to make it process a lot more than two values.

```js
const array2 = [40, 41, 42, 39, 38];

// 40 > 41 > 42 > 39 > 38
reduce(array2, max);
// Expected output: 42
```

Turns out the pattern we've been following to create the `callback` for `reduce` has a name in functional programming, this one is called a **Semigroup**. When you have two values of the same type and a way to combine them, you are in the presence of a semigroup. So, *two values* + *way of combine them* = *Semigroup*.

You can prove you have a function that follows the rules of a semigroup, all you need to do is make sure it is associative. For example with our `max` function we can do.

```js
const max_1 = max(max(40, 42), 41); // => 42
const max_2 = max(40, max(42, 41)); // => 42

max_1 === max_2
// Expected output: true
```

See? Doesn't matter which order you group your operation, it yields the same result. Now we know that it'll work if we combine it with `reduce` and an array of numbers.

Can these rules apply to a more complex data type? Of course. In javascript we already have a few types that fit the description. Think about arrays for a moment, in the array prototype we have the `concat` method that can merge two arrays into a new one.

```js
function concat(one, another) {
  return one.concat(another);
}
```

With this we have.

```
Array + Array -> Array
```

Okay, the second parameter of `concat` doesn't have to be an array but let's ignore that for a second. If we use `concat` with `reduce` we get.

```js
const array3 = [[40, 41], [42], [39, 38]];

// [40, 41] + [42] + [39, 38]
reduce(array3, concat);
// Expected output: [40, 41, 42, 39, 38]
```

Now if you wanted you could create a function that flattens one level of a multidimensional array, isn't that great? And just like with numbers we don't have to stick with just the built-in functions. If we have a helper function that works with two arrays and it's associative we can combine it with `reduce`.

Say we have a function that joins the unique items of two arrays.

```js
function union(one, another) {
  const set = new Set([...one, ...another]);
  return Array.from(set);
}
```

Good, it works with two values of the same type but let's see if it's an associative operation.

```js
const union_1 = union(union([40, 41], [40, 41, 42]), [39]);
const union_2 = union([40, 41], union([40, 41, 42], [39]));

union_1.join(',') == union_2.join(',');
// Expected output: true
```

Yes, it follows the rules, that means that we can process multiple arrays if we use it with `reduce`.

```js
const array4 = [
  ['hello'],
  ['hello', 'awesome'],
  ['world', '!'],
  ['!!', 'world']
];

reduce(array4, union);
// Expected output: [ "hello", "awesome", "world", "!", "!!" ]
```

### Some resistance

You may have notice that in all our examples the data always has the right type, this isn't always the case in the "real world". Sometimes we get in situations where the first element of the array is not a valid input for our `callback`.

Imagine we want to use `concat` yet again but this time the array we have is this one.

```js
const array5 = [40, 41, [42], [39, 38]];
```

If we try to `reduce` it.

```js
reduce(array5, concat);
```

We get this.

```
TypeError: one.concat is not a function
```

It happens because in the first iteration `one`'s value is the number `40` which doesn't have `concat` method. What do we do? It is considered a good practice to pass a fixed initial value to avoid these kind of bugs. But we have a problem, we can't pass an initial value to our `reduce`. We're going to fix that.

```diff
- function reduce(arr, callback) {
-   let state = arr[0];
-   let rest = arr.slice(1);
+ function reduce(arr, ...args) {
+   if(args.length === 1) {
+     var [callback] = args;
+     var state = arr[0];
+     var rest = arr.slice(1);
+   } else if(args.length >= 2) {
+     var [state, callback] = args;
+     var rest = arr;
+   }
    for(const value of rest) {
     state = callback(state, value);
    }

    return state;
  }
```

To fix the previous mistake what we'll do is pass `reduce` an empty array as an initial value.

```js
reduce(array5, [], concat);
// Expected output: [ 40, 41, 42, 39, 38 ]
```

The error is gone and we have the array we wanted. But notice that the empty array not only fixed the error, it didn't influence the end result of the operation. Like numbers with the arrays we have the notion of an empty element that we can use in our functions without causing a fatal error in our program.

The empty array can be seen as an **identity element**, a neutral value that when applied to a function doesn't have an effect on the end result. Guess what, this behavior also has name in functional programming, it is known as a **Monoid**. When we have a semigroup with an identity element we get a monoid. So, *semigroup* + *identity element* = *Monoid*.

We can prove that arrays behave like a monoid in our functions.

```js
// Concat
const concat_1 = concat([], ['hello']); // => ["hello"]
const concat_2 = concat(['hello'], []); // => ["hello"]

concat_1.join(',') == concat_2.join(',');
// Expected output: true

// Union
const union_3 = union([], ['hello']); // => ["hello"]
const union_4 = union(['hello'], []); // => ["hello"]

union_3.join(',') == union_4.join(',');
// Expected output: true
```

Why does it matter? Think about this: how many times you had to write an `if` statement to guard against a `null` value or `undefined`? If we can represent an "empty value" in a safe way we prevent a whole category of errors in our programs. 

Another situation where monoids come in handy is when we want to perform an "unsafe" action on a value. We can use a reference to an empty value to make this unsafe operation while keeping the other values on the array intact.

Imagine that we have pieces of information scattered over several objects and we want to merge all those pieces.

```js
const array6 = [
  {name: 'Harold'},
  {lastname: 'Cooper'},
  {state: 'wrong'}
];
```

Normally you would use the [spread syntax](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Spread_syntax) to merge all these things, but let's say we live in a world where that is not possible. Fear not, we have a nice utility function that can do it.

```js
Object.assign;
```

If you think about it `Object.assign` also follows the pattern.

```
TypeA + TypeA -> TypeA
```

We give it two objects and it gives us back yet another object. But the catch is that it mutates the one we pass in the first parameter. So if we do this.

```js
reduce(array6, Object.assign);
// Expected value: { "name": "Harold", "lastname": "Cooper", "state": "wrong" } 
```

Looks like everything is good, but it's not. If check you `array6[0]` you'll see that it was changed, you definitely don't want that. Fortunately objects in javascript also behave like a monoid so they have a valid "empty value" we can use. So the right way of using it would be this.

```js
reduce(array6, {}, Object.assign);
// Expected value: { "name": "Harold", "lastname": "Cooper", "state": "wrong" }

array6
// Expected value: [ { "name": "Harold" }, { "lastname": "Cooper" }, { "state": "wrong" } ]
```

We can say that when we work with an array of values that follow the rules of the monoids we can be certain that `reduce` will be a good choice to process that.

## Beyond arrays

If we can implement a version of `reduce` for arrays then it wouldn't be weird to think that other people have implemented something similar in other data types. Knowing how `reduce` works could be useful if you use a library that has a method like that.

For example, in [mithril-stream](https://mithril.js.org/stream.html) there is a method called `scan` that has the following signature.

```
Stream.scan(fn, accumulator, stream)
```

That `fn` variable must be a function that follows this pattern.

```
(accumulator, value) -> result | SKIP
```

Recognize that? I hope so. Those are the same requirements `reduce` has. Okay, but what does `scan` do? It executes the function `fn` when the source (`stream`) produces a new value. `fn` gets called with the current state of the accumulator and the new value on the stream, the returned value then becomes the new state of the accumulator. Does that sound familiar?

You can test `scan` with our function `union` and see how it behaves.

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

You should be able to see how the list only adds unique values.

You can see a modified version of that in this pen.

{{ codepen(id="NWGrozo", title="A different reduce", load_js=true) }}

Our knowledge of the method `reduce` (and maybe a little bit of semigroups and monoids) can help us create helper function that can be reuse in different data types. How cool is that?

## Conclusion

Even though I didn't mention the many things you can do with `reduce` now you have the tools to be able to identify the situations where this method can be applied effectively, even if you're not sure you can make the necessary tests to know if the operation you want to do has the right properties.

## Sources

- [Practical Category Theory: Monoids (video)](https://www.youtube.com/watch?v=Qnkn4612ZIQ)
- [Funcadelic.js](https://github.com/thefrontside/funcadelic.js)
- [Functional JavaScript: How to use array reduce for more than just numbers](https://jrsinclair.com/articles/2019/functional-js-do-more-with-reduce/)
- [Array.prototype.reduce (MDN)](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/Reduce)
- [Fantasy Land](https://github.com/fantasyland/fantasy-land#fantasy-land-specification)
