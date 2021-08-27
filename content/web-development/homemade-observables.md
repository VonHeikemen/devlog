+++
title = "Homemade observables"
description = "Making an observable from scratch (for educational purposes only)"
date = 2018-08-12
lang = "en"
[taxonomies]
tags = ["javascript", "learning" , "reactive-programming"]
[extra]
canonical_url = "https://dev.to/vonheikemen/homemade-observables-4ab3"
shared = [
  ["dev.to", "https://dev.to/vonheikemen/homemade-observables-4ab3"],
  ["Hashnode", "https://vonheikemen.hashnode.dev/homemade-observables"]
]
+++

On this episode we will build our own implementation of an observable. I hope that by the end of this post we gain a better understanding of this pattern that is used in libraries like RxJS.

## About Observables

### What is it?

Lets start with **my** definition of observable.

> An Observable is a function that follows a convention and is used to connect a data source with a consumer.

In our case a data source is something that produces values. And, a consumer is something that receives values from a data source.

### Fun facts

#### Observables are lazy

That means that they would not do any kind of work until it's absolutely necessary. Nothing will happen until you subscribe to them.

#### They can emit multiple values

Depending on the data source you can receive a finite number of values or an infinite stream of values.

#### They can be synchronous or asynchronous

It all depends on their internal implementation. You can setup an observable that process some stream of data in a synchronous way or create one from an event that can happen over time.

### Some rules

Remember when I said that observables follow a convention? Well, we are going to make our own arbitrary rules that our implementation will follow. These will be important because we are going to build a little ecosystem around our observables.

Here we go:

1. An observable instance will have a `subscribe` method.
2. The observable "factory" will take a `subscriber` function as a parameter.
3. The `subscriber` function will take an `observer` object as a parameter.
4. The `observer` object can implement these methods `next`, `error` and `complete`.

Now, lets do stuff.

### The code

#### Factory function

```javascript
function Observable(subscriber) {
  return {
    subscribe: observer => subscriber(observer)
  };
}

// I swear to you, this works.
```

That is less magical than I thought. What we see here is that the **Observable** factory is just a way to postpone the work that has to be done until you call subscribe. The `subscriber` function is doing the heavy lifting, that's good because we can do whatever we want in there, is what will make our observables useful.

So far I haven't done a really good job explaining the `observer` and the `subscriber` roles. I hope it'll become clear when you see them in action.

## A use case

Say that we want to convert an array into an Observable. How can we do this?
 
Lets think about what we know: 

* We can do all of our logic inside the `subscriber` function.
* We can expect an observer object with three methods, `next`, `error` and `complete`

We can use the methods of the observer object as communication channels. The `next` function will receive the values that our data source gives us. The `error` will handle any errors we throw at it, it will be like the `catch` function in the `Promise` class. And, we will use the `complete` method when the data source is done producing values.

Our array to observable function could look like this.

```javascript
function fromArray(arr) {
  return Observable(function(observer) {
    try {
      arr.forEach(value => observer.next(value));
      observer.complete();
    } catch (e) {
      observer.error(e);
    }
  });
}

// This is how we use it

var arrayStream = fromArray([1, 2, 3, 4]);

arrayStream.subscribe({
  next: value => console.log(value),
  error: err => console.error(err),
  complete: () => console.info('Nothing more to give')
});

// And now watch all the action on the console
```

### Be safe

Right now the observer object is basically a lawless town, we could do all sorts of weird stuff like sending yet another value to `next` even after we call the `complete` method. Ideally our observables should give us some guarantees, like:

* The methods on the observer object should be optional.
* The `complete` and `error` methods need to call the unsubscribe function (if there is one).
* If you unsubscribe, you can't call `next`, `complete` or `error`.
* If the `complete` or `error` method were called, no more values are emitted. 

### Interactive example

We can actually start doing some interesting things with what we learned so far. In this example I made a helper function that let me create an observable from a DOM event.

{{ codepen(id="wxNYPV", title="Observables - basics", load_js=true) }}

Now we will learn how we can manipulate existing Observables to extend their behavior.

## It all starts with operators

This time we'll create some utility functions, and tweak a little bit our current Observable implementation, in order to create more flexible features with them.  

Operators are functions that allow us to extend the behavior of an observable with a chain of functions. Each of this functions can take an observable as a data source and returns a new observable.

Lets keep the array theme in here and create a **map** operator that emulates the native map function of the Array prototype, but for observables. Our operator will do this: take a value, apply a function that will perform some transformation and return a new value.

Lets give it a try:

First step, get the transform function and the data source, then return a new observable that we can use.

