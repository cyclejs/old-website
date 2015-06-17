## Functional Unidirectional Dataflow

**TODO SVG DIAGRAM COMES HERE**

Cycle's core abstraction is Human-Computer Interaction modelled as an interplay between two pure functions: `human()` and `computer()`. The computer outputs what the human takes as input, and vice-versa, leading to the fixed point equation `x = human(computer(x))`, where `x` is an event stream. The human and the computer are mutually observed. This is what we call "Functional Unidirectional Dataflow", or "Reactive Dialogue", and as an app developer you only need to specify the `computer()` function.

- - -

## Honestly Reactive

The building blocks in Cycle are Observables from [RxJS](https://github.com/Reactive-Extensions/RxJS), which simplifies code related to events, asynchrony, and errors. Structuring the app with RxJS also separates concerns, because Observables decouples data production from data consumption. As result, apps in Cycle have nothing comparable to imperative calls such a `setState()`, `forceUpdate()`, `replaceProps()`, `handleClick()`, etc.
