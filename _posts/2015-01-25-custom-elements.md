---
title: "Custom elements"
tags: chapters
---

Some parts of our UI share the same *looks* and *behavior*. Looks are the rendering of VTrees from state, in other words *View*. Behavior is what happens when the user generates DOM events related to that UI, in other words, *Intent*. Let's call these UI parts as *Widget*.

> <h4 id="what-is-a-widget">What is a "Widget"?</h4>
>
> We will refer to "Widget" as a small reusable user interface program with: (1) an interface to interact with a parent UI program, (2) no business logic.
>
> Examples of Widgets in user interfaces: a slider with a dynamic label, a dropdown selector list, a knob in an audio-editing program, a button with special behavior, an animated checkbox.

We need a way of encapsulating looks and behavior together, in order to reuse them seamlessly. View and Intent both relate to state, which relates to Model. So we actually need a way of encapsulating all three Model, View, Intent as a reusable component to implement a Widget.

Isn't MVI an architecture for Cycle.js UI programs? Does this mean a Widget is a Cycle.js application? Well, "*if it looks like a duck, swims like a duck, and quacks like a duck, then it probably [is a duck](https://en.wikipedia.org/wiki/Duck_test).*"

**Custom elements are small Cycle.js apps to implement Widgets.** Here is the skeleton implementation of most custom elements:

{% highlight js %}
function myCustomElement(responses) {
  // Use responses.DOM to get DOM events
  // happening on the elements from this
  // custom element.
  //
  // Use responses.props to get properties
  // passed to this custom elements from
  // the parent Cycle.js app.
  // ...
  return {
    DOM: vtree$,
    events: {
      foo: foo$,
      bar: bar$ // 'bar' is a custom DOM event
                // emitted whenever the Observable
                // bar$ emits.
    }
  };
}
{% endhighlight %}

Notice the similar structure to a `main()` function for a Cycle.js app:

{% highlight js %}
function main(responses) {
  return {
    driver1: // ...
    driver2: // ...
  };
}
{% endhighlight %}

Let's learn by doing: how to create custom elements and use them in a Cycle.js application.

<h2 id="a-labeled-slider-custom-element">A labeled slider custom element</h2>

