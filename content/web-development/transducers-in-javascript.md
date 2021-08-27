+++
title = "Transducers in javascript" 
description = "Digging a little bit into the world of transducers (in javascript)"
date = 2020-12-27
lang = "en"
[taxonomies]
tags = ["javascript", "functional-programming"]
[extra]
shared = [
  ["dev.to", "https://dev.to/vonheikemen/transducers-in-javascript-3gfm"],
  ["Hashnode", "https://vonheikemen.hashnode.dev/transducers-in-javascript"]
]
+++

What if I told you we can extract the essence of list operations like `map` and `filter` and apply them in other kinds of collections beyond arrays? What if I told you that I can implement `filter` only once and apply that exact same function in multiple types of collections? That is the idea behind transducers. Today we are going to learn what are they, how they work and how can we use them.

## Requirements

Before we begin there are a couple of things you need to know:

* [How Array.reduce works](@/web-development/learn-fp/reduce-how-and-when.md)
* [What is a reducer](@/web-development/the-case-for-reducers.md)

It would also help a lot if you are familiar with these concepts:

* First class functions
* Higher order functions
* Closures

If you don't know what any of that means, don't worry too much. Just know that in javascript we can treat functions like any other type of data.

Let us begin.

## What are transducers?

The word transducer has a long history. If you look for the definition you're going to find something like this:

