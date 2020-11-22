+++
title = "The case for reducers"
description = "Reducers are good for many things, let's find out what those things are"
date = 2020-11-22
lang = "en"
[taxonomies]
tags = ["javascript"]
+++

In a [previous post](@/web-development/learn-fp/reduce-how-and-when.md) I talked about `.reduce`, how it worked and (what I think) it's ideal use case, this time around I'll cover some other use cases where `.reduce` could be a good fit. Now, you don't have to read that post but I will assume that you at least know how `Array.reduce` works. By the end of this post I hope that you learn how to recognize the places where `.reduce` would work perfectly.

## What are we looking for?

Patterns, we are looking for patterns. Well... just one. And to know what is it that we are looking for we need to take a look at the requirements of a `reducer`. Think about `reducers`, when you create one for `Array.reduce` sometimes it looks like this.

```js
function (accumulator, value) {
  /*
    some logic
  */
  return accumulator;
}
```

We usually return a modified copy of `accumulator` but that's not important right now, the point is that we return the same "type" we got in the first parameter. Then the **shape of the function** would be something like this.

```
(Accumulator, Value) -> Accumulator
```

This is a concrete example but I want you to see it in a more abstract way. What we are really after are functions that have this shape.

```
(A, B) -> A
```

This is basically it. For a `reducer` to do it's job the only thing it needs is a binary function capable of returning the same type of its first parameter.

Still confused? Don't worry I'll spend the rest of this post showing examples where this pattern might show up.

## Use cases

### Accumulators

I guess this is the part where I show you a scenario where we sum an array of numbers of something like that. Let's not do that. Let's try a more complex scenario where an accumulator might be used.

Imagine we are in a codebase for some kind of blog system and we are making the profile page for the user. We want to show all the tags where the user has at least one article. You might want to retrieve that data from your database using a crazy query but that would take too much time, let's do a prototype first.

So before we do things the appropriate way we transform the array of posts into a [Set](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Set) of tags using `Array.reduce`, just to have something to work with.

```js
// Pretend these are complex objects
const posts = [
  { tags: ["javascript", "discuss"] },
  { tags: ["javascript", "react", "vue-is-better"] },
  { tags: ["discuss"] },
  { tags: ["javascript"] },
];

function dangerously_add_tags(acc, post) {
  for(let value of post.tags) {
    acc.add(value);
  }

  return acc;
}

posts.reduce(dangerously_add_tags, new Set());
```

This is the result.

```
Set(4) [ "javascript", "discuss", "react", "vue-is-better" ]
```

Think about the shape of our reducer. We have a `Set` with tags as our accumulator and our `value` is a "post object". We could say we have this.

```
(Set, Object) -> Set
```

Technically `Object` can't be any object, it has to have a `tags` property. So is more like.

```
(Set, Post) -> Set
```

Anyway, this has the pattern I was talking about `(A, B) -> A`. The implementation of `dangerously_add_tags` demands that `B` must be of type `Post`. But in order for that function to be an effective `reducer` it needs to be able to return the same type of the first parameter, and we do that by returning `accumulator`.

### Transformations

You've probably heard that you can implement other array methods using `.reduce`, while this is an interesting piece of trivia it's not very useful to do so. Why would you? Doesn't make any sense to me. What is useful about it is that you can combine the features of this methods into one. Have you ever wanted to filter and map at the same time? With `.reduce` you can.

Let's reuse our `posts` data here too.

```js
const posts = [
  {
    category: "javascript",
    tags: ["javascript", "discuss"]
  },
  {
    category: "frameworks",
    tags: ["javascript", "react", "vue-is-better"]
  },
  {
    category: "watercooler",
    tags: ["discuss"]
  },
  {
    category: "functional programming",
    tags: ["javascript"]
  },
];
```

What want to do this time is filter the ones that have the tag `discuss`, for those who pass the filter we want to get the category and capitalize it. How would that look like?

```js
function capitalize(str) {
  return str[0].toUpperCase() + str.slice(1);
}

function filter_map_posts(acc, post) {
  // We're filtering, y'all
  if(post.tags.includes('discuss')) {
    return acc.concat(
      // this is the mapping part
      capitalize(post.category)
    );
  }
	
  return acc;
}

posts.reduce(filter_map_posts, []);
```

Here is our result.

```
Array [ "Javascript", "Watercooler" ]
```

Why does that work? Because if you check what the `reducer` does you would get this.

```
(Array, Post) -> Array
```

### Coordinating

If you've seen any library that has a focus on functional programming chances are you've come across a function called `pipe`. This function is used to compose any arbitrary quantity of functions. The interface is something like this.

```js
pipe(
  some_function,
  another,
  serious_stuff,
  side_effects_ahead,
);
```

The idea here is that we "pipe" the result of one function to the next one in the list. Is effectively coordinating function calls. In this case the example above could be written like this.

```js
function pipe(arg) {
  return side_effects_ahead(serious_stuff(another(some_function(arg))));
}
```

If you're wondering why do I bring this up, is because we can implement `pipe` using `.reduce`. If you squint your eyes a little bit you'll notice that what is happening in here is that we are applying functions to arguments. That's it. We are not doing anything else.

So what?

It's a binary operation! We turn that into a function.

```js
function apply(arg, fn) {
  return fn(arg);
}
```

You know what works well with binary operations? Our friend `.reduce`.

```js
function pipe(...fns) {
  return function(some_arg) {
    return fns.reduce(apply, some_arg);
  };
}
```

