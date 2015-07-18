## Functional Unidirectional Dataflow

<p>
  {% include /img/human-computer-diagram.svg %}
</p>

Cycle's core abstraction is Human-Computer Interaction modelled as an interplay between two pure functions: `human()` and `computer()`. The computer outputs what the human takes as input, and vice-versa, leading to the fixed point equation `x = human(computer(x))`, where `x` is an Observable. The human and the computer are mutually observed. This is what we call "Functional Unidirectional Dataflow", or "Reactive Dialogue", and as an app developer you only need to specify the `computer()` function.

- - -

## Example

{% highlight js %}
import {run} from '@cycle/core';
import {h, makeDOMDriver} from '@cycle/dom';

function main(responses) {
  return {
    DOM: responses.DOM.get('.field', 'input')
      .map(ev => ev.target.value)
      .startWith('')
      .map(name =>
        h('div', [
          h('label.label', 'Name:'),
          h('input.field', {attributes: {type: 'text'}}),
          h('hr'),
          h('h1.header', 'Hello ' + name)
        ])
      )
  };
}

run(main, {
  DOM: makeDOMDriver('.js-container')
});
{% endhighlight %}

The computer function is `main()`, with input `responses` as a collection of Response [Observables](http://reactivex.io/intro.html) (event streams from [ReactiveX](http://reactivex.io/)), and outputs a collection of Request Observables. The human function is represented by the DOM Driver in the code above, because in the context of a web application, the DOM is a proxy to the user. The responsibility of `main()` is to transform DOM Response Observables to DOM Request Observables, through a chain of RxJS operators. To learn more about this approach, the [documentation](/getting-started.html) will guide you through more details.

- - -

## Fully Reactive

The building blocks in Cycle are Observables from [RxJS](https://github.com/Reactive-Extensions/RxJS), which simplify code related to events, asynchrony, and errors. Structuring the application with RxJS also separates concerns, because Observables decouple data production from data consumption. As a result, apps in Cycle have nothing comparable to imperative calls such as `setState()`, `forceUpdate()`, `replaceProps()`, `handleClick()`, etc. In the [reactive pattern](/observables.html#reactive-programming), no module has methods of the type `foo.update()`, which normally leak responsibility of handling state living in `foo`. You can write code with single responsibilities throughout.

- - -

## Sliceable

Most frameworks claim to provide Separation of Concerns, but often they prescribe rigid containers where to place your code: Models, Views, Controllers, Components, Routes, Services, Dispatcher, Stores, Actions, Templates, etc. Cycle has none of that. Instead, pure functions over Observables and immutable data structures (such as from [mori](https://swannodette.github.io/mori/) or [Immutable.js](https://facebook.github.io/immutable-js/)) allow you to *slice* your program wherever you wish.

- - -

## Supports...

- **Virtual DOM rendering**: Cycle comes with a driver to interface with the DOM through [virtual-dom](https://github.com/Matt-Esch/virtual-dom), a fast diff & patch library.
- **Universal JavaScript**: In relation to the *virtual-dom* driver, there is a HTML-generating driver function for server-side rendering, enabling code reuse.
- **thisless JavaScript**: The use of functions and RxJS Observables allow for a JavaScript programming style without the `this` keyword. Cycle.js encourages you to create apps with functional practices. Without `this`, you can write more reusable code and define logic without tightly coupling it to data. See it for yourself, `this` cannot be found in [Cycle.js TodoMVC](https://github.com/staltz/todomvc-cycle/tree/master/src).
- **Good testability**: With functions and Observables, testing is mostly a matter of feeding input and inspecting the output.
- **Extensibility**: write your own driver or use community-built drivers to make use of React, React Native, AJAX, Web Sockets, or other side effects.
