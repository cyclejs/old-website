## Functional Unidirectional Dataflow

![Human-Computer Interaction in Cycle.js](/img/human-computer-diagram.svg)

Cycle's core abstraction is Human-Computer Interaction modelled as an interplay between two pure functions: `human()` and `computer()`. The computer outputs what the human takes as input, and vice-versa, leading to the fixed point equation `x = human(computer(x))`, where `x` is an Observable. The human and the computer are mutually observed. This is what we call "Functional Unidirectional Dataflow", or "Reactive Dialogue", and as an app developer you only need to specify the `computer()` function.

- - -

## Example

{% highlight js %}
import Cycle from 'cyclejs';
let {h} = Cycle;

function main(drivers) {
  return {
    DOM: drivers.DOM.get('.field', 'input')
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

Cycle.run(main, {
  DOM: Cycle.makeDOMDriver('.js-container')
});
{% endhighlight %}

The input of `main` is `drivers`, a collection of Observables from the "external world". `drivers.DOM` is a queryable collection of Observable user events happening on elements on the DOM. Query it using a getter: `get(selector, eventType)` returns an Observable of `eventType` events happening on elements specified by `selector`. This goes through a series of RxJS operations to produce an Observable of virtual DOM elements created with the `h()` helper, which is returned and tagged as `DOM`. Function `Cycle.run()` will take your `main` function and circularly connect it to the specified "driver functions". The DOM driver function acts on behalf of the user: takes the tagged `DOM` Observable of virtual elements returned from `main()`, shows that on the screen as a side effect, and outputs Observables of user interaction events. The result of this is Human-Computer Interaction, i.e. a dialogue between `main()` and the DOM driver function, happening under the container element selected by `'.js-container'`.

- - -

## Fully Reactive

The building blocks in Cycle are Observables from [RxJS](https://github.com/Reactive-Extensions/RxJS), which simplify code related to events, asynchrony, and errors. Structuring the application with RxJS also separates concerns, because Observables decouple data production from data consumption. As a result, apps in Cycle have nothing comparable to imperative calls such as `setState()`, `forceUpdate()`, `replaceProps()`, `handleClick()`, etc. The reactive pattern makes it possible for no module to have methods of the type `foo.update()` which leak responsibility of handling state living in `foo`. You can write code with single responsibilities throughout.

- - -

## Sliceable

Most frameworks claim to provide Separation of Concerns, but often they prescribe rigid containers where to place your code: Models, Views, Controllers, Components, Routes, Services, Dispatcher, Stores, Actions, Templates, etc. Cycle has none of that. Instead, pure functions over Observables and immutable data structures (such as from [mori](https://swannodette.github.io/mori/) or [Immutable.js](https://facebook.github.io/immutable-js/)) allow you to *slice* your program wherever you wish. 

- - -

## Supports...

- **Virtual DOM rendering**: Cycle comes with a driver to interface with the DOM through [virtual-dom](https://github.com/Matt-Esch/virtual-dom), a fast diff & patch library.
- **Universal JavaScript**: In relation to the *virtual-dom* driver, there is a related HTML-generating driver function for server-side rendering, enabling code reuse.
- **thisless JavaScript**: The use of functions and RxJS Observables allow for a JavaScript programming style without the pitfalling `this`. Code is clearer and less prone to `this`-related bugs. See it for yourself, `this` cannot be found in [Cycle.js TodoMVC](https://github.com/staltz/todomvc-cycle/tree/master/js).
- **Good testability**: With functions and Observables, testing is mostly a matter of feeding input and inspecting the output. You can also trivially mock the `human()` function, or any driver function.
- **Extensibility**: write your own driver or use community-built drivers to make use of React, React Native, AJAX, Web Sockets, or other side effects.
