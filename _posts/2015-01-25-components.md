---
title: "Components"
tags: chapters
---

User interfaces are usually made up of many reusable pieces: buttons, charts, sliders, hoverable avatars, smart form fields, etc. In many frameworks, including Cycle.js, these are called components. However, in this framework they have a special property.

#### Any Cycle.js app can be reused as a component in a larger Cycle.js app.

How is that so? In any framework you can build a program that just makes one slider. By now, we know how to make a Cycle.js `main()` that makes a smart slider widget. Then, since `main()` is just a function taking inputs from the external world and generating outputs in return, we can just call that function inside a larger Cycle.js app.

Each of these "small Cycle.js `main()` functions" are called **dataflow components**. The sources which a dataflow component receives are Observables provided by its parent, and sinks are Observables given back to the parent. All along, we have been building dataflow components, because the `main()` given to `Cycle.run(main, drivers)` is also a dataflow component. Its parent are the drivers, because that is where its sources come from and where its sinks go to.

<p>
  {% include img/dataflow-component.svg %}
</p>

To learn by doing, let's just make a dataflow component for a single labeled slider. It should take user events as input, and generate a virtual DOM Observable of a slider element. Besides the virtual DOM, it might also output a value: the observable of slider values. It might also take attributes (from its parent) as input to customize some behavior or looks. These are sometimes called *props* ("properties") in other frameworks.

<h2 id="a-labeled-slider-custom-element">A labeled slider component</h2>

A labeled slider has two parts: a label and slider, side by side, where the label always displays the current dynamic value of the slider.

<a class="jsbin-embed" href="http://jsbin.com/ritajuxele/embed?output">JS Bin on jsbin.com</a>

Every labeled slider has some properties:

 - Label text (`'Weight'`, `'Height'`, etc)
 - Unit text (`'kg'`, `'cm'`, etc)
 - Min value
 - Max value
 - Initial value

These props can be encoded as an object, wrapped in an Observable, and passed to our `main()` function as a *source* input:

{% highlight js %}
function main(sources) {
  const props$ = sources.props$;
  // ...
  return sinks;
}
{% endhighlight %}

To use this main function, we call `Cycle.run`:

{% highlight js %}
Cycle.run(main, {
  props$: () => Observable.of({
    label: 'Weight', unit: 'kg', min: 40, initial: 70, max: 140
  }),
  DOM: makeDOMDriver('#app')
});
{% endhighlight %}

Remember that even though we are building a component, we are assuming our labeled slider to be our main program. Then, because `props$` are an input given to the labeled slider from its parent, the only parent of `main()` in this case is `Cycle.run`. That is why we need to configure `props$` as a driver.

The other input to this labeled slider program is the DOM source representing user events:

{% highlight diff %}
 function main(sources) {
+  const DOMSource = sources.DOM;
   const props$ = sources.props$;
   // ...
   return sinks;
 }
{% endhighlight %}

The remaining of the program is rather easy given that we've written the same labeled slider in the two previous chapters. However, this time we take props as the initial value.

{% highlight js %}
function main(sources) {
  const initialValue$ = sources.props$
    .map(props => props.initial)
    .first();

  const newValue$ = sources.DOM
    .select('.slider')
    .events('input')
    .map(ev => ev.target.value);

  const value$ = initialValue$.concat(newValue$);

  const vtree$ = Observable.combineLatest(sources.props$, value$,
    (props, value) =>
      div('.labeled-slider', [
        span('.label',
          props.label + ' ' + value + props.unit
        ),
        input('.slider', {
          type: 'range', min: props.min, max: props.max, value
        })
      ])
  );

  const sinks = {
    DOM: vtree$,
    value$,
  };
  return sinks;
}
{% endhighlight %}

You might have noticed that besides the virtual DOM output, we also return the `value$` Observable as a sink:

{% highlight diff %}
   // ...
   const sinks = {
     DOM: vtree$,
+    value$,
   };
   return sinks;
 }
{% endhighlight %}

This value Observable is crucial to be sent to the parent if the parent wishes to use the numeric value for some calculations, such as that of BMI. In the program we wrote above, the parent of `main()` are the drivers. The drivers don't need to use `value$`, that's why we don't need a driver named `value$`. However, when the parent of the slider component is another dataflow component, like in the next section, then `value$` will be important.

<h2 id="using-a-component">Using a component</h2>

Now that our dataflow component for a labeled slider is ready, we can use it in the context of a larger application. First, we will rename our component to `LabeledSlider`, and `main()` will refer to our larger application.

