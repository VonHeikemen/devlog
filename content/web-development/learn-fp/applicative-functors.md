+++
title = "What are these applicative functors you speak of?" 
description = "let's use javascript to learn some applicatives for a greater good"
date = 2020-08-11
lang = "en"
[taxonomies]
tags = ["javascript", "functional-programming"]
+++

What are they indeed. Our goal for today will be to learn about applicative functors using javascript. Yes, javascript. Don't judge me, it's what I know. We'll cover things like how to create them, how you can spot them in the wild and a somewhat useful use case.

Okay, let's start from the beginning.

## What is a functor?

From a "technical" point of view you can think of them as containers of some sort. You see, the simplest way to implement a functor is by wrapping a value inside a data structure, then provide a method to interact with that value. This method is usually called `map`, its only purpose is to give us access to the value so we can transform it and then put the result back into the wrapper structure.

Let's see `map` in action. To make this less scary we'll look at a data type that we use all the time in javascript, arrays.

```js
const numbers = [1];
const plus_one = (number) => number + 1;

numbers.map(plus_one);
// [ 2 ]
```

What happens here?

We have a number wrapped in an array, we use `map` to gain access to it and transform it using a callback function, and then the new value of the number gets wrapped in another array. That's it. That's basically the behaviour you want in a functor.

Now, arrays are not the only ones that have this behaviour, there is another data type that acts like this, `Promise`. In a `Promise` we don't have a `map` but we have a `then` which is close enough.

```js
const number = Promise.resolve(1);
const plus_one = (number) => number + 1;

number.then(plus_one);
// Promise { <state>: "pending" }
// 2
```

