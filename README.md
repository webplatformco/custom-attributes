# Custom Attributes

[toc]

## Motivation

A lot of reusable UI functionality is better expressed as composable traits or behaviors on existing elements, rather than whole new HTML elements (_has a_ rather than _is a_).

The platform already has this capability, through attributes.
Imagine if global attributes like `title`, `popover`, `lang`, `hidden` had to be implemented as elements.
Yet, this is the only tool web component authors have today.

Additionally, certain web components have an attribute counterpart to link them to another element (or link _to_ them from another element).
A native example here is `<input list>` and `<datalist>`. While `<datalist>` could be implemented as a web component, there is no authorland counterpart for the `list` attribute.

Besides the philosophical data modeling argument, overcomponentization introduces tangible problems. 
Unlike framework components that can compile to much shallower DOM trees, inserting a whole new element in the DOM has a **cost**. 
It affects **selector matching, DOM traversal, styling**, and many other things.

Inserting a custom element is not even allowed in all contexts.
Consider this:

```html
<sortable-table>
<table>
	<thead><!-- elided --></thead>
	<tr>
		<td-value value="0.5">
			<td>Half</td>
		</td-value>
	</tr>
</table>
</sortable-table>
```

Even when wrapping an element to add additional functionality is a viable solution, the ergonomics are considerably worse, with a significantly lower [signal-to-noise ratio](https://lea.verou.me/blog/2025/user-effort/#signal-to-noise). Compare:

```html
<dt-format type="relatve"><time datetime="2025-11-15"></time></dt-format>
```

with:
```html
<time datetime="2025-11-15" dt-format="relative"></time>
```

Being able to define custom attributes that can be used on any element also addresses several pain points around extending built-ins,
which was one of the most prominent pain points around Web Components per State of HTML 2025.
Authors can simply do `<button my-button>` rather than having to define their own `<my-button>` component that emulates or wraps buttons, introducing a ton of complexity.

## Prior art

### Related proposals

- [Minimal custom attributes extending `Attr`](https://github.com/WICG/webcomponents/issues/1029#issuecomment-3597708609) by @keithamus
	- [Reflection via `attr.value`](https://github.com/WICG/webcomponents/issues/1029#issuecomment-2455166850)
- [Proposal: Custom attributes for all elements, enhancements for more complex use cases](https://github.com/WICG/webcomponents/issues/1029) by @leaverou 
- [Custom Element Features, Built-in Enhancements, Itemscope Managers](https://github.com/WICG/webcomponents/issues/1000)
- [Original custom attributes proposal (naming only)](https://github.com/whatwg/html/issues/2271) by @leaverou 
- [Element Behaviors](https://github.com/lume/element-behaviors) by @lume
- [@lume's custom attributes proposal](https://github.com/lume/custom-attributes) by @lume

### Userland

- [VueJS custom directives](https://vuejs.org/guide/reusability/custom-directives)
- [Angular directives](https://angular.dev/guide/directives/attribute-directives)
- [Svelte attachments](https://svelte.dev/docs/svelte/@attach)
- [SolidJS custom directives](https://www.solidjs.com/tutorial/bindings_directives)
- [Alpine.js custom directives](https://alpinejs.dev/advanced/extending)

### Other

- [Custom attributes for all elements (TPAC 2025 breakout)](https://github.com/w3c/tpac2025-breakouts/issues/46)

## Design principles

Generalizing the [PoC](https://www.w3.org/TR/design-principles/#priority-of-constituencies) as [consumers > producers](https://lea.verou.me/blog/2025/user-effort/#consumers-over-producers), we end up with this expanded PoC: 
1. End-users 
2. HTML authors 
3. Custom attribute authors 
4. Implementors 
5. Spec authors 
6. Philosophical purity

## Design decisions

> [!Important]
> These boxes are used for conclusions, based on the prose before them.

### How to specify?

The prevailing pattern seems to be defining a subclass of `Attr`.

This has several benefits:
- Existing API to use (e.g. `this.ownerNode` to refer to the host element)
- Existing mental model around "upgrading" nodes
- Because attribute nodes are accessible via `element.attributes`, this also provides a clash-free way to hang methods and other values.
- `Attr` is even an `EventTarget` so in theory attributes could even dispatch events

There are also some downsides:
- `Attr` is an old API, and comes with baggage. E.g. now we need to define how to handle namespaces too.

> [!Important]
> Despite drawbacks, extending `Attr` seems like a very internally consistent solution and solves many problems.

### Reflection

The proposal that hosted most of the discussion proposed handling attribute-property reflection as well, as this is a big pain point when using WC APIs directly.

However, this opens this up to a lot of API design complexity and increases the API surface, while a custom attributes API can ship without it and still cover use cases.

> [!Important]
> Let’s defer handling attribute-property reflection and provide sufficient low-level primitives to allow authors to make their own decisions.

### Which `Attr` property stores the JS-facing value?

Even if authors handle attribute-property reflection themselves, at the very least there needs to be a designated slot to hold the reflected value (which by default would be a string mirroring the attribute value).

Keith [proposed](https://github.com/WICG/webcomponents/issues/1029#issuecomment-2455166850) simply specifying accessors on `attr.value`:

```js
class extends Attr {
  get value() {
    return Number(super.value)
  }
  set value(value) {
    super.value = value;
  }
}
```

While elegant, this approach has several downsides:
- Internal consistency: No built-in attributes work that way. In fact, `Attr.prototype.value` is [defined](https://dom.spec.whatwg.org/#dom-attr-value) to be a string.
- In many cases there are very big differences between the JS-facing value and its string representation.
For example, consider the `style` attribute and `element.style`, which is a whole object!
- Even when conversion is idempotent, we want to avoid any roundtrips that are not absolutely necessary, since these conversions are not always cheap.
- While `Attr` is not very widely used directly, being a very old API means there can be any number of scripts depending on `attr.value` being a string.

> [!Important]
> Let’s use a separate property (e.g. `data`, `parsed`, etc) to hold the converted value. `Attr` would define it as an accessor over `this.value`, but authors can override it so that it does different things. 

This approach allows authors to decide for themselves what the source of truth would be.
If they'd prefer, they can even define `data` it as a class field, with `value` being the accessor that proxies it.

### API surface

Many native features add methods etc to the element.
E.g. the `popover` attribute also adds `showPopover()`.

However, just like reflection, trying to specify this adds additional complexity, and is not strictly necessary:
with the model of `Attr` subclasses, authors can always hang methods on their `Attr` subclass, and they will be accessible via `element.attributes.attrName.methodName()`.

Authors can use additional JS features to improve ergonomics, such as [first-class protocols](https://github.com/tc39/proposal-first-class-protocols), [decorators](https://github.com/tc39/proposal-decorators),
or even monkey-patching, at their own risk. 

> [!Important]
> It doesn't look like we need a primitive for this.

If we want to make things easier, we *could* have a lifecycle hook for registration that lets authors react to the attribute being registered on an element.

```js 
class MyAttr extends Attr {
	// elided 
	
	static whenDefined(ElementConstructor) {
		console.log("hey " + ElementConstructor.name);
	}
}

HTMLInputElement.customAttributes.define("my-attr", MyAttr);
// prints "hey HTMLInputElement"
```

### Scoping

Some proposals involve a global `customAttributes` registry, while in others `customAttributes` is a property of specific element classes, with `HTMLElement` serving as the global one.

Many use cases only involve specific element types and don't make sense in the global scope.

Additionally, the **same attribute** name may have entirely different meanings depending on the context (e.g. `for` is often used generically for element linking, and can mean completely different things).

Global attributes would need to also work on SVG, MathML etc, which could delay the entire feature if global attributes are the MVP we go with.

Additionally, as @annevk [points out](https://github.com/WICG/webcomponents/issues/1029#issuecomment-3597830700):
> `CustomElementRegistry` can be scoped to documents, shadow roots, and **elements**. And `document.customElementRegistry` is probably what we want to mimic for anything new. Not sure we should add another global accessor for this.

Given the number of use cases around specific element types and the complexity of handling SVG at this early stage, it seems prudent to **scope to specific element classes**.

### Same attribute on multiple element types

There are many use cases where **the same attribute needs to apply to multiple element types**, without it being global.
Examples abound in the platform: `href`, `src`, several form control attributes, loading attributes like `loading` or `crossorigin`, etc.

Therefore, it should be possible to **register the same attribute to multiple classes**, since it is not always feasible to use inheritance to register an attribute on multiple elements.

E.g. consider a `persist-value` attribute that is placed on form controls to persist their values in localStorage whenever they are edited.
We may want to register it on built-ins like `HTMLInputElement`, `HTMLTextAreaElement`, `HTMLSelectElement` by default:

```js 
// persist-value.js
export class PersistAttr extends Attr {
	// elided
}

// Add to native form elements by default
HTMLInputElement.customAttributes.define("persist-value", PersistAttr);
HTMLTextAreaElement.customAttributes.define("persist-value", PersistAttr);
HTMLSelectElement.customAttributes.define("persist-value", PersistAttr);
```

Then, consumers may want to additionally register it on custom form controls they use:
```js 
import { PersistAttr, RangeSlider } from "./attrs/persist-value.js";

RangeSlider.customAttributes.define("persist-value", PersistAttr);
```

### Naming

Originally, **hyphens** were suggested, as a way to mirror custom element naming rules. 
However, there are many exceptions in the web platform making this a bit awkward, e.g.:

- A ton of SVG presentation attributes (e.g. `fill-opacity`)
- `aria-*`
- `allow-charset`
- `http-equiv`

Additionally, many custom element attributes use hyphens.
On two different social media polls, about half of authors voted that they prioritize readability over [platform consistency](https://www.w3.org/TR/design-principles/#naming-consistency) (which recommends concatcase):
- https://x.com/LeaVerou/status/1863812496106389819
- https://front-end.social/@leaverou/113587139842885936

<!-- 
Additionally, as @jakearchibald [pointed out](https://github.com/WICG/webcomponents/issues/1029#issuecomment-2454991200), even attributes without dashes often camelCase in JS, which would make reflection clash (e.g. `readonly` → `readOnly`).
Of course that is not a problem if we don't handle reflection automatically. 
-->

Another option would be a specific **prefix**, though given that the attribute name itself often needs to be namespaced with a library prefix, this would only produce acceptable ergonomics if very short, e.g. a single character (`#`, `$`, `:`, `@` etc).

### Lifecycle hooks

What does `connectedCallback` etc mean in the context of an attribute?
Do they still correspond to the *element* being connected, or the *attribute* being specified on the element? Or when both are true?

@DeepDoge [makes a good case](https://github.com/WICG/webcomponents/issues/1029#issuecomment-3599188181) for the latter:

> IMO custom attributes are composable behavior units, kind of like a superset of extended custom elements. So, `connectedCallback()` should run only when both are true:
> 
> * The attribute is attached to an element
> * That element is connected to the DOM
> 
> If we simplify it even more, it should trigger when the attribute is connected to the DOM, not when attribute is connected to an element.
> 
> Just like how a custom element's `connectedCallback()` gets triggered when the element is actually connected to the DOM, not when it has a parent.
> 
> The whole point of `connectedCallback()` / `disconnectedCallback()` is to initialize or clean things up depending on whether the thing (element or attribute) is live in the DOM.
> 
> So should it be called "when the attribute is connected, or when the element is connected?": It should be called when the attribute is connected to the DOM, which also requires element its connect to be connected to the DOM as well. I think we can all agree that "Connected" means "Connected to the DOM".

We could probably define similar semantics for other lifecycle callbacks (`disconnectedCallback`, `adoptedCallback` etc), though they may be less straightforward.

It would be good to do a review of use cases to see how common it is to need lifecycle hooks around the element itself, that are separate from those of the attribute node. Assuming these are niche, they can always be addressed via `MutationObserver` improvements down the line (e.g. [observing connectedness is already an open feature request](https://github.com/whatwg/dom/issues/533))



### Traits involving multiple attributes

While not MVP, there are many use cases where a feature utilizes multiple attributes.
A common pattern is when one attribute enables the feature, and the rest customize it.

For example, in the web platform there is `<template shadowrootmode="open">`, but also several `shadowroot*` attributes that provide parameters (`shadowrootdelegatesfocus` etc).

Another pattern is where multiple attributes work together to specify a DSL. For example [Vue directives](https://vuejs.org/api/built-in-directives.html) (`v-if`, `v-for`, `v-on` etc).

Therefore, a nice-to-have would be to have a way for the attribute to react to attribute changes of *other* attributes on the element, possibly by reusing `observedAttributes` and `attributeChangedCallback()`.

## Current Proposal

Putting all of the above together gives us a strawman to facilitate discussion.

New static members on `HTMLElement`:
- `customAttributes` of type `CustomAttributeRegistry`, with the same methods as `CustomElementRegistry` (we may want to define a `Registry` superclass that they both inherit from). This is a distinct instance per subclass, and an element recognizes attributes registered on any of its superclasses.

New static members on `Attr`:
- `observedAttributes` with the same semantics as on `HTMLElement`

Custom attributes are defined as a subclass of `Attr`.
Lifecycle callbacks are available:
- `connectedCallback`: Executed when the attribute is present on the element and `this.ownerElement.isConnected` is true.
- `disconnectedCallback`: Executed when the attribute is no longer connected (see above)
- `connectedMoveCallback()`: TBD
- `adoptedCallback()`: Fired when the attribute node is moved to another element (e.g. via `setAttributeNode`) OR `ownerElement` is moved to another document.
- `attributeChangedCallback()`: Executed when any of the attributes in `this.constructor.observedAttributes` are changed, added, removed, or replaced. The attribute itself is always observed whether it’s specified in `observedAttributes` or not.