```javascript
function map(transformFn, source$) {
  return Observable(function(observer) {
    // to be continued...
  });
}
```

Here comes the cool part, the source that we get is an observable and that means we can subscribe to it to get some values.

```javascript
function map(transformFn, source$) {
  return Observable(function(observer) {
    // remember to keep returning values from your functions.
    // This will return the unsubcribe function
    return source$.subscribe(function(value) {
      // to be continued...
    });
  });
}
```

Now we need to pass the result of the transformation to the observer so we can "see" it when we subscribe to this new observable.

```javascript
function map(transformFn, source$) {
  return Observable(function(observer) {
    return source$.subscribe(function(value) {
      // ****** WE ARE HERE ******
      var newValue = transformFn(value);
      observer.next(newValue);
      // *************************
    });
  });
}
```

There is a lot of indentation and returns going on in here. We can "fix" that if we use arrow functions all the way.

```javascript
function map(transformFn, source$) {
  return Observable(observer => 
    source$.subscribe(value => observer.next(
      transformFn(value)
    ))
  );
}

// that didn't do much for the indentation. 
// Well, you can't win them all.
```

We still need to use the operator and right now this will be it.

```javascript
function fromArray(arr) {
  return Observable(function(observer) {
    arr.forEach(value => observer.next(value));
    observer.complete();
  });
}

var thisArray = [1, 2, 3, 4];
var plusOne   = num => num + 1;
var array$    = map(plusOne, fromArray(thisArray));

array$.subscribe(value => console.log(value));
```

This doesn't feel very chainy. In order to use more of this map functions we would have to nest them, and that ain't right. Don't worry, we'll get to that in a moment.

## Pipe all the things

We will create a helper function that will allow us to use one or more operators that can modify an observable source. 

This function will take a collection of functions, and each function in the collection will use the return value of the previous function as an input.

First, I'm going to show how this could be done as a standalone helper function.

```javascript
function pipe(aFunctionArray, initialSource) {
  var reducerFn = function(source, fn) {
    var result = fn(source);
    return result;
  };

  var finalResult = aFunctionArray.reduce(reducerFn, initialSource);

  return finalResult;
}
```

In here the **reduce** function loops over the array and for each element in it executes **reducerFn**. Inside reducerFn in the first loop, **source** will be **initialSource** and in the rest of the loops **source** will be whatever you return from reducerFn. The **finalResult** is just the last result returned from reducerFn.

With some modifications (ES6+ goodness included) we can use this helper function within our Observable factory to make it more flexible. Our new factory would now look like this:

```javascript
function Observable (subscriber) {
  var observable = {
    subscribe: observer => subscriber(SafeObserver(observer)),
    pipe: function (...fns) {
      return fns.reduce((source, fn) => fn(source), observable);
    }
  }

  return observable; 
}
```

We need to do one more thing to make sure our operators are compatible with this new pipe function. For example, our current **map** operator expects both **transformFn** and **source** at the same time. That just won't happen inside pipe. Will have to split it into two functions, one that will take the initial necessary parameters to make it work and another one that takes the source observable.

There are a couple of ways we can do this.

```javascript
// Option 1
function map(transformFn) {
  // Instead of returning an observable 
  // we return a function that expects a source
  return source$ => Observable(observer => 
    source$.subscribe(value => observer.next(
      transformFn(value)
    ))
  );
}

// Option 2
function map(transformFn, source$) {
  if(source$ === undefined) {
    // we'll return a function 
    // that will "remember" the transform function
    // and expect the source and put in its place.
    
    return placeholder => map(transformFn, placeholder);
  }

  return Observable(observer => 
    source$.subscribe(value => observer.next(
      transformFn(value)
    ))
  );
}
```

And finally we can extend our observable in this way:

```javascript
var thisArray = [1, 2, 3, 4];
var plusOne   = num => num + 1;
var timesTwo  = num => num * 2;

var array$ = fromArray(thisArray).pipe(
  map(plusOne),
  map(timesTwo),
  map(num => `number: ${num}`),
  // ... many more operators
);

array$.subscribe(value => console.log(value));
```

Now we are ready to create more operators.

## Exercise time

Lets say that we have a piece of code that prints a "time string" to the console every second, and stops after five seconds (because why not). This guy right here:

```javascript
function startTimer() {
  var time = 0;
  var interval = setInterval(function() {
    time = time + 1;

    var minutes = Math.floor((time / 60) % 60).toString().padStart(2, '0');
    var seconds = Math.floor(time % 60).toString().padStart(2, '0');
    var timeString = minutes + ':' + seconds;

    console.log(timeString);

    if(timeString === '00:05') {
      clearInterval(interval);
    }
  }, 1000);
}
```

