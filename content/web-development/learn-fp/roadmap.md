+++
title = "Learning functional programming: A roadmap" 
description = "Table of content for the series: Functional programming for your everyday javascript"
date = 2020-04-03
lang = "en"
[taxonomies]
tags = ["javascript", "functional-programming", "learning"]
[extra]
shared = [
  ["dev.to", "https://dev.to/vonheikemen/what-i-know-about-functional-programming-38mc"],
  ["Hashnode", "https://vonheikemen.hashnode.dev/learn-fp-in-js"]
]
+++

Learning about functional programming is not an easy task, specially if you search for articles that have concrete examples of the concepts they try to teach. I have been learning about this paradigm for a while and I want it to share the notes I have taken, the ones I've turn into articles, and also the source material where I got the information.

Even though all these articles are related I didn't plan on writing them. So, I'll be presenting some sort of guide (a suggestion) on the order they should be read.

## The basics

To begin I would like you to see the video of the talk that convinced me to try learning this paradigm. The talk is about what is and what isn't functional programming, it also shows some examples of the core principles using javascript.

{{ youtube(id="qtsbZarFzm8") }}

To complement that video I wrote my own notes.

- [Pure functions and why they are a good idea](@/web-development/learn-fp/pure-functions.md)

- [Dealing with side effects and pure functions in javascript](@/web-development/learn-fp/dealing-with-side-effects-and-pure-functions.md)

### Further reading

- [An introduction to functional programming](https://codewords.recurse.com/issues/one/an-introduction-to-functional-programming)

## A very special tool

If you read everything so far you already have enough knowledge to add some functional style to your everyday coding. You don't have to know every trick in the book to start seeing the benefits of this paradigm.

So, I want you to pay close attention to something called **partial application**, just like the concept of a **pure function**, partial application can help you a lot even if you decide you don't want to write code in a functional style.

This are my notes on the topic (with practical examples): 

- [Partial application](@/web-development/learn-fp/partial-application.md).

If you are convinced that this is useful then watch this video, in here you can see the kind of thing you can accomplish.

{{ youtube(id="m3svKOdZijA") }}

## How to put the pieces together

Now, knowing the basics is one thing, it's a whole other deal knowing how to use them in the most effective way. You already have the tools but you might be wondering how all of this fits together, that is our next step.

In this article we are going to learn how to use everything we have learned.

- [Composition techniques](@/web-development/learn-fp/composition-techniques.md)

Just in case you missed it. In this talk (the source of the previous article) you can see in more detail what composition is about.

{{ youtube(id="vDe-4o8Uwl8") }}

## One step further

By now you must know how to manipulate functions and make them do your bidding. But I bet there are still things you want to know in more detail, two in particular: Functors and Monads. So, I'll try my best to tell you how you can use them to your advantage.

- [Have you met Functors?](@/web-development/learn-fp/the-power-of-map.md)
- [Something about Applicative Functors](@/web-development/learn-fp/applicative-functors.md)
- [An Introduction to Monads](@/web-development/learn-fp/an-introduction-to-monads-in-js.md)
- [Using a Maybe](@/web-development/learn-fp/using-a-maybe.md)

## Extra content

- [Reduce: how and when](@/web-development/learn-fp/reduce-how-and-when.md)
- [The case for reducers](@/web-development/the-case-for-reducers.md)
- [Transducers in javascript](@/web-development/transducers-in-javascript.md)
- [Lenses: an alternative to getters and setters](@/web-development/learn-fp/lenses-a-k-a-composable-getters-and-setters.md)
- [Exploring Fantasy Land](@/web-development/tagged-unions-and-fantasy-land.md)

### More interesting talks

If you're still wondering what you can do just by composing functions just watch this.

- [Mary had a little lambda](https://www.youtube.com/watch?v=7BsfMMYvGaU)
- [Oh Composable World!](https://www.youtube.com/watch?v=SfWR3dKnFIo)

## 'Til next time

If you got here and read everything then you know as much as I do. Got nothing else to show you. Whether you decided to adopt a fully functional style or not I hope you learned something that you can apply in your everyday coding.
