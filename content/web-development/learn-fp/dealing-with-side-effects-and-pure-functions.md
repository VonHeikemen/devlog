+++
title = "Dealing with side effects and pure functions in javascript"
description = "A few ideas of how to use pure functions in the real world"
date = 2020-01-05
lang = "en"
[taxonomies]
tags = ["javascript", "functional-programming"]
[extra]
canonical_url = "https://dev.to/vonheikemen/dealing-with-side-effects-and-pure-functions-in-javascript-16mg"
shared = [
  ["dev.to", "https://dev.to/vonheikemen/dealing-with-side-effects-and-pure-functions-in-javascript-16mg"],
  ["Hashnode", "https://vonheikemen.hashnode.dev/dealing-with-side-effects-and-pure-functions-in-javascript"]
]
+++

Have you ever heard the term "pure function"? What about "side effects"? If you have then probably you've heard that side effects are evil and should be avoided at all cost (just like `var`.) Here is the problem, if you write javascript you probably want to cause those side effects (specially if you get paid to write javascript) So the solution here is not to avoid all the side effects but to control them. I going to show you a few things that you can do to make your pure functions and your side effects get along just fine.

Before we start let us just do a little recap on some terms, so we can all be in the same page.

## Concepts

### Pure function

For the sake of simplicity let us say that a pure function is a function whose output is only determined by its input and has no observable effect on the outside world. The main benefit they provide (in my opinion) is predictability, if you give them the same input values they will always return you the same output. Lets look at some examples.

This one is pure.

```js
function increment(number) {
  return number + 1;
}
```

This one isn't

```js
Math.random();
```

And these are tricky.

```js
const A_CONSTANT = 1;

function increment(number) {
  return number + A_CONSTANT;
}

module.exports ={
  increment
};
```

```js
function a_constant() {
  return 1;
}

function increment(number) {
  return number + a_constant();
}
```

### Side effects

I will call a side effect to anything that compromises the purity of a function. The list includes but is not limited to:

 - Changing (mutate) an external variable in any way.
 - Showing things in the screen.
 - Writing to a file.
 - Making an http request.
 - Spawn a process.
 - Saving data in a database.
 - Calling other functions with side-effects.
 - DOM manipulation.
 - Randomness.

So, any action that can change the "state of the world" is a side effect.

## How do we use those things together?

You're probably still thinking about that side effect list, is basically everything javascript is good for and yet some people still tell you to avoid them. Don't fear I come bearing suggestions.

### Good old function composition

Another way of saying it will be: good old separation of concerns. This is the non complicated way. If there is a way to break apart a computation from a side effect then put the computation on a function and give the output to the function/block that has the side effect.

It could be as simple as doing something like this.

```js
function some_process() {
  const data = get_data_somehow();
  const clean_data = computation(data);
  const result = save(clean_data);

  return result;
}
```

Now, `some_process` still isn't pure but that's okay, we are writing javascript we don't need everything to be pure, what we need is to keep our sanity. By splitting the side effects from the pure computation we have created three independent functions that solve only one problem at a time.  You could even take it one step further and use a helper function like [pipe](https://ramdajs.com/docs/#pipe) to get rid of those intermediate variables and compose those functions directly.

```js
const some_process = pipe(get_data_somehow, computation, save);
```

But now we have created another problem, what happens when we want to make a side effect in the middle of one those? What do we do? Well if a helper function created the problem then I say use another helper function to get out of it. Something like this would work.

```js
function tap(fn) {
  return function (arg) {
    fn(arg);
    return arg;
  }
}
```

This will allow you to place a function with a side effect in the middle of chain of functions while keeping data flow.

```js
const some_process = pipe(
  get_data_somehow,
  tap(console.log),
  computation,
  tap(a_side_effect),
  save
);
```

There is argument to be made against these type of things, some people would argue that now all your logic is scattered all over the place and that you have to move around to actually know what the function does. I really don't mind, it's a matter of preference.

Let's get back to business, did you see `tap`'s signature? Look at it: `tap(fn)`. It takes a callback as a parameter lets see how we can use that to our advantage.

### Make someone else handle the problem

As we all know life isn't always so simple, sometimes we just can't make that sweet pipeline of functions. In some situations we need to do some side-effect in the middle of a process and when that happens we can always cheat. In javascript we can treat functions as values which lets us do funny things like passing functions as parameters to other functions. This way the function can have the flexibility to execute a side effect when we need to while maintaining some of the predictability that we know and love.

Say for example that you have a function that is already pure and does something to a collection of data but now for some reason you need log the original and the transformed values right after the transformation happens. What you can do is add a function as a parameter and call it in the right moment.

