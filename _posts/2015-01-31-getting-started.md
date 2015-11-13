---
title:  "Getting Started"
tags: chapters
---

<h2 id="npm">npm</h2>

The recommended channel for downloading Cycle.js as a package is through [npm](http://npmjs.org/), which follows the [CommonJS](http://wiki.commonjs.org/wiki/CommonJS) spec. Create a new directory and run this inside that directory:

{% highlight text %}
npm install rx @cycle/core @cycle/dom
{% endhighlight %}

This installs RxJS, Cycle *Core*, and Cycle *DOM*. RxJS and *Core* are the minimum required API to work with Cycle.js. *Core* includes a single function `run()`, and Cycle *DOM* is the standard DOM Driver providing a way to interface with the DOM. **RxJS is a peer dependency of Cycle *Core***.

Packages of the type `@org/package` are [npm scoped packages](https://docs.npmjs.com/getting-started/scoped-packages), supported if your npm installation is version 2.11 or higher. Check your npm version with `npm --version` and upgrade in order to install Cycle.js.

In case you are not dealing with a DOM-interfacing web application, you can omit `@cycle/dom` when installing.

<h2 id="first-steps">First steps</h2>

We recommend the use of a bundling tool such as [browserify](http://browserify.org/) or [webpack](http://webpack.github.io/), in combination with ES6 (a.k.a. ES2015) through a transpiler (e.g. [Babel](http://babeljs.io/)). Most of the code examples in this documentation assume some basic familiarity with ES6. Once your build system is set up, **write your main JavaScript source file like**:

{% highlight js %}
import Cycle from '@cycle/core';
import CycleDOM from '@cycle/dom';

// ...
{% endhighlight %}

The imported `Cycle` object on the first line contains one important function: `run(main, drivers)`, where `main` is the entry point for our whole application, and `drivers` is a record of driver functions labeled by some name.

**Create the `main` function and the `drivers` record:**

{% highlight js %}
import Cycle from '@cycle/core';
import CycleDOM from '@cycle/dom';

function main() {
  // ...
}

let drivers = {
  DOM: CycleDOM.makeDOMDriver('#app')
};

Cycle.run(main, drivers);
{% endhighlight %}

`makeDOMDriver(container)` from Cycle *DOM* returns a driver function to interact with the DOM. This function is registered under the key `DOM` in the `drivers` object above.

**Send messages from `main` to the `DOM` driver:**

{% highlight js %}
import Rx from 'rx';
import Cycle from '@cycle/core';
import CycleDOM from '@cycle/dom';

function main() {
  return {
    DOM: Rx.Observable.interval(1000)
      .map(i => CycleDOM.h(
        'h1', '' + i + ' seconds elapsed'
      ))
  };
}

let drivers = {
  DOM: CycleDOM.makeDOMDriver('#app')
};

Cycle.run(main, drivers);
{% endhighlight %}

We have filled the `main()` function with some code: returns an object which has an RxJS Observable defined under the name `DOM`. This indicates `main()` is sending the Observable as messages to the DOM driver. The Observable emits Virtual DOM elements displaying `${i} seconds elapsed` changing over time every second, where `${i}` is replaced by `0`, `1`, `2`, etc.

**Catch messages from `DOM` into `main` and vice-versa:**

{% highlight js %}
import Cycle from '@cycle/core';
import {makeDOMDriver, h} from '@cycle/dom';

function main(drivers) {
  return {
    DOM: drivers.DOM.select('input').events('click')
      .map(ev => ev.target.checked)
      .startWith(false)
      .map(toggled =>
        h('div', [
          h('input', {type: 'checkbox'}), 'Toggle me',
          h('p', toggled ? 'ON' : 'off')
        ])
      )
  };
}

let drivers = {
  DOM: makeDOMDriver('#app')
};

Cycle.run(main, drivers);
{% endhighlight %}

Function `main()` now takes `drivers` as input. Just like the output `main()` produces, the input `drivers` follow the same structure: an object containing `DOM` as a property. `drivers.DOM` is a queryable collection of Observables. Use `drivers.DOM.select(selector).events(eventType)` to get an Observable of `eventType` DOM events happening on the element(s) specified by `selector`. This `main()` function takes the Observable of clicks happening on `input` elements, and maps those toggling events to Virtual DOM elements displaying a togglable checkbox.

We used the `h()` helper function to create virtual DOM elements, but you can also use JSX with Babel. The following only works if you are building with Babel: (1) add the line `/** @jsx hJSX */` at the top of the file, (2) install the [babel-plugin-transform-react-jsx](http://babeljs.io/docs/plugins/transform-react-jsx/) npm package and specify a pragma for jsx as shown in the following example:, (3) import hJSX as `import {hJSX} from '@cycle/dom';`, and then you can utilize JSX instead of `h()`:

{% highlight json %}
{
  "plugins": [
    ["transform-react-jsx", { "pragma": "DOM.hJSX" }]
  ]
}
{% endhighlight %}

{% highlight html %}
/** @jsx hJSX */
import Cycle from '@cycle/core';
import {makeDOMDriver, hJSX} from '@cycle/dom';

function main(drivers) {
  return {
    DOM: drivers.DOM.select('input').events('click')
      .map(ev => ev.target.checked)
      .startWith(false)
      .map(toggled =>
        <div>
          <input type="checkbox" /> Toggle me
          <p>{toggled ? 'ON' : 'off'}</p>
        </div>
      )
  };
}

let drivers = {
  DOM: makeDOMDriver('#app')
};

Cycle.run(main, drivers);
{% endhighlight %}

This example portrays the most common problem-solving pattern in Cycle.js: formulate the computer's behavior as a function of Observables: continuously listen to driver events and continuously provide messages (in our case, Virtual DOM elements) to the drivers. Read the next chapter to get familiar with this pattern.

<h2 id="cyclejs-as-a-script">Cycle.js as a script</h2>

In the rare occasion you need Cycle.js as a standalone JavaScript file, you can download them on the [Releases](https://github.com/cyclejs/cycle-core/releases) page at GitHub.

- Download the latest [Cycle Core](https://github.com/cyclejs/cycle-core/releases)
- Download the latest [Cycle DOM](https://github.com/cyclejs/cycle-dom/releases)
