---
layout: post
title:  "API Validation"
date:   2021-04-12 11:11:11 -0500
categories: api
---

Who is going to validate the validator?

## Sad experience with APIs

In WebGPU group, as well as Khronos (Vulkan Portability TSG), we design graphics APIs,
and we use a lot of existing ones.
Quite often, an API specification is unclear. This happens with Khronos APIs (see [issues](https://github.com/KhronosGroup/Vulkan-Docs/issues)), and it happens with DirectX and Metal, where the documentation may
just omit a few important details. In either case, the last word is given to the validation,
i.e. "let's see what the validator thinks" (see [Metal maximum buffer size](https://github.com/gpuweb/gpuweb/issues/1371) example).

This is unfortunate for many reasons:
  1. There is no discovery mechanism for validation errors that are not in the documentation. They hit us by surprise.
  1. Validation errors don't provide the full context. For example, the maximum buffer size differs by hardware.
  1. Validation code may be incomplete or buggy (see [VVL issues](https://github.com/KhronosGroup/Vulkan-ValidationLayers/issues)).

Unless you have connections to the teams writing the drivers, API specifications, or validation layers,
consider yourself in a world of pain.
Since Khronos produces standards and not software, the validation layers (or VVL for short) were developed by LunarG,
contracted by Valve. Think about it: a separate organization contributes VVL for Khronos's standard.
Developers at LunarG are equally likely to misinterpret the specification, they are technically users of.
Unlike with D3D12, the GPU-assisted validation in Vulkan came more than 3 years after the specification release.
The whole thing feels more like a "welcome addition" than an integral part of the standard.

## Ideal standard

Let's go back to the question of who is going to validate the validator.
I believe the answer should be "nobody", because the validation is the sanest place
where the exact API constraints can be defined.

Why are we writing the tons and tons of "Valid Usage" rules in Vulkan specification (that are repeated for different entry points)?
Why are we pretending that the specification itself, written in human language, can be the source of truth,
given that we know how human communication can be ambiguous? We are guilty of this in WebGPU as well,
writing "algorithms" in the spec with pseudo-code (like [this one](https://web.archive.org/web/20210315165356/https://gpuweb.github.io/gpuweb/#abstract-opdef-validating-gpusamplerdescriptor)).
And then we write the actual validation inside our browsers, which has to agree with the spec text, and
the CTS is written to assume this agreement and enforce it.

I think, an ideal specification doesn't need to be all formal.
It should provide intuition and understanding of concepts. It should have pictures and examples.
And instead of rigorously specifying the rules, it could just link to... the validation code.
Yes, the reference validator would be an essential product of the working group.
It could be written in an extra concise style, putting focus on read-ability:
```rust
fn validate_sampler_descriptor(&self, device: &GPUDevice, desc: &GPUSamperDescriptor) {
  if !device.is_valid() {
    self.issue_validation_error("device is invalid");
  }
  if desc.lodMinClamp < 0 {
    self.issue_validation_error("lodMinClamp is negative");
  }
  ...
}
```

The point is to have this code runnable.
So it's not a "formal" specification that is interpreted by humans, and can only really be checked by humans.
Instead, it's actual machine code, that is checked by a machine, and also happens to be readable for humans.
There is no ambiguity, the validation code becomes the source of truth for API expectations.
It would be developed in the open and available to anybody, similar to [W3C validator](https://validator.w3.org/).

And as an API user, I'd much rather read this validation code than
interpret the pseudo-code we have in various specifications. And as an API editor, I'd much rather express
the ideas in a language that is formal and checkable. Maybe one day, I'll have a chance to do it this way.