From the [last chapter](/model-view-intent.html#pursuing-dry) we saw the need to have a labeled slider: a label and slider, side by side, where the label always displays the current dynamic value of the slider.

<a class="jsbin-embed" href="http://jsbin.com/gadavevawe/embed?output">JS Bin on jsbin.com</a>

Every labeled slider has some properties:

 - Label text (`'Weight'`, `'Height'`, etc)
 - Unit text (`'kg'`, `'cm'`, etc)
 - Min value
 - Max value
 - Initial value

We want to use our labeled slider like any other DOM element. If we can express `h('div')` or `h('select')`, we should also be able to express `h('labeled-slider')`. And we give properties (or "attributes") to it as such:

{% highlight js %}
h('labeled-slider#weight', {
  label:'Weight', unit:'kg', min: 40, max: 140, initial: 70
})
{% endhighlight %}

We have access to these properties as `responses.props` from the custom element implementation function:

{% highlight js %}
function labeledSlider(responses) {
  // Observable of the 'label' property
  let label$ = responses.props.get('label');
  // ...
  return requests;
}
{% endhighlight %}

`responses.props.get(propName)` returns an Observable of `propName` values. Use `responses.props.getAll()` to get an Observable of the properties object containing all properties: `{label, unit, min, max, initial}`. Creating `h('labeled-slider', propsObject)` and using `responses.props` is how our parent Cycle.js app can communicate to our small custom element Cycle.js app. What about the opposite direction?

The labeled slider needs to communicate to the parent application whenever a new value is set on the slider, so e.g. our BMI calculator can update the BMI result. The labeled slider can emit a custom DOM event `newValue`. To declare this `newValue` event in the custom element's implementation, we return an Observable as part of `labeledSlider`'s `request` output.

{% highlight js %}
function labeledSlider(responses) {
  let newValue$ = // we need to create this Observable
  // ...
  let requests = {
    DOM: vtree$,
    events: { // a collection of all custom DOM events
      newValue: newValue$
    }
  };
  return requests;
}
{% endhighlight %}

We now just need to make use of `responses.props` and `responses.DOM`, create `newValue$` and `vtree$`, and return these. Here is the full implementation of the labeled slider custom element:

{% highlight js %}
function labeledSlider(responses) {
  let initialValue$ = responses.props.get('initial').first();
  let newValue$ = responses.DOM.select('.slider').events('input')
    .map(ev => ev.target.value);
  let value$ = initialValue$.concat(newValue$);
  let props$ = responses.props.getAll();
  let vtree$ = Rx.Observable
    .combineLatest(props$, value$, (props, value) =>
      h('div.labeled-slider', [
        h('span.label', [
          props.label + ' ' + value + props.unit
        ]),
        h('input.slider', {
          type: 'range',
          min: props.min,
          max: props.max,
          value: value
        })
      ])
    );

  return {
    DOM: vtree$,
    events: {
      newValue: newValue$
    }
  };
}
{% endhighlight %}

Because this `labeledSlider()` function is a Cycle.js app, we can refactor it with MVI architecture.

{% highlight js %}
function labeledSlider(responses) {
  function intent(DOM) {
    return {
      changeValue$: DOM.select('.slider').events('input')
        .map(ev => ev.target.value)
    };
  }

  function model(context, actions) {
    let initialValue$ = context.props.get('initial').first();
    let value$ = initialValue$.concat(actions.changeValue$);
    let props$ = context.props.getAll();
    return Rx.Observable.combineLatest(props$, value$,
      (props, value) => { return {props, value}; }
    );
  }

  function view(state$) {
    return state$.map(state => {
      let {label, unit, min, max} = state.props;
      let value = state.value;
      return h('div.labeled-slider', [
        h('span.label', [label + ' ' + value + unit]),
        h('input.slider', {type: 'range', min, max, value})
      ])
    });
  }

  let actions = intent(responses.DOM);
  let vtree$ = view(model(responses, actions));

  return {
    DOM: vtree$,
    events: {
      newValue: actions.changeValue$
    }
  };
}
{% endhighlight %}

Now `labeledSlider()` is a small Cycle.js implementing our custom element, but we still don't know how to call this function in the context of the parent Cycle.js app. We do this by registering the labeled slider with an element name (`<labeled-slider>`) when we create the DOM Driver:

{% highlight js %}
let domDriver = CycleDOM.makeDOMDriver('#app', {
  'labeled-slider': labeledSlider // our function
});
{% endhighlight %}

This API was designed in a way that mimics [Web Components](http://webcomponents.org/). A Cycle.js app can make use of `'labeled-slider'` and stay oblivious to its implementation, making no assumptions whether it was implemented as a Cycle.js Custom Element or as a Web Component.

The complete BMI application using labeled slider custom elements is:

{% highlight js %}
import Cycle from '@cycle/core';
import {h, makeDOMDriver} from '@cycle/dom';

function labeledSlider(responses) {
  function intent(DOM) {
    return {
      changeValue$: DOM.select('.slider').events('input')
        .map(ev => ev.target.value)
    };
  }

  function model(context, actions) {
    let initialValue$ = context.props.get('initial').first();
    let value$ = initialValue$.concat(actions.changeValue$);
    let props$ = context.props.getAll();
    return Rx.Observable.combineLatest(props$, value$,
      (props, value) => { return {props, value}; }
    );
  }

  function view(state$) {
    return state$.map(state => {
      let {label, unit, min, max} = state.props;
      let value = state.value;
      return h('div.labeled-slider', [
        h('span.label', [label + ' ' + value + unit]),
        h('input.slider', {type: 'range', min, max, value})
      ])
    });
  }

  let actions = intent(responses.DOM);
  let vtree$ = view(model(responses, actions));

  return {
    DOM: vtree$,
    events: {
      newValue: actions.changeValue$
    }
  };
}

function calculateBMI(weight, height) {
  const heightMeters = height * 0.01;
  return Math.round(weight / (heightMeters * heightMeters));
}

function intent(DOM) {
  return {
    changeWeight: DOM.select('#weight').events('newValue')
      .map(ev => ev.detail),
    changeHeight: DOM.select('#height').events('newValue')
      .map(ev => ev.detail)
  };
}

function model(actions) {
  return Rx.Observable
    .combineLatest(
      actions.changeWeight.startWith(70),
      actions.changeHeight.startWith(170),
      (weight, height) => {
        return {
          weight,
          height,
          bmi: calculateBMI(weight, height)
        };
      }
    );
}

function view(state) {
  return state.map(({weight, height, bmi}) =>
    h('div', [
      h('labeled-slider#weight', {
        key: 1, label: 'Weight', unit: 'kg',
        min: 40, initial: weight, max: 140
      }),
      h('labeled-slider#height', {
        key: 2, label: 'Height', unit: 'cm',
        min: 140, initial: height, max: 210
      }),
      h('h2', 'BMI is ' + bmi)
    ])
  );
}

function main({DOM}) {
  return {
    DOM: view(model(intent(DOM)))
  };
}

Cycle.run(main, {
  DOM: makeDOMDriver('#app', {
    'labeled-slider': labeledSlider
  })
});
{% endhighlight %}

<a class="jsbin-embed" href="http://jsbin.com/rubaqakiyo/embed?output">JS Bin on jsbin.com</a>

<h2 id="custom-element-with-children-property">Custom element with `children` property</h2>

The interface for your custom element can also get passed a VTree to be embedded as a children. We can change our labeled slider to receive the label as a child VTree instead of a property string `label`.

To provide children, just give an array to the `h()` helper like we normally do with typical elements:

{% highlight js %}
h('div', [
  h('labeled-slider#weight', {/* properties */}, [
    h('h2', 'Weight')
  ]),
  h('labeled-slider#height', {/* properties */}, [
    h('h4', 'Height')
  ])
])
{% endhighlight %}

In the custom element's implementation function `labeledSlider()`, we can access these children simply through `responses.props.get('children')`, or as the `children` property in the props object emitted by `responses.props.getAll()` Observable.

{% highlight js %}
function labeledSlider(responses) {
  let initialValue$ = responses.props.get('initial').first();
  let newValue$ = responses.DOM.select('.slider').events('input')
    .map(ev => ev.target.value);
  let value$ = initialValue$.concat(newValue$);
  let props$ = responses.props.getAll();
  let vtree$ = Rx.Observable
    .combineLatest(props$, value$, (props, value) =>
      h('div.labeled-slider', [
        h('div.label',
          // props.children is an array with all
          // the VTree objects passed into this
          // custom element.
          props.children.concat(value + props.unit)
        ),
        h('input.slider', {
          type: 'range',
          min: props.min,
          max: props.max,
          value: value
        })
      ])
    );

  return {
    DOM: vtree$,
    events: {
      newValue: newValue$
    }
  };
}
{% endhighlight %}

<a class="jsbin-embed" href="http://jsbin.com/qafepavuwo/embed?output">JS Bin on jsbin.com</a>

<h2 id="tips-and-best-practices">Tips and best practices</h2>

Using a custom element is similar to using typical DOM elements, with a few differences: event data is sent through `event.detail`, and the recommended use of `key` property on the virtual DOM elements.

Here is the Intent code interpreting actions from the labeled sliders:

{% highlight js %}
function intent(DOM) {
  return {
    changeWeight: DOM.select('#weight').events('newValue')
      .map(ev => ev.detail),
    changeHeight: DOM.select('#height').events('newValue')
      .map(ev => ev.detail)
  };
}
{% endhighlight %}

Notice `map(ev => ev.detail)`. Because custom elements emit [custom DOM events](https://developer.mozilla.org/en-US/docs/Web/API/CustomEvent), they come with a read-only `detail` property, which is where data goes when emitted by the Observable internal to the custom element. The property `detail` follows Web Specifications.

When we render a custom element, we usually should provide a unique identifier under the `key` property. This is [`virtual-dom` specific](https://github.com/Matt-Esch/virtual-dom/blob/master/docs/vnode.md#arguments), and helps to resolve ambiguities.

{% highlight js %}
function view(state) {
  return state.map(({weight, height, bmi}) =>
    h('div', [
      h('labeled-slider#weight', {
        key: 1, label: 'Weight', unit: 'kg',
        min: 40, initial: weight, max: 140
      }),
      h('labeled-slider#height', {
        key: 2, label: 'Height', unit: 'cm',
        min: 140, initial: height, max: 210
      }),
      h('h2', 'BMI is ' + bmi)
    ])
  );
}
{% endhighlight %}

Notice `h('labeled-slider', { key: 1, ... })`.

> <h4 id="why-use-key">Why use `key`?</h4>
>
> The use of `key` is necessary for specifying that the element in question is still the same even though its properties might have changed.
>
> Here is a counterexample: Imagine you have a custom element `<gif-with-filters>` which plays an animated gif, but you can specify a color `filter` such as "black and white" or "sepia". If you don't use a `key` for this element, when the `filter` property changes (for instance from BW to sepia), we will have an ambiguous situation. The virtual DOM diff and patch will not know whether you want to:
>
> 1.  Replace the current `<gif-with-filters>` BW with a **new** `<gif-with-filters>` sepia, hence **restarting** the gif animation **or**
> 2.  Keep the same `<gif-with-filters>` element but just swap the filter from BW to sepia without restarting the animation.
>
> To fix this ambiguity we use keys. If you want (1), then you provide a different key when you change the filter. If you want (2), then you provide the same key but change the filter property. Both cases (1) and (2) should be equally easy to express, and should be a responsibility of the app developer.

Another good practice is to avoid making custom elements for anything, but instead reserving them for implementing Widgets only. The rule of thumb is: if it contains business logic, it shouldn't be a custom element. In most cases you need just a function from state to VTree.

Other special behavior can normally be implemented as a function from Observable to Observable. For instance, say you have a `vtree$` Observable. You can wrap it with additional looks or behavior by transforming the `vtree$` Observable with RxJS operators.

<h4 id="recap">Recap</h4>

- Use `event.detail` to get data sent in the custom event
- Use `key` when creating VTrees for custom elements
- Avoid creating custom elements with app business logic
- Avoid custom elements if you can just use a pure function
- Avoid custom elements to implement anything else than a Widget
