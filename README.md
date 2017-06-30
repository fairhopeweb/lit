# lit-html
HTML template literals in JavaScript

`lit-html` lets you describe HTML templates with JavaScript template literals, and efficiently render and re-render those templates to DOM.

## Example

```javascript
const sayHello = (name) => html`<div>Hello ${name}!</div>`;

const container = document.querySelector('#container');
sayHello('Steve').renderTo(container);
// renders <div>Hello Steve!</div> to container

sayHello('Kevin').renderTo(container);
// updates to <div>Hello Kevin!</div>, but only updates the ${name} part
```

## API

### `html(callSite, ...expressions): TemplateResult`
`html` is a template tag for [template literals](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Template_literals), which parses the literal as HTML and returns a `TemplateResult`.

### `TemplateResult`

`TemplateResult` is a class that holds a `Template` object parsed from a template literal, and the values from its expressions.

#### `renderTo(container): void`

Renders a `TemplateResult`'s tempalte to a container using the result's values. For re-renders, only the dynamic parts are updated.

#### Property `template`

A reference to the parsed `Template` object.

#### Property `values`

The values returned by the tempalte literal's expressions.

## Status

`lit-html` is very new, under initial development, and not production-ready.

 * It uses JavaScript modules, and there's no build set up yet, so out-of-the-box it only runs in Safari 1.0 and Chrome Canary (with the Experimental Web Platform features flag on).
 * It has a growing test suite, but it has only been run manually on Chrome Canary and Safari 10.1.
 * Much more test coverage is needed for complex templates, especially template composition and Function and Iterable values.
 * It has not been benchmarked thouroughly yet.
 * The API is likely to change, especially `renderTo()` and the `Template` class used by extensions.

Even without a build configuration, `lit-html` minified with `babili` and gzipped measures in at less than 1.5k. We will strive to keep the size extremely small.

## How it Works

`html` does not return DOM nodes, unlike many other HTML with tagged template literal examples, but returns a `TemplateResult` - an object that contains a template and the values from expressions in the template - which can then be used to create or update DOM.

The template is created only the first time `html` is called on a particular template literal. On every subsequent call of `html` on the same template literal the exact same template is returned, only the values change.

To call `html` multiple times on the same template literal, it'll usually be placed in a function:

```javascript
let count = 1;
const render = () => html`count: ${count++}`;
```

The template object is based on an actual HTML `<template>` element and created by setting it's `innerHTML`, utilizing the browser's HTML parser. The HTML is not created by concatenating the string literals expression values, but by joining the literal parts with special markers. The template object finds and remembers the locations of the markers (called "parts"). This makes the system safer from XSS attacks: a value bound to an attribute can't close the attribute, text content is automatically escaped, etc.

When a template is rendered it is cloned along with the part metadata. Values are set via `setAttribute()` and `textContent`. Some state is stored on the container to indicate that a template was already rendered there. Subsequent renders use that state to update only the dynamic parts, not the entire template instance.

### Rough Algorithm

#### `html`:

1. `html` is invoked with `strings` and `values`.

    `strings` are the string literal parts as a `TemplateStringsArray`. The same instance is returned for every evaluation of a particular template literal.

2. Look up a cached template keyed by `strings`, otherwise...

3. Create a new template:

     1. Join `strings` with a special marker, `{{}}`. This creates a template string that looks like a Polymer template with empty expressions.

     2. Create a new `<template>` element and set its `innerHTML` to the generated template string.

     3. Crawl the `<template>` contents, looking for markers and remember their location by index in `Part` objects. Text-content expressions create new empty text nodes which are used as placeholders.

4. Return a `TemplateResult` with the template and the values.

#### `TemplateResult.renderTo(container)`:

1. Look for an existing `TemplateInstance` on `container`

2. If an instance exists, check that it's from the same `Template` as this `TemplateResult`

3. If the instance is from the same `Template`, remove its content from `container`.

4. If an instance doesn't exist for the node, create one from the `Template`:

    1. Clone the `Template`'s `<template>` contents.

    2. Iterate through the cloned nodes and create new "instance parts" for nodes that have `Part`s. An "instance part" is just a {part, node} record.

5. Update. For every `Part`:

    1. If it's an `AttributePart`, build up an attibute value then set the attribute.
    
    2. If it's a `NodePart`, get the value (trampoline thunks, render nested templates, etc), then either append a document fragment or set the `textContent`.

6. If this is the first render, append the result DOM to `container`.

## Use in Components

HTML templates could easily be the basis for component rendering, similar to JSX in React. A component base class can call an instance method that returns a `TemplateResult` and then apply it to the shadow root:

