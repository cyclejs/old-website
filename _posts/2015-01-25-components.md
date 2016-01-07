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

The remainder of the program is rather easy given that we've written the same labeled slider in the two previous chapters. However, this time we take props as the initial value.

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

  const bmi$ = Observable.combineLatest(weightValue$, heightValue$,
    (weight, height) => {
      const heightMeters = height * 0.01;
      const bmi = Math.round(weight / (heightMeters * heightMeters));
      return bmi;
    }
  );

  return {
    DOM: bmi$.combineLatest(weightVTree$, heightVTree$,
      (bmi, weightVTree, heightVTree) =>
        div([
          weightVTree,
          heightVTree,
          h2('BMI is ' + bmi)
        ])
      )
  };
}
{% endhighlight %}

<a class="jsbin-embed" href="http://jsbin.com/xobuyesezo/embed?output">JS Bin on jsbin.com</a>

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

For the DOM source and DOM sink, we can use a unique identifier string as namespace for the virtual DOM element. First, we patch the DOM sink, adding a className to the VTrees it emits.

{% highlight diff %}
 function main(sources) {
   // ...

   const weightSlider = LabeledSlider(weightSources);
   const heightSlider = LabeledSlider(heightSources);

   const weightVTree$ = weightSlider.DOM
+    .map(vtree => {
+      vtree.className += ' weight';
+      return vtree;
+    });
   const weightValue$ = weightSlider.value$;

   const heightVTree$ = heightSlider.DOM
+    .map(vtree => {
+      vtree.className += ' height';
+      return vtree;
+    });
   const heightValue$ = heightSlider.value$;

   // ...
 }
{% endhighlight %}

This will result in the following rendered HTML:

{% highlight html %}
<div class="labeled-slider weight">
  <span class="label">Weight 70kg</span>
  <input class="slider" type="range" min="40" max="150">
</div>
{% endhighlight %}

For querying user events on these rendered sliders, the `weightSlider` dataflow component should detect user events *only* from the `<div class="labeled-slider weight">` element and its descendants when `sources.DOM.select('.slider').events('input')` is called.

In the context of the labeled slider component, **`sources.DOM.select()` should refer only to the DOM that was created by the corresponding `vtree$` sink in that component**.

We can achieve that by narrowing down the DOM source before it is given to the component, using the same className we patched on the sink, like this:

{% highlight diff %}
 function main(sources) {
   // ...
   const weightSources = {
-    DOM: sources.DOM,
+    DOM: sources.DOM.select('.weight'),
     props$: weightProps$
   };
   const heightSources = {
-    DOM: sources.DOM,
+    DOM: sources.DOM.select('.height'),
     props$: heightProps$
   };

   const weightSlider = LabeledSlider(weightSources);
   const heightSlider = LabeledSlider(heightSources);
   // ...
 }
{% endhighlight %}

> <h4 id="what-does-sources-dom-select-do">What does <code>sources.DOM.select()</code> do?</h4>
>
> We have used `.select(selector).events(eventType)` many times previously to get an Observable emitting DOM events of type `eventType` happening on the `selector` element(s).
>
> In the code above, we called `sources.DOM.select(selector)` without `.events(eventType)`. It returns a **new** DOM source, on which we can call again `select()` or `events()`.
>
> `select('.foo').select('.bar').events('click')` returns an Observable of click events happening on `'.foo .bar'` elements. In other words, these are all clicks happening on `'.bar'` elements descendants of `'.foo'` elements. The first call, `select('.foo')`, allows us to "narrow down" the DOM source.

The code we wrote for isolating sources and sinks looks like boilerplate. Ideally we want to avoid manually managing scopes for each component instance using classNames:

{% highlight js %}
function main(sources) {
  // ...
  const weightSources = {
    DOM: sources.DOM.select('.weight'), props$: weightProps$
  };
  const heightSources = {
    DOM: sources.DOM.select('.height'), props$: heightProps$
  };
  // ...
  const weightVTree$ = weightSlider.DOM
    .map(vtree => {
      vtree.className += ' weight';
      return vtree;
    });
  // ...
  const heightVTree$ = heightSlider.DOM
    .map(vtree => {
      vtree.className += ' height';
      return vtree;
    });
  // ...
}
{% endhighlight %}

To avoid repeating code, such as the `.map(vtree => ...)` which patches the VTree, we could extract the functionality into functions: `isolateDOMSink()` and `isolateDOMSource()`.

