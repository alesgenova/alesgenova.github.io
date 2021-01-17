---
layout: post
title: "Running Rust in WebAssembly in a Pool of Concurrent Web Workers in JavaScript"
image_dir: 2021-1-16-concurrent-wasm-workers
---
![_config.yml]({{ site.baseurl }}/images/{{ page.image_dir }}/diagram.png)

>
> Your scientists were so preoccupied with whether or not they could, that they didn't stop to think whether they should.
> 
> <p align="right">- Ian Malcolm</p>
>

I would like to share a little experiment I did for no other reason than to show I could.

In this proof of concept application, the main application starts a pool of web workers that it later uses to offload a series of heavy tasks.

The task in question is to to render a single frame of a simple 3D scene using ray-tracing (path-tracing). The computationally intensive rendering is performed by a `rust` library compiled to WebAssembly.

These are the tools I used:
- __`post-me`__ to communicate with the workers using a `Promise` API: [Blog](https://alesgenova.github.io/introducing-post-me/) - [Github](https://github.com/alesgenova/post-me)
- My own toy ray tracing engine I implemented in rust a couple of years ago: [Github](https://github.com/alesgenova/ray-tracer)
- __`wasm-bindgen`__ to compile rust to wasm and create js bindings: [Github](https://github.com/rustwasm/wasm-bindgen)
- Small in-house task queue to dispatch tasks to workers when available.
- `react` for the scheleton of the app.

If you would like to run this madness, an instance of this application is deployed [here](https://alesgenova.github.io/ray-tracer-app/).

[![_config.yml]({{ site.baseurl }}/images/{{ page.image_dir }}/ray_tracer.gif)](https://alesgenova.github.io/ray-tracer-app/)

If you would like to see the details of the implementation, you can find the source code of the app on [Github](https://github.com/alesgenova/ray-tracer-app)

## Bonus

Using a similar approach, I also created an app that can detect the pitch of sounds being captured by the device's microphone.

[Try it out ](https://alesgenova.github.io/pitch-detection-app/)

[![_config.yml]({{ site.baseurl }}/images/{{ page.image_dir }}/pitch_detection.gif)](https://alesgenova.github.io/pitch-detection-app/)

- Source code: [Github](https://github.com/alesgenova/pitch-detection-app)
- Pitch detection library: [Github](https://github.com/alesgenova/pitch-detection)