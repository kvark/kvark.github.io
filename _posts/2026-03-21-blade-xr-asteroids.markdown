---
layout: post
title:  "Blade XR Asteroids"
date:   2026-03-21 01:01:01 -0500
categories: blade xr
---

Just as Meta turns around and shifts its attention to AI, following basically everybody, I want to tell you about XR. As an owner of Quest 3S, I always wondered if it's easy to build for this device. Now I know.

## Blade XR

[Blade](https://github.com/kvark/blade/tree/main/blade-graphics) is the GPU abstraction layer I've been toying with lately. Recently, I've integrated support for OpenXR to it, while testing a new example on the Quest 3S. Blade-graphics would allow XR to create a Vulkan instance if requested, and Blade-engine would know to render to both eyes. But that was just the beginning - I had to overcome a few more blockers on the way to this:

![asteroids screenshot](/resource/blade-asteroids.jpg)

First, it's the ray tracing part. While Quest 3S declares support for ray tracing, the visuals [appeared concerning](https://github.com/kvark/blade/pull/301). And we can't expect any reasonable performance from this chip. The most reliable solution is to rasterize, like the old times. To achieve that, I've split [blade-render](https://github.com/kvark/blade/tree/main/blade-render) rendering pipeline into separate rasterization and ray tracing parts. The goal is to support both while sharing as much WGSL code as possible, internally. Also, sharing the vertex definitions, assets, etc.

The other problem is the Adreno GPU. The driver for it appeared to be very cranky, and would produce `VK_ERROR_UNKNOWN` when compiling any non-trivial pipeline. After hours of debugging, I concluded that it doesn't like the inline uniform blocks - a feature of Vulkan I was unconditionally using for convenience of having uniform data without explicit buffer management. Lacking a smaller workaround, I had to add support for old-school uniform buffers, managed internally in a bit of a hacky way. Hopefully, this will be useful to diagnose other GPU/driver issues outside of Adreno.

When these blockers were addressed, it was just a matter of implementing the logic. You are supposed to fly through an asteroid field and shoot them down. I hooked up the physics from [blade-engine](https://github.com/kvark/blade/tree/main/blade-engine) and got the movement, shooting, and collisions all handled. All of the objects in the scene are procedurally generated for simplicity. I had to have some particle effects (comets!), so I lifted this logic out of the "particle" example into its own crate and extended it to work with the rasterizing renderer.

The end result (aka "Asteroids") appears to be pretty fun! Easy to get carried away while testing :) See the [full video](https://vimeo.com/1175802946?share=copy&fl=sv&fe=ci). Hoping to release all changes on crates in the coming days.
