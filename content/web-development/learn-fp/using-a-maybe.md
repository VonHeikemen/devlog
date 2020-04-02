+++
title = "Functional programming for your everyday javascript: Using a Maybe"
description = "We'll see how an implementation of the Maybe Type can influence how we write code"
date = 2019-10-28
lang = "en"
[taxonomies]
tags = ["javascript", "functional-programming", "learning"]
[extra]
canonical_url = "https://dev.to/vonheikemen/functional-programming-for-your-everyday-javascript-using-a-maybe-126j"
+++

Have you ever heard about monads and how great they are? Maybe you have but you still don't get it. Well... I'm not here to tell you what they are, I won't try to sell them to you either, what I will do is show you an example of how would it be if you use them in your javascripts.

We'll do something fun, lets solve a fairly trivial problem in an unnecessary complicated way. 

Suppose that we have a dictionary stored in a json file or a plain js object.

```js
{
    "accident": ["An unexpected, unfortunate mishap, failure or loss with the potential for harming human life, property or the environment.", "An event that happens suddenly or by chance without an apparent cause."], 
    "accumulator": ["A rechargeable device for storing electrical energy in the form of chemical energy, consisting of one or more separate secondary cells.\\n(Source: CED)"],
    "acid": ["A compound capable of transferring a hydrogen ion in solution.", "Being harsh or corrosive in tone.", "Having an acid, sharp or tangy taste.", "A powerful hallucinogenic drug manufactured from lysergic acid.", "Having a pH less than 7, or being sour, or having the strength to neutralize  alkalis, or turning a litmus paper red."],
    
     // ... moar words and meanings
    
    "Paris": ["The capital and largest city of France."]
  }
```

We want a form that lets a user search one of this words and then shows the meaning(s). This is simple, right? What could possibly go wrong?

Because everyone loves HTML we'll start with that.

```html
<form id="search_form">
  <label for="search_input">Search a word</label>
  <input id="search_input" type="text">
  <button type="submit">Submit</button>
</form>

<div id="result"></div>
```

In the first version we will just try get one those values based on the user input.

```js
// main.js

// magically retrieve the data from a file or whatever
const entries = data();

function format(results) {
  return results.join('<br>'); // I regret nothing
}

window.search_form.addEventListener('submit', function(ev) {
  ev.preventDefault();
  let input = ev.target[0];
  window.result.innerHTML = format(entries[input.value]);
});
```

Naturally the first thing we try to search is "acid." And behold here are the results.

> A compound capable of transferring a hydrogen ion in solution.
Being harsh or corrosive in tone.
Having an acid, sharp or tangy taste.
A powerful hallucinogenic drug manufactured from lysergic acid.
Having a pH less than 7, or being sour, or having the strength to neutralize alkalis, or turning a litmus paper red.

Now we search for "paris", I'm sure it's there. What did we get? Nothing. Not exactly nothing, we got.

> TypeError: results is undefined

We also got an unpredictable submit button that sometime works and sometimes doesn't. So what do we want? What do we really, really want? Safety, objects that don't crash our application, we want reliable objects. 

What we will do is implement containers that let us describe the flow of execution without worrying about the value they hold. Sounds good, right? Let me show you what I mean with a little javascript. Try this.

```js
const is_even = num => num % 2 === 0;

const odd_arr = [1,3,4,5].filter(is_even).map(val => val.toString());
const empty_arr = [].filter(is_even).map(val => val.toString());

console.log({odd_arr, empty_arr});
```

Did it throw an exception on the empty array? (if it did let me know). Isn't that nice? Doesn't it feel all warm and fuzzy knowing that the array methods would do the right thing even if there isn't anything to work with? That is what we want.

You might be wondering couldn't we just write a few `if` statements and be done with it? Well... yeah, but where is the fun in that? We all know that chaining functions is cool, and we are fans of functional programming, we do what every functional programming savvy does: **hide things under a function**.

So we are going to hide an `if` statement (or maybe a couple), if the value we evaluate is undefined-ish we return a wrapper that will know how to behave no matter what happens.

```js
// maybe.js
// (I would like to apologize for the many `thing`s you'll see)

function Maybe(the_thing) {
  if(the_thing === null 
     || the_thing === undefined 
     || the_thing.is_nothing
  ) {
    return Nothing();
  }

  // I don't want nested Maybes
  if(the_thing.is_just) {
    return the_thing;
  }

  return Just(the_thing);
}
```

