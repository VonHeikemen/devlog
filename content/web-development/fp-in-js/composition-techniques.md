+++
title = "Functional programming for your everyday javascript: Composition techniques"
description = "An introduction to common patterns used in functional programming"
date = 2020-03-30
lang = "en"
[taxonomies]
tags = ["javascript", "functional-programming", "learning"]
+++

Today we are going to talk about function composition. The art of creating big things with "simple" pieces. It will be even better if you don't know anything about functional programming, this will be an introduction to common concepts and patterns used in that paradigm that can be implemented in javascript. What I'm about to show you is not a magical formula to make your code more readable or bug free, that's not how any of this works. I do believe that it can help solve some problems, but in order to do that in the best way you need to keep a few things in mind. So, before I show you any implementation we are going to talk about some concepts and a little bit about philosophy. 

## What you need to know

### What is function composition?

It's a mechanism that allows us to combine two or more function into a new function.

It looks like a simple idea, haven't we all at some point in our lives combined a couple of functions? But do we really think about composition when we create them? What will help us make functions already designed to be combined?

### Philosophy

Function composition is more effective if you follow certain principles.

- The function should only have one purpose, one responsibility.
- Always think the returned value will be consumed by another function.

You've probably heard this before, it's a fragment of the [unix philosophy](https://en.wikipedia.org/wiki/Unix_philosophy#Origin). Ever wondered how come `bash`, despite having a weird syntax and many limitations, is so popular? Those two principle are a big part. A lot of the software designed for that environment is specially made to be a reusable component, and when you "connect" two or more the result is another program that can be connected with other unknown programs.

For some it might seem silly or even excessive to have many little functions that do just one thing, specially if what they do looks useless, but I can prove to you that every function can be valuable in the right context.

I'll try to setup a situation where we can put in practice these principles.

> Note: I apologize in advance for the bad use of `cat` and `grep`, I'll do it just to prove a point about composition.

Say that we want to extract the value of variable named `HOST` that's inside a `.env` file. Let's try to do this in `bash`.

This is the file.

```
ENV=development
HOST=http://locahost:5000
```

To show the content of the file in the screen we use `cat`.

```sh
cat .env
```

To filter that content and search the line we want we use `grep`, provide the pattern of the thing we want and the content of the file.

```sh
cat .env | grep "HOST=.*"
```

To get the value we use `cut`, this is going to take the result provided by `grep` and it's going to divide it using a delimiter, then it will give us the section of the string we tell it.

```sh
cat .env | grep "HOST=.*" | cut --delimiter="=" --fields=2
```

That should give us.

```
http://locahost:5000
```

If we put that chain of commands in a script or a function inside our `.bashrc` we will effectively have a command that can be used in the same way by yet other commands that we don't even know about. That is the kind of flexibility and power that we want to have.

I hope by now you know what kind of things you need to consider when you create a function but there is just one more thing I would like to tell you.

### Functions are things

Let's turn around and put our attention on javascript. Have you ever heard the phrase "first-class function"? It means that functions can be treated just like any other value. Let's compare with arrays.

- You can assign them to variables

```js
const numbers = ['99', '104'];
const repeat_twice = function(str) {
  return str.repeat(2);
};
```

- Pass them around as arguments to a function

```js
function map(fn, array) {
  return array.map(fn);
}

map(repeat_twice, numbers);
```

- Return them from other functions

```js
function unary(fn) {
  return function(arg) {
    return fn(arg);
  }
}

const safer_parseint = unary(parseInt);

map(safer_parseint, numbers);
```

Why am I showing you this? You have to be aware of this particular thing about javascript because we are going to create many helper functions, like `unary`, that manipulate other functions. It may take a while to get use to the idea of treating functions like data but it's something you should definitely put in practice, is just one those patterns that you see a lot in functional programming.

## Composition in practice

Let's get back to our example with the `.env`. We'll recreate what we did with `bash`. First we'll take a very direct approach, then we'll explore the flaws of our implementation and try to fix them.

So, we've done this before, we know what to do. Let's start by creating a function for each step.

- Get the content of the file.

```js
const fs = require('fs');

function get_env() {
  return fs.readFileSync('.env', 'utf-8');
}
```

- Filter the content based on a pattern.

```js
function search_host(content) {
  const exp = new RegExp('^HOST=');
  const lines = content.split('\n');

  return lines.find(line => exp.test(line));
}
```

- Get the value.

```js
function get_value(str) {
  return str.split('=')[1];
}
```

We're ready. Let's see what we can do to make these functions work together.

### Natural composition

I've already mentioned that our first try would be direct, the functions are ready and now the only thing we need to do is execute them in sequence.

```js
get_value(search_host(get_env()));
```

This is the perfect setup for function composition, the output of a function becomes the input of the next one, which is the same thing the `|` symbol does in `bash`. But unlike `bash`, in here the data flow goes from right to left.

Now let's imagine that we have two more functions that do something with the value of `HOST`.

```js
test(ping(get_value(search_host(get_env()))));
```

Okay, now things are starting to get a little awkward, it's still on a manageable level but the amount of parenthesis in it bothers me. This would be the perfect time to put all those things in a function and group them in a more readable way but let's not do that yet, first we get help.

### Automatic composition

This is where our new found knowledge about functions starts being useful. To solve our parenthesis problem we are going to "automate" the function calls, we'll make a function that takes a list of functions, calls them one by one and makes sure the output of one becomes the input of the next. 

```js
function compose(...fns) {
  return function _composed(...args) {
    // Index of the last function
    let last = fns.length - 1;

    // Call the last function
    // with arguments of `_composed`
    let current_value = fns[last--](...args);

    // loop through the rest in the opposite direction
    for (let i = last; i >= 0; i--) {
      current_value = fns[i](current_value);
    }

    return current_value;
  };
}
```

Now we can do this.

```js
const get_host = compose(get_value, search_host, get_env);

// get_host is `_composed`
get_host();
```

Our parenthesis problem is gone, we can add more functions without hurting the readability.

```js
const get_host = compose(
  test,
  ping,
  get_value,
  search_host,
  get_env
);

get_host();
```

Just like in our first try, in here the data flows from right to left. If you want to flip the order you'd do it like this.

```js
function pipe(...fns) {
  return function _piped(...args) {
    // call the first function
    // with the arguments of `_piped`
    let current_value = fns[0](...args);

    // loop through the rest in the original order
    for (let i = 1; i < fns.length; i++) {
      current_value = fns[i](current_value);
    }

    return current_value;
  };
}
```

Behold.

```js
const get_host = pipe(get_env, search_host, get_value);

get_host();
```

All of this is great, but like I said before what we got here is the perfect setup. Our composition can only handle functions that take one parameter, and doesn't support flow control. That's not a bad thing, we should design our code so we can make this kind of composition more common but as we all know...

### It's not always easy

Even in our example the only reason we were able to compose those functions was because we included everything we needed inside in the code, and we completely ignored the error handling. But not all is lost, there are ways to get over the limitations.

Before we move on I would like to change the example code, I'll make it look more like the `bash` implementation.

```js
const fs = require('fs');

function cat(filepath) {
  return fs.readFileSync(filepath, 'utf-8');
}

function grep(pattern, content) {
  const exp = new RegExp(pattern);
  const lines = content.split('\n');

  return lines.find(line => exp.test(line));
}

function cut({ delimiter, fields }, str) {
  return str.split(delimiter)[fields - 1];
}
```

They are not exactly like their `bash` counterparts but they do the job. But now if we wanted to put them together it would have to be like this.

```js
cut({delimiter: '=', fields: 2}, grep('^HOST=', cat('.env')));
```

It works but I'd say that is barely acceptable, I can still understand what is going on but I wouldn't want to add a single thing to that chain. If we want to use `pipe` we'll have to overcome our first obstacle.

#### Functions with multiple inputs

The solution to this is **partial application** and lucky for us javascript has a great support for the things we want to do. Our goal is simple, we are going to pass some of the parameters a function needs but without calling it. We want to be able to do this.

```js
const get_host = pipe(
  cat,
  grep('^HOST='), 
  cut({ delimiter: '=', fields: 2 })
);

get_host('.env');
```

To make this possible we are going to rely on a technique called **currying**, this consists on turning a multiple parameter function into several one parameter functions. The way we do this is by taking one parameter at a time, just keep returning functions until we get everything we need. We will do this to `grep` and `cut`.

```diff
- function grep(pattern, content) {
+ function grep(pattern) {
+   return function(content) {
      const exp = new RegExp(pattern);
      const lines = content.split('\n');
 
      return lines.find(line => exp.test(line));
+   }
  }

- function cut({ delimiter, fields }, str) {
+ function cut({ delimiter, fields }) {
+   return function(str) {
      return str.split(delimiter)[fields - 1];
+   }
  }
```

In situations where is not possible to make a normal function support currying we can use the [bind](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Function/bind) method in the `Function` prototype.

```js
const get_host = pipe(
  cat,
  grep.bind(null, '^HOST='), 
  cut.bind(null, { delimiter: '=', fields: 2 })
);
```

Lastly, if everything else looks too complex you always have the possibility to create an arrow function inline.

```js
const get_host = pipe(
  cat,
  content => grep('^HOST=', content), 
  str => cut({ delimiter: '=', fields: 2 }, str)
);
```

That should be enough to solve any kind of problem you face when you deal with multiple parameters. Let's move on.

#### Functions with multiple outputs

Multiple outputs? I mean functions whose return value can have more than one type. This happens when we have functions that respond differently depending on how we use them or in what context. We have that kind of functions in our example. Let's take a look at `cat`.

```js
function cat(filepath) {
  return fs.readFileSync(filepath, 'utf-8');
}
```

Inside `cat` we have `readFileSync`, that's the one that reads the file in our system, an action that can fail for many reasons. It means that `cat` can return a `String` if everything goes well but can also throw an error if anything goes wrong. We need to handle both cases.

Unfortunately for us exceptions are not the only thing we need to worry about, we also need to deal with the absence of values. In `grep` we have this line.

```js
lines.find(line => exp.test(line));
```

The `find` method is the one that evaluates each line of the file. As you can imagine that can fail, maybe it just doesn't find what we are looking for. Unlike `readFileSync`, `find` doesn't throw an error, it just returns `undefined`. It's not like `undefined` is bad, it's that we don't have any use for it. Assuming that the result will always be a `String` is what can cause an error.

How do we handle all this?

**Functors** && **Monads** (sorry for the big words). Giving an appropiate explanation of those two would take too much time so we are just going to focus on the practical aspects. For the time being you can think of them as data types that need to obey some laws (you can find some of them here: [Fantasy land](https://github.com/fantasyland/fantasy-land#fantasy-land-specification)).

Where do we start? With functors.

- Functors

Let's create a data type that is capable of calling a function in the right context at the right time. You have seen one before: arrays. Try this.

```js
const add_one = num => num + 1;
const number = [41];
const empty = [];

number.map(add_one); // => [42]
empty.map(add_one);  // => []
```

See? `map` called `add_one` just once, on the `number` array. It didn't do anything on the `empty` array, didn't halt the execution of the script by throwing an error, it just returned an array. That's the behavior that we want.

We will make that on our own. Let's create a data type called `Result`, it will represent an action that may or may not be successful. It will have a `map` method that will only execute the provided callback when the action had the expected outcome.

```js
const Result = {};

Result.Ok = function(value) {
  return {
    map: fn => Result.Ok(fn(value)),
  };
}

Result.Err = function(value) {
  return {
    map: () => Result.Err(value),
  };
}
```

We have our functor but now you might be wondering is that it? How does that help? We are taking it one step at a time. Let's use it with `cat`.

```js
function cat(filepath) {
  try {
    return Result.Ok(fs.readFileSync(filepath, 'utf-8'));
  } catch(e) {
    return Result.Err(e);
  }
}
```

What do we gain with this? Give it a chance.

```js
cat('.env').map(console.log);
```

You still have the same question on your mind, I can see it. Now try to add the other functions.

> Note: I'm going to assume that you can use currying to achieve partial application.

```js
cat('.env')
  .map(grep('^HOST='))
  .map(cut({ delimiter: '=', fields: 2 }))
  .map(console.log);
```

See that? That chain of `map`s looks a lot like `compose` or `pipe`. We did it, we got our composition back, and now with error handling (kinda).

I want to do something. That pattern, the one with the `try/catch`, I want to put that in a function.

```js
 Result.make_safe = function(fn) {
  return function(...args) {
    try {
      return Result.Ok(fn(...args));
    } catch(e) {
      return Result.Err(e);
    }
  }
 }
```

Now we can transform `cat` without even touching its code.

```js
const safer_cat = Result.make_safe(cat);

safer_cat('.env')
  .map(grep('^HOST='))
  .map(cut({ delimiter: '=', fields: 2 }))
  .map(console.log);
```

You may want to do something in case something goes wrong, right? Let's make that possible.

```diff
  const Result = {};
 
  Result.Ok = function(value) {
    return {
      map: fn => Result.Ok(fn(value)),
+     catchMap: () => Result.Ok(value),
    };
  }
 
  Result.Err = function(value) {
    return {
      map: () => Result.Err(value),
+     catchMap: fn => Result.Err(fn(value)),
    };
  }
```

Now we can make mistakes and be confident we are doing something about it.

```js
const safer_cat = Result.make_safe(cat);
const show_error = e => console.error(`Whoops:\n${e.message}`);

safer_cat('what?')
  .map(grep('^HOST='))
  .map(cut({ delimiter: '=', fields: 2 }))
  .map(console.log)
  .catchMap(show_error);
```

Yes, I know, all of this is cute and useful but at some point you'll want to take the value out of the `Result`. I get it, javascript is not a language where this pattern is a common thing, you may want to go "back to normal". Let's add a function that can let us extract the value out in either case.

```diff
  const Result = {};
 
  Result.Ok = function(value) {
    return {
      map: fn => Result.Ok(fn(value)),
      catchMap: () => Result.Ok(value),
+     cata: (error, success) => success(value)
    };
  }
 
  Result.Err = function(value) {
    return {
      map: () => Result.Err(value),
      catchMap: fn => Result.Err(fn(value)),
+     cata: (error, success) => error(value)
    };
  }
```

With this we can choose what to do at the end of every action.

```js
const constant = arg => () => arg;
const identity = arg => arg;

const host = safer_cat('what?')
  .map(grep('^HOST='))
  .map(cut({ delimiter: '=', fields: 2 }))
  .cata(constant("This ain't right"), identity)

// ....
```

> Note: If you're asking why `cata`, it comes from **catamorphism**, another one of those terms people use in functional programming.

Now let's create a data type that can handle the problem we have with `grep`. In this case what we want to do is handle the absence of a value.

```js
const Maybe = function(value) {
  if(value == null) {
    return Maybe.Nothing();
  }

  return Maybe.Just(value);
}

Maybe.Just = function(value) {
  return {
    map: fn => Maybe.Just(fn(value)),
    catchMap: () => Maybe.Just(value),
    cata: (nothing, just) => just(value)
  };
}

Maybe.Nothing = function() {
  return {
    map: () => Maybe.Nothing(),
    catchMap: fn => fn(),
    cata: (nothing, just) => nothing()
  };
}

Maybe.wrap_fun = function(fn) {
  return function(...args) {
    return Maybe(fn(...args));
  }
}
```

We're going to use it to wrap `grep` with a `Maybe`, to test this we'll use the original `cat` to take the content from the file.

```js
const maybe_host = Maybe.wrap_fun(grep('^HOST='));

maybe_host(cat('.env'))
  .map(console.log)
  .catchMap(() => console.log('Nothing()'));
```

That should show `http://localhost:5000`. And if we change the pattern `^HOST=` it should show `Nothing()`.

So, we created safer versions of `cat` and `grep` but you should see what happens when they get together.

```js
safer_cat('.env')
  .map(maybe_host)
  .map(res => console.log({ res }));
  .catchMap(() => console.log('what?'))
```

You get this.

```
{
  res: {
    map: [Function: map],
    catchMap: [Function: catchMap],
    cata: [Function: cata]
  }
}
```

Wait, what's happening? Well, we have a `Maybe` trapped inside a `Result`. Maybe you didn't see that one coming but other people did, and they have the solution.

- Monads

It turns out that monads are functors with extra powers. The thing we care about right now is that they solve the nesting issue. Let's make some adjustments.

```diff
  Result.Ok = function(value) {
    return {
      map: fn => Result.Ok(fn(value)),
      catchMap: () => Result.Ok(value),
+     flatMap: fn => fn(value),
      cata: (error, success) => success(value)
    };
  }

  Result.Err = function(value) {
    return {
      map: () => Result.Err(value),
      catchMap: fn => Result.Err(fn(value)),
+     flatMap: () => Result.Err(value),
      cata: (error, success) => error(value)
    };
  }
```

```diff
  Maybe.Just = function(value) {
    return {
      map: fn => Maybe.Just(fn(value)),
      catchMap: () => Maybe.Just(value),
+     flatMap: fn => fn(value),
      cata: (nothing, just) => just(value),
    };
  }

  Maybe.Nothing = function() {
    return {
      map: () => Maybe.Nothing(),
      catchMap: fn => fn(),
+     flatMap: () => Maybe.Nothing(),
      cata: (nothing, just) => nothing(),
    };
  }
```

The `flatMap` method behaves just like `map` but with the added benefit that it lets us get rid of those extra "layers" that mess around with our composition. Make sure to use `flatMap` with functions that return other monads because this is not the safest implementation.

> Note: Yes, arrays are also monads. They have a `map` method and `flatMap` method and they obey all the laws.

Let's test `maybe_host` again.

```js
 safer_cat('.env')
  .flatMap(maybe_host)
  .map(res => console.log({ res }));
  .catchMap(() => console.log('what?'))
```

That should give us.

```
{ res: 'HOST=http://localhost:5000' }
```

We're ready to compose everything back together.

```js
const safer_cat = Result.make_safe(cat);
const maybe_host = Maybe.wrap_fun(grep('^HOST='));
const get_value = Maybe.wrap_fun(cut({delimiter: '=', fields: 2}));

const host = safer_cat('.env')
  .flatMap(maybe_host)
  .flatMap(get_value)
  .cata(
    () => 'http://127.0.0.1:3000',
    host => host
  );

// ....
```

And if we want to use `pipe` or `compose`?

```js
const chain = fn => m => m.flatMap(fn);
const unwrap_or = fallback => fm => 
  fm.cata(() => fallback, value => value);


const safer_cat = Result.make_safe(cat);
const maybe_host = Maybe.wrap_fun(grep('^HOST='));
const get_value = Maybe.wrap_fun(cut({delimiter: '=', fields: 2}));

const get_host = pipe(
  safer_cat,
  chain(maybe_host),
  chain(get_value),
  unwrap_or('http://127.0.0.1:3000')
);

get_host('.env');
```

You can check out the whole code here: [link](https://gist.github.com/VonHeikemen/0e6d4950bfe91229ee06eee2e3c74515).

## Still want to learn more?

There are many things that I didn't mention because it would take too much time but if you want to learn more about it, I have prepare some material.

- [Partial application](@/web-development/fp-in-js/partial-application.md)
- [About Functors](@/web-development/fp-in-js/the-power-of-map.md)
- [Using a Maybe](@/web-development/fp-in-js/using-a-maybe.md)
- [Pure functions and side effects](@/web-development/fp-in-js/dealing-with-side-effects-and-pure-functions.md)

## Conclusion

A lot of people talk about the nice things about composition, how it makes code more declarative and clean, but they never show you the tough parts. I hope I've done that, show the tough parts and how to overcome them. Composing functions it's truly an art, it takes practice and time to get use to some ideas (like the idea of functions being things).

## Sources

- [The Power of Composition (video)](https://www.youtube.com/watch?v=vDe-4o8Uwl8)
- [Oh Composable World! (video)](https://www.youtube.com/watch?v=SfWR3dKnFIo)
- [Mary had a little lambda (video)](https://www.youtube.com/watch?v=7BsfMMYvGaU)
- [Functional JavaScript - Functors, Monads, and Promises](https://dev.to/joelnet/functional-javascript---functors-monads-and-promises-1pol)
