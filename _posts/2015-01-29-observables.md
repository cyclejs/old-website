---
title:  "Observables"
tags: chapters
---

The name "Observable" immediately indicates a relation to the [Observer pattern](https://en.wikipedia.org/wiki/Observer_pattern). This pattern is key in many flavors of the [Model-View-Controller](https://en.wikipedia.org/wiki/Model%E2%80%93view%E2%80%93controller) architectural pattern for user interfaces. For instance, typically the View observes changes in the Model. In the Observer pattern, this means the Model would be the "Subject" being observed.

However, an Observable is not exactly the same concept as a traditional Subject from the Observer pattern, because Observables share features with Iterables from the [Iterator pattern](https://en.wikipedia.org/wiki/Iterator_pattern) as well.

Observables originated from [ReactiveX](http://reactivex.io/intro.html), a Reactive Programming library. Reactivity is an important aspect in Cycle.js, and part of the core principles that led to the creation of this framework. There is a lot of confusion surrounding what Reactive means, so let's focus on that topic for a while.

## Reactive Programming

Say you have a module Foo and a module Bar. A *module* can be considered to be an object of an [OOP](https://en.wikipedia.org/wiki/Object-oriented_programming) class, or any other mechanism of encapsulating state. Let's assume all code lives in some module. Here we have an arrow from Foo to Bar, indicating that Foo somehow affects state living inside Bar.

<p>
  {% include img/modules-foo-bar.svg %}
</p>

A practical example of such arrow would be: *whenever Foo does a network request, increment a counter in B*. If all code lives in some module, **where does this arrow live?** Where is it defined? The typical choice would be to write code inside Foo which calls a method in Bar to increment the counter.

{% highlight js %}
// This is inside the Foo module

function onNetworkRequest() {
  // ...
  Bar.incrementCounter();
  // ...
}
{% endhighlight %}

Because Foo owns the relationship "*when network request happens, increment counter in Bar*", we say the arrow lives at the arrow tail, i.e., Foo.

<p>
  {% include img/passive-foo-bar.svg %}
</p>

Bar is **passive**: it allows other modules to change its state. Foo is proactive: it is responsible for making Bar's state function correctly. The passive module is unaware of the existence of the arrow which affects it.

The alternative to this approach inverts the ownership of the arrow, without inverting the arrow's direction.

<p>
  {% include img/reactive-foo-bar.svg %}
</p>

With this approach, Bar listens to an event happening in Foo, and manages its own state when that event happens.

{% highlight js %}
// This is inside the Bar module

Foo.addOnNetworkRequestListener(() => {
  self.incrementCounter(); // self is Bar
});
{% endhighlight %}

Bar is **reactive**: it is fully responsible for managing its own state by reacting to external events. Foo, on the other hand, is unaware of the existence of the arrow originating from its network request event.

What is the benefit of this approach? Inversion of Control, mainly because Bar is responsible for itself. Plus, we can hide Bar's `incrementCounter()` as a private function. In the passive case, it was required to have `incrementCounter()` public, which means we are exposing Bar's internal state management outwards. It also means if we want to discover how does Bar's counter work, we need to find all usages of `incrementCounter()` in the codebase. In this regard, Reactive and Passive seem to be duals of each other.

|                      | Passive                 | Reactive      |
|----------------------|-------------------------|---------------|
| How does it work?    | *Find usages*           | Look inside   |

On the other hand, when applying the Reactive pattern, if you want to discover which modules does an event in a Listenable module affect, you must find all usages of that event.

|                      | Proactive               | Listenable    |
|----------------------|-------------------------|---------------|
| What does it affect? | Look inside             | *Find Usages* |

Passive/Proactive programming has been the default way of working for most programmers in imperative languages. Sometimes the Reactive pattern is used, but sporadically. The selling point for widespread Reactive programming is to build self-responsible modules which focus on their own functionality rather than changing external state. This leads to Separation of Concerns.

The challenge with Reactive programming is this paradigm shift where we attempt to choose the Reactive/Listenable approach by default, before considering Passive/Proactive. After rewiring your brain to think Reactive-first, the learning curve flattens and most tasks become straight-forward, specially when using a Reactive library like RxJS.

## RxJS Observables

Reactive programming can be implemented with: event listeners, [Bacon.js](http://baconjs.github.io/), [Kefir](https://rpominov.github.io/kefir/), [EventEmitter](https://nodejs.org/api/events.html), [Actors](https://en.wikipedia.org/wiki/Actor_model), and more. Even [spreadsheets](https://en.wikipedia.org/wiki/Reactive_programming) utilize the same idea of the cell formula defined at the arrow head. The above definition of Reactive programming is not limited to RxJS, and does not conflict with previous definitions of Reactive Programming. RxJS was chosen as the reactive tool in Cycle.js.

> #### Why RxJS?
> 
> Among JavaScript reactive programming libraries (often claiming to be "Functional Reactive Programming"), RxJS is a pioneer. It has inspired Bacon.js, which in turn has inspired Kefir. All major contenders to RxJS are inspired by it.
> 
> There is also an [ongoing effort](https://github.com/zenparsing/es-observable) by the [TC39 committee](http://www.ecma-international.org/memento/TC39.htm) to insert the `Observable` into JavaScript ([ECMA262](https://github.com/tc39/ecma262)), like [Promise](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise) is in ES6. This would enable the future replacement of RxJS with a native `Observable`, making Cycle.js a very lightweight framework.

The precise definition of an Observable is given at [ReactiveX.io documentation site](http://reactivex.io/intro.html). The Observable was first introduced in [Rx.NET](https://github.com/Reactive-Extensions/Rx.NET), then ported to [JavaScript](https://github.com/Reactive-Extensions/RxJS), [Java](https://github.com/ReactiveX/RxJava), and other languages. *Reactive Extensions*, or *ReactiveX*, is the name of this collection of libraries across languages.

In short, an *Observable* is an event stream which can emit zero or more events, and may or may not finish. If it finishes, then it does so by either emitting an error or a special "complete" event.

{% highlight text %}
Observable contract: (OnNext)* (OnCompleted|OnError){0,1}
{% endhighlight %}

As an example, here is an typical Observable: it emits some events, then eventually completes.

<p>
  {% include img/completed-observable.svg %}
</p>

Observables can be listened, just like EventEmitters and DOM events can.

{% highlight js %}
myObservable.subscribe(
  function handleNextEvent(event) {
    // do something with `event`
  },
  function handleError(error) {
    // do something with `error`
  },
  function handleCompleted() {
    // do something
  }
);
{% endhighlight %}

Notice there are 3 handlers: one for events, one for errors, and one for "complete". You can omit the last two in case you are only interested in handling "onNext" events.

RxJS Observables become very useful when you can transform them with pure functions, creating new Observables on top of existing ones. Given an Observable of click events, you can make an Observable of "double click" events.

{% highlight js %}
let doubleClickObservable = clickObservable
  // collect an array of events after 250ms of event silence
  .buffer(() => clickObservable.debounce(250))
  // allow only arrays of length 2
  .map(arr => arr.length)
  .filter(x => x === 2);
{% endhighlight %}

[Succinctness is Power](http://www.paulgraham.com/power.html), and RxJS operators demonstrate that you can achieve a lot with a few well-placed operators. These are only a few among the [wide palette of available operators](http://reactivex.io/documentation/operators.html) on Observables.

Knowing the basics of ReactiveX is a pre-requisite to getting work done with Cycle.js. Instead of teaching RxJS on this site, we recommend a few great learning resources, in case you need to learn more:

- [The introduction to Reactive Programming you've been missing](https://gist.github.com/staltz/868e7e9bc2a7b8c1f754): a thorough introduction to RxJS by Cycle.js author Andre Staltz.
- [Introduction to Rx](http://introtorx.com/): an online book focused on Rx.NET, but most concepts map directly to RxJS.
- [ReactiveX.io](http://reactivex.io/): official cross-language documentation site for ReactiveX.
- [Learn Rx](http://reactive-extensions.github.io/learnrx/): an interactive tutorial with arrays and Observables, by Jafar Husain.
- [RxJS lessons at Egghead.io](https://egghead.io/technologies/rx)
- [RxJS GitBook](http://xgrommx.github.io/rx-book/)
- [RxMarbles](http://rxmarbles.com/): interactive diagrams of RxJS operators, built with Cycle.js.
- [Async JavaScript at Netflix](https://www.youtube.com/watch?v=XRYN2xt11Ek): video of Jafar Husain introducing RxJS.

## Observables in Cycle.js

Now we are able to answer some questions left from the previous chapter, such as "What are the types of `senses` and `actuators`?" and what does it mean to say "the computer and the human are mutually observed".

In the simplest case, the computer generates pixels on the screen, and the human generates mouse and keyboard events. The computer can observe the user's input events, and the human observes the screen generated by the computer. Notice we can model each of these parts as *Observables*.

- Computer's output: a screen Observable.
- Human's output: an Observable of mouse/keyboard events.

The `computer()` function takes the human's output as input, and vice-versa. They mutually observe each other's output. In JavaScript, we could write the computer function as a simple chain of RxJS transformations on the input observable.

{% highlight js %}
function computer(userEventsObservable) {
  return userEventsObservable
    .map(event => /* ... */)
    .filter(somePredicate)
    .flatMap(transformItToScreenPixels);
}
{% endhighlight %}

While doing the same with the `human()` function would be elegant, we cannot do that as a simple chain of RxJS operators because we need to leave the JavaScript environment and affect the external world. While conceptually the `human()` function can exist, in practice we need to use *driver* functions in order to reach the external world.

**Drivers** are proxies to the user, and each driver represents one aspect of external effects. For instance, the DOM Driver takes a "screen" Observable generated by the computer, and returns Observables of mouse and keyboard events. In between, the DOM Driver function produces "*write*" side effects to render elements on the DOM, and catches "*read*" side effects to detect user interaction. This way, the DOM Driver function can act on behalf of the user.

> #### Request and response
> 
> From the perspective of the user, the Observable of "screen" generated by the computer are *requests*, and the user's reaction to those requests (mouse and keyboard events) are *responses*. 
> 
> Here is a another way of seeing it. In a dialogue, one side listens to *questions* (requests), and generates *answers* (responses). It is not a problem if *request/response* resemble the Client-Server model, because a Cycle.js network request will actually be called *request* in the dialogue abstraction. The concepts are the same, except in Cycle.js the words *request* and *response* are generalized to include other agents. For instance, the User can be regarded as a remote server to which we are sending and receiving messages. 
> 
> In this documentation, you will sometimes read about the computer taking responses and generating requests. You can understand this as the computer doing new requests to the human, given what the user responded previously to the computer.

Joining both parts, we have a computer function, often called `main()`, and a driver function, where the output of one is the input of the other.

{% highlight text %}
y = domDriver(x)
x = main(y)
{% endhighlight %}

The circular dependency above cannot be solved if `=` means assignment, because that would be equivalent to the command `x = g(f(x))`, and `x` is undefined on the right-hand side.

This is where Cycle.js comes in. You only need to specify `main()` and `domDriver()`, and give it to Cycle.js `run()` which will connect them together circularly.

{% highlight js %}
function main(responses) {
  let requests = {
    DOM: // transform responses.DOM through a series
         // of RxJS operators
  };
  return requests;
}

let drivers = {
  DOM: makeDOMDriver('#app') // a Cycle.js helper factory
};

Cycle.run(main, drivers); // solve the circular dependency
{% endhighlight %}

This is how the name "*Cycle.js*" came to be. It is a framework to solve the cyclic dependency of Observables that emerges in the dialogue (mutual observation) between Human and Computer.

[Read next about basic examples](/basic-examples.html) applying what we've learned so far of Cycle.js.
