## Your app and the external world as a circuit

<p>
  {% include /img/cycle-nested-frontpage.svg %}
</p>

Cycle's core abstraction is your application as a pure function `main()` where inputs are observable side effects (*sources*) from the external world and outputs (*sinks*) are commands or updates to affect the external world. These side effects in the external world are managed by *drivers*: plugins that handle DOM effects, HTTP effects, etc.

The internals of `main()` are built using Reactive programming primitives, which maximizes separation of concerns, providing a clean and fully declarative way of organizing your code. The *dataflow* is plainly visible, making it easy to read and understand the code.

## Example

{% highlight text %}
npm install rx @cycle/core @cycle/dom
{% endhighlight %}

{% highlight js %}
import Cycle from '@cycle/core';
import {div, label, input, h1, makeDOMDriver} from '@cycle/dom';

function main(sources) {
  const sinks = {
    DOM: sources.DOM.select('.field').events('input')
      .map(ev => ev.target.value)
      .startWith('')
      .map(name =>
        div([
          label('Name:'),
          input('.field', {attributes: {type: 'text'}}),
          h1('Hello ' + name)
        ])
      )
  };
  return sinks;
}

Cycle.run(main, { DOM: makeDOMDriver('#app-container') });
{% endhighlight %}

<table class="homepage-features">
<tr>
<td><h2>Functional<br />and<br />Reactive</h2></td>
<td markdown="1">Functional means "clean", and Reactive means "separated". Cycle.js apps are made of pure functions, which means you know they simply take inputs and generate outputs, without performing any side effects. The building blocks are Observables from [RxJS](https://github.com/Reactive-Extensions/RxJS), a Reactive programming library which simplifies code related to events, asynchrony, and errors. Structuring the application with RxJS also separates concerns, because all [dynamic updates to a piece of data are co-located](/observables.html#reactive-programming) and impossible to change from outside. As a result, apps in Cycle are entirely `this`-less and have nothing comparable to imperative calls such as `setState()` or `foo.update()`.
</td>
</tr>
</table>

<table class="homepage-features">
<tr>
<td markdown="1">Cycle.js is a framework with very few concepts to learn. The core API has just one function: `run(app, drivers)`. Besides that, there are **Observables**, **functions**, **drivers** (plugins for different types of side effects), and a helper function to isolate scoped components. This is a framework with very little amount of "magic". Most of the building blocks are just JavaScript functions. Usually the lack of "magic" leads to very verbose code, but since RxJS Observables are able to build complex dataflows with a few operations, you will come to see how apps in Cycle.js are small and readable.
</td>
<td><h2>Simple<br />and<br />Concise</h2></td>
</tr>
</table>

<table class="homepage-features">
<tr>
<td><h2>Extensible<br />and<br />Testable</h2></td>
<td markdown="1">Drivers are plugin-like simple functions that take messages from sinks and call imperative functions. All side effects are contained in drivers. This means your application is just a pure function, and it becomes easy to swap drivers around. The community has built drivers for [React Native](https://github.com/cyclejs/cycle-react-native), [HTML5 Notification](https://github.com/cyclejs/cycle-notification-driver), [Socket.io](https://github.com/cgeorg/cycle-socket.io), etc. Sources and sinks can be easily used as [Adapters and Ports](https://iancooper.github.io/Paramore/ControlBus.html). This also means testing is mostly a matter of feeding inputs and inspecting the output. No deep mocking needed. You application is just a pure transformation of data.
</td>
</tr>
</table>

## Explicit dataflow

In every Cycle.js app, each of the Observable declarations is a node in a dataflow graph, and the dependencies between declarations are arrows. This means there is a one-to-one correspondence between your code and *minimap*-like graph of the dataflow between external inputs and external outputs.

<table class="homepage-features">
<tr>
<td>
<p class="dataflow-minimap" markdown="1">
  {% include /img/dataflow-minimap.svg %}
</p>
</td>
<td class="dataflow-minimap-code" markdown="1">
{% highlight js %}
function main(sources) {
  const decrement$ = sources.DOM
    .select('.decrement').events('click').map(ev => -1);

  const increment$ = sources.DOM
    .select('.increment').events('click').map(ev => +1);

  const action$ = Observable.merge(decrement$, increment$);
  const count$ = action$.startWith(0).scan((x,y) => x+y);

  const vtree$ = count$.map(count =>
    div([
      button('.decrement', 'Decrement'),
      button('.increment', 'Increment'),
      p('Counter: ' + count)
    ])
  );
  return { DOM: vtree$ };
}
{% endhighlight %}
</td>
</tr>
</table>

In many frameworks the flow of data is *implicit*: you need to build a mental model of how data moves around in your app. In Cycle.js, the flow of data is clear by reading your code.

## Supports...

- [**Virtual DOM rendering**](https://github.com/cyclejs/cycle-dom)
- [**Server-side rendering**](https://github.com/cyclejs/cycle-examples/tree/master/isomorphic)
- [**JSX**](http://cycle.js.org/getting-started.html)
- [**React Native**](https://github.com/cyclejs/cycle-react-native)
- [**Time travelling**](https://github.com/cyclejs/cycle-time-travel)
- [**Routing with the History API**](https://github.com/cyclejs/cycle-history)
- [**And more...**](https://github.com/vic/awesome-cyclejs)
