+++
date = '2025-09-15T8:00:00+08:00'
draft = false
title = 'Morden Javascript Tutorial - An Introduction'
tags = ['JavaScript']
+++

## An Introduction to JavaScript
Let’s see what’s so special about JavaScript, what we can achieve with it, and what other technologies play well with it.

### Why is it call JavaScript?

JavaScript initially called "Live Script". But Java was popular at that time, so it was decided that positioning a new language as a "younger brother" of Java would help.
But as it evolved, JavaScript became a fully independent language with its won specification called [ECMAScript](http://en.wikipedia.org/wiki/ECMAScript), and now it has no relation to Java at all.

Today, JavaScript can execute on any device that has a special program called [the JavaScript engine](https://en.wikipedia.org/wiki/JavaScript_engine).
The browser has an embedded engine called a "JavaScript virtual machine".

Different engines hav different "codenames":
- [V8](https://en.wikipedia.org/wiki/V8_(JavaScript_engine)): in Chrome, Opera and Edge
- [SpiderMonkey](https://en.wikipedia.org/wiki/SpiderMonkey): in Firefox


### What can in-browser JavaScript do?
Morden JavaScript is a "safe" programming language.
It does not provide low-level access to memory or CPU, because it was initially created for browsers which do not require it.

Node.js supports functions that allow JavaScript to read/write arbitrary files, perform network requests, etc.
In-browser JavaScript can do everything related to webpage manipulation, interaction with the user, and the webserver.

- Add new HTML to the page, change the content and modify styles
- React to user actions like mouse clicks
- Send network requests
- Get and set cookies
- Remember the data on the client-side

### What can't in-browser JavaScript do?
For safety reasons, it can't:  
- Limited access to files.  
  No access to OS functions.  
  Require explicit permission with camera/microphone devices.

- Different tabs/windows generally do not know about each other.  
  This is called "Same Origin Policy" (同源策略).

- Easilly communicate over the server where the current page came from.  
  Its ability to receive data from other sites/domains is crippled.  
  It requires explicit agreement from the remote side.

Such limitations do not exist if JavaScript is used outside of the browser, for example on a server.


### What makes JavaScript unique?
- Full integration with HTML/CSS
- Simple things are done simply
- Supported by all major browsers and enabled by default

JavaScript is the only browser technology that combines these three things.


### Languages "over" JavaScript
The syntax of JavaScript not suit everyone's needs. Recently a plethora of new languages appered, which are transpiled to JavaScript before they run in the browser.

- [CoffeeScript](https://coffeescript.org/) is "syntactic sugar" for JavaScript. It introduces shorter syntax, allowing us to write clearer and more precise code. Ruby devs like it.
- [TypeScript](https://www.typescriptlang.org/) is concentrated on add "strict data typing" to simplify the development and support of complex systems. It's developed by Microsoft.
- [Flow](https://flow.org/) also adds data typing, but in a different way. Developed by Facebook.
- [Dart](https://www.dartlang.org/) is a standalone languages that has its own engine that runs in non-browser environments, but also can transpiled to JavaScript. Developed by Google.
- [Brython] is a Python transpiler to JavaScript but enables the writing of applications in pure Python without JavaScript.
- [Kotlin] is a modern, concise and safe programming language that target the browser or Node. Developed by JetBrains.

## Manuals and Specifications

### Specification
[The ECMA-262 specification](https://www.ecma-international.org/publications/standards/Ecma-262.htm) contains the most in-depth, detailed and formalized information about JavaScript.

A new specification version is released every year. Between these releases, the latest specification draft is at [https://tc39.es/ecma262/](https://tc39.es/ecma262/).

To read about new bleeding-edge features, including those that are “almost standard” (so-called “stage 3”), see proposals at [https://github.com/tc39/proposals](https://github.com/tc39/proposals).

### Manuals
[MDN (Mozillz) JavaScript Reference](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference) is the main manual with examples and other information.
It's great to get in-depth information about individual language functions, methods etc.

### Compatibility tables
JavaScript is a developing language, new features get added regularly.

To see their support among browser-based and other engines, see:

- [https://caniuse.com](https://caniuse.com) – per-feature tables of support, e.g. to see which engines support modern cryptography functions: [https://caniuse.com/#feat=cryptography](https://caniuse.com/#feat=cryptography).
- [https://kangax.github.io/compat-table](https://kangax.github.io/compat-table) – a table with language features and engines that support those or don’t support.
