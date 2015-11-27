---
title: "Model-View-Intent"
tags: chapters
---

In the [previous chapter](/basic-examples.html#body-mass-index-calculator) we wrote a program entirely inside the `main()` function. This isn't a good idea, and we need to do something about it.

{% highlight js %}
import Cycle from '@cycle/core';
import {h, makeDOMDriver} from '@cycle/dom';

function main({DOM}) {
  let changeWeight$ = DOM.select('#weight').events('input')
    .map(ev => ev.target.value);
  let changeHeight$ = DOM.select('#height').events('input')
    .map(ev => ev.target.value);
  let state$ = Cycle.Rx.Observable.combineLatest(
    changeWeight$.startWith(70),
    changeHeight$.startWith(170),
    (weight, height) => {
      let heightMeters = height * 0.01;
      let bmi = Math.round(weight / (heightMeters * heightMeters));
      return {weight, height, bmi};
    }
  );

  return {
    DOM: state$.map(({weight, height, bmi}) =>
      h('div', [
        h('div', [
          'Weight ' + weight + 'kg',
          h('input#weight', {type: 'range', min: 40, max: 140, value: weight})
        ]),
        h('div', [
          'Height ' + height + 'cm',
          h('input#height', {type: 'range', min: 140, max: 210, value: height})
        ]),
        h('h2', 'BMI is ' + bmi)
      ])
    )
  };
}

Cycle.run(main, {
  DOM: makeDOMDriver('#app')
});
{% endhighlight %}

We have plenty of anonymous functions which could be refactored away from `main`, such as the BMI calculation, VTree rendering, etc.

{% highlight js %}
import Cycle from '@cycle/core';
import {h, makeDOMDriver} from '@cycle/dom';

// NEW!
function renderWeightSlider(weight) {
  return h('div', [
    'Weight ' + weight + 'kg',
    h('input#weight', {type: 'range', min: 40, max: 140, value: weight})
  ]);
}

// NEW!
function renderHeightSlider(height) {
  return h('div', [
    'Height ' + height + 'cm',
    h('input#height', {type: 'range', min: 140, max: 210, value: height})
  ]);
}

// NEW!
function calculateBMI(weight, height) {
  let heightMeters = height * 0.01;
  return Math.round(weight / (heightMeters * heightMeters));
}

// UPDATED!
function main({DOM}) {
  let changeWeight$ = DOM.select('#weight').events('input')
    .map(ev => ev.target.value);
  let changeHeight$ = DOM.select('#height').events('input')
    .map(ev => ev.target.value);
  let state$ = Cycle.Rx.Observable.combineLatest(
    changeWeight$.startWith(70),
    changeHeight$.startWith(170),
    (weight, height) =>
      ({weight, height, bmi: calculateBMI(weight, height)})
  );

  return {
    DOM: state$.map(({weight, height, bmi}) =>
      h('div', [
        renderWeightSlider(weight),
        renderHeightSlider(height),
        h('h2', 'BMI is ' + bmi)
      ])
    )
  };
}

Cycle.run(main, {
  DOM: makeDOMDriver('#app')
});
{% endhighlight %}

`main` still has to handle too many concerns. Can we do better? Yes, we can, by using the insight that `state$.map(state => someVTree)` is a *View* function: renders visual elements as a transformation of state. Let's introduce `function view(state$)`.

{% highlight js %}
import Cycle from '@cycle/core';
import {h, makeDOMDriver} from '@cycle/dom';

function renderWeightSlider(weight) {
  return h('div', [
    'Weight ' + weight + 'kg',
    h('input#weight', {type: 'range', min: 40, max: 140, value: weight})
  ]);
}

function renderHeightSlider(height) {
  return h('div', [
    'Height ' + height + 'cm',
    h('input#height', {type: 'range', min: 140, max: 210, value: height})
  ]);
}

function calculateBMI(weight, height) {
  let heightMeters = height * 0.01;
  return Math.round(weight / (heightMeters * heightMeters));
}

// NEW!
function view(state$) {
  return state$.map(({weight, height, bmi}) =>
    h('div', [
      renderWeightSlider(weight),
      renderHeightSlider(height),
      h('h2', 'BMI is ' + bmi)
    ])
  );
}

// UPDATED!
function main({DOM}) {
  let changeWeight$ = DOM.select('#weight').events('input')
    .map(ev => ev.target.value);
  let changeHeight$ = DOM.select('#height').events('input')
    .map(ev => ev.target.value);
  let state$ = Cycle.Rx.Observable.combineLatest(
    changeWeight$.startWith(70),
    changeHeight$.startWith(170),
    (weight, height) =>
      ({weight, height, bmi: calculateBMI(weight, height)})
  );

  return {
    DOM: view(state$)
  };
}

Cycle.run(main, {
  DOM: makeDOMDriver('#app')
});
{% endhighlight %}

Now, `main` is much smaller. But is it doing *one thing*? We have `changeWeight$`, `changeHeight$`, `state$`, and the return using `view(state$)`. Normally when we work with a *View*, we also have a *Model*. What Models normally do is **manage state**. In our case, however, we have `state$` which is self-responsible for its own changes, because it is [reactive](/observables.html#reactive-programming). But anyway we have code that defines how `state$` depends on `changeWeight$` and `changeHeight$`. We can put that code inside a `model()` function.

{% highlight js %}
import Cycle from '@cycle/core';
import {h, makeDOMDriver} from '@cycle/dom';

function renderWeightSlider(weight) {
  return h('div', [
    'Weight ' + weight + 'kg',
    h('input#weight', {type: 'range', min: 40, max: 140, value: weight})
  ]);
}

function renderHeightSlider(height) {
  return h('div', [
    'Height ' + height + 'cm',
    h('input#height', {type: 'range', min: 140, max: 210, value: height})
  ]);
}

function calculateBMI(weight, height) {
  let heightMeters = height * 0.01;
  return Math.round(weight / (heightMeters * heightMeters));
}

// NEW!
function model(changeWeight$, changeHeight$) {
  return Cycle.Rx.Observable.combineLatest(
    changeWeight$.startWith(70),
    changeHeight$.startWith(170),
    (weight, height) =>
      ({weight, height, bmi: calculateBMI(weight, height)})
  );
}

function view(state$) {
  return state$.map(({weight, height, bmi}) =>
    h('div', [
      renderWeightSlider(weight),
      renderHeightSlider(height),
      h('h2', 'BMI is ' + bmi)
    ])
  );
}

// UPDATED!
function main({DOM}) {
  let changeWeight$ = DOM.select('#weight').events('input')
    .map(ev => ev.target.value);
  let changeHeight$ = DOM.select('#height').events('input')
    .map(ev => ev.target.value);
  let state$ = model(changeWeight$, changeHeight$);

  return {
    DOM: view(state$)
  };
}

Cycle.run(main, {
  DOM: makeDOMDriver('#app')
});
{% endhighlight %}

`main` still defines `changeWeight$` and `changeHeight$`. What are these Observables? They are event streams of *Actions*. In the [previous chapter about basic examples](/basic-examples.html#increment-and-decrement-a-counter) we had an `action$` Observable for incrementing and decrementing a counter. These Actions are deduced or interpreted from DOM events. Their names indicate the user's *intentions*. We can group these Observable definitions in an `intent()` function:

{% highlight js %}
import Cycle from '@cycle/core';
import {h, makeDOMDriver} from '@cycle/dom';

function renderWeightSlider(weight) {
  return h('div', [
    'Weight ' + weight + 'kg',
    h('input#weight', {type: 'range', min: 40, max: 140, value: weight})
  ]);
}

function renderHeightSlider(height) {
  return h('div', [
    'Height ' + height + 'cm',
    h('input#height', {type: 'range', min: 140, max: 210, value: height})
  ]);
}

function calculateBMI(weight, height) {
  let heightMeters = height * 0.01;
  return Math.round(weight / (heightMeters * heightMeters));
}

// NEW!
function intent(DOM) {
  return {
    changeWeight$: DOM.select('#weight').events('input')
      .map(ev => ev.target.value),
    changeHeight$: DOM.select('#height').events('input')
      .map(ev => ev.target.value)
  };
}

// UPDATED!
function model(actions) {
  return Cycle.Rx.Observable.combineLatest(
    actions.changeWeight$.startWith(70),
    actions.changeHeight$.startWith(170),
    (weight, height) =>
      ({weight, height, bmi: calculateBMI(weight, height)})
  );
}

function view(state$) {
  return state$.map(({weight, height, bmi}) =>
    h('div', [
      renderWeightSlider(weight),
      renderHeightSlider(height),
      h('h2', 'BMI is ' + bmi)
    ])
  );
}

// UPDATED!
function main({DOM}) {
  let actions = intent(DOM);
  let state$ = model(actions);
  return {
    DOM: view(state$)
  };
}

Cycle.run(main, {
  DOM: makeDOMDriver('#app')
});
{% endhighlight %}

`main` is finally small enough, and works on one level of abstraction, defining how actions are created from DOM events, flowing to model and then to view, and finally back to the DOM. Because these steps are a chain, we can refactor `main` to compose those three functions `intent`, `model`, and `view` together:

{% highlight js %}
function main({DOM}) {
  return {DOM: view(model(intent(DOM)))};
}
{% endhighlight %}

Seems like we cannot achieve a simpler format for `main`.

<h4 id="recap">Recap</h4>

- `intent()` function
  - Purpose: interpret DOM events as user's intended actions
  - Input: DOM Driver responses
  - Output: Action Observables
- `model()` function
  - Purpose: manage state
  - Input: Action Observables
  - Output: `state$` Observable
- `view()` function
  - Purpose: visually represent state from the Model
  - Input: `state$` Observable
  - Output: Observable of VTree as the DOM Driver request

**Is Model-View-Intent an architecture?** Is this a new architecture? If so, how is it different to Model-View-Controller?

<h2 id="what-mvc-is-really-about">What MVC is really about</h2>

[Model-View-Controller](https://en.wikipedia.org/wiki/Model%E2%80%93view%E2%80%93controller) (MVC) has existed since the 80s as the cornerstone architecture for user interfaces. It has inspired multiple other important architectures such as [MVVM](https://en.wikipedia.org/wiki/Model_View_ViewModel) and [MVP](https://en.wikipedia.org/wiki/Model%E2%80%93view%E2%80%93presenter).

MVC is characterized by the Controller: a component which manipulates the other parts, updating them accordingly whenever the user does an action.

<p>
  {% include img/mvc-diagram.svg %}
</p>

The Controller in MVC is incompatible with our reactive ideals, because it is a proactive component (implying either passive Model or passive View). However, the original idea in MVC was a method for translating information between two worlds: that of the computer's digital realm and the user's mental model. In Trygve's own words:

> *The essential purpose of MVC is to bridge the gap between the human user's mental model and the digital model that exists in the computer.* <br />&#8211; [Trygve Reenskaug](http://heim.ifi.uio.no/~trygver/themes/mvc/mvc-index.html), inventor of MVC

We can keep the MVC ideal while avoiding a proactive Controller. In fact, if you observe our `view()` function, it does nothing else than transform state (digital model in the computer) to a visual representation useful for the user. View is a translation from one language to another: from binary data to English and other human-friendly languages.

<p>
  {% include img/view-translation.svg %}
</p>

The opposite direction should be also a straightforward translation from the user's actions to *new* digital data. This is precisely what `intent()` does: interprets what the user is trying to affect in the context of the digital model.

<p>
  {% include img/intent-translation.svg %}
</p>

Model-View-Intent (MVI) is **reactive**, **functional**, and follows the **core idea in MVC**. It is reactive because Intent observes the User, Model observes the Intent, View observes the Model, and the User observes the View. It is functional because each of these components is expressed as a [referentially transparent](https://en.wikipedia.org/wiki/Referential_transparency_%28computer_science%29) function over Observables. It follows the original MVC purpose because View and Intent bridge the gap between the user and the digital model, each in one direction.

> <h4 id="why-css-selectors-for-querying-dom-events">Why CSS selectors for querying DOM events?</h4>
>
> Some programmers get concerned about `DOM.select(selector).events(eventType)` being a bad practice because it resembles spaghetti code in jQuery-based programs. They would rather prefer the virtual DOM elements to specify handler callbacks for events, such as `onClick={this.handleClick()}`.
>
> The choice for selector-based event querying in Cycle *DOM* is an informed and rational decision. This strategy enables MVI to be reactive and is inspired by the [open-closed principle](https://en.wikipedia.org/wiki/Open/closed_principle).
>
> **Important for reactivity and MVI.** If we had Views with `onClick={this.handleClick()}`, it would mean Views would *not* be anymore a simple translation from digital model to user mental model, because we also specify what happens as a consequence of the user's actions. To keep all parts in a Cycle.js app reactive, we need the View to simply declare a visual representation of the Model. Otherwise the View becomes a Proactive component. It is beneficial to keep the View responsible only for declaring how state is visually represented: it has a [single responsibility](https://en.wikipedia.org/wiki/Single_responsibility_principle) and is friendly to UI designers. It is also conceptually aligned with the [original View in MVC](http://heim.ifi.uio.no/~trygver/1979/mvc-2/1979-12-MVC.pdf): "*... a view should never know about user input, such as mouse operations and
keystrokes.*"
>
> **Adding user actions shouldn't affect the View.** If you need to change Intent code to grab new kinds of events from the element, you don't need to modify code in the VTree element. The View stays untouched, and it should, because translation from state to DOM hasn't changed.
>
> The MVI strategy in Cycle DOM is to name *all* elements in your View with appropriate semantic classnames. Then you do not need to worry which of those can have event handlers, if all of them can. The classname is the common artifact which the View (DOM request) and the Intent (DOM response) can use to refer to the same element.

MVI is an architecture, but in Cycle it is nothing else than simply a function decomposition of `main()`.

<p>
  {% include img/main-eq-mvi.svg %}
</p>

In fact, MVI itself just naturally emerged from our refactoring of `main()` split into functions. This means Model, View, and Intent are not rigorous containers where you should place code. Instead, they are just a convenient way of organizing code, and are very cheap to create because they are simply functions. Whenever convenient, you should split a function if it becomes too large. Use MVI as a guide on how to organize code, but don't confine your code within its limits if it doesn't make sense.

This is what it means to say Cycle.js is *sliceable*. MVI is just one way of slicing `main()`.

> <h4 id="sliceable">"Sliceable"?</h4>
>
> We mean the ability to refactor the program by extracting pieces of code without having to significantly modify their surroundings. Sliceability is a feature often found in functional programming languages, especially in LISP-based languages like [Clojure](https://en.wikipedia.org/wiki/Clojure), which use S-expressions to enable treating [*code as data*](https://en.wikipedia.org/wiki/Homoiconicity).

<h2 id="pursuing-dry">Pursuing DRY</h2>

As good programmers writing good codebases, we must follow [DRY: Don't Repeat Yourself](https://en.wikipedia.org/wiki/Don%27t_repeat_yourself). The MVI code we wrote is not entirely DRY.

For instance, the View rendering of the sliders share a significant amount of code. And in the Intent, we have some duplication of the `DOM.select()` Observables.

{% highlight js %}
function renderWeightSlider(weight) {
  return h('div', [
    'Weight ' + weight + 'kg',
    h('input#weight', {type: 'range', min: 40, max: 140, value: weight})
  ]);
}

function renderHeightSlider(height) {
  return h('div', [
    'Height ' + height + 'cm',
    h('input#height', {type: 'range', min: 140, max: 210, value: height})
  ]);
}

function intent(DOM) {
  return {
    changeWeight$: DOM.select('#weight').events('input')
      .map(ev => ev.target.value),
    changeHeight$: DOM.select('#height').events('input')
      .map(ev => ev.target.value)
  };
}
{% endhighlight %}

We could create functions to remove this duplication, as such:

{% highlight js %}
function renderSlider(label, value, unit, id, min, max) {
  return h('div', [
    '' + label + ' ' + value + unit,
    h('input#' + id, {type: 'range', min, max, value})
  ]);
}

function renderWeightSlider(weight) {
  return renderSlider('Weight', weight, 'kg', 'weight', 40, 140);
}

function renderHeightSlider(height) {
  return renderSlider('Height', height, 'cm', 'height', 140, 210);
}

function getSliderEvent(DOM, id) {
  return DOM.select('#' + id).events('input').map(ev => ev.target.value);
}

function intent(DOM) {
  return {
    changeWeight$: getSliderEvent(DOM, 'weight'),
    changeHeight$: getSliderEvent(DOM, 'height')
  };
}
{% endhighlight %}

But this still isn't ideal: we seem to have *more* code now. What we really want is just to render a *labeled slider*, where the label updates in real-time when the slider is changed. We want to be able to use this in the View: `h('my-labeled-slider')`. For that, we need [custom elements](/components.html).
