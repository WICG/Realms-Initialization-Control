# Same Origin Regulation Policy [PROPOSAL]

## Authors:

- [Gal Weizman](https://github.com/weizman)

## Participate

- [Issue tracker](https://github.com/weizman/Same-Origin-Regulation-Policy/issues)

## Table of Contents

TBD

## "What's Same Origin Regulation Policy?"

This is a proposal for some API serving a **security** need to allow programmers to regulate and control "non-top same-origin realms" 
(also known as iframes, tabs, windows, etc) in terms of how they shape up when being created within a certain application.

The most basic premise for this proposal is that an application should have the **privilage** to fully configure its environment
to **any desired resolution** before loading any other internal/external JS code - **for whatever reason**.

With that practice, apps can prevent certain operations, monitor them, virtualize them to behave differently for alternating purposes
such as defending their application and more - **to a resultion current security APIs (e.g. CSP) do not offer.**

## "What's the problem then? That's what JavaScript's for!"

While it's true this can (and should) be accomplished with JavaScript, with what the web can currently offer us,
the code we create to express this practice **only applies to the context of the application itself.**

So in a scenario where an iframe is loaded into the application in its same origin, our code **won't run
in the context of the iframe** automatically the same way it did at the top level context.

**_This_** is the problem we wish to propose a solution for - we want the code we create for shaping
what the app can or can't do to **apply automatically to all potential contexts of the application** (so this practice is actually useful!).

## "Not sure we understand..."

Fair enough. Best explain with an example. Consider the following application, `alert-is-not-allowed.com`:

```html
<html>
  <head>
    <title> alert-is-not-allowed.com </title>
    <script>
      window.alert = function tamedAlert(m) {
        console.log(
          'NOTICE:',
          'alert messages are not welcome here, as we find them offensive!',
          'for your convenience, we log the message to console instead of alerting it:', m
        );
      }
    </script>
  </head>
  <body>
    <h1> Welcome to <u> alert-is-not-allowed.com </u> - where alerts are not allowed! </h1>
  </body>
</html>
```

`alert-is-not-allowed.com` hate alert messages, so they have their own app where they use JavaScript
to express their decision to forbid alert messages in the page.

A fair question would be: 

> "_But why do they need to disable it using JavaScript? Can't they just not use it?_"

Ideally, that would be true. But in reality, web apps are mainly composed by a lot of **code they 
don't actually write, control or even aware of!**

It is only justified that `alert-is-not-allowed.com` will use code to express their desired
limitations, in an environment where they **cannot truly trust all the code they ship.**

Did that last sentence sound absurd to you? Well, this is the **current landscape of 
supply chain driven development** - and that's **not going to change anytime soon**.

Because in reality, `alert-is-not-allowed.com` being a complex web app, would probably look more like this 
(taken from https://edition.cnn.com/sport):

![Screenshot 2023-12-07 at 17 43 42](https://github.com/weizman/Same-Origin-Regulation-Policy/assets/13243797/bb8d086c-6db2-4905-bda1-fd3f53e17b9a)

A live DOM including an **endless list of scripts flying in from everywhere** 
with an ability to constantly update **without asking for permission** whatsoever!

So since `alert-is-not-allowed.com` have no control over what these scripts might do,
its **only option** for disabling `alert` messages is by redefining their behaviour using JavaScript.

## "Ok, well.. if JavaScript solves this, what's the problem then?"

If only that was this simple. The problem is, JavaScript code can give birth to new contexts (aka realms)
where each context has its own set of DOM, APIs, execution environment, and more (learn more about realms [here](https://weizmangal.com/2022/10/28/what-is-a-realm-in-js)).

This means, that if one of those scripts `alert-is-not-allowed.com` includes decides to disobey the rules
set by the app, they can **easily** do so by simply form a new realm, and use its `alert` instance instead of the
one the app limited:

```html
  <script src="https://some-ad-service.com/ad-service.js">
    let alert = window.alert;
    if (alert.name === 'tamedAlert') {
      const ifr = document.createElement('iframe'); // iframes introduce new realms
      alert = document.body.appendChild(ifr).contentWindow.alert;
    }
    alert('I refuse to play by your rules - this shall successfully pop an alert message!');
  </script>
```

And since browsers and the web currently offer no way to regulate that - 
poor old `alert-is-not-allowed.com` is left **defenseless** against the might alert boxes.

## "Enough with the drama, alert boxes aren't that big of a problem!"

Fair enough. However, this was clearly just an example, and in reality this gap
actually prevents builders from designing and implementing resilient security solutions 
for JavaScript web apps. There are so much innovative efforts waiting to fulfil their potential
and provide robust security for the web, failing to do so due to having no control over this gap.

---------


## Introduction

[The "executive summary" or "abstract".
Explain in a few sentences what the goals of the project are,
and a brief overview of how the solution works.
This should be no more than 1-2 paragraphs.]

## Goals [or Motivating Use Cases, or Scenarios]

[What is the **end-user need** which this project aims to address?]

## Non-goals

[If there are "adjacent" goals which may appear to be in scope but aren't,
enumerate them here. This section may be fleshed out as your design progresses and you encounter necessary technical and other trade-offs.]

## User research

[If any user research has been conducted to inform the design choices presented
discuss the process and findings. 
We strongly encourage that API designers consider conducting user research to
verify that their designs meet user needs and iterate on them,
though we understand this is not always feasible.]

## [API 1]

[For each related element of the proposed solution - be it an additional JS method, a new object, a new element, a new concept etc., create a section which briefly describes it.]

```js
// Provide example code - not IDL - demonstrating the design of the feature.

// If this API can be used on its own to address a user need,
// link it back to one of the scenarios in the goals section.

// If you need to show how to get the feature set up
// (initialized, or using permissions, etc.), include that too.
```

[Where necessary, provide links to longer explanations of the relevant pre-existing concepts and API.
If there is no suitable external documentation, you might like to provide supplementary information as an appendix in this document, and provide an internal link where appropriate.]

[If this is already specced, link to the relevant section of the spec.]

[If spec work is in progress, link to the PR or draft of the spec.]

## [API 2]

[etc.]

## Key scenarios

[If there are a suite of interacting APIs, show how they work together to solve the key scenarios described.]

### Scenario 1

[Description of the end-user scenario]

```js
// Sample code demonstrating how to use these APIs to address that scenario.
```

### Scenario 2

[etc.]

## Detailed design discussion

### [Tricky design choice #1]

[Talk through the tradeoffs in coming to the specific design point you want to make.]

```js
// Illustrated with example code.
```

[This may be an open question,
in which case you should link to any active discussion threads.]

### [Tricky design choice 2]

[etc.]

## Considered alternatives

[This should include as many alternatives as you can,
from high level architectural decisions down to alternative naming choices.]

### [Alternative 1]

[Describe an alternative which was considered,
and why you decided against it.]

### [Alternative 2]

[etc.]

## Stakeholder Feedback / Opposition

[Implementors and other stakeholders may already have publicly stated positions on this work. If you can, list them here with links to evidence as appropriate.]

- [Implementor A] : Positive
- [Stakeholder B] : No signals
- [Implementor C] : Negative

[If appropriate, explain the reasons given by other implementors for their concerns.]

## References & acknowledgements

[Your design will change and be informed by many people; acknowledge them in an ongoing way! It helps build community and, as we only get by through the contributions of many, is only fair.]

[Unless you have a specific reason not to, these should be in alphabetical order.]

Many thanks for valuable feedback and advice from:

- [Person 1]
- [Person 2]
- [etc.]