> A transducer is a device that converts energy from one form to another. Usually a transducer converts a signal in one form of energy to a signal in another. -- [Wikipedia](https://en.wikipedia.org/wiki/Transducer)

We're definitively not talking about devices in this post. But it does come close to what we actually want. You see, transducer (in our context) will help us process data from a collection and can also potentially transform the entire collection from one data type to another.

This next definition gets closer to what we want to achieve:

> Composable algorithmic transformations

I know, it doesn't seem like it helps. So, the idea here is that we can compose operations in a declarative and efficient way, that can also be used in multiple types of data. That's it. Of course it's easier said than done.

How do we do all that?

Good question. This is going to be a trip, better start with baby steps. First, let's ask ourselves...

## Why?

I'll answer that with an example. Imagine a common scenario. Say that we have an array and we want to filter it. What do we do? Use `.filter`.

```js
const is_even = number => number % 2 === 0;
const data = [1, 2, 3];

data.filter(is_even);
// Array [ 2 ]
```

All looks good. Now we get a new requirement, we need to transform the values that pass the test. No problem, we can use `.map` for that.

```js
const is_even = number => number % 2 === 0;
const add_message = number => `The number is: ${number}`;

const data = [1, 2, 3];

data.filter(is_even).map(add_message);
// Array [ "The number is: 2" ]
```

Great. Everything is fine... until one day, because of reasons, we are forced to change `data` and make it a [Set](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Set). After we make the change we see this.

```
Uncaught TypeError: data.filter is not a function
```

How can we solve this? One way would be using a `for..of` loop.

```js
const is_even = number => number % 2 === 0;
const add_message = number => `The number is: ${number}`;

const data = new Set([1, 2, 3]);
const filtered = new Set();

for(let number of data) {
  if(is_even(number)) {
    filtered.add(add_message(number));
  }
}

filtered;
// Set [ "The number is: 2" ]
```

The good news is this would work on any data type that implements the [iterable protocol](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Iteration_protocols). The bad news is that in order to add another "operation" we need to change the code inside the `for` loop.

Wait... what's wrong with that?

Bear with me for a moment. Let's compare. Say that we have our loop.

```js
for(let number of data) {

}
```

What do we do when we want to filter? Add code inside the block.

```diff
  for(let number of data) {
+   if(is_even(number)) {
+     filtered.add(number);
+   }
  }
```

What do we do when we want to transform a value? Add code inside the block.

```diff
  for(let number of data) {
    if(is_even(number)) {
-     filtered.add(number);
+     filtered.add(add_message(number));
    }
  }
```

This is going to happen every time we want to add a feature to our loop. Ever heard of the phrase "open for extension, but closed for modification."? That's exactly what I want. Right now to extend the `for` loop I need to modify it, it's not like is a terrible idea, is just that we can find a more "elegant" way to achieve our goal.

Now let's take a look at our first version, the one that had `data` as an array. We want to filter, what do we do? Add a function.

```js
data.filter(is_even);
```

We want to transform things, what do we do? Add a function.

```diff
- data.filter(is_even);
+ data.filter(is_even).map(add_message);
```

See what I mean? I'm not going to claim this is better, let's just say it's more "expressive". In this case when we want to extend our process we compose functions.

But as we all know this is not a perfect solution. We already came across a problem: not every collection implements these methods. Another problem that may arise has to do with performance. Each method is the equivalent of a `for` loop, so it may not be the best idea to have a long chain of `filter`s and `map`s.

This is where transducers shine, with them we can build a chain of operations in a way that's efficient and declarative. They are not going to be as fast as a `for` loop, but it may be good way to improve performance when you have a long chain of functions and a collection with many, many items.

Unlike array methods transducers are not attached to a prototype, this gives us the opportunity to reuse the exact same function in multiple types of collections. We could for example implement `filter` as a transducer once and use that with arrays, `Set`s, [generators](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/function*) and other types. Sound great, right?

## How do they work?

The magic behind transducers lies in a term I mentioned in the requirements section: `reducer`. Specifically higher order `reducer`s.

"Higher order reducer". Now that's a lot. Breathe, take a moment, move on when you're ready.

For the time being you can think of transducers as functions that take a `reducer` as an argument and return another `reducer`. It turns out that (with a little bit of magic) we can combine `reducer`s using function composition. This handy little feature is the one that will allow us to make a chain of operations like the one in our example where we had `filter` and then `map`. Now, it won't look exactly the same, our transducers would compose like this.

```js
compose(filter(is_even), map(add_message));
```

Before you ask, there is nothing magical in `compose`. That is a fairly generic function. The only thing it does is passing values from one function to the next. We can implement that ourselves.

```js
function compose(...fns) {
  const apply = (arg, fn) => fn(arg);
  return (initial) => fns.reduceRight(apply, initial);
}
```

When we combine transducers using `compose` what we get in return is another transducer. But that's not the end of the story, because a transducer returns a `reducer` we need to do something with that, and what other function do you know that needs a `reducer`? Our friend `reduce`, of course. We'll be treating `reduce` like a protocol, it will give us the opportunity to process each item in the collection and also transform the collection itself.

Enough theory for now, let's do something. Let's make a `filter` transducer.

### Making a transducer

#### Step 1: Gather all the arguments

First things first, we have to create the function and gather everything we need. What do we need? A function that should return `true` or `false`, a predicate.

```js
function filter(predicate) {

}
```

That's a good start but is not enough. We know that at some point we need to compose this with another transducer. So we also need to receive a `reducer`, this will be the next "step" in the composition.

```js
function filter(predicate, next) {
  
}
```

If it's still not clear, remember in our previous example we wanted this.

```js
compose(filter(is_even), map(add_message));
```

Here is what's going to happen, `map(add_message)` is going to give us a `reducer` and that is going to be the `next` parameter in `filter`.

Some of you might be thinking that's not going to work, I only pass `is_even` to `filter`, how are we going to get `next`? Let's deal with that later.

#### Step 2: Return a reducer

In practice a `reducer` is nothing more than a binary function. Let's return that.

```js
function filter(predicate, next) {
  return function reducer(state, value) {
    // ???
  };
}
```

#### Step 3: Implement the rest

Okay, so we are (almost) done with the structure of the transducer. What comes next is the logic for our operation. And what we want to do is copy the behavior of `Array.filter`.

```js
function filter(predicate, next) {
  return function reducer(state, value) {
    if(predicate(value)) {
      return next(state, value);
    }

    return state;
  };
}
```

In here we take the predicate, we evaluate it and decide if we want to move on to the next step.

#### Step 4: Partial application

Here is where the magic comes. We know how we want to use `filter` but right now it's not going to work. `filter` needs to be smart enough to know when is going to execute our logic. When is that? When we have gathered all the arguments.

```js
function filter(predicate, next) {
  if(arguments.length === 1) {
    return (_next) => filter(predicate, _next);
  }

  return function reducer(state, value) {
    if(predicate(value)) {
      return next(state, value);
    }

    return state;
  };
}
```

This is just one way to achieve partial application. It doesn't have to be this way.

### Using a transducer

In theory we already have something useful. Now we need a `reduce` function. Fortunately the `Array` prototype has one that we can use. Let's start our test with just one transducer.

```js
const is_even = number => number % 2 === 0;

const data = [1, 2, 3];

const combine = (state, value) => (state.push(value), state);

data.reduce(filter(is_even, combine), []);
// Array [ 2 ]
```

It actually works! Now let's expand our data set. Say that now we have negative numbers in `data`, but we don't want those. Let's create another filter. Here is where composition comes into play.

```js
const is_even = number => number % 2 === 0;
const is_positive = number => number > 0;

const data = [-2, -1, 0, 1, 2, 3];

const combine = (state, value) => (state.push(value), state);

const transducer = compose(filter(is_positive), filter(is_even));

data.reduce(transducer(combine), []);
// Array [ 2 ]
```

Nice, we got the same result. Let's do something else, how about adding another operation?

```js
function map(transform, next) {
  if(arguments.length === 1) {
    return (_next) => map(transform, _next);
  }

  return function reducer(state, value) {
    return next(state, transform(value));
  };
}
```

The behavior is the same from `Array.map`. In this case we transform the value before going to the next step. Let's put that in our example.

```js
const data = [-2, -1, 0, 1, 2, 3];

const transducer = compose(
  filter(is_positive),
  filter(is_even),
  map(add_message)
);

data.reduce(transducer(combine), []);
// Array [ "The number is: 2" ]
```

This is good, very good. There is one more detail we need to address, compatibility. I did mention that transducers work on different types but in here we are using `Array.reduce`. We actually need to control the `reduce` function, so let's make our own.

Since javascript has the iterable protocol, we can use that to save ourselves some troubles. With this our transducers will be compatible with multiple types of collections.

```js
function reduce(reducer, initial, collection) {
  let state = initial;

  for(let value of collection) {
    state = reducer(state, value);
  }

  return state;
}
```

To test this let's change our example, now `data` is going to be a `Set`. For this to work we need to change the `combine` function, so that it knows how to assemble a `Set`. We also need to change the initial value for `reduce`. Everything else stays the same.

```js
const data = new Set([-2, -1, 0, 1, 2, 3]);

const combine = (state, value) => state.add(value);

const transducer = compose(
  filter(is_positive),
  filter(is_even),
  map(add_message)
);

reduce(transducer(combine), new Set(), data);
// Set [ "The number is: 2" ]
```

Do note that the result doesn't have to be a `Set`, we can convert `data` from a `Set` to an `Array` if we wanted to. Again, we just need a different combine function and a new initial value in `reduce`.

Everything is awesome but there is one more thing we can do to improve the "experience". We can create a helper function called `transduce`, which will basically take care of some details for us.

```js
function transduce(combine, initial, transducer, collection) {
  return reduce(transducer(combine), initial, collection);
}
```

It doesn't look like a big deal, I know. The benefit we get from this is greater control over the `reduce` function, now we could have multiple implementations and choose which one to use according to the type of `collection`. For now we are just going to stick with our home made `reduce`.

Taking this one step further, we could even match a data type with a "combine" function so it's easier to use.

```js
function curry(arity, fn, ...rest) {
  if (arity <= rest.length) {
    return fn(...rest);
  }

  return curry.bind(null, arity, fn, ...rest);
}

const Into = {
  array: curry(2, function(transducer, collection) {
    const combine = (state, value) => (state.push(value), state);
    return transduce(combine, [], transducer, collection);
  }),
  string: curry(2, function(transducer, collection) {
    const combine = (state, value) => state.concat(value);
    return transduce(combine, "", transducer, collection)
  }),
  set: curry(2, function(transducer, collection) {
    const combine = (state, value) => state.add(value);
    return transduce(combine, new Set(), transducer, collection);
  }),
};
```

Now we can have that smart partial application but this time that effect is handled by the `curry` function. So we can use it like this.

```js
const data = [-2, -1, 0, 1, 2, 3];

const transducer = compose(
  filter(is_positive),
  filter(is_even),
  map(add_message)
);

Into.array(transducer, data);
// Array [ "The number is: 2" ]
```

Or this.

```js
const some_process = Into.array(compose(
  filter(is_positive),
  filter(is_even),
  map(add_message)
));

some_process(data);
// Array [ "The number is: 2" ]
```

> You can check the full example [here](https://gist.github.com/VonHeikemen/a6b2b2e27ea999e87ebc30cf9c039295).

Now we possess truly reusable "operations". We didn't have to implement a `filter` for `Set` and another for arrays. In this contrived example it might not look like much, but imagine having an arsenal of operations like [RxJS](https://rxjs.dev/api) and being able to apply it to different kinds of collections. And the only thing you need to do to make it compatible is to provide a `reduce` function. The composition model also encourage us to solve our problems one function at a time.

There is one more thing you need to know.

## This isn't their final form

So far I've been showing transducers as functions that return a `reducer`, but that was just to show you the idea behind them. These things work but the problem is they are limited. There are a few things that our implementation doesn't support.

* An initialization hook: If the initial value is not provided, the transducer should have the opportunity to produce one.

* Early termination: A transducer should be able to send a "signal" to terminate the process and return the current value processed. Almost like the `break` keyword in a `for` loop.

* A completion hook: A function that runs at the end of the process, basically when there are no more values to process.

Because of this many articles that talk about transducer tell you to use a library. 

The only libraries I know that have support for transducers are these:

* [transducers-js](https://github.com/cognitect-labs/transducers-js)
* [ramda](https://ramdajs.com/docs/)

## Follow the protocol

We know what make transducers tick, now let's find out how one might implement a transducer the right way. For this we will follow the [protocol](https://github.com/cognitect-labs/transducers-js#the-transducer-protocol) established in the *transducer-js* library.

The rules say that a transducer must be an object with this shape.

```js
const transducer = {
  '@@transducer/init': function() {
    return /* ???? */;
  },
  '@@transducer/result': function(state) {
    return state;
  },
  '@@transducer/step': function(state, value) {
    // ???
  }
};
```

* **@@transducer/init**: This is where we can return an initial value, if for some reason we need one. The default behavior for this is to delegate the task to the next transducer in the composition, with a little luck someone might return something useful.

* **@@transducer/result**: This one runs when the process in completed. As with `@@transducer/init`, the default behavior that is expected is to delegate the task to the next step.

* **@@transducer/step**: This is where the core logic for the transducers lies. This is basically the `reducer` function.

We are not done yet, we also need a way to signal the end of the process and return the current value we have so far. For this, the protocol gives us a special object they call `reduced`. The idea is that when the `reduce` function "sees" this object it terminates the whole process. `reduced` should have this shape.

```js
const reduced = {
  '@@transducer/reduced': true,
  '@@transducer/value': something // the current state of the process
};
```

### A true transducer

Now it is time to apply everything we have learned so far. Let's re-implement `filter`, the right way. We can do it, it will mostly stay the same.

We begin with a function that returns an object.

```js
function filter(predicate, next) {
  return {

  };
}
```

For the `init` hook, what do we need to do? Nothing, really. Then we delegate.

```diff
  function filter(predicate, next) {
    return {
+     '@@transducer/init': function() {
+       return next['@@transducer/init']();
+     },
    };
  }
```

When the process is completed, what do we need to do? Nothing. You know the drill.

```diff
  function filter(predicate, next) {
    return {
      '@@transducer/init': function() {
        return next['@@transducer/init']();
      },
+     '@@transducer/result': function(state) {
+       return next['@@transducer/result'](state);
+     },
    };
  }
```

For the grand finale, the `reducer` itself.

```diff
  function filter(predicate, next) {
    return {
      '@@transducer/init': function() {
        return next['@@transducer/init']();
      },
      '@@transducer/result': function(state) {
        return next['@@transducer/result'](state);
      },
+     '@@transducer/step': function(state, value) {
+       if(predicate(value)) {
+         return next['@@transducer/step'](state, value);
+       }
+
+       return state;
+     },
    };
  }
```

Oops, let's not forget the secret sauce.

```diff
  function filter(predicate, next) {
+   if(arguments.length === 1) {
+     return (_next) => filter(predicate, _next);
+   }

    return {
      '@@transducer/init': function() {
        return next['@@transducer/init']();
      },
      '@@transducer/result': function(state) {
        return next['@@transducer/result'](state);
      },
      '@@transducer/step': function(state, value) {
        if(predicate(value)) {
          return next['@@transducer/step'](state, value);
        }
 
        return state;
      },
    };
  }
```

We have our transducer, now we have a problem: we don't have a `reduce` function capable of using it.

### reduce enhanced

We need to do a few tweaks to our `reduce`. 

Remember this.

```js
function reduce(reducer, initial, collection) {
  let state = initial;

  for(let value of collection) {
    state = reducer(state, value);
  }

  return state;
}
```

First, we need to use the `init` hook.

```diff
- function reduce(reducer, initial, collection) {
+ function reduce(transducer, initial, collection) {
+   if(arguments.length === 2) {
+     collection = initial;
+     initial = transducer['@@transducer/init']();
+   }
+
    let state = initial;

    for(let value of collection) {
      state = reducer(state, value);
    }

    return state;
  }
```

When the function gets two arguments the collection will be stored in `initial` and `collection` will be `undefined`, so what we do is put `initial` in `collection` and give our transducer the chance to give us an initial state.

Next, we call the `reducer` function, which is now in `@@transducer/step`.

```diff
  function reduce(transducer, initial, collection) {
    if(arguments.length === 2) {
      collection = initial;
      initial = transducer['@@transducer/init']();
    }
 
    let state = initial;

    for(let value of collection) {
-     state = reducer(state, value);
+     state = transducer['@@transducer/step'](state, value);
    }

    return state;
  }
```

Now we need to evaluate the return value of the `reducer` and see if we should stop the process.

```diff
  function reduce(transducer, initial, collection) {
    if(arguments.length === 2) {
      collection = initial;
      initial = transducer['@@transducer/init']();
    }
 
    let state = initial;

    for(let value of collection) {
      state = transducer['@@transducer/step'](state, value);
+
+     if(state != null && state['@@transducer/reduced']) {
+       state = state['@@transducer/value'];
+       break;
+     }
    }

    return state;
  }
```

Lastly we need to make sure our transducer knows the process is done.

```diff
  function reduce(transducer, initial, collection) {
    if(arguments.length === 2) {
      collection = initial;
      initial = transducer['@@transducer/init']();
    }
 
    let state = initial;

    for(let value of collection) {
      state = transducer['@@transducer/step'](state, value);
 
      if(state != null && state['@@transducer/reduced']) {
        state = state['@@transducer/value'];
        break;
      }
    }

-   return state;
+   return transducer['@@transducer/result'](state);
  }
```

But I'm not done yet. There is an extra step I will like to do. You might notice that I renamed `reducer` to `transducer`, I would like for this to keep working with "normal" `reducer`s like the ones we use with `Array.reduce`. So, we will create a transducer that just wraps an existent `reducer`.

```js
function to_transducer(reducer) {
  if(typeof reducer['@@transducer/step'] == 'function') {
    return reducer;
  }

  return {
    '@@transducer/init': function() {
      throw new Error('Method not implemented');
    },
    '@@transducer/result': function(state) {
      return state;
    },
    '@@transducer/step': function(state, value) {
      return reducer(state, value);
    }
  };
}
```

Now let's use it in `reduce`.

```diff
  function reduce(transducer, initial, collection) {
+   transducer = to_transducer(transducer);
+
    if(arguments.length === 2) {
      collection = initial;
      initial = transducer['@@transducer/init']();
    }
 
    let state = initial;

    for(let value of collection) {
      state = transducer['@@transducer/step'](state, value);
 
      if(state != null && state['@@transducer/reduced']) {
        state = state['@@transducer/value'];
        break;
      }
    }

    return transducer['@@transducer/result'](state);
  }
```

Now is time to test the result of all our hard work.

```js
const is_positive = number => number > 0;

const data = [-2, -1, 0, 1, 2, 3];
const combine = (state, value) => (state.push(value), state);

reduce(filter(is_positive, to_transducer(combine)), [], data);
// Array(3) [ 1, 2, 3 ]
```

Awesome, everything works just fine. But this is too much work. This is why we have that `transduce` helper function, but right now it's missing something, we need to add `to_transducer`.

```js
function transduce(combine, initial, transducer, collection) {
  return reduce(
    transducer(to_transducer(combine)),
    initial,
    collection
  );
}
```

Let's go again.

```js
const is_positive = number => number > 0;

const data = [-2, -1, 0, 1, 2, 3];
const combine = (state, value) => (state.push(value), state);

transduce(combine, [], filter(is_positive), data);
// Array(3) [ 1, 2, 3 ]
```

Now let's test the composition.

```js
const is_even = number => number % 2 === 0;
const is_positive = number => number > 0;

const data = [-2, -1, 0, 1, 2, 3];
const combine = (state, value) => (state.push(value), state);

const transducer = compose(filter(is_positive), filter(is_even));

transduce(combine, [], transducer, data);
// Array [ 2 ]
```

> You can check the full example [here](https://gist.github.com/VonHeikemen/a70479feeb59a26cd5f217ea752cf115)

Now we are officially done. There is nothing else to do. I think you already have enough information to make your own transducers.

## Conclusion

You made it! You reached the end of the post. I must congratulate you, specially if you understood everything in a single read, this is not an easy one. Celebrate, you deserve it.

Anyway, today we learned that transducers (in javascript) are transformations that work across multiple types of collections, as long as they provide a compatible `reduce` function. They also have some handy features like early termination (just like a `for` loop), they provide hooks that run at the beginning and end of a process, and they can compose directly just like regular functions. Lastly, in theory they also should be efficient, though they are not faster than a `for` loop. Regardless, they may not be the fastest things around but their compatibility with different types of collections and the declarative nature of the composition makes them a powerful tool.

## Sources

- [Functional-Light JavaScript | Appendix A: Transducing](https://github.com/getify/Functional-Light-JS/blob/master/manuscript/apA.md/#appendix-a-transducing)
- [Transducers: Supercharge your functional JavaScript](https://www.jeremydaly.com/transducers-supercharge-functional-javascript/)
- [Magical, Mystical JavaScript Transducers](https://jrsinclair.com/articles/2019/magical-mystical-js-transducers/)
- [Transducers: Efficient Data Processing Pipelines in JavaScript](https://medium.com/javascript-scene/transducers-efficient-data-processing-pipelines-in-javascript-7985330fe73d)
- ["Transducers" by Rich Hickey (video)](https://www.youtube.com/watch?v=6mTbuzafcII)
- [transducers-js](https://github.com/cognitect-labs/transducers-js)

