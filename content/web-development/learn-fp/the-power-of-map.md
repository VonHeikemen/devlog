+++
title = "Functional programming for your everyday javascript: The power of map"
description = "Let's see what makes map so special"
date = 2020-02-16
lang = "en"
[taxonomies]
tags = ["javascript", "functional-programming", "learning"]
[extra]
canonical_url = "https://dev.to/vonheikemen/functional-programming-for-your-everyday-javascript-the-power-of-map-4n9m"
shared = [
  ["dev.to", "https://dev.to/vonheikemen/functional-programming-for-your-everyday-javascript-the-power-of-map-4n9m"],
  ["Hashnode", "https://vonheikemen.hashnode.dev/the-power-of-map"]
]
+++

This time we are going to take a look inside the world of `functors` and what makes them so special. Functors is one of those terms that you hear every now and then when people talk about functional programming but when the time comes to explain it they either bombard you with more abstract terms or tell you just the details you need to know. Since I have no knowledge of category theory I'm not going to pretend that I know exactly what a functor is, I'm just going show enough for you to know how you can spot them in the wild and how you can use them to your advantage.

## Seriously, what is a functor?

I'm convinced that the term is difficult to understand because one, you need to some other terms in order to get the whole picture and two, the theory behind it doesn't really translate very well in code. But it wouldn't hurt to have at least a clue of what they are.

You can think of them as a relation that exist between two sets of values. I know it's vague, this will make sense in a second. Say we have two arrays.

```js
const favorite_numbers  = [42, 69, 73];
const increased_numbers = [43, 70, 74];
```

Nice, we have a set `favorite_numbers` and a set `increased_numbers`, they are two separate arrays in different variables but we all know that there is a connection between those two, but more importantly we can express that connection with code. Imagine that the array `increased_numbers` doesn't exist but we still need those numbers, to make them appear again all we need is our good old friend `map`.

```js
const increased_numbers = favorite_numbers.map(num => num + 1);
```

`map` will go through every number, increase it and put it in a new array which brings `increased_numbers` back into existence. Even though `increased_numbers` is something we made, we didn't create it out nowhere, we didn't magically invent `43`, `70` and `74`. All we did was describe a relation between those numbers and our `favorite_numbers`.

So, is that the whole history? Are functors just arrays? The answer to that is a big no. Arrays are just a freakishly convenient way to illustrate a common use of functors. This leaves a question in the air.

## How do you recognize them?

I often hear other people describing functors as boxes. I for one don't think they're wrong because using a container data structure is one of easiest ways to implement a functor. The box analogy is specially funny 'cause in javascript we use brackets to create arrays, so you can actually create a functor by putting a value in a box. See.

```js
// A value
1;

// A box
[];

// Look, a value in a box.
[1];

// I regret nothing.
```

Going back to the original question, how do we recognize them? Okay, so it turns out that there are rules.

### Da rules

Again I'll be using array of numbers just because is convenient but this rules must apply to any structure that wants to be in the functor club.

#### Identity

Given the `identity` function.

```js
function identity(x) {
  return x;
}
```

`value` and `value.map(identity)` must be equivalent.

For example.

```js
[1,2,3];               // => [1,2,3]
[1,2,3].map(identity); // => [1,2,3]
```

Why is this important? What does this tell us?

Valid questions. This tells us that the `map` function must preserve the shape of the data structure. In our example, if we map an array of three elements we must receive a new array of three elements. If we had an array of a hundred elements, using `.map(identity)` should return an array of a hundred elements. You get the point.

#### Composition

Given two functions `fx` and `gx` the following must be true.

`value.map(fx).map(gx)` and `value.map(arg => gx(fx(arg)))` must be equivalent.

Example time.

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

If you know how `Array.map` works this feels like 'well duh!'. This actually gives you a chance to optimize your code for readability or performance. In the case of arrays, multiple calls to `map` can have a big impact on performance when the number of elements in the list grows.

And that's it. Those two rules are all you need to know to spot a functor.

## Does it always has to be .map?

I guess by now you wish to know what other things out there follow those rules that I just mentioned, if not I'll tell you anyway. There is another popular structure that also follows the rules and that is `Promise`. Let's see.

```js
// A value
1;

// A box
Promise.resolve;

// Look, a value in a box
Promise.resolve(1);

// Identity rule
Promise.resolve(1).then(identity); // => 1 (in the future)

// Composition
Promise.resolve(1).then(add_one).then(times_two);        // => 4
Promise.resolve(1).then(num => times_two(add_one(num))); // => 4
```

To be fair, `Promise.then` behaves more like `Array.flatMap` than `Array.map` but we will ignore that.

Fine, we have `Array` and we have `Promise` both are containers of some sort and both have methods that follow the rules. But what if they didn't have those methods, what if `Array.map` didn't exist? Would that mean that `Array` is no longer a functor? Do we lose all the benefits?

Let's take a step back. If `Array.map` doesn't exists then `Array` is no longer a functor? I don't know, I'm not an FP lawyer. Do we lose all the benefits? No, we could still treat arrays as functors, we just lose the super convenient `.map` syntax. We can create our own `map` outside of the structure.

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

See? Is not that bad. And it works.

```js
// Identity rule
List.map(identity, [1]); // => [1]

// Composition
List.map(times_two, List.map(add_one, [1]));   // => [4]
List.map(num => times_two(add_one(num)), [1]); // => [4]
```

