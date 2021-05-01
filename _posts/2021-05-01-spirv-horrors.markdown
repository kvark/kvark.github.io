---
layout: post
title:  "Horrors of SPIR-V"
date:   2021-05-01 11:11:11 -0500
categories: spirv
---

# History

For WebGPU community group, the debate on whether or not we can adopt SPIR-V for the API needs was probably the most contentious of all, it was discussed for years. Me and a lot of other folks, especially from Google, were deeply convinced that SPIR-V is a great portable format. It's binary, well defined (in Khronos) by the very folks who make the hardware for running it, and there is a growing ecosystem around it. What can possibly go wrong?

The other side also had [sound arguments](https://kvark.github.io/webgpu-debate/SPIR-V.component.html). Apple had an evolving proposal for a text based language, which seemed too novel to accept. Eventually, the group agreed on the compromise (called "Tint", [developed by Google](https://docs.google.com/presentation/d/1qHhFq0GJtY_59rNjpiHU--JW4bW4Ji3zWei-gM6cabs/edit#slide=id.p)): have a text format that we fully specify, but keeping SPIR-V semantics for all operations ("bijective to SPIR-V").

Oh, how wrong we were.... Once the language designers hands are untied, they become driven by first principles. These principles are: the users need to write the code, and the implementations need to validate it, and portably translate to MSL, SPIR-V, and HLSL. Decision after decision, it turned out that following SPIR-V semantics was in conflict with these first principles, and the new language, now called WGSL, drifted away from it. The last (desperate, in hindsight) attempt to keep it aligned to SPIR-V, called [preserving developer trust](https://github.com/gpuweb/gpuweb/files/4446559/Preserving.developer.trust.via.careful.SPIR-V.WGSL.conversions.2020-04-06.pdf), did not get universally accepted.

## Experiments

SPIR-V ecosystem did exist, but inside [gfx-rs](https://github.com/gfx-rs/gfx) community we were fairly convinced that we [couldn't proceed](https://gfx-rs.github.io/2019/07/13/javelin.html) with [SPIRV-Cross](https://github.com/KhronosGroup/SPIRV-Cross). This was based on real data from [Dota2 testing](https://gfx-rs.github.io/2018/08/10/dota2-macos-performance.html) on [gfx-portability](https://github.com/gfx-rs/portability), as well as user feedback (linking with C++ is tough). We needed something more flexible, more robust and performant... we needed to write it in Rust (as with all the things)! So the ecosystem argument was known to be weak. Tools *would* have to be developed, and we were ready to do this from the ground up. So shortly after Tint proposal, I started prototyping [Naga](https://github.com/gfx-rs/naga).

This work gave me a deeper insight into how parsing of WGSL is supposed to work (never did parsing before), what the intermediate representation ("IR" for short) needs to look like (never worked on compilers), and most importantly - how to translate it from and to SPIR-V. In the beginning, it was easy, simply because Tint proposal had SPIR-V semantics, and so I just took a subset of SPIR-V concepts into our IR. Slowly but steadily, as the group was discussing different aspects of WGSL, our IR was taking steps away from SPIR-V, and often landed closer to WGSL. It ended up borrowing from both.

# Horrors

Some aspects of SPIR-V seemed entirely strange, and this post was meant to tell that story, instead of derailing into all this historical background. Note that people tend to be very vocal about advantages (or the good parts) of SPIR-V, so here we are focusing on the dark side only. And yes, I like rants, even if some of these horrors will just show my incompetence in the domain.

### Validator

Having `spirv-val` is great. I'd love to see a similar thing for HLSL and MSL. But unfortunately, it's not comprehensive. It's happy to ignore some of the decorations it doesn't recognize (like `RowMajor` on matrix types instead of fields of a structure), even if some of these decorations actually screw up the driver badly. We hit one of these cases, where there was a mismatch between `BufferBlock`, `NonWritable`, and actually accessing the buffer data. We decorated the buffer incorrectly, but validation was happy, and RenderDoc just went totally nuts while debugging through the shader.

### Unique types

When writing SPIR-V, you can't have two integer types of the same width. Or two texture types that match. All non-composite types have to be unique for some reason. We don't have this restriction in Naga IR, it just doesn't seem to make any sense. For example, what if I'm writing a shader, and I basically want to use the same "int32" type for both indices and lengths. I may want to name these types differently (in SPIR-V via `OpName` decorations) and use in different contexts. Nope, forbidden.

### Entry points

Somehow, despite the fact that a module can contain arbitrary number of unrelated entry points, a module with zero entry points is invalid! It took [a lot of blood](https://github.com/gpuweb/gpuweb/pull/1340) to convince the group that this doesn't make sense as a limitation.

Later I was pointed to the fact that all the existing production tools for generating SPIR-V (namely, `glslang` and `dxc`) only do so for a concrete entry point. So the multiple entry points is there in spirit, but discriminated in practice by the driver bugs, as well as this little validation rule.

### Int vs Uint

Integer types in SPIR-V have a sign, but a number of operations don't actually care: `OpIEqual`, `INotEqual`, `OpIAdd`, `OpISub`, `OpIMul`, and possibly others. Naga's type system requires "==" types to match, for example, and so we have to specifically detect the cases where SPIR-V signedness is ignored, and work around it by injecting casts into IR.

This isn't a bad decision for SPIR-V that targets low level GPU machine code. But it isn't helping if you are trying to use SPIR-V as an intermediate userland format.

### I/O structs

A vertex shader can declare the relevant built-ins (like "Position") in the "output" storage class, and write to them. However, it can also declare an output "interface struct" with a bunch of special built-ins and an empty name... Perhaps, it's a mechanism that helps with tessellation shaders. But for us in Naga land that caused more trouble than good.

### Layout sizes

Structs in SPIR-V appear to always be unbound on the right side. Each member has a name, but nothing tells you how big a struct is. If it's in an array, then the array stride would define how big it is. If it's a field in another struct, then you could see how much space is given for it (unless it's the last member). So the struct layout is explicitly provided in an "open-ended" way, and it causes issues for the IR that tries to have full understanding of the types it operates on. It just feels a bit strange that SPIR-V is uber-explicit in the layout, except for this aspect.

### Storage classes

They are simply confusing. What is "UniformConstant"? How is it different from "Uniform"? Storage buffers started off in this class, only to be assigned their own "StorageBuffer" class later on with an extension...
Fortunately, David [explained](https://github.com/gpuweb/gpuweb/issues/1105#issuecomment-702837083) all the classes and suggested a nice scheme for WGSL. In this scheme, all the opaque things, such as samplers and textures, belonged to "Handle" class. I wish SPIR-V had that from the start.

### Variables

SPIR-V presents a nice duopoly between SSA values and variables. Accessing a variable requires `OpLoad`/`OpStore`, so it feels heavier, and conceptually it comes with an associated storage. It's a beautiful scheme! The problem is - most drivers don't care about what *you* think should be a variable. Most just run [variable-eliminating](https://github.com/gpuweb/gpuweb/issues/622#issuecomment-602861860) optimization passes instead.
So this whole distinction is practically useless: the explicit `OpLoad` doesn't mean anything for a local variable.

### Arrays by value

Arrays in C are painful to pass arround. Unfortunately, it's contagious. Somehow, SPIR-V simply doesn't have instructions of dynamically indexing an array that's referred by value. It's happy to dynamically index a vector, or to index an array by pointer to it. But if you have an SSA id for an array, this is asking for trouble.

In Rust or WGSL (but not C, khm), for example, you can have an array stored somewhere, and it doesn't matter if it's in memory or "on the stack". But it does in SPIR-V, even though (as explained in the previous point) the drivers already take the liberty of deciding for you what actually goes into local memory and what not.

A proper workaround for this would be quite complicated. It would detect when/how values should be promoted to variables, and kept as local variables. A lazy workaround would just constantly store+index+load an array whenever it's indexed by value.

### Image layers

When sampling an image that is an array, or getting its dimensions, the layer indices always get appended to the coordinates vector. I understand that SPIR-V did that to comply with GLSL and HLSL. But instead, it could say that a layer can be selected via an `ImageOperand` bit, just like the LOD selection is done. Maybe even always treat is as integer.

### OpFmod

Apparently, `OpFmod` is [different](https://github.com/gpuweb/gpuweb/issues/1696) from any [fmod](https://www.cplusplus.com/reference/cmath/fmod/) definition you can find, including MSL's `fmod` and HLSL's `%` on floats. The thing that matches them is called `OpFrem`. Maybe just an unfortunate choice of words here, or an intentional attempt to break new ground.

### Multiple documents

There is no single SPIR-V spec that you can read. There is SPIR-V format itself, then SPIR-V environment specification for Vulkan, and then the Vulkan spec. It's often unclear where to even start looking for an answer.

For example, it's surprisingly hard to find out what the spec says about out-of-bounds behavior on buffers and storage images. It's "undefined" quite literally - the spec evades talking about this. Vulkan spec has a bit of wording in the "robust buffer access" feature and related extensions, but only about bits touched by the extensions.

## Control flow

Now here is the boss of this mini-game. Control flow in SPIR-V is represented via a graph. Graphs are much more flexible than what humans express in code, called "structural" control flow.

### Naming

> Continue Block: A block containing a branch to an OpLoopMerge instruction’s Continue Target.

The continue block is *not* the block that does the continue. It's the block that jumps to the continue "target", which corresponds to WGSL's [continuing statement](https://github.com/gpuweb/gpuweb/blob/fd63b6628a0fe5a641424bc5872a9d84b7c78440/wgsl/index.bs#L4866). Figuring out this difference took a lot of head scratching.

### Structure

SPIR-V doesn't completely erase the structure of the source shader: all the branching instructions (such as loops, conditions, and switches) are guaranteed to start with a header instruction, and merge into a merge block. Well, except when they don't *really* merge there.

The talented [contributor](https://github.com/MatusT) of the CFG resolution logic in Naga eventually concluded:
> you can’t really rely on the merge block as a point where two paths actually merge

The actual logic of reconstructing the control flow (in a structured way, which our IR expects) becomes [rather messy](https://github.com/gfx-rs/naga/pull/688), full of sacred knowledge and handling of the edge cases. Hopefully, Matus publishes some details about this separately.

### Regions

When we are looking at a block in SPIR-V, and it introduces an SSA id, we may assume that the relevant section of HLSL, MSL, WGSL, or Naga's IR would be able to define this SSA id within the block. But that's not always the case.

With structured control flow, for example, a "default" case of a `switch` will have it's own scope, always. So if it introduces any constants into the namespace, they'll only be visible to the code within this compound statement (of the switch case). But in SPIR-V, the introduced SSA id is visible to all the blocks dominated by the current block. So we hit a situation (on real shaders!) where a switch has only the "default" case, and nothing else. This means the code has to flow through this block, and consequently the ids defined in this block are visible to the blocks after the switch...

There are multiple ways to solve it (or work around it), but neither is elegant.

# Conclusion

I had a chance to work with SPIR-V rather intimately, and I'm becoming convinced that most of the defenders of this format (who like to criticize WGSL in turn) have little experience actually doing anything with it, other than maybe passing it through SPIRV-Cross, or reflecting some of the type information from it. I was one of them.

SPIR-V is probably a good format for what it was made for: driver compilers. It's not as good for the intermediate portable representations of your shaders.