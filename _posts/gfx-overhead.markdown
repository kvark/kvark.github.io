---
layout: post
title:  "Overhead analysis for Vulkan Portability"
date:   2017-08-10 11:11:11 -0500
categories: api 3d vulkan
---

One of the design goals for the portability API is to keep any overhead (when translating to other APIs) to be minimum, optimally providing a zero-cost abstraction. In this article, we'll dissect the potential sources of the overhead into groups and analyze the prospect of each, suggesting possible solutions. The problem in question is very broad, but we'll spice it with examples raised in [Vulkan Portability Initiative](https://www.khronos.org/blog/khronos-announces-the-vulkan-portability-initiative).

## Another unit of compilation

When adding an indirection layer from inside a program, given a language with zero-cost abstractions like C++ or Rust, it is possible to have the layer completely optimized away. However, the library will be provided as a static/dynamic binary, which would prevent the linker to inline the calls. That means doubling the cost of a function invocation (as opposed to execution) compared to a native API.

Solutions:
  1. whole program optimization
  2. pure header library
    - locks into using C/C++
    - inconvenient
    - long compile times

## Native API differences

Some aspects of the native APIs don't exactly match together. This is amplified by the flexible nature of Vulkan, which tends to provide the richest feature set comparing to D3D12 and Metal.

For example, Vulkan allows command buffers to be re-used, and so does D3D12. In Metal, however, it's not directly supported. If this ability is exposed unconditionally, the Metal backend would have to record all the encoded commands on the side to translating them to the corresponding `MTL*CommandEncoder` interface. 

When the user requests to use the command buffer again, Metal backend would have to re-encode the native command buffer on the spot, which means a considerable delay to otherwise inexpensive operation of submitting a command buffer for execution.

Solutions:
  1. more granularity of device capabilities
  2. pressure other platforms to add native support for missing features

## Skewed idiomaticity

An API typically is associated with a certain way of thinking and approaching the problems to solve. Providing a Vulkan-like front-end to an "alien" API would skew the users to think in terms of Vulkan and what is efficient in it, as opposed to the native APIs.

For example, Vulkan has render sub-passes. Organizing the graphics workload in sub-passes allows tiled hardware (typically found in mobile devices) to re-use intermediate rendering results without a road-trip to VRAM. This can be a big optimization, yielding up to 50% performance increase as well as reduced power usage.

No other API has sub-passes. It is straightforward to emulate them by treating each sub-pass as an independent pass. However, ignoring the fact of intermediate results go back and forth to VRAM would cause the graphics pipeline to wait for these transfers and stall. With a non-multi-pass approach the user would insert an independent graphics job between producers and consumers of data, just to hide the memory latency by not immediately waiting for the producer job to finish.

When the user writes for Vulkan exclusively, they can have a firm believe that the driver optimizes the sub-passes (e.g. by reordering the work) for the non-tiled hardware. When Vulkan translates to D3D12 and Metal, there is no such luxury.

Solutions:
  1. device feature
    - either a soft one, serving as a hint that sub-passes are efficient
    - or a hard one, for allowing the use of multiple sub-passes in general
  2. pray that other native APIs will catch up one day

## Conclusion

We categorized the possible sources of performance overhead by ascending the ladder of abstraction. We started from the low level compiler units, proceeded through the warts of API translation, and finished on idiomatical differences of native APIs.

There are no good solutions to these problems. Our task (as a technical subgroup) is to strike for the balance between making minimal diversion from Vulkan and providing optimal performance on other backends, while keeping the API simple and explicit.
