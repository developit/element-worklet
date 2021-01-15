# Element Worklet

_Created:_ 2018-09-20 | _Updated:_ 2021-01-15

This proposal outlines a (potentially impossible!) idea for moving entire Elements off the main thread.

Conceptually, think [OOPIFs](https://www.chromium.org/developers/design-documents/oop-iframes) except well-suited to first-party content,
and implemented as lightweight [Worklets](https://developer.mozilla.org/en-US/docs/Web/API/Worklet).

## On the UI Thread:  Element Worklet

The Custom Elements API is extended with support for registering Element Worklets. As with all
Worklets, registration is done via a URL resolving to a Module Script. A single Element Worklet
script can register multiple Custom Elements.

```js
customElements.addModule('./lazy-image.mjs');
```

All Custom Elements controlled by a given Worklet are upgraded once the Worklet has been initialized,
following the same lifecycle as Custom Elements defined on the main thread.

Aside from the URL-based Worklet script registration and instantiation, Custom Elements defined using
Element Worklet function like a standard Custom Element:

```html
<!-- declarative: -->
<lazy-image src="cat.jpeg">

<script>
  // imperative:
  const dog = document.createElement('lazy-image');
  dog.setAttribute('src', 'dog.png');
  document.body.appendChild(dog);
</script>
```

### Pros & Cons

<table>
<thead>
<tr>
<th><strong>Pros</strong></th>
<th><strong>Cons</strong></th>
</tr>
</thead>
<tbody>
<tr>
<td><ul>
  <li>Re-uses existing Element registry</li>
  <li>Leverages Custom Element semantics</li>
  <li>Well-defined lifecycle and upgrade model</li>
</ul></td>
<td><ul>
  <li>Limited DOM API surface</li>
  <li>No declarative registration **</li>
</ul></td>
</tr>
</tbody>
</table>

> \*\* However, Element Worklet lends itself more to declarative registration than straight Custom Elements.

## Inside an Element Worklet

Element Worklets build on the same foundation as the other existing types of Worklets.
The module URL given to `customElements.addModule()` is immediately fetched and evaluated within a new
`ElementWorkletGlobalScope`. This may occur on the rendering thread, in a background thread or a thread pool.

```js
// lazy-image.mjs
class LazyImageElement extends WorkletElement {
  constructor() {
    // Light DOM is accessible to the worklet:
    this.getAttribute('src')  // cat.jpeg

    // ...but it is read-only
    this.firstElementChild.src = 'foo'  // Error

    // and scoped to the element (no parent references)
    this.parentNode  // null

    // in fact, there are no meaningful instance properties defined on WorkletElements at all.
    // to facilitate serialization, all data must flow through attributes or MessageChannel:

    // there's a corresponding `lazyImage.port` on the main thread representation.
    this.port.postMessage('hello');

    // As such, Shadow DOM is required in order to produce UI:
    this.shadow = this.createShadow();
    this.shadow.innerHTML = 'some stuff';
  }

  // all lifecycle functions the same as Custom Elements v1:

  static get observedAttributes() {
    return ['src']
  }

  attributeChangedCallback(name, oldValue, newValue) {}
  connectedCallback() {}
  disconnectedCallback() {}
}

// Registering an Element is the same as on the main thread:
customElements.define('lazy-image', LazyImageElement);
```

**Data Flow:** Custom Element properties are not reflected on a WorkletElement, only attributes.
Attribute changes are observed the same as they are in Custom Elements, defined eagery by a static `observedAttributes` property on the element's constructor.
Attribute changes changes invoke `attributeChangedCallback()` in the worklet, and may be batched.

**Data Sharing:** The main thread and worklet instances of a Custom Element each have a `.port` property, which are ports of an instance-specific `MessageChannel` that can be used for message passing. This is synonymous with the `processor.port` interface used by AudioWorkletNode/AudioWorkletProcessor.

## Element Upgrade Process

When a Custom Element defined by an ElementWorklet is instantiated on the main thread,
a new corresponding WorkletElement instance is created within its associated `ElementWorkletGlobalScope`:

1. all unknown elements begin marked as unresolved (`HTMLUnknownElement` / `:not(:resolved)`)
2. `customElements.addModule()` is invoked and an ElementWorklet instantiated
3. `customElements.define()` is invoked within the Worklet
4. Elements in the document with the given Custom Element name are enqueued for upgrade

Upgrading Worklet-backed Elements is a two-phase process. First, the Element representation is
created on the main thread and a MessagePort is created for it, with one port assigned as a `port`
instance property. At this point, the Element is still considered to be in an unresolved state.
Then, inside the Worklet, a corresponding WorkletElement is instantiated and assigned the other
message port.

When a Worklet-backed Element is connected (via `appendChild()`, the parser or another means),
the WorkletElement instance's `connectedCallback()` lifecycle method is invoked.
As with a standard Custom Element, `attributeChangedCallback()` is also invoked with the initial value of each observed attribute.
Once all the callbacks have been invoked, the Element's representation in the parent document is marked as resolved and the process is complete.

## Use Cases

**Ads:** Advertisements currently use iframes for encapsulation, a technique of increasing cost as
the effects of Spectre mitigations make their way into browsers.  Element Worklet could provide a
lightweight alternative to iframes for this use-case.

**Third Party Embeds:**  Embedded content like comment widgets, chats and helpdesks all of these
currently use some combination of same-origin scripting and iframes, usually mixing origins
(eg: a script in the embedder context communicating with an iframe from the embedee's context).
Moving from `<script>` + `<iframe>` to Element Worklet seems like a reasonable fit for this case.

**AMP:** The semantics defined in this proposal map reasonably well to `<amp-script>`, and a prototype
of Element Worklet has been built using `worker-dom`, the library that underpins `<amp-script>`.
AMP's approach is much more broadly applicable than Element Worklet, seeking to support arbitrary
third-party code running in a sandboxed DOM environment. However, it's possible a solution like worker-dom
would be able to leverage something like Element Worklet to simplify Element registration and upgrades,
and to mitigate transfer overhead between threads.

**Lazy Loading:** Component-based frameworks and libraries strive to provide solutions for lazily
downloading, instantiating and rendering portions of an application. This process is entirely
implemented in userland, which has the unfortunate side effect of making it invisible to the browser.
In certain scenarios, it may be possible to use Element Worklet as the underly mechanism for lazily
loading and rendering pieces of a component-based User Interface.

**UI Component Libraries:** If this model can be shown to provide performance guarantees for Element
registration, upgrade and rendering, it's possible a UI library would choose to provide their components
as worklet-backed Elements through the use of one or more Element Worklets. This could have interesting
implications for performance, since it would provide a way to impose performance guarantees. This is a
safety net developers do not currently have for prebuilt modules. The (large) portions of a typical app
that are defined by code installed from npm would have less ability to negatively impact the performance
of first-party code.

## Open Questions

- What DOM APIs are available from within an ElementWorkletGlobalScope?
- Can `WorkletElement` provide a sufficient API surface to allow current libraries and approaches to be
  reused with minimal modification?
- Is the level of encapsulation illustrated in this proposal too limiting?  Does it fail to meet the
  needs of use-cases like embedded video players?
- Should Element Worklet provide an analog for Custom Element property getters/setters? Could custom
  properties (and/or methods?) defined on a WorkletElement subclass be reflected asynchronously to the
  main thread (in the style of [Comlink](https://github.com/GoogleChromeLabs/comlink)? Without this,
  complex data types for Worklet-backed elements would need to be serialized as attributes.
- Would it be possible to allow defining a priority when registering Element Worklet modules?
  This would unlock a host of use-cases in which worklet code could be considered untrusted.