{% highlight diff %}
 function main(sources) {
   // ...
   const weightSources = {
-    DOM: sources.DOM.select('.weight'), props$: weightProps$
+    DOM: isolateDOMSource(sources.DOM, 'weight'), props$: weightProps$
   };
   const heightSources = {
-    DOM: sources.DOM.select('.height'), props$: heightProps$
+    DOM: isolateDOMSource(sources.DOM, 'height'), props$: heightProps$
   };
   // ...
-  const weightVTree$ = weightSlider.DOM
-    .map(vtree => {
-      vtree.className += ' weight';
-      return vtree;
-    });
+  const weightVTree$ = isolateDOMSink(weightSlider.DOM, 'weight');
   // ...
-  const heightVTree$ = heightSlider.DOM
-    .map(vtree => {
-      vtree.className += ' height';
-      return vtree;
-    });
+  const heightVTree$ = isolateDOMSink(heightSlider.DOM, 'height');
   // ...
 }
{% endhighlight %}

Since these are very useful helper functions, they are packaged in Cycle DOM. They are available as static functions under the DOM source: `sources.DOM.isolateSource` and `sources.DOM.isolateSink`. This is how the `main()` function looks like when we use those functions:

{% highlight js %}
function main(sources) {
  const weightProps$ = Observable.of({
    label: 'Weight', unit: 'kg', min: 40, initial: 70, max: 150
  });
  const heightProps$ = Observable.of({
    label: 'Height', unit: 'cm', min: 140, initial: 170, max: 210
  });

  const {isolateSource, isolateSink} = sources.DOM;

  const weightSources = {
    DOM: isolateSource(sources.DOM, 'weight'), props$: weightProps$
  };
  const heightSources = {
    DOM: isolateSource(sources.DOM, 'height'), props$: heightProps$
  };

  const weightSlider = LabeledSlider(weightSources);
  const heightSlider = LabeledSlider(heightSources);

  const weightVTree$ = isolateSink(weightSlider.DOM, 'weight');
  const weightValue$ = weightSlider.value$;

  const heightVTree$ = isolateSink(heightSlider.DOM, 'height');
  const heightValue$ = heightSlider.value$;

  // ...
}
{% endhighlight %}

The code above shows how we need to manually process the sources and sinks of a child component to make sure each child is run in an isolated context. It would be better, however, if we could just "isolate" a component and make source and sink isolation happen under the hood.

