---
layout: post
title:  "Feasibility of low-level GPU access on the Web"
date:   2018-02-10 11:11:11 -0500
categories: web gpu
---

This is a follow up to [Defining the Web platform](http://kvark.github.io/web/3d/api/mozilla/2017/03/21/web-platform.html) post from a year ago. It doesn't try to answer questions but instead attempts to figure out what are the right questions to ask.

## Trade-offs

As the talks within [WebGPU](https://www.w3.org/community/gpu/) community group progress, it becomes apparent that the disagreements lie in more domains than simply technical. It's about what the Web is today, and what we want it to become tomorrow. The further we go towards the rabbit hole of low-level features, the more we have to stretch the very definition of the Web:

  1. harder to guarantee security:
      - if the descriptors can be purely GPU driven (like Metal argument buffers), the browser can't guarantee that they point to owned and initialized GPU memory.
      - if there are many exposed flags for device capabilities, it's easy for a script to fingerprint users based on that profile.
  2. harder to achieve portability:
      - any use of incoherent memory
      - rich memory types spectrum (like Vulkan memory types/heaps) means that different platforms would have to take different code paths if implemented correctly, and that's difficult to test
  3. makes the API increasingly more complex to use, which Web developers may not welcome:
      - declaring a Vulkan like render pass with multiple passes is a challenge
  4. finally, all this goes into performance characteristics:
      - potentially lower CPU cost, given the bulk of validation work moved from the driver onto the browser
      - more consistent framerate, given better control of scheduled GPU work

## Consistency

It is clear that the native API have reached decent consistency in their respective programming models. Both Metal and Vulkan appear to be positioned at their local maximas, while I can't confidently state the same about NXT that is unclear:

```
 /*|     /**\         ?*?
/  | .. /    \ ..... ? ? ?
Metal   Vulkan        NXT
```

One can't just take a small step away from an existing API and remain consistent. Finding a whole new local maxima is hard because it means that the previous API designers (Khronos, Microsoft, Apple) either haven't discovered it or simply discarded it for being inferior. And while we have Microsoft and Apple representatives in the group, we lack the opinion of Vulkan designers from Khronos.

## Origins

My [first prototype](http://kvark.github.io/3d/api/2016/12/16/webmetal.html) was based on Metal, after which I had a chance to talk a lot about the future of Web GPU access with fellow Mozillians. The feedback I got was very consistent:
  - use existing Vulkan API, making necessary adjustments to provide security and portability
  - provide maximum *possible* performance by making a platform, let higher level libraries build on it and expose simpler APIs to the users

A similar position was expressed by some of the [Vulkan Portability](https://www.khronos.org/blog/khronos-announces-the-vulkan-portability-initiative) members. And this is where we've been going so far with the development of [portability layer](https://github.com/gfx-rs/portability) and experiments with Vulkan-based WebIDL [implementation in Servo](https://github.com/kvark/webgpu-servo). Check out the [recent talk](https://www.youtube.com/watch?v=ApPJqWD9cDk) at Fosdem 2018 as well as our [2017 report](http://gfx-rs.github.io/2017/12/30/this-year.html) for more details. Unfortunately, this direction faces strong opposition from the rest of W3C group.

## Portability

Interestingly, there is an existing precedent of providing a less portable Web platform API (quote from an [article by Lin Clark](https://hacks.mozilla.org/2017/06/avoiding-race-conditions-in-sharedarraybuffers-with-atomics/)):
> `SharedArrayBuffers` could result in race conditions. This makes working with `SharedArrayBuffers` hard. We don’t expect application developers to use `SharedArrayBuffers` directly.
> But library developers who have experience with multithreaded programming in other languages can use these new low-level APIs to create higher-level tools. Then application developers can use these tools without touching `SharedArrayBuffers` or `Atomics` directly.

So if the question is "Does Web API *have* to be portable?", the answer is a definite "no". It's a nice property, but not a requirement, and can be justified by other factors, such as performance. Speaking of which... the case for performance gains in `SharedArrayBuffers` appears to be strong, which convinced the browser vendors to agree on exposing this low-level API. Now, can we apply the same reasoning to get Vulkan level of explicitness and portability to the Web? Can we rely on user libraries to make it more accessible and portable? Having some performance metrics would be great, but obtaining them appears to be extremely difficult.

Note: currently `SharedArrayBuffers` are disabled on all browsers due to the high-resolution timers in them discovered to be exploitable. One could argue that this is related to them being low-level, and thus we shouldn't take them as a good example for a Web API. This is countered by the fact that the disabling is temporary, and the usefulness of SABs is completely clear (see [ECMA security issue](https://github.com/tc39/security/issues/3), also note this [comment](https://github.com/tc39/security/issues/3#issuecomment-364214608)).

## Performance

What would be a good benchmark that shows the cost of memory barriers and types being implicit in the API?

  1. we need to have the same GPU-intensive workloads running on Vulkan and Metal, preferably utilizing multiple command queues and advanced features like multiple passes or secondary command buffers
  2. ... on the same hardware, which means Windows installed on a MacBook. Even then, we may see the differences in how OS schedules hardware access, how drivers are implemented, how applications settings are configured on different OSes, etc.
  3. the code paths split between Vulkan/Metal should be fairly high. If it's done via an API abstraction layer like Vulkan Portability, then the results are obviously going to be skewed towards this API. Splitting at high level means taking most advantage of the native API features, but also means more work for the developer.

Until that benchmarking is done, we can't reasonably argue in favor of performance when there are concrete sacrifices in portability, API complexity (and security, to an extent) that come with it... Defending Vulkan-like approach goes like this:

  - **A**: pipeline barriers are confusing, error-prone, and not evidently fast, let's make them implicit. Look, our native API does that, and it's successful.
  - **M**: but... tracking the actual layout and access flags on our side is hard (for the backends that require them), and it gets really difficult when multiple queues are involved that share resources
  - **G**: let's only share resources with immutable access flags then, and transition ownership explicitly between queues otherwise
  - **A**: manually synchronizing submissions between queues is hard to get right anyway, prone to portability issues, etc. Let's only have a single queue!

And so it goes... one aspect leading to another, proving that the existing APIs are consistent and local maxima, and that Metal is technically easier to fit the shoes of the Web.

## Result

Ok, this post is turning into a rant, rather inevitably... Sorry!

The unfortunate part of the story is that the group will not agree on an existing API, no matter what it is, because it would be biased towards a specific platform. And while Mozilla and Apple are at least conceptually aligned to existing API concepts, Google has been actively trying to come up with a new one in [NXT](https://github.com/google/nxt-standalone). As a result, not only the multi-platform applications would have to add support for yet another API when ported to the Web, that API is also going to be targeted from native via Emscripten and/or WebAssembly, and Google is all into providing the standalone (no-browser) way of using it, effectively [adding to the list](https://xkcd.com/927/) of native APIs... Without any IHV backing, and promoted by... browser vendors? That is the path the group appears to be moving unconsciously in, instead of building on top of existing knowledge and research.

The current struggle of developing the Web GPU API comes down to many factors. One of the, if not the most, important ones here is that the parties have different end results envisioned. Some would like the JavaScript use to be nice, idiomatic, and error-resistant. Some mostly care about WebAssembly being fast. Some can't afford a badly written application looking different on different mobile phones. Some can't expect developers to be smart enough to use a complicated API. Some leave WebGL as a backup choice for a simple and accessible but slower API.

And the fact we are discussing this within W3C doesn't help either. We don't have immediate access to Khronos IHV experts and ISV advisory panels. We desperately need feedback from actual target users on what they want. Who knows, maybe the real answer is that we are better off without yet another API at all?
