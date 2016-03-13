---
title:  "Documentation"
tags: chapters
---

## Cycle *Core* [v6.0.0](https://github.com/cyclejs/cycle/releases/tag/v6.0.0) API: `Cycle` object

- [`run`](#run)

### <a id="run"></a> `run(main, drivers)`

Takes an `main` function and circularly connects it to the given collection
of driver functions.

The `main` function expects a collection of "driver source" Observables
as input, and should return a collection of "driver sink" Observables.
A "collection of Observables" is a JavaScript object where
keys match the driver names registered by the `drivers` object, and values
are Observables or a collection of Observables.

##### Arguments:

- `main :: Function` a function that takes `sources` as input and outputs a collection of `sinks` Observables.
- `drivers :: Object` an object where keys are driver names and values are driver functions.

##### Return:

*(Object)* an object with two properties: `sources` and `sinks`. `sinks` is the collection of driver sinks, and `sources` is the collection
of driver sources, that can be used for debugging or testing.

- - -

## Cycle *DOM* [v8.0.0](https://github.com/cyclejs/cycle-dom/releases/tag/v8.0.0) API: `CycleDOM` object

- [`makeDOMDriver`](#makeDOMDriver)
- [`makeHTMLDriver`](#makeHTMLDriver)
- [`h`](#h)
- [`hyperscript-helpers`](#hyperscript-helpers)
- [`hJSX`](#hJSX)
- [`svg`](#svg)
- [`mockDOMSource`](#mockDOMSource)

### <a id="makeDOMDriver"></a> `makeDOMDriver(container, options)`

A factory for the DOM driver function. Takes a `container` to define the
target on the existing DOM which this driver will operate on. The output
("source") of this driver is a collection of Observables queried with:
`DOMSource.select(selector).events(eventType)` returns an Observable of
events of `eventType` happening on the element determined by `selector`.
Just `DOMSource.select(selector).observable` returns an Observable of the
DOM element matched by the given selector. Also,
`DOMSource.select(':root').observable` returns an Observable of DOM element
corresponding to the root (or container) of the app on the DOM. The
`events()` function also allows you to specify the `useCapture` parameter
of the event listener. That is, the full function signature is
`events(eventType, useCapture)` where `useCapture` is by default `false`.

##### Arguments:

- `container :: String|HTMLElement` the DOM selector for the element (or the element itself) to contain the rendering of the VTrees.
- `options :: Object` an options object containing additional configurations. The options object is optional. These are the parameters
that may be specified:
  - `onError`: a callback function to handle errors. By default it is
  `console.error`.

##### Return:

*(Function)* the DOM driver function. The function expects an Observable of VTree as input, and outputs the source object for this
driver, containing functions `select()` and `dispose()` that can be used
for debugging and testing.

- - -

### <a id="makeHTMLDriver"></a> `makeHTMLDriver()`

A factory for the HTML driver function.

##### Return:

*(Function)* the HTML driver function. The function expects an Observable of Virtual DOM elements as input, and outputs an Observable of
strings as the HTML renderization of the virtual DOM elements.

- - -

### <a id="h"></a> `h`

A shortcut to [virtual-hyperscript](
https://github.com/Matt-Esch/virtual-dom/tree/master/virtual-hyperscript).
This is a helper for creating VTrees in Views.

- - -

### <a id="hyperscript-helpers"></a> `hyperscript-helpers`

Shortcuts to
[hyperscript-helpers](https://github.com/ohanhi/hyperscript-helpers).
This is a helper for writing virtual-hyperscript. Create virtual DOM
elements with `div('.wrapper', [ h1('Header') ])` instead of
`h('div.wrapper', [ h('h1', 'Header') ])`.

- - -

### <a id="hJSX"></a> `hJSX()`

An adapter around virtual-hyperscript `h()` to allow JSX to be used easily
with Babel. Place the [Babel configuration comment](
http://babeljs.io/docs/plugins/transform-react-jsx/) `@jsx hJSX` at
the top of the ES6 file, make sure you import `hJSX` with
`import {hJSX} from '@cycle/dom'`, and then you can use JSX to create
VTrees.

- - -

### <a id="svg"></a> `svg`

A shortcut to the svg hyperscript function.

- - -

### <a id="mockDOMSource"></a> `mockDOMSource(mockedSelectors)`

A testing utility which aids in creating a queryable collection of
Observables. Call mockDOMSource giving it an object specifying selectors,
eventTypes and their Observables, and get as output an object following the
same format as the DOM Driver's source. Example:

{% highlight js %}
const userEvents = mockDOMSource({
  '.foo': {
    'click': Rx.Observable.just({target: {}}),
    'mouseover': Rx.Observable.just({target: {}})
  },
  '.bar': {
    'scroll': Rx.Observable.just({target: {}})
  }
});

// Usage
const click$ = userEvents.select('.foo').events('click');
{% endhighlight %}

##### Arguments:

- `mockedSelectors :: Object` an object where keys are selector strings and values are objects. Those nested objects have eventType strings as keys
and values are Observables you created.

##### Return:

*(Object)* fake DOM source object, containing a function `select()` which can be used just like the DOM Driver's source. Call
`select(selector).events(eventType)` on the source object to get the
Observable you defined in the input of `mockDOMSource`.

- - -

## Cycle *HTTP* Driver [v7.0.0](https://github.com/cyclejs/cycle-http-driver/releases/tag/v7.0.0) API: `CycleHTTPDriver` object

- [`makeHTTPDriver`](#makeHTTPDriver)

### <a id="makeHTTPDriver"></a> `makeHTTPDriver(options)`

HTTP Driver factory.

This is a function which, when called, returns a HTTP Driver for Cycle.js
apps. The driver is also a function, and it takes an Observable of requests
as input, and generates a metastream of responses.

**Requests**. The Observable of requests should emit either strings or
objects. If the Observable emits strings, those should be the URL of the
remote resource over HTTP. If the Observable emits objects, these should be
instructions how superagent should execute the request. These objects
follow a structure similar to superagent's request API itself.
`request` object properties:

- `url` *(String)*: the remote resource path. **required**
- `method` *(String)*: HTTP Method for the request (GET, POST, PUT, etc).
- `query` *(Object)*: an object with the payload for `GET` or `POST`.
- `send` *(Object)*: an object with the payload for `POST`.
- `headers` *(Object)*: object specifying HTTP headers.
- `accept` *(String)*: the Accept header.
- `type` *(String)*: a short-hand for setting Content-Type.
- `user` *(String)*: username for authentication.
- `password` *(String)*: password for authentication.
- `field` *(Object)*: object where key/values are Form fields.
- `attach` *(Array)*: array of objects, where each object specifies `name`,
`path`, and `filename` of a resource to upload.
- `withCredentials` *(Boolean)*: enables the ability to send cookies from
the origin.
- `redirects` *(Number)*: number of redirects to follow.
- `eager` *(Boolean)*: whether or not to execute the request regardless of
  usage of its corresponding response. Default value is `false` (i.e.,
  the request is lazy). Main use case is: set this option to `true` if you
  send POST requests and you are not interested in its response.

**Responses**. A metastream is an Observable of Observables. The response
metastream emits Observables of responses. These Observables of responses
have a `request` field attached to them (to the Observable object itself)
indicating which request (from the driver input) generated this response
Observable. The response Observables themselves emit the response object
received through superagent.

##### Arguments:

- `options :: Object` an object with settings options that apply globally for all requests processed by the returned HTTP Driver function. The
options are:
- `eager` *(Boolean)*: execute the HTTP eagerly, even if its
  response Observable is not subscribed to. Default: **false**.

##### Return:

*(Function)* the HTTP Driver function
