+++
title = "Taking a look at finite state machines"
description = "Let's see if finite state machines are useful"
date = 2019-11-07
lang = "en"
[taxonomies]
tags = ["javascript", "learning", "beginners"]
[extra]
canonical_url = "https://dev.to/vonheikemen/taking-a-look-at-finite-state-machines-2g9i"
+++

## The finite who-- what?

It is a way of modeling the behavior of a system. The idea is that your "system" can only be in one state at any given time, and an input (or event) can trigger the transition to another state.

## What kind of problems does it solve?

Invalid state. How many times have you used a flag or attribute like "disabled" to prevent a user from doing something they shouldn't do? By setting the rules of our system we can avoid these kind of problems.

## How does that look like in javascript?

I'm very glad you asked. The real reason I'm writing this is to show you a library that I saw the other day. We are going to use [robot3](https://thisrobot.life/) to built a random quote machine.

We will make a card that displays a quote and below that we'll have a button that will fetch another quote.

We'll do it one step at a time. Let's first prepare the states. 

Our card will be either `idle` or `loading`. Create a machine with that.

```js
import {
  createMachine,
  state,
  interpret
} from 'https://unpkg.com/robot3@0.2.9/machine.js';

const mr_robot = createMachine({
  idle: state(),
  loading: state()
});
```

In here each `state` is a key in the "setup object" that we pass to `createMachine`, but also notice that it needs to be a `state` object, which we create with the `state` function.

Now we need transitions. Our `idle` state will switch to `loading` if a `fetch` event happens, `loading` will go back to `idle` if a `done` is dispatched. 

```diff
 import {
  createMachine,
  state,
+ transition,
  interpret
 } from 'https://unpkg.com/robot3@0.2.9/machine.js';

const mr_robot = createMachine({
-  idle: state(),
-  loading: state()
+  idle: state(transition('fetch', 'loading')),
+  loading: state(transition('done', 'idle'))
 });
```

`transition` is the thing that connects our states. It's first parameter is the name of the event that will trigger the transition, the second parameter is the "destination" state it will switch to. The rest of `transition`'s parameters can be a list of function that will be executed when this transition is triggered.

Looks lovely, but uhm... how do we test it? The machine by itself doesn't do anything. We need to give our new machine to the `interpret` function which will give us a "service" that can dispatch events. To prove that we are actually doing something we'll also give a handler to `interpret`, it will be like a 'onchange', it will listen to state changes.

```js
const handler = ({ machine }) => {
  console.log(machine.current);
}

const { send } = interpret(mr_robot, handler);
```

Now you can see if it's alive.

```js
send('fetch');
send('fetch');
send('fetch');
send('done');

// You should see in the console
// loading (3)
// idle
```

Dispatching `fetch` will turn the current state to `loading` and `done` will get it back to `idle`. I see you're not impressed. That's fine. Let's try something, let's add another state `end` and make `loading` switch to that, then dispatch `done` and see what happens. 

```diff
 const mr_robot = createMachine({
   idle: state(transition('fetch', 'loading')),
-   loading: state(transition('done', 'idle'))
+   loading: state(transition('done', 'end')),
+   end: state()
 });
```
```js
send('done');

// You should see in the console
// idle
```

Sending `done` while `idle` doesn't trigger a `loading` state, it stays in `idle` because that state doesn't have a `done` event. And now...

```js
// We do the usual flow.

send('fetch');
send('done');

// You should have
// loading
// end

// Now try again `fetch`
send('fetch');

// You should have
// end
```

If you send `fetch` (or any other event) while in `end` state will give you `end` every single time. Why? Because you can't go anywhere, `end` doesn't have transitions.

I hope you see why this is useful. If not, I apologize for all the `console.log`ing.

Going back to our current machine. This what we got so far.

```js
 import {
  createMachine,
  state,
  transition,
  interpret
} from 'https://unpkg.com/robot3@0.2.9/machine.js';

const mr_robot = createMachine({
  idle: state(transition('fetch', 'loading')),
  loading: state(transition('done', 'idle'))
});

const handler = ({ machine }) => {
  console.log(machine.current);
}

const { send } = interpret(mr_robot, handler);
```

But this is still not enough, now we need to get some data when we enter the `loading` state. Let's first fake our quote fetching function.

```js
function get_quote() {
  // make a random delay, 3 to 5 seconds.
  const delay = random_number(3, 5) * 1000;

  const promise = new Promise(res => {
    setTimeout(() => res('<quote>'), delay);
  });
  
  // sanity check
  promise.then(res => (console.log(res), res));

  return promise;
}
```

To make it work with our state machine we will use a function called `invoke`, this utility calls an "async function" (a function that returns a promise) when you enter a `state` then when the promise resolves it sends a `done` event (if it fails it sends a `error` event).

```diff
  import {
   createMachine,
   state,
+  invoke,
   transition,
   interpret
 } from 'https://unpkg.com/robot3@0.2.9/machine.js';

 const mr_robot = createMachine({
   idle: state(transition('fetch', 'loading')),
-  loading: state(transition('done', 'idle')),
+  loading: invoke(get_quote, transition('done', 'idle')),
 });
```

If you test `send('fetch')` you should see in the console.

```
loading

// wait a few seconds...

<quote>
idle
```

By now I hope you're all wondering where do we actually keep the data? There is a handy feature in `createMachine` that let us define a "context" object that will be available to us in the function that we attach to our `transitions`.

```js
const context = ev => ({
  data: {},
});
```

```diff
  const mr_robot = createMachine({
    idle: state(transition('fetch', 'loading')),
    loading: invoke(get_quote, transition('done', 'idle')),
- });
+ }, context);
```

Next we'll use another utility. We will pass a third parameter to `loading`'s transition, a hook of some sort that will modify the context object. This utility is called `reduce` and it looks like this.

```js
reduce((ctx, ev) => ({ ...ctx, data: ev.data }))
```

It takes the current context, a payload (here named `ev`) and whatever you return from it becomes your new context. We add that to the `loading` state.

```diff
  import {
   createMachine,
   state,
   invoke,
   transition,
+  reduce,
   interpret
 } from 'https://unpkg.com/robot3@0.2.9/machine.js';

 const mr_robot = createMachine({
   idle: state(transition('fetch', 'loading')),
-  loading: invoke(get_quote, transition('done', 'idle')), 
+  loading: invoke(
+    get_quote, 
+    transition(
+      'done',
+      'idle',
+      reduce((ctx, ev) => ({ ...ctx, data: ev.data }))
+    )
+  ),
 }, context);
```

Sanity check time. How do we know that works? We modify `interpret`'s handler.

```js
const handler = ({ machine, context }) => {
  console.log(JSON.stringify({ 
    state: machine.current,
    context
  }));
}
```

You should see this.

```
{'state':'loading','context':{'data':{}}}

// wait a few seconds...

{'state':'idle','context':{'data':'<quote>'}}
```

We are ready. Let's show something in the browser.

```html
<main id="app" class="card">
  <section id="card" class="card__content">
     <div class="card__body">
        <div class="card__quote">
          quote
        </div>

        <div class="card__author">
          -- author
        </div>
      </div>
      <div class="card__footer">
        <button id="load_btn" class="btn btn--new">
          More
        </button>
        <a href="#" target="_blank" class="btn btn--tweet">
          Tweet
        </a>
      </div> 
  </section> 
</main>
```

```css
body {
  display: flex;
  justify-content: center;
  align-items: center;
  min-height: 95vh;
  background: #ddd;
  font-size: 1em;
  color: #212121;
}

.card {
  width: 600px;
  background: white;
  box-shadow: 0 2px 5px 0 rgba(0, 0, 0, 0.16), 0 2px 10px 0 rgba(0, 0, 0, 0.12);
}

.card__content {
  color: #212121;
  padding: 20px;
}

.card__content--loader {
  height: 95px;
  display: flex;
  align-items: center;
  justify-content: center
}

.card__body {
 padding-bottom: 15px;
}

.card__author {
  padding-top: 10px;
  font-style: italic;
}

.card__footer {
  width: 100%;
  display: flex;
  justify-content: space-between;
}

.btn {
  color: #fff;
  cursor: pointer;
  margin-top: 10px;
  margin-left: 10px;
  border-radius: 0.4rem;
  text-decoration: none;
  display: inline-block;
  padding: .3rem .9rem;
}

.btn--new {
  background-color: #2093be;
  border: 0.1rem solid #2093be;
  
}

.btn--tweet {
  background-color: #0074d9;
  border: 0.1rem solid #0074d9;
}

.btn:hover {
  background: #3cb0fd;
  border: 0.1rem solid #3cb0fd;
  text-decoration: none;
}

.hide {
  display: none;
}
```

Now the last piece of the puzzle, the side effects. We need to attach another function to our transitions so we can update the DOM. We could use `reduce` again but is just rude to have side effects on something called `reduce` (just don't) We will bring another utility made for that, `action`.

But first we must prepare. Update the context object with the necessary dependencies. (This step is not necessary, this is just me being allergic to global variables)

```diff
 const context = ev => ({
   data: {},
+  dom: {
+    quote: document.querySelector('.card__quote'),
+    author: document.querySelector('.card__author'),
+    load_btn: window.load_btn,
+    tweet_btn: document.querySelector('.btn--tweet'),
+    card: window.card
+  }
 });
```

Create the side effects. At this point you should make sure that `get_quote` actually returns an object with a `quote` and `author` property.

```js
function update_card({ dom, data }) {
  dom.load_btn.textContent = 'More';
  dom.quote.textContent = data.quote;
  dom.author.textContent = data.author;

  const web_intent = 'https://twitter.com/intent/tweet?text=';
  const tweet = `${data.quote} -- ${data.author}`;
  dom.tweet_btn.setAttribute(
    'href', web_intent + encodeURIComponent(tweet)
  );
}

function show_loading({ dom }) {
  dom.load_btn.textContent = 'Loading...';
}
```

Put everything together.

```diff
  import {
   createMachine,
   state,
   invoke,
   transition,
   reduce,
+  action,
   interpret
 } from 'https://unpkg.com/robot3@0.2.9/machine.js';

 const mr_robot = createMachine({
-  idle: state(transition('fetch', 'loading')),
+  idle: state(transition('fetch', 'loading', action(show_loading))),
   loading: invoke(
     get_quote, 
     transition(
       'done',
       'idle',
       reduce((ctx, ev) => ({ ...ctx, data: ev.data })),
+      action(update_card)
     )
   ),
 }, context);
```

By now everything kinda works but it looks bad when it loads for the first time. Let's make another loader, one that hides the card while we fetch the first quote.

Let's start with the HTML.

```diff
 <main id="app" class="card">
-  <section id="card" class="card__content">
+  <section class="card__content card__content--loader"> 
+    <p>Loading</p> 
+  </section>
+  <section id="card" class="hide card__content">
     <div class="card__body">
       <div class="card__quote">
         quote
       </div>

       <div class="card__author">
          -- author
       </div>
     </div>
     <div class="card__footer">
       <button id="load_btn" class="btn btn--new">
         More
       </button>
       <a href="#" target="_blank" class="btn btn--tweet">
         Tweet
       </a>
     </div> 
   </section> 
 </main>
```

We'll make another state, `empty`. We can reuse our original `loading` state for this. Make a factory function that returns the loading transition.

```js
const load_quote = (...args) =>
  invoke(
    get_quote,
    transition(
      'done',
      'idle',
      reduce((ctx, ev) => ({ ...ctx, data: ev.data })),
      ...args
    ),
    transition('error', 'idle')
  );
```
```diff
 const mr_robot = createMachine({
   idle: state(transition('fetch', 'loading', action(show_loading))),
-  loading: invoke(
-    get_quote, 
-    transition(
-      'done',
-      'idle',
-      reduce((ctx, ev) => ({ ...ctx, data: ev.data })),
-      action(update_card)
-    )
-  ),
+  loading: load_quote(action(update_card))
 }, context);
```

Now we use this to hide the first loader and show the quote when it's ready.

```diff
 const context = ev => ({
   data: {},
   dom: {
     quote: document.querySelector('.card__quote'),
     author: document.querySelector('.card__author'),
+    loader: document.querySelector('.card__content--loader'),
     load_btn: window.load_btn,
     tweet_btn: document.querySelector('.btn--tweet'),
     card: window.card
   }
 });
```

```js
function hide_loader({ dom }) {
  dom.loader.classList.add('hide');
  dom.card.classList.remove('hide');
}
```

```diff
 const mr_robot = createMachine({
+  empty: load_quote(action(update_card), action(hide_loader)),
   idle: state(transition('fetch', 'loading', action(show_loading))),
   loading: load_quote(action(update_card))
 }, context);
-
- const handler = ({ machine, context }) => {
-  console.log(JSON.stringify({ 
-    state: machine.current,
-    context
-  }));
- }
+ const handler = () => {};

 const { send } = interpret(mr_robot, handler);
+
+ const fetch_quote = () => send('fetch');
+
+ window.load_btn.addEventListener('click', fetch_quote);
```

Let's see it work.

{{ codepen(id="OJJvQzR", title="Finite Random Quote Machine", load_js=true) }}

## So is this state machine thing helpful?

I hope so. Did you notice we made a bunch of test and created the blueprint of the quote machine even before writing any HTML? I think that's cool. 

Did you try to click the 'loading' button while loading? Did it triggered a bunch of call to `get_quote`? That is because we made (sort of) impossible that a `fetch` event can happen during `loading`. 

Not only that, the behavior of the machine and the effects on the outside world are separated. Depending on how you like to write code that may be a good or a bad thing.

## Want to know more?
* [XState (concepts)](https://xstate.js.org/docs/about/concepts.html)
* [robot3 - docs](https://thisrobot.life/)
* [Understanding State Machines](https://www.freecodecamp.org/news/state-machines-basics-of-computer-science-d42855debc66/)
