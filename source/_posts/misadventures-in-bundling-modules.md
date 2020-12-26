---
title: Misadventures in Bundling Modules
description: Native ES2015 module browser support can't come soon enough.
thumbnail: 0_51ak67whYuVXrn1G.jpg
date: 2017-10-16 20:39:18
tags:
    - Programming
    - JavaScript
---

<!-- markdownlint-disable no-inline-html -->
![Image](0_51ak67whYuVXrn1G.jpg)<span class="caption">Image: [Christopher Robin Ebbinghaus](https://unsplash.com/@cebbbinghaus)</span>
<!-- markdownlint-enable no-inline-html -->

If you are writing modular JavaScript source code using ES2015’s import keyword, you are likely transpiling your code so that it can execute in web browsers presently available to most users. As much as it is a pleasure to maintain your source code as modules, it's been a bumpy road to reach a common implementation of modules in the JavaScript execution environments themselves, resulting in a rather large and confusing set of technologies. The problem described later requires a bit of knowledge about the relationship between [UMD](https://github.com/umdjs/umd), [AMD](http://requirejs.org/docs/whyamd.html), and [Require.JS](http://requirejs.org). [This post](http://davidbcalhoun.com/2014/what-is-amd-commonjs-and-umd) by David Calhoun gives a great overview.

[An embeddable widget library](https://success.mindtouch.com/Integrations/Touchpoints), that our team develops and maintains, bundles the source JavaScript to simplify delivery to developers and integrators who rely on the library to embed in their websites. We didn't notice that we had bundled them using UMD (Universal Module Definition) syntax. This led to problems when the library was added web applications leveraging the AMD syntax, like those depending on Require.JS. Transpiling to UMD included `window.require` as part of our bundle, leading to conflicts with the DOM’s `window.require` defined by the AMD-powered web application.

We corrected our mistake by transpiling to a global format. A word of caution: be careful when transpiling module syntax as there are many options for the target output. If you don’t have control over the webpage in which your code will execute (such as the case of an embeddable widget library), it's wise to consider plain-old globals.