{% highlight diff %}
-function main(sources) {
+function LabeledSlider(sources) {
   const initialValue$ = sources.props$
     .map(props => props.initial)
     .first();

   // ...

   return sinks;
 }

+function main(sources) {
+  // Call LabeledSlider() here...
+}
{% endhighlight %}

Since `LabeledSlider` is just a function, we can call it with some sources to get its sinks as output.

{% highlight js %}
function main(sources) {
  const props$ = Observable.of({
    label: 'Radius', unit: '', min: 10, initial: 30, max: 100
  });
  const childSources = {DOM: sources.DOM, props$};
  const labeledSlider = LabeledSlider(childSources);
  const childVTree$ = labeledSlider.DOM;
  const childValue$ = labeledSlider.value$;

  // ...
}
{% endhighlight %}

> <h4 id="why-name-components-with-capitalcase">Why name components with CapitalCase?</h4>
>
> You probably noticed we named the dataflow component as `LabeledSlider`. Usually in JavaScript, capitalized names are used for classes and constructor functions. Since Cycle.js uses functional programming techniques heavily, Object-oriented programming conventions are irrelevant, there are rarely classes in Cycle.js apps.
>
> For this reason, capitalized names become available in the functional programming flavor of JavaScript. We will follow the convention of capitalized names such as `FooButton` for dataflow components (in other words, small Cycle.js apps). Their camel-case counterpart such as `fooButton` will refer to the output of `FooButton` function when called, i.e., the sinks object.

Now we have `childVTree$` and `childValue$` as sinks from the labeled slider, available for use as regular Observables in the context of the `main()` parent. We use `childValue$` to render a circle with radius equal to the slider's value, and use `childVTree$` to embed the slider's virtual DOM in the parent's virtual DOM:

{% highlight js %}
function main(sources) {
  // ...

  const childVTree$ = labeledSlider.DOM;
  const childValue$ = labeledSlider.value$;

  const vtree$ = childVTree$.withLatestFrom(childValue$,
    (childVTree, value) =>
      div([
        childVTree,
        div({style: {
          backgroundColor: '#58D3D8',
          width: String(value) + 'px',
          height: String(value) + 'px',
          borderRadius: String(value * 0.5) + 'px'
        }})
      ])
    );
  return {
    DOM: vtree$
  };
}
{% endhighlight %}

As a result, we get a Cycle.js program where the labeled slider controls the size of a rendered circle.

<a class="jsbin-embed" href="http://jsbin.com/lobigimovo/embed?output">JS Bin on jsbin.com</a>

<h2 id="multiple-instances-of-the-same-component">Multiple instances and isolation</h2>

Our labeled sliders were originally built for the BMI example, so we should see how the component we just built can be used in the BMI example.

The naïve approach is to simply call `LabeledSlider()` twice, once with props for weight, and again with props for height:

{% highlight js %}
function main(sources) {
  const weightProps$ = Observable.of({
    label: 'Weight', unit: 'kg', min: 40, initial: 70, max: 150
  });
  const heightProps$ = Observable.of({
    label: 'Height', unit: 'cm', min: 140, initial: 170, max: 210
  });

  const weightSources = {DOM: sources.DOM, props$: weightProps$};
  const heightSources = {DOM: sources.DOM, props$: heightProps$};

  const weightSlider = LabeledSlider(weightSources);
  const heightSlider = LabeledSlider(heightSources);

  const weightVTree$ = weightSlider.DOM;
  const weightValue$ = weightSlider.value$;

  const heightVTree$ = heightSlider.DOM;
  const heightValue$ = heightSlider.value$;

  // ...
}
{% endhighlight %}

However, this creates a bug. At least one of the labeled sliders does not work: its label does not change when the slider moves. Can you see why? Pay attention to the implementation of `LabeledSlider` with this piece of code:

{% highlight js %}
function LabeledSlider(sources) {
  // ...

  const newValue$ = sources.DOM
    .select('.slider')
    .events('input')
    .map(ev => ev.target.value);

  const value$ = initialValue$.concat(newValue$);

  // ...
}
{% endhighlight %}

Suppose we just ran this function for the weight labeled slider. The line `sources.DOM.select('.slider')` **will attempt to select all** `.slider` **elements on the entire DOM tree managed by this app**. This means both the `.slider` in the weight component and the `.slider` in the height component. As a result, the weight component will detect changes to both the height slider and the weight slider, which is a bug.

A component should not leak its output to other components, and it should not be able to detect outputs from other sibling components. In order keep the nice property of "a component is just a Cycle.js app", we want two properties:

- A component's **sources** are not affected by other components' sources.
- A component's **sinks** do not affect other components' sinks.

In order to achieve these properties, we need to modify the sources when they enter the component, and also modify the sinks when they are returned from the component. To make sources and sinks isolated from influence of other components, we need to introduce a scope for the current component.

For the DOM source and DOM sink, we can use a unique identifier string as namespace for the virtual DOM element.

DRAFT: We add a className on the sink (the vtree)

DRAFT: code code code for the map(vtree => add classname) on the sink

DRAFT: Show the HTML output we get.

DRAFT: and also we narrow down the DOM source by doing a select() for that same className

DRAFT: code code code for the DOM source

QUOTEBOX: DOM source scope selection

DRAFT: Show both source and sink narrow-down code.

DRAFT: Extract those two things as isolateSource and isolateSink

DRAFT: Show how these two are attached to sources.DOM

DRAFT: Introduce isolate() as a helper library

DRAFT: Show how isolate calls both isolateSource and isolateSink for every source and sink that has such method. Show how we can call isolate(component, 'weight')

DRAFT: Show how we can call instead isolate(component) without providing the namespace, and how its auto-generated for us.

## Overview

DRAFT: Call isolate by default on components and you'll be safe against global collisions, and each component can work as if it would be the only one in the application.

DRAFT: From a component's perspective, it should make no assumption on what the parent is.

DRAFT: isolateSource and isolateSink is a driver-specific function. Each driver should define how its sources and sinks should be isolated.

DRAFT: Recap: The role of sources and sinks as interfaces, where the top-most component is main, and its parents are drivers that take sinks and give sources.

- - -

<h2 id="using-components-in-a-parent">Using components in a parent</h2>

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




- - -

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

<a class="jsbin-embed" href="http://jsbin.com/negulukoxo/embed?output">JS Bin on jsbin.com</a>

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

<a class="jsbin-embed" href="http://jsbin.com/wubuwunaka/embed?output">JS Bin on jsbin.com</a>

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

<a class="jsbin-embed" href="http://jsbin.com/xamehogoze/embed?output">JS Bin on jsbin.com</a>

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
