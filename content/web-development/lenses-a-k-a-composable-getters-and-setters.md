+++
title = "Lenses A.K.A. composable getters and setters"
description = "Let's build a mostly adequate lenses implementation"
date = 2019-11-27
lang = "en"
[taxonomies]
tags = ["javascript", "functional-programming"]
[extra]
canonical_url = "https://dev.to/vonheikemen/lenses-a-k-a-composable-getters-and-setters-16jd"
+++

This time around we will figure out what are lenses, how do they look like in javascript and hopefully will build a mostly adequate implementation.

Let us first take a step back and ask.

## What are getters and setters?

This are functions that have one goal, they get or set value. But of course that is not the only thing they are good for. Most use cases I've seen involve triggering a side-effect when a value changes or put some validations to prevent undesired behavior.

In javascript you could make them explicit by doing something like this.

```javascript
function Some() {
  let thing = 'stuff';

  return {
    get_thing() {
      // you can do anything in here
      return thing;
    },
    set_thing(value) {
      // same in here.
      thing = value;
    }
  }
}

let obj = Some();
obj.get_thing(); // => 'stuff'

obj.set_thing('other stuff');

obj.get_thing(); // => 'other stuff'
```

Or you could make then implicit.

```javascript
let some = {};

Object.defineProperty(some, 'thing', {
  get() {
    return 'thing';
  },
  set(value) {
    console.log("can't touch this");
  }
});

some.thing // => 'thing'
some.thing = 'what?';

//
// can't touch this
//

some.thing // => 'thing'
```

But what is so wrong in there that some people feel the need to use something like lenses?

Let's start with that second example. I can tell you that some people don't like magical things, just the thought of a function being executed without their knowledge is bad enough.

The first example is bit more interesting. Let's see it again.

```javascript
obj.get_thing(); // => 'stuff'

obj.set_thing('other stuff');

obj.get_thing(); // => 'other stuff'
```

You use `get_thing` you get `stuff`, so far so good. But here is the problem you use it again in exactly the same way and yet you get `other stuff`. You kinda have to keep track of the last call to `set_thing` in order to know what you would get. We don't have the ability to predict the result from `get_thing`, you can't be 100% sure what it will do without looking around (or knowing) others parts of the code.

## Is there a better way?

I wouldn't say better. Let us just try lenses, you can decide later if you like them or not.

What do we need? Lenses are a functional programing thing so the first thing we will do is create helper functions. This will be the first version of getters and setters.

```javascript
// Getter
function prop(key) {
  return obj => obj[key];
}

// Setter
function assoc(key) {
  return (val, obj) => Object.assign({}, obj, {[key]: val});
}
```

Now for the "constructor."

```javascript
function Lens(getter, setter) {
  return { getter, setter };
}

// That is it.
```

You'll notice that `Lens` does absolutely nothing, I'm doing that on purpose. You can already tell that most of the work is in the getter and setter. Your lens are going to be as robust as your getter and setter implementations.

Now we need to make them do something, we will make three little functions. 

`view`: gets a value

```javascript
function view(lens, obj) {
   return lens.getter(obj);
}
```

`over`: transforms a value using a callback

```javascript
function over(lens, fn, obj) {
  return lens.setter(
    fn(lens.getter(obj)),
    obj
  );
}
```

`set`: replaces a value

```javascript
function always(val) {
  return () => val;
}

function set(lens, val, obj) {
  // don't you love reusability?
  return over(lens, always(val), obj);
}
```

It's time for a test drive.

Let's say we have an object named `alice`.

```javascript
const alice = {
  name: 'Alice Jones',
  address: ['22 Walnut St', 'San Francisco', 'CA'],
  pets: { dog: 'joker', cat: 'batman' }
};
```

We'll start with something simple, inspect the values. This is how you would do it.

```javascript
const result = view(
  Lens(prop('name'), assoc('name')),
  alice
);

result // => "Alice Jones"
```

I see you're not impressed and that's fine. I just wrote a lot characters just to get a name. But here is the thing, these are standalone functions. We can always compose and create new ones. Let's start with that `Lens(prop, assoc)` bit, we will put that in a function because we will use it a lot.

```javascript
function Lprop(key) {
  return Lens(prop(key), assoc(key));
}
```

And now...

```javascript
const result = view(Lprop('name'), alice);

result // => "Alice Jones"
```

You could even take it one step further and make a function that just expects the object that holds the data.

```javascript
const get_name = obj => view(Lprop('name'), obj);

// or with partial application

const get_name = view.bind(null, Lprop('name'));

// or using a curry utility.
// view = curry(view);

const get_name = view(Lprop('name'));

// and you can also do this with `set` and `over`
```

Enough of that. Going back to our test, let's try `over`. Let's transform the name to uppercase.

```javascript
const upper = str => str.toUpperCase();
const uppercase_alice = over(Lprop('name'), upper, alice);

// see?
get_name(uppercase_alice) // => "ALICE JONES"

// sanity check
get_name(alice)           // => "Alice Jones"
```

