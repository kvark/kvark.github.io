---
layout: post
title:  "Missing structure in technical discussions"
date:   2020-06-30 11:11:11 -0500
categories: tech arguments
---

People are amazing creatures. When discussing a complex issue, they are able to keep multiple independent
arguments in their heads, the pieces of supporting and disproving evidence, and can collapse this system
into a concrete solution. We can spend hours navigating through the issue comments on Github, reconstructing
the points of view, and making sense of the discussion. Problem is: we don't actually want to apply this
superpower and waste time nearly as often.

## Problem with technical discussions

Have you heard of `async` in Rust? Ever wondered why the core team opted into a completely new syntax for this feature? Let's dive in and find out! Here is [#57640](https://github.com/rust-lang/rust/issues/57640) with 512 comments, kindly asking everyone to check [#50547](https://github.com/rust-lang/rust/issues/50547) (with just 308 comments) before expressing their point of view. Following this discussion must have been exhausting. I don't know how it would be possible to navigate it without the [summary](https://github.com/rust-lang/rust/issues/57640#issuecomment-456624074) [comments](https://github.com/rust-lang/rust/issues/57640#issuecomment-456625030).

Another example is the loop syntax in WebGPU. Issue [#569](https://github.com/gpuweb/gpuweb/issues/569) has only 70 comments, with multiple attempts to [summarize](https://github.com/gpuweb/gpuweb/issues/569#issuecomment-610041440) the discussion in the middle. It would probably take a few hours at the minimum to get a gist of the group reasoning for somebody from the outside. And that doesn't include the call transcripts.

Github has emojis which allow certain comments to show more support. Unfortunately, our nature is such that comments are getting liked when we agree with them, not when they advance the discussion in a constructive way. They are all over the place and don't really help.

What would help though is having a non-linear structure for the discussion. Trees! They make following HN and Reddit threads much easier, but they too have problems. Sometimes, a really important comment is buried deep in one of the branches. Plus, trees don't work well for a dialog, when there is some back-and-forth between people.

That brings us to the point: most technical discussions are *terrible*. Not in a sense that people can't make good points and progress through it, but rather that there is no structure to a discussion, and it's too hard to follow. What I see in reality is a lot of focus from a very few dedicated people, and delegation by the other ones to those focused. Many views get misrepresented, and many perspectives never heard, because the flow of comments quickly filters out most potential participants.

## Structured discussion

My first stop in the search of a solution was on [Discourse](https://www.discourse.org/). It is successfully used by many communities, including [Rust users](https://users.rust-lang.org/). Unfortunately, it still has linear structure, and doesn't bring a lot to the table on top of Github. Try following [this discussion
](https://users.rust-lang.org/t/rust-2020-growth/34956) about Rust in 2020 for example.

Then I looked at platforms designed specifically for a structured argumentation. One of the most popular today is [Kialo](https://www.kialo.com/). I haven't done a good evaluation on it, but it seemed that Kialo isn't targeted at engineers, and it's a platform that we'd have to register in for use. Wishing to use Markdown with a system like that, I stumbled upon [Argdown](https://argdown.org/), and realized that it concluded my search.

Argdown introduces a syntax for defining the structure of an argument in text. Statements, arguments, propositions, conclusions - it has it all, written simply in your text editor (especially if its VSCode, for which there is a plugin), or in the [playground](https://argdown.org/sandbox/html). It has command-line tools to produce all sorts of derivatives, like dot graphs, web components, JSON, you name it, from an `.argdown` file. Naturally, formatting with Markdown in it is also supported.

That discovery led me to two questions. (1) - what would an *existing* debate look like in such a system? And (2) - how could we shift the workflow towards using one?

So I picked the most contentious topic in WebGPU discussions and tried to reconstruct it. Topic was about choosing the shading language, and why SPIR-V wasn't accepted. It was discussed by the W3C group over the course of 2+ years, and it's evident that there is [some](https://github.com/gpuweb/gpuweb/issues/847#issuecomment-642497578) [misunderstanding](https://news.ycombinator.com/item?id=23089745) of why the decision was made to go with WGSL, taking Google's Tint proposal as a starting point.

I attempted to reconstruct the debate in https://github.com/kvark/webgpu-debate, building the [SPIR-V.argdown](https://github.com/kvark/webgpu-debate/blob/b28252ea99aef8989858ece09f74dc0758e3d46c/arguments/SPIR-V.argdown) with the first version of the argumentation graph, solving (1). The repository accepts pull requests that are checked by CI for syntax correctness, inviting everyone to collaborate, solving (2). Moreover, the artifacts are automatically uploaded to Github-pages, rendering the [discussion](https://kvark.github.io/webgpu-debate/SPIR-V.component.html) in a way that is easy to explore.

## Way forward

I'm excited to have this new way of preserving and growing the structure of a technical debate. We can keep using the code hosting platforms, and arguing on the issues and PR, while solidifying the core points in these `.argdown` files. I hope to see it applied more widely to the workflows of technical working groups.
