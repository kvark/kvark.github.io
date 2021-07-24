---
layout: post
title:  "Problems with decision making at W3C"
date:   2021-07-24 11:11:11 -0500
categories: web api
---

This post is inspired by [Open Decision-Making](https://web.stanford.edu/~ouster/cgi-bin/decisions.php) by John Ousterhout. Being actively involved in WebGPU design (mostly within W3C) for more than 4.5 years, I developed strong opinions on how the decisions are made. I identified a number of issues, which I want to share here.

# Issues with WebGPU at W3C

## Agreeing on goals

Clearly stating goals isn't always possible. Different parties have different takes on what is good for the Web, mixed with what's good for GPU users. Imagine the goal "need to transfer memory from user to GPU" - the topic that was discussed for eternity ([#138](https://github.com/gpuweb/gpuweb/issues/138), [#154](https://github.com/gpuweb/gpuweb/issues/154), [#491](https://github.com/gpuweb/gpuweb/issues/491), and others).

Agreeing on goals can be more difficult and time consuming than agreeing on solutions. See WGSL FAQ, where we tried to explain the direction we took, and why: [#562](https://github.com/gpuweb/gpuweb/pull/562) followed by [#576](https://github.com/gpuweb/gpuweb/pull/576) and [#687](https://github.com/gpuweb/gpuweb/pull/687), with nothing landing in the end.

## Gathering input

Gathering input requires [investigations](https://github.com/gpuweb/gpuweb/issues?q=label%3Ainvestigation+). Investigations are hard, and people doing them can (sometimes unconciously) present the results skewed towards their emerging positions on the topic. There is a lot of trust involved.

Public voting works best when each voter is independent and well informed. This is very far from reality in practice. For example, Google + Microsoft + Intel make up the majority of the group, all focusing on the same implementation. This makes them naturally correlated, raising the internal correlation within the group.

Being well informed includes understanding of the context. The context can be both deeply technical (the inner spheres of GPU architectures) and deeply philosophical (what is the Web?). In practice the level of understanding varies from one issue to another, and the group is very uneven.

When inviting opinions from the world outside of the working group, these issues are quadrupled. There is no stake, little trust, and a lot of [time waste](https://github.com/gpuweb/gpuweb/issues/566).

## Reaching consensus

Large discussions have limited usefulness and are sometimes harmful in that it requires deep technical expertise and experience in order to *weight* different pros/cons properly. I think this is the biggest problem in both the "Open Decision-Making" post, and the group operation in W3C. You can get people exposed to all of the arguments, but if they weight the strength of the arguments vastly differently, there isn't going to be a good solution.

It's hard to write the guideline on weighting arguments. Essentially it comes down to knowledge and working mechanisms of delegation and trust.

Documenting the arguments is rarely prioritized or done. And when doing so, takes [another round](https://github.com/kvark/webgpu-debate/pull/2) of debate on which arguments are valid. I attempted to fix this by manually [structuring the arguments in ArgDown](http://kvark.github.io/tech/arguments/2020/06/30/technical-discussions.html), but the benefit of this effort is limited if the process is not tightly integrated into the group workflow.

Chairs can influence which topics are discussed by forming the [agenda](https://drive.google.com/drive/u/0/folders/0B6yb23j9HAmDSDNTcWM0a0lxRU0?resourcekey=0-XPJFvypVCQaKRvC0kgMhDg), or by using the "air time" because they naturally need to speak anyway. When doing so, it's easy to forget to "switch hats" and use the chair position to sway the discussion towards their view. There is a lot of responsibility, and it's easy to screw up. Good chairs are hard to find, yet somehow the process of chair selection is neglected.

The implementor buy-in is often lost: everybody feels responsible for the Web, but only a few people actually implement the logic and maintain the code. Interestingly, the future implementors are also often most informed on the technical details.

Sometimes, consensus is reached quick, and it can be a problem. Easy consensus may be a sign of the group not being well-informed, too internally correlated, or even just tired of the process.

### Nemawashi

[Nemawashi](https://en.wikipedia.org/wiki/Nemawashi) technique ruins the transparency of decision making.
Negotiating with each party individually is a great way to:
  - push with the weight of your organization
  - loose the important points/concerns in the process, since the individual discussions are isolated and not very well documented
  - raise the internal correlation of the group

You can *probably* do Nemawashi transparently, by documenting the outcomes of each meeting publicly, and collecting all the arguments into a single shared registry (an ArgDown file!). However, I haven't seen this in real life. The technique naturally paves the way for dark patterns in negotiation.

# On Khronos

I can only judge based on a few meetings and the result APIs, so this opinion is not well informed. I think Khronos WG decision making is largely fine with regards to the guidelines. Participants have their stakes, often well informed, and subgroups are formed for specific issues.

The problem is that the guidelines are insufficient here. Vulkan API suffers from the lack of coherent vision. It often feels like a construct made from blocks designed individually under different constraints. For example, memory management is fully explicit, while tile memory is hidden behind the complicated sub-passes.

So perhaps, in addition to discussing the way to expose each of the HW blocks individually, Khronos could also have a steering subgroup that evens out the API and makes it consistent.

## Conclusion

Decision making is hard when the groups are formed from a diverse set of people. I think we understand the basic principles of what makes decision making good. Applying them in practice is hard.

A general conclusion I draw is that the whole idea of the process/workflow replacing subjective reasoning is flawed. Now, instead of the bugs in one head, you deal with bugs in the whole complex system made of humans. I think the most efficient ways require trust in fully subjective behavior. One can see the forming of sub-groups as an example of this: the group delegates trust to fewer focused individuals. I'd like to see more frameworks based on trust.