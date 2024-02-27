# R.I.C.A

> _**R**ealms **I**nitialization **C**ontrol **A**PI proposal (to introduce **security** to same origin realms in web applications)_

The **R**ealms **I**nitialization **C**ontrol **A**PI (refered to as **RICA**) should allow developers to securely tap into the creation moment of same origin realms within their web application in order to tame and control them.

While from a **usability** point of view this is already natively provided by the platform by using `load` events of iframes for example:

```javascript
const iframe = document.createElement('iframe');
iframe.src = 'https://payment-iframe.com/';
iframe.style.display = 'none';
iframe.onload = () => iframe.style.display = 'block';
document.body.appendChild(iframe);
```

RICA attempts to provide this from a **security** point of view, which requires addressing same origin realms initialization significantly different.

## Table of contents
 
TBD

## Terminology

> _Feel free to skip if these are already familiar to you_

* Capabilities - APIs provided by the JavaScript and/or browser execution environment (e.g. `Array`, `Object`, `fetch`, `document`, etc).
* Realms - the technical terminology for the execution environment provided in JavaScript along with a distinct set of capabilities mostly reachable by a representing global object. In the browser environment, the global object is accessible via `window`. Realms can be formed in various ways in the browser ecosystem, most commonly by creating `iframe`s, opening tabs or popup windows using `open` API, using `Worker`s and more. Each of those have its own set of capabilities, its own execution environment and its own global object - its own realm ([learn more](https://weizmangal.com/2022/10/28/what-is-a-realm-in-js/)).

## Motivation

The problem with same origin realms (also known as the [same origin concern](https://weizmangal.com/content/pdf/The%20same%20origin%20concern.pdf)) isn't intuitive to comprehend becase it comes into play only when trying to harnest the power of virtualization in favor of providing advanced security to web apps, which isn't a common usage for JavaScript to begin with, as it primarily aimed at addressing security concerns.

However, we see the importance of such use cases - while JavaScript is a great language for creating composable software, it isn't designed well enough to do so securely, especially when adjusted to run in the browser and interact with the DOM.

And since there is a clear rise in the need for easily composing web applications due to the increase in supply chain driven development, it is important to allow so.

Unfortuantly, securing a supply chain is inherently far harder than defending against good old XSS - while the latter is a rather straight forward lack of input sanitization, telling good code from bad is more abstract when in the supply chain of an application. That's why there are so many different angles to attempt to address this issue, whether at the application's runtime or its build time.

One great way to approach this problem at runtime is by virtualization - redefining JavaScript APIs at runtime (commonly known as monkey patching) to behave similarly while hardening their security to make an effort in telling bad code from good code.

Best explained by an example.

## Example