Same thing happens here, we have a value in a structure (here a `Promise`), a method gives us access through a callback (that's `then`) and the new value gets wrapped in another instance of the same structure.

And that's the pattern. We covered what we needed to know about functors for now. If you want to know more about them check out this article: [The Power of Map](@/web-development/learn-fp/the-power-of-map.md).

Ready to move on?

## Applicatives

Applicatives are just functors with extra features. They give you the ability to merge two functors together. Specifically, they allow you to apply a function inside a functor to a value that's also inside a functor.

Wait... What? A functor that has function inside?

Yes. Putting a function inside a functor, like doing this.

```js
const plus_one = (number) => number + 1;

// And then you put it in a box

[plus_one];

// Or

Promise.resolve(plus_one);
```

Why would someone do that?

Good question. The answer is, you wouldn't. I mean in the context of javascript is not a common thing to do. Doesn't mean applicatives are useless to us.

Back to our definition. Normally if you have a function and a value you would be able to apply the function using this syntax: `some_function(some_value)`. That doesn't work if both are inside another structure. To "fix" this, applicatives have a method called `ap` (short for apply) which takes care of unwrapping each functor and applying the function to the value.

At this point I would love to show an example of a built-in data type that follows the rules of applicatives but I don't know of any. But do not fear, let's take this as an opportunity to do something else.

## Building an Applicative from scratch

In order to keep this simple we are just going to make a thin wrapper around the `Promise` class. We are going to make `Promise` feel more functor-y and applicative-ish. Where do we start?

- The goal

We want to make a "lazy promise". Usually a `Promise` executes the "task" we give it immediately but we don't want that now, this time we want to control when the task gets called. To achieve our goal we are going to create a method called `fork`, this will be the one that actually builds the `Promise` and sets the callbacks for success and failure.

```js
function Task(proc) {
  return {
    fork(err, success) {
      const promise = new Promise(proc);
      return promise.then(success).catch(err);
    }
  }
}
```

Awesome. Now let's compare this we a normal `Promise`.

```js
let number = 0;
const procedure = function(resolve, reject) {
  const look_ma = () => {
    console.log(`IT WORKED ${++number} times`);
    resolve();
  };

  setTimeout(look_ma, 1000);
};

new Promise(procedure); // This one is already running

Task(procedure); // This one doesn't do anything
Task(procedure)  // This does
  .fork(
    () => console.error('AAHHH!'),
    () => console.log('AWW')
  );
```

If you run that you should get these messages after 1 second.

```
IT WORKED 1 times
IT WORKED 2 times
AWW
```

Now that we have what we want, let's go to the next step.

- Make it functor

As you know applicatives are functors, it means that now we need a `map`.

Let's go over one more time. What is the expected behaviour of `map`?

1. It should give us access to the inner value through a callback function.
2. It should return a new container of the same type. In our case it should return another `Task`.

```diff
  function Task(proc) {
    return {
+     map(fn) {
+       return Task(function(resolve, reject) {
+         const promise = new Promise(proc);
+         promise.then(fn).then(resolve).catch(reject);
+       });
+     },
      fork(err, success) {
        const promise = new Promise(proc);
        return promise.then(success).catch(err);
      }
    }
  }
```

What happens there? Well, first we receive an `fn` argument that's our callback. Then, we return a new `Task`. Inside that new `Task` we build the promise, just like in fork but this time it's "safer" because it doesn't run immediately. After that we just chain functions to the `promise` in their respective order, first the `fn` callback to transform the value, then the `resolve` function that will "end" the current task and finally the `catch` gets the `reject` function from the current task.

We can test this now.

```js
const exclaim = (str) => str + '!!';
const ohh = (value) => (console.log('OOHH'), value);

Task((resolve) => resolve('hello'))
  .map(exclaim)
  .map(ohh)
  .fork(console.error, console.log);
```

If you run it as is you should get this.

```
OOHH
hello!!
```

But if you remove the `fork` you should get this.

```
```

Yes, a whole lot of nothing. Now we are done with the functory stuff.

- Let's Apply

We are half way there now. We have our functor pattern going on, now we need to make `ap` happen.

The way I see it `ap` is just like `map` but with a plot twist: the function we want to apply it's trapped inside another `Task` [*dramatic music plays in the background*].

With that idea in our minds we can write `ap`.

```diff
  function Task(proc) {
    return {
      map(fn) {
        return Task(function(resolve, reject) {
          const promise = new Promise(proc);
          promise.then(fn).then(resolve).catch(reject);
        });
      },
+     ap(Fn) {
+       return Task(function(resolve, reject) {
+         const promise = new Promise(proc);
+         const success = fn => promise.then(fn);
+         Fn.fork(reject, success).then(resolve);
+       });
+     },
      fork(err, success) {
        const promise = new Promise(proc);
        return promise.then(success).catch(err);
      }
    }
  }
```

Spot the difference? Don't worry I'll tell you anyway, the difference is that in order to get the callback function we use the `fork` of `Fn` instead of a raw `Promise`. That's it. Now see if it works.

```js
const to_uppercase = (str) => str.toUpperCase();
const exclaim = (str) => str + '!!';

const Uppercase = Task((resolve) => resolve(to_uppercase));
const Exclaim = Task((resolve) => resolve(exclaim));
const Hello = Task((resolve) => resolve('hello'));

Hello.ap(Uppercase).ap(Exclaim)
  .fork(console.error, console.log);
```

We made it! Now we can merge values and functions inside applicatives! But we can't enter the applicative functors club just yet, we still need something more.

- The forgotten ingredient

Applicatives must be able to put any value into the most simple unit of your structure.

The `Promise` class actually has something like that. Instead of doing this.

```js
new Promise((resolve) => resolve('hello'));
```

We usually do this.

```js
Promise.resolve('hello');
```

And after we use `Promise.resolve` we can immediately start calling methods like `then` and `catch`. That's what our `Task` is missing.

For this new "feature", we will need a static method. This one has different names in the wild, some call it "pure" others call it "unit" and the lazy ones call it "of".

```js
Task.of = function(value) {
  return Task((resolve) => resolve(value));
};
```

We can finally say we have an applicative functor.

## Something you can use in your day to day coding

Been able to create your own data type is nice, but wouldn't it be better if could just apply these patterns to existing types?

I have a good news and bad news. The good news is that we totally can. The bad news is that it will be a bit awkward.

Let's keep going with the `Task` theme we got going on. Let's say that we want to use `map` and `ap` with a `Promise` but we don't want create a new data type. What do we do? Some good old functions will do.

If you know the patterns and behaviours you should be looking for, writing some static functions in an object will be enough. This what our `Task` would look like as static functions (minus the "lazy" part).

```js
const Task = {
  of(value) {
    return Promise.resolve(value);
  },
  map(fn, data) {
    return data.then(fn);
  },
  ap(Fn, data) {
    return Fn.then(fn => data.then(value => fn(value)));
  }
};
```

If you want to `map` you'll do something like this.

```js
const to_uppercase = (str) => str.toUpperCase();

Task.map(to_uppercase, Task.of('hello'))
  .then(console.log);
```

`ap` also works in the same way.

```js
const exclaim = (str) => str + '!!';

Task.ap(Task.of(exclaim), Task.of('hello'))
  .then(console.log);
```

I can feel your scepticism from here. Be patient, this will be good. Now, `map` looks kinda useful but `ap` not so much, right? Don't worry, we can still use `ap` for a greater good. What if I told you we can have like an "enhanced" version of `map`? Our `map` just works with functions that receive one argument and that's good but sometimes we need more.

Say that we have a function that needs two arguments but every time we use it those arguments come from two different promises. In our imaginary situation we have these functions. 

```js
function get_username() {
  return new Promise((resolve) => {
    const fetch_data = () => resolve('john doe'); 
    setTimeout(fetch_data, 1000);
  });
}

function get_location() {
  return new Promise((resolve) => {
    const fetch_data = () => resolve('some place'); 
    setTimeout(fetch_data, 500);
  });
}

function format_message(name, place) {
  return `name: ${name} | place: ${place}`;
}
```

When we use `format_message` its arguments almost every time come from those other functions `get_username` and `get_location`. They are asynchronous, so you might be tempted to use `Async/await` but that wouldn't be the best idea. Those two don't depend on each other, we will be wasting time if we make them run sequentially when they could be running concurrently. One solution can be found in the form of `Promise.all`, and it looks like this.

```js
Promise.all([get_username(), get_location()])
  .then(([name, place]) => format_message(name, place))
  .then(console.log);
```

There you go. That works. But we can do better because we have applicatives on our side. Besides, we already wrote that `Task` object with all those functions. Let's add one more static function to `Task` that does the same thing `Promise.all` is doing for us here.

```js
Task.liftA2 = function(fn, A1, A2) {
  const curried = a => b => fn(a, b);
  return Task.ap(Task.map(curried, A1), A2);
};
```

I'll explain the name later. Now let's see it action.

```js
Task.liftA2(format_message, get_username(), get_location())
  .then(console.log);
```

Isn't this just slightly better? 

And yes, several arguments could be made against this particular implementation of `liftA2` and the `Task` itself, but all the patterns I've shown would work just fine with most of the applicative you can find in the wild.

As a fun exercise you can try to implement `map` and `ap` for [Set](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Set)s. See what kind of funny things you discover in the process.

Anyway, about that name `liftA2`. In functional programming when you take a function and make it work with container types like functors it is said that you're "lifting" the function to the "context" of that container. What do I mean by context? Okay, in the world of arrays when you use `Array.map` your function gets applied multiple times, in the context of a `Promise` your function runs only when the `Promise` is resolved. See what I mean? Good. The `A2` part? Well, you know, it only works with binary functions so... that's why.

There is still one more trick you can do with applicatives but I still don't fully understand how it works, so maybe next time I'll show you that.

## Conclusion

What did we learn today, class?

- We learned about functors: 
  * What they do.
  * What pattern they should follow.
- We learned about applicatives:
  * What they are.
  * What they do.
  * How to make one from scratch.
  * How to make an `ap` even if the data type doesn't have a built-in method to support the applicative pattern.
  * And that `liftA2` thingy that looks kinda of cool.

Y'all learned all that? My goodness. You're the best.

Okay, I guess my job here is done.

## Sources

- [Fantasy Land](https://github.com/fantasyland/fantasy-land)
- [Static Land](https://github.com/fantasyland/static-land)
- [Fantas, Eel, and Specification 8: Apply](http://www.tomharding.me/2017/04/10/fantas-eel-and-specification-8/)
- [Fantas, Eel, and Specification 9: Applicative](http://www.tomharding.me/2017/04/17/fantas-eel-and-specification-9/)
- [Professor Frisby's Mostly Adecuate Guide to Functional Programming. Chapter 10: Applicative Functors](https://mostly-adequate.gitbooks.io/mostly-adequate-guide/ch10.html)
- [Learn you a Haskell: Functors, Applicative Functors and Monoids](http://learnyouahaskell.com/functors-applicative-functors-and-monoids#applicative-functors)
