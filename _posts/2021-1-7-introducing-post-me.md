---
layout: post
title: "Use web Workers and other Windows through a simple Promise API "
image_dir: 2021-1-7-introducing-post-me
---

![_config.yml]({{ site.baseurl }}/images/{{ page.image_dir }}/diagram.png)

[post-me](https://github.com/alesgenova/post-me) is a typescript library that provides a simple Promise API for bidirectional communication with web workers and other windows (iframes, popups, etc.).

## 0. TLDR

With [__post-me__](https://github.com/alesgenova/post-me) it is easy for a parent (for example the main app) and a child (for example a worker) to expose methods and custom events to each other.

Main features:
- 🔁 Parent and child can both __expose__ __methods__ and/or __events__.
- 🔎 __Strong typing__ of method names, arguments, return values, as well as event names and payloads.
- 🤙 Seamlessly pass __callbacks__ to the other context to get progress or partial results.
- 📨 __Transfer__ arguments/return values/payloads when needed instead of cloning.
- 🔗 Establish __multiple__ concurrent __connections__.
- 🌱 __No dependencies__: 2kb gzip bundle.
- 🧪 Excellent __test coverage__.
- 👐 Open source (MIT): [https://github.com/alesgenova/post-me](https://github.com/alesgenova/post-me)

Below is a minimal example of using post-me to communicate with a web worker. In this example, the worker exposes two methods (`sum` and `mul`) and a single event (`ping`) to the parent. The parent could expose methods and events as well.

Install:
```
npm install post-me
```

Parent code:
```typescript
import { ParentHandshake, WorkerMessenger } from 'post-me';

const worker = new Worker('./worker.js');

const messenger = new WorkerMessenger({ worker });

ParentHandshake(messenger).then((connection) => {
  const remoteHandle = connection.remoteHandle();

  // Call methods on the worker and get the result as a promise
  remoteHandle.call('sum', 3, 4).then((result) => {
    console.log(result); // 7
  });

  // Listen for a specific custom event from the worker
  remoteHandle.addEventListener('ping', (payload) => {
    console.log(payload) // 'Oh, hi!'
  });
});
```

Worker code:
```typescript
import { ChildHandshake, WorkerMessenger } from 'post-me';

// Methods exposed by the worker: each function can either return a value or a Promise.
const methods = {
  sum: (x, y) => x + y,
  mul: (x, y) => x * y
}

const messenger = WorkerMessenger({worker: self});
ChildHandshake(messenger, methods).then((connection) => {
  const localHandle = connection.localHandle();

  // Emit custom events to the app
  localHandle.emit('ping',  'Oh, hi!');
});
```

In this more complex [interactive demo](https://codesandbox.io/s/github/alesgenova/post-me-demo) a parent application communicates with a web worker and a child iframe. You can play around with it on [codesandbox](https://codesandbox.io/s/github/alesgenova/post-me-demo).

[![_config.yml]({{ site.baseurl }}/images/{{ page.image_dir }}/demo.png)](https://codesandbox.io/s/github/alesgenova/post-me-demo){:target="_blank"}

## 1. History
A few months ago I was using the [postmate](https://github.com/dollarshaveclub/postmate) library at work, to expose methods from my application (which was inside an iframe) to its parent app.

While postmate worked okay initially, I soon started running into some major limitations:
- You can call a method with arguments, but you can't get its return value.
- You can get the return value of a method, but only if the method takes no arguments.
- No typescript support, makes it hard to enforce the correctness of API exposed by the parent/child across teams
- If a method throws an error, it can't be caught by the other end.
- Only the child can expose methods and events.
- It works with iframes only.

I thought that it could be a fun weekend project to try and implement a new library that would overcome all of the shortcomings I had found, and that would provide first-class typescript support.

The first working version of post-me came together in a couple of days during the Thanksgiving break, and I was quite happy with it.

I soon realized that what I had wrote could be easily adapted to interface with web workers and beyond, making it more useful than the somewhat niche demand for communicating with iframes.

Now, after a few iterations, I believe post-me is ready to be introduced to a larger audience, and I hope it can be useful to some.

## 2. Typescript
Using typescript you can ensure that the parent and the child are using each other's methods and events correctly. Most coding mistakes will be caught during development by the typescript compiler.

Thanks to __post-me__ extensive typescript support, the correctness of the following items can be statically checked during development:
- Method names
- Argument number and types
- Return values type
- Event names
- Event payload type

Let's rewrite the small example above in typescript!

Types code:
```typescript
// types.ts

export type WorkerMethods = {
  sum: (x: number, y: number) => number;
  mul: (x: number, y: number) => number;
}

export type WorkerEvents = {
  'ping': string;
}
```

Parent Code:
```typescript
import {
 ParentHandshake, WorkerMessenger, RemoteHandle
} from 'post-me';

import { WorkerMethods, WorkerEvents } from './types';

const worker = new Worker('./worker.js');

const messenger = new WorkerMessenger({ worker });

ParentHandshake(messenger).then((connection) => {
  const remoteHandle: RemoteHandle<WorkerMethods, WorkerEvents>
    = connection.remoteHandle();

  // Call methods on the worker and get the result as a Promise
  remoteHandle.call('sum', 3, 4).then((result) => {
    console.log(result); // 7
  });

  // Listen for a specific custom event from the app
  remoteHandle.addEventListener('ping', (payload) => {
    console.log(payload) // 'Oh, hi!'
  });

  // The following lines have various mistakes that will be caught by the compiler
  remoteHandle.call('mul', 3, 'four'); // Wrong argument type
  remoteHandle.call('foo'); // 'foo' doesn't exist on WorkerMethods type
});
```

Worker code:
```typescript
import { ChildHandshake, WorkerMessenger, LocalHandle } from 'post-me';

import { WorkerMethods, WorkerEvents } from './types';

const methods: WorkerMethods = {
  sum: (x: number, y: number) => x + y,
  mul: (x: number, y: number) => x * y,
}

const messenger = WorkerMessenger({worker: self});
ChildHandshake(messenger, methods).then((connection) => {
  const localHandle: LocalHandle<WorkerMethods, WorkerEvents>
    = connection.localHandle();

  // Emit custom events to the worker
  localHandle.emit('ping',  'Oh, hi!');
});
```

## 3. Other Windows
As mentioned earlier, post-me can establish the same level of bidirectional communications not only with workers but with other windows too (e.g. iframes).

Internally, the low level differences between communicating with a `Worker` or a `Window` have been abstracted, and the `Handshake` will accept any object that implements the `Messenger` interface defined by post-me.

This approach makes it easy for post-me to be extended by its users.

A `Messenger` implementation for communicating between window is already provided in the library (`WindowMessenger`).

Here is an example of using post-me to communicate with an iframe.

Parent code:
```typescript
import { ParentHandshake, WindowMessenger } from 'post-me';

// For safety it is strongly adviced to pass the explicit child origin instead of '*'
const messenger = new WindowMessenger({
  localWindow: window,
  remoteWindow: childWindow,
  remoteOrigin: '*'
});

ParentHandshake(messenger).then((connection) => {/* ... */});
```

Child code:
```typescript
import { ChildHandshake, WindowMessenger } from 'post-me';

// For safety it is strongly adviced to pass the explicit child origin instead of '*'
const messenger = new WindowMessenger({
  localWindow: window,
  remoteWindow: window.parent,
  remoteOrigin: '*'
});

ChildHandshake(messenger).then((connection) => {/* ... */});
```

## 4. Debugging
You can optionally output the internal low-level messages exchanged between the two ends.

To enable debugging, simply decorate any `Messenger` instance with the provided `DebugMessenger` decorator.

You can optionally pass to the decorator your own logging function (a glorified `console.log` by default), which can be useful to make the output more readable, or to inspect messages in automated tests.

```typescript
import { ParentHandshake, WorkerMessenger, DebugMessenger } from 'post-me';

import debug from 'debug';          // Use the full feature logger from the 'debug' library
// import { debug } from 'post-me'; // Or the lightweight implementation provided

let messenger = new WorkerMessenger(/* ... */);
// To enable debugging of each message exchange, decorate the messenger with DebugMessenger
const log = debug('post-me:parent'); // optional
messenger = DebugMessenger(messenger, log);

ParentHandshake(messenger).then((connection) => {/* ... */});
```

Output:

![_config.yml]({{ site.baseurl }}/images/{{ page.image_dir }}/debug.png)

## 5. Conclusion
Thank you for reading, I hope post-me can be useful to other people as well.
If you would like to try out or contribute to the library, the source code is available on [GitHub](https://github.com/alesgenova/post-me).
