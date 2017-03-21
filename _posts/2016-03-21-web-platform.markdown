---
layout: post
title:  "Defining the Web platform"
date:   2017-03-21 11:11:11 -0500
categories: web 3d api mozilla
---

Observing the ideological difference in the definitions of the Web in the context of 3D graphics APIs.

## Proposals

Mozilla has just made an official proposal for the new 3D API on the Web called [Obsidian](https://github.com/KhronosGroup/WebGLNext-Proposals/pull/2). It follows Apple's [WebGPU](https://webkit.org/blog/7380/next-generation-3d-graphics-on-the-web/) and is likely to be followed by Google's [NXT](https://github.com/gpuweb/nxt-standalone). The plan of the [working group](https://www.phoronix.com/scan.php?page=news_item&px=WebGL-Next-Proposals-Repo) is to gather proposals and then start discussing them while trying to converge/agree to a single API. Then there's going to be rounds and rounds of discussions, polishing, prototyping, and so forth, resulting in early implementations in the major browsers somewhere in 2018.

The working group is good at chopping through technical issues, especially when given a good [starting point](http://www.pcworld.com/article/2894036/mantle-is-a-vulkan-amds-dead-graphics-api-rises-from-the-ashes-as-opengls-successor.html). Interestingly, the current disagreements are not technical but rather philosophical. What is the Web? What basic principles should all the Web APIs adhere to? I'm very new to all that, and I'm speaking on behalf of myself, not expressing Mozilla's point of view.

## Spectrum

We could imagine a spectrum of volatility in the API guarantees:
  - On the far left side, we could see safety critical interfaces, like [OpenGL SC](https://www.khronos.org/openglsc/). They are designed to be deterministic and testable in order to minimize implementation and safety certification costs. They could control machinery your life depends on, and no compromises are made to sacrifice safety.
  - Somewhere in the middle we see [WebGL](https://www.khronos.org/webgl/). It necessarily conforms to the security principles of the web platform. It avoids undefined behavior (for the most part), since UB prevents writing strong conformance tests, and may introduce security vulnerabilities. Browsers are required to validate most of the state before it's passed to the GL driver, making WebGL calls a CPU [bottleneck](https://www.mail-archive.com/emscripten-discuss@googlegroups.com/msg01674.html) of the applications.
  - On the far right we see [Vulkan](https://www.khronos.org/vulkan/). It defines "valid usage" rules, which set limited guarantees on the executing code. A step away from these rules can result in anything from a harmless color change to the collapse of the universe. All the trade-offs are made for explicit hardware control and maximum performance. For example, using multiple sub-passes in a deferred renderer on a mobile phone can get you [30% performance](https://youtu.be/y-EBiswp3qU?t=2431) increase and drastic reduction in bandwidth and power consumption.

If I was to put _WebGPU_ and _NXT_ on this scale, they'd probably land somewhere near _WebGL_. They provide several important concepts of the next-gen APIs, such as command buffers and pipeline state objects, without compromising safety or consistency across browser implementations. It would be a much welcome upgrade from _WebGL_, a straightforward path to more efficient Web applications.

_Obsidian_, however, would be much closer to the far right side. It still guarantees that any application is secure, but it also defines "valid usage" rules. The difference with Vulkan is that derailing from the valid usage scenario doesn't throw your expectations into the bin. Instead, the undefined behavior is classified, and certain aspects of it are allowed to happen (when API is misused) as long as they don't compromise security. It trades performance capability and determinism over the execution consistency. An invalid application may behave differently in various environments.

## Future

We can continue with the established route and develop the new 3D API conservatively by putting a similar set of guarantees to _WebGL_. I'm hoping that we can also push the boundaries and guarantees of the Web platform farther in order to achieve higher performance and start competing with the native application stores, like App Store or Steam. The trade-off is twice as sensitive for VR, which requires more graphics workload and finer latency control. At this point, it's not clear if the risk (of becoming less consistent) is worth it.

I'm looking forward to the debate on what the Web is, and I'm excited to hear the feedback on our API proposal. I want the Web to be great (again?), to overcome the walled gardens of native platforms and let the users freely exchange the content. I want to type "play Quake V" into the address bar, click the first search result, and actually play the game in the browser, even when running on a Chromebook.