Are you thinking what I'm thinking? Probably not. This is what I'm thinking, if we can map arrays without a `.map` then nothing can stop us from doing the same thing with plain objects, because after all, objects can also hold sets of values.

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

// Why stop at `map`? 
// Based on this you can also create a `filter` and `reduce`
```

Let's see it.

```js
// Identity rule
Obj.map(identity, {some: 1, prop: 2}); // => {some: 1, prop: 2}

// Composition
Obj.map(times_two, Obj.map(add_one, {some: 1, prop: 2})); // => {some: 4, prop: 6}
Obj.map(num => times_two(add_one(num)), {some: 1, prop: 2}); // => {some: 4, prop: 6}
```

## Do It Yourself

All this talk about arrays and plain objects is useful but now I feel like we know enough to make our own functor, the rules seem to be very simple. Let's do something vaguely useful. Have you ever heard of Observables? Good, because we are going to something like that. We'll make a simpler version of [mithril-stream](https://mithril.js.org/stream.html), it'll be fun.

The goal here to handle a stream of values over time. The API of our utility will be this.

```js
// Set initial state
const num_stream = Stream(0);

// Create a dependent stream
const increased = num_stream.map(add_one);

// Get the value from a stream
num_stream(); // => 0

// Push a value to the stream
num_stream(42); // => 42

// The source stream updates
num_stream(); // => 42

// The dependent stream also updates
increased(); // => 43
```

Let's start with the getter and setter function.

```js
function Stream(state) {
  let stream = function(value) {
    // If we get an argument we update the state
    if(arguments.length > 0) {
      state = value;
    }

    // return current state
    return state;
  }

  return stream;
}
```

This should work.

```js
// Initial state
const num_stream = Stream(42);

// Get state
num_stream(); // => 42

// Update
num_stream(73);

// Check
num_stream(); // => 73
```

We know we want a `map` method but what is the effect we want? We want the callback to listen to the changes of the source stream. Let's start with the listener part, we want to store an array of listeners and execute each one right after the state changes.

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

Now we go for the `map` method, but is not going to be just any method, we need to follow the rules:

- Identity: When `map` is called it needs to preserve the shape of the structure. This means that we need to return a new stream.

- Composition: Calling `map` multiple times must be equivalent of composing the callbacks supplied to those `map`s.

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
    // Create new instance with transformed state.
    // This will execute the callback when calling `map`
    // this might not be what you want if you use a 
    // function that has side effects. Just beware.
    let target = Stream(fn(state));

    // Transform the value and update stream
    const listener = value => target(fn(value));

    // Update the source listeners
    listeners.push(listener);

    return target;
  }

  return stream;
}
```

Let's test the rules. We begin with identity.

```js
// Streams are like a cascade
// the first is the most important
// this is the one that triggers all the listeners
const num_stream = Stream(0);

// Create dependent stream
const identity_stream = num_stream.map(identity); 

// update the source
num_stream(42);

// Check
num_stream();      // => 42
identity_stream(); // => 42
```

Now let's check the composition rule.

```js
// Create source stream
const num_stream = Stream(0);

// Create dependents
const map_stream = num_stream.map(add_one).map(times_two);
const composed_stream = num_stream.map(num => times_two(add_one(num)));

// Update source
num_stream(1);

// Check
map_stream();      // => 4
composed_stream(); // => 4
```

Our job is done. But is this any useful? Can you do something with it? Well yes, you could use it in event handlers to manipulate user input. Like this.

{{ codepen(id="dyoMJRw", title="an fmap example", load_js=true) }}

### More examples

I think by now you understand really well what functors do, but if you still want to see more examples you can check out this articles. 

- [Handling the absence of a value](@/web-development/learn-fp/using-a-maybe.md)
- [Handling side effects](https://jrsinclair.com/articles/2018/how-to-deal-with-dirty-side-effects-in-your-pure-functional-javascript/)

## Conclusion
The only question that remains is "what is the benefit of using functors?"

I'll do my best here:

- This pattern allows you to focus on one problem at time. The `map` function handles how you get the data and in the callback you can focus only on processing the data.

- Reusability. This style of programming really encourage the creation of single purpose function that a lot of the times can become useful even across projects.

- Extensibility through composition. People have mixed feelings about this one, specially if we are talking about arrays. This is another thing that functors encourage, that is using chains of functions to implement a procedure.

## Sources
- [Why is map called map?](https://dev.to/techgirl1908/why-is-map-called-map-2l03)
- [Fantasy land](https://github.com/fantasyland/fantasy-land)
- [Static land](https://github.com/fantasyland/static-land)
- [funcadelic.js](https://github.com/thefrontside/funcadelic.js)
- [How to deal with dirty side effects in your pure functional JavaScript](https://jrsinclair.com/articles/2018/how-to-deal-with-dirty-side-effects-in-your-pure-functional-javascript/)
- [What’s more fantastic than fantasy land? An Introduction to Static land](https://jrsinclair.com/articles/2020/whats-more-fantastic-than-fantasy-land-static-land/)
- [Your easy guide to Monads, Applicatives, & Functors](https://medium.com/@lettier/your-easy-guide-to-monads-applicatives-functors-862048d61610)

