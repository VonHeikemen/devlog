+++
title = "Functional programming for your everyday javascript: Partial application"
description = "Solving the mystery of why sometimes the callback goes first"
date = 2020-03-10
lang = "en"
[taxonomies]
tags = ["javascript"]
[extra]
canonical_url = "https://dev.to/vonheikemen/functional-programming-for-your-everyday-javascript-partial-application-6dm"
+++

Today we are here to solve a mystery, the mystery of why some people choose to create functions that take a callback as the first argument. You might be thinking that the answer is partial application and you would be half right, but partial application is just the means to an end, the real reason to do such a thing is to enable a "better" function composition. But before we get into details of partial application let's explore how we do things now.

## How we do things

When we create a function we usually sort the arguments by some kind of importance/priority level, where the most important goes first. As a result, when we create a function that works on a piece of data it becomes the first thing on the list, it's followed by less important configuration arguments and the last thing are optional arguments that we can omit. 

Say that we want to create a function that picks specific properties from a plain object. Let's think of what we need. The object, that was your first thought? It's natural, you don't want to omit it by accident when you call the function. That leaves the keys that we are going to choose as the last argument.

```js
function pick(obj, keys) {
  let result = {};
  
  for(key of keys) {
    result[key] = obj[key];
  }
  
  return result;
}
```

