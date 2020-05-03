---
layout: post
title:  "Point of WebGPU on native"
date:   2020-05-03 11:11:11 -0500
categories: web gpu native
---

WebGPU is a new graphics and compute API designed on the grounds of W3C organization (mostly) by the browser vendors. It's designed for the Web, used by JavaScript and WASM applications, and driven by the shared principles of Web APIs. It doesn't have to be *only* for the Web though. In this post, I want to share the vision of why WebGPU on native platforms is important to me. This is highly subjective and doesn't represent any organization I'm in.

## Story

The story of WebGPU-native is as old as the API itself. The initial hearings at Khronos had the same story heard at both the exploration "3D portability" meeting, and the "WebGL Next" one, told by the very same people. These meetings had similar goals: find a good portable intersection of the native APIs, which by that time (2016) clearly started diverging and isolating in their own ecosystems. The differences were philosophical: the Web prioritized security and portability, while the native wanted more performance. This split manifested in creation of two real working groups: one in W3C building the Web API, and another - "Vulkan Portability" technical subgroup in Khronos. Today, I'm the only person ("ambassador") who is active in both groups, simply because we implement both of these APIs on top of [gfx-rs](https://github.com/gfx-rs/gfx).

Vulkan Portability attempts to make Vulkan available everywhere by layering it on top of Metal, D3D12, and others. Now, everybody starts using Vulkan, celebrate, and never look back, right? Not exactly. Vulkan may be fast, but its definition of "portable" is quite weak. Developing an app that doesn't trigger Vulkan validation warnings(!) on any platform is extremely challenging. Reading Vulkan spec is exciting and eye opening in many respects, but applying it in practice is a different experience. Vulkan doesn't provide low-level API to all GPU hardware, it has the head of a lion, the body of a bear, and the tail of a crocodile. Each piece of Vulkan matches *some* hardware (e.g. `vkImageLayout` matches AMD's), but the other hardware treats it as a no-op at best, and as a burden at worst. It's a jack of all trades.

Besides, real Vulkan isn't everywhere. More specifically, there are no drivers for Intel Haswell/Broadwell iGPUs on Windows, it's forbidden on Windows UWP (including ARM), it's below 50% on Android, and totally absent on macOS and iOS. Vulkan Portability aims to solve it, but it's another fairly complex layer for your application, especially considering the shader translation logic of [SPIRV-Cross](https://github.com/KhronosGroup/SPIRV-Cross), and it's still a WIP.

In gfx-rs community, we made a huge bet on Vulkan API when we [decided to align](https://gfx-rs.github.io/2018/04/09/vulkan-portability.html) our low-level Rusty API to it (see our [Fosdem 2018 talk](https://archive.fosdem.org/2018/schedule/event/rust_vulkan_gfx_rs/)) instead of figuring out our own path. But when we (finally) understood that Vulkan API is simply unreachable by most users, we also realized that WebGPU on native is precisely the API we need to offer them. We saw a huge potential in a modern, usable, yet low-level API. Most importantly - WebGPU is safe, and that's something Rust expresses in the type system, making this property highly desired for any library.

This is where [wgpu](https://github.com/gfx-rs/wgpu) project [got kicked off](https://gfx-rs.github.io/2019/03/06/wgpu.html). For a long while, it was *only* implementing WebGPU on native, and only recently became a [part of Firefox](https://hacks.mozilla.org/2020/04/experimental-webgpu-in-firefox/) (Nightly only).

## Promise

Let's start with a bold controversial statement: there was never a time where developers could target a single API and reach the users, consistently with high quality. On paper, there was OpenGL, and it was indeed everywhere. In practice, however, OpenGL drivers on both desktop and mobile were poor. It was wild west: full of bugs and hidden secrets on how to convince the drivers to not do the wrong thing. Most games targeted Windows, where at some point D3D10+ was miles ahead of OpenGL in both the model it presented to developers, and the quality of drivers. Today, Web browsers on Windows don't run on OpenGL, even though they accept WebGL APIs, and Firefox has WebRender on OpenGL, and Chromium has SkiaGL. They run that all on top of [Angle](https://github.com/google/angle), which translates OpenGL to D3D11...

This is where WebGPU on native comes on stage. Don't get fooled by the "Web" prefix here: it's a sane native API that is extremely portable, fairly performant, and very easy to get right when targeting it. It's an API that could be your default choice for writing a bullet hell shooter, medical visualization, or teaching graphics programming at school. And once you target it, you get a nice little bonus of being able to deploy on the Web as well.

From the very beginning, Google had both native and in-browser use of their implementation, which is now called [Dawn](https://dawn.googlesource.com/dawn). We have a shared interest in allowing developers to target a shared "WebGPU on native" target instead of a concrete "Dawn" or "wgpu-native". Therefore, we are collaborating on a [shared header](https://github.com/webgpu-native/webgpu-headers), and we'll provide the C-compatible libraries implementing it. The specification is still a moving target, and we haven't yet aligned our external C interface to the shared header, but that's the plan.

## Target market

If you are developing an application today, the choice of the tech stack looks something like this:
  1. target OpenGL only, accept the performance hits and bugs, potentially route via Angle.
  2. target a graphics library like [bgfx](https://github.com/bkaradzic/bgfx) or [sokol-gfx](https://github.com/floooh/sokol), accept working without a spec, also be ready to fix bugs and submit patches.
  3. target Vulkan Portability, accept the complexity.
  4. develop your own graphics backends on selected platforms, accept the development costs.
  5. pick an engine like Unity/Unreal/Lumberyard that gives you portable graphics, accept all the features that come with it.

Today's engines are pretty good! They are powerful and cheap to use. But I think, a lot of appeal to them was driven by the lack of proper solutions in the space below. Dealing with different graphics APIs on platforms is hard. Leaving it to a third-party library means putting a lot of trust in it.

If WebGPU on native becomes practical, it would be strictly superior to (1) and (2). It will be supported by an actual specification, a rich conformance test suite, big corporations, and tested to death by the use in browsers. The trade-offs it provides would make it a solid choice even for developers who'd otherwise go for (3), (4), and (5), but not all of them of course.

I can see WebGPU on native being a go-to choice for amateur developers, students, indie professionals, mobile game studios, and many other groups. It could be *the* default GPU API, if it can deliver on its promises of safety, performance, and portability. We have a lot of interest and early adopters, as well as big forces in motion to make this real.