There is nothing wrong with this piece of code. I mean, it does the job, it's predictable, and everything you need to know about it is there in plain sight. But you know, we are in a refactoring mood and we just learned something new. We'll turn this into an observable thingy.

First things first, lets make a couple of helper function that handle the formatting and time calculations.

```javascript
function paddedNumber(num) {
  return num.toString().padStart(2, '0');
}

function readableTime(time) {
  var minutes = Math.floor((time / 60) % 60);
  var seconds = Math.floor(time % 60);
 
  return paddedNumber(minutes) + ':' + paddedNumber(seconds);
}
```

Now lets handle the time. _setInterval_ is a great candidate for a data source, it takes a callback in which we could produce values, it also has a "cleanup" mechanism. It just makes the perfect observable.

```javascript
function interval(delay) {
  return Observable(function(observer) {
    var counter   = 0;
    var callback  = () => observer.next(counter++);
    var _interval = setInterval(callback, delay);
    
    observer.setUnsubscribe(() => clearInterval(_interval));
    
    return observer.unsubscribe;
  });
}
```

This is amazing, we now have really reusable way to set and destroy an interval.

You may have notice that we are passing a number to the observer, we are not calling it _seconds_ because the **delay** can be any arbitrary number. In here we're not keeping track of the time, we are merely counting how many times the callback has been executed. Why? Because we want to make every observable factory as generic as possible. We can always modify the value that it emits by using operators.

This how we could use our new interval function.

```javascript
// pretend we have our helper functions in scope.

var time$ = interval(1000).pipe(
  map(plusOne),
  map(readableTime)
);

var unsubscribe = time$.subscribe(function(timeString) {
  console.log(timeString);
  
  if(timeString === '00:05') {
    unsubscribe();
  }
});
```

That's better. But that _if_ bothers me. I feel like that behavior doesn't belong in there. You know what? I'll make an operator that can unsubscribe to the interval after it emits five values.

```javascript
// I'll named "take" because naming is hard.
// Also, that is how is called in other libraries.

function take(total) {
  return source$ => Observable(function(observer) {
    // we'll have our own counter because I don't trust in the values
    // that other observables emits
    var count = 0;
    var unsubscribeSource = source$.subscribe(function(value) {
      count++;
      // we pass every single value to the observer.
      // the subscribe function will still get every value in the stream 
      observer.next(value);
      
      if (count === total) {
        // we signal the completion of the stream and "destroy" the thing
        observer.complete();
        unsubscribeSource();
      }
    });
  });
}
``` 

Now we can have a self destructing timer. Finally.

```javascript
// pretend we have our helper functions in scope.

var time$ = interval(1000).pipe(
  map(plusOne),
  map(readableTime),
  take(5)
);

time$.subscribe({
  next: timeString => console.log(timeString),
  complete: () => console.info("Time's up")
});
``` 

## Playgrounds

I made a couple of pens so you can play around with this stuff. [This pen](https://codepen.io/VonHeikemen/pen/OwQYxG) contains all the Observable related code that I wrote for this posts and them some more.

And this is the pen for the exercise.

{{ codepen(id="VGZXZa", title="Observables - boring timer") }}

## Conclusion

Observables are a powerful thing, with a little bit of creativity you can turn anything you want into an observable. Really. A promise, an AJAX request, a DOM event, an array, a time interval and anything you can imagine can be a source of data that can be wrapped in an observable.

They are a powerful abstraction. They can let you process streams of data one chunk at a time. Not only that, but also let you piece together solutions that can be compose by generic functions and custom functions specific to the problem at hand. 

Fair warning though. They are not the ultimate solution to every problem. You'll have to decide if the complexity is worth it. Like in the exercise, we lose the simplicity of the _startTimer_ in order to gain some flexibility (that we could've achieve some other way).

## Sources
 * [Learning Observable By Building Observable](https://medium.com/@benlesh/learning-observable-by-building-observable-d5da57405d87)
 * [Observables, just powerful functions?](https://medium.com/@kevinkreuzer/observables-just-powerful-functions-a033c355b22c)
 * [Who’s Afraid of Observables?](https://netbasal.com/whos-afraid-of-observables-bde0dc4f48cc)
 * [Understanding mergeMap and switchMap in RxJS](https://netbasal.com/understanding-mergemap-and-switchmap-in-rxjs-13cf9c57c885)
 * [JavaScript — Observables Under The Hood](https://netbasal.com/javascript-observables-under-the-hood-2423f760584)
 * [Github repository - zen-observable](https://github.com/zenparsing/zen-observable)
 * [Understanding Observables](https://dev.to/supermanitu/understanding-observables)

