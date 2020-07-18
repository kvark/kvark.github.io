---
layout: post
title:  "wgpu API tracing infrastructure"
date:   2020-07-18 11:11:11 -0500
categories: wgpu debug test ron
---

[wgpu](https://github.com/gfx-rs/wgpu) is a native WebGPU implementation in Rust, developed by [gfx-rs](https://gfx-rs.github.io/) community with help of Mozilla. It's still an emerging technology, and it has many users:
  - Gecko and Servo - for implementing WebGPU in the browsers
  - [wgpu-rs](https://github.com/gfx-rs/wgpu-rs) - for using idiomatically from Rust
  - [wgpu-native](https://github.com/gfx-rs/wgpu-native/) - for using from C language and others binding to C ([Scopes](https://nest.pijul.com/porky11/wgpu), [Python](https://github.com/pygfx/wgpu-py), even [Julia](https://github.com/cshenton/WebGPU.jl)), striving for C header compatibility with [Dawn](https://dawn.googlesource.com/) via the shared [webgpu-headers](https://github.com/webgpu-native/webgpu-headers).

Given the diversity of platforms and configurations it runs, and the variety of users, the questions of reproducing issues, debugging, and testing the implementation were critical to resolve.

## Prior art

Fortunately, I had some success in the past rolling out serialization-based infrastructures for capturing and testing complex pipelines.

### gfx-rs Warden

First, it was [Warden test framework](https://github.com/gfx-rs/gfx/pull/1589) in `gfx-rs`. It defined serializable types for all of `gfx-rs` commands, and also allowed describing different scenes, test-cases, and expectations. All the data was hand-written in [RON](https://github.com/ron-rs/ron) format, which by the time was quite young, and not used anywhere seriously. The ability to test `gfx-rs` without code was very exciting to us, and in general it worked out OK. In the end, we haven't written too many tests, mostly because we aggressively tested with Vulkan CTS (over [gfx-portability](https://github.com/gfx-rs/portability)) instead, which was enormous. The separation of scenes and workloads also ended up with a few gotchas and a less-than-elegant implementation. It was also a bit awkward to write the implicit synchronization code in Warden for grabbing back the results, or re-initializing the state between tests.

### WebRender capturing

The other related project was done in WebRender: the [capturing infrastructure]({% post_url 2018-01-23-wr-capture-infra %}). The purpose of this one was different: assist in reproducing and debugging issues. It serialized the pipeline at two different stages to disk, allowing the capture to be transferred to a different computer and reproduced in a simple standalone tool we called [Wrench](https://github.com/servo/webrender/tree/b14b5fe6db3d140c83ca10d9ca8f2fcadb7b8c13/wrench). The beauty of it is that we'd mess with the RON files by hands: remove items, or whole files, change values, just playing around and seeing how the problem reacts. Even if something goes off-rails, and your capture fails to replay, it was often possible to tweak it into a working state.

Overall, it was a huge success, and it became an indispensable tool in the arsenal of Firefox graphics team. Reproducing a bug in Firefox was half the problem, debugging it within Firefox was another half. The capturing infra solved both. However, I wanted to do more with it: I wanted to have a "portable" representation of a WR scene defined with a conversion to the regular WR scene. With this, we'd be able to route all the reftests through it, replacing the hand-parsed YAML format. This part of the story never happened - there were (and still are) more important things to do.

## wgpu trace/replay

Now, `wgpu` is fresh from the oven, and I wanted to roll in something as the best of both worlds. 

First problem was the incoming flow of bugs reported by users of `wgpu-rs`, users of Python API, users of Gecko, on different platforms, with closed source code, and so on. Reproducing these issues and debugging them was quite challenging. We figured that `wgpu` was the place where all the roads met, and we needed to serialize everything that reaches that intersection, to be replayed independently, on a different machine. We defined a [serialization format](https://github.com/gfx-rs/wgpu/pull/619) that we'd save all the incoming commands into at [device timeline](https://gpuweb.github.io/gpuweb/#device-timeline). We introduced a standalone "player" tool to replay the traces, which once again were stored as RON files.

With this in, all we needed from a bug reporter was a zipped API trace attached to an issue, and a git revision of the code they used. WebGPU is truly a *portable* GPU API, so the captures are easy to replay on a different machine. This is very unlike low-level traces, such as Vulkan traces, or Metal GPU captures - replaying them mostly did *not* work (your hardware has different limits, different memory types, features, etc). And there was nothing to do if it failed, unlike with our API traces, where you could just look at RON itself and nudge it to work. All in all, working with bugs became joyful, but we didn't stop there.

Another glaring problem was that `wgpu` repository didn't have any means to test itself. Originally, all of `wgpu`, `wgpu-core`, and `wgpu-native` were a part of the same repository, so we had the examples to check if the changes were sane. But when it was time to integrate into Gecko, we wanted to minimize the code that [mozilla-central](https://hg.mozilla.org/mozilla-central/) needs to vendor, so we moved everything but the core logic out into separate repositories. Rust does wonders with "if it compiles, it works" motto, but riding without any tests was still an insane idea. Requiring the developers to coordinate patches with multiple repositories, just so they can prove the changes still work, started hurting our productivity.

To resolve this, we implemented a Rust integration test with a simple [description of tests and expectations](https://github.com/gfx-rs/wgpu/pull/803). With just a little bit of magic, we made it so `cargo test` casually enumerates the supported GPU backends on the developer's machine, and runs the tests through it. The tests are described in the same RON format as API traces: they are basically sequences of actions, be it resource creation, or command submission. I call it "player-based GPU testing", or "playtest" for short. I don't know how far we are going to go with them, given that WebGPU API is being developed with it's own CTS (which we'll be able to run via browsers or NodeJS bindings to `wgpu-native`). At the very least, we'd want to cover the features in `wgpu-rs` [example matrix](https://github.com/gfx-rs/wgpu-rs/tree/8e4d0015862507027f3a6bd68056c64568d11366/examples#feature-matrix), to allow developers feel safe when landing patches in `wgpu`.

## Future work

Personally, I'm hugely excited for this `wgpu` infrastructure for tracing and playtesting. It's very powerful, and it covers a lot of ground. However, it's still early days, and there is a few rough corners and limitations.

One of the most annoying thing about the serialization format is `bitflags`. We use them aggressively in both `wgpu` and `gfx-rs`. Today, we have to use numerical representations of them, e.g. `usage: (bits: 0x41)`, which is neither readable or writable. I hacked a [small wrapper](https://github.com/kvark/bitflags-serial) around `bitflags` a while ago to serialize it nicely, e.g. `[VERTEX | MAP_READ]`. Still trying to figuring out the best way to approach this (upstream [issue](https://github.com/bitflags/bitflags/issues/147))...

Another interesting feature could be to test for exact errors. WebGPU group decided to not require the implementations to report specified errors (and instead, just throw more generic errors with a string description), based on experience with WebGL where this introduced unnecessary complexity. We want stronger guarantees in `wgpu` though, so being able to test the exact error variants would totally make sense even in the presence of upstream CTS.

We are looking forward to use this technology more :)