The first step of `pipe` is gathering the list of functions and turn that into a proper array. Step two is returning the function that will trigger the function calls and get the initial state for our `.reduce`. At the end when you have everything in place, `.reduce` will take care of the rest. You can watch it in action.

```js
const post = { 
  category: "javascript",
  tags: ["javascript", "discuss"] 
}

function capitalize(str) {
  return str[0].toUpperCase() + str.slice(1);
}

function get_prop(key) {
  return function(obj) {
    return obj[key];
  }
}

function exclaim(str) {
  return str + "!!";
}

const exciting_category = pipe(
  get_prop("category"),
  capitalize,
  exclaim
);

exciting_category(post);
// => Javascript!!
```

Cool, cool. Now, how on earth does `apply` follow the pattern?

Ah, good question. It's weird but we can still make it make sense (I guess). Look at it this way.

```
(Anything, Function) -> Anything
```

If you have a unit of literally anything and a function, `apply` will work. Keep in mind that in here there is no guarantee that your pipeline of function will not explode, that's your responsibility.

### State changes over time

Bonus track!! This is for the frontend developers out there.

If you have spend any amount of time reading about javascript libraries for state management maybe you've have heard of this thing called [redux](https://redux.js.org/). This library takes an interesting approach because it expects the user (the developer) to provide a `reducer` to handle state changes. Some people like that, others don't like it. But whether you're team redux or not, they're approach does make a ton of sense when you think about it. I'll show you.

Let's start with the `reducer`. In this case we need one with this shape.

```
(State, Action) -> State
```

`State` and `Action` are just objects. There is nothing fancy happening. The `State` will look different depending on the application, the developers can do anything they want with it. The `Action` on the other hand must have a `type` property, and `redux` enforces this.

Let's pretend this is our app's state.

```js
const state = {
  count: 40,
  flag: false
};
```

Yes, a miracle of engineering.

Now that we now how `State` looks like, and we also know what an `Action` needs, we can write our `reducer`.

```js
function reducer(state, action) {
  switch(action.type) {
    case 'add':
      return {
        ...state,
        count: state.count + 1,
      };
    case 'subtract':
      return {
        ...state,
        count: state.count - 1,
      };
    case 'toggle_flag':
      return {
        ...state,
        flag: !state.flag,
      };
    default:
      return state;
  }
}
```

This is the funny part: we don't need `redux` to test this. I mean, this is just a generic `reducer`, we could just try it with `Array.reduce` first. If you do this you can see what it does right away.

```js
const actions = [
  { type: 'add' },
  { type: 'add' },
  { type: 'subtract' },
  { type: 'add' },
  { type: 'subtract' },
  { type: 'add' },
  { type: 'toggle_flag' }
];

actions.reduce(reducer, state);
```

`actions.reduce` should give you another "instance" of your state. In our case after applying all those actions we should get this.

```js
{
  count: 42,
  flag: true
}
```

And there you have it, the core feature of `redux` without `redux`.

Let's take it one step further and introduce the concept of time. For this we will introduce a fake `redux` store. The store will be "real" but it'll be a cheap imitation. Let's do this.

```js
function Store(reducer, state) {
  let _listener = null;

  const get_state = function() {
    return state;
  };

  const subscribe = function(listener) {
    _listener = listener;
  };

  const dispatch = function(action) {
    state = reducer(state, action);
    _listener && _listener();

    return action;
  };

  return { get_state, dispatch, subscribe };
}
```

All good? You know what's happening in there? The part we care about the most is `dispatch`. This right here.

```js
const dispatch = function(action) {
  state = reducer(state, action);
  _listener && _listener();

  return action;
};
```

This takes care of the process of updating the current `State`. Like I mentioned before, the `reducer` is the one that deals with the logic that dictates **how** the state will change. The `Store` takes care of logic that dictates **when** the state is updated. Enough about that, let's try it.

```js
function delay(ms) {
  return new Promise(resolve => setTimeout(resolve, ms));
}

const store = Store(reducer, state);
store.subscribe(function() {
  console.log(store.get_state());
});

(async function() {
  store.dispatch({ type: 'add' });
  await delay(500);

  store.dispatch({ type: 'add' });
  await delay(500);

  store.dispatch({ type: 'subtract' });
  await delay(700);

  store.dispatch({ type: 'add' });
  await delay(400);

  store.dispatch({ type: 'subtract' });
  await delay(800);

  store.dispatch({ type: 'add' });
  await delay(100);

  store.dispatch({ type: 'toggle_flag' });
})();
```

You should have this messages on your screen (or browser console) with a little delay between each of them.

```
- { count: 41, flag: false }
- { count: 42, flag: false }
- { count: 41, flag: false }
- { count: 42, flag: false }
- { count: 41, flag: false }
- { count: 42, flag: false }
- { count: 42, flag: true }
```

Did you notice that the end result is the same as with `Array.reduce`? Now that's cool.

If you want to play around with this using the real `redux`, you can mess around with this pen.

{{ codepen(id="zYBgObJ", title="Using redux", load_js=true) }}

## Conclusion

I do hope by now `reducers` appear less scary for you. Remember, it's just.

```
(A, B) -> A
```

That's it. There is no magic. If you can make any function behave like that, it'll work wonderfully inside anything that acts like `.reduce`.

## Sources
- [Array.prototype.reduce()](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/Reduce)
- [Reduce: how and when](@/web-development/learn-fp/reduce-how-and-when.md)
- [Redux: Store](https://redux.js.org/api/store)