> Note: We are not the only ones that think like this, check out [lodash pick](https://lodash.com/docs/#pick)

Now, say that we have a `user` object and we want to hide any "sensitive" data. We would use it like this.

```js
const user = {
  id: 7,
  name: "Tom",
  lastname: "Keen",
  email: "noreply@example.com",
  password: "hudson"
};

pick(user, ['name', 'lastname']); 

// { name: "Tom", lastname: "Keen" }
```

That works great, but what happens when we need to work with an array of users? 

```js
const users = [
  {
    id: 7,
    name: "Tom",
    lastname: "Keen",
    email: "noreply@example.com",
    password: "hudson"
  },
  {
    id: 30,
    name: "Smokey",
    lastname: "Putnum",
    email: "noreply@example.com",
    password: "carnival"
  },
  {
    id: 69,
    name: "Lady",
    lastname: "Luck",
    email: "noreply@example.com",
    password: "norestforthewicked"
  }
];
```

We are force to iterate over the array and apply the function.

```js
users.map(function(user) {
  return pick(user, ['name', 'lastname']);
});

/*
[
  {"name": "Tom", "lastname": "Keen"},
  {"name": "Smokey", "lastname": "Putnum"},
  {"name": "Lady", "lastname": "Luck"}
]
*/
```

Is not that bad. And you know what? That callback actually looks useful. We could put it in another place and give it a name.

```js
function public_info(user) {
  return pick(user, ['name', 'lastname']);
}

users.map(public_info);
```

What is actually happening? What we do here is bind the second argument to the function with the value `['name', 'lastname']` and force `pick` to wait for the user data to be executed. 

Now let's take this example one step further, pretend that `Async/Await` doesn't exists and that the `users` array comes from a `Promise`, maybe an http requests using `fetch`. What do we do?

```js
fetch(url).then(function(users) {
  users.map(function(user) {
    return pick(user, ['name', 'lastname']);
  })
});
```

Now that is bad. Maybe some arrow functions can make it better?

```js
fetch(url).then(users => users.map(user => pick(user, ['name', 'lastname'])));
```

Is it better? A question for another day. We prepared for this, we have the `public_info` function let's use it.

```js
fetch(url).then(users => users.map(public_info));
```

This is acceptable, I like it. If we wanted we could make another function that binds `public_info` to `.map`.

```js
function user_list(users) {
  return users.map(public_info);
}
```

So now we get.

```js
fetch(url).then(user_list);
```

Let's see everything we needed for that.

```js
function pick(obj, keys) {
  // code...
}

function public_info(user) {
  return pick(user, ['name', 'lastname']);
}

function user_list(users) {
  return users.map(public_info);
}

fetch(url).then(user_list);
```

What if I told you that we can create `public_info` and `user_list` in another way? What if we could have this?

```js
const public_info = pick(['name', 'lastname']);
const user_list = map(public_info);

fetch(url).then(user_list);
```

Or put everything inline if that is your jam.

```js
fetch(url).then(map(pick(['name', 'lastname'])));
```

We can have it but first we'll need to change the way we think about functions a little bit. 

## Thinking differently

Instead of thinking of priority we should start thinking in dependencies and data. When you're creating a function just ask yourself, out of all this arguments what is the most likely to change? Put that as your last argument.

Let's make a function that takes the first elements of something. What do we need? We need that "something" and also the number of elements we are going to take. Of those two, which is most likely to change? It's the data, that "something".

```js
function take(count, data) {
  return data.slice(0, count);
}
```

In a normal situation you would use it like this.

```js
take(2, ['first', 'second', 'rest']);

// ["first", "second"]
```

But with a little bit of magic (which will be revealed soon) you can reuse it like this.

```js
const first_two = take(2);

first_two(['first', 'second', 'rest']);
```

This way ordering your arguments gets even more convenient when callbacks are involved. Let's "reverse" `Array.filter` arguments and see what we can do.

```js
function filter(func, data) {
  return data.filter(func);
}
```

We start simple, exclude falsey values from an array.

```js
filter(Boolean, [true, '', null, 'that']);

// => [ true, "that" ]
```

That's good and it could be better if we add more context.

```js
const exclude_falsey = filter(Boolean);

exclude_falsey([true, '', null, 'that']);
```

I'm hoping you can see the possibilities that this kind of pattern can provide. There are libraries (like [Ramda](https://ramdajs.com/docs/)) that use this approach to build complex functions by assembling smaller single purpose utilities. 

Enough talking, let's see now how we can do this ourselves.

## This is the way

Like with everything in javascript you can do this in a million ways, some are more convenient than others, some require a little bit of magic. Let us begin.

### The built-in magic of bind

Turns out that we don't need to do anything extraordinary to bind values to the arguments of a function because every function has a method called [bind](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Function/bind). The syntax is not as convenient as the one I showed but it gets close. Another thing that you have to be aware of is that the first argument to `Function.bind` is the "context", that is the value of the keyword `this` inside the function. This is the basic usage.

```js
const exclude_falsey = filter.bind(null, Boolean);

exclude_falsey([true, '', null, 'that']);
```

### The magic within

This one requires some work and it involves another enigmatic keyword, the `arguments`. What we will do is leverage the fact that `arguments` is an array-like structure that has a length, we will count the argument the function gets and if its less than what we want we return another function. Sounds confusing?

```js
function filter(func, data) {

  // This is it. We are counting.
  if(arguments.length === 1) {
    // if .length is 1 that means we got `func`
    // it also means we don't have `data`
    // so we return another function that
    // remembers `func` and wait for `data`
    return arg => filter(func, arg);
  }

  return data.filter(func);
}
```

Now it is possible to do this.

```js
const exclude_falsey = filter(Boolean);

exclude_falsey([true, '', null, 'that']);
```

And also.

```js
filter(Boolean, [true, '', null, 'that']);
```

Isn't that nice?

### A simple approach?

And of course we can also create our bind utility. With the help of the spread operator we can collect arguments and simply apply them to a callback. 

```js
function bind(func, ...first_args) {
  return (...rest) => func(...first_args, ...rest);
}
```

The first step gets the function and collects a list of arguments into an array, then we return a function that collects another list of arguments and finally call `func` with everything.

```js
const exclude_falsey = bind(filter, Boolean);

exclude_falsey([true, '', null, 'that']);
```

The cool thing about this one is that if you flip `first_args` with `rest` you have a `bind_last` function.

### No more magic

I do have mixed feelings about this one but it really is the most simple.

```js
function filter(func) {
  return function(data) {
    return data.filter(func);
  }
}
```

Which is equivalent to this.

```js
const filter = func => data => data.filter(func);
```

The idea is to take one argument at a time in separate functions. Basically, keep returning functions until you have all the arguments you need. This is what people call "currying". How do you use it?

```js
const exclude_falsey = filter(Boolean);

exclude_falsey([true, '', null, 'that']);
```

That is one case. This is the other.

```js
filter (Boolean) ([true, '', null, 'that']);
```

Notice the extra pair of parenthesis? That's the second function. You'll need one pair for each argument you provide.

### Curry it for me

Going back to the subject of magic, you can "automate" the process of currying using a helper function.

```js
function curry(fn, arity) {
  if (arguments.length === 1) {
    // Guess how many arguments
    // the function needs.
    // This doesn't always work.
    arity = fn.length;
  }

  // Omit `fn` and `arity`, gather the rest
  const rest = Array.prototype.slice.call(arguments, 2);

  // Do we have what we need?
  if (arity <= rest.length) {
    return fn.apply(fn, rest);
  }

  // Execute `curry.bind` with `fn`, `arity` and `rest` as arguments
  // it will return a function waiting for more arguments
  return curry.bind.apply(curry, [null, fn, arity, ...rest]);
}
```

With it you can transform your existing functions or create new ones that support currying from the start.

```js
const curried_filter = curry(filter);

const exclude_falsey = curried_filter(Boolean);

exclude_falsey([true, '', null, 'that']);
```

Or.

```js
const filter = curry(function(func, data) {
  return data.filter(func); 
});
```

That's it folks. Hope you had a good time reading.

## Sources
- [Hey Underscore, You're Doing It Wrong! (video)](https://www.youtube.com/watch?v=m3svKOdZijA)
- [Partial Application in JavaScript](http://benalman.com/news/2012/09/partial-application-in-javascript/)

