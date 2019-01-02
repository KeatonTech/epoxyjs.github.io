---
description: >-
  Epoxy's built-in DOM binding functions allow you to create Epoxy-driven web
  apps your way, without needing to learn a whole framework
---

# Epoxy DOM

## Basic DOM Binding

At the most basic level, Epoxy DOM lets you bind computed values from Epoxy Models to various properties of an HTML element. These functions are essentially thin wrappers around the `computed` function that use the resulting observable to update the DOM.

#### The following binding functions are available from the `epoxyjs/dom` package.

```typescript
/**
 * Attaches the result of a listenable computation to a specified attribute
 * of an Element.
 */
function bindAttribute(element: Element, attribute: string, compute: ()=>any) {}

/**
 * Attaches the result of a listenable computation to a specified CSS style
 * of an Element.
 */
function bindStyle(element: HTMLElement, property: string, compute: ()=>string) {}

/**
 * Attaches the result of a listenable computation to the HTML content of
 * an element. This should be used sparingly as it can easily be exploited for
 * XSS vulnerabilities.
 */
function bindInnerHTML(element: Element, compute: ()=>string) {}

/**
 * Attaches the result of a listenable computation to the presence of a given
 * class on an Element. When the computation evaluates to true, the class will
 * be present on the element, and likewise will be excluded when the computation
 * evaluates to false.
 */
function bindClass(element: Element, className: string, compute: ()=>boolean) {}

/**
 * Creates children for a given element as dictated by a listenable array.
 * Adding items to the array will add children, removing them will remove children.
 */
function appendChildrenFor<T>(
    element: Element,           // Element whose children to replace
    list: Listenable<Array<T>>, // Listenable list that will drive the binding
    render: (T)=>Element        // Function that creates elements
) {}

/**
 * Adds each child in a listenable array to this element. Children will be added
 * or removed as items are added or removed from the list.
 */
function appendBoundChildren(
    element: Element,                 // Element whose children to replace
    list: Listenable<Array<Element>>, // Listenable list that will drive the binding
) {}
```

These methods make it easy to add Epoxy functionality to an existing Javascript UI, including one using a different UI framework. It has one caveat though, and it's that by default the bindings will not get cleaned up when the element is removed from the DOM. This can lead to a fairly large memory leak, so Epoxy keeps track of element subscriptions and allows them to be removed with the `removeElement(el: Element) {}` function \(also available on `epoxyjs/dom`\). This function will clean up any live bindings on the element and any of its children.

Of course, it's easy to forget to do this, especially when an external framework is involved. Luckily, Epoxy DOM provides an easier API for constructing UIs that calls removeElement automatically.

## Building UIs with EpoxyDomElement

`EpoxyDomElement` is a builder-style class that wraps a normal HTMLElement but integrates automatically with the `removeElement` function and provides some handy syntactic sugar. Let's look at an example:

```typescript
const model = makeListenable({
    isLoggedIn: false,
    userName: 'Belinda'
});

new EpoxyDomElement('main') /** This argument is the HTML tag name */
    .setAttribute('label', 'The main app body')
    .appendConditionalChild(() => model.isLoggedIn, () => {
        new EpoxyDomElement('h1')
            .bindInnerHTML(() => `Hello ${model.userName}!`)
    });
    .appendConditionalChild(() => !model.isLoggedIn, () => {
        new EpoxyDomElement('h2')
            .setInnerHTML(() => 'Please log in')
    });
    .appendTo(document.body);  
```

This snippet creates an element that displays a greeting to logged-in users and a login message to non-logged-in users, and then appends that element to the body. The view will update as the model changes, and unsubscribe bindings when elements are deleted \(for example when model.isLoggedIn changes and one of the header elements is replaced with the other\).

While this syntax certainly isn't as elegant as something like NG Templates or JSX, it gets the job done without any custom compilation steps, and doesn't require learning a whole new markup language.

{% hint style="info" %}
EpoxyDomElement is fully compatible with SVG elements, as well as HTML.
{% endhint %}

## Custom elements with EpoxyDomComponent

Epoxy DOM's concept of 'components' is much lighter-weight than that of most other JS libraries. It is possible to use them in the same way as you'd use a native web component, but that is not enforced. An Epoxy component is just a subclass of EpoxyDomElement that contains an encapsulated style property.

```typescript
class MyButtonComponent extends EpoxyDomComponent {

    /**
     * This style property is automatically added to document.head when the first
     * instance of this class is initialized.
     */
    static style = `
        :host {
            background: #eee;
            padding: 20px;
            border: 1px solid #333;
        }
    `;
    
    constructor(inputs: Listenable<{
        title: string,
        label: string,
    }>) {
        super('button');
        
        this
            .bindAttribute('aria-label', () => inputs.label)
            .bindInnerHtml(() => inputs.title)
    }
    
    onClick(): Observable<undefined> {
        return new Observable((observer) => {
            this.addEventListener('click', observer.next());
        });
    }
}

/** Adds a new button to the body */
const button = new MyButtonComponent(makeListenable({
        title: 'Click for an alert',
        label: 'Button that makes an alert box',
    })
    .appendTo(document.body);

/** Listens for click events on the button. */
button
    .onClick()
    .subscribe(() => alert('clicked'));
```

This example demonstrates setting a style, passing in inputs, and listening for events using Epoxy DOM Components. It should not be taken as the canonical way of doing things, however. Epoxy DOM Components are just normal Javascript classes and so you can format them however you'd like.

### CSS Encapsulation

One common problem in modern web development is preventing CSS styles from one part of your app from messing with the look of another part of your app. It is hard to enforce rules preventing developers from adding overly-broad selectors \(eg. `a:hover`\), and there's always the possibility that you will accidentally use the same class name or id in two different places. The solution to this problem is CSS encapsulation, which ties styles to a particular 'component' of the app and does not allow them to affect any other component.

An Epoxy element's CSS scope extends from its root element down to its child elements until it reaches another EpoxyDomComponent, in which case that component applies its own CSS styles. Generally this will mean that an EpoxyDomComponent's styles apply to any elements it itself creates, but – since Epoxy's concept of a component is more fluid than other JS frameworks – this is not always the case. For example, if another component appends a new element to your component it will receive your component's CSS scope.

#### Special CSS Selectors

One thing you may notice from the previous example is the `:host` selector, which merely references the top-level element of a component, which is the one you create when you call `super()`. In this case super was called with 'button' so a `<button/>` element was created as the root. `:host` can be used either by itself or in conjunction with other selectors, such as `:host:hover` to match the top-level element whenever the user is hovering their cursor over it.

The other special CSS selector supported by Epoxy DOM is `/deep/` which breaks out of CSS encapsulation. This allows a component to override styles set by its child components.

{% hint style="info" %}
Use /deep/ sparingly, as it can make style debugging difficult. It's basically the modern equivalent of the !important declaration. Consider using [CSS variables](https://developer.mozilla.org/en-US/docs/Web/CSS/Using_CSS_variables) where possible.
{% endhint %}

