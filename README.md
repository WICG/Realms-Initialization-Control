# R.I.C.A

> _**R**ealms **I**nitialization **C**ontrol **A**PI proposal (to introduce **security** to same origin realms in web applications)_

The **R**ealms **I**nitialization **C**ontrol **A**PI (will be refered to as **RICA**) should allow developers to securely tap into the creation moment of same origin realms within their web application in order to tame and control them.

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
* Realms - the technical terminology for the execution environment provided in JavaScript along with a distinct set of capabilities mostly reachable by a representing global object. In the browser environment, the global object is accessible via `window`. Realms can be formed in various ways in the browser ecosystem, most commonly by creating `iframe`s, opening tabs or popup windows using `open` API, using `Worker`s and more. Each of those have its own set of capabilities, its own execution environment and its own global object - its own realm ([learn more](https://weizmangal.com/2022/10/28/what-is-a-realm-in-js/)https://weizmangal.com/2022/10/28/what-is-a-realm-in-js/).
