---
layout: post
title:  "Sayonara, Mozilla"
date:   2022-02-02 11:11:11 -0500
categories: mozilla
---

## WebGPU

Back when I joined Mozilla in Oct 2016, I was eager to build a new API on the Web. In one of the first few days at work, while "drinking from the firehose" of new information, I reached to [Vlad Vukicevich](https://github.com/vvuk) (the creator of WebGL and WebVR) across the table, asking if there is anything in the pipeline to overthrow WebGL. And since then, I embarked on the epic journey to make this real.

There's been a lot of ups and downs. The handful of [prototypes]({% post_url 2016-12-16-webmetal %}) I built for Servo got all scraped. Our [affair with Vulkan]({% post_url 2021-06-20-vulkan-alignment %}) was educational but also [distracting]({% post_url 2018-02-10-low-level-gpu-web %})) from the goal. And yet, I became a pillar of WebGPU and Vulkan Portability. I'm an WebGPU spec editor, and I helped driving many of the aspects of the API. I'm happy to see that WebGPU is stabilizing, it's mostly done. Even the WGSL part of it is likely going to settle now. Hopefully, we'll see the API released by the end of this year.

Implementation of WebGPU in Gecko [is built]({% post_url 2019-12-20-gecko-webgpu %}) on a Rusty software stack that I'm proud of. It's the [wgpu](https://github.com/gfx-rs/wgpu) and [naga](https://github.com/gfx-rs/naga/) powering it, as well as many other projects in Rust community. We have first class validation and full safety, [debugging layer]({% post_url 2020-07-18-wgpu-api-tracing %}), multi-threading, support on all major platforms, and a welcoming community of contributors. I'm hoping to see `wgpu` continuing to strive, ideally without relying on me as a single point of failure.

## WebRender

Another big project at Mozilla for me was [WebRender](https://github.com/servo/webrender/). The task of "porting" WebRender to Gecko ended up with a lot more work than anybody hoped. Architecture changed in many ways, the functionality grew significantly, and we had to deal with tons of [driver bugs](https://github.com/servo/webrender/wiki/Driver-issues). It's a very different product now than what Servo demonstrated. Since I was basically hired to help kicking off this effort, I had a chance to lay down some of the foundational work in clipping, spatial hierarchy, 3d transforms, [debugging infrastructure]({% post_url 2018-01-23-wr-capture-infra %}), and more. Working on a large Rust project with a good team was truly a joy. Realistically speaking, it was mostly the team, but I'm glad I could help :) And now, WebRender has fully shipped, it's everywhere. So whatever happens next, I can always open the browser and pat myself on the back with "I worked on this!".

## What's next?

I'm parting ways with Mozilla. It's a great place to work, and it's been very kind to me all this time. I worked with the nicest technologies, had practically full operational freedom, and even got a team to lead at the end. I'll still root for Mozilla's mission to succeed, but my path will steer away from the Web. As to "why?" - there are many nuances, and I shouldn't explain them here.

My participation in WebGPU Working Group is probably ending as well. It would be strange to continue doing the spec editor work without investing as much time, and without representing a browser company with an actual implementation.

I'm going to be busy teaching cars to drive by showing them nice pictures of the real world. For this, I'm moving to the Tesla HQ in Bay Area.