```js
function transform(onchange, data) {
  let result = Array.isArray(data) ? [] : {};
  for(let key in data) {
    result[key] = data[key] + 1;
    onchange(data[key], result[key]);
  }

  return result;
}
```

This technically fulfills some of the requirements of a pure function, the output (and behavior) of the function is still determined by its input, it just so happens that one of those inputs is a function that can trigger any side effect.  Again, the goal in here is not to fight against the nature of javascript and have everything be 100% pure, we want to control when the side effect happens. So in this case the one who controls whether or not to have side effects is the caller of the function. One extra benefit of this is that if you want to use that function in a unit test to prove that it still works as expected the only thing you'll need to do is supply its arguments, you don't have grab any mocking library to test it. 

You may be wondering why put the callback as the first parameter, this is really about personal preference. If you put the `thing` that changes the most frequently in the last position you make it easier to do partial application, that is binding the values of the parameters without executing the function. For example you could use `transform.bind` to create a specialized function which already has the `onchange` callback.

### Lazy effects

The idea here is to delay the inevitable. Instead of performing the side effect right away what you do is provide a way for the caller of your function to execute the side-effect when they see fit. You can do this in a couple of ways.

#### Using function wrappers

As I mentioned before in javascript you can treat functions as values and one thing you can do with values is returning them from functions. I'm talking about functions that return functions. We already saw how useful that can be and if you think about is not that crazy, how many times have you seen something like this?

```js
function Stuff(thing) {
  
  // setup

  return {
    some_method() {
      // code...
    },
    other() {
      // code...
    }
  }
}
```

This is an old school "constructor." Before, in the good ol' days of ES5, this was one way of emulating classes. Is a regular function that returns an object, and is we all know objects can have methods. What we want to do is little bit like that, we want convert the block that contains the side effect into a function and return it.

```js
function some_process(config) {

  /*
   * do some pure computation with config
   */

  return function _effect() {
   /*
    * do whatever you want in here
    */ 
  }
}
```

This way we give the caller of our function the opportunity to use the side effect when they want, and they can even pass it around and compose it with other functions. Interestingly enough this is not a very common pattern, maybe because there are other ways to achieve the same goal.

#### Using data structures

Another way to create lazy effects is to wrap a side effect inside a data structure. What we want to do is to treat our effects as regular data, have the ability to manipulate them and even chain other effects in safe way (I mean without executing them). You've probably seen this before, one example that I can think of is Observables. Take a look at this code that uses rxjs.

```js
// taken from:
// https://www.learnrxjs.io/operators/creation/create.html

/*
  Increment value every 1s, emit even numbers.
*/
const evenNumbers = Observable.create(function(observer) {
  let value = 0;
  const interval = setInterval(() => {
    if (value % 2 === 0) {
      observer.next(value);
    }
    value++;
  }, 1000);

  return () => clearInterval(interval);
});
```

The result of `Observable.create` not only delays the execution of `setInterval` but also gives you the ability to call `evenNumbers.pipe` to chain other observables that can also have other side effects. Now of course Observables and rxjs aren't the only way, we can create our own effect type. If we want to create one all we need is a function to execute the effect and another one that lets us compose effects.

```js
function Effect(effect) {
  return {
    run(...args) {
      return effect(...args);
    },
    map(fn) {
      return Effect(arg => fn(effect(arg)));
    }
  };
}

```

It may not look like much but this is actually enough to be useful. You can start composing your effects without triggering any changes to the environment. You can now do stuff like this.

```js
const persist = (data) => {
  console.log(`saving ${data} to a database...`);
  return data.length ? true : false;
};
const show_message = result => result 
  ? console.log('we good') 
  : console.log('we not good');

const save = Effect(persist).map(show_message);

save.run('some stuff');
// saving some stuff to a database...
// we good

save.run('');
// saving  to a database...
// we not good 
```
If you have used `Array.map` to compose data transformations you'll be feeling right at home when using `Effect`, all you have to do is provide the functions with the side effect and at the of the chain the resulting `Effect` will know what to do when you are ready to call it.

I've only scratched the surface of what you can do with `Effect`, if you want to learn more try to search the term `functor` and `IO Monad`, I promise you is going to be fun. 

## What now?

Now you click on the link in the end of the post, it's a really good article (basically a better version of this one). 

I hope now you are confident enough to start writing pure functions in your code and combine them with the convenient side effects that javascript lets you do.

## Sources
- [How to deal with dirty side effects in your pure functional JavaScript](https://jrsinclair.com/articles/2018/how-to-deal-with-dirty-side-effects-in-your-pure-functional-javascript/)