It's `set`'s turn.

```javascript
const alice_smith = set(Lprop('name'), 'Alice smith', alice);

get_name(alice_smith) // => "Alice smith"

// sanity check
get_name(alice)       // => "Alice Jones"
```

That's all nice but the name is just one property, what about nested object keys or arrays? Ah, you see now that is where it gets awkward with our current implementation. Right now you could do the following.

```javascript
let dog = Lens(
  obj => prop('dog')(prop('pets')(obj)),
  obj => assoc('dog')(assoc('pets')(obj))
);

view(dog, alice); // => "joker"

// or bring a `compose` utility

dog = Lens(
  compose(prop("dog"), prop("pets")),
  compose(assoc("dog"), assoc("pets"))
);

view(dog, alice); // => "joker"
```

I hear you. Don't worry, I wouldn't let you write stuff like that. It is because of situations like this one that people say stuff like "just use [Ramda](https://ramdajs.com/)" (and those people are right) But what makes ramda so special?

## Making it special

If you go to ramda's documentation and search "lens" you'll see that they have a `lensProp` function which is basically our `Lprop`. And if you go to the source you'll see something like this.

```javascript
function lensProp(k) {
  return lens(prop(k), assoc(k));
}
```

Look at that. But now the comments on their source and documentation suggest that it also works with just one property. Let's go back to our "lens" search on their site. Now we will check that curious `lensPath` function. It is exactly what we want. Once again we check out the source.

```javascript
function lensPath(p) {
  return lens(path(p), assocPath(p));
}

// Welcome to functional programming, y'all.
``` 

The secret sauce it's made of other functions that don't have any specific ties to lenses. Isn't that just nice?

What is in that `path` function? Let's check it out. I'll show you a slightly different version, but it works just the same.

```javascript
function path(keys, obj) {
  if (arguments.length === 1) {
    // this is for currying
    // they do this by wrapping `path`
    // with a helper function
    // but this is what happens
    // they return a function that remembers `keys`
    // and expects `obj`
    return path.bind(this, keys);
  }

  var result = obj;
  var idx = 0;
  while (idx < keys.length) {
    // we don't like null
    if (result == null) {
      return;
    }
    
    // this is how we get the nested keys
    result = result[keys[idx]];
    idx += 1;
  }

  return result;
}
```

I'll do the same with `assocPath`. For this one they make use of some internal helpers but again this is what happens.

```javascript
function assocPath(path, value, obj) {
  // again with the currying stuff
  // this is why they have a helper function
  if (arguments.length === 1) {
    return assocPath.bind(this, path);
  } else if (arguments.length === 2) {
    return assocPath.bind(this, path, value);
  }

  // check for an empty list
  if (path.length === 0) {
    return value;
  }

  var index = path[0];

  // Beware: recursion ahead.
  if (path.length > 1) {
    var is_empty =
      typeof obj !== 'object' || obj === null || !obj.hasOwnProperty(index);

    // if the current object is "empty"
    // we need to create a new one
    // otherwise we pick the object at `index`
    var next = is_empty
      ? typeof path[1] === 'number'
        ? []
        : {}
      : obj[index];

    // we start again the process
    // but now with a reduced `path`
    // and `next` as the new `obj`
    value = assocPath(Array.prototype.slice.call(path, 1), value, next);
  }

  // the base cases
  // we either have to copy an array
  // or an object
  if (typeof index === 'number' && Array.isArray(obj)) {
    // make a 'copy' of the array
    var arr = [].concat(obj);

    arr[index] = value;
    return arr;
  } else {
    // old school 'copy'
    var result = {};
    for (var p in obj) {
      result[p] = obj[p];
    }

    result[index] = value;
    return result;
  }
}
```

With our new found knowledge we can create an `Lpath` function and improve `Lprop`.

```javascript
function Lpath(keys) {
  return Lens(path(keys), assocPath(keys));
}

function Lprop(key) {
  return Lens(path([key]), assocPath([key]));
}
```

Now we can do more stuff, like playing with `alice` pets.

```javascript
const dog_lens = Lpath(['pets', 'dog']);

view(dog_lens, alice);     // => 'joker'

let new_alice = over(dog_lens, upper, alice);
view(dog_lens, new_alice); // => 'JOKER'

new_alice = set(dog_lens, 'Joker', alice);
view(dog_lens, new_alice); // => 'Joker'
```

All of this works great but there is just one tiny detail, the lenses that the current constructor creates aren't composable. Imagine that we have three lenses from differents files or something and we want to combine them like this.

```javascript
compose(pet_lens, imaginary_lens, dragon_lens);
```

This wouldn't work because `compose` expects a list of functions and our lenses are objects. But we can fix this (in a very funny way) with some functional programming trickery.

Let's start with our lenses constructor. Instead of returning an object we are going to return a "curried" function that takes a callback, an object and returns a **Functor** (a thing that has `map` method and follows [this rules](https://github.com/fantasyland/fantasy-land#functor))