Such is the purpose of [`isolate()`](https://github.com/cyclejs/isolate) (`npm install @cycle/isolate`), a helper function which handles calls to `isolateSource` and `isolateSink` for us. `isolate(Component, scope)` takes a dataflow component function `Component` as input, and outputs a dataflow component function which isolates the sources to `scope`, runs `Component`, then isolates its sinks to `scope` as well. Here is a heavily simplified implementation of `isolate()`:

{% highlight js %}
function isolate(Component, scope) {
  return function IsolatedComponent(sources) {
    const {isolateSource, isolateSink} = sources.DOM;
    const isolatedDOMSource = isolateSource(sources.DOM, scope);
    const sinks = Component({DOM: isolatedDOMSource});
    const isolatedDOMSink = isolateSink(sinks.DOM, scope);
    return {
      DOM: isolatedDOMSink
    };
  };
}
{% endhighlight %}

This allows us to simplify the `main()` function with two labeled slider components:

{% highlight diff %}
 function main(sources) {
   const weightProps$ = Observable.of({
     label: 'Weight', unit: 'kg', min: 40, initial: 70, max: 150
   });
   const heightProps$ = Observable.of({
     label: 'Height', unit: 'cm', min: 140, initial: 170, max: 210
   });

-  const {isolateSource, isolateSink} = sources.DOM;
   const weightSources = {
-    DOM: isolateSource(sources.DOM, 'weight'), props$: weightProps$
+    DOM: sources.DOM, props$: weightProps$
   };
   const heightSources = {
-    DOM: isolateSource(sources.DOM, 'height'), props$: heightProps$
+    DOM: sources.DOM, props$: heightProps$
   };

+  const WeightSlider = isolate(LabeledSlider, 'weight');
+  const HeightSlider = isolate(LabeledSlider, 'height');

-  const weightSlider = LabeledSlider(weightSources);
+  const weightSlider = WeightSlider(weightSources);
-  const heightSlider = LabeledSlider(heightSources);
+  const heightSlider = HeightSlider(heightSources);

-  const weightVTree$ = isolateSink(weightSlider.DOM, 'weight');
+  const weightVTree$ = weightSlider.DOM;
   const weightValue$ = weightSlider.value$;

-  const heightVTree$ = isolateSink(heightSlider.DOM, 'height');
+  const heightVTree$ = heightSlider.DOM;
   const heightValue$ = heightSlider.value$;

   // ...
 }
{% endhighlight %}

Notice the line which creates the `WeightSlider` component:

{% highlight js %}
const WeightSlider = isolate(LabeledSlider, 'weight');
{% endhighlight %}

`isolate()` takes a non-isolated component `LabeledSlider` and restricts it to the `'weight'` scope, creating `WeightSlider`. The scope `'weight'` is only used in this line of code, and nowhere else. We can simplify this code a bit more, by making the scope parameter implicit:

{% highlight js %}
const WeightSlider = isolate(LabeledSlider);
{% endhighlight %}

This does the same as previously, except the scope parameter is unique and autogenerated. The scope string itself was irrelevant to us, so we let `isolate()` generate some scope string for us.

> <h4 id="is-isolate-referentially-transparent">Is <code>isolate()</code> referentially transparent?</h4>
>
> If we leave the scope parameter implicit for both weight and height sliders, then the code becomes
>
> `const WeightSlider = isolate(LabeledSlider);`<br />
> `const HeightSlider = isolate(LabeledSlider);`
>
> Because the right-hand side is the same, does this mean `WeightSlider` and `HeightSlider` are the same component? **Certainly not.**
>
> `isolate()` with an implicit scope parameter is **not** referentially transparent. In other words, calling `isolate()` with an implicit scope is "impure". `WeightSlider` and `HeightSlider` are not the same components. Each one has its own unique scope parameter.
>
> On the other hand, when using an explicit scope parameter, then `isolate()` is referentially transparent. In other words, `Foo` and `Fuu` are the same here:
>
> `const Foo = isolate(LabeledSlider, 'myScope');`<br />
> `const Fuu = isolate(LabeledSlider, 'myScope');`
>
> Since Cycle.js follows functional programming techniques, usually most of its API is referentially transparent. `isolate()` is an exception, for convenience. If you want referential transparency everwhere, then provide explicit scope parameters. If you want convenience and you know how `isolate()` works, then use implicit scope parameters.

If we compare our last code with the code we initially started out naïvely for `main()` to make the BMI calculator, the only difference is the use of `isolate()` on child components:

{% highlight diff %}
 function main(sources) {
   const weightProps$ = Observable.of({
     label: 'Weight', unit: 'kg', min: 40, initial: 70, max: 150
   });
   const heightProps$ = Observable.of({
     label: 'Height', unit: 'cm', min: 140, initial: 170, max: 210
   });

   const weightSources = {DOM: sources.DOM, props$: weightProps$};
   const heightSources = {DOM: sources.DOM, props$: heightProps$};

-  const weightSlider =         LabeledSlider(weightSources);
+  const weightSlider = isolate(LabeledSlider)(weightSources);
-  const heightSlider =         LabeledSlider(heightSources);
+  const heightSlider = isolate(LabeledSlider)(heightSources);

   const weightVTree$ = weightSlider.DOM;
   const weightValue$ = weightSlider.value$;

   const heightVTree$ = heightSlider.DOM;
   const heightValue$ = heightSlider.value$;

   const bmi$ = Observable.combineLatest(weightValue$, heightValue$,
     (weight, height) => {
       const heightMeters = height * 0.01;
       const bmi = Math.round(weight / (heightMeters * heightMeters));
       return bmi;
     }
   );

   return {
     DOM: bmi$.combineLatest(weightVTree$, heightVTree$,
       (bmi, weightVTree, heightVTree) =>
         div([
           weightVTree,
           heightVTree,
           h2('BMI is ' + bmi)
         ])
       )
   };
 }
{% endhighlight %}

The takeaway is: **when creating multiple instances of the same type of component, just remember to `isolate` each.**

<a class="jsbin-embed" href="http://jsbin.com/bicoziniqu/embed?output">JS Bin on jsbin.com</a>

> <h4 id="Should-I-always-call-isolate-manually">Should I always call <code>isolate()</code> manually?</h4>
>
> While it seems that you need to manually call `isolate()` every time you want to instantiate a component, in practice you can automate it.
>
> Instead of exporting the original non-isolated component, like this:
>
> `export function OriginalComponent(sources)`<br />
> `  // ...`<br />
> `}`<br />
>
> just export a function that calls `isolate()`:
>
> `export Component = sources =>`<br />
> `  isolate(OriginalComponent)(sources)`
>
> Doing this gives the consumer automatic isolation without having to think about it.

## Recap

To achieve reusability, ***any* Cycle.js app is simply a function that can be reused as a component in larger Cycle.js app**. Sources and sinks are the interface between the application and the drivers, but they are also the interface between a child component and its parent.

<p>
  {% include img/nested-components.svg %}
</p>

From a component's perspective, it should make no assumption on what the parent is. The parent could either be the drivers if the component is used as the `main()`, or the parent could be any other component. For this reason, a component should assume its sources contain only data related to itself. Therefore, the sources and sinks of a component must be *isolated*.

Use `isolateSource` and `isolateSink` to separate the execution contexts of sibling components or unrelated components. Use `isolate` to create a component that automatically applies `isolateSource` and `isolateSink`. This way your codebase will be safe against [*collisions*](https://en.wikipedia.org/wiki/Collision_%28computer_science%29), and each component can work as if it would be the only one in the application.

Each driver should define static functions `isolateSource` and `isolateSink`. We only saw those functions implemented for the DOM Driver, but there are other use cases with other drivers where it makes sense to apply the same isolation techniques. To learn more, read about [Drivers](/drivers.html).