This wrappers are not going to be your standard by the book `Maybe` you see in a proper functional programming language. We will cheat a little in the name of convenience and side effects. Also their methods will be named after the methods in the Option type you find in Rust (I like those names better). Here is where the magic happens.

```js
// maybe.js

// I lied, there will be a lot of cheating and `fun`s.

function Just(thing) {
  return {
    map: fun => Maybe(fun(thing)),
    and_then: fun => fun(thing),
    or_else: () => Maybe(thing),
    tap: fun => (fun(thing), Maybe(thing)),
    unwrap_or: () => thing,
    
    filter: predicate_fun => 
      predicate_fun(thing) 
        ? Maybe(thing) 
        : Nothing(),
    
    is_just: true,
    is_nothing: false,
    inspect: () => `Just(${thing})`,
  };
}

function Nothing() {
  return {
    map: Nothing,
    and_then: Nothing,
    or_else: fun => fun(),
    tap: Nothing,
    unwrap_or: arg => arg,

    filter: Nothing,

    is_just: false,
    is_nothing: true,
    inspect: () => `Nothing`,
  };
}
```

What is the purpose of these methods?

- `map`: Applies the function `fun` to `the_thing` and wraps it again on a Maybe to keep the party going... I mean to keep the shape of the object, so you can keep chaining functions.
- `and_then`: This is mostly an escape hatch. Apply the function `fun` and let fate decide.
- `or_else`: It is the `else` to your `map` and `and_then`. The other path. The "what if is not there?"
- `tap`: These one is there just for the side effects. If you see it then it's probably affecting something outside of it's scope (or maybe is just the perfect place to put a `console.log`).
- filter: It "lets you go through" if the predicate function returns something truthy.
- `unwrap_or`: This is how you get `the_thing` out. You'll want this when you're done chaining methods and you're ready to get back to the imperative world.

Lets go back to our form and see it in action. We'll make a function `search` that may o may not retrieve a match to the user's query. If it does we'll chain other functions that will be executed in a "safe context."

```js
// main.js

const search = (data, input) => Maybe(data[input]);

const search_word = word => search(entries, word)
  .map(format)
  .unwrap_or('word not found');
```

And now we replace our unholy old way with the new safe(r) function.

```diff
 window.search_form.addEventListener('submit', function(ev) {
   ev.preventDefault();
   let input = ev.target[0];
-  window.result.innerHTML = format(entries[input.value]);
+  window.result.innerHTML = search_word(input.value);
 });
```

Now we test. Search for "accident."

> An unexpected, unfortunate mishap, failure or loss with the potential for harming human life, property or the environment.
An event that happens suddenly or by chance without an apparent cause.

Now Paris. Search for "paris."

> word not found

It didn't freeze the button, that's good. But I know Paris is there. If you check you'll see that is "Paris." We'll just capitalize the user input so they don't have to. First we'll try to search the exact input, if that fails we'll try the capitalize way.

```js
// main.js

function create_search(data, exact) {
  return input => {
    const word = exact ? input : capitalize(input);
    return Maybe(data[word]);
  }
}

function capitalize(str) {
  return str.charAt(0).toUpperCase() + str.slice(1);
}
```

Change the search function.

```diff
- const search = (data, input) => Maybe(data[input]);
+ const search = create_search(entries, true);
+ const search_name = create_search(entries, false);
-
- const search_word = word => search(entries, word)
+ const search_word = word => search(word)
+   .or_else(() => search_name(word))
    .map(format)
    .unwrap_or('word not found');
```

Very nice. This what we got so far in main.js if you wanna see the whole picture.

```js
// main.js

const entries = data();

function create_search(data, exact) {
  return input => {
    const word = exact ? input : capitalize(input);
    return Maybe(data[word]);
  }
}

function capitalize(str) {
  return str.charAt(0).toUpperCase() + str.slice(1);
}

function format(results) {
  return results.join('<br>');
}

const search = create_search(entries, true);
const search_name = create_search(entries, false);

const search_word = word => search(word)
  .or_else(() => search_name(word))
  .map(format)
  .unwrap_or('word not found');

window.search_form.addEventListener('submit', function(ev) {
  ev.preventDefault();
  let input = ev.target[0];
  window.result.innerHTML = search_word(input.value);
});
```