```javascript
function Lens(getter, setter) {
  return fn => obj => {
    const apply = focus => setter(focus, obj);
    const functor = fn(getter(obj));
    return functor.map(apply);
  };
}
```

What's with the a `fn => obj =>` stuff? That is going to help us with our `compose` situation. Now after you provide the `getter` and `setter` you get a function, and that is what makes `compose` happy.

And `functor.map`? That is going to make sure that we can still use a lens as unit (like `Lprop('pets')`) but also a part of a chain using `compose`.

In case you are wondering what the good folks at ramda do different, they use their own bulletproof implementation of `map`.

Now we modify `view` and `over`. Starting with `view`.

```javascript
function view(lens, obj) {
  const constant = value => ({ value, map: () => constant(value) });
  return lens(constant)(obj).value;
}
```

That `constant` thing might look like is too much, but it does the job. Things can get crazy in those `compose` chains, that just makes sure the value you want stays safe.

What about `over`? It will do almost the same thing, except that in this case we do need to use the `setter` function.

```javascript
function over(lens, fn, obj) {
  const identity = value => ({ value, map: setter => identity(setter(value)) });
  const apply = val => identity(fn(val));
  return lens(apply)(obj).value;
}
```

And now we should have a mostly adequate `Lens` implementation. The whole thing without dependencies (`path` and `assocPath`) should look like this.

```javascript
function Lens(getter, setter) {
  return fn => obj => {
    const apply = focus => setter(focus, obj);
    const functor = fn(getter(obj));
    return functor.map(apply);
  };
}

function view(lens, obj) {
  const constant = value => ({ value, map: () => constant(value) });
  return lens(constant)(obj).value;
}

function over(lens, fn, obj) {
  const identity = value => ({ value, map: setter => identity(setter(value)) });
  const apply = val => identity(fn(val));
  return lens(apply)(obj).value;
}

function set(lens, val, obj) {
  return over(lens, always(val), obj);
}

function Lprop(key) {
  return Lens(path([key]), assocPath([key]));
}

function Lpath(keys) {
  return Lens(path(keys), assocPath(keys));
}

function always(val) {
  return () => val;
}
```

But can you believe me if I said it works? You shouldn't. Let's make some tests. We'll bring back `alice` and add her sister `calie`.

```javascript
const alice = {
  name: "Alice Jones",
  address: ["22 Walnut St", "San Francisco", "CA"],
  pets: { dog: "joker", cat: "batman", imaginary: { dragon: "harley" } }
};

const calie = {
  name: "calie Jones",
  address: ["22 Walnut St", "San Francisco", "CA"],
  pets: { dog: "riddler", cat: "ivy", imaginary: { dragon: "hush" } },
  friend: [alice]
};
```

And because we planned ahead we have some lenses already available.

```javascript
// some generic lens
const head_lens = Lprop(0);

// specific lens
const bff_lens = compose(Lprop('friend'), head_lens); 
const imaginary_lens = Lpath(['pets', 'imaginary']);
```

Say that we want to do something with their `dragons`, all we have to do is `compose`.

```javascript
const dragon_lens = compose(imaginary_lens, Lprop('dragon'));

// just for fun
const bff_dragon_lens = compose(bff_lens, dragon_lens); 

// demo
const upper = str => str.toUpperCase();

// view
view(dragon_lens, calie);         // => "hush"
view(bff_dragon_lens, calie);     // => "harley"

// over
let new_calie = over(dragon_lens, upper, calie);
view(dragon_lens, new_calie);     // => "HUSH"

new_calie = over(bff_dragon_lens, upper, calie);
view(bff_dragon_lens, new_calie); // => "HARLEY"

// set
new_calie = set(dragon_lens, 'fluffykins', calie);
view(dragon_lens, new_calie);     // => "fluffykins"

new_calie = set(bff_dragon_lens, 'pumpkin', calie);
view(bff_dragon_lens, new_calie); // => "pumpkin"
```

So we just manipulated a deeply nested object property by composing lenses. If you're not excited then I don't know what to tell you. We just solve a problem by composing functions! 

These things can be hard to sell because they require for you to write in a certain style in order to make the most out of it. And for people who write javascript there are libraries out there that solve the same problem in a more convenient way, or at least in a way that is more suitable for their style. 

Anyway, if you're still interested in seeing lenses in a non trivial context checkout [this repository](https://github.com/kwasniew/hyperapp-realworld-example-app), it is a [real world example app](https://github.com/gothinkster/realworld) (kinda like medium.com clone) that uses hyperapp to handle the frontend. In it the author chose to use lenses to handle state of the app.

## Sources

- [ramda - docs](https://ramdajs.com/docs/)
- [fp-lenses.js](https://gist.github.com/branneman/f06bd451f74e5bc1725db23be682d4fe)
- [Lambda World 2018 - Functional Lenses in JavaScript (video)](https://www.youtube.com/watch?v=IoVaArsh6tM)
