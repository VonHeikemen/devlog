+++
title = "Pure functions and why they are a good idea" 
description = "Answering the question what do we gain by using pure function?"
date = 2020-04-02
lang = "en"
[taxonomies]
tags = ["javascript", "functional-programming", "learning"]
+++

When we talk about functional programming very few things can be as important as pure functions. People who write code in this style make a considerable effort to contain as much logic as they can in pure functions, I'll try to explain some of the reasons behind this. But, first things first...

## What's a pure function?

A function whose output is only determined by its input and has no observable effect on the outside world (has no side effects).

### Benefits

I want to focus on the benefits they provide to us humans who read and interpret code in our minds.

- They are predictable

Given the same inputs, it always produces the same output. This is one of the most relevant properties they have, and to me is the most important. It gives us the ability to test with relative ease how effective is our solution. 

Say that we have a function that transforms every letter in a string to uppercase. What do we need to prove that it works? The function, some input data and the expected output.

```js
to_uppercase('hello') == 'HELLO';
```

There is no need to emulate an entire environment or to use special tools, we just compare the result with the expected output. This gives us confidence in what we create, because we can prove with certainty that it works appropriately.

- Comprehension

When it comes to code we spend more time reading and analyzing than writing it. Communication is one aspect that we always need to consider. In theory, a pure function would need the least amount of context in order to understand its behavior because everything you need to know about it is (or at least it should) in the body and its arguments.

Another property that pure function have is **referential transparency**, this means that we can replace a function call with its return value.

For example, this.

```js
to_uppercase('hi') + ', user';
```

Can be replaced by this.

```js
'HI, user';
```

It means that when you understand what a pure function does you can mentally replace the function call with the return value.

- Composition

If you have created a pure function there is a very strong chance that what you have is an independent component that can be reused in different contexts. Being independent and reusable makes them the perfect candidates to be combined with other components. Think about it, if you combine a pure function with another into a new function, it results in yet another pure function. This is one powerful feature allows you to create complex procedures by composing "simple" pieces.

## Further reading

As good as pure functions are at some point you need to abandon the idea of purity and cause some effect on the outside world (show something on the screen, make an HTTP request, etc..) Because of that I have prepared more material with more details about this topic.

- [Composition techniques](@/web-development/learn-fp/composition-techniques.md)

- [Dealing with side effects and pure functions in javascript ](@/web-development/learn-fp/dealing-with-side-effects-and-pure-functions.md)

## Sources

- [Functional Programming in JS: What? Why? How? (video)](https://www.youtube.com/watch?v=qtsbZarFzm8&feature=youtu.be)
- [An introduction to functional programming](https://codewords.recurse.com/issues/one/an-introduction-to-functional-programming)
- [Functional-Light JavaScript - Chapter 5: Reducing Side Effects](https://github.com/getify/Functional-Light-JS/blob/master/manuscript/ch5.md/#chapter-5-reducing-side-effects)
