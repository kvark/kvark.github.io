---
layout: post
title:  "Shader translation benchmark"
date:   2022-02-17 01:01:01 -0500
categories: naga shader
---

TL;DR: [new benchmark](https://github.com/kvark/shader-translation-benchmark) shows
how Naga is orders of magnitude faster at translating shaders than anything else out there.

## Problem Domain

Shader translation is an act of producing a platform-specific shader format, such as SPIR-V or MSL,
from a shader written by humans. It allows authors to write in one language and run on different platforms.
The source languages are often GLSL, HLSL, [WGSL](https://gpuweb.github.io/gpuweb/wgsl/), or even [Rust](https://github.com/EmbarkStudios/rust-gpu).

The tools are often invoked by build scripts, or by engines at runtime. So they are largely out of sight,
and only become discussed if anything goes wrong.
[SPIRV-Cross](https://github.com/KhronosGroup/SPIRV-Cross) (SPIRV -> anything)
and [glslang](https://github.com/KhronosGroup/glslang) (GLSL -> SPIRV) (which is also powering [shaderc](https://github.com/google/shaderc))
are the most established solutions, widely used in production. With [WebGPU raising](https://gfx-rs.github.io/2019/03/06/wgpu.html),
there is a new generation of tools come into play.

- [Naga](https://github.com/gfx-rs/naga) is a pure Rust shader translation library supporting multiple frontends and backends
  (see the [original announcement](https://gfx-rs.github.io/2019/07/13/javelin.html) under Javelin name), powering [wgpu](https://github.com/gfx-rs/wgpu)
  (and transitively - Gecko/Firefox).
- [Tint](https://dawn.googlesource.com/tint/) is Google's shader translator powering [Dawn](https://dawn.googlesource.com/dawn)
  (and transitively - Blink/Chromium).

Both of these new libraries focus on translating WGSL to everything, but they can also process SPIR-V,
and Naga can even work with GLSL inputs. This is used both on native and the Web (by compiling Naga to WASM).

As [gfx-portability](https://github.com/gfx-rs/portability) was migrating to Naga,
we [benchmarked the time](https://gfx-rs.github.io/2021/05/09/dota2-msl-compilation.html)
it spent in processing Dota2 shaders on macOS.
Results were very convincing, showing 4x improvement in shader loading times, versus SPIRV-Cross.

## Methodology

### Setup

I picked a few end-to-end tests for benchmarking, such that there is going to be some competition in doing the operation,
and code paths are not exercized multiple times (i.e. all frontends and all backends are different).
These shaders are found in real projects in Rust ecosystem, they attempt to represent naturally written shaders.

Generally, each group is tested with the following steps:
```rust
let lib = init_library();
let test = load_test_data();
start_timer();
for test in tests {
	lib.process(test);
}
stop_timer();
lib.exit();
```

My machine is the [Framework laptop]({% post_url 2021-10-17-framework-nixos %}) with Core i7-1185G7 at 3.00GHz.
All the runs are single-threaded as far as I'm aware, and not swapping memory.

### Caveats

There are no detailed checks about the validity of the output or matching the semantics of the input shaders.
All of the libraries involved are used in many working projects, and the assumption is that the results are somewhat valid.
We don't require the same physical *form* of produced shaders.
For example, Tint may produce `std::string`, while Naga produces `String`.
Input/output time is excluded from timing. Everything is ready in memory for tests to start.

We are also not comparing the performance of shaders on GPU. I'm sure there are differences there, but I don't know if
they are significant enough to explain the difference in translation times.

In the benchmark, Naga is doing only minimal IR (Intermediate Representation) validation.
Generally, in WebGPU/wgpu Naga validates all of the IR produced from the input,
since it's required for safety and correctness. However, it doesn't matter for end-to-end tests,
since the assumption is that the author is confident in their correctness (at run-time, at least),
and that something else is going to consume them and validate as needed. For example,
when you have GLSL shipped in a Web game, and you use a Naga WASM build to convert it to WGSL,
there is no need to validate anything, since the produced WGSL will be validated by the browser anyway.
We also found that validation in Naga is extremely fast (roughly 7 times faster than WGSL parsing),
so including it wouldn't affect the results much.

### Reproducibility

In order to reproduce the results, check out the [git repository](https://github.com/kvark/shader-translation-benchmark)
with submodules and follow the regular `cmake` build process. Naga and Tint have thin FFI layers to communicate with C,
which is the language of the benchmark. `make chart` also invokes a Rust program that grabs the output
and writes down the vertical bar charts into `visual/products` folder, which is what you'll see below.

## Results

### GLSL to SPIR-V

This is a classical path used by many Vulkan applications. This is also how Vulkan was tested at the beginning in Khronos.
We took [Bevy](https://bevyengine.org/)'s PBR shaders in GLSL as the input.

![bar chart](/resource/GLSL2SPIRV.svg)

Nobody else processes GLSL inputs, so we are only comparing Naga with Glslang. Naga is roughly *30x* faster.
I'm sure Glslang does a lot of work in validating the incoming GLSL that we aren't,
so that explains some of that difference.
It's also probably trying to optimize the result. We'll be working on configuring the build to do less of that.

### WGSL to GLSL

This is the standoff between WGSL parsers to some extent, so only the new generation of libraries are at play: Naga and Tint.
There is a few shaders from wgpu's examples, and also one of the beefier shader from [Rusty Vangers](https://vange.rs/).
This fragment shader does ray marching across a multi-level terrain.
Practically speaking, this path is used when running WebGPU on an OpenGL backend.
We have many users doing so by compiling wgpu applications for WASM WebGL2 target.

![bar chart](/resource/WGSL2GLSL.svg)

Naga is about *13x* faster than Tint on that test.

### SPIR-V to MSL

Finally, this is more of a "classic" test, similar to what I did in the Dota2 benchmark.
This is what Vulkan Portability path takes on Apple platforms.
Also, wgpu/dawn applications that accept SPIR-V on native platforms (but not the Web).
The inputs are taken from [Veloren](https://veloren.net/) project. These are the heaviest shaders they use.

![bar chart](/resource/SPIRV2MSL.svg)

Basically, the middle finger. The linear scale of the graph starts to break up here.
Naga is *80x* faster than Tint and just *10x* faster than SPIRV-Cross.
I'm sure we are just hitting some weird edge case in Tint, but can't say the same about SPIRV-Cross.

## Appendix

### Naga

Naga was architectured from the start to be efficient.
All of the data structures are highly localized and cache-friendly.
There are minimum heap allocations, and maximum reuse of intermediate products.
At the same time, we haven't done any micro-optimizations yet.
For example, access to all internal objects goes through regular Rust array indexing,
which incurs a release assertion for bounds checking.

Donald Knuth:
> Premature optimization is the root of all evil

I believe our approach is paying off, contrary to that Knuth's statement.
Maintaining an architecture is easier than rebuilding it from scratch late in the project lifetime.
We've been following strict rules in Naga internal API, some of them surprising new contributors.
For example, our `enum TypeInner` is not `impl Clone`, even though it could be. In some cases,
it would be convenient to have. My concern with that was that once we start cloning it everywhere,
it would be too easy to accidentally do it when it's not necessary.
And cloning would potentially involve heap allocation for it, so should be avoided.

My lesson from this, if I'm allowed to draw one, is:
"do not implement something in Rust just because you can, instead think about what you can get away with".
Looking at you, [C-COMMON-TRAITS](https://rust-lang.github.io/api-guidelines/interoperability.html#c-common-traits).

### Context

Seeing the benchmark results helped me to understand the context behind some of the suggestions in WebGPU land:

- [HN#23089066](https://news.ycombinator.com/item?id=23089066) - skip run-time translation in WebGPU.
- [gpuweb#2119](https://github.com/gpuweb/gpuweb/issues/2119) - allow shader compilation being asynchronous.
- [gpuweb#2571](https://github.com/gpuweb/gpuweb/issues/2571) - support binary WGSL form.

All of these needs are based on an assumption that shader translation takes a lot of code and running time (slow!).
This is much less of a problem with Naga and wgpu, even if it's still a problem in general.

I believe by this point you might be convinced that Naga's WGSL parser is very fast.
However, it's worth noting that loading the IR directly (using [bincode](https://crates.io/crates/bincode)) appears to be 7x faster.
You can see it yourself by running `make bench` inside Naga.

### In Motion

I'm confident that Tint will see drastic improvements over the years to come.
It has a very strong team at Google driving it, which was focused on correctness first and speed later.
It's borderline unfair to take it at its current shape!
But we can't wait indefinitely - we started later, and we've spent only a fraction of
engineering resources in Naga than any of the competitors.
So we are happy to capture the results here and now.
