+++
title = "An introduction to Monads (in js)" 
description = "Trying to explain monads without loosing our minds"
date = 2020-09-02
lang = "en"
[taxonomies]
tags = ["javascript", "functional-programming"]
+++

Oh the infamous M word. The one we don't speak about in javascript. Well, today we are going talk about it, specifically we are going to "review" one definition I really like, the only one that doesn't make my head explode. In order to keep our sanity we are just going to explore the aspects we can model using javascript. Everyone ready? Let us begin.

Here it is. This is the easy one, I swear. Monads are...

> pointed functors that can flatten.

You said you were ready. Anyway, we can do this. Once you understand the behaviour of a functor the rest will fall into place.

## Enter Functors

From a *javascripty* point of view you can think of them as containers with a very special feature: they allow you to transform their inner value in any way you see fit without leaving said container.

Isn't that intriguing? How would that look like in code. Let's try to make the simplest functor we can think of.

### The Box

```js
function Box(data) {
  return {
    map(fn) {
      return Box(fn(data));
    }
  }
}
```

What happens in here? Well, we created a `Box` specifically designed to hold a `data` value and the only way to gain access to the value is through the `map` method. This `map` thing takes a function `fn` as an argument, applies that function to `data` and puts the result back in another `Box`. I must tell you that not all functors look like this, but in general this is the pattern they all follow. Let's use it.

```js
const xbox = Box('x');
const to_uppercase = (str) => str.toUpperCase();

xbox.map(to_uppercase).map(console.log);
// => X
// => Object { map: map() }
```

So, that `Box` seems um... useless. Yeah, that's by design but not mine, this is actually the `Identity` functor. It may not be useful in our day to day coding but for educational purposes it works like a charm.

What is the benefit of these functor things? By adding this tiny layer of abstraction we can separate an "effect" from a pure computation. To illustrate this let's take a look at one functor with an actual purpose.
 
### A familiar face

You may or may not know this already but arrays follow the pattern I have described for the `Box`. Check this out.

```js
const xbox = ['x'];
const to_uppercase = (str) => str.toUpperCase();

xbox.map(to_uppercase);
// => Array [ "X" ]
```

The array is a container, it has a `map` method which allows us to transform the value it holds inside, and the transformed value gets wrapped again in a new array.

Okay, that's fine, but what is the "effect" of an array? They give you the ability to hold multiple values inside one structure, that's what they do. `Array.map` in particular makes sure your callback function is applied to every value inside the array. It doesn't matter if you have a 100 items in your array or none at all, `.map` takes care of the logic that deals with **when** it should apply the callback function so you can focus on **what** to do with the value.

And of course you can use functors for so much more, like error handling or null checks, even async tasks can be modelled with functors. Now, I would love to keep talking about this but we have to go back to the monad definition.

## The Pointed part

So, we need our functors to be "pointed". This is a fancy way of telling us that we need a helper function that can put any value into the simplest unit of our functor. This function is known as "pure", other names include "unit" and "of".

Let's look at arrays one more time. If we put a value into the simplest unit of an array, what do we get? Yes, an array with just one item. Interestingly enough there is a built-in function for that.

```js
Array.of('No way');
// => Array [ "No way" ]

Array.of(42);
// => Array [ 42 ]

Array.of(null);
// => Array [ null ]
```

This helper function is specially useful if the normal way of creating your functor is somewhat convoluted. With this function you could just wrap any value you want and start `.map`ping right away. Well... there is more to it, but that's the main idea. Let's keep going.

## Into the Flatland

Now we are getting into the heart of the problem. Wait... what is exactly the problem?

Imagine this situation, we have a number in a `Box` and we want to use `map` to apply a function called `action`. Something like this.

```js
const number = Box(41);
const action = (number) => Box(number + 1);

const result = number.map(action);
```

Everything seems fine until you realise `action` returns another `Box`. So `result` is in fact a `Box` inside another `Box`: `Box(Box(42))`. And now in order to get to the new value you have to do this.

```js
result.map((box) => box.map((value) => {/* Do stuff */}));
```

That's bad. No one wants to work with data like that. This is where monads can help us. They are functors that have the "ability" to merge these unnecessary nested layers. In our case it can transform `Box(Box(42))` into `Box(42)`. How? With the help of a method called `join`.

This is how it looks like for our `Box`.

```diff
  function Box(data) {
    return {
      map(fn) {
        return Box(fn(data));
      },
+     join() {
+       return data;
+     }
    }
  }
```

I know what you're thinking, it doesn't look like I'm joining anything. You may even suggest that I change the name to "extract". Just hold it right there. Let's go back to our `action` example, we are going to fix it.

```js
const result = number.map(action).join();
```

Ta-da! Now we get a `Box(42)`, we can get to the value we want with just one `map`. Oh come on, you're still giving me the look? Okay, let's say I change the name to `extract`, now it's like this. 

```js
const result = number.map(action).extract();
```

Here is the problem, if I read that line alone I would expect `result` to be a "normal" value, something I can use freely. I'm going to be a little bit upset when I find I have to deal with a `Box` instead. On the other hand, if I read `join`, I know that `result` it's still a monad and I can prepare for that.

You may think "Okay I got it, but you know what? I write javascript I'm just going to ignore these functor things and I won't need monads". Totally valid, you could do that. The bad news is **arrays are functors**, so you can't escape them. The good news is **arrays are monads**, so when you get into this situation of nested structures (and you will) you can fix that easily.

So, arrays don't have a `join` method... I mean they do, but it's called `flat`. Behold.

```js
[[41], [42]].flat();
// => Array [ 41, 42 ]
```

There you go, after calling `flat` you can move on without worrying about any extra layer getting in your way. That's it, in practice that's the essence of monads and the problem they solve.

Before I go I need to cover one more thing.

## Monads In Chains

It turns out this combination of `map/join` is so common that there is actually a method that combines the features of those two. This one also has multiple names in the wild: "chain", "flatMap", "bind", ">>=" (in haskell). Arrays in particular call it `flatMap`.

```js
const split = str => str.split('/');

['some/stuff', 'another/thing'].flatMap(split);
// => Array(4) [ "some", "stuff", "another", "thing" ]
```

How cool is that? Instead having an array with two nested arrays, we have just one big array. This is so much easier to handle than a nested structure. 

But not only does it save you a few keystroke but it also encourage function composition in the same way `map` does. You could do something like this.

```js
monad.flatMap(action)
  .map(another)
  .map(cool)
  .flatMap(getItNow);
```

I'm not saying you should do this with arrays. I'm saying that if you do make your own monad, you can compose functions in this style. Just remember, if the function returns a monad you need `flatMap`, if not use `map`.

## Conclusion

We learned that monads are just functors with extra features. In other words they magical containers that... don't like to hold other containers inside? Let's try again: they are magical onions with... nevermind, they are magical, let's leave it at that.

They can be used to add an "effect" to any regular value. So we can use them for things like error handling, asynchronous operations, dealing with side effects, and a whole bunch of other things.

We also learned that you either love them or hate them and there is nothing in between.

## Sources
- [Professor Frisby's Mostly Adequate Guide to Functional Programming. Chapter 9: Monadic Onions](https://mostly-adequate.gitbooks.io/mostly-adequate-guide/content/ch09.html)
- [Funcadelic.js](https://github.com/thefrontside/funcadelic.js)
- [Fantasy Land](https://github.com/fantasyland/fantasy-land)

