---
layout: post
title:  "WebRender capture infrastructure"
date:   2018-01-23 11:11:11 -0500
categories: webrender debug ron
---

For over a year now, I've been hacking on [WebRender](https://github.com/servo/webrender). It was born in [Servo](https://github.com/servo/servo) as an experimental way to batch the painting and compositing of the web content on GPU. Today it's a solid piece of engineering that's going to mainline Firefox as the next big Rust-written component within the [Quantum project](https://medium.com/mozilla-tech/a-quantum-leap-for-the-web-a3b7174b3c12). You can read more about WebRender on our [team's blog](https://mozillagfx.wordpress.com/) as well as this wonderfully illustrated [article by Lin Clark](https://hacks.mozilla.org/2017/10/the-whole-web-at-maximum-fps-how-webrender-gets-rid-of-jank/).


## WebRender pipeline

The way of input items to the screen goes through a series of steps, or levels. Each following level gets conceptually closer to the GPU, and the end result is a sequence of OpenGL commands:

  1. On the user side, there is some representation of the workloads that gets transcribed into calls to `DisplayListBuilder` methods. The result, in a shape of serialized DLs (`BuiltDisplayList`, or "display lists") is sent to RB (`RenderBackend`).
  2. On the RB side, there are scenes (`Scene`) and resources (`ImageResource`, `FontTemplate`, `FontInstance`, etc). The main task of RB is to build those scenes into tiling frames and cull everything that can be culled. Scene building involves computing all the transforms and bounds of the clip-scroll tree, populating GPU data for instances, batching and re-ordering draw calls, etc.
  3. The rendered document (as a result of the previous step) is mostly represented by a render task tree. Some of the tasks are about drawing batched primitives, some are filters and composition operations. The produced task tree depends on cached resources, re-used render tasks from previous frames, and GPU cache. The latter is a giant blob of data that mixes all the parameters needed by shaders to draw primitives.
  4. `Renderer` traverses the tasks and issues OpenGL commands, eventually showing up on screen.
  

## Bug hunting
> If there's something weird... 
> And it don't look good... 
> Who you gonna call?..
> W-R-Capture!

Rust eliminates a fair share of most annoying and dangerous issues related to memory and low-level resource management. The strong type system helps us being confident about jumping between coordinate systems, handling large enumerations, and generally enforcing the logic invariants if the internal API bits. However, he higher level logical errors are still possible, and happen all the time. Once we raise above the expressiveness capability of Rust, we only have our common sense to rely on. And of course, powerful debugging tools!

A logic error may occur on one of the described levels, causing havoc and eventually manifesting in panics, flickering, and other glitches. Processing data from one level to the next is not a pure function: there is a lot of context getting mixed in the process, e.g. images from other tabs in the texture cache, fonts and glyphs, allocation of render task surfaces, free lists structure, and so on. Our capability to investigate and eliminate the issues depends on the ability to consistently reproduce the affected workloads on multiple levels.

The new capture infrastructure was designed to allow us to inspect and "freeze" deep levels of WebRender pipeline. It was delivered in a sequence of PRs: [#2232](https://github.com/servo/webrender/pull/2232), [#2305](https://github.com/servo/webrender/pull/2305), [#2326](https://github.com/servo/webrender/pull/2326), and [#2332](https://github.com/servo/webrender/pull/2332).

### Level 1: user input

This level should really be zero, if not for markdown forcing 1-based indexing. Technically, WebRender hasn't done anything with the data yet at this point, so the user is only themselves to blame :)

WebRender has recording functionality that writes out a continuous stream of commands, conceptually sitting somewhere half-way to the next level. It can be enabled programatically via `RendererOptions`, or by enabling `ENABLE_WR_RECORDING` environmental variable when running Firefox. Recording produces a binary blob, which one can then attempt to replay with our `wrench` tool, and even save out the individual frames as YAML/RON/JSON.

The exact experience of using this workflow may vary. I haven't gotten much luck tracing down graphics issues with help of recording, expressing my frustration in [an issue](https://github.com/servo/webrender/issues/2231). In particular, the ability to save/load YAML frames is frequently broken because the translation isn't done through [Serde](https://serde.rs).

### Level 2: scenes and resources

This is where capturing starts showing off. User can issue `RenderApi::save_capture(path, CaptureBits::SCENE)` call to trigger a dump to disc (at the specified folder path) of all the scene data and resource templates. Serialization of internal structs is done through Serde and uses [RON](https://github.com/ron-rs/ron) format by default. All the external images and blobs are getting read and dumped to raw files.

This is what the render backend state looks like at this level:
```yaml
    default_device_pixel_ratio: 1,
    enable_render_on_scroll: false,
    frame_config: (
        enable_scrollbars: false,
        default_font_render_mode: Subpixel,
        debug: false,
        dual_source_blending_is_supported: true,
        dual_source_blending_is_enabled: true,
    ),
    documents: {
        ((1), 0): (
            window_size: (2560, 1242),
            inner_rect: ((0, 0), (2560, 1242)),
            layer: 0,
            pan: (0, 0),
            device_pixel_ratio: 1,
            page_zoom_factor: 1,
            pinch_zoom_factor: 1,
        ),
    },
    resources: (
        font_templates: {
            ((3), 4): Native((
                pathname: "/usr/share/fonts/dejavu/DejaVuSans.ttf",
                index: 0,
            )),
            // .. some more ..
        },
        font_instances: {
            ((3), 324): (
                font_key: ((3), 164),
                size: 1800,
                color: (r: 0, g: 0, b: 0, a: 255),
                bg_color: (r: 0, g: 0, b: 0, a: 0),
                render_mode: Alpha,
                subpx_dir: Horizontal,
                flags: (
                    bits: 6,
                ),
                platform_options: Some((
                    lcd_filter: Legacy,
                    hinting: Light,
                )),
                variations: [
                ],
                transform: (
                    scale_x: 1,
                    skew_x: 0,
                    skew_y: 0,
                    scale_y: 1,
                ),
            ),
            // ... a few hundred more ...
        },
        image_templates: {
            ((3), 254): (
                data: "externals/1",
                descriptor: (
                    format: BGRA8,
                    width: 32,
                    height: 32,
                    stride: Some(128),
                    offset: 0,
                    is_opaque: false,
                ),
                epoch: (0),
                tiling: None,
            ),
            // ... many more ...
        },
    ),
```

### Level 3: render task tree and gpu cache

This level is where things can go wrong, and these are most difficult to track down, especially when external blobs and textures are involved. Structures here are most diverse and complex. Making the RON dumps less readable to human eyes. For example, here is what a batch looks like:
```yaml
    key: (
        kind: Transformable(AxisAligned, BorderCorner),
        blend_mode: PremultipliedAlpha,
        textures: (
            colors: (Invalid, Invalid, Invalid),
        ),
    ),
    instances: [
        (
            data: (39168, 7, 2147483647, 1704039, 243, 0, 0, 0),
        ),
        (
            data: (39168, 7, 2147483647, 1704039, 243, 1, 0, 0),
        ),
    ],
    item_rects: [
        ((0, 148), (2536, 558)),
    ],
```

In order to trigger this sort of capture, user needs to include `CaptureBits::FRAME` flag (or just use `all()`) in the `save_capture` call. It takes longer because all the GPU side caches and targets need to be read back to main memory for saving.

Capturing logic is very careful about preserving the semantics of items. External texture contents, for example, are loaded into textures and provided as external. The difference is that capturing now serves as manager for those, installing its own handlers for output frames and external images.

### Level 4: draw calls and pixels

This level is where we are no longer principally different from any other graphics application/game. WebRender issues a number of OpenGL API instructions for resource updates and draw calls, expecting them to show up on screen. We also provide debug markers and events alongside the actual workload in order to easy navigation of captured content. Several tools exist to intercept these calls and markers.

`apitrace` is a simple program that records the stream of GL commands from the application start. It is somewhat similar to WebRender's recording ability, just on a different level. Whenever we look up the state on a replay, `apitrace` goes from zero to the required draw call by re-issuing all the commands, which can take long time.

`RenderDoc` is a modern GPU inspector supporting many graphics APIs. We use it to capture the GPU state, then find a particularly offending call by either looking at the graphics state, comparing with another GPU capture, or both. It's been truly irreplaceable in our day to day work, and was the primary source of inspiration for our capturing infrastructure.


## Usage

Captures can be replayed with `wrench load <path>` command. The loading will automatically figure out if it's a `SCENE` level capture of a `FRAME` level. In the latter case, no scene is going to be re-build, and Wrench will just show directly what is being captured, freezing the nasty flickering bugs and glitches in place for careful investigation.

Captured RON files are fully readable and editable by the user. When linking to WebRender directly, one can enable "png" feature to get the image contents saved as PNGs, where applicable, for cleaner inspection.

The idea is for captures to be fully portable between operating systems and environments. The only hard constraint is using the exact same version of WebRender, since we are serializing internal structures, and those change from one revision to another.

Firefox integration is coming up soon for all Nightly users. It will be accessible via a hotkey combo (Ctrl + Shift + 3, don't ask me why), without the need to have any special configuration or run parameters.

We want the captures to be made by users and uploaded to servers conveniently, similarly to how the Gecko Profiler plugin works. Given a link, it's easy to post it to a bug, greatly reducing the difficulty for us to fix it.

### Analysis

TL;DR: capturing allows us to slice the full WebRender pipeline state at particular levels, save it to disk in a readable form, and load in a simplified environment for further inspection.

It is made possible by the Rust power (and Servo in particular) of serializing all the things in the world with a simple attribute on top of a type:

> Searching 89 files for "#[cfg_attr(feature = "capture", derive(Deserialize, Serialize)]"
> ...
> 104 matches across 23 files

RON, on the other hand, proves to be useful at representing these vast amounts of Rust types. I hope to see it applied in more projects in the future.

Equipped with capturing ability, now I feel prepared to tackle the range of correctness issues in WebRender preventing us from shipping it in Firefox for millions of users.