But is that all we want in life? No, of course not, we want love but since javascript can't give us that we'll settle for a little "suggest word" feature. I want to search "accu" and have a confirm dialog telling me "Did you mean accumulator?"

We'll need help with this one, we'll bring a dependency, one that can perform a fuzzy search on the entries: [fuzzy-search](https://github.com/wouter2203/fuzzy-search#readme). So we add the following. 

```js
// main.js

import FuzzySearch from 'https://unpkg.com/fuzzy-search@3.0.1/src/FuzzySearch.js';

const fzf = new FuzzySearch(
  Object.keys(entries),
  [],
  {caseSensitive: false, sort: true}
);
```

But again we can't perform a safe operation 'cause the moment we try to get a match from an empty array the whole thing will fall apart. So what do we do? We hide things under a function.

```js
// main.js

function suggest(word) {
  const matches = fzf.search(word);
  return Maybe(matches[0]);
}
```

Fuzzy search is ready, now lets throw in a super awesome confirm dialog. You'll love it.

```js
// main.js

function confirm_word(value) {
  if(value && confirm(`Did you mean ${value}`)) {
    return value;
  }
}
```

We combine the new functions with our `search`.

```js
// main.js

const suggest_word = value => () => suggest(value)
  .map(confirm_word)
  .map(search);
```

Add the feature to `search_word`.

```diff
 const search_word = word => search(word)
   .or_else(() => search_name(word))
+  .or_else(suggest_word(word))
   .map(format)
   .unwrap_or('word not found');
```

That works! But lets say we are allergic to `if` statements and not to mention that it's just rude to return `undefined` from a function. We can do better.

```diff
 function confirm_word(value) {
-  if(value && confirm(`Did you mean ${value}`)) {
-    return value;
-  }
+  return confirm(`Did you mean ${value}`);
 }
```

```diff
 const suggest_word = value => () => suggest(value)
-  .map(confirm_word)
+  .filter(confirm_word)
   .map(search);
```

Something bugs me. I search "accu", the dialog pops in, I confirm the suggestion and the results appears. But "accu" it's still there in the input, it's awkward. Lets update the input with the right word.

```js
const update_input = val => window.search_form[0].value = val;
```
```diff
 const suggest_word = value => () => suggest(value)
   .filter(confirm_word)
+  .tap(update_input)
   .map(search);
```

Want to see it in action? There you go.

{{ codepen(id="JjjNvLE", title="Maybe I got your word", load_js=true) }}

## Bonus track

> *Warning*: The main point of the post (which is me showing that codepen example) was already accomplished. What follows is a strange experiment to see if I could make that `Maybe` function support asynchronous operations. If you are tired just skip everything and check out the last example code.  

Now you might be saying: this is cute and all but in the "real world" we make http requests, query a database, make all sorts of asynchronous stuff, can this still be useful in that context?

I hear you. Our current implementation just supports normal blocking tasks. You would have to break the chain of `Maybes` the moment a `Promise` shows up. 

But what if... listen... we make a promise aware `Just`. We can do that, an `AsyncJust`? `JustAsync`? Oh, that's awful.

If you don't know, a `Promise` is a data type that javascript uses to coordinate future events. To do so it uses a method called `then` that takes a callback (it also has `catch` for when things go wrong) So if we hijack what goes into that `then` then we can keep our nice `Maybe` interface.

How good are you following a bunch of callbacks?

Here I go. Let me show you the `Future`.

```js
// Don't judge me. 

function Future(promise_thing) { 
  return {
    map: fun => Future(promise_thing.then(map_future(fun))),
    and_then: fun => Future(promise_thing.then(map_future(fun))),
    or_else: fun => Future(promise_thing.catch(fun)),
    tap: fun => Future(promise_thing.then(val => (fun(val), val))),
    unwrap_or: arg => promise_thing.catch(val => arg),

    filter: fun => Future(promise_thing.then(filter_future(fun))), 

    is_just: false,
    is_nothing: false,
    is_future: true,
    inspect: () => `<Promise>`
  };
}
```

If we remove the noise maybe we could understand better.

```js
// In it's very core is callbacks all the way.

{
  map: fun => promise.then(fun),
  and_then: fun => promise.then(fun),
  or_else: fun => promise.catch(fun),
  tap: fun => promise.then(val => (fun(val), val))),
  unwrap_or: arg => promise.catch(val => arg),

  filter: fun => promise.then(fun), 
}
```

- `map`/`and_then`: these do the same thing because you can't get out of a `Promise`. 
- `or_else`: puts your callback in the `catch` method to mimic an `else` behavior.
- `tap`: uses `then` to peek at the value. Since this is for side effects we return the value again.
- `unwrap_or`: It will return the promise so you can use `await`. If everything goes well the original value of the `Promise` will be returned when you `await`, else the provided argument will be returned. Either way the promise doesn't throw an error because the `Future` attached the `catch` method to it.
- `filter`: these one is a special kind of `map` that's why `filter_future` exists.
- Almost all these methods return a new `Future` 'cause `promise.then` returns a new `Promise`.

What makes the `Future` weird is what happens inside `map`. Remember `map_future`?

```js
function map_future(fun) { // `fun` is the user's callback
  return val => {
    /* Evaluate the original value */
    let promise_content = val;

    // It needs to decide if the value of the Promise
    // can be trusted
    if(Maybe(promise_content).is_nothing) {
      Promise.reject();
      return;
    }

    // If it is a Just then unwrap it.
    if(promise_content.is_just) {
      promise_content = val.unwrap_or();
    }

    /* Evaluate the return value of the user's callback */

    // Use Maybe because I have trust issues.
    // For the javascript world is undefined and full of errors.
    const result = Maybe(fun(promise_content));

    if(result.is_just) {
      // If it gets here it's all good.
      return result.unwrap_or();
    }

    // at this point i should check if result is a Future
    // if that happens you are using them in a wrong way
    // so for now I don't do it 

    // There is something seriously wrong.
    return Promise.reject();
  }
}
```

Now `filter_future`.

```js
function filter_future(predicate_fun) { // the user's function
  return val => {
    const result = predicate_fun(val);

    // Did you just returned a `Promise`?
    if(result.then) {
      // You did! That's why you can't have nice things.
 
      // peek inside the user's promise.
      const return_result = the_real_result => the_real_result 
        ? val
        : Promise.reject();

      // keep the promise chain alive.
      return result.then(return_result);
    }

    return result ? val : Promise.reject();
  }
}
```

There is one last thing I would like to do and that is create a helper function to convert a regular value into a `Future`.

```js
Future.from_val = function(val) {
  return Future(Promise.resolve(val));
}
```

All we have to do now to support a `Future` in a `Maybe` is this.

```diff
 function Maybe(the_thing) {
   if(the_thing === null 
     || the_thing === undefined 
     || the_thing.is_nothing
   ) {
     return Nothing();
   }
-
-  if(the_thing.is_just) {
+  if(the_thing.is_future || the_thing.is_just) {
     return the_thing;
    }

    return Just(the_thing);
 }
```

But the million dollar question remains. Does it actually work?

I have [CLI version](https://github.com/VonHeikemen/maybe-type-in-js) of this. And here is the same codepen example with some tweaks: I added the `Future` related functions, the confirm dialog is actually a dialog ([this one](https://github.hubspot.com/vex/)) and the event listener is now an async function that can `await` the result.

{{ codepen(id="oNNwagG", title="Maybe I will promise you a word") }}

### Bonus bonus edit
That is how it looks like when we cheat. If we didn't cheat it would be like [this](https://codepen.io/VonHeikemen/pen/QWWYJwZ).

## Other resources
- [Option/Maybe, Either, and Future Monads in JavaScript, Python, Ruby, Swift, and Scala](https://www.toptal.com/javascript/option-maybe-either-future-monads-js)
- [The Marvellously Mysterious JavaScript Maybe Monad](https://jrsinclair.com/articles/2016/marvellously-mysterious-javascript-maybe-monad/)
- [Monad Mini-Series: Functors (video)](https://www.youtube.com/watch?v=pgq-Pfg6ul4)
- [Oh Composable World! (video)](https://www.youtube.com/watch?v=SfWR3dKnFIo)