```javascript
class MyElement extends CoolBaseElement {

  static get observedProperties() {
    return ['message', 'name'];
  }

  title = `About lit-html`;
  body = `It's got potential.`;

  render() { return html`
    <h1>${this.title}</h1>
    <p>${this.body}</p>
  `;}
}
```

## Benefits over HTML templates

`lit-html` has basically all of the benefits of HTML-in-JS systems like JSX, like:

### Lighter weight

There's no need to load an expression parser and evaluator.

### Seamless access to data

Since template literals are evaluated in JavaScript, their expressions have access to every variable in that scope, including globals, module and block scopes, and `this` inside methods.

If the main use of templates is to inject values into HTML, this breaks down a major barrier between templates and values.

### Faster expression evaluation

They're just JavaScript expressions.

### IDE support by default

In a type-checking environment like TypeScript, expressions are checked because they are just regular script. Hover-over docs and code-completion just work as well.

### Case-sensitive parsing

Template literals preserve case, even though the HTML parser doesn't for attribute names. `lit-html` extracts the raw attribute names, which is useful for template syntaxes that use attribute syntax to set properties on elements.

## Benefits over JSX

### Native syntax

No tooling required. Understood by all JS editors and tools.

### No VDOM overhead

`lit-html` only re-renders the dynamic parts of a template, so it doesn't produce a VDOM tree of the entire template contents, it just produces new values for each expression. By completely skipping static template parts, it saves work.

### Scoped

JSX requires that the compiler be configured with the function to compile tags to. You can't mix two different JSX configurations in the same file.

The `html` template tag is just a variable, probably an imported function. You can have any number of similar functions in the same JS scope, or set `html` to different implementations.

### Templates are values

JSX translates to function calls, and can't be manipulated on a per-template basis at runtime. `lit-html` produces a template object at runtime, which can be further processed by libraries like ShadyCSS.

### CSS-compatible syntax

Because template literals use `${}` as the expression delimiter, CSS's use of `{}` isn't interpreted as an expression. You can include style tags in your templates as you would expect:

```javascript
html`
  <style>
    :host {
      background: burlywood;
    }
  </style>
`
```

## Features

### Simple expressions and literals

Anything coercible to strings are supported:

```javascript
const render = () => html`foo is ${foo}`;
```

### Attribute-value Expressions

```javascript
const render = () => html`<div class="${blue}"></div>`;
```

### Safety

Because `lit-html` templates are parsed before values are set, they are safer than generating HTML via string-concatenation. Attributes are set via `setAttribute()` and node text via `textContent`, so the structure of template instances cannot be accidentally changed by expression values, and values are automatically escaped.

### Case-sensitive Attribute Names

Attribute parts store both the HTML-parsed name and the raw name pulled from the string literal. This allows extensions, such as those that might set properties on elements using attribute syntax, to get case-sensitive names.

```javascript
const render = () => html`<div someProp="${blue}"></div>`;
render().template.parts[0].rawName === 'someProp';
```

### Functions/Thunks

A function value is called with no arguments, in a try/catch block to be safe from exceptions:

```javascript
let data;
const render = () => html`foo = ${_=>data.foo}`;
```

Here, `data.foo` throws because `data` is undefined, but the rest of the template renders.

Thunks are trampolined so they can return other thunks.

### Arrays/Iterables

```javascript
const items = [1, 2, 3];
const render = () => html`items = ${items.map((i) => `item: ${i})}`;
```

```javascript
const items = {
  a: 1,
  b: 23,
  c: 456,
};
const render = () => html`items = ${Object.entries(items)}`;
```

### Nested Templates

```javascript
const header = html`<h1>${title}</h1>`;
const render = () => html`
  ${header}
  <p>And the body</p>
`;
```

These features compose so you can include iterables of thunks that return arrays of nested templates, etc...

### Extensibility

`lit-html` is designed to be extended by more opinionated flavors of template syntaxes. For instance, `lit-html` doesn't support declarative event handlers or property setting out-of-the-box. A layer on top can add that while exposing the same API, by wrapping the `html` tag in a new tag and modifying the result.

This is accomplished by allowing the `TemplatePart`s of a template, which are responsible for setting new values into the DOM,  to be replaced with new implementations.

Some examples of possible extensions:

 * Property setting: Attribute expressions in templates could set properties on node.
 * Event handlers: Specially named attributes can install event handlers.
 * HTML values: `lit-html` sets `textContent` by default. Extensions could allow setting `innerHTML` or injecting existing DOM nodes.

## Small Size

`lit-html` is less than 1.5k minified and gzipped.

## Future Work

### Stateful Values

In order to support stateful repeat/if like `dom-repeat` and `dom-if` a value should be able to control it's rendering somewhat. TBD.

### Async Support

Promises and Async Iterables should be supported natively.

### Higher-Order Templates examples

#### `If(cond, then, else)`

An if-directive that retains the `then` and `else` _instances_ for fast switching between the two states, like `<dom-if>`.

Example:

```javascript
const render = () => html`
  ${If(state === 'loading',
    html`<div>Loading...</div>`,
    html`<p>${message}</p>`)}
`;
```

#### `Repeat(items, keyfn, template)`

A loop that supports efficient re-ordering by using item keys.

Example:

```javascript
const render = () => html`
  <ul>
    ${Repeat(items, (i) => i.id, (i, index) => html`
      <li>${index}: ${i.name}</li>`)}
  </ul>
`;
```

#### `Guard(guardExpr, template)`

Only re-renders an instance if the guard expression has changed since the last render.

Since all expressions in a template literal are evaluated when the literal is evaluated, you may want to only evaluate some expensive expressions when certain other values (probably it's dependencies change). `Guard` would memoize function and only call it if the guard expression changed.

Example:

```javascript
const render = () => html`
  <div>Current User: ${Guard(user, () => user.getProfile())}</div>
